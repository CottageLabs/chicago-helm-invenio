# DigitalOcean Cost Estimate for Invenio RDM Helm Chart

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

## DigitalOcean Cost Components

### 1. DOKS Kubernetes Cluster
- **Control plane**: **FREE** (major advantage over AWS)
- **HA Control plane** (optional): $40/month

### 2. Droplet Worker Nodes

#### Basic Droplets (Shared vCPU)
| Config | vCPU | RAM | Price |
|--------|------|-----|-------|
| Basic | 2 | 4GB | $20/month |
| Basic | 4 | 8GB | $48/month |

#### General Purpose Droplets (Dedicated vCPU - Recommended)
| Config | vCPU | RAM | Price |
|--------|------|-----|-------|
| General Purpose | 2 | 8GB | $63/month |
| General Purpose | 4 | 16GB | $126/month |

#### CPU-Optimized Droplets (Dedicated vCPU)
| Config | vCPU | RAM | Price |
|--------|------|-----|-------|
| CPU-Optimized | 2 | 4GB | $42/month |
| CPU-Optimized | 4 | 8GB | $84/month |

For the default deployment (8-12 vCPUs, 16-32GB RAM):
- **Dev/Test**: 3x Basic 4GB/2vCPU → $60/month
- **Small Prod**: 3x General Purpose 8GB → $189/month
- **Medium Prod**: 3x General Purpose 16GB → $378/month

### 3. Block Storage (Volumes)
- **Price**: $0.10/GB/month
- Estimate for PostgreSQL, OpenSearch, Redis, RabbitMQ PVCs: 50-200GB → $5-20/month
- **Note**: DigitalOcean Volumes are ReadWriteOnce only. For ReadWriteMany, consider NFS or DigitalOcean Spaces with s3fs.

### 4. Load Balancer
- **Standard**: $12/month
- Includes: SSL termination, Let's Encrypt, HTTP/3 support

### 5. Data Transfer
- **Included**: 2,000-6,000 GiB/month (varies by droplet size)
- **Overage**: $0.01/GiB
- **VPC traffic**: FREE

## Quick Estimate Tool

Use the [DigitalOcean Pricing Calculator](https://www.digitalocean.com/pricing/calculator):

1. Add **Kubernetes** → Select node pool size and count
2. Add **Volumes** → Block storage for each PVC
3. Add **Load Balancer** → If exposing via LB
4. Review bandwidth allocation

## Rough Monthly Estimates

| Tier | Nodes | Est. Total |
|------|-------|------------|
| Dev/Test | 3x Basic 4GB | ~$80-100/month |
| Small Prod | 3x General Purpose 8GB | ~$210-250/month |
| Medium Prod | 3x General Purpose 16GB | ~$400-450/month |

## Comparison with AWS (US-EAST-2)

| Component | AWS | DigitalOcean | Savings |
|-----------|-----|--------------|---------|
| K8s Control Plane | $73/month | FREE | $73/month |
| 3x Medium Nodes | ~$280/month | ~$189/month | ~$91/month |
| 100GB Storage | ~$8/month | ~$10/month | -$2/month |
| Load Balancer | ~$16/month | $12/month | $4/month |
| **Total (Small Prod)** | **~$377/month** | **~$211/month** | **~44% savings** |

## Cost Optimization Options

1. **Use DigitalOcean Managed Databases** (instead of in-cluster):
   - Managed PostgreSQL: Starting at $15/month
   - Managed Redis: Starting at $15/month
   - Benefits: Automated backups, standby nodes, maintenance

2. **DigitalOcean Spaces** (S3-compatible object storage):
   - $5/month for 250GB + 1TB transfer
   - Good for file storage instead of block volumes

3. **Reserved Droplets**: Contact DigitalOcean for volume discounts

## Considerations

### Advantages of DigitalOcean
- Free Kubernetes control plane
- Simpler pricing model
- Generous bandwidth included
- Lower barrier to entry

### Considerations
- Fewer regions than AWS (no Ohio equivalent - closest is NYC or Toronto)
- No native ReadWriteMany storage (need workarounds)
- Smaller ecosystem of managed services
- Less granular IAM controls

## DigitalOcean Regions

Closest regions to US-EAST-2 (Ohio):
- **NYC1, NYC2, NYC3** - New York
- **TOR1** - Toronto

## Sources

- [DigitalOcean Kubernetes Pricing](https://www.digitalocean.com/pricing/kubernetes)
- [DigitalOcean Droplet Pricing](https://www.digitalocean.com/pricing/droplets)
- [DigitalOcean Pricing Calculator](https://www.digitalocean.com/pricing/calculator)
- [DigitalOcean Pricing Documentation](https://docs.digitalocean.com/products/kubernetes/details/pricing/)
