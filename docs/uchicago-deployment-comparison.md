# University of Chicago InvenioRDM Deployment Comparison

**Date**: February 2026
**Purpose**: Cost and architecture comparison for knowledge.uchicago.edu replacement

## Requirements Summary

| Requirement | Value |
|-------------|-------|
| Initial records | 100,000 |
| Initial storage | 500 GB |
| Annual growth | ~100 GB/year |
| Max users | ~2,000 (student system) |
| Availability | High (multi-AZ, out-of-timezone support) |
| Daily indexing | Up to 100 records (peak during submission deadlines) |

## Architecture Overview

Both platforms require the following components:

| Component | Purpose | HA Requirement |
|-----------|---------|----------------|
| Kubernetes cluster | Container orchestration | Multi-AZ nodes |
| PostgreSQL | Primary database | Standby replica |
| OpenSearch | Full-text search, indexing | 3-node cluster |
| Redis | Caching, sessions | Replica |
| Object Storage | File storage (S3-compatible) | Built-in durability |
| Load Balancer | Traffic ingress | Multi-AZ |

## Helm Chart Resource Requirements

The Invenio Helm chart defaults require careful node sizing:

| Component | Instances | Memory Each | Total |
|-----------|-----------|-------------|-------|
| Web + nginx sidecar | 6 | ~750Mi | ~4.5GB |
| Worker | 2 | ~750Mi | ~1.5GB |
| Worker beat | 1 | ~300Mi | ~0.3GB |
| Flower | 1 | ~200Mi | ~0.2GB |
| RabbitMQ (in-cluster) | 1 | ~512Mi | ~0.5GB |
| System overhead (per node) | 3 | ~1GB | ~3GB |
| **Total** | | | **~10GB** |

**Note:** The chart defaults resource requests to `{}` (unset). For production, explicit requests/limits should be configured. During rolling deployments, old and new pods run simultaneously, requiring additional headroom.

**Node sizing options:**

| Option | Nodes | Total RAM | Headroom | Monthly Cost |
|--------|-------|-----------|----------|--------------|
| 3x t3.medium | 3 | 12GB | Tight | $90 |
| **3x t3.large (recommended)** | 3 | 24GB | Comfortable | $180 |
| 4x t3.medium | 4 | 16GB | OK | $120 |

## AWS EKS Estimate

**Region**: US-EAST-2 (Ohio)

| Component | Configuration | Monthly Cost |
|-----------|---------------|--------------|
| EKS Control Plane | Managed, multi-AZ | $73 |
| EKS Nodes | 3x t3.large (8GB RAM) across 3 AZs | $180 |
| OpenSearch Service | 2x t3.small.search (multi-AZ) | $54 |
| RDS PostgreSQL | db.t3.small Multi-AZ | $60 |
| ElastiCache Redis | cache.t3.micro with replica | $24 |
| S3 | 500 GB Standard | $15 |
| ALB | Application Load Balancer | $20 |
| Data Transfer | Estimated egress | $25 |
| **Total** | | **~$451/month** |

### AWS Advantages
- Mature HA guarantees with well-documented SLAs
- Native Gateway API controller support
- Larger ecosystem and community knowledge base
- More regions for disaster recovery options
- Better IAM and security controls
- OpenSearch multi-AZ is cost-effective

### AWS Considerations
- Control plane cost ($73/month)
- More complex IAM setup
- Steeper learning curve

## DigitalOcean Estimate

**Region**: NYC1/NYC3 (New York)

| Component | Configuration | Monthly Cost |
|-----------|---------------|--------------|
| DOKS Control Plane | Managed, free | $0 |
| DOKS Nodes | 3x General Purpose 8GB Droplets | $189 |
| Managed OpenSearch | 3-node General Purpose (HA) | $267 |
| Managed PostgreSQL | 1GB + standby node | $30 |
| Managed Redis | 1GB | $15 |
| Spaces | 500 GB | $10 |
| Load Balancer | Standard | $12 |
| Data Transfer | Included in droplet allocation | $0 |
| **Total** | | **~$478/month** (adjusted for equivalent node sizing)

