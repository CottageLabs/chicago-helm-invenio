# OpenSearch operations

The in-cluster OpenSearch (3 nodes, StatefulSet `invenio-opensearch-master`)
holds the search indices, which can be rebuilt from the database, and the
view/download statistics, which can't. Stats are backed up by daily snapshots
to S3; see [backups-and-restores.md](backups-and-restores.md).

Check cluster health:

```bash
kubectl -n invenio exec invenio-opensearch-master-0 -c opensearch -- \
  curl -s localhost:9200/_cat/health?v
```

---

## Rolling restarts and the readiness probe

The OpenSearch chart's default readiness probe only checks that port 9200 is
open. During a rolling restart, Kubernetes therefore restarts the next node as
soon as the previous one opens its port — before its shards have recovered.
The first rollout of the backup changes (Helm revision 45, 25 September 2026) hit exactly that: the cluster went `red`
briefly and was down to two nodes with 163 unassigned replicas when the next
node was taken down.

The `readinessProbe` in `values-uchicago.yaml` (added in revision 46; see [setup step 3.3](../setup/aws-backups.md#33-configure-the-opensearch-subchart)) fixes this. A newly started container is only
Ready once the cluster reports `green`; the probe then records that in
`/tmp/.opensearch-reached-green` and from then on only checks that HTTP
responds. So:

- a rolling restart waits for `green` between nodes;
- a running cluster that later goes `yellow` (e.g. a node restarting) doesn't
  pull every node out of the `invenio-opensearch-master` Service that Invenio
  connects through;
- a waiting node can still join the cluster and recover shards, because
  discovery uses the headless service, which includes not-ready pods.

If a rollout stalls on a node that's `Running` but `0/1` Ready, the cluster
can't reach `green`. Find out why before forcing it:

```bash
kubectl -n invenio exec invenio-opensearch-master-0 -c opensearch -- \
  curl -s 'localhost:9200/_cluster/allocation/explain?pretty'
```

A common cause is a node's disk passing the 85% low watermark, so replicas
can't be allocated (see [Disk usage](#disk-usage) below). Once it's
understood, the gate can be released by hand on the waiting pod:

```bash
kubectl -n invenio exec <pod> -c opensearch -- touch /tmp/.opensearch-reached-green
```

When the security plugin is enabled (the `FIXME` in `opensearch.yml`), the
probe's `curl` commands will need credentials and HTTPS.

---

## Disk usage

The OpenSearch PVCs are 8Gi each
and were 57–62% full in September 2026, with the stats indices growing by
several hundred MB a month. At 85% a node stops receiving new shards, the
cluster can't get back to `green`, and rolling restarts stall (see above).
Snapshots of a `yellow` cluster still succeed, since they copy primaries, but
plan a volume resize (the `gp3` StorageClass allows expansion) before then.

To resize, raise the PVC request on each of the three PVCs (they are created
from the StatefulSet's `volumeClaimTemplates`, which can't be changed in place,
so edit the PVCs directly), for example:

```bash
for i in 0 1 2; do
  kubectl -n invenio patch pvc invenio-opensearch-master-invenio-opensearch-master-$i \
    -p '{"spec":{"resources":{"requests":{"storage":"16Gi"}}}}'
done
```

The EBS CSI driver expands the volumes and filesystems online.

Don't change `opensearch.persistence.size` in `values-uchicago.yaml` on its
own: a StatefulSet's `volumeClaimTemplates` can't be changed, so the next
`helm upgrade` would be rejected. To bring the template in line (so that a
recreated PVC gets the new size), delete the StatefulSet object while leaving
its pods and PVCs running, then upgrade:

```bash
kubectl -n invenio delete statefulset invenio-opensearch-master --cascade=orphan
# set opensearch.persistence.size in values-uchicago.yaml, then:
helm upgrade invenio ./charts/invenio -n invenio \
  -f values-uchicago.yaml -f values-uchicago-private.yaml
```
