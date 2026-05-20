# AWS Setup for uchicago.cottagelabs.com

This document describes how to reproduce the AWS infrastructure and Kubernetes
deployment for the UChicago InvenioRDM instance.

## Prerequisites

- AWS CLI configured with a profile that has admin access to the target account
- `kubectl` installed
- `helm` installed (v3+)
- Docker (for building and pushing the application image)

### Install eksctl

```bash
mkdir -p ~/bin
ARCH=amd64 && PLATFORM=$(uname -s)_$ARCH
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_${PLATFORM}.tar.gz"
tar -xzf eksctl_${PLATFORM}.tar.gz
mv eksctl ~/bin/eksctl
rm eksctl_${PLATFORM}.tar.gz
echo 'export PATH=$PATH:$HOME/bin' >> ~/.bashrc
export PATH=$PATH:$HOME/bin
```

---

## One-time AWS account setup

### Activate cost allocation tag

```bash
AWS_PROFILE=<your-profile> aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=project,Status=Active \
  --region us-east-1
```

### Create ECR repository

```bash
AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)

AWS_PROFILE=<your-profile> aws ecr create-repository \
  --repository-name chicago-invenio \
  --region us-east-2 \
  --image-scanning-configuration scanOnPush=true \
  --tags Key=project,Value=chicago-invenio Key=Name,Value=chicago-invenio
```

The image URI will be:
```
${AWS_ACCOUNT}.dkr.ecr.us-east-2.amazonaws.com/chicago-invenio:<tag>
```

Update `values-uchicago.yaml` with this registry value.

---

## DNS — Route 53

The subdomain `uchicago.cottagelabs.com` is delegated to Route 53 from GoDaddy.
This must be done before requesting the ACM certificate, as ACM validates via DNS.

### Create hosted zone

```bash
AWS_PROFILE=<your-profile> aws route53 create-hosted-zone \
  --name uchicago.cottagelabs.com \
  --caller-reference "chicago-invenio-$(date +%s)" \
  --hosted-zone-config Comment="UChicago InvenioRDM"
```

Note the four NS records returned and add them as NS records for `uchicago`
in GoDaddy. All DNS under `uchicago.cottagelabs.com` is then managed in Route 53.

> **Do not** manually create an A record for `uchicago.cottagelabs.com` — External
> DNS (installed below) will create and manage it automatically after the first
> `helm install`. A manually created record will not have TXT ownership and
> External DNS will not update it when the ALB changes.

---

## TLS — AWS Certificate Manager (ACM)

TLS is handled by ACM (free). The ALB uses ACM certificates natively; no
in-cluster TLS controller is required.

### 1. Request a certificate

```bash
CERT_ARN=$(AWS_PROFILE=<your-profile> aws acm request-certificate \
  --domain-name uchicago.cottagelabs.com \
  --validation-method DNS \
  --region us-east-2 \
  --tags Key=project,Value=chicago-invenio Key=Name,Value=uchicago-cottagelabs-com \
  --query 'CertificateArn' --output text)
echo "Certificate ARN: $CERT_ARN"
```

### 2. Add the DNS validation CNAME to Route 53

```bash
# Get the validation record
AWS_PROFILE=<your-profile> aws acm describe-certificate \
  --certificate-arn $CERT_ARN --region us-east-2 \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
```

Add the returned CNAME to the Route 53 hosted zone. ACM will issue the
certificate automatically — usually within a few minutes of DNS propagating.

### 3. Reference the certificate ARN in `values-uchicago.yaml`

```yaml
ingress:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: <cert-arn>
```

### 4. Verify the certificate is issued

```bash
AWS_PROFILE=<your-profile> aws acm describe-certificate \
  --certificate-arn $CERT_ARN \
  --region us-east-2 \
  --query 'Certificate.{Status:Status,Domain:DomainName,Expiry:NotAfter}' \
  --output table
```

Expected output:
```
--------------------------------------------------------
|                  DescribeCertificate                 |
+---------------------------+----------------+---------+
|          Domain           |    Expiry      | Status  |
+---------------------------+----------------+---------+
|  uchicago.cottagelabs.com |  1793491199.0  |  ISSUED |
+---------------------------+----------------+---------+
```