### DigitalOcean Advantages
- Free Kubernetes control plane
- Simpler pricing model
- Generous included bandwidth
- Lower barrier to entry
- Good developer experience

### DigitalOcean Considerations
- 3-node OpenSearch HA is expensive ($267/month)
- Fewer regions than AWS
- No native ReadWriteMany storage
- Less mature HA guarantees
- Smaller ecosystem

## Cost Comparison Summary

| Platform | Monthly Cost | Annual Cost |
|----------|--------------|-------------|
| **AWS EKS** | ~$451 | ~$5,412 |
| **DigitalOcean** | ~$478 | ~$5,736 |
| **Difference** | $27/month | $324/year |

**Winner: AWS EKS** - Similar cost with better HA guarantees and ecosystem.

## Why AWS Wins for This Use Case

The primary cost driver difference is **managed OpenSearch with HA**:

| Platform | OpenSearch HA Cost |
|----------|-------------------|
| AWS (2x t3.small.search multi-AZ) | $54/month |
| DigitalOcean (3-node General Purpose) | $267/month |

DigitalOcean's free control plane ($73 savings) is offset by their expensive OpenSearch clustering.

### Scenarios Where DigitalOcean Wins

DigitalOcean would be cheaper (~$170/month) if you:
- Accept single-node OpenSearch (no HA)
- Self-host OpenSearch in-cluster
- Use a third-party service like Bonsai (~$50/month)

These options are **not recommended** for a production university system requiring out-of-timezone support.

## 5-Year Cost Projection

Assuming 100 GB/year storage growth:

| Year | Storage | AWS Monthly | DO Monthly |
|------|---------|-------------|------------|
| 1 | 500 GB | $451 | $478 |
| 2 | 600 GB | $454 | $481 |
| 3 | 700 GB | $457 | $484 |
| 4 | 800 GB | $460 | $487 |
| 5 | 900 GB | $463 | $490 |

Storage growth has minimal cost impact on either platform.

## High Availability Mitigations

> **Note:** For a detailed cost-benefit analysis comparing Multi-AZ vs Single-AZ deployments, see [ha-justification.md](ha-justification.md). Single-AZ reduces costs by ~$132/month but requires manual recovery (~1-2 hours) for AZ failures.

### What is an Availability Zone (AZ)?

An AWS region (e.g., us-east-2/Ohio) contains multiple Availability Zones (e.g., us-east-2a, us-east-2b, us-east-2c). Each AZ is a physically separate data center with independent power, cooling, and networking. Spreading resources across AZs ensures a local disaster (fire, flood, power failure) affects only one AZ, not the entire system.

### HA Configuration by Component

| Component | HA Configuration | Protects Against | Failover Time |
|-----------|------------------|------------------|---------------|
| EKS Nodes | 3x t3.large across 3 AZs | Node failure, AZ outage | ~1-2 min (pod reschedule) |
| OpenSearch | 2-node multi-AZ | Node failure, AZ outage | Automatic, near-instant |
| RDS PostgreSQL | Multi-AZ standby | Primary failure, AZ outage | ~60-120 seconds |
| ElastiCache Redis | Replica node | Primary failure | ~seconds |
| S3 | Built-in (11 9s durability) | Data loss, AZ outage | N/A (always available) |
| ALB | Multi-AZ (automatic) | AZ outage | Automatic |
| EKS Control Plane | AWS-managed multi-AZ | Control plane failure | Automatic |

### What This Protects Against

- Single node/instance failures (automatic recovery)
- Single AZ outages (workload continues in other AZs)
- Database corruption (RDS automated backups)
- 3am pages for infrastructure issues

### What This Does NOT Protect Against

