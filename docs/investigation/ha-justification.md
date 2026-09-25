# High Availability Justification

**Date**: February 2026
**Purpose**: Compare HA (multi-AZ) vs single-AZ deployment options for UChicago InvenioRDM
**Related**: [uchicago-deployment-comparison.md](uchicago-deployment-comparison.md)

## Overview

This document compares two deployment strategies:

1. **Multi-AZ (High Availability)**: Resources spread across multiple Availability Zones with automatic failover
2. **Single-AZ with Backups**: All resources in one AZ, recovery via backup restoration

## Cost Comparison

| Component | Multi-AZ (HA) | Single-AZ | Savings |
|-----------|---------------|-----------|---------|
| EKS Nodes | 3x t3.large ($180) | 3x t3.large ($180) | $0 |
| OpenSearch | 2-node multi-AZ ($54) | 1-node ($27) | $27 |
| RDS PostgreSQL | Multi-AZ ($60) | Single-AZ ($30) | $30 |
| ElastiCache Redis | With replica ($24) | No replica ($12) | $12 |
| S3 | $15 | $15 | $0 |
| ALB | $20 | $20 | $0 |
| Data Transfer | $25 | $25 | $0 |
| Cross-region backups | - | ~$10 | -$10 |
| **Total** | **$451/month** | **~$319/month** | **~$132/month** |

**Annual cost difference:** ~$1,584

## Availability Comparison

### Multi-AZ (HA)

| Failure Scenario | Impact | Recovery |
|------------------|--------|----------|
| Single node failure | None | Automatic pod reschedule (~1-2 min) |
| Single AZ outage | None | Automatic failover (seconds to minutes) |
| RDS primary failure | None | Automatic failover (~60-120 sec) |
| OpenSearch node failure | None | Replica serves requests |
| Redis primary failure | None | Replica promoted |

**Expected uptime:** 99.9%+ (excluding application-level issues)

### Single-AZ with Backups

| Failure Scenario | Impact | Recovery |
|------------------|--------|----------|
| Single node failure | Degraded | Automatic pod reschedule (~1-2 min) |
| AZ outage | **Full outage** | Manual restore (~1-2 hours) |
| RDS failure | **Database outage** | Restore from snapshot (~15-30 min) |
| OpenSearch failure | **Search outage** | Restore from snapshot (~15-30 min) |
| Redis failure | **Cache outage** | Pod reschedule, cold cache (~2-5 min) |

**Expected uptime:** 99.5%+ (dependent on AZ stability and restore automation)

## Recovery Procedure for Single-AZ

If the primary AZ fails, the following steps are required:

| Step | Description | Estimated Time |
|------|-------------|----------------|
| 1 | Detect failure (monitoring alert) | 5-15 min |
| 2 | Provision infrastructure in new AZ (Terraform/IaC) | 15-30 min |
| 3 | Restore RDS from latest snapshot | 15-30 min |
| 4 | Restore OpenSearch from S3 snapshot | 15-30 min |
| 5 | Deploy application (Helm) | 5-10 min |
| 6 | Verify services and update DNS | 5-10 min |
| **Total** | | **~1-2 hours** |

**Prerequisites for this recovery:**
- Infrastructure as Code (Terraform) ready for alternate AZ
- Automated RDS snapshot copy to another region/AZ
- OpenSearch snapshots stored in S3 (cross-AZ durable)
- Documented runbook or automated recovery scripts
- DNS TTL set low enough for quick switchover

## Backup Requirements for Single-AZ

| Data | Backup Method | Frequency | Retention | Cost |
|------|---------------|-----------|-----------|------|
| PostgreSQL | RDS automated snapshots | Daily | 7 days | Included |
| PostgreSQL | Cross-region snapshot copy | Daily | 7 days | ~$5/month |
| OpenSearch | Snapshot to S3 | Daily | 7 days | ~$2/month |
| S3 files | Cross-region replication | Continuous | - | ~$3/month |
| **Total backup cost** | | | | **~$10/month** |

## Risk Assessment

### AZ Failure Probability

AWS Availability Zones are highly reliable. Historical data suggests:
- Individual AZ outages: ~1-2 per year, lasting minutes to hours
- Full AZ failures are rare but do occur

### Impact Assessment for UChicago InvenioRDM

| Factor | Assessment |
|--------|------------|
| User base | ~2,000 students |
| Usage pattern | Academic, primarily business hours |
| Peak periods | Submission deadlines |
| Data criticality | High (research data, theses) |
| Acceptable downtime | ? |

## Decision Matrix

| Factor | Multi-AZ (HA) | Single-AZ |
|--------|---------------|-----------|
| Monthly cost | $451 | $319 |
| Annual cost | $5,412 | $3,828 |
| AZ failure recovery | Automatic | 1-2 hours manual |
| Operational complexity | Lower | Higher (need runbooks) |
| 3am incident response | Not required | May be required |
| Data loss risk | Near-zero | Up to 24 hours |

## Recommendation

### Choose Multi-AZ ($451/month) if:
- Out-of-timezone support (no one to run restore at 3am)
- Submission deadlines cannot tolerate multi-hour outages
- Limited operational capacity to maintain restore runbooks
- Risk tolerance is low

### Choose Single-AZ ($319/month) if:
- ~$1,600/year savings is significant
- Multi-hour outages during rare AZ failures are acceptable
- Team can maintain and test restore procedures
- Willing to accept potential 3am incident response

## Hybrid Options

### Option A: Single-AZ with Warm Standby (~$380/month)
- Run primary in single AZ
- Maintain infrastructure-as-code for rapid re-deploy
- Keep RDS read replica in another AZ (adds ~$30)
- Recovery time: ~30 minutes

### Option B: Multi-AZ with Smaller Instances (~$400/month)
- Keep multi-AZ architecture
- Reduce OpenSearch to t3.micro nodes
- Accept slower search performance

## Conclusion

The $132/month (~$1,600/year) cost difference between Multi-AZ and Single-AZ is the price of:
- Automatic failover vs manual recovery
- Seconds of downtime vs hours of downtime
- No 3am pages vs potential 3am incident response

For a student-facing university system with out-of-timezone support requirements, **Multi-AZ is recommended** despite the higher cost.

If budget constraints require Single-AZ, ensure:
1. Restore procedures are documented and tested quarterly
2. Monitoring alerts are configured for AZ health
3. Infrastructure-as-code is ready for alternate AZ deployment
4. Stakeholders accept the 1-2 hour recovery time for AZ failures
