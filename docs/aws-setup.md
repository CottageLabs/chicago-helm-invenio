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

### Add ALB alias record once deployed

Once the Invenio ingress has an ALB address:

```bash
kubectl get ingress invenio -n invenio
```

Because `uchicago.cottagelabs.com` is the zone apex, a CNAME is not permitted.
Use a Route 53 **ALIAS A record** instead:

```bash
ALB_HOSTNAME=<alb-hostname-from-above>
ZONE_ID=<route53-zone-id>

# Get the ALB's hosted zone ID
ALB_ZONE=$(AWS_PROFILE=<your-profile> aws elbv2 describe-load-balancers \
  --region us-east-2 \
  --query "LoadBalancers[?DNSName=='${ALB_HOSTNAME}'].CanonicalHostedZoneId" \
  --output text)

AWS_PROFILE=<your-profile> aws route53 change-resource-record-sets \
  --hosted-zone-id ${ZONE_ID} \
  --change-batch "{
    \"Changes\": [{
      \"Action\": \"CREATE\",
      \"ResourceRecordSet\": {
        \"Name\": \"uchicago.cottagelabs.com.\",
        \"Type\": \"A\",
        \"AliasTarget\": {
          \"HostedZoneId\": \"${ALB_ZONE}\",
          \"DNSName\": \"${ALB_HOSTNAME}\",
          \"EvaluateTargetHealth\": true
        }
      }
    }]
  }"
```

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

## Deploy Invenio

### 1. Create the namespace and secrets

> These are not stored in the repo. Store the passwords in a password manager.

```bash
kubectl create namespace invenio

kubectl create secret generic invenio-db-secret \
  --from-literal=password="<db-password>" \
  --namespace invenio

kubectl create secret generic invenio-mq-secret \
  --from-literal=password="<mq-password>" \
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