### 5. Verify DNS and TLS end-to-end

Check NS delegation has propagated:
```bash
dig NS uchicago.cottagelabs.com +short
```

Expected output:
```
ns-1440.awsdns-52.org.
ns-1788.awsdns-31.co.uk.
ns-235.awsdns-29.com.
ns-668.awsdns-19.net.
```

Check the HTTPS connection and certificate:
```bash
curl -v --max-time 15 https://uchicago.cottagelabs.com 2>&1 | \
  grep -E "Connected|SSL connection|subject|issuer|SSL certificate|< HTTP"
```

Expected output:
```
* Connected to uchicago.cottagelabs.com (3.149.89.226) port 443
* SSL connection using TLSv1.2 / ECDHE-RSA-AES128-GCM-SHA256 / prime256v1 / rsaEncryption
*  subject: CN=uchicago.cottagelabs.com
*  issuer: C=US; O=Amazon; CN=Amazon RSA 2048 M01
*  SSL certificate verify ok.
< HTTP/2 200
```

---

## EKS cluster

### Create the cluster

```bash
AWS_PROFILE=<your-profile> eksctl create cluster -f cluster.yaml
```

This takes ~15 minutes and automatically writes the kubeconfig entry.

### Rename the context

eksctl generates a verbose context name in the form
`<user>@<cluster>.<region>.eksctl.io`. Rename it for convenience:

```bash
kubectl config rename-context \
  "$(kubectl config current-context)" \
  chicago-invenio
```

---

## AWS Load Balancer Controller

### 1. Install the EKS Pod Identity Agent addon

```bash
AWS_PROFILE=<your-profile> aws eks create-addon \
  --cluster-name chicago-invenio \
  --addon-name eks-pod-identity-agent \
  --region us-east-2

AWS_PROFILE=<your-profile> aws eks wait addon-active \
  --cluster-name chicago-invenio \
  --addon-name eks-pod-identity-agent \
  --region us-east-2
```

### 2. Create the IAM policy

```bash
curl -sLo /tmp/alb-iam-policy.json \
  https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

AWS_PROFILE=<your-profile> aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy-chicago \
  --policy-document file:///tmp/alb-iam-policy.json \
  --tags Key=project,Value=chicago-invenio
```

### 3. Create the IAM role

```bash
AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)

AWS_PROFILE=<your-profile> aws iam create-role \
  --role-name AWSLoadBalancerControllerRole-chicago \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }' \
  --tags Key=project,Value=chicago-invenio

AWS_PROFILE=<your-profile> aws iam attach-role-policy \
  --role-name AWSLoadBalancerControllerRole-chicago \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT}:policy/AWSLoadBalancerControllerIAMPolicy-chicago
```

### 4. Create service account and pod identity association

```bash
kubectl create serviceaccount aws-load-balancer-controller -n kube-system

AWS_PROFILE=<your-profile> aws eks create-pod-identity-association \
  --cluster-name chicago-invenio \
  --region us-east-2 \
  --namespace kube-system \
  --service-account aws-load-balancer-controller \
  --role-arn arn:aws:iam::${AWS_ACCOUNT}:role/AWSLoadBalancerControllerRole-chicago
```

### 5. Install via Helm

```bash
helm repo add eks https://aws.github.io/eks-charts && helm repo update

VPC_ID=$(AWS_PROFILE=<your-profile> aws eks describe-cluster \
  --name chicago-invenio --region us-east-2 \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=chicago-invenio \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-2 \
  --set vpcId=${VPC_ID}
```

---

## EFS (shared persistent storage)

### 1. Create security group and filesystem