| Risk | Mitigation Required |
|------|---------------------|
| Application bugs | Testing, staging environment |
| Misconfiguration | GitOps, change review |
| Region-wide AWS outage | Multi-region (expensive, likely overkill) |
| Data corruption at app level | Application-level backups, S3 versioning |
| RabbitMQ failure (in-cluster) | See below |

### RabbitMQ: The Weak Spot

RabbitMQ runs in-cluster (single instance by default). If it fails:
- Workers can't process tasks until it recovers
- Pod will reschedule, but queue state may be lost

**Options:**

| Option | Cost | Trade-off |
|--------|------|-----------|
| Accept the risk | $0 | Tasks retry after recovery; minor delay |
| Amazon MQ (managed) | ~$30/month | Full HA, managed service |
| RabbitMQ cluster in-chart | $0 | More complex, uses more node resources |

**Recommendation:** For this workload (up to 100 ingests/day at peak), accepting the risk is reasonable. Ingests will queue and complete once RabbitMQ recovers (~1-2 minutes for pod reschedule).

## Recommended AWS Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        AWS Region (us-east-2)               │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   EKS Cluster                        │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐              │    │
│  │  │  Node   │  │  Node   │  │  Node   │              │    │
│  │  │t3.large │  │t3.large │  │t3.large │              │    │
│  │  │  AZ-a   │  │  AZ-b   │  │  AZ-c   │              │    │
│  │  │ - Web   │  │ - Web   │  │ - Web   │              │    │
│  │  │ - Worker│  │ - Worker│  │ - Worker│              │    │
│  │  └─────────┘  └─────────┘  └─────────┘              │    │
│  └─────────────────────────────────────────────────────┘    │
│                              │                              │
│              ┌───────────────┼───────────────┐              │
│              ▼               ▼               ▼              │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐    │
│  │ RDS PostgreSQL│  │  OpenSearch   │  │  ElastiCache  │    │
│  │  (Multi-AZ)   │  │  (Multi-AZ)   │  │    Redis      │    │
│  └───────────────┘  └───────────────┘  └───────────────┘    │
│                              │                              │
│                              ▼                              │
│                      ┌───────────────┐                      │
│                      │      S3       │                      │
│                      │  (11 9s)      │                      │
│                      └───────────────┘                      │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
                      ┌───────────────┐
                      │     ALB       │
                      │ (Gateway API) │
                      └───────────────┘
                               │
                               ▼
                          Internet
```

## Cost Optimization Options (AWS)

### Immediate Savings
1. **Reserved Instances** (1-year, no upfront): ~15-20% savings on EC2/RDS
2. **Savings Plans** (1-year compute): ~20% savings

### Future Considerations
1. **Spot Instances** for worker nodes (with disruption handling)
2. **Graviton instances** (ARM-based): ~20% cheaper, requires testing
3. **S3 Intelligent-Tiering** for infrequently accessed files

### Potential Optimized Cost (with 1-year commitments)
| Component | On-Demand | Reserved/Savings Plan |
|-----------|-----------|----------------------|
| EKS Nodes | $180 | ~$144 |
| RDS | $60 | ~$48 |
| OpenSearch | $54 | ~$43 |
| Other | $157 | $157 |
| **Total** | $451 | **~$392/month** |

## Next Steps

1. **Finalize routing**: Implement Gateway API (preferred) or Ingress
2. **Plan initial import**: Strategy for 100k records bulk import
3. **CI/CD pipeline**: Deployment automation
4. **Monitoring**: CloudWatch, alerts for out-of-timezone support
5. **Backup strategy**: Verify RDS snapshots, S3 versioning

## References

- [AWS Pricing Calculator](https://calculator.aws/#/)
- [DigitalOcean Pricing Calculator](https://www.digitalocean.com/pricing/calculator)
- [Amazon OpenSearch Pricing](https://aws.amazon.com/opensearch-service/pricing/)
- [DigitalOcean Managed OpenSearch](https://www.digitalocean.com/products/managed-databases-opensearch)
