# Documentation

Documentation for the UChicago InvenioRDM deployment (`knowledge.uchicago.edu`)
on AWS EKS.

## [investigation/](investigation/)

Early research into hosting options and costs, done before choosing AWS. Kept
for reference; not maintained against the live deployment.

- [uchicago-deployment-comparison.md](investigation/uchicago-deployment-comparison.md) — cost and architecture comparison of hosting options
- [ha-justification.md](investigation/ha-justification.md) — multi-AZ vs single-AZ
- [aws-cost-estimate.md](investigation/aws-cost-estimate.md), [azure-cost-estimate.md](investigation/azure-cost-estimate.md), [digitalocean-cost-estimate.md](investigation/digitalocean-cost-estimate.md)

## [setup/](setup/)

How the infrastructure and deployment were built — follow these to reproduce
it, or to see why it is the way it is.

- [aws-setup.md](setup/aws-setup.md) — AWS infrastructure and Kubernetes deployment
- [initial_deployment.md](setup/initial_deployment.md) — secrets, data import and first-run steps
- [production-cutover-knowledge-uchicago-edu.md](setup/production-cutover-knowledge-uchicago-edu.md) — moving to the production hostname
- [aws-monitoring-and-alerting.md](setup/aws-monitoring-and-alerting.md) — CloudWatch Container Insights and alarms
- [aws-backups.md](setup/aws-backups.md) — backups of the database, uploaded files and statistics
- [aws-cost-reductions.md](setup/aws-cost-reductions.md) — cost savings made and planned, with the measurements behind them
- [aws-cloudfront-cache.md](setup/aws-cloudfront-cache.md) — CloudFront in front of the ALB for file downloads (**not yet implemented**)

## [maintenance/](maintenance/)

Running the live service.

- [upgrades.md](maintenance/upgrades.md) — deploying releases, reinstalling, EKS version upgrades
- [backups-and-restores.md](maintenance/backups-and-restores.md) — checking backups ran, and restoring
- [opensearch.md](maintenance/opensearch.md) — rolling restarts and disk usage
- [delete-file-from-record.md](maintenance/delete-file-from-record.md) — removing a single file from a published record
