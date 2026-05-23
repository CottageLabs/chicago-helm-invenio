# Production Cutover Guide: `knowledge.uchicago.edu`

This guide describes the patch set and runbook to cut over the current deployment from `uchicago.cottagelabs.com` to `knowledge.uchicago.edu`.

## Checklist

- [ ] Update runtime Helm values for production hostname.
- [ ] Remove pre-production basic auth from chart values.
- [ ] Update private values with ACM certificate ARN for `knowledge.uchicago.edu`.
- [ ] Update docs and examples to the production domain.
- [ ] Confirm OAuth redirect URIs include the production domain.
- [ ] Run pre-cutover checks.
- [ ] Deploy via Helm upgrade.
- [ ] Run post-cutover validation.

---

## 1) Required patch list (files in git)

### A. `values-uchicago.yaml`

Apply the following changes:

1. Header comment
   - `## Target: uchicago.cottagelabs.com` -> `## Target: knowledge.uchicago.edu`

2. Invenio hostname
   - `invenio.hostname: uchicago.cottagelabs.com` -> `invenio.hostname: knowledge.uchicago.edu`

3. Remove pre-production basic auth configuration
   - Remove `web.extraVolumes` entry that mounts `invenio-basic-auth`.
   - Remove `nginx.extraVolumeMounts` entry mounting `/etc/nginx/auth`.
   - Remove `nginx.extraServerConfig` auth directives:
     - `auth_basic "Restricted";`
     - `auth_basic_user_file /etc/nginx/auth/htpasswd;`

### B. `values-uchicago-private.yaml.example`

- Update the top comment to reference `knowledge.uchicago.edu`.

### C. `docs/aws-setup.md`

Update domain references and examples:

- Replace `uchicago.cottagelabs.com` with `knowledge.uchicago.edu` where this doc is used for production cutover.
- Update ACM request example (`--domain-name`).
- Update TLS validation examples (`curl`, expected cert subject).
- Update ExternalDNS example domain filter to:
  - `--set "domainFilters[0]=knowledge.uchicago.edu"`
- Remove basic auth secret creation guidance (`invenio-basic-auth`) from deployment steps.

### D. `docs/upgrades.md`

- Update healthcheck URL from old hostname to `https://knowledge.uchicago.edu`.

### E. `site/chicago_invenio/config/settings.py` (app repo)

In `/home/richard/Dropbox/Code/chicago-invenio/site/chicago_invenio/config/settings.py`:

- Update fallback trusted host entry:
  - `'uchicago.invenio.cottagelabs.com'` -> `'knowledge.uchicago.edu'`

> Note: Helm currently injects host-related env vars from chart templates. This fallback update reduces risk in non-Helm/dev paths.

---

## 2) Required private config change (gitignored)

In `values-uchicago-private.yaml` (not committed):

- Set `ingress.annotations.alb.ingress.kubernetes.io/certificate-arn` to an ACM certificate ARN valid for `knowledge.uchicago.edu`.

Example shape:

```yaml
ingress:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:<account-id>:certificate/<cert-id>
```

---

## 3) SSL/TLS and key handling

- TLS is terminated at AWS ALB using ACM.
- You do **not** generate TLS private keys manually in this setup; ACM manages key generation and storage.
- A hostname change requires a matching ACM certificate and Ingress annotation update.

Not part of TLS cutover:

- `INVENIO_SECRET_KEY` and related salts in Kubernetes secret (chart `templates/secret.yaml`) are app/session secrets, not TLS cert keys.
- Do not rotate these during hostname cutover unless intentionally planning logout/token invalidation impact.

---

## 4) Identity/OAuth cutover requirements

Update your Okta (or other IdP) app registration to include production callback URLs, typically:

- `https://knowledge.uchicago.edu/oauth/authorized/chi`

Recommended transition practice:

- Keep old and new redirect URIs configured until cutover is stable.

---

