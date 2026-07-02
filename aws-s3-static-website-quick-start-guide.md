# AWS S3 Static Website Quick-Start Guide

**Version:** `v1.2.2`  
**Supplement type:** Quick-start companion guide  
**Based on full guide:** [`aws-s3-static-website-setup.md`](aws-s3-static-website-setup.md)  
**Source guide version:** `v1.2.0`  
**Recommended path:** Shared setup → Track B → CloudFront + HTTPS + private S3 bucket  
**Best for:** Getting the calculator site online quickly using the recommended CloudFront/OAC architecture.

This is the short version of the full AWS S3 static website guide. It keeps the most important steps, warnings, settings, and checks, but skips most of the deeper explanations, comparisons, and reference sections.

Use the full guide when you need more explanation about S3 website endpoints, CloudFront origins, OAC, bucket policies, Cloudflare DNS, certificates, error pages, clean URLs, troubleshooting, or optional advanced hardening.

---

## Before you start

You need:

- An AWS account.
- Access to the AWS Management Console.
- Access to Cloudflare DNS for `jeffskone.com`.
- These static website files ready on your computer:

```text
index.html
style.css
calculator.js
```

Recommended example values:

| Item | Value |
|---|---|
| S3 bucket | `jefsko-calculator-site` |
| S3 bucket region | `us-west-2` |
| CloudFront/ACM certificate region | `us-east-1` |
| Primary domain | `calculator.jeffskone.com` |
| Short alias | `calc.jeffskone.com` |

Important:

```text
S3 bucket region: us-west-2
CloudFront viewer certificate in ACM: us-east-1
```

CloudFront viewer certificates from AWS Certificate Manager must be created in `us-east-1`, even if the S3 bucket is in `us-west-2`.

---

## What this builds

Recommended final architecture:

```text
Visitor
  ↓
calculator.jeffskone.com or calc.jeffskone.com
  ↓
Cloudflare DNS
  ↓
Amazon CloudFront
  ↓
OAC-signed request
  ↓
Private S3 bucket: jefsko-calculator-site
```

Recommended final access-control state:

| Area | Final setting |
|---|---|
| S3 Static website hosting | Disabled unless intentionally needed |
| S3 Block Public Access | Enabled |
| S3 bucket policy | No public-read statement; allow only CloudFront OAC |
| CloudFront origin | Normal S3 bucket origin, not S3 website endpoint |
| CloudFront default root object | `index.html` |
| Cloudflare website DNS records | Point to the CloudFront distribution domain |
| ACM validation DNS records | DNS only |

---

## Cost, safety, and security warning

This setup can create AWS resources that may cost money, especially CloudFront requests, data transfer, logs, WAF if enabled, or extra test resources. For a tiny static site, costs are usually small, but they are not automatically zero.

Do not upload private files, secrets, API keys, passwords, backups, or personal documents to the website bucket.

For the final recommended setup, do **not** leave the bucket publicly readable. Use CloudFront OAC and keep S3 private.

---

## Quick-start steps

### 1. Prepare the local website files

Make sure these files exist locally:

```text
index.html
style.css
calculator.js
```

In `index.html`, references should be relative and match the filenames exactly:

```html
<link rel="stylesheet" href="style.css">
<script src="calculator.js" defer></script>
```

**Checkpoint:** Open `index.html` locally and confirm the calculator works before uploading.

---

### 2. Create the S3 bucket

1. Open **AWS Console → S3**.
2. Choose **Create bucket**.
3. Bucket type:

```text
General purpose
```

4. Bucket name:

```text
jefsko-calculator-site
```

5. AWS Region:

```text
US West (Oregon) us-west-2
```

6. Object Ownership:

```text
ACLs disabled (recommended)
```

7. Block Public Access:

```text
Block all public access: On
```

8. Bucket Versioning:

```text
Disable
```

9. Default encryption:

```text
SSE-S3 / Amazon S3 managed keys
```

10. Tags are optional. Reasonable examples:

```text
Project = CalculatorSite
Environment = Test
Owner = Jeff
ManagedBy = Manual
Purpose = StaticWebsite
```

11. Choose **Create bucket**.

**Checkpoint:** The bucket exists in S3 as `jefsko-calculator-site`.

---

