# Backups and restores

Day-to-day operation of the backups for the production UChicago InvenioRDM
instance: checking they ran, and restoring from them. For what is backed up,
why, and how it was set up, see
[setup/aws-backups.md](../setup/aws-backups.md).

| Data | Backup | Daily | Weekly |
|---|---|---|---|
| Database | RDS automated backups + AWS Backup plan `chicago-invenio-rds` | 14 days, point-in-time | 365 days |
| Uploaded files | AWS Backup plan `chicago-invenio-efs` | 14 days | 365 days (cold after 30) |
| View/download stats | OpenSearch snapshots to S3 (`s3-snapshots` repository, `daily-snapshots` policy) | 90 days | — |

All scheduled backups start at 08:00 UTC.

The commands below assume:

```bash
export AWS_PROFILE=<your-profile>
AWS_ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

# curl inside an OpenSearch pod (the security plugin is disabled)
os() { kubectl -n invenio exec invenio-opensearch-master-0 -c opensearch -- \
  curl -s -H 'Content-Type: application/json' "$@"; }
```

---

## Checking backups ran

AWS Backup jobs (EFS daily/weekly, RDS weekly):

```bash
aws backup list-backup-jobs \
  --by-backup-vault-name chicago-invenio-backup-vault \
  --region us-east-2 \
  --query 'BackupJobs[*].{Resource:ResourceArn,State:State,Created:CreationDate,Pct:PercentDone}' \
  --output table
```

Look for `COMPLETED`. A job left `EXPIRED` didn't start within its start
window; `FAILED` jobs show the reason with
`aws backup describe-backup-job --backup-job-id <id> --region us-east-2`.

RDS automated backups:

```bash
aws rds describe-db-instances --region us-east-2 \
  --db-instance-identifier chicago-invenio \
  --query 'DBInstances[0].{Retention:BackupRetentionPeriod,LatestRestorable:LatestRestorableTime}'
```

`LatestRestorable` should be within the last few minutes.

OpenSearch snapshots:

```bash
os 'localhost:9200/_cat/snapshots/s3-snapshots?v&s=end_epoch'

# SM policy status, including the last run and any failure
os 'localhost:9200/_plugins/_sm/policies/daily-snapshots/_explain?pretty'
```

Each snapshot should be `SUCCESS` with `failed_shards` 0.

---

## Restoring the database and files together

The database records which files belong to which record; the files themselves
are on EFS. When restoring both, restore them to the **same point in time**,
otherwise records can point at files that don't exist (or files exist that no
record references). All scheduled backups start at 08:00 UTC for this reason.
Retention is aligned so that a matching pair always exists:

- **Last 14 days:** a daily EFS recovery point, plus RDS point-in-time
  recovery to that recovery point's exact completion time.
- **Last year:** the weekly EFS and weekly RDS recovery points from the same
  Sunday.

---

## Restoring

Untested backups aren't backups. Test-restore each of the three
**quarterly**, using the test procedures below; they restore into new
resources or renamed indices and don't touch production data.

### Uploaded files (EFS)

List recovery points:

```bash
aws backup list-recovery-points-by-resource --region us-east-2 \
  --resource-arn arn:aws:elasticfilesystem:us-east-2:${AWS_ACCOUNT}:file-system/fs-0e8f6bacbcc4515bf \
  --query 'RecoveryPoints[*].{Arn:RecoveryPointArn,Created:CreationDate,Status:Status}' \
  --output table
```

The Invenio volume is an EFS access point rooted at
`/pvc-619771b2-c302-4e87-9ad6-767289544248` on the filesystem; paths in a
restore are relative to the filesystem root, so they start with that prefix.

AWS Backup EFS restores never overwrite existing files. Restoring to the
**existing** filesystem writes into a new `aws-backup-restore_<timestamp>`
directory at the filesystem root, from which files can be copied back
(e.g. from the terminal pod, after mounting the root). Restoring to a **new**
filesystem creates a separate filesystem entirely — use this for test
restores.

**Item-level restore** (specific files or directories, up to 10 paths) into
the existing filesystem — the common case for "a file was deleted":

```bash
cat > /tmp/efs-restore-metadata.json <<'EOF'
{
  "file-system-id": "fs-0e8f6bacbcc4515bf",
  "newFileSystem": "false",
  "ItemsToRestore": "[\"/pvc-619771b2-c302-4e87-9ad6-767289544248/<path/under/data>\"]"
}
EOF

aws backup start-restore-job --region us-east-2 \
  --recovery-point-arn <recovery-point-arn> \
  --iam-role-arn arn:aws:iam::${AWS_ACCOUNT}:role/AWSBackupDefaultServiceRole \
  --metadata file:///tmp/efs-restore-metadata.json
```

