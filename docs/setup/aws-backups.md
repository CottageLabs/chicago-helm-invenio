# Backups for chicago-invenio

This document describes how the production UChicago InvenioRDM instance is
backed up, and how the backups were set up. For checking backups and
restoring from them, see
[maintenance/backups-and-restores.md](../maintenance/backups-and-restores.md).

## What needs backing up

InvenioRDM has three stores of source-of-truth data:

| Data | Where it lives | Backup mechanism | Schedule (UTC) | Retention |
|---|---|---|---|---|
| Database (records, users, communities, file metadata) | RDS PostgreSQL `chicago-invenio` | RDS automated backups (native) | Daily, 10:09–10:39 window | 14 days, with point-in-time recovery |
| | | AWS Backup, plan `chicago-invenio-rds` | Weekly, Sun 08:00 | 365 days |
| Uploaded files | EFS `fs-0e8f6bacbcc4515bf`, mounted as the `shared-volume` PVC at `/opt/invenio/var/instance/data` (Invenio's `default-location`) | AWS Backup, plan `chicago-invenio-efs` | Daily 08:00, weekly Sun 08:00 | 14 days (daily), 365 days (weekly) |
| View and download statistics | In-cluster OpenSearch, `chicago-invenio-stats-*` and `chicago-invenio-events-stats-*` indices | OpenSearch snapshots to S3 bucket `chicago-invenio-opensearch-snapshots` via the `repository-s3` plugin, scheduled by a Snapshot Management policy | Daily 08:00 | 90 days |

Not backed up, by design:

| Resource | Why |
|---|---|
| ElastiCache Redis | Cache and rate-limit state only |
| RabbitMQ (in-cluster, including its `persistence-invenio-mq-server-0` PVC) | Transient task queue |
| OpenSearch search indices (records, communities, vocabularies, requests, etc.) | Included in the snapshots anyway, but they can always be rebuilt from the database with `invenio rdm rebuild-all-indices` |
| `chicago-invenio-import-data` S3 bucket | One-off migration staging bucket, job complete — candidate for decommissioning |

### Why OpenSearch uses snapshots rather than volume backups

The statistics indices are **not** derivable from the database. Invenio writes
raw usage events to `events-stats-*` indices and aggregates them into
`stats-*` indices; neither exists anywhere else. This deployment's download
history goes back to December 2018 (migrated from the previous system), so
losing OpenSearch would lose that history permanently.

EBS snapshots of the OpenSearch PVCs are not a valid backup. The cluster has
three nodes with shards spread across them, and each volume would be captured
at a slightly different moment — restoring would mean reassembling a cluster
from mismatched disks, with no guarantee it recovers. OpenSearch's snapshot
API is the supported mechanism: it produces a consistent, incremental,
restorable copy per index.

### Restoring the database and files together

The database records which files belong to which record, and the files live
on EFS, so they must be restored to the same point in time. Retention is
aligned so that a matching pair always exists (daily for 14 days, weekly for
a year), and all scheduled backups start at 08:00 UTC. The procedure is in
[maintenance/backups-and-restores.md](../maintenance/backups-and-restores.md#restoring-the-database-and-files-together).

---

## Part 1 — AWS Backup (EFS files and weekly RDS)

Resources are selected by ARN rather than by tag. The account is shared with
another project, and tag-based selection would silently pick up anything that
happened to carry the same tag.

All commands below assume:

```bash
export AWS_PROFILE=<your-profile>
AWS_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
```

and pass `--region us-east-2` explicitly (the CLI may not honour `AWS_REGION`
and fall back to the profile's default region).

### 1.1 Create the AWS Backup IAM service role

Skip if `AWSBackupDefaultServiceRole` already exists (the AWS Backup console
creates it the first time it's used there).

```bash
cat > /tmp/backup-trust-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "backup.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name AWSBackupDefaultServiceRole \
  --assume-role-policy-document file:///tmp/backup-trust-policy.json \
  --tags Key=project,Value=chicago-invenio

aws iam attach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup

aws iam attach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores

rm /tmp/backup-trust-policy.json
```

### 1.2 Create the backup vault

```bash
aws backup create-backup-vault \
  --backup-vault-name chicago-invenio-backup-vault \
  --backup-vault-tags project=chicago-invenio \
  --region us-east-2
```

This uses the AWS-managed `aws/backup` KMS key. Pass `--encryption-key-arn` for
a customer-managed key instead.

Check that EFS and RDS are enabled for AWS Backup in the region (they are by
default):

```bash
aws backup describe-region-settings --region us-east-2 \
  --query 'ResourceTypeOptInPreference.{EFS:EFS,RDS:RDS}'
```

### 1.3 EFS backup plan

Daily backups kept for 14 days, weekly backups kept for a year, matching the
database retention. On Sundays the two rules overlap and AWS Backup takes a
single backup under the rule with the longer retention (the weekly one), so
Sunday is still covered.

Weekly recovery points move to cold storage after 30 days. EFS backups stay
incremental in cold storage, and data only moves there once no warm recovery
point still references it. In practice the current contents of the filesystem
stay in warm storage (the daily backups always reference them) and what
moves to cold is data that has since been deleted or replaced on EFS. AWS
requires `DeleteAfterDays` to be at least 90 days more than
`MoveToColdStorageAfterDays`, and the transition can't be changed for a
recovery point once it has moved.

```bash
cat > /tmp/efs-backup-plan.json <<'EOF'
{
  "BackupPlanName": "chicago-invenio-efs",
  "Rules": [
    {
      "RuleName": "daily",
      "TargetBackupVaultName": "chicago-invenio-backup-vault",
      "ScheduleExpression": "cron(0 8 * * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 480,
      "Lifecycle": { "DeleteAfterDays": 14 }
    },
    {
      "RuleName": "weekly",
      "TargetBackupVaultName": "chicago-invenio-backup-vault",
      "ScheduleExpression": "cron(0 8 ? * SUN *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 480,
      "Lifecycle": { "MoveToColdStorageAfterDays": 30, "DeleteAfterDays": 365 }
    }
  ]
}
EOF

EFS_PLAN_ID=$(aws backup create-backup-plan \
  --backup-plan file:///tmp/efs-backup-plan.json \
  --backup-plan-tags project=chicago-invenio \
  --region us-east-2 \
  --query BackupPlanId --output text)

cat > /tmp/efs-backup-selection.json <<EOF
{
  "SelectionName": "chicago-invenio-efs",
  "IamRoleArn": "arn:aws:iam::${AWS_ACCOUNT}:role/AWSBackupDefaultServiceRole",
  "Resources": [
    "arn:aws:elasticfilesystem:us-east-2:${AWS_ACCOUNT}:file-system/fs-0e8f6bacbcc4515bf"
  ]
}
EOF

aws backup create-backup-selection \
  --backup-plan-id ${EFS_PLAN_ID} \
  --backup-selection file:///tmp/efs-backup-selection.json \
  --region us-east-2

rm /tmp/efs-backup-plan.json /tmp/efs-backup-selection.json
```

The completion window is 8 hours because the first backup copies the whole
filesystem (~720 GB as of September 2026). Later backups are incremental.

If `create-backup-selection` fails with an invalid role error straight after
step 1.1, wait a minute for IAM to propagate and retry.

### 1.4 RDS weekly backup plan

Daily database backups come from RDS's native automated backups, with
14 days of point-in-time recovery (Part 2). This plan adds a weekly snapshot
kept for a year. It's a separate plan from EFS because a plan's
selection applies to all of its rules.

The schedule stays clear of the RDS automated backup window (10:09–10:39 UTC)
and the maintenance window (Thu 04:07–04:37 UTC). AWS recommends not
scheduling AWS Backup jobs within an hour of either, as overlapping jobs can
be delayed or fail.

```bash
cat > /tmp/rds-backup-plan.json <<'EOF'
{
  "BackupPlanName": "chicago-invenio-rds",
  "Rules": [
    {
      "RuleName": "weekly",
      "TargetBackupVaultName": "chicago-invenio-backup-vault",
      "ScheduleExpression": "cron(0 8 ? * SUN *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 120,
      "Lifecycle": { "DeleteAfterDays": 365 }
    }
  ]
}
EOF

RDS_PLAN_ID=$(aws backup create-backup-plan \
  --backup-plan file:///tmp/rds-backup-plan.json \
  --backup-plan-tags project=chicago-invenio \
  --region us-east-2 \
  --query BackupPlanId --output text)

cat > /tmp/rds-backup-selection.json <<EOF
{
  "SelectionName": "chicago-invenio-rds",
  "IamRoleArn": "arn:aws:iam::${AWS_ACCOUNT}:role/AWSBackupDefaultServiceRole",
  "Resources": [
    "arn:aws:rds:us-east-2:${AWS_ACCOUNT}:db:chicago-invenio"
  ]
}
EOF

aws backup create-backup-selection \
  --backup-plan-id ${RDS_PLAN_ID} \
  --backup-selection file:///tmp/rds-backup-selection.json \
  --region us-east-2

rm /tmp/rds-backup-plan.json /tmp/rds-backup-selection.json
```

Unlike the native automated backups, snapshots taken by AWS Backup are **not**
deleted if the RDS instance is deleted.

To find the plan IDs later:

```bash
aws backup list-backup-plans --region us-east-2 \
  --query "BackupPlansList[?starts_with(BackupPlanName, 'chicago-invenio')].[BackupPlanName,BackupPlanId]" \
  --output text
```

### 1.5 Verify

The scheduled jobs are the simplest test: the first daily EFS backup runs at
the next 08:00 UTC, and the first weekly RDS backup the next Sunday at
08:00 UTC. Check on them with:

```bash
aws backup list-backup-jobs \
  --by-backup-vault-name chicago-invenio-backup-vault \
  --region us-east-2 \
  --query 'BackupJobs[*].{Resource:ResourceArn,State:State,Created:CreationDate,Pct:PercentDone}' \
  --output table
```

To test immediately instead, start on-demand jobs. **Do this outside
UChicago working hours:** the first EFS backup copies the whole filesystem
(~720 GB) and takes several hours, and an RDS snapshot of a single-AZ instance
briefly suspends I/O on the database.

```bash
for ARN in \
  arn:aws:elasticfilesystem:us-east-2:${AWS_ACCOUNT}:file-system/fs-0e8f6bacbcc4515bf \
  arn:aws:rds:us-east-2:${AWS_ACCOUNT}:db:chicago-invenio; do
  aws backup start-backup-job \
    --backup-vault-name chicago-invenio-backup-vault \
    --resource-arn ${ARN} \
    --iam-role-arn arn:aws:iam::${AWS_ACCOUNT}:role/AWSBackupDefaultServiceRole \
    --lifecycle DeleteAfterDays=7 \
    --region us-east-2
done
```

### 1.6 Optional: Vault Lock

Vault Lock puts the vault into WORM (write-once-read-many) mode, so recovery
points can't be deleted early — including by a compromised admin credential.

> **This is difficult to undo.** In `compliance` mode, once the grace period
> elapses, retention **cannot be shortened or the lock removed by anyone,
> including AWS**. Start with `governance` mode (removable by a sufficiently
> privileged principal) and only move to `compliance` once the plans are
> settled.

The minimum and maximum must cover every retention period that uses the vault:
the 7-day on-demand test backups in 1.5 up to the 365-day weekly backups.

```bash
# Governance mode (removable) — recommended first
aws backup put-backup-vault-lock-configuration \
  --backup-vault-name chicago-invenio-backup-vault \
  --min-retention-days 7 \
  --max-retention-days 365 \
  --region us-east-2

# Compliance mode (irreversible after --changeable-for-days) — only once
# you're confident:
# aws backup put-backup-vault-lock-configuration \
#   --backup-vault-name chicago-invenio-backup-vault \
#   --min-retention-days 7 \
#   --max-retention-days 365 \
#   --changeable-for-days 3 \
#   --region us-east-2
```

### 1.7 Optional: cross-region copy

A single-region backup doesn't protect against the loss of us-east-2. To copy
recovery points to another region, create a vault there (step 1.2 with
`--region us-west-2`) and add `CopyActions` to the relevant rules:

```json
"CopyActions": [
  {
    "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:<aws-account-id>:backup-vault:chicago-invenio-backup-vault-dr",
    "Lifecycle": { "DeleteAfterDays": 365 }
  }
]
```

Cross-region copies add data transfer and a second copy of storage.

---

## Part 2 — RDS native automated backups

Automated backups are enabled when the instance is created (see the RDS
section of [aws-setup.md](aws-setup.md), which uses 14 days' retention and deletion
protection). Retention is 14 days to match the daily EFS backups. The
production instance was originally created with 7, and was raised with:

```bash
aws rds modify-db-instance --region us-east-2 \
  --db-instance-identifier chicago-invenio \
  --backup-retention-period 14 \
  --apply-immediately
```

Changing the retention between two non-zero values doesn't cause downtime.
Point-in-time recovery reaches back further as backups accumulate, so the full
14 days is available two weeks after the change.

To check:

```bash
aws rds describe-db-instances --region us-east-2 \
  --db-instance-identifier chicago-invenio \
  --query 'DBInstances[0].{Retention:BackupRetentionPeriod,Window:PreferredBackupWindow,LatestRestorable:LatestRestorableTime,DeletionProtection:DeletionProtection}'
```

---

## Part 3 — OpenSearch snapshots to S3

OpenSearch writes snapshots directly to S3 using the `repository-s3` plugin.
A Snapshot Management (SM) policy — part of the `opensearch-index-management`
plugin, already present in the image — takes and prunes them on a schedule.

### Why IRSA and not Pod Identity

The rest of this cluster uses EKS Pod Identity, but the `repository-s3` plugin
in OpenSearch 2.18 bundles AWS SDK 2.20.86, which predates Pod Identity
support. The plugin does support IAM Roles for Service Accounts (IRSA). The
cluster already has an IAM OIDC provider, so IRSA works without extra cluster
setup.

The plugin runs under the Java security manager and can only read files under
the OpenSearch config directory. So rather than relying on the EKS webhook
(which mounts the token under `/var/run/secrets`), the chart values below
mount a projected service account token directly into
`/usr/share/opensearch/config/aws-irsa/token` and point the plugin at it.

### 3.1 Create the S3 bucket

```bash
aws s3api create-bucket \
  --bucket chicago-invenio-opensearch-snapshots \
  --region us-east-2 \
  --create-bucket-configuration LocationConstraint=us-east-2

aws s3api put-public-access-block \
  --bucket chicago-invenio-opensearch-snapshots \
  --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-tagging \
  --bucket chicago-invenio-opensearch-snapshots \
  --tagging 'TagSet=[{Key=project,Value=chicago-invenio}]'
```

S3 encrypts new objects with SSE-S3 by default.

**Don't add S3 lifecycle rules that expire objects in this bucket.** Snapshots
are incremental and share files between them; only OpenSearch knows which
files are still referenced. Retention is handled by the SM policy in step 3.6.

### 3.2 Create the IAM role for OpenSearch (IRSA)

```bash
OIDC_PROVIDER=$(aws eks describe-cluster --name chicago-invenio --region us-east-2 \
  --query 'cluster.identity.oidc.issuer' --output text | sed 's#^https://##')

cat > /tmp/opensearch-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::${AWS_ACCOUNT}:oidc-provider/${OIDC_PROVIDER}" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_PROVIDER}:sub": "system:serviceaccount:invenio:invenio-opensearch",
          "${OIDC_PROVIDER}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

cat > /tmp/opensearch-s3-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:ListBucketMultipartUploads",
        "s3:ListBucketVersions"
      ],
      "Resource": "arn:aws:s3:::chicago-invenio-opensearch-snapshots"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::chicago-invenio-opensearch-snapshots/*"
    }
  ]
}
EOF

aws iam create-role \
  --role-name chicago-invenio-opensearch-snapshots \
  --assume-role-policy-document file:///tmp/opensearch-trust-policy.json \
  --tags Key=project,Value=chicago-invenio

aws iam put-role-policy \
  --role-name chicago-invenio-opensearch-snapshots \
  --policy-name s3-snapshot-repository \
  --policy-document file:///tmp/opensearch-s3-policy.json

rm /tmp/opensearch-trust-policy.json /tmp/opensearch-s3-policy.json
```

### 3.3 Configure the OpenSearch subchart

In `values-uchicago.yaml` (committed), the `opensearch:` section is as follows.
The `opensearch.yml` block **replaces** the one in `charts/invenio/values.yaml`,
so it repeats the existing lines and adds the `s3.client.default.*` settings.

```yaml
opensearch:
  enabled: true
  ## Install repository-s3 at container start, for snapshots to S3.
  ## See docs/setup/aws-backups.md.
  plugins:
    enabled: true
    installList:
      - repository-s3
  ## Dedicated service account, matched by the IRSA role's trust policy.
  rbac:
    create: true
    serviceAccountName: invenio-opensearch
  config:
    opensearch.yml: |
      cluster.name: invenio-opensearch
      network.host: 0.0.0.0
      # FIXME: re-enable security and configure TLS/auth
      plugins:
        security:
          disabled: true
      s3.client.default.region: us-east-2
      # Regional endpoint: the global s3.amazonaws.com endpoint 307-redirects
      # requests for buckets outside us-east-1 until DNS has propagated.
      s3.client.default.endpoint: s3.us-east-2.amazonaws.com
      # Relative to the config dir; see extraVolumeMounts below
      s3.client.default.identity_token_file: aws-irsa/token
  ## A newly started node only becomes Ready once the cluster is green, so a
  ## rolling restart waits for shards to recover before restarting the next
  ## node (the chart default only checks the port is open). After the first
  ## green, the probe only checks that HTTP responds, so a cluster that later
  ## goes yellow doesn't pull every node out of the Service.
  ## Discovery uses the headless service, which includes not-ready pods, so a
  ## waiting node can still join and recover shards.
  ## If a rollout stalls because the cluster can't reach green for an
  ## unrelated reason, see docs/maintenance/opensearch.md.
  readinessProbe:
    # null removes the chart's default tcpSocket check (Helm merges maps,
    # and a probe can only have one handler)
    tcpSocket: null
    exec:
      command:
        - sh
        - -c
        - |
          START_FILE=/tmp/.opensearch-reached-green
          if [ -f "$START_FILE" ]; then
            curl -sf -o /dev/null http://localhost:9200/
          else
            curl -sf -o /dev/null 'http://localhost:9200/_cluster/health?wait_for_status=green&timeout=1s' \
              && touch "$START_FILE"
          fi
    periodSeconds: 5
    timeoutSeconds: 5
    failureThreshold: 3
  ## IRSA web identity token, mounted inside the config dir so the
  ## repository-s3 plugin is allowed to read it under the security manager.
  ## AWS_ROLE_ARN is set in values-uchicago-private.yaml.
  extraVolumes:
    - name: aws-irsa-token
      projected:
        sources:
          - serviceAccountToken:
              audience: sts.amazonaws.com
              expirationSeconds: 86400
              path: token
  extraVolumeMounts:
    - name: aws-irsa-token
      mountPath: /usr/share/opensearch/config/aws-irsa
      readOnly: true
```

What each part is for:

- `plugins` — installs `repository-s3` when each container starts.
- `rbac` — creates the `invenio-opensearch` service account, which the IRSA
  role's trust policy (step 3.2) is scoped to. Without it, pods run as the
  namespace's `default` service account.
- `s3.client.default.endpoint` — the plugin otherwise uses the global
  `s3.amazonaws.com` endpoint, which answers requests for a newly created
  bucket outside us-east-1 with `307 Temporary Redirect` until DNS has
  propagated (up to 24 hours). The plugin doesn't follow the redirect, so
  verification fails with *"Please re-send this request to the specified
  temporary endpoint"*. The regional endpoint avoids this entirely.
- `s3.client.default.identity_token_file` + `extraVolumes`/`extraVolumeMounts`
  — the IRSA token, mounted inside the config directory where the plugin is
  allowed to read it. Setting at least one IRSA option is also what makes the
  plugin read `AWS_ROLE_ARN` from the environment.
- `readinessProbe` — makes rolling restarts wait for `green`; see
  [maintenance/opensearch.md](../maintenance/opensearch.md#rolling-restarts-and-the-readiness-probe).

The role ARN contains the account ID, so it goes in
`values-uchicago-private.yaml` (gitignored). The plugin reads `AWS_ROLE_ARN`
from the environment:

```yaml
opensearch:
  extraEnvs:
    - name: AWS_ROLE_ARN
      value: arn:aws:iam::<aws-account-id>:role/chicago-invenio-opensearch-snapshots
```

Also add that block to `values-uchicago-private.yaml.example`.

Notes:

- With `plugins.enabled`, the chart runs `opensearch-plugin install` every
  time a pod starts, downloading the plugin from `artifacts.opensearch.org`.
  If that becomes a problem (slow starts, or no egress), build a custom image
  `FROM opensearchproject/opensearch:2.18.0` with the plugin preinstalled.
- The plugin version always matches the running OpenSearch version, so an
  OpenSearch upgrade picks up the matching plugin automatically.

### 3.4 Roll out

Check what the upgrade will change first, as in
[maintenance/upgrades.md](../maintenance/upgrades.md#3-check-what-the-upgrade-will-change):
the Helm release can lag behind the repo, so an upgrade may apply more than
you expect.

Then, with the cluster `green`:

```bash
kubectl -n invenio exec invenio-opensearch-master-0 -c opensearch -- \
  curl -s localhost:9200/_cat/health?v

helm upgrade invenio ./charts/invenio -n invenio \
  -f values-uchicago.yaml -f values-uchicago-private.yaml

kubectl -n invenio rollout status statefulset/invenio-opensearch-master
```

Expect 10–15 minutes for the three nodes. Confirm the plugin is loaded on all
three:

```bash
kubectl -n invenio exec invenio-opensearch-master-0 -c opensearch -- \
  curl -s 'localhost:9200/_cat/plugins?v&h=name,component' | grep repository-s3
```

The OpenSearch readiness probe (3.3) makes each node wait for the cluster to
be `green` before Kubernetes restarts the next one. See
[maintenance/opensearch.md](../maintenance/opensearch.md#rolling-restarts-and-the-readiness-probe)
for how it works and what to do if a rollout stalls.

### 3.5 Register the snapshot repository

The commands in this and later sections run `curl` inside an OpenSearch pod
(the security plugin is disabled, so no credentials are needed). Define a
helper for convenience:

```bash
os() { kubectl -n invenio exec invenio-opensearch-master-0 -c opensearch -- \
  curl -s -H 'Content-Type: application/json' "$@"; }

os -X PUT 'localhost:9200/_snapshot/s3-snapshots' -d '{
  "type": "s3",
  "settings": {
    "bucket": "chicago-invenio-opensearch-snapshots",
    "base_path": "invenio-opensearch"
  }
}'

# Checks every node can write to and read from the bucket
os -X POST 'localhost:9200/_snapshot/s3-snapshots/_verify?pretty'
```

A verify failure mentioning credentials or `AccessDenied` usually means a
mismatch between the service account name and the role's trust policy, or a
missing `AWS_ROLE_ARN`.

### 3.6 Create the Snapshot Management policy

Daily snapshots of all Invenio indices, kept for 90 days. Snapshots only copy
primary shards and are incremental: old monthly stats indices don't change,
so each daily snapshot only uploads what's new.

```bash
os -X POST 'localhost:9200/_plugins/_sm/policies/daily-snapshots' -d '{
  "description": "Daily snapshot of chicago-invenio indices to S3 (see docs/setup/aws-backups.md)",
  "creation": {
    "schedule": { "cron": { "expression": "0 8 * * *", "timezone": "UTC" } }
  },
  "deletion": {
    "schedule": { "cron": { "expression": "0 9 * * *", "timezone": "UTC" } },
    "condition": { "max_age": "90d", "min_count": 7 }
  },
  "snapshot_config": {
    "repository": "s3-snapshots",
    "indices": "chicago-invenio-*",
    "include_global_state": false,
    "ignore_unavailable": true,
    "partial": false
  }
}'
```

`min_count` keeps at least 7 snapshots, even if they're older than 90 days
(e.g. if snapshotting stops for a while).

### 3.7 Verify

Take one snapshot by hand rather than waiting for 08:00:

```bash
os -X PUT 'localhost:9200/_snapshot/s3-snapshots/manual-initial?wait_for_completion=true' -d '{
  "indices": "chicago-invenio-*",
  "include_global_state": false
}' | head -c 600; echo

os 'localhost:9200/_cat/snapshots/s3-snapshots?v'

# SM policy status, including the last run and any failure
os 'localhost:9200/_plugins/_sm/policies/daily-snapshots/_explain?pretty'
```

The manual snapshot isn't managed by the SM policy. Delete it once the
scheduled ones are running:
`os -X DELETE localhost:9200/_snapshot/s3-snapshots/manual-initial`.

---

## Part 4 — Checking and restoring

Checking that backups ran, and restoring from them, is covered in
[maintenance/backups-and-restores.md](../maintenance/backups-and-restores.md).

---

## Cost

Rough monthly estimates at us-east-2 list prices, as of September 2026:

| Item | Estimate |
|---|---|
| EFS backups (~720 GB current contents at $0.05/GB-month warm, plus data deleted or replaced on EFS during the year, mostly at $0.01/GB-month cold) | ~$40–50, growing with the repository |
| RDS weekly snapshots, 1 year (incremental, 20 GB instance) | < $5 |
| OpenSearch snapshots in S3 (~7 GB of primary shards, incremental) | < $1 |

Backups are incremental, so a year of weekly EFS recovery points costs
roughly one copy of the current filesystem plus whatever was deleted or
replaced during the year, not 52 full copies. The cost mostly tracks the size
of the repository itself.

Cold storage (step 1.3) only reduces the cost of data that no longer exists on
EFS. Since the repository mostly gains files rather than losing them, expect
a modest saving. Restoring from a recovery point more than 30 days old is
slower and incurs a cold-storage restore fee — see
[AWS Backup pricing](https://aws.amazon.com/backup/pricing/).

Tag all resources `project=chicago-invenio` (as the commands above do) so they
appear in the project's Cost Explorer view.

See also [OpenSearch disk usage](../maintenance/opensearch.md#disk-usage):
the OpenSearch volumes need resizing before they fill, or rolling restarts
will stall.

---

## Implementation record

What was actually set up, so the current state can be checked against this
document. Account-specific values (account ID, role ARNs) are deliberately
left out.

### 25 September 2026

| Step | Resource | Notes |
|---|---|---|
| 1.1 | IAM role `AWSBackupDefaultServiceRole` | With `AWSBackupServiceRolePolicyForBackup` and `…ForRestores`. Created at 14:48 UTC by an earlier draft of this procedure, then kept, as it matches step 1.1 exactly |
| 1.2 | Backup vault `chicago-invenio-backup-vault` | AWS-managed `aws/backup` KMS key; created at the same time as the role. Vault Lock not enabled |
| 1.3 | Backup plan `chicago-invenio-efs` (ID `b2dc5093-ca67-42c7-a9d1-4a7189de814e`) | Selection by ARN for `fs-0e8f6bacbcc4515bf`. The earlier draft had also tagged the filesystem `backup=true`; that tag was removed as it isn't used |
| 1.4 | Backup plan `chicago-invenio-rds` (ID `2d4b1503-f2dc-418b-ad3c-bad8023d2cfc`) | Selection by ARN for `db:chicago-invenio` |
| 1.5 | — | On-demand test backups deliberately **not** run (set up during UChicago working hours). First scheduled EFS backup due 26 Sep 08:00 UTC; first weekly RDS backup 27 Sep 08:00 UTC |
| 2 | RDS `chicago-invenio` backup retention 7 → 14 days | `modify-db-instance --apply-immediately`; no restart. The full 14-day window is available from about 9 October 2026 |
| 3.1 | S3 bucket `chicago-invenio-opensearch-snapshots` | Public access blocked, SSE-S3, tagged |
| 3.2 | IAM role `chicago-invenio-opensearch-snapshots` | IRSA trust for `system:serviceaccount:invenio:invenio-opensearch`; inline policy `s3-snapshot-repository` |
| 3.3–3.4 | Helm revision 45 | Plugin, service account, IRSA token and role ARN. Also brought the release in line with `web` HPA `maxReplicas: 6`, which had been applied with `kubectl patch` on 15 Sep after revision 44. The default port-only readiness probe let the rollout restart nodes before shards recovered; the cluster went briefly `red` and recovered to `green` at 15:36 UTC |
| 3.3–3.4 | Helm revision 46 | Added the regional S3 endpoint (the repository verify in 3.5 had failed with a 307 from the global endpoint) and the wait-for-green readiness probe. Rollout confirmed to hold each new node unready until `green` |
| 3.5 | Repository `s3-snapshots` (bucket `chicago-invenio-opensearch-snapshots`, base path `invenio-opensearch`) | `_verify` succeeded on all three nodes via IRSA |
| 3.6 | SM policy `daily-snapshots` | First scheduled snapshot 26 Sep 08:00 UTC |
| 3.7 | Snapshot `manual-initial` | 232 indices, 232/232 shards, ~3 minutes, 7.4 GB in S3. Not managed by the SM policy: delete it once the scheduled snapshots are running |

Still to do:

- Check the first scheduled runs: EFS (26 Sep), OpenSearch (26 Sep), RDS weekly (27 Sep), with the commands in [maintenance/backups-and-restores.md](../maintenance/backups-and-restores.md#checking-backups-ran).
- Delete the `manual-initial` snapshot once scheduled snapshots exist.
- First quarterly test restore of each ([maintenance/backups-and-restores.md](../maintenance/backups-and-restores.md#restoring)).