```bash
VPC_ID=$(AWS_PROFILE=<your-profile> aws eks describe-cluster \
  --name chicago-invenio --region us-east-2 \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

EFS_SG=$(AWS_PROFILE=<your-profile> aws ec2 create-security-group \
  --group-name chicago-invenio-efs \
  --description "EFS mount targets for chicago-invenio" \
  --vpc-id ${VPC_ID} \
  --region us-east-2 \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=project,Value=chicago-invenio},{Key=Name,Value=chicago-invenio-efs}]' \
  --query 'GroupId' --output text)

AWS_PROFILE=<your-profile> aws ec2 authorize-security-group-ingress \
  --group-id ${EFS_SG} --protocol tcp --port 2049 \
  --cidr 192.168.0.0/16 --region us-east-2

EFS_ID=$(AWS_PROFILE=<your-profile> aws efs create-file-system \
  --region us-east-2 \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --encrypted \
  --tags Key=project,Value=chicago-invenio Key=Name,Value=chicago-invenio-efs \
  --query 'FileSystemId' --output text)

# Wait for the filesystem to be available before creating mount targets
until AWS_PROFILE=<your-profile> aws efs describe-file-systems \
  --file-system-id ${EFS_ID} --region us-east-2 \
  --query 'FileSystems[0].LifeCycleState' --output text | grep -q available; do
  echo "Waiting for EFS..."; sleep 5
done
```

### 2. Create mount targets in private subnets (one per AZ)

Get the private subnet IDs (those with `MapPublicIpOnLaunch=false`):

```bash
AWS_PROFILE=<your-profile> aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=${VPC_ID}" \
  --query 'Subnets[?MapPublicIpOnLaunch==`false`].{ID:SubnetId,AZ:AvailabilityZone}' \
  --region us-east-2 --output table
```

Create one mount target per AZ:

```bash
for SUBNET in <private-subnet-a> <private-subnet-b> <private-subnet-c>; do
  AWS_PROFILE=<your-profile> aws efs create-mount-target \
    --file-system-id ${EFS_ID} \
    --subnet-id ${SUBNET} \
    --security-groups ${EFS_SG} \
    --region us-east-2
done
```

### 3. Install EFS CSI driver with Pod Identity

```bash
AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)

AWS_PROFILE=<your-profile> aws eks create-addon \
  --cluster-name chicago-invenio \
  --addon-name aws-efs-csi-driver \
  --region us-east-2

# Create IAM role for the EFS CSI controller
AWS_PROFILE=<your-profile> aws iam create-role \
  --role-name AmazonEKS_EFS_CSI_DriverRole-chicago \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }' \
  --tags Key=project,Value=chicago-invenio

AWS_PROFILE=<your-profile> aws iam attach-role-policy \
  --role-name AmazonEKS_EFS_CSI_DriverRole-chicago \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy

AWS_PROFILE=<your-profile> aws eks wait addon-active \
  --cluster-name chicago-invenio \
  --addon-name aws-efs-csi-driver \
  --region us-east-2

AWS_PROFILE=<your-profile> aws eks create-pod-identity-association \
  --cluster-name chicago-invenio \
  --region us-east-2 \
  --namespace kube-system \
  --service-account efs-csi-controller-sa \
  --role-arn arn:aws:iam::${AWS_ACCOUNT}:role/AmazonEKS_EFS_CSI_DriverRole-chicago

kubectl rollout restart deployment efs-csi-controller -n kube-system
kubectl rollout status deployment efs-csi-controller -n kube-system
```

### 4. Apply storage classes

Update the `fileSystemId` in `k8s/storageclasses.yaml` with `${EFS_ID}`, then:

```bash
kubectl apply -f k8s/storageclasses.yaml

# Remove gp2 as default — gp3 (from storageclasses.yaml) is now the default
kubectl patch storageclass gp2 \
  -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "false"}}}'
```

---

## S3 bucket mount (import data)

The terminal pod mounts an S3 bucket at `/mnt/import-data` for staging import
files. The bucket is backed by the Mountpoint for Amazon S3 CSI driver, which
exposes a bucket as a read/write filesystem. It is suitable for dropping in
files and reading them once — not for random writes or database use.

The PersistentVolume, PersistentVolumeClaim, and StorageClass are defined in
`k8s/storageclasses.yaml` and applied in the EFS step above. The terminal
volume mount is configured in `values-uchicago.yaml`.

### 1. Create the S3 bucket

```bash
AWS_PROFILE=<your-profile> aws s3 mb s3://chicago-invenio-import-data \
  --region us-east-2
```

