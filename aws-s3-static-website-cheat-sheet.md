# AWS S3 Static Website Cheat Sheet

**Version:** `v1.2.2`  
**Supplement type:** Cheat sheet  
**Full guide:** [`aws-s3-static-website-setup.md`](aws-s3-static-website-setup.md)  
**Quick-start guide:** [`aws-s3-static-website-quick-start-guide-v1.2.2.md`](aws-s3-static-website-quick-start-guide-v1.2.2.md)  
**Source guide version:** `v1.2.0`

## Path

```text
Shared setup
  ↓
CloudFront + ACM + Cloudflare DNS
  ↓
Private S3 bucket with OAC
```

Recommended final path:

```text
Visitor → Cloudflare DNS → CloudFront → OAC → private S3 bucket
```

## Use this when

Use this after you understand the full guide or quick-start guide and want a compact repeat checklist.

This cheat sheet assumes the recommended final setup: CloudFront in front of a private S3 bucket.

## Minimum prerequisites

- AWS account.
- Cloudflare DNS access for `jeffskone.com`.
- Website files:

```text
index.html
style.css
calculator.js
```

- S3 bucket region:

```text
us-west-2
```

- ACM certificate region for CloudFront:

```text
us-east-1
```

## Example values

| Item | Value |
|---|---|
| Bucket | `jefsko-calculator-site` |
| Primary domain | `calculator.jeffskone.com` |
| Alias domain | `calc.jeffskone.com` |
| S3 region | `us-west-2` |
| CloudFront certificate region | `us-east-1` |

## Fast setup

1. Create S3 bucket:

```text
jefsko-calculator-site
us-west-2
ACLs disabled
Block all public access: On
Default encryption: SSE-S3
```

2. Upload files to the bucket root:

```text
index.html
style.css
calculator.js
```

3. Request ACM certificate in `us-east-1`:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

4. Add ACM validation CNAMEs in Cloudflare:

```text
Type: CNAME
Proxy status: DNS only
TTL: Auto
```

5. Create CloudFront distribution:

```text
Origin: normal S3 bucket origin
Origin access: OAC
Signing behavior: Sign requests recommended
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed methods: GET, HEAD
Default root object: index.html
Alternate domain names: calculator.jeffskone.com, calc.jeffskone.com
Certificate: ACM certificate from us-east-1
```

6. Add or keep OAC bucket policy:

```text
Principal: cloudfront.amazonaws.com
Action: s3:GetObject
Resource: arn:aws:s3:::jefsko-calculator-site/*
Condition: AWS:SourceArn = CloudFront distribution ARN
```

7. Make sure S3 is private:

```text
Block all public access: On
No public-read Principal "*" bucket policy
S3 Static website hosting: Disabled unless intentionally needed
```

8. Add Cloudflare website CNAMEs:

```text
calculator → <DISTRIBUTION_DOMAIN> → DNS only initially
calc       → <DISTRIBUTION_DOMAIN> → DNS only initially
```

9. Test final URLs:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

10. Optional: invalidate CloudFront after updates:

```bash
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/*"
```

## Final target state

| Area | Target |
|---|---|
| Custom domains | Work over HTTPS |
| CloudFront distribution domain | Works by default unless blocked |
| S3 website endpoint | Does not work publicly in final private setup |
| Direct S3 object URL | Does not work publicly |
| Bucket policy | Allows CloudFront OAC, not public read |
| Cloudflare routing | Website CNAMEs point to CloudFront |

## Do not share / avoid / warning

- Do not upload secrets or private files to the website bucket.
- Do not leave public-read S3 access enabled for the final setup.
- Do not use the S3 website endpoint as the CloudFront origin when you want OAC/private S3.
- Do not create the CloudFront viewer certificate in `us-west-2`; use `us-east-1`.
- Do not proxy ACM validation CNAMEs through Cloudflare.
- Do not point Cloudflare website DNS to the S3 website endpoint for the recommended final setup.

## Optional quick S3-only test

Track A test path:

```text
Enable S3 Static website hosting
Turn off Block Public Access
Add temporary public-read policy
Test S3 website endpoint over HTTP
```

Stop there only if HTTP-only public S3 hosting is acceptable.

Before final Track B use:

```text
Remove public-read policy
Turn Block Public Access back on
Disable S3 Static website hosting unless intentionally needed
Use CloudFront/OAC/custom domains
```

## Cleanup

After CloudFront works:

- Remove the public-read bucket policy statement if Track A was used.
- Turn S3 Block Public Access back on.
- Disable S3 Static website hosting unless intentionally needed.
- Delete unused or wrong-region ACM certificates after the correct `us-east-1` certificate is working.
- Delete unused test CloudFront distributions only after disabling them first.
- Delete unused test buckets after emptying them.

## Common fixes

| Problem | Fix |
|---|---|
| Certificate not available in CloudFront | Request/import the certificate in `us-east-1`. |
| ACM validation stuck | Check Cloudflare CNAME name/value and keep it DNS-only. |
| Custom domain fails | Add hostname to CloudFront Alternate domain names and point Cloudflare CNAME to the distribution domain. |
| Root `/` fails | Set CloudFront Default root object to `index.html`. |
| `/index.html` works but `/about/` fails | Add a CloudFront Function rewrite or link directly to real object paths. |
| S3 website endpoint still works | Disable S3 Static website hosting and remove public access. |
| Direct S3 object URL works | Recheck Block Public Access and bucket policy. |
| CloudFront shows old content | Invalidate `/*` or wait for cache expiration. |
| Standard CloudFront domain works | Normal default behavior; optionally block it with a CloudFront Function. |

## Useful placeholders

| Placeholder | Meaning |
|---|---|
| `<DISTRIBUTION_DOMAIN>` | CloudFront domain such as `d111111abcdef8.cloudfront.net` |
| `<DISTRIBUTION_ID>` | CloudFront distribution ID |
| `<ACCOUNT_ID>` | AWS account ID |
| `<CERTIFICATE_ARN>` | ACM certificate ARN from `us-east-1` |
| `<BUCKET_NAME>` | `jefsko-calculator-site` |
| `<OBJECTS_ARN>` | `arn:aws:s3:::jefsko-calculator-site/*` |
