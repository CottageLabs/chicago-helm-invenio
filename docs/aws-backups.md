# AWS Backup for chicago-invenio

This document describes how to set up [AWS Backup](https://docs.aws.amazon.com/aws-backup/)
to protect the volumes behind the production UChicago InvenioRDM instance.

AWS Backup was chosen over Amazon Data Lifecycle Manager (DLM) because DLM only
manages EBS snapshots — it has no EFS support. This deployment has both an EFS
filesystem (shared uploads/records storage) and EBS-backed volumes (OpenSearch),
so a single AWS Backup plan covers both instead of stitching two tools together.

## What's covered

| Resource | Included? | Why |
|---|---|---|
| EFS `fs-0e8f6bacbcc4515bf` (shared Invenio uploads/records, see `k8s/storageclasses.yaml`) | Yes | Primary source-of-truth volume |
| OpenSearch PVC (EBS/gp3, in-cluster subchart) | Yes | Rebuildable from Postgres/EFS, but backing it up avoids a slow reindex after data loss |
| RDS PostgreSQL | No | Already has native automated backups and point-in-time recovery — see the RDS section in `docs/aws-setup.md` |
| ElastiCache Redis | No | Cache/broker only, no source-of-truth data |
| RabbitMQ (in-cluster) | No | Transient queue state |
| `chicago-invenio-import-data` S3 bucket | No | One-off migration staging bucket, job complete — candidate for decommissioning rather than backup |

Resources are selected by tag (`backup=true`), not by hardcoded resource ID, so
any future volume you tag the same way is picked up automatically without
editing the backup plan.

---

## 1. Create the AWS Backup IAM service role

AWS Backup needs a service role with permission to back up and restore the
resource types above.

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

AWS_PROFILE=<your-profile> aws iam create-role \
  --role-name AWSBackupDefaultServiceRole \
  --assume-role-policy-document file:///tmp/backup-trust-policy.json \
  --tags Key=project,Value=chicago-invenio

AWS_PROFILE=<your-profile> aws iam attach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup

AWS_PROFILE=<your-profile> aws iam attach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores

rm /tmp/backup-trust-policy.json
```

Skip this step if `AWSBackupDefaultServiceRole` already exists in the account
(the AWS Backup console creates it automatically the first time you use it there).

---

## 2. Create the backup vault

```bash
AWS_PROFILE=<your-profile> aws backup create-backup-vault \
  --backup-vault-name chicago-invenio-backup-vault \
  --backup-vault-tags Key=project,Value=chicago-invenio \
  --region us-east-2
```

This uses the AWS-managed `aws/backup` KMS key by default. Pass
`--encryption-key-arn` if you want a dedicated customer-managed key instead.

---

## 3. Tag the resources to protect

### EFS filesystem

```bash
AWS_PROFILE=<your-profile> aws efs tag-resource \
  --resource-id fs-0e8f6bacbcc4515bf \
  --tags Key=backup,Value=true \
  --region us-east-2
```

### OpenSearch's EBS volumes

OpenSearch runs as a 3-node StatefulSet, each with its own dynamically
provisioned 8Gi gp3 PVC (`invenio-opensearch-master-invenio-opensearch-master-{0,1,2}`,
labelled `app.kubernetes.io/name=opensearch`). None of the volume IDs are
known ahead of time, so look them up via the bound PersistentVolumes and tag
all three:

```bash
PV_NAMES=$(kubectl -n invenio get pvc \
  -l app.kubernetes.io/name=opensearch \
  -o jsonpath='{.items[*].spec.volumeName}')

for PV_NAME in ${PV_NAMES}; do
  VOLUME_ID=$(kubectl get pv "${PV_NAME}" -o jsonpath='{.spec.csi.volumeHandle}')

  AWS_PROFILE=<your-profile> aws ec2 create-tags \
    --resources ${VOLUME_ID} \
    --tags Key=backup,Value=true \
    --region us-east-2
done
```

If the StatefulSet is later scaled up or down, re-run this loop so the tag
set matches the current PVCs.

---

## 4. Create the backup plan

Two-tier retention: nightly backups for short-term recovery, weekly backups
kept longer for slower-to-notice data issues. The 08:00 UTC schedule runs
off-peak for US Central time.

```bash
cat > /tmp/backup-plan.json <<'EOF'
{
  "BackupPlanName": "chicago-invenio-backup-plan",
  "Rules": [
    {
      "RuleName": "daily",
      "TargetBackupVaultName": "chicago-invenio-backup-vault",
      "ScheduleExpression": "cron(0 8 * * ? *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 180,
      "Lifecycle": { "DeleteAfterDays": 35 }
    },
    {
      "RuleName": "weekly",
      "TargetBackupVaultName": "chicago-invenio-backup-vault",
      "ScheduleExpression": "cron(0 8 ? * SUN *)",
      "StartWindowMinutes": 60,
      "CompletionWindowMinutes": 180,
      "Lifecycle": { "DeleteAfterDays": 90 }
    }
  ]
}
EOF

AWS_PROFILE=<your-profile> aws backup create-backup-plan \
  --backup-plan file:///tmp/backup-plan.json \
  --backup-plan-tags Key=project,Value=chicago-invenio \
  --region us-east-2

rm /tmp/backup-plan.json
```

Note the `BackupPlanId` returned — you'll need it for the next step. If you
missed it:

```bash
BACKUP_PLAN_ID=$(AWS_PROFILE=<your-profile> aws backup list-backup-plans \
  --region us-east-2 \
  --query "BackupPlansList[?BackupPlanName=='chicago-invenio-backup-plan'].BackupPlanId" \
  --output text)
```

### Optional: cross-region copy

A single-region backup doesn't protect against a regional AWS outage. To copy
each recovery point to another region (e.g. us-west-2), add a `CopyActions`
block to a rule before creating the plan:

```json
"CopyActions": [
  {
    "DestinationBackupVaultArn": "arn:aws:backup:us-west-2:<account-id>:backup-vault:chicago-invenio-backup-vault-dr",
    "Lifecycle": { "DeleteAfterDays": 90 }
  }
]
```

This requires a destination vault to already exist in that region (repeat
step 2 with `--region us-west-2`). Skip this if single-region protection is
sufficient for now.

---

## 5. Create the backup selection

This tells the plan which tagged resources to actually back up.

```bash
AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)

