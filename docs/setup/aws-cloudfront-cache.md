# CloudFront caching for uchicago.cottagelabs.com

> **Status: not yet implemented** (as of September 2026), and **needs
> rewriting before use**: see [aws-cost-reductions.md](aws-cost-reductions.md#cloudfront-planned)
> for the revised approach (no caching; add `knowledge.uchicago.edu`; fix
> visitor IPs) and the cost analysis.

This document sets up a CloudFront distribution in front of the ALB to cache
file downloads at the edge. Context: Elastic Load Balancing cost turned out
to be mostly Data Transfer Out — 1.37 TB in June, 2.41 TB in
July, almost certainly record/dataset file downloads — not the load balancer
itself. Repeat downloads of the same file currently re-transit the ALB every
time; CloudFront caching them at the edge avoids that repeat egress.

This does not touch DNS delegation or the Cloudflare zone — Route 53 stays
authoritative, CloudFront just becomes the new origin target for the existing
alias record.

> **Read before enabling in production:** CloudFront's cache key for the file
> path pattern below does not include cookies or auth headers. If a
> restricted-access record's file is served through this path and gets
> cached, it could be served to a *different, unauthorized* user from the
> edge cache without InvenioRDM's access check running again. Before pointing
> production traffic at the distribution, test explicitly: request a
> restricted-access record's file, inspect the response `Cache-Control`
> header, and confirm (via the `X-Cache` response header CloudFront adds
> automatically) that it comes back `Miss from cloudfront` on every request
> rather than being served from cache. Only proceed once that's confirmed —
> if InvenioRDM doesn't send a restrictive `Cache-Control` on those
> responses, don't enable caching for this path without fixing that first.

---

## 1. Request the CloudFront ACM certificate (us-east-1)

CloudFront requires its certificate in `us-east-1` regardless of which region
the distribution serves — this is a **separate** certificate from the
existing one used by the ALB's listener in `us-east-2` (see the TLS section
in [aws-setup.md](aws-setup.md)); that one is untouched.

```bash
CF_CERT_ARN=$(AWS_PROFILE=<your-profile> aws acm request-certificate \
  --domain-name uchicago.cottagelabs.com \
  --validation-method DNS \
  --region us-east-1 \
  --tags Key=project,Value=chicago-invenio Key=Name,Value=uchicago-cottagelabs-com-cloudfront \
  --query 'CertificateArn' --output text)
echo "CloudFront certificate ARN: $CF_CERT_ARN"
```

### Add the DNS validation CNAME to Route 53

```bash
VALIDATION=$(AWS_PROFILE=<your-profile> aws acm describe-certificate \
  --certificate-arn $CF_CERT_ARN --region us-east-1 \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord')

VALIDATION_NAME=$(echo $VALIDATION | python3 -c "import sys,json; print(json.load(sys.stdin)['Name'])")
VALIDATION_VALUE=$(echo $VALIDATION | python3 -c "import sys,json; print(json.load(sys.stdin)['Value'])")

cat > /tmp/cf-cert-validation.json <<EOF
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "${VALIDATION_NAME}",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{ "Value": "${VALIDATION_VALUE}" }]
      }
    }
  ]
}
EOF

AWS_PROFILE=<your-profile> aws route53 change-resource-record-sets \
  --hosted-zone-id Z02806991N89FM62782JH \
  --change-batch file:///tmp/cf-cert-validation.json

rm /tmp/cf-cert-validation.json
```

### Wait for validation

```bash
AWS_PROFILE=<your-profile> aws acm wait certificate-validated \
  --certificate-arn $CF_CERT_ARN --region us-east-1
```

---

## 2. Create the CloudFront distribution

Two cache behaviors matter here, evaluated in the order listed (CloudFront
matches the first path pattern that fits, so the more specific one has to
come first):

1. **Draft file uploads** (`/api/records/*/draft/files/*/content*`) — these
   use `PUT` for authenticated uploads, not just `GET` downloads. Must stay
   fully dynamic (`CachingDisabled`) and allow all methods. Listed first so
   it takes priority over the broader pattern below (`*` in a CloudFront
   path pattern matches slashes too, so the download pattern would otherwise
   also match draft upload URLs).