Update the `bucketName` in `k8s/storageclasses.yaml` if you use a different name.

### 2. Grant the node IAM role S3 access

Find the node instance role:

```bash
AWS_PROFILE=<your-profile> aws eks describe-nodegroup \
  --cluster-name chicago-invenio \
  --nodegroup-name workers \
  --region us-east-2 \
  --query 'nodegroup.nodeRole' \
  --output text
```

Add an inline policy named `s3-import-data-access` to that role via the AWS
Console (**IAM → Roles → `<node-instance-role>` → Add permissions → Create
inline policy**):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": "arn:aws:s3:::chicago-invenio-import-data/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::chicago-invenio-import-data"
    }
  ]
}
```

The deployed node instance role is
`eksctl-chicago-invenio-nodegroup-w-NodeInstanceRole-K5x1EY0Kf6Fp`.

### 3. Install the Mountpoint S3 CSI driver

The `chicago-invenio-devops-role` requires additional EKS permissions to manage
addons. Add an inline policy named `eksctl-addon-policy` to that role via the
AWS Console:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeClusterVersions",
        "eks:CreateAddon",
        "eks:DescribeAddon",
        "eks:DescribeAddonVersions",
        "eks:DescribeAddonConfiguration",
        "eks:UpdateAddon",
        "iam:GetOpenIDConnectProvider"
      ],
      "Resource": "*"
    }
  ]
}
```

Then install the addon:

```bash
eksctl create addon \
  --name aws-mountpoint-s3-csi-driver \
  --cluster chicago-invenio \
  --region us-east-2
```

### 4. Apply the storage classes

The S3 StorageClass, PV, and PVC are included in `k8s/storageclasses.yaml`
alongside the EFS and gp3 classes. If already applied in the EFS step, no
further action is needed:

```bash
kubectl apply -f k8s/storageclasses.yaml
```

---

## RDS PostgreSQL

Invenio uses an external RDS PostgreSQL instance rather than the in-cluster subchart,
giving automated backups, snapshots, and point-in-time recovery.

### 1. Create the security group

Allow port 5432 from the EKS shared node security group only.

```bash
VPC_ID=$(AWS_PROFILE=<your-profile> aws eks describe-cluster \
  --name chicago-invenio --region us-east-2 \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

NODE_SG=$(AWS_PROFILE=<your-profile> aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=${VPC_ID}" "Name=group-name,Values=eks-cluster-sg-chicago-invenio-*" \
  --query 'SecurityGroups[0].GroupId' --output text --region us-east-2)

RDS_SG=$(AWS_PROFILE=<your-profile> aws ec2 create-security-group \
  --group-name chicago-invenio-rds \
  --description "RDS PostgreSQL for chicago-invenio" \
  --vpc-id ${VPC_ID} \
  --region us-east-2 \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=project,Value=chicago-invenio},{Key=Name,Value=chicago-invenio-rds}]' \
  --query 'GroupId' --output text)

AWS_PROFILE=<your-profile> aws ec2 authorize-security-group-ingress \
  --group-id ${RDS_SG} \
  --protocol tcp --port 5432 \
  --source-group ${NODE_SG} \
  --region us-east-2
```

### 2. Create the DB subnet group

```bash
AWS_PROFILE=<your-profile> aws rds create-db-subnet-group \
  --db-subnet-group-name chicago-invenio-rds \
  --db-subnet-group-description "RDS subnet group for chicago-invenio" \
  --subnet-ids <private-subnet-a> <private-subnet-b> <private-subnet-c> \
  --tags Key=project,Value=chicago-invenio \
  --region us-east-2
```

(Get the private subnet IDs from the EFS section above.)

### 3. Create the RDS instance

```bash
AWS_PROFILE=<your-profile> aws rds create-db-instance \
  --db-instance-identifier chicago-invenio \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 16 \
  --master-username invenio \
  --manage-master-user-password \
  --db-name invenio \
  --db-subnet-group-name chicago-invenio-rds \
  --vpc-security-group-ids ${RDS_SG} \
  --no-publicly-accessible \
  --storage-type gp3 \
  --allocated-storage 20 \
  --backup-retention-period 7 \
  --deletion-protection \
  --tags Key=project,Value=chicago-invenio Key=Name,Value=chicago-invenio \
  --region us-east-2

# Wait for the instance to be available (~10 minutes)
AWS_PROFILE=<your-profile> aws rds wait db-instance-available \
  --db-instance-identifier chicago-invenio --region us-east-2
```