cat > /tmp/backup-selection.json <<EOF
{
  "SelectionName": "chicago-invenio-tagged-volumes",
  "IamRoleArn": "arn:aws:iam::${AWS_ACCOUNT}:role/AWSBackupDefaultServiceRole",
  "ListOfTags": [
    {
      "ConditionType": "STRINGEQUALS",
      "ConditionKey": "backup",
      "ConditionValue": "true"
    }
  ]
}
EOF

AWS_PROFILE=<your-profile> aws backup create-backup-selection \
  --backup-plan-id ${BACKUP_PLAN_ID} \
  --backup-selection file:///tmp/backup-selection.json \
  --region us-east-2

rm /tmp/backup-selection.json
```

---

## 6. Optional: Vault Lock (production hardening)

Vault Lock puts the backup vault into WORM (write-once-read-many) mode,
protecting recovery points from deletion — including by a compromised admin
credential or AWS support.

> **This is difficult to undo.** In `compliance` mode, once the grace period
> elapses, retention **cannot be shortened or the lock removed by anyone,
> including AWS** — only lengthened. Use `governance` mode first if you want
> an escape hatch (a sufficiently privileged principal can still remove a
> governance lock) while you confirm the plan behaves as expected, then
> switch to `compliance` once you're confident.

```bash
# Governance mode (removable) — recommended first
AWS_PROFILE=<your-profile> aws backup put-backup-vault-lock-configuration \
  --backup-vault-name chicago-invenio-backup-vault \
  --min-retention-days 35 \
  --max-retention-days 90 \
  --region us-east-2

# Compliance mode (irreversible after the changeable-until window) — only
# once you're confident in the plan:
# AWS_PROFILE=<your-profile> aws backup put-backup-vault-lock-configuration \
#   --backup-vault-name chicago-invenio-backup-vault \
#   --min-retention-days 35 \
#   --max-retention-days 90 \
#   --changeable-for-days 3 \
#   --region us-east-2
```

---

## 7. Verify backups are running

```bash
AWS_PROFILE=<your-profile> aws backup list-backup-jobs \
  --by-backup-vault-name chicago-invenio-backup-vault \
  --region us-east-2 \
  --query 'BackupJobs[*].{Resource:ResourceArn,State:State,Created:CreationDate}'
```

You can also trigger an on-demand backup instead of waiting for the next
scheduled window, e.g. to confirm the plan works immediately after setup:

```bash
AWS_PROFILE=<your-profile> aws backup start-backup-job \
  --backup-vault-name chicago-invenio-backup-vault \
  --resource-arn arn:aws:elasticfilesystem:us-east-2:${AWS_ACCOUNT}:file-system/fs-0e8f6bacbcc4515bf \
  --iam-role-arn arn:aws:iam::${AWS_ACCOUNT}:role/AWSBackupDefaultServiceRole \
  --region us-east-2
```

---

## 8. Test a restore

Untested backups aren't backups. Recommended cadence: **quarterly**.

```bash
# Find a recovery point to restore
AWS_PROFILE=<your-profile> aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name chicago-invenio-backup-vault \
  --region us-east-2 \
  --query 'RecoveryPoints[*].{Arn:RecoveryPointArn,Resource:ResourceArn,Created:CreationDate}'

# Restore the EFS recovery point into a new, separate filesystem for
# verification — do NOT restore over the production filesystem.
AWS_PROFILE=<your-profile> aws backup start-restore-job \
  --recovery-point-arn <recovery-point-arn> \
  --iam-role-arn arn:aws:iam::${AWS_ACCOUNT}:role/AWSBackupDefaultServiceRole \
  --metadata file:///tmp/efs-restore-metadata.json \
  --region us-east-2
```

`efs-restore-metadata.json` for an EFS restore needs at minimum a new
`newFileSystem: "true"`, plus performance/throughput mode — see the
[AWS Backup EFS restore metadata reference](https://docs.aws.amazon.com/aws-backup/latest/devguide/restoring-efs.html)
for the exact fields required at restore time.

Once restored, mount the new filesystem (or a scratch PVC pointing at it) and
spot-check that files are present and readable before deleting the test
filesystem.