2. **Published file downloads** (`/api/records/*/files/*/content*`) — the
   actual cost driver. `GET`/`HEAD` only, cached via the managed
   `CachingOptimized` policy (respects the origin's `Cache-Control`/`Expires`
   headers, bounded to a 1-day default TTL if the origin doesn't set one).
3. **Everything else** (default behavior) — `CachingDisabled`, all methods,
   full cookie/header/query-string pass-through (`AllViewer`). This keeps
   the rest of the app — search, API, sessions, admin — behaving exactly as
   it does today, unchanged.

```bash
ALB_DOMAIN=k8s-invenio-invenio-045be042be-95786982.us-east-2.elb.amazonaws.com
CALLER_REF="chicago-invenio-$(date +%s)"

cat > /tmp/cf-distribution.json <<EOF
{
  "CallerReference": "${CALLER_REF}",
  "Comment": "chicago-invenio file-download caching",
  "Enabled": true,
  "HttpVersion": "http2and3",
  "IsIPV6Enabled": true,
  "Aliases": {
    "Quantity": 1,
    "Items": ["uchicago.cottagelabs.com"]
  },
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "chicago-invenio-alb",
        "DomainName": "${ALB_DOMAIN}",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "https-only",
          "OriginSslProtocols": { "Quantity": 1, "Items": ["TLSv1.2"] },
          "OriginReadTimeout": 30,
          "OriginKeepaliveTimeout": 5
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "chicago-invenio-alb",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
    "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3",
    "Compress": true,
    "AllowedMethods": {
      "Quantity": 7,
      "Items": ["GET", "HEAD", "OPTIONS", "PUT", "PATCH", "POST", "DELETE"],
      "CachedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] }
    }
  },
  "CacheBehaviors": {
    "Quantity": 2,
    "Items": [
      {
        "PathPattern": "/api/records/*/draft/files/*/content*",
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
      },
      {
        "PathPattern": "/api/records/*/files/*/content*",
        "TargetOriginId": "chicago-invenio-alb",
        "ViewerProtocolPolicy": "redirect-to-https",
        "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
        "OriginRequestPolicyId": "216adef6-5c7f-47e4-b989-5492eafa07d3",
        "Compress": false,
        "AllowedMethods": {
          "Quantity": 2,
          "Items": ["GET", "HEAD"],
          "CachedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] }
        }
      }
    ]
  },
  "PriceClass": "PriceClass_All",
  "ViewerCertificate": {
    "ACMCertificateArn": "${CF_CERT_ARN}",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  }
}
EOF

DISTRIBUTION_ID=$(AWS_PROFILE=<your-profile> aws cloudfront create-distribution \
  --distribution-config file:///tmp/cf-distribution.json \
  --query 'Distribution.Id' --output text)
echo "Distribution ID: $DISTRIBUTION_ID"

rm /tmp/cf-distribution.json
```

The managed policy IDs used above are fixed AWS constants (same across every
account):
- `4135ea2d-6df8-44a3-9df3-4b5a84be39ad` — `CachingDisabled`
- `658327ea-f89d-4fab-a63d-7e88639e58f6` — `CachingOptimized`
- `216adef6-5c7f-47e4-b989-5492eafa07d3` — `AllViewer` (origin request policy)

`PriceClass_All` uses every edge location. Switch to `PriceClass_100`
(US/Canada/Europe only) if UChicago's user base is mostly there and you want
to trim CloudFront's own cost — check actual request geography first via
CloudFront reports once it's been live a while.

### Wait for the distribution to deploy

```bash
AWS_PROFILE=<your-profile> aws cloudfront wait distribution-deployed \
  --id $DISTRIBUTION_ID
```

This can take 5–15 minutes.

---

## 3. Test before switching DNS

Query the distribution directly (via its own `*.cloudfront.net` domain, or
with a `Host` header override) before repointing production traffic:

```bash
CF_DOMAIN=$(AWS_PROFILE=<your-profile> aws cloudfront get-distribution \
  --id $DISTRIBUTION_ID --query 'Distribution.DomainName' --output text)

# General app traffic — should behave identically to hitting the ALB directly
curl -sI -H "Host: uchicago.cottagelabs.com" "https://${CF_DOMAIN}/"

# A public file download — check for `x-cache: Miss from cloudfront` on the
# first request and `Hit from cloudfront` on the second
curl -sI -H "Host: uchicago.cottagelabs.com" \
  "https://${CF_DOMAIN}/api/records/<id>/files/<key>/content"

# The restricted-access check described in the warning above — repeat with
# a restricted record's file and confirm it stays `Miss from cloudfront`
# every time.
```

---

## 4. Point Route 53 at the distribution

Once satisfied with testing, swap the existing ALIAS records from the ALB to
the CloudFront distribution. CloudFront's alias hosted zone ID
(`Z2FDTNDATAQYW2`) is a fixed AWS constant, the same for every distribution.

```bash
cat > /tmp/route53-cloudfront.json <<EOF
{
  "Changes": [
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "uchicago.cottagelabs.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "${CF_DOMAIN}",
          "EvaluateTargetHealth": false
        }
      }
    },
    {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "uchicago.cottagelabs.com",
        "Type": "AAAA",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "${CF_DOMAIN}",
          "EvaluateTargetHealth": false
        }
      }
    }
  ]
}
EOF

AWS_PROFILE=<your-profile> aws route53 change-resource-record-sets \
  --hosted-zone-id Z02806991N89FM62782JH \
  --change-batch file:///tmp/route53-cloudfront.json

rm /tmp/route53-cloudfront.json
```

Verify:

```bash
dig +short uchicago.cottagelabs.com
```

Should now return `*.cloudfront.net`-associated IPs rather than the ALB's.

---

## 5. Roll back if needed

If something's wrong, point the ALIAS records back at the ALB
(`k8s-invenio-invenio-045be042be-95786982.us-east-2.elb.amazonaws.com`,
hosted zone `Z3AADJGX6KTTL2`) using the same `change-resource-record-sets`
pattern as step 4 — this is exactly what the records looked like before this
change.