Invenio stores each file under a path derived from its file instance ID
(e.g. `ab/cd/…/data`); the path for a given file is the `uri` column of the
`files_files` table.

**Full test restore** to a new filesystem:

```bash
cat > /tmp/efs-restore-metadata.json <<'EOF'
{
  "newFileSystem": "true",
  "CreationToken": "chicago-invenio-restore-test",
  "Encrypted": "true",
  "PerformanceMode": "generalPurpose"
}
EOF
```

then run the same `start-restore-job`. Spot-check files on the new filesystem
(it needs a mount target in the cluster's subnets to be mounted from a pod),
then delete it — a full copy of ~720 GB costs roughly $200/month in EFS
Standard while it exists.

See the
[AWS Backup EFS restore reference](https://docs.aws.amazon.com/aws-backup/latest/devguide/restoring-efs.html)
for all metadata fields.

### Database (RDS)

Every RDS restore creates a **new** instance; the original is untouched.

Within the last 14 days — point-in-time recovery, e.g. to match an EFS recovery
point's completion time:

```bash
aws rds restore-db-instance-to-point-in-time --region us-east-2 \
  --source-db-instance-identifier chicago-invenio \
  --target-db-instance-identifier chicago-invenio-restore \
  --restore-time <YYYY-MM-DDTHH:MM:SSZ> \
  --db-subnet-group-name <same as source> \
  --vpc-security-group-ids <same as source>
```

Older than 14 days — from a weekly AWS Backup recovery point:

```bash
aws backup list-recovery-points-by-resource --region us-east-2 \
  --resource-arn arn:aws:rds:us-east-2:${AWS_ACCOUNT}:db:chicago-invenio \
  --query 'RecoveryPoints[*].{Arn:RecoveryPointArn,Created:CreationDate}' --output table

# Shows the restore metadata AWS Backup expects, pre-filled from the source
aws backup get-recovery-point-restore-metadata --region us-east-2 \
  --backup-vault-name chicago-invenio-backup-vault \
  --recovery-point-arn <recovery-point-arn>
```

Edit that metadata (set a new `DBInstanceIdentifier`), save it as JSON, and
pass it to `aws backup start-restore-job` as for EFS above.

To switch Invenio over to a restored instance, set
`postgresqlExternal.hostname` in `values-uchicago-private.yaml` to the new
endpoint and `helm upgrade`. Then rebuild the search indices so they match
the restored database:

```bash
kubectl -n invenio exec deploy/invenio-web -c web -- invenio rdm rebuild-all-indices
```

For a test, connect to the restored instance, run a few sanity queries
(record counts, a recent record), then delete it.

### Statistics (OpenSearch)

```bash
os 'localhost:9200/_cat/snapshots/s3-snapshots?v&s=end_epoch'
```

**Test restore** — restore one index under a different name and compare it.
The OpenSearch volumes are only 8Gi each and over half full, so test with a
single small index, not everything:

```bash
os -X POST 'localhost:9200/_snapshot/s3-snapshots/<snapshot>/_restore?wait_for_completion=true' -d '{
  "indices": "chicago-invenio-stats-record-view-2026",
  "include_global_state": false,
  "include_aliases": false,
  "rename_pattern": "chicago-invenio-(.+)",
  "rename_replacement": "restoretest-$1"
}'

os 'localhost:9200/_cat/indices/chicago-invenio-stats-record-view-2026,restoretest-*?v&h=index,docs.count'

os -X DELETE 'localhost:9200/restoretest-*'
```

**Real restore** — an index can't be restored over an open index with the
same name. Delete (or close) the damaged indices first, then restore them
with their aliases:

```bash
os -X DELETE 'localhost:9200/<damaged-index>'

os -X POST 'localhost:9200/_snapshot/s3-snapshots/<snapshot>/_restore' -d '{
  "indices": "<damaged-index>",
  "include_global_state": false
}'
```

Events recorded between the snapshot and the restore are lost for the
restored indices. Search (non-stats) indices don't need restoring from a
snapshot; rebuild them with `invenio rdm rebuild-all-indices`.

For a total loss of the OpenSearch cluster: bring up a fresh cluster with the
same chart values, register the repository ([setup step 3.5](../setup/aws-backups.md#35-register-the-snapshot-repository)), restore
`chicago-invenio-stats-*,chicago-invenio-events-stats-*` from the latest
snapshot, then run `invenio rdm rebuild-all-indices` for the rest.
