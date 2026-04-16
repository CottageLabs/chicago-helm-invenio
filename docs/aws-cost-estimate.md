# AWS Cost Estimate for Invenio RDM Helm Chart

**Region**: US-EAST-2 (Ohio)
**Date**: January 2026

## Default Deployment Resources

| Component | Replicas | CPU (suggested) | Memory (suggested) |
|-----------|----------|-----------------|-------------------|
| Web | 6 | 500m-1000m each | 500Mi-1Gi each |
| Worker | 2 | 500m-1000m each | 500Mi-1Gi each |
| Worker Beat | 1 | 500m-2000m | 200Mi-500Mi |
| Flower | 1 | 20m-100m | 125Mi-250Mi |
| PostgreSQL | 1+ | varies | varies |
| Redis | 1+ | varies | varies |
| RabbitMQ | 1+ | varies | varies |
| OpenSearch | 1+ | varies | varies |

**Storage**: 10Gi shared volume (ReadWriteMany) + PVCs for each database

## AWS Cost Components

### 1. EKS Cluster
- **Control plane**: $0.10/hour (~$73/month)

### 2. EC2 Worker Nodes (estimate for defaults)
For the default deployment, you'll need roughly:
- **Total CPU**: ~8-12 vCPUs (app pods + databases)
- **Total Memory**: ~16-32 GB

Suggested node configurations:
- **Small**: 3x `t3.large` (2 vCPU, 8GB) ~$180/month
- **Medium**: 2x `m6i.xlarge` (4 vCPU, 16GB) ~$280/month
- **Production**: 3x `m6i.xlarge` ~$420/month

### 3. Storage (EBS)
- **gp3 volumes** for PostgreSQL, OpenSearch, Redis, RabbitMQ PVCs
- Estimate: 50-200 GB total → $4-16/month
- For ReadWriteMany (shared-volume): Consider **EFS** → ~$0.30/GB/month + access charges

### 4. Load Balancer
- **ALB** (if using Ingress): ~$16/month + LCU charges
- **NLB**: ~$16/month + data processing

### 5. Data Transfer
- Egress to internet: $0.09/GB after first 100GB

## Quick Estimate Tool

Use the [AWS Pricing Calculator](https://calculator.aws/#/):

1. Add **Amazon EKS** → 1 cluster
2. Add **Amazon EC2** → Select instance type × count
3. Add **Amazon EBS** → gp3 volumes for each PVC
4. Add **Amazon EFS** → If using ReadWriteMany storage
5. Add **Elastic Load Balancing** → ALB or NLB

## Rough Monthly Estimates

| Tier | Nodes | Total Cost |
|------|-------|------------|
| Dev/Test | 2x t3.medium | ~$150-200/month |
| Small Prod | 3x t3.large | ~$300-400/month |
| Medium Prod | 3x m6i.xlarge | ~$500-700/month |

## Cost Optimization Options

1. **Use managed services instead of in-cluster databases**:
   - **RDS PostgreSQL** instead of bundled PostgreSQL
   - **ElastiCache Redis** instead of bundled Redis
   - **Amazon OpenSearch Service** instead of bundled OpenSearch
   - **Amazon MQ** instead of bundled RabbitMQ

2. **Reserved Instances/Savings Plans**: 30-60% savings for 1-3 year commits

3. **Spot Instances**: For worker nodes (with proper disruption handling)

## Notes

- Prices are estimates based on on-demand pricing as of January 2026
- Actual costs will vary based on usage, data transfer, and configuration choices
- Consider AWS Free Tier eligibility for new accounts
