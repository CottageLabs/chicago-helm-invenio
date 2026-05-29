# Upgrade procedures

This document covers two independent upgrade concerns:

1. [InvenioRDM application release](#inveniordm-application-release) — deploying a new version of the application or chart
2. [EKS Kubernetes version upgrade](#eks-kubernetes-version-upgrade) — the underlying cluster runtime

---

## InvenioRDM application release

Use this procedure to deploy a new version of the application image, update
chart values, or upgrade the Helm chart itself.

### 1. Build and push the application image

If you are not set up with the gpg password manager, `docker login` may fail with a "credential store error". To work around this, you can disable the credential store for Docker by creating a config file with an empty `credsStore`:

```bash 
mkdir -p ~/.docker
  echo '{"credsStore": ""}' > ~/.docker/config.json
```

then

```bash
AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)
ECR="${AWS_ACCOUNT}.dkr.ecr.us-east-2.amazonaws.com"

AWS_PROFILE=<your-profile> aws ecr get-login-password --region us-east-2 \
  | docker login --username AWS --password-stdin ${ECR}

cd ../chicago-invenio
rm Pipfile.lock  # ensure dependencies are up to date
invenio-cli packages lock
docker build -t chicago-invenio:<version> .
docker tag chicago-invenio:<version> ${ECR}/chicago-invenio:<version>
docker push ${ECR}/chicago-invenio:<version>
```

Update `values-uchicago.yaml` with the new image tag and commit/push before proceeding.

### 2. Update chart dependencies (if the chart version changed)

```bash
helm dependency update charts/invenio
```

### 3. Dry-run to check for issues

```bash
helm upgrade invenio charts/invenio \
  --namespace invenio \
  --values values-uchicago.yaml \
  --values values-uchicago-private.yaml \
  --dry-run
```

Review the diff for any unexpected changes before applying.

### 4. Apply the upgrade

```bash
helm upgrade invenio charts/invenio \
  --namespace invenio \
  --values values-uchicago.yaml \
  --values values-uchicago-private.yaml
```

### 5. Monitor the rollout

```bash
kubectl get all -n invenio
kubectl rollout status deployment/invenio-web -n invenio
kubectl rollout status deployment/invenio-worker -n invenio
kubectl get pods -n invenio
```

### 6. Run a smoke test

```bash
curl -s -o /dev/null -w "%{http_code}" https://uchicago.cottagelabs.com
```

Expected: `200`. If the site returns an error, roll back:

```bash
helm rollback invenio -n invenio
```

---

## Reinstalling from scratch

Use this procedure when you need a clean reinstall (e.g. after a failed initial
deployment) without losing persistent data in RDS or EFS.

### What persists across a reinstall

| Resource | Why |
|---|---|
| RDS database | External to the chart; not touched by `helm uninstall` |
| EFS shared volume (`shared-volume` PVC) | `helm.sh/resource-policy: keep` annotation |
| Invenio secret (salts) | `helm.sh/resource-policy: keep` annotation |
| RabbitMQ PVC | Managed by the cluster operator, not the Helm chart |

OpenSearch PVCs are **not** kept and must be deleted manually — see below.

### Procedure

```bash
helm uninstall invenio -n invenio

# Delete OpenSearch PVCs (data is rebuilt by the init job)
kubectl delete pvc \
  invenio-opensearch-master-invenio-opensearch-master-0 \
  invenio-opensearch-master-invenio-opensearch-master-1 \
  invenio-opensearch-master-invenio-opensearch-master-2 \
  -n invenio

helm install invenio charts/invenio \
  --namespace invenio \
  --values values-uchicago.yaml \
  --values values-uchicago-private.yaml
```

> **Note:** The init job (`invenio.init: true`) is idempotent — it handles
> pre-existing DB tables, roles, users, and OpenSearch indices gracefully, so
> there is no need to wipe RDS between reinstalls unless you specifically want
> a clean database.

### Monitoring the init job

```bash
kubectl logs -n invenio -l job-name=invenio-install-init -f
```

The job may restart once if OpenSearch isn't ready when it first runs — this is
expected. Once it completes, set `invenio.init: false` and run `helm upgrade` to
prevent it running again on future upgrades.

---

## EKS Kubernetes version upgrade

EKS requires upgrading one minor version at a time (e.g. 1.32 → 1.33 → 1.34).
Repeat the steps in this section for each version step.

> **Before you start:** check the [EKS Kubernetes versions page](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
> for any breaking changes or deprecations in the target version.

### 1. Update `cluster.yaml`

Bump the `metadata.version` field in `cluster.yaml` to the target version,
then commit and push so the file stays in sync with the live cluster.

### 2. Upgrade the control plane

```bash
AWS_PROFILE=<your-profile> eksctl upgrade cluster \
  --name chicago-invenio \
  --version <target-version> \
  --approve
```

This takes ~10 minutes. The control plane is upgraded first; node groups follow.

### 3. Upgrade the managed node group

```bash
AWS_PROFILE=<your-profile> eksctl upgrade nodegroup \
  --cluster chicago-invenio \
  --name workers \
  --kubernetes-version <target-version>
```

Nodes are replaced one at a time with a rolling update, so Invenio should
remain available throughout.

### 4. Upgrade EKS addons

Upgrade each addon after the node group is ready:

```bash
for ADDON in aws-ebs-csi-driver coredns kube-proxy vpc-cni; do
  AWS_PROFILE=<your-profile> aws eks update-addon \
    --cluster-name chicago-invenio \
    --addon-name ${ADDON} \
    --region us-east-2 \
    --resolve-conflicts OVERWRITE
done
```

### 5. Verify

```bash
kubectl get nodes -o wide          # all nodes should show the new version
kubectl get pods -A | grep -v Running  # no pods should be stuck
```
