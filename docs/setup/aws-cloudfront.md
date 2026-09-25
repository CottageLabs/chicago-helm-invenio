# CloudFront in front of the ALB

A CloudFront distribution sits in front of the ALB for both
`knowledge.uchicago.edu` and `uchicago.cottagelabs.com`, **without caching**.
Its only purpose is cost: AWS doesn't charge for data transfer from an ALB to
CloudFront, and CloudFront's own data transfer out is cheaper than the ALB's,
with 1 TB a month free. **Live since 25 September 2026.** See
[aws-cost-reductions.md](aws-cost-reductions.md#cloudfront) for the
numbers and for why Cloudflare isn't an alternative.

```
visitor → CloudFront (no caching) → ALB → nginx → uwsgi (Invenio)
```

**Why no caching.** Caching isn't needed for the saving, and caching file
downloads risks serving a restricted record's files to the wrong user: the
cache key wouldn't include the session cookie, so a cached response could be
returned without Invenio's access check running again. With the managed
`CachingDisabled` policy, every request goes to the ALB exactly as it does
without CloudFront.

**DNS.** `knowledge.uchicago.edu` is a CNAME, in UChicago's DNS, to
`uchicago.cottagelabs.com`, whose A/AAAA ALIAS records are in our Route 53
zone (`Z02806991N89FM62782JH`, delegated from Cloudflare). Pointing those
records at CloudFront moves both hostnames; UChicago's DNS doesn't change.

---

## 1. Certificate (us-east-1)

CloudFront only uses ACM certificates from `us-east-1`. This is separate from
the ALB's certificate in `us-east-2`, and covers the same two names.

```bash
CF_CERT_ARN=$(AWS_PROFILE=<your-profile> aws acm request-certificate --region us-east-1 \
  --domain-name knowledge.uchicago.edu \
  --subject-alternative-names uchicago.cottagelabs.com \
  --validation-method DNS \
  --tags Key=project,Value=chicago-invenio Key=Name,Value=chicago-invenio-cloudfront \
  --query CertificateArn --output text)

AWS_PROFILE=<your-profile> aws acm wait certificate-validated --region us-east-1 \
  --certificate-arn $CF_CERT_ARN
```

No new DNS records are needed. ACM uses the same validation CNAME for a given
domain in every region of an account, and both are already published for the
ALB's certificate (see
[production-cutover-knowledge-uchicago-edu.md](production-cutover-knowledge-uchicago-edu.md)):

| Domain | Validation record | Published in |
|---|---|---|
| `knowledge.uchicago.edu` | `_41f4866f832b62efbb8b2f711a440630.knowledge.uchicago.edu` | UChicago's DNS |
| `uchicago.cottagelabs.com` | `_03a3ebe4b4f5f68e839d3608dd1d2541.uchicago.cottagelabs.com` | Route 53 |

These must stay in place for both certificates to renew. If a validation
ever doesn't complete, check with `dig CNAME <record>`; for
`knowledge.uchicago.edu`, UChicago IT manage the record.

## 2. Create the distribution

```bash
ALB_DOMAIN=k8s-invenio-invenio-045be042be-95786982.us-east-2.elb.amazonaws.com

cat > /tmp/cf-distribution.json <<EOF
{
  "CallerReference": "chicago-invenio-$(date +%s)",
  "Comment": "chicago-invenio: no caching; in front of the ALB to avoid ALB data transfer charges",
  "Enabled": true,
  "HttpVersion": "http2and3",
  "IsIPV6Enabled": true,
  "PriceClass": "PriceClass_100",
  "Aliases": { "Quantity": 2, "Items": ["knowledge.uchicago.edu", "uchicago.cottagelabs.com"] },
  "ViewerCertificate": {
    "ACMCertificateArn": "${CF_CERT_ARN}",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "chicago-invenio-alb",
      "DomainName": "${ALB_DOMAIN}",
      "CustomOriginConfig": {
        "HTTPPort": 80,
        "HTTPSPort": 443,
        "OriginProtocolPolicy": "https-only",
        "OriginSslProtocols": { "Quantity": 1, "Items": ["TLSv1.2"] },
        "OriginReadTimeout": 60,
        "OriginKeepaliveTimeout": 55
      }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "chicago-invenio-alb",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
    "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3",
    "Compress": false,
    "AllowedMethods": {
      "Quantity": 7,
      "Items": ["GET", "HEAD", "OPTIONS", "PUT", "PATCH", "POST", "DELETE"],
      "CachedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] }
    }
  }
}
EOF

AWS_PROFILE=<your-profile> aws cloudfront create-distribution-with-tags \
  --distribution-config-with-tags "{\"DistributionConfig\": $(cat /tmp/cf-distribution.json), \"Tags\": {\"Items\": [{\"Key\": \"project\", \"Value\": \"chicago-invenio\"}]}}" \
  --query 'Distribution.[Id,DomainName]' --output text

AWS_PROFILE=<your-profile> aws cloudfront wait distribution-deployed --id <distribution-id>
```