### 3. Upload the website files

1. Open the S3 bucket.
2. Choose **Upload**.
3. Add or drag-and-drop:

```text
index.html
style.css
calculator.js
```

4. Choose **Upload**.
5. Confirm the files appear directly in the bucket root.

Correct:

```text
index.html
style.css
calculator.js
```

Incorrect:

```text
calculator-site/index.html
calculator-site/style.css
calculator-site/calculator.js
```

**Checkpoint:** The three files are visible at the bucket root.

---

### 4. Optional: Do a quick S3-only HTTP test

This step is optional. It is useful for learning and quick testing, but it is **not** the final recommended setup.

To test directly from S3:

1. Open the bucket.
2. Go to **Properties → Static website hosting**.
3. Choose **Edit**.
4. Enable static website hosting.
5. Hosting type:

```text
Host a static website
```

6. Index document:

```text
index.html
```

7. Error document:

```text
index.html
```

8. Save changes.
9. Go to **Permissions → Block public access**.
10. Temporarily turn off **Block all public access**.
11. Add a public-read bucket policy for the website files.

Example temporary Track A policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForStaticWebsiteFiles",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::jefsko-calculator-site/*"
    }
  ]
}
```

12. Test the S3 website endpoint shown in the bucket properties.

**Checkpoint:** The S3 website endpoint opens the calculator over HTTP.

Important: this is a temporary public S3 test. For the final CloudFront/OAC setup, remove this public-read policy and turn Block Public Access back on.

---

## Stop here if your goal is only a quick S3 test

If you only wanted to prove that S3 static website hosting works, you can stop after the S3 website endpoint test.

But remember:

- The S3 website endpoint is HTTP-only.
- The bucket is publicly readable while Track A public access is enabled.
- This is not the recommended final setup for `calculator.jeffskone.com` or `calc.jeffskone.com`.

For the recommended final setup, continue to Track B.

---

### 5. Request the CloudFront certificate in ACM

1. Open **AWS Certificate Manager**.
2. Switch the AWS Region to:

```text
US East (N. Virginia) us-east-1
```

3. Choose **Request certificate**.
4. Choose **Request a public certificate**.
5. Add these domain names:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

6. Choose DNS validation.
7. Request the certificate.

**Checkpoint:** The certificate exists in ACM and shows `Pending validation`.

---

### 6. Validate the ACM certificate in Cloudflare

1. Open the certificate in ACM.
2. Copy each ACM validation CNAME name and value.
3. Open **Cloudflare → jeffskone.com → DNS → Records**.
4. Add each validation CNAME.
5. Set proxy status to:

```text
DNS only
```

6. Wait for ACM status to change to:

```text
Issued
```

**Checkpoint:** The ACM certificate is `Issued` in `us-east-1`.

---

### 7. Create the CloudFront distribution

1. Open **AWS Console → CloudFront**.
2. Choose **Create distribution**.
3. Distribution name:

```text
calculator-site
```

4. Origin type:

```text
Amazon S3
```

5. Origin domain: choose the normal S3 bucket origin, not the S3 website endpoint.

Good origin style:

```text
jefsko-calculator-site.s3.us-west-2.amazonaws.com
```

Avoid for the recommended final setup:

```text
jefsko-calculator-site.s3-website-us-west-2.amazonaws.com
```

6. Origin access:

```text
Origin Access Control / OAC
```

7. Signing behavior:

```text
Sign requests / Sign requests (recommended)
```

8. Viewer protocol policy:

```text
Redirect HTTP to HTTPS
```

9. Allowed HTTP methods:

```text
GET, HEAD
```

10. Default root object:

```text
index.html
```

11. Alternate domain name (CNAME):

```text
calculator.jeffskone.com
calc.jeffskone.com
```

12. Custom SSL/TLS certificate: choose the issued ACM certificate from `us-east-1`.
13. TLS/security policy: choose the current AWS-recommended TLS 1.2-or-newer policy.
14. Create the distribution and wait until deployment finishes.

**Checkpoint:** CloudFront gives you a distribution domain like `d111111abcdef8.cloudfront.net`, and `https://<DISTRIBUTION_DOMAIN>/` opens the calculator.

---

### 8. Make S3 private and allow only CloudFront

