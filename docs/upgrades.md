# Upgrade procedures

This document covers two independent upgrade concerns:

1. [InvenioRDM application release](#inveniordm-application-release) — deploying a new version of the application or chart
2. [EKS Kubernetes version upgrade](#eks-kubernetes-version-upgrade) — the underlying cluster runtime

---

## InvenioRDM application release

Use this procedure to deploy a new version of the application image, update
chart values, or upgrade the Helm chart itself.

### 1. Build and push the application image

```bash
AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)
ECR="${AWS_ACCOUNT}.dkr.ecr.us-east-2.amazonaws.com"

AWS_PROFILE=<your-profile> aws ecr get-login-password --region us-east-2 \
  | docker login --username AWS --password-stdin ${ECR}

cd ../chicago-invenio
docker build -t chicago-invenio:<version> .
docker tag chicago-invenio:<version> ${ECR}/chicago-invenio:<version>
docker push ${ECR}/chicago-invenio:<version>
```

Update `values-uchicago.yaml` with the new image tag before proceeding.

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
