# AWS cost reductions

Changes made (or planned) to reduce the AWS bill for the UChicago InvenioRDM
deployment, with the measurements behind each decision so they can be
revisited if usage changes.

When reporting costs for this project, filter Cost Explorer by the
`project=chicago-invenio` tag: the AWS account is shared with another project.

## Where the money goes

August 2026, from Cost Explorer (tagged `project=chicago-invenio`):

| Item | Monthly cost | Notes |
|---|---|---|
| EC2 nodes (3× t3.xlarge) | ~$370 | On-demand; no Savings Plan or reserved instances |
| EFS storage | ~$216 | 776 GB, all in Standard at $0.30/GB-month — addressed below |
| Data transfer out (via ALB) | ~$166 | 1.4–2.4 TB/month, June–September; see [CloudFront](#cloudfront-planned) |
| NAT gateway, ALB hours and LCUs | ~$67 | |
| RDS, ElastiCache, EBS, ECR, other | ~$80 | |

## Summary

| Change | Status | Estimated saving |
|---|---|---|
| [EFS Infrequent Access tiering](#efs-infrequent-access-tiering) | Done, 25 Sep 2026 | ~$150–170/month |
| [Correct visitor IP addresses](#correct-visitor-ip-addresses-planned) | Planned | None directly; fixes rate limiting and stats, and is needed before CloudFront |
| [CloudFront in front of the ALB](#cloudfront-planned) | Planned | ~$85/month (pay-as-you-go) to ~$135/month (Pro plan) |
| [Compute Savings Plan](#compute-savings-plan-not-pursued) | Not pursued | ~25–30% of EC2 on-demand spend |

---

## EFS Infrequent Access tiering

### What was done

A lifecycle policy moves files not read for 30 days from EFS Standard
($0.30/GB-month) to EFS Infrequent Access. Files stay in IA when read; they
don't move back to Standard. The command is in
[aws-setup.md, EFS step 5](aws-setup.md#5-move-rarely-read-files-to-infrequent-access).

### Why it's safe: throughput

The filesystem uses **Bursting** throughput. Its baseline throughput is
50 KiB/s per GiB of data **in the Standard storage class only** (reads are
metered at a third of the rate), with a minimum of 1 MiB/s, and burst credits
allow up to 100 MiB/s. Moving most data to IA therefore lowers the baseline:
from about 38 MiB/s with 776 GB in Standard, to about 4 MiB/s if ~75 GB stays
there.

Measured over the 30 days to 25 September 2026 (CloudWatch, `AWS/EFS`):

| Metric | Value |
|---|---|
| Data read | 2.7 TB (~88 GB/day) |
| Data written | 1.9 GB |
| Metered throughput | median 0.25 MiB/s, busiest hour 3.05 MiB/s |
| Burst credit balance | never below the 2.1 TiB maximum |

The site's usage is well under even the reduced baseline, so credits should
stay full. The `chicago-invenio-efs-burst-credits-low` alarm (see
[aws-monitoring-and-alerting.md](aws-monitoring-and-alerting.md#4-efs-burst-credits-alarm))
fires if they drop below 1 TiB.

### Decisions

- **Stay on Bursting throughput rather than Elastic.** Elastic has cheaper IA
  storage ($0.016 vs $0.025/GB-month) but charges $0.03/GB for every read —
  about $80/month at 2.7 TB of reads.
- **Don't move files back to Standard on access**
  (`TransitionToPrimaryStorageClass` is left unset). With it set, a single
  download of a 5 GB file would move it back to Standard for at least 30 days
  (~$1.50) instead of costing a $0.05 IA read.
- **30 days**, the default. Files smaller than 128 KiB, and metadata, always
  stay in Standard.
- **Not the Archive storage class.** It requires Elastic throughput.

### Expected cost

us-east-2 prices from the AWS Pricing API, September 2026, Bursting mode:

| | Price | Before | After (if ~90% of data isn't read for 30 days) |
|---|---|---|---|
| Standard storage | $0.30/GB-month | 776 GB → $216 | ~75 GB → ~$23 |
| IA storage | $0.025/GB-month | — | ~700 GB → ~$18 |
| IA reads | $0.01/GB | — | up to ~$27 (if all reads came from IA) |
| Tiering into IA | $0.01/GB | — | ~$7, once |
| **Total** | | **~$216/month** | **~$45–70/month** |

### Checking progress

Files start moving once they've gone 30 days without being read. To see how
much has moved:

```bash
AWS_PROFILE=<your-profile> aws efs describe-file-systems --region us-east-2 \
  --file-system-id fs-0e8f6bacbcc4515bf \
  --query 'FileSystems[0].SizeInBytes.{Standard:ValueInStandard,IA:ValueInIA}'
```

### Reverting

To move files back to Standard as they're read:

```bash
AWS_PROFILE=<your-profile> aws efs put-lifecycle-configuration --region us-east-2 \
  --file-system-id fs-0e8f6bacbcc4515bf \
  --lifecycle-policies '[{"TransitionToIA":"AFTER_30_DAYS"},{"TransitionToPrimaryStorageClass":"AFTER_1_ACCESS"}]'
```

Or pass `--lifecycle-policies '[]'` to stop tiering (files already in IA stay
there until read with `AFTER_1_ACCESS` set).

---

## Correct visitor IP addresses (planned)

Not a saving in itself, but a prerequisite for CloudFront and a bug in its
own right.

**Problem.** Invenio doesn't see visitors' real IP addresses: it sees the
ALB's private IPs (`192.168.20.25`, `192.168.36.255`, `192.168.64.9` in
September 2026). Invenio only honours `X-Forwarded-For` when `WSGI_PROXIES`
(or `PROXYFIX_CONFIG`) is set, and neither is. Confirmed from the rate
limiter's keys in Redis, which contain the user agent plus an ALB IP.

**Effect.**

- *Rate limiting*: anonymous visitors are limited per user agent and ALB node,
  so everyone using the same browser version shares one 500-requests-a-minute
  limit per ALB node. A busy period could produce 429 errors for real users.
- *Statistics*: invenio-stats identifies visitors by IP address plus user
  agent, so different people using the same browser are merged, and unique
  visitor counts are likely understated.

**Fix (to be worked out).** Set `INVENIO_WSGI_PROXIES` to the number of
proxies in front of the app. Today the chain is ALB → nginx, so
`X-Forwarded-For` reaching the app is `<client>, <ALB IP>` and the value would
be 2. CloudFront adds a hop, making it 3. Because traffic moves between the
two paths gradually during a DNS change, an alternative is nginx's `real_ip`
module with trusted ranges (the VPC and CloudFront's published ranges), which
handles both. Needs testing either way: a wrong value lets clients spoof
their IP.

---

## CloudFront (planned)

The saving comes from pricing, not caching. Data transfer from the ALB to
CloudFront is free, and CloudFront's own data transfer out is cheaper,
with a large free allowance.

Measured, June–September 2026: 1.4–2.4 TB/month out via the ALB (average
~$150/month at $0.09/GB), and 6–11M requests/month.

| Option (prices as of September 2026) | Estimated monthly cost | Saving |
|---|---|---|
| Today (ALB direct) | ~$150 | — |
| Pay-as-you-go: first 1 TB and 10M requests free, then $0.085/GB (US/EU) | ~$65 | ~$85/month |
| Pro flat-rate plan: $15/month, 50 TB and 10M requests | $15 | ~$135/month |

The Pro plan's allowances aren't hard limits (AWS only adjusts delivery for
sustained, substantial excess, after notifications), but 6–11M requests is
close to its 10M; requests blocked by the plan's WAF don't count. It requires
a WAF web ACL on the distribution. Start on pay-as-you-go and move to Pro once
real CloudFront request numbers are known.

[aws-cloudfront-cache.md](aws-cloudfront-cache.md) needs rewriting before
use:

- **Drop caching** (`CachingDisabled` on every behaviour). It isn't needed for
  the saving, and caching file downloads risks serving restricted files to
  the wrong users.
- **Add `knowledge.uchicago.edu`** as an alternate domain name, with a
  us-east-1 certificate covering it. `knowledge.uchicago.edu` is a CNAME to
  `uchicago.cottagelabs.com`, so switching the Route 53 record moves both,
  but validating the certificate needs a DNS record from UChicago IT.
- **Fix visitor IPs** at the same time (see above).
- **Test** logins, large uploads, downloads, and that stats are still
  recorded, before switching DNS. Rollback is re-pointing the Route 53 record
  at the ALB.

---

## Compute Savings Plan (not pursued)

The three t3.xlarge nodes (~$370/month) run on-demand, with no Savings Plan
or reserved instances. A 1-year Compute Savings Plan typically saves 25–30%.
It's a financial commitment rather than an engineering change, and a Savings
Plan applies across the whole AWS account, including the other project that
shares it.

---

## Implementation record

| Date | Change |
|---|---|
| 25 Sep 2026 | EFS lifecycle policy `TransitionToIA: AFTER_30_DAYS` on `fs-0e8f6bacbcc4515bf`. CloudWatch alarm `chicago-invenio-efs-burst-credits-low` created. |

Follow-ups:

- Late October 2026: check how much data has moved to IA, the EFS line in
  Cost Explorer, and that burst credits stayed full; replace the estimates
  above with actual figures.