A new distribution receives no traffic until DNS points at it.

Settings, and why:

| Setting | Value | Why |
|---|---|---|
| Cache policy | `CachingDisabled` (`4135ea2d-…`, AWS managed) | Nothing is cached; see above |
| Origin request policy | `AllViewer` (`216adef6-…`, AWS managed) | Forwards all headers (including `Host`), cookies and query strings, so the app sees requests as before |
| Allowed methods | all 7 | Uploads (`PUT`), forms and API (`POST`, `DELETE`, …) |
| Origin protocol | HTTPS only | With `Host` forwarded, CloudFront uses it for SNI and certificate checks; the ALB's certificate covers both names |
| Origin read timeout | 60 s | Matches the ALB's idle timeout (`idle_timeout.timeout_seconds` = 60). Maximum 120 s |
| Origin keep-alive | 55 s | Must be below the ALB's idle timeout |
| Compress | off | Only applies to cached responses |
| Price class | `PriceClass_100` | North American and European edge locations only. On pay-as-you-go, traffic is billed at their rates, so visitors elsewhere cost less, at the price of some extra latency for them |

Limits checked in September 2026: CloudFront allows request bodies up to
64 GB (nginx allows 50 GB for file uploads; the largest stored file was
6.5 GB) and has no size limit on uncached responses.

## 3. Test before switching DNS

Send requests to the distribution while presenting the real hostname, so SNI
and the `Host` header are what they will be after the switch:

```bash
CF=<distribution-domain>.cloudfront.net

curl -sI --connect-to knowledge.uchicago.edu:443:${CF}:443 https://knowledge.uchicago.edu/ \
  | grep -iE '^HTTP|^x-cache|^via|^server'
```

Responses should come from nginx (`server: nginx`) with
`x-cache: Miss from cloudfront` — every request is a miss, because nothing is
cached. Check at least: the home page, a search (`/search?q=…`), the API
(`/api/records?size=1`), a record page, a public file download, and a
restricted file (must still be refused without logging in).

To test uploads, use a personal access token (Settings → Applications) on a
restricted draft, and delete the draft afterwards. Larger files are uploaded
as multipart: initialise the file with
`{"key": …, "size": …, "transfer": {"type": "M", "parts": N, "part_size": …}}`,
`PUT` each part to the URLs in the response's `links.parts`, then `POST` to
the file's `commit` link. (A single `PUT` to `…/content` of a multipart file
is rejected.)

To test logging in and uploading in a browser, temporarily point the hostname
at CloudFront on your own machine: look up one of the distribution's IPs
(`dig +short ${CF}`) and add `<ip> knowledge.uchicago.edu` to your hosts file.
Remove it afterwards.

## 4. Switch DNS, then update PROXYFIX_CONFIG

Point the A and AAAA ALIAS records for `uchicago.cottagelabs.com` at the
distribution. `Z2FDTNDATAQYW2` is CloudFront's fixed hosted zone ID for
ALIAS records.

```bash
cat > /tmp/route53-cloudfront.json <<EOF
{
  "Comment": "Point uchicago.cottagelabs.com at CloudFront",
  "Changes": [
    { "Action": "UPSERT", "ResourceRecordSet": { "Name": "uchicago.cottagelabs.com", "Type": "A",
      "AliasTarget": { "HostedZoneId": "Z2FDTNDATAQYW2", "DNSName": "${CF}", "EvaluateTargetHealth": false } } },
    { "Action": "UPSERT", "ResourceRecordSet": { "Name": "uchicago.cottagelabs.com", "Type": "AAAA",
      "AliasTarget": { "HostedZoneId": "Z2FDTNDATAQYW2", "DNSName": "${CF}", "EvaluateTargetHealth": false } } }
  ]
}
EOF

AWS_PROFILE=<your-profile> aws route53 change-resource-record-sets \
  --hosted-zone-id Z02806991N89FM62782JH \
  --change-batch file:///tmp/route53-cloudfront.json
```

Then set `x_for` to 2 in `values-uchicago.yaml` and deploy:

```yaml
INVENIO_PROXYFIX_CONFIG: '{"x_for": 2, "x_proto": 0}'
```

