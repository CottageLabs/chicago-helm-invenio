# Azure AKS Cost Estimate for Invenio RDM Helm Chart

**Region**: East US 2
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

## Azure Cost Components

### 1. AKS Cluster Control Plane

| Tier | Cost | SLA | Best For |
|------|------|-----|----------|
| **Free** | $0/month | None | Dev/Test |
| **Standard** | ~$72/month | 99.9-99.95% | Production |
| **Premium** | ~$72/month + LTS | 99.9-99.95% | Long-term support needs |

### 2. VM Worker Nodes (Dsv5 Series - Recommended)

| VM Size | vCPU | Memory | On-Demand | Spot (~75% off) |
|---------|------|--------|-----------|-----------------|
| Standard_D2s_v5 | 2 | 8 GB | $70/month | ~$17/month |
| Standard_D4s_v5 | 4 | 16 GB | $141/month | ~$34/month |
| Standard_D8s_v5 | 8 | 32 GB | $281/month | ~$68/month |

#### Alternative: B-Series (Burstable - Dev/Test)

| VM Size | vCPU | Memory | On-Demand |
|---------|------|--------|-----------|
| Standard_B2s | 2 | 4 GB | ~$30/month |
| Standard_B4ms | 4 | 16 GB | ~$120/month |

For the default deployment (8-12 vCPUs, 16-32GB RAM):
- **Dev/Test**: 3x Standard_D2s_v5 → $210/month
- **Small Prod**: 3x Standard_D4s_v5 → $423/month
- **With Spot nodes**: 3x Standard_D4s_v5 Spot → ~$102/month

### 3. Azure Managed Disks

| Disk Type | Size | Price |
|-----------|------|-------|
| Standard SSD (E10) | 128 GB | ~$10/month |
| Premium SSD (P10) | 128 GB | ~$17/month |
| Premium SSD (P20) | 512 GB | ~$68/month |

Estimate for all PVCs: 50-200GB → $10-40/month

**ReadWriteMany Storage**:
- **Azure Files (Standard)**: ~$0.06/GB/month
- **Azure Files (Premium)**: ~$0.16/GB/month
- **Azure NetApp Files**: Starting ~$0.12/GB/month

### 4. Load Balancer

| Type | Cost |
|------|------|
| Basic | Free (being deprecated) |
| Standard | ~$18/month + $0.005/rule/hour |

### 5. Data Transfer

- **Inbound**: Free
- **Outbound (first 100 GB)**: Free
- **Outbound (100GB - 10TB)**: $0.087/GB
- **VNet peering**: $0.01/GB

## Quick Estimate Tool

Use the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/):

1. Add **Azure Kubernetes Service (AKS)** → Select tier
2. Add **Virtual Machines** → Select VM size × count
3. Add **Managed Disks** → For each PVC
4. Add **Azure Files** → If using ReadWriteMany
5. Add **Load Balancer** → Standard LB

## Rough Monthly Estimates

| Tier | Config | Est. Total |
|------|--------|------------|
| Dev/Test | Free AKS + 3x D2s_v5 | ~$230-280/month |
| Small Prod | Standard AKS + 3x D4s_v5 | ~$520-600/month |
| Prod w/Spot | Standard AKS + 3x D4s_v5 Spot | ~$200-250/month |

## Comparison with Other Providers

| Component | AWS | DigitalOcean | Azure |
|-----------|-----|--------------|-------|
| K8s Control Plane | $73/month | Free | Free or $72 |
| 3x Medium Nodes | ~$280/month | ~$189/month | ~$423/month |
| 100GB Storage | ~$8/month | ~$10/month | ~$17/month |
| Load Balancer | ~$16/month | $12/month | ~$18/month |
| **Small Prod Total** | **~$377/month** | **~$211/month** | **~$530/month** |

**Note**: Azure becomes more competitive with:
- Spot VMs (up to 90% savings)
- Reserved Instances (up to 72% savings for 3-year)
- Azure Hybrid Benefit (if you have Windows Server licenses)

## Cost Optimization Options

### 1. Spot Virtual Machines
- Up to 90% discount on compute
- Good for worker pods, batch jobs
- Not recommended for stateful workloads
- Use multiple node pools (on-demand for critical, spot for workers)

### 2. Reserved Instances
| Term | Savings |
|------|---------|
| 1-year | ~40% |
| 3-year | ~60-72% |

### 3. Azure Hybrid Benefit
- Use existing Windows Server or SQL licenses
- Save up to 40% on Windows VMs

### 4. Use Azure Managed Services
- **Azure Database for PostgreSQL**: Starting ~$25/month (flexible server)
- **Azure Cache for Redis**: Starting ~$16/month
- **Azure Service Bus**: Alternative to RabbitMQ, ~$10/month
- **Azure Cognitive Search**: Alternative to OpenSearch (more expensive)

### 5. Dev/Test Pricing
- Up to 55% discount with Visual Studio subscription
- Use Azure Dev/Test subscription type

## Considerations

### Advantages of Azure AKS
- Free tier available (good for dev/test)
- Deep integration with Azure ecosystem (AD, Key Vault, Monitor)
- Virtual Nodes (serverless containers via ACI)
- Strong enterprise support and compliance certifications
- Azure Files provides native ReadWriteMany

### Considerations
- Higher base VM costs than AWS/DO
- Pricing complexity (many SKUs and options)
- Best value requires commitment (reserved instances)
- Spot availability varies by region/size

## Azure Regions Near Ohio

- **East US 2** (Virginia) - Closest major region
- **East US** (Virginia)
- **Central US** (Iowa)
- **North Central US** (Illinois) - Closest geographically

## Sources

- [Azure AKS Pricing](https://azure.microsoft.com/en-us/pricing/details/kubernetes-service/)
- [AKS Pricing Tiers Documentation](https://learn.microsoft.com/en-us/azure/aks/free-standard-pricing-tiers)
- [Azure VM Comparison - Vantage](https://instances.vantage.sh/azure)
- [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/)
- [AKS Cost Optimization Guide](https://sedai.io/blog/optimizing-azure-kubernetes-service-costs)