## 5) Pre-cutover checks

Run from `/home/richard/Dropbox/Code/chicago-helm-invenio`:

```bash
helm dependency update charts/invenio
```

```bash
helm upgrade invenio charts/invenio \
  --namespace invenio \
  --values values-uchicago.yaml \
  --values values-uchicago-private.yaml \
  --dry-run
```

```bash
kubectl get ingress invenio -n invenio -o yaml
```

---

## 6) Deploy

```bash
helm upgrade invenio charts/invenio \
  --namespace invenio \
  --values values-uchicago.yaml \
  --values values-uchicago-private.yaml
```

---

## 7) Post-cutover validation

### A. DNS and ingress

```bash
kubectl get ingress invenio -n invenio
```

```bash
dig +short knowledge.uchicago.edu
```

### B. TLS certificate served

```bash
curl -Iv https://knowledge.uchicago.edu
```

```bash
openssl s_client -connect knowledge.uchicago.edu:443 -servername knowledge.uchicago.edu </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

### C. App behavior

- Verify homepage and search load over HTTPS.
- Verify login flow (including SSO redirect + callback).
- Verify API endpoint health at `https://knowledge.uchicago.edu/api`.

---

## 8) Rollback notes

If immediate rollback is needed:

1. Re-point DNS/ingress/certificate settings to previous hostname.
2. Re-run Helm upgrade using prior values.
3. Keep both old/new OAuth redirect URIs temporarily to avoid login failures during transition.

---

## 9) Ownership summary

- Platform/DevOps: ACM cert, DNS/ExternalDNS, ingress, Helm release.
- Application: trusted-host fallback, OAuth callback registration.
- Security: confirm no basic auth remains and HTTPS is enforced.

## 10) How to get an ACM certificate for `knowledge.uchicago.edu`

Use ACM DNS validation in the same region as the ALB/EKS cluster (`us-east-2` in this setup).

### A. Request the certificate

```bash
AWS_PROFILE=<your-profile>
CERT_ARN=$(AWS_PROFILE="$AWS_PROFILE" aws acm request-certificate \
  --domain-name knowledge.uchicago.edu \
  --validation-method DNS \
  --region us-east-2 \
  --query 'CertificateArn' \
  --output text)

echo "$CERT_ARN"
```

Optional transition setup (if you want one cert covering both old and new hostnames):

```bash
AWS_PROFILE=<your-profile>
CERT_ARN=$(AWS_PROFILE="$AWS_PROFILE" aws acm request-certificate \
  --domain-name knowledge.uchicago.edu \
  --subject-alternative-names uchicago.cottagelabs.com \
  --validation-method DNS \
  --region us-east-2 \
  --query 'CertificateArn' \
  --output text)

echo "$CERT_ARN"
```

### B. Get the DNS validation record from ACM

```bash
AWS_PROFILE=<your-profile> aws acm describe-certificate \
  --certificate-arn "$CERT_ARN" \
  --region us-east-2 \
  --query 'Certificate.DomainValidationOptions[].ResourceRecord' \
  --output table
```

Create the returned CNAME record in the DNS zone that serves `knowledge.uchicago.edu`.

### C. Wait for ACM status `ISSUED`

```bash
AWS_PROFILE=<your-profile> aws acm describe-certificate \
  --certificate-arn "$CERT_ARN" \
  --region us-east-2 \
  --query 'Certificate.{Status:Status,Domain:DomainName}' \
  --output table
```

Proceed once status is `ISSUED`.

### D. Update Helm private values with the cert ARN

In `values-uchicago-private.yaml`:

```yaml
ingress:
  annotations:
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-2:<account-id>:certificate/<cert-id>
```

### E. Important notes

- The ACM certificate must be in the same region as the ALB (`us-east-2` here).
- Keep the ACM DNS validation CNAME in place for automatic renewal.
- You do not generate TLS private keys manually in this ALB + ACM setup.