If you completed the optional S3-only test, remove public access now.

1. Open **S3 → jefsko-calculator-site → Permissions**.
2. Remove the entire public-read policy statement with:

```json
"Principal": "*"
```

3. Turn **Block all public access** back on.
4. Keep or add the CloudFront OAC bucket policy.

Example OAC read-only bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipalReadOnly",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::jefsko-calculator-site/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
        }
      }
    }
  ]
}
```

Replace:

```text
<ACCOUNT_ID>
<DISTRIBUTION_ID>
```

5. Disable S3 Static website hosting unless you intentionally need the S3 website endpoint.

**Checkpoint:** CloudFront still works, but direct public S3 access should not work.

---

### 9. Point Cloudflare DNS to CloudFront

In Cloudflare DNS, create website CNAME records that point to the CloudFront distribution domain.

For `calculator.jeffskone.com`:

```text
Type: CNAME
Name: calculator
Target: <DISTRIBUTION_DOMAIN>
Proxy status: DNS only initially
TTL: Auto
```

For `calc.jeffskone.com`:

```text
Type: CNAME
Name: calc
Target: <DISTRIBUTION_DOMAIN>
Proxy status: DNS only initially
TTL: Auto
```

Use the CloudFront distribution domain name, not the distribution ID, S3 bucket name, S3 website endpoint, or CloudFront IP addresses.

**Checkpoint:** Cloudflare DNS records exist and point to the CloudFront distribution domain.

---

### 10. Test the final URLs

Test:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

Also test:

```text
https://<DISTRIBUTION_DOMAIN>
http://<S3_WEBSITE_ENDPOINT>
https://jefsko-calculator-site.s3.us-west-2.amazonaws.com/index.html
```

Expected final behavior:

| URL | Expected result |
|---|---|
| `https://calculator.jeffskone.com` | Works |
| `https://calc.jeffskone.com` | Works |
| `https://<DISTRIBUTION_DOMAIN>` | Works by default, unless optionally blocked |
| S3 website endpoint | Should not work if S3 website hosting/public access is disabled |
| Direct S3 object URL | Should not work publicly if the bucket is private |

**Checkpoint:** The custom domains work over HTTPS, and S3 is no longer directly public.

---

### 11. Update the website later

When you edit the site:

1. Upload the changed file or files to S3.
2. Create a CloudFront invalidation if the change does not show up.

AWS CLI example:

```bash
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/*"
```

**Checkpoint:** The updated site appears at the custom domains.

---

## Optional advanced items

Use the full guide for these:

- Blocking the standard `*.cloudfront.net` distribution domain.
- Adding clean URL rewrites such as `/about/ → /about/index.html`.
- Configuring `404.html` with CloudFront custom error responses.
- SPA fallback behavior.
- Cloudflare proxy mode after the DNS-only setup is working.

---

## If something fails

| Problem | Most likely fix |
|---|---|
| ACM certificate stays pending | Confirm ACM validation CNAMEs exist in Cloudflare and are DNS-only. |
| Certificate does not appear in CloudFront | Confirm it was requested in `us-east-1`. |
| CloudFront root URL fails but `/index.html` works | Set CloudFront Default root object to `index.html`. |
| Custom domain does not work | Confirm CloudFront Alternate domain names include the hostname and Cloudflare CNAME points to the distribution domain. |
| S3 website endpoint still works after final setup | Disable S3 Static website hosting and remove public-read bucket policy. |
| Direct S3 object URL works publicly | Check Block Public Access and remove public `Principal: "*"` read policy. |
| CloudFront returns old files | Create an invalidation or wait for cache expiration. |
| `/about/` does not load `/about/index.html` | Use a CloudFront Function rewrite or link directly to `/about/index.html`. |

---

## Where to read more

Use the full guide when you need:

- Deep explanation of Track A vs. Track B.
- Bucket naming and custom-domain matching rules.
- Certificate and wildcard examples.
- OAC bucket policy details.
- CloudFront behavior settings.
- Cloudflare DNS-only vs. proxied behavior.
- Direct S3 access testing.
- CloudFront Functions.
- Custom error responses.
- Troubleshooting and reference commands.

Full guide: [`aws-s3-static-website-setup.md`](aws-s3-static-website-setup.md)
