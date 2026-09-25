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
| [Correct visitor IP addresses](#correct-visitor-ip-addresses) | Done, 25 Sep 2026 | None directly; fixes rate limiting and stats, and is needed before CloudFront |
| [CloudFront in front of the ALB](#cloudfront) | Done, 25 Sep 2026 (pay-as-you-go) | ~$85/month (pay-as-you-go) to ~$135/month (Pro plan) |
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

## Correct visitor IP addresses

Not a saving in itself, but a prerequisite for CloudFront and a bug in its
own right. Fixed 25 September 2026 (Helm revision 47).

**Problem.** Invenio saw the ALB's private IPs (`192.168.20.25`,
`192.168.36.255`, `192.168.64.9`) as every visitor's address, because it only
honours `X-Forwarded-For` when `PROXYFIX_CONFIG` (or the deprecated
`WSGI_PROXIES`) is set, and neither was. Confirmed from the rate limiter's
keys in Redis, which contained the user agent plus an ALB IP.

**Effect, before the fix.**

- *Rate limiting*: anonymous visitors were limited per user agent and ALB
  node, so everyone using the same browser version shared one
  500-requests-a-minute limit per ALB node.
- *Statistics*: invenio-stats identifies visitors by IP address plus user
  agent, so different people using the same browser were merged and unique
  visitor counts were understated. **Expect a step up in unique-visitor
  figures from 25 September 2026**; download and view counts shouldn't change
  much.

**Fix.** In `values-uchicago.yaml`, under `invenio.extraConfig` (rendered into
the `invenio-config` ConfigMap, which web and worker pods load as
environment variables):

```yaml
INVENIO_PROXYFIX_CONFIG: '{"x_for": 1, "x_proto": 0}'
```

Invenio parses `INVENIO_*` variables with `ast.literal_eval`, so this becomes
a dict and `invenio_base.wsgi.wsgi_proxyfix` wraps the app in
`werkzeug.middleware.proxy_fix.ProxyFix(x_for=1, x_proto=0)`. No change to
the application image or `settings.py` is needed. Deployment topology belongs
in the Helm values.

**Why `x_for: 1`.** `x_for` is the number of trusted proxies that append to the
`X-Forwarded-For` header the app receives, and the app uses the entry that
many places from the right.

- The ALB is in `append` mode (`routing.http.xff_header_processing.mode`), so
  it adds the connecting client's IP to whatever the client sent.
- nginx's `uwsgi_param X-Forwarded-For $proxy_add_x_forwarded_for` sets a
  separate uwsgi variable, not the `HTTP_X_FORWARDED_FOR` header the app
  reads, so it adds nothing.
- Measured in the uwsgi logs (which log `X-Forwarded-For`), 6 hours before
  the change: 25,218 requests with one entry (the client) and 272 with two,
  such as `127.0.0.1, 45.148.10.18` — clients spoofing a first entry, with
  their real IP appended by the ALB. `x_for: 1` uses the ALB's entry, so
  spoofed values are ignored; `x_for: 2` would let those clients choose their
  address.

**Why `x_proto: 0`.** `ProxyFix` trusts `X-Forwarded-Proto` by default, which
would change the URL scheme the app sees from `http` to `https`. That's a
separate change with its own risks, so it's left off.

**Use `PROXYFIX_CONFIG`, not `WSGI_PROXIES`.** `WSGI_PROXIES` is deprecated
and also trusts `X-Forwarded-Proto`. (`invenio_base`'s `WERKZEUG_GTE_014`
flag reads backwards — it's `False` on modern Werkzeug — so both settings do
work.)

**Tested before deploying** by building the app in a web pod with the new
variable and passing production-shaped headers through the wrapped app:
`X-Forwarded-For: 128.135.204.120` → `remote_addr` `128.135.204.120`;
`127.0.0.1, 45.148.10.18` → `45.148.10.18`; scheme unchanged.

**Verified after deploying:** new rate-limiter keys in Redis contain public
IPs.

**With CloudFront**, the chain becomes CloudFront → ALB → nginx: CloudFront
appends the viewer's IP, then the ALB appends CloudFront's, so `x_for` must
become 2 when the DNS switch happens. Requests still reaching the ALB
directly during the switch have only one entry; `ProxyFix` leaves the address
unchanged when there are fewer entries than `x_for`, so those fall back to
the ALB IP rather than being spoofable.

---

## CloudFront

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

Set up 25 September 2026, on pay-as-you-go. See
[aws-cloudfront.md](aws-cloudfront.md) for the configuration, the tests run
before the switch, and rollback. In short:

- **No caching** (`CachingDisabled`), so there's no risk of serving a
  restricted file from a cache.
- **Both hostnames on one distribution**, with a us-east-1 certificate. It
  validated using the records UChicago already publish for the ALB's
  certificate, so no DNS changes were needed at UChicago.
  `knowledge.uchicago.edu` is a CNAME to `uchicago.cottagelabs.com`, so
  switching that Route 53 record moved both.
- **`PROXYFIX_CONFIG` `x_for` raised to 2** (see
  [above](#correct-visitor-ip-addresses)).

### Why not Cloudflare

`cottagelabs.com` is on Cloudflare, but that doesn't cover this site.
`uchicago.cottagelabs.com` is **delegated** from the Cloudflare zone to Route
53 with NS records, and Cloudflare only proxies A/AAAA/CNAME records in its
own zone. The hostname resolves straight to the ALB (no `cf-ray` header), so
none of this traffic goes through Cloudflare today.

Moving it onto Cloudflare's proxy instead of adding CloudFront wouldn't save
the money:

- **The saving depends on free origin transfer, which only CloudFront gets.**
  AWS waives data transfer from the ALB to CloudFront. Cloudflare is just
  another internet client to AWS, so every uncached byte would still be
  billed at $0.09/GB. Cloudflare would only save money by *caching* file
  downloads, with the restricted-file risk above.
- **`knowledge.uchicago.edu` wouldn't work** without Cloudflare for SaaS
  (custom hostnames, a paid add-on) or UChicago moving its own DNS to
  Cloudflare. It's a CNAME into another Cloudflare account's zone, which
  Cloudflare rejects.
- **Upload size**: Cloudflare's Free and Pro plans cap request bodies at
  100 MB.
- **Terms**: Cloudflare's self-serve plans restrict using the CDN mainly to
  serve large volumes of non-HTML files, which is what this site's traffic
  is.

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
| 25 Sep 2026 | `INVENIO_PROXYFIX_CONFIG` set (Helm revision 47). During the rollout, three new web pods were restarted by their startup probe before the app finished loading — fixed in revision 48 (next row). |
| 25 Sep 2026 | Web `startupProbe.failureThreshold` 5 → 20 (Helm revision 48); see [maintenance/upgrades.md](../maintenance/upgrades.md#web-pod-startup-time). |

Follow-ups:

- Late October 2026: check how much data has moved to IA, the EFS line in
  Cost Explorer, and that burst credits stayed full; replace the estimates
  above with actual figures.