### 4. Get the endpoint and update values-uchicago-private.yaml

```bash
AWS_PROFILE=<your-profile> aws rds describe-db-instances \
  --db-instance-identifier chicago-invenio --region us-east-2 \
  --query 'DBInstances[0].Endpoint.Address' --output text
```

Update `postgresqlExternal.hostname` in `values-uchicago-private.yaml` with this value.

### 5. Create the Kubernetes DB secret

The master password is stored in AWS Secrets Manager (managed automatically by RDS).
Retrieve it and create the cluster secret:

```bash
SECRET_ARN=$(AWS_PROFILE=<your-profile> aws rds describe-db-instances \
  --db-instance-identifier chicago-invenio --region us-east-2 \
  --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text)

DB_PASSWORD=$(AWS_PROFILE=<your-profile> aws secretsmanager get-secret-value \
  --secret-id ${SECRET_ARN} --region us-east-2 \
  --query 'SecretString' --output text | python3 -c "import sys,json; print(json.load(sys.stdin)['password'])")

kubectl create secret generic invenio-db-secret \
  --from-literal=password="${DB_PASSWORD}" \
  --namespace invenio

# Disable automatic rotation — the password is stored in a static Kubernetes
# secret and pods must be restarted to pick up any change. Without this,
# RDS will rotate the password every 7 days and break the application.
AWS_PROFILE=<your-profile> aws secretsmanager cancel-rotate-secret \
  --secret-id ${SECRET_ARN} --region us-east-2
```

---

## ElastiCache Redis

Invenio uses an external ElastiCache Redis instance rather than the in-cluster subchart.

> **Note:** Redis auth is currently disabled. This is acceptable for a VPC-internal
> cache but should be hardened before production use.

### 1. Create the security group

Allow port 6379 from the EKS shared node security group only.

```bash
VPC_ID=$(AWS_PROFILE=<your-profile> aws eks describe-cluster \
  --name chicago-invenio --region us-east-2 \
  --query 'cluster.resourcesVpcConfig.vpcId' --output text)

NODE_SG=$(AWS_PROFILE=<your-profile> aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=${VPC_ID}" "Name=group-name,Values=eks-cluster-sg-chicago-invenio-*" \
  --query 'SecurityGroups[0].GroupId' --output text --region us-east-2)

REDIS_SG=$(AWS_PROFILE=<your-profile> aws ec2 create-security-group \
  --group-name chicago-invenio-redis \
  --description "ElastiCache Redis for chicago-invenio" \
  --vpc-id ${VPC_ID} \
  --region us-east-2 \
  --tag-specifications 'ResourceType=security-group,Tags=[{Key=project,Value=chicago-invenio},{Key=Name,Value=chicago-invenio-redis}]' \
  --query 'GroupId' --output text)

AWS_PROFILE=<your-profile> aws ec2 authorize-security-group-ingress \
  --group-id ${REDIS_SG} \
  --protocol tcp --port 6379 \
  --source-group ${NODE_SG} \
  --region us-east-2
```

### 2. Create the cache subnet group

```bash
AWS_PROFILE=<your-profile> aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name chicago-invenio-redis \
  --cache-subnet-group-description "ElastiCache subnet group for chicago-invenio" \
  --subnet-ids <private-subnet-a> <private-subnet-b> <private-subnet-c> \
  --tags Key=project,Value=chicago-invenio \
  --region us-east-2
```

(Get the private subnet IDs from the EFS section above.)

### 3. Create the ElastiCache cluster