Via CloudFront, `X-Forwarded-For` reaching the app is
`<visitor>, <CloudFront edge>`: CloudFront appends the visitor's IP and the
ALB appends CloudFront's. With `x_for: 1` the app would see CloudFront edge
IPs as visitors (the bug fixed in
[aws-cost-reductions.md](aws-cost-reductions.md#correct-visitor-ip-addresses)).
Switch DNS first: while the web pods roll out, requests via CloudFront are
seen as coming from CloudFront's IPs, which is harmless for a few minutes.
Doing it the other way round would briefly let clients reaching the ALB
directly spoof their IP.

## 5. Verify

```bash
dig +short uchicago.cottagelabs.com          # CloudFront IPs, not the ALB's
curl -sI https://knowledge.uchicago.edu/ | grep -iE '^x-cache|^via'
```

After the `x_for` change has rolled out, new rate-limiter keys in Redis should
contain visitors' public IPs, not CloudFront's (see the check in
[aws-cost-reductions.md](aws-cost-reductions.md#correct-visitor-ip-addresses)).

Over the following days, the ALB's `DataTransfer-Out-Bytes` in Cost Explorer
should fall to almost nothing, replaced by CloudFront data transfer.

## 6. Roll back

Point both records back at the ALB (hosted zone `Z3AADJGX6KTTL2`, the fixed
ID for ALBs in us-east-2), then set `x_for` back to 1:

```bash
cat > /tmp/route53-alb.json <<EOF
{
  "Comment": "Point uchicago.cottagelabs.com back at the ALB",
  "Changes": [
    { "Action": "UPSERT", "ResourceRecordSet": { "Name": "uchicago.cottagelabs.com", "Type": "A",
      "AliasTarget": { "HostedZoneId": "Z3AADJGX6KTTL2", "DNSName": "k8s-invenio-invenio-045be042be-95786982.us-east-2.elb.amazonaws.com", "EvaluateTargetHealth": true } } },
    { "Action": "UPSERT", "ResourceRecordSet": { "Name": "uchicago.cottagelabs.com", "Type": "AAAA",
      "AliasTarget": { "HostedZoneId": "Z3AADJGX6KTTL2", "DNSName": "k8s-invenio-invenio-045be042be-95786982.us-east-2.elb.amazonaws.com", "EvaluateTargetHealth": true } } }
  ]
}
EOF

AWS_PROFILE=<your-profile> aws route53 change-resource-record-sets \
  --hosted-zone-id Z02806991N89FM62782JH \
  --change-batch file:///tmp/route53-alb.json
```

These are the exact values from before the switch (see the implementation
record below).

---

## Later

- **Pro flat-rate plan** ($15/month). Decided 25 September 2026: run on
  pay-as-you-go for a week to confirm the setup and the actual CloudFront
  numbers, then move to Pro. See
  [aws-cost-reductions.md](aws-cost-reductions.md#moving-to-the-pro-plan)
  for the checks and the WAF precautions.
- **Only accept traffic from CloudFront at the ALB**, using the
  `com.amazonaws.global.cloudfront.origin-facing` managed prefix list in the
  ALB's security group. This stops the ALB being used directly (bypassing
  CloudFront, and with `x_for: 2`, letting clients choose their IP). The
  AWS Load Balancer Controller manages that security group, so this needs
  doing through the Ingress annotations.

---

## Implementation record

| Date | Step | Details |
|---|---|---|
| 25 Sep 2026 | 1 | Certificate `…62932abb-7ab9-4767-b8fb-be366ef6814b` (us-east-1) issued immediately; both validation records were already published. Two unused certificate requests from May 2026 in us-east-2 that had ended in `VALIDATION_TIMED_OUT` were deleted, along with two unused issued us-east-2 certificates (`0815df7a…` for `uchicago.cottagelabs.com` only, `b8441dbd…` for `knowledge.uchicago.edu` only). Remaining: the ALB's `b5f3cd42…` (us-east-2) and CloudFront's `62932abb…` (us-east-1), both covering both names |
| 25 Sep 2026 | 2 | Distribution `E240LA3XOMZ4VT` (`dfiqzt5vs9rwd.cloudfront.net`) created |
| 25 Sep 2026 | 3 | Tested through the distribution with `--connect-to`: home, search, record page, API, OAI-PMH, login page, public file download — all 200 from nginx, `Miss from cloudfront`; restricted (embargoed) record's files 403 as via the ALB; HTTP → HTTPS redirect; `uchicago.cottagelabs.com`. Range requests return the whole file (200), same as via the ALB. Rate-limiter keys confirmed CloudFront edge IPs (`130.176.x`) are recorded with `x_for: 1`. SSO login tested in a browser via a hosts-file entry. Uploads tested with a personal access token on a restricted draft (then deleted): 1 MB single upload, and a 150 MB multipart upload (3 × 50 MB parts, the way the deposit form uploads large files); checksums matched after downloading back through CloudFront |
| 25 Sep 2026 | 4 | 17:50 UTC: Route 53 A/AAAA for `uchicago.cottagelabs.com` switched to the distribution (change `C0429592F7EKZDPBWHYT`, in sync after 34 s; public resolvers followed within a minute). `INVENIO_PROXYFIX_CONFIG` `x_for` 1 → 2 in Helm revision 49 |

Route 53 records before the switch (for rollback):

| Record | Type | ALIAS target |
|---|---|---|
| `uchicago.cottagelabs.com` | A | `k8s-invenio-invenio-045be042be-95786982.us-east-2.elb.amazonaws.com` (ALB), hosted zone `Z3AADJGX6KTTL2`, `EvaluateTargetHealth: true` |
| `uchicago.cottagelabs.com` | AAAA | same |