```bash
AWS_PROFILE=<your-profile> aws elasticache create-cache-cluster \
  --cache-cluster-id chicago-invenio \
  --cache-node-type cache.t3.small \
  --engine redis \
  --num-cache-nodes 1 \
  --cache-subnet-group-name chicago-invenio-redis \
  --security-group-ids ${REDIS_SG} \
  --no-transit-encryption-enabled \
  --tags Key=project,Value=chicago-invenio Key=Name,Value=chicago-invenio \
  --region us-east-2

# Wait for the cluster to be available (~5 minutes)
AWS_PROFILE=<your-profile> aws elasticache wait cache-cluster-available \
  --cache-cluster-id chicago-invenio --region us-east-2
```

### 4. Get the endpoint and update values-uchicago-private.yaml

```bash
AWS_PROFILE=<your-profile> aws elasticache describe-cache-clusters \
  --cache-cluster-id chicago-invenio --region us-east-2 \
  --show-cache-node-info \
  --query 'CacheClusters[0].CacheNodes[0].Endpoint.Address' --output text
```

Update `redisExternal.hostname` in `values-uchicago-private.yaml` with this value.

---

## RabbitMQ (Cluster Operator)

Invenio uses a RabbitMQ instance deployed via the [RabbitMQ Cluster Operator](https://www.rabbitmq.com/kubernetes/operator/operator-overview)
rather than the in-cluster Bitnami subchart. Unlike RDS and ElastiCache, this is
not an external AWS service — it runs entirely within EKS. The operator runs in
`rabbitmq-system` and manages `RabbitmqCluster` resources; the cluster itself runs
in the `invenio` namespace alongside the application.

### 1. Install the operators

The cluster operator manages `RabbitmqCluster` resources. The messaging topology
operator (with cert-manager) manages `User` and `Permission` resources.

```bash
# cert-manager (required by the topology operator)
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true

kubectl wait --for=condition=Available deployment/cert-manager-webhook \
  -n cert-manager --timeout=120s

# RabbitMQ Cluster Operator
helm repo add rabbitmq-operator https://rabbitmq.github.io/rabbitmq-operator && \
  helm repo update rabbitmq-operator

helm install rabbitmq-operator rabbitmq-operator/rabbitmq-cluster-operator \
  -n rabbitmq-system --create-namespace

# RabbitMQ Messaging Topology Operator
kubectl apply -f https://github.com/rabbitmq/messaging-topology-operator/releases/latest/download/messaging-topology-operator-with-certmanager.yaml

kubectl wait --for=condition=Available deployment/messaging-topology-operator \
  -n rabbitmq-system --timeout=120s
```

### 2. Create the credentials secret

The `User` CR imports credentials from `invenio-mq-secret`, which must exist before
applying the manifest. Create it with a strong password:

```bash
kubectl create secret generic invenio-mq-secret \
  --from-literal=username=invenio \
  --from-literal=password="<strong-password>" \
  --namespace invenio
```

> Store this password in your password manager.

### 3. Deploy the RabbitMQ cluster

```bash
kubectl apply -f k8s/rabbitmq.yaml
```

Wait for the cluster to be ready:

```bash
kubectl wait rabbitmqcluster invenio-mq -n invenio \
  --for=condition=AllReplicasReady --timeout=300s
```

Invenio connects as user `invenio` using the password from `invenio-mq-secret`. No further
credential lookup is needed.

---

## External DNS

External DNS automatically keeps the Route 53 alias record in sync with the
ALB address. This means teardown and reinstall require no manual DNS updates.

### 1. Create the IAM role

```bash
AWS_PROFILE=<your-profile> aws iam create-policy \
  --policy-name chicago-invenio-external-dns-policy \
  --tags Key=project,Value=chicago-invenio \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["route53:ChangeResourceRecordSets", "route53:ListTagsForResource"],
        "Resource": "arn:aws:route53:::hostedzone/*"
      },
      {
        "Effect": "Allow",
        "Action": ["route53:ListHostedZones", "route53:ListResourceRecordSets"],
        "Resource": "*"
      }
    ]
  }'

AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)

AWS_PROFILE=<your-profile> aws iam create-role \
  --role-name chicago-invenio-external-dns-role \
  --tags Key=project,Value=chicago-invenio \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "pods.eks.amazonaws.com"},
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }'

AWS_PROFILE=<your-profile> aws iam attach-role-policy \
  --role-name chicago-invenio-external-dns-role \
  --policy-arn arn:aws:iam::${AWS_ACCOUNT}:policy/chicago-invenio-external-dns-policy
```

### 2. Create the service account and pod identity association

```bash
kubectl create serviceaccount external-dns -n kube-system

AWS_PROFILE=<your-profile> aws eks create-pod-identity-association \
  --cluster-name chicago-invenio \
  --region us-east-2 \
  --namespace kube-system \
  --service-account external-dns \
  --role-arn arn:aws:iam::${AWS_ACCOUNT}:role/chicago-invenio-external-dns-role
```

### 3. Install via Helm

```bash
helm repo add external-dns https://kubernetes-sigs.github.io/external-dns/ && helm repo update external-dns

helm install external-dns external-dns/external-dns \
  -n kube-system \
  --set provider.name=aws \
  --set "env[0].name=AWS_DEFAULT_REGION" \
  --set "env[0].value=us-east-2" \
  --set policy=upsert-only \
  --set registry=txt \
  --set txtOwnerId=chicago-invenio \
  --set "domainFilters[0]=uchicago.cottagelabs.com" \
  --set serviceAccount.create=false \
  --set serviceAccount.name=external-dns
```

External DNS reconciles every minute. After a reinstall, the Route 53 record
will update automatically once the new ALB is assigned to the ingress.

---

## Deploy Invenio

### 1. Create the namespace and secrets

Before deploying, complete the RDS, ElastiCache, and RabbitMQ sections above — they
create the required secrets (`invenio-db-secret`, `invenio-mq-secret`) and provide
the values needed in `values-uchicago-private.yaml`.

Copy the example and fill in your values:

```bash
cp values-uchicago-private.yaml.example values-uchicago-private.yaml
# edit values-uchicago-private.yaml
```

Then create the namespace and remaining secrets:

```bash
kubectl create namespace invenio

# invenio-db-secret and invenio-mq-secret are created in the RDS and RabbitMQ
# sections above — ensure those steps are complete before continuing.

# Sysadmin user password (used by the init job to create the admin account)
kubectl create secret generic invenio-sysadmin-secret \
  --from-literal=password="<strong-password>" \
  --namespace invenio

# Basic auth (pre-production only — remove nginx.extraVolumeMounts,
# nginx.extraServerConfig, and web.extraVolumes from values-uchicago.yaml
# when going live, and delete this secret)
HASH=$(openssl passwd -apr1 '<password>')
kubectl create secret generic invenio-basic-auth \
  --from-literal=htpasswd="admin:${HASH}" \
  --namespace invenio
```

### 2. Build and push the application image

```bash
AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)
ECR="${AWS_ACCOUNT}.dkr.ecr.us-east-2.amazonaws.com"

AWS_PROFILE=<your-profile> aws ecr get-login-password --region us-east-2 \
  | docker login --username AWS --password-stdin ${ECR}

cd ../chicago-invenio
docker build -t chicago-invenio:latest .
docker tag chicago-invenio:latest ${ECR}/chicago-invenio:latest
docker push ${ECR}/chicago-invenio:latest
```

### 3. Install the chart

```bash
helm dependency update charts/invenio

helm install invenio charts/invenio \
  --namespace invenio \
  --values values-uchicago.yaml \
  --values values-uchicago-private.yaml
```

> **Note:** `helm install` will time out waiting for the post-install init job
> (which runs DB migrations and can take several minutes). This is expected —
> the job runs successfully in the background. Check its status with:
> ```bash
> kubectl logs job/invenio-install-init -n invenio
> ```
> Once the job shows completion, proceed to step 4.

### 4. After first successful deployment

Set `invenio.init: false` in `values-uchicago.yaml` and upgrade to clear
the failed Helm state and disable the init job for future upgrades:

```bash
helm upgrade invenio charts/invenio \
  --namespace invenio \
  --values values-uchicago.yaml \
  --values values-uchicago-private.yaml
```

### 5. Add the Route 53 alias record

Once `kubectl get ingress invenio -n invenio` shows an ALB address,
follow the DNS section above to create the alias record.
