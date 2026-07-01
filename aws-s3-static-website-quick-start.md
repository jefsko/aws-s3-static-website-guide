# AWS S3 Static Website Quick Start

**Filename:** `aws-s3-static-website-quick-start.md`  
**Version:** `v1.2.0`  
**Last updated:** 2026-07-01  
**Primary AWS Region for the S3 bucket:** `us-west-2` — US West (Oregon)  
**Required AWS Region for CloudFront viewer certificates:** `us-east-1` — US East (N. Virginia)

Working example:

- S3 bucket: `jefsko-calculator-site`
- Primary domain: `calculator.jeffskone.com`
- Short alias domain: `calc.jeffskone.com`
- Website files:
  - `index.html`
  - `style.css`
  - `calculator.js`

---

## Table of Contents

1. [Summary](#1-summary)
2. [How to use this guide: shared setup, Track A, and Track B](#2-how-to-use-this-guide-shared-setup-track-a-and-track-b)
3. [Recommended setup](#3-recommended-setup)
4. [Prerequisites](#4-prerequisites)
5. [Project file structure](#5-project-file-structure)
6. [Track A vs. Track B](#6-track-a-vs-track-b)
7. [Bucket naming rules and domain-name matching](#7-bucket-naming-rules-and-domain-name-matching)
8. [Custom domain options: S3-only vs. CloudFront](#8-custom-domain-options-s3-only-vs-cloudfront)
9. [Certificates, wildcard certificates, and subdomain levels](#9-certificates-wildcard-certificates-and-subdomain-levels)
10. [Shared setup: Create the S3 bucket](#10-shared-setup-create-the-s3-bucket)
11. [Shared setup: Upload the website files](#11-shared-setup-upload-the-website-files)
12. [Track A: Enable S3 static website hosting](#12-track-a-enable-s3-static-website-hosting)
13. [Track A: Make the bucket publicly readable](#13-track-a-make-the-bucket-publicly-readable)
14. [Track A: Test the S3 website endpoint](#14-track-a-test-the-s3-website-endpoint)
15. [Track B: Request an ACM certificate for CloudFront](#15-track-b-request-an-acm-certificate-for-cloudfront)
16. [Track B: Validate the ACM certificate using Cloudflare DNS](#16-track-b-validate-the-acm-certificate-using-cloudflare-dns)
17. [Track B: Create the CloudFront distribution](#17-track-b-create-the-cloudfront-distribution)
18. [Track B: Make S3 private again and allow only CloudFront](#18-track-b-make-s3-private-again-and-allow-only-cloudfront)
19. [Track B: Add custom domains to CloudFront](#19-track-b-add-custom-domains-to-cloudfront)
20. [Track B: Point Cloudflare DNS to CloudFront](#20-track-b-point-cloudflare-dns-to-cloudfront)
21. [Track B: Test the final HTTPS custom domains](#21-track-b-test-the-final-https-custom-domains)
22. [Final access expectations and URL behavior](#22-final-access-expectations-and-url-behavior)
23. [Access control deep dive: S3, CloudFront, and custom domains](#23-access-control-deep-dive-s3-cloudfront-and-custom-domains)
24. [Updating the website after setup](#24-updating-the-website-after-setup)
25. [Troubleshooting](#25-troubleshooting)
26. [Optional alternatives outside AWS](#26-optional-alternatives-outside-aws)
27. [Reference commands](#27-reference-commands)
28. [Variables, ARNs, and placeholders reference](#28-variables-arns-and-placeholders-reference)
29. [Recommended AWS tags](#29-recommended-aws-tags)
30. [AWS and Cloudflare console UI drift note](#30-aws-and-cloudflare-console-ui-drift-note)
31. [Official references](#31-official-references)
32. [Changelog](#32-changelog)

---

## 1. Summary

This guide shows how to host a simple static website on AWS using Amazon S3.

Your website has three files:

```text
index.html
style.css
calculator.js
```

For this guide, keep all three files in the root of the S3 bucket.

There are two setup tracks:

| Track | Name | What it gives you | Best for |
|---|---|---|---|
| Track A | Minimal S3-only setup | A public S3 website URL using HTTP | Quick testing |
| Track B | Recommended HTTPS/custom-domain setup | CloudFront, HTTPS, AWS Certificate Manager, Cloudflare DNS, and custom domains | Real use |

The final recommended URLs are:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

The most important idea:

> S3 is where the files live. CloudFront is the public HTTPS front door. Cloudflare manages DNS for your domain.

The recommended final architecture is:

```text
Visitor
  ↓
calculator.jeffskone.com or calc.jeffskone.com
  ↓
Cloudflare DNS
  ↓
Amazon CloudFront
  ↓
Private S3 bucket: jefsko-calculator-site
```

---

### What changed in v1.2.0

This version expands the CloudFront and S3 access-control guidance.

The most important clarification is this:

```text
For the recommended final setup, CloudFront should use a normal S3 bucket origin with OAC.
S3 Static website hosting should be disabled unless you intentionally need the S3 website endpoint.
The custom domain should point to CloudFront, not directly to S3.
```

This version also adds more detail about Cloudflare CNAME records, ACM validation records, CloudFront default root object behavior, custom error responses, CloudFront Functions, and how to test whether direct S3 access and the default CloudFront domain are working or blocked.

---

### What changed in v1.1.0

This version clarifies the setup flow after reader feedback.

The most important clarification is this:

```text
Track B is not a continuation of the public S3 website endpoint setup.
Track B does reuse the same S3 bucket and uploaded files.
```

So the guide now separates the work into three phases:

```text
Shared setup
  ↓
Optional Track A: S3-only HTTP test
  ↓
Recommended Track B: CloudFront + HTTPS + private S3 bucket
```

---

## 2. How to use this guide: shared setup, Track A, and Track B

This guide uses “tracks,” but Track B is not completely independent from the earlier S3 steps.

Both Track A and Track B start with the same shared setup:

1. Create the S3 bucket.
2. Upload `index.html`, `style.css`, and `calculator.js`.

After that, choose your path:

- If you want the fastest HTTP-only test, continue with Track A.
- If you want the recommended HTTPS custom-domain setup, continue with Track B.

For the final recommended Track B setup, **S3 static website hosting is not required**. CloudFront can use the regular S3 bucket origin with Origin Access Control, which lets the bucket remain private.

### Which sections should I follow?

| Topic | Applies to Track A? | Applies to Track B? | Notes |
|---|---:|---:|---|
| Create S3 bucket | Yes | Yes | Shared setup. |
| Upload website files | Yes | Yes | Shared setup. |
| Enable S3 static website hosting | Yes | No, not for the final recommended setup | Only needed for S3 website endpoint testing. |
| Make bucket publicly readable | Yes | No, not for the final recommended setup | Track B should use CloudFront OAC and keep S3 private. |
| Test S3 website endpoint | Yes | Optional | Useful only as a quick HTTP test. |
| Request ACM certificate | No | Yes | Required for HTTPS custom domains. |
| Create CloudFront distribution | No | Yes | Required for the recommended setup. |
| Make S3 private again | Only if Track A was done first | Yes | Final Track B should not leave the bucket public. |
| Add Cloudflare DNS records | No | Yes | Required for custom domains. |

If you are following Track B from the beginning, you can skip the public S3 website endpoint test. If you are new to S3, Track A can still be a helpful temporary test before moving to the recommended CloudFront setup.

### Recommended reading path

For most readers, use this path:

```text
1. Read the Summary and Recommended setup.
2. Complete Shared setup: create the bucket and upload files.
3. Choose one path:
   - Track A for a quick HTTP-only S3 test.
   - Track B for the recommended HTTPS custom-domain setup.
4. If you did Track A first, clean up public S3 access after Track B works.
```

---

## 3. Recommended setup

For your project, use this setup:

```text
S3 bucket region: us-west-2
S3 bucket name: jefsko-calculator-site
CloudFront certificate region: us-east-1
Primary domain: calculator.jeffskone.com
Alias/short domain: calc.jeffskone.com
```

Why this setup?

- `us-west-2` is a good region choice for Seattle / West Coast usage.
- `jefsko-calculator-site` is a good bucket name because it is lowercase, descriptive, and uses dashes.
- CloudFront gives you HTTPS.
- CloudFront lets one bucket serve multiple domain names.
- CloudFront lets the bucket stay private.
- Cloudflare DNS is already where the domain is managed.

Important AWS region detail:

> Use `us-west-2` for the S3 bucket, but use `us-east-1` for the ACM certificate that CloudFront uses.

This is not a Seattle/West Coast preference issue. It is an AWS CloudFront requirement. CloudFront viewer certificates from AWS Certificate Manager must be requested or imported in `us-east-1`.


### Recommended final access-control target

For the recommended final Track B setup, aim for this state:

| Area | Recommended final setting |
|---|---|
| S3 Static website hosting | Disabled, unless you intentionally need S3 website endpoint behavior |
| S3 Block Public Access | Enabled |
| S3 bucket policy | No public-read statement; allow only the CloudFront distribution through OAC |
| CloudFront origin | Normal S3 bucket origin, not the S3 website endpoint |
| CloudFront Default root object | `index.html` |
| Cloudflare website DNS records | Point custom hostnames to the CloudFront distribution domain |
| Cloudflare ACM validation records | DNS only |
| CloudFront standard distribution domain | Works by default; optionally block with CloudFront Function |

The recommended mental model is:

```text
User
  ↓
https://calculator.jeffskone.com or https://calc.jeffskone.com
  ↓
Cloudflare DNS
  ↓
CloudFront distribution
  ↓
OAC-signed request
  ↓
Private S3 bucket origin
```

---

### Why is the CloudFront certificate in `us-east-1` if the bucket is in `us-west-2`?

This is confusing at first because most AWS services feel regional.

For this project, there are two different regional ideas happening at the same time:

```text
S3 bucket origin:
  us-west-2
  This is where the source website files live.

CloudFront viewer certificate:
  us-east-1
  This is where CloudFront expects ACM certificates for viewer HTTPS.
```

Think of it this way:

```text
Viewer/browser HTTPS certificate = CloudFront-side setting
Website file storage = S3-side setting
```

The certificate for `calculator.jeffskone.com` and `calc.jeffskone.com` is used when a visitor's browser connects to CloudFront over HTTPS. That is called **viewer HTTPS** because the viewer is the person/browser visiting the site.

CloudFront is a global edge service, not a normal single-region web server. AWS requires ACM certificates used for CloudFront viewer HTTPS to be created or imported in:

```text
US East (N. Virginia) us-east-1
```

After the certificate is associated with the CloudFront distribution, AWS distributes it to CloudFront's edge locations. That does **not** mean your website files moved to Virginia. Your S3 bucket can still be in:

```text
US West (Oregon) us-west-2
```

So the recommended mental model is:

```text
Create/store website files in us-west-2.
Create the CloudFront viewer certificate in us-east-1.
Connect them through the CloudFront distribution.
```

Why not create the certificate in `us-west-2`?

Because CloudFront will not use an ACM viewer certificate from `us-west-2`. If you request the certificate there, it may be valid for other AWS services in that region, but it will not appear as the usable CloudFront viewer certificate when you configure alternate domain names.

A certificate in `us-west-2` could still be useful for regional services such as an Application Load Balancer or a regional API Gateway custom domain. It is just not the right certificate location for CloudFront viewer HTTPS.

Official AWS docs say the same thing in a few places:

- CloudFront ACM certificates for viewer HTTPS must be in `us-east-1`.
- ACM certificates are regional resources.
- ACM certificates in `us-east-1` associated with CloudFront are distributed to the geographic locations configured for the CloudFront distribution.


---


## 4. Prerequisites

You need:

1. An active AWS account.
2. Access to the AWS Management Console.
3. Access to Cloudflare for:

```text
jeffskone.com
```

4. These website files ready on your computer:

```text
index.html
style.css
calculator.js
```

5. Your files should work locally before uploading.

Optional but recommended:

1. AWS CLI installed.
2. AWS CLI configured.
3. A local project folder containing the website files.

Example local folder:

```text
calculator-site/
├── index.html
├── style.css
└── calculator.js
```

To check whether AWS CLI is installed:

```bash
aws --version
```

To check whether AWS CLI credentials are working:

```bash
aws sts get-caller-identity
```

---


## 5. Project file structure

Keep the files in the root of the bucket:

```text
/
├── index.html
├── style.css
└── calculator.js
```

In `index.html`, make sure the references are relative and match the filenames exactly.

Example:

```html
<link rel="stylesheet" href="style.css">
<script src="calculator.js" defer></script>
```

Important:

S3 object names are case-sensitive.

These are not the same file:

```text
calculator.js
Calculator.js
CALCULATOR.js
```

Use:

```text
index.html
style.css
calculator.js
```

---


## 6. Track A vs. Track B

### Track A: Minimal S3-only setup

Track A gets your site online quickly using only S3.

You will get a URL similar to:

```text
http://jefsko-calculator-site.s3-website-us-west-2.amazonaws.com
```

or:

```text
http://jefsko-calculator-site.s3-website.us-west-2.amazonaws.com
```

Do not guess the exact URL. AWS will show it after static website hosting is enabled.

Track A is good for:

- Fast testing.
- Learning S3 static website hosting.
- Confirming your files work after upload.

Track A limitations:

- Uses HTTP, not HTTPS.
- Requires the bucket to be public.
- Does not directly give you `https://calculator.jeffskone.com`.
- Is not the recommended final public setup.

### Track B: Recommended setup

Track B adds:

- CloudFront
- HTTPS
- AWS Certificate Manager
- Cloudflare DNS
- Custom domains
- A private S3 bucket

Track B gives you:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

Track B is the recommended final setup.

### Should Track B be in the main guide or an appendix?

Track B belongs in the main guide.

Reason:

This is not a random optional appendix. It is the recommended production-style path. Track A is the quick test. Track B is the real final setup.

Appendices are better for deeper reference material, alternate providers, cleanup, cost notes, and advanced wildcard examples.

---

### Shared setup dependency

Track B is not a continuation of the public S3 website endpoint setup, but it does reuse the same S3 bucket and uploaded files.

Use this distinction:

```text
Shared setup:
- Create the bucket.
- Upload files.

Track A:
- Enable S3 static website hosting.
- Make the bucket public.
- Test the S3 website endpoint.

Track B:
- Request ACM certificate.
- Create CloudFront distribution.
- Use Origin Access Control.
- Point custom domains to CloudFront.
- Keep or restore S3 bucket privacy.
```

If your goal is the recommended HTTPS custom-domain setup, complete the shared S3 bucket creation and file upload steps first. Then go to Track B.

You do not need to complete the public S3 website endpoint steps unless you want a quick temporary test.

---

## 7. Bucket naming rules and domain-name matching

Your chosen bucket name is:

```text
jefsko-calculator-site
```

This is a good name.

It is:

- Lowercase.
- Descriptive.
- Uses dashes.
- Has no spaces.
- Has no underscores.
- Does not contain periods.

S3 bucket names must be globally unique across AWS, not just unique inside your AWS account.

So this might fail:

```text
calculator
```

Because someone else on AWS may already own it.

This is much better:

```text
jefsko-calculator-site
```

If that bucket name is already taken, use a suffix:

```text
jefsko-calculator-site-2026
jefsko-calculator-site-prod
jefsko-calculator-site-v1
```

Avoid:

```text
JefskoCalculatorSite
jefsko_calculator_site
jefsko calculator site
```

Why?

- Uppercase letters are not allowed.
- Underscores are not allowed.
- Spaces are not allowed.
- Dashes are the safest separator.

### Does the bucket need to match the custom domain?

It depends on the architecture.

#### S3-only custom domain

If you point a custom hostname directly to S3, then yes, the bucket name needs to match the hostname exactly.

For example, this hostname:

```text
calculator.jeffskone.com
```

would need this bucket:

```text
calculator.jeffskone.com
```

This hostname:

```text
calc.jeffskone.com
```

would need this bucket:

```text
calc.jeffskone.com
```

This is not just a style recommendation. It is how S3 virtual-hosted website requests work. S3 uses the hostname in the request to decide which bucket should answer.

So if a browser requests:

```text
http://calculator.jeffskone.com/
```

S3 sees the hostname:

```text
calculator.jeffskone.com
```

and looks for a bucket with that exact name.

That is why S3-only custom-domain tutorials normally tell you to name the bucket exactly after the domain or subdomain.

#### CloudFront custom domain

If you put CloudFront in front of S3, the bucket name does not need to match the custom domain.

That means this is fine:

```text
Bucket: jefsko-calculator-site
Domains:
  calculator.jeffskone.com
  calc.jeffskone.com
```

CloudFront receives requests for the custom domains and then fetches files from the S3 bucket.

This is one of the reasons CloudFront is the better final setup.

---

### Dashes vs. periods in bucket names

A practical rule:

```text
Use dashes for normal S3 bucket names.
Use periods only when you intentionally need the bucket name to exactly match an S3-only custom domain.
```

| Use case | Recommended bucket name style | Example |
|---|---|---|
| Recommended CloudFront setup | Dashes | `jefsko-calculator-site` |
| S3-only custom domain for `calculator.jeffskone.com` | Periods, because exact hostname match is required | `calculator.jeffskone.com` |
| General private S3 bucket | Dashes | `project-name-static-site` |
| Bucket used with HTTPS virtual-hosted S3 requests | Avoid periods | `calculator-jeffskone-com` |

For this guide’s recommended CloudFront setup, this is preferred:

```text
jefsko-calculator-site
```

over this:

```text
calculator.jeffskone.com
```

Why?

Because the bucket does not need to match the custom domain when CloudFront is in front of S3. Also, AWS recommends avoiding periods in bucket names for best compatibility except for buckets used only for static website hosting. Periods can create HTTPS certificate-name matching issues with S3 virtual-hosted-style access.

For S3-only custom-domain hosting, the period-based bucket name is not just style. It is what allows S3 to match the incoming hostname to the bucket.

---

## 8. Custom domain options: S3-only vs. CloudFront

There are two main ways to connect a custom domain.

### Option 1: S3-only custom domain

Example:

```text
calculator.jeffskone.com → S3 website endpoint
```

This can work for HTTP.

But there are major limitations:

1. The bucket name must match the hostname exactly.
2. S3 website endpoints do not support HTTPS.
3. If you want both `calculator.jeffskone.com` and `calc.jeffskone.com`, you usually need one bucket per hostname or one bucket that redirects to the other.
4. The bucket must be public.
5. You do not get CloudFront caching or easy HTTPS.

Example S3-only bucket layout:

```text
Bucket 1:
  calculator.jeffskone.com
  Contains website files

Bucket 2:
  calc.jeffskone.com
  Redirects to calculator.jeffskone.com
```

That is more annoying than it looks.

It is okay if you only want a quick HTTP-only static site.

It is not the recommended final setup for this guide.

### Option 2: CloudFront custom domain

Example:

```text
calculator.jeffskone.com → CloudFront → S3
calc.jeffskone.com       → CloudFront → S3
```

This is the recommended setup.

Benefits:

1. HTTPS works.
2. One S3 bucket can serve multiple domains.
3. The bucket name does not need to match the domains.
4. The bucket can be private.
5. CloudFront can redirect HTTP to HTTPS.
6. CloudFront can cache the site near visitors.
7. Cloudflare DNS can point both hostnames to the same CloudFront distribution.

Recommended final setup:

```text
Cloudflare DNS:
  calculator.jeffskone.com → CloudFront distribution
  calc.jeffskone.com       → CloudFront distribution

CloudFront:
  Alternate domain names:
    calculator.jeffskone.com
    calc.jeffskone.com

S3:
  Bucket:
    jefsko-calculator-site
```

---


## 9. Certificates, wildcard certificates, and subdomain levels

For HTTPS, the certificate must cover the hostname the visitor types into the browser.

For this project, the browser hostnames are:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

So the certificate must cover both of those names.

### Recommended certificate for this project

Use an exact-name certificate with these names:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

This is the simplest and clearest option.

### Wildcard certificate option

You could also request:

```text
*.jeffskone.com
```

That would cover first-level subdomains like:

```text
calculator.jeffskone.com
calc.jeffskone.com
www.jeffskone.com
anything.jeffskone.com
```

But it does not cover the apex/root domain:

```text
jeffskone.com
```

And it does not cover deeper subdomains like:

```text
test.calc.jeffskone.com
alpha.calculator.jeffskone.com
```

Why?

Because ACM wildcard certificates only protect one subdomain level. Put another way, wildcard certificates cover one subdomain level at a time.

### If you want `*.calc.jeffskone.com`

If you want hostnames like:

```text
alpha.calc.jeffskone.com
beta.calc.jeffskone.com
test.calc.jeffskone.com
```

then request a certificate name like:

```text
*.calc.jeffskone.com
```

That covers one level below `calc.jeffskone.com`.

It covers:

```text
alpha.calc.jeffskone.com
beta.calc.jeffskone.com
test.calc.jeffskone.com
```

It does not cover the parent name:

```text
calc.jeffskone.com
```

It also does not cover deeper names like:

```text
one.two.calc.jeffskone.com
```

If you want both the parent and the wildcard children, include both:

```text
calc.jeffskone.com
*.calc.jeffskone.com
```

If you want all three of these:

```text
calculator.jeffskone.com
calc.jeffskone.com
*.calc.jeffskone.com
```

then the ACM certificate should include:

```text
calculator.jeffskone.com
calc.jeffskone.com
*.calc.jeffskone.com
```

### Can you create a wildcard for more than one level?

Not with one normal wildcard name.

This does not work as a public TLS certificate pattern:

```text
*.*.jeffskone.com
```

This also does not work:

```text
*.calc.*.com
```

The wildcard must replace one full DNS label. In normal public TLS certificate behavior, it protects only that one level.

Use multiple names instead.

Example:

```text
*.jeffskone.com
*.calc.jeffskone.com
*.dev.calc.jeffskone.com
```

And include any parent names you need exactly:

```text
jeffskone.com
calc.jeffskone.com
dev.calc.jeffskone.com
```

### CloudFront wildcard alternate domain names vs. certificate wildcard names

This topic is easy to mix up.

CloudFront can use wildcard alternate domain names, but every hostname still needs valid HTTPS certificate coverage.

For practical HTTPS setup, think certificate-first:

```text
Hostname visitor uses
  ↓
Must be covered by CloudFront certificate
  ↓
Must be added to CloudFront as exact or wildcard alternate domain name
  ↓
Must have DNS pointing to CloudFront
```

Example:

If you want:

```text
alpha.calc.jeffskone.com
beta.calc.jeffskone.com
```

then you need:

```text
Certificate:
  *.calc.jeffskone.com

CloudFront alternate domain name:
  *.calc.jeffskone.com

Cloudflare DNS:
  *.calc CNAME <cloudfront-domain-name>
```

If you also want:

```text
calc.jeffskone.com
```

then add the exact parent too:

```text
Certificate:
  calc.jeffskone.com
  *.calc.jeffskone.com

CloudFront alternate domain names:
  calc.jeffskone.com
  *.calc.jeffskone.com

Cloudflare DNS:
  calc   CNAME <cloudfront-domain-name>
  *.calc CNAME <cloudfront-domain-name>
```

### Common certificate choices

| Goal | Certificate names |
|---|---|
| Only current calculator domains | `calculator.jeffskone.com`, `calc.jeffskone.com` |
| Any first-level subdomain of `jeffskone.com` | `*.jeffskone.com` |
| Root domain plus any first-level subdomain | `jeffskone.com`, `*.jeffskone.com` |
| Anything one level below `calc.jeffskone.com` | `*.calc.jeffskone.com` |
| `calc.jeffskone.com` plus anything one level below it | `calc.jeffskone.com`, `*.calc.jeffskone.com` |
| Current calculator domains plus future calc children | `calculator.jeffskone.com`, `calc.jeffskone.com`, `*.calc.jeffskone.com` |

### My recommendation

For this guide, use:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

Do not use a wildcard yet unless you already know you want many related subdomains.

Reason:

- Exact names are easier to understand.
- Exact names are easier to troubleshoot.
- They match your current project perfectly.
- You can request a new certificate later if your domain needs expand.

Important:

You cannot add names to an existing ACM certificate. If you later want to add `*.calc.jeffskone.com`, request a new certificate with the full updated list and update the CloudFront distribution to use the new certificate.

---


## 10. Shared setup: Create the S3 bucket

This shared setup creates the S3 bucket used by both Track A and Track B.

### Step 1: Sign in to AWS

1. Go to the AWS Management Console.
2. Sign in to your AWS account.

---

### Step 2: Open S3

1. In the top AWS search box, search for:

```text
S3
```

2. Choose **S3**.
3. You should now be on the Amazon S3 page.

---

### Step 3: Create the bucket

1. Choose **Create bucket**.
2. For bucket type, choose:

```text
General purpose
```

3. For bucket name, enter:

```text
jefsko-calculator-site
```

4. For AWS Region, choose:

```text
US West (Oregon) us-west-2
```

Because you are in Seattle / the West Coast, `us-west-2` is a sensible default.

---

### Step 4: Object Ownership

For **Object Ownership**, choose:

```text
ACLs disabled (recommended)
```

This keeps the bucket simpler and avoids object-level permission confusion.

---

### Step 5: Block Public Access

For now, leave this enabled:

```text
Block all public access: On
```

We will intentionally change it later for Track A.

---

### Step 6: Bucket Versioning

For this simple quick start, choose:

```text
Disable
```

Optional note:

Versioning is useful later, but it is not necessary for the first setup.

---

### Step 7: Tags

Tags are optional for this small project, but they are useful once your AWS account has more resources.

For a beginner setup, either skip tags or add a few simple ones:

```text
Project = CalculatorSite
Environment = Test
Owner = Jeff
ManagedBy = Manual
Purpose = StaticWebsite
```

Do not put secrets, passwords, API keys, or sensitive personal information in tags. Tags are metadata and may be visible in places you do not expect.

---

### Step 8: Default encryption and Bucket Key

Use the default:

```text
Server-side encryption with Amazon S3 managed keys (SSE-S3)
```

You may also see a field named **Bucket Key** under encryption.

For this guide, if you are using the default S3-managed encryption option, leave encryption settings at their defaults.

S3 Bucket Keys mainly matter when using AWS KMS encryption, also called SSE-KMS. They can reduce AWS KMS request costs for KMS-encrypted objects. For this simple static website, the default S3-managed encryption setting is fine.

Practical recommendation:

```text
Default encryption: SSE-S3 / Amazon S3 managed keys
Bucket Key: leave default / not applicable unless using SSE-KMS
```

---

### Step 9: Object Lock

Leave Object Lock disabled.

---

### Step 10: Create bucket

1. Review the settings.
2. Choose **Create bucket**.

If AWS says the bucket name is already taken, use:

```text
jefsko-calculator-site-2026
```

If you change the bucket name, update every later command and policy that uses `jefsko-calculator-site`.

---


## 11. Shared setup: Upload the website files

### Step 1: Open the bucket

1. In S3, choose **Buckets**.
2. Choose:

```text
jefsko-calculator-site
```

---

### Step 2: Upload files

You can upload the files either by choosing **Add files** or by dragging and dropping the files into the upload area.

1. Choose **Upload**.
2. Choose **Add files**, or drag and drop the files into the upload page.
3. Select:

```text
index.html
style.css
calculator.js
```

4. Choose **Upload**.
5. Wait for the upload to complete.
6. Return to the bucket.

---

### Step 3: Confirm the files are in the root

You should see:

```text
index.html
style.css
calculator.js
```

They should not be inside a folder.

Do not upload them like this:

```text
calculator-site/index.html
calculator-site/style.css
calculator-site/calculator.js
```

For this guide, they should be directly in the bucket root.

---


## 12. Track A: Enable S3 static website hosting

### Step 1: Open the bucket Properties tab

1. Open the bucket:

```text
jefsko-calculator-site
```

2. Choose the **Properties** tab.
3. Scroll down to **Static website hosting**.
4. Choose **Edit**.

---

### Step 2: Enable website hosting

1. For **Static website hosting**, choose:

```text
Enable
```

2. For **Hosting type**, choose:

```text
Host a static website
```

3. For **Index document**, enter:

```text
index.html
```

4. For **Error document**, enter:

```text
index.html
```

For this simple calculator site, using `index.html` as the error document is fine.

Later, you can create a real error page:

```text
error.html
```

5. Choose **Save changes**.

---

### Step 3: Copy the S3 website endpoint

After saving, the **Static website hosting** section should show a **Bucket website endpoint**.

It will look similar to:

```text
http://jefsko-calculator-site.s3-website-us-west-2.amazonaws.com
```

or:

```text
http://jefsko-calculator-site.s3-website.us-west-2.amazonaws.com
```

Use the exact endpoint AWS gives you.

At this point, the endpoint may still show an access error. That is expected until the bucket is made public.

---

### What does static website hosting mean?

S3 static website hosting is a special S3 feature that makes a bucket behave like a simple website server.

When enabled, S3 provides a website endpoint and knows which file to serve as the home page, such as:

```text
index.html
```

When disabled, S3 is just object storage. The files still exist, but S3 does not expose a website endpoint for the bucket.

Track A uses S3 static website hosting.

Track B does not need S3 static website hosting for the final recommended setup because CloudFront can read from the regular S3 bucket origin using Origin Access Control.

For the clean final recommended setup, disable S3 Static website hosting after CloudFront is working unless you intentionally need S3 website endpoint features such as S3 website redirects or S3 folder-style index behavior.

Disabling S3 Static website hosting does **not** delete your files and does **not** disable CloudFront. It only disables the S3 website endpoint behavior. CloudFront can still serve the same static files from the normal private S3 bucket origin.

---

## 13. Track A: Make the bucket publicly readable

Warning:

This makes the files in this bucket readable by anyone on the internet.

That is okay only if the bucket contains public website files:

```text
index.html
style.css
calculator.js
```

Do not put private files, secrets, keys, passwords, backups, or personal documents in this bucket.

---

### Step 1: Edit Block Public Access

1. Open the bucket:

```text
jefsko-calculator-site
```

2. Choose the **Permissions** tab.
3. Find **Block public access (bucket settings)**.
4. Choose **Edit**.
5. Turn off:

```text
Block all public access
```

6. AWS will show a warning.
7. Confirm that you understand this bucket can become public.
8. Save changes.

---

### Step 2: Add a bucket policy

1. Stay on the **Permissions** tab.
2. Find **Bucket policy**.
3. Choose **Edit**.
4. Paste this policy.

Important: this example uses your bucket name in an ARN. The object ARN pattern is:

```text
arn:aws:s3:::jefsko-calculator-site/*
```

The bucket ARN is:

```text
arn:aws:s3:::jefsko-calculator-site
```

The object ARN pattern adds `/*` to mean “all objects inside this bucket.”

Paste this policy:

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

5. Choose **Save changes**.

This allows anyone to read the files. It does not allow people to upload, edit, or delete files.

---


## 14. Track A: Test the S3 website endpoint

1. Go back to the bucket **Properties** tab.
2. Scroll to **Static website hosting**.
3. Copy the **Bucket website endpoint**.
4. Open it in your browser.

Example format:

```text
http://jefsko-calculator-site.s3-website-us-west-2.amazonaws.com
```

Use the real endpoint from AWS.

You should see your calculator site.

---


## 15. Track B: Request an ACM certificate for CloudFront

Track B is the recommended final setup.

Important:

> Even though your S3 bucket is in `us-west-2`, the CloudFront certificate must be requested in `us-east-1`.

### Why this Track B step uses `us-east-1`

This certificate is not for the S3 bucket itself. It is for the browser-to-CloudFront HTTPS connection.

That is why this guide deliberately uses two AWS regions:

```text
S3 bucket:
  us-west-2
  Good West Coast bucket location for the source files.

ACM certificate for CloudFront:
  us-east-1
  Required CloudFront viewer-certificate location.
```

If you accidentally request the certificate in `us-west-2`, do not panic. Nothing is ruined. Request a new certificate in `us-east-1`, validate that one in Cloudflare, and use the `us-east-1` certificate with CloudFront.

You can leave the incorrect regional certificate alone temporarily, but to avoid confusion later, delete it after the correct CloudFront certificate is issued and working.


### Recommended certificate names

For this project, request a certificate for:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

---

### Step 1: Open AWS Certificate Manager

1. In the AWS Console, use the Region selector in the top-right.
2. Change the region to:

```text
US East (N. Virginia) us-east-1
```

3. In the top AWS search box, search for:

```text
Certificate Manager
```

4. Open **AWS Certificate Manager**.

---

### Step 2: Request a public certificate

1. Choose **Request**.
2. Choose:

```text
Request a public certificate
```

3. Choose **Next**.

---

### Step 3: Add domain names

In **Fully qualified domain name**, enter:

```text
calculator.jeffskone.com
```

Then choose **Add another name to this certificate**.

Add:

```text
calc.jeffskone.com
```

Final list:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

Optional wildcard version, if you decide you want future first-level subdomains under `calc`:

```text
calculator.jeffskone.com
calc.jeffskone.com
*.calc.jeffskone.com
```

Use the simpler exact-name certificate unless you already know you need wildcard subdomains.

---

### Step 4: Choose validation method

Choose:

```text
DNS validation - recommended
```

DNS validation works well because Cloudflare manages your DNS.

---

### Step 5: Key algorithm

Use the default unless you have a reason to change it.

A common default is:

```text
RSA 2048
```

---

### Step 6: Request the certificate

1. Review the request.
2. Choose **Request**.

The certificate will probably show:

```text
Pending validation
```

That is expected.

---


## 16. Track B: Validate the ACM certificate using Cloudflare DNS

ACM will give you DNS records to create.

These are usually CNAME records with names that start with an underscore.

Example only:

```text
Name:
_abc123.calculator.jeffskone.com

Type:
CNAME

Value:
_xyz987.acm-validations.aws
```

Do not use that exact example. Use the exact values ACM gives you.

---

### Step 1: Open the certificate in ACM

1. In AWS Certificate Manager, open the certificate you just requested.
2. Look for the domain validation section.
3. For each domain, find the CNAME record name and CNAME value.

You need:

```text
CNAME name
CNAME value
```

---

### Step 2: Open Cloudflare DNS

1. Sign in to Cloudflare.
2. Choose the domain:

```text
jeffskone.com
```

3. Go to:

```text
DNS > Records
```

---

### Step 3: Add the ACM validation CNAME record

For each ACM validation record:

1. Choose **Add record**.
2. Type:

```text
CNAME
```

3. Name:

```text
<the exact CNAME name from ACM>
```

Cloudflare may automatically shorten the displayed name. That is okay if the full hostname is still correct.

4. Target:

```text
<the exact CNAME value from ACM>
```

5. Proxy status:

```text
DNS only
```

This is important. ACM validation records should not be proxied through Cloudflare.

6. TTL:

```text
Auto
```

7. Choose **Save**.

Repeat for each validation record ACM provides.

---

### Step 4: Wait for validation

Go back to AWS Certificate Manager.

The status should eventually change from:

```text
Pending validation
```

to:

```text
Issued
```

This can take a few minutes. Sometimes it takes longer.

Do not continue with CloudFront custom domains until the certificate is issued.

---


## 17. Track B: Create the CloudFront distribution

For the recommended setup, CloudFront should use the regular S3 bucket origin with Origin Access Control.

This means:

- CloudFront can read the bucket.
- The public internet cannot directly read the bucket.
- Visitors access the site through CloudFront, not directly through S3.

This is better than using the S3 website endpoint as the CloudFront origin.

Important:

For Track B, CloudFront does **not** need S3 static website hosting. It uses the regular S3 bucket origin.

---

### Step 1: Open CloudFront

1. In the AWS Console search box, search for:

```text
CloudFront
```

2. Open **CloudFront**.
3. Choose **Create distribution**.

This starts the distribution creation wizard. You are not finished yet. Later, after you review all settings, you will choose **Create distribution** again to submit the configuration.

---

### Step 2: Enter distribution name and description

If CloudFront asks for a distribution name, use a short descriptive name.

Recommended value:

```text
calculator-site
```

Other reasonable options:

```text
calculator-site-prod
jefsko-calculator-site
calculator-jeffskone-com
```

The distribution name is mostly for your own organization in the AWS Console. It does not have to match the domain name, but it should be easy to recognize later.

If CloudFront asks for a description, use something like:

```text
Static calculator website for calculator.jeffskone.com and calc.jeffskone.com
```

The description is optional, but useful. Use it to remind yourself what the distribution serves and which domains point to it.

---

### Step 3: Choose distribution type or app type

If the console asks what kind of distribution or app you are creating, choose something like:

```text
Single website or app
```

This is the normal choice for a simple static calculator site.

---

### Step 4: Choose the origin

For the origin:

1. Choose origin type:

```text
Amazon S3
```

2. Choose or browse for the bucket:

```text
jefsko-calculator-site
```

3. Use the S3 bucket origin, not the S3 website endpoint.

Good origin style:

```text
jefsko-calculator-site.s3.us-west-2.amazonaws.com
```

Avoid using the S3 website endpoint as the recommended final origin:

```text
jefsko-calculator-site.s3-website-us-west-2.amazonaws.com
```

Why?

The regular S3 bucket origin lets CloudFront use Origin Access Control. That allows CloudFront to read the bucket while keeping the bucket private from direct public access.

The S3 website endpoint is public and HTTP-only. It must be configured as a custom origin and cannot use Origin Access Control.

---

### Step 5: Configure Origin Access Control

The CloudFront console may present security settings during distribution creation. Depending on the current UI, you may see wording such as:

```text
Enable security
Origin access
Origin Access Control
OAC
Use recommended origin settings
Sign requests
```

For an S3 bucket origin, choose the option that lets CloudFront securely access the S3 bucket while keeping the bucket private.

Recommended values:

```text
Origin access: Origin access control settings / OAC
Signing behavior: Sign requests / Sign requests (recommended)
```

If CloudFront offers to update the S3 bucket policy automatically, allow it. If it does not, this guide includes a manual bucket policy later.

Do not confuse OAC with AWS WAF security protections:

```text
OAC = how CloudFront privately accesses S3.
WAF = optional web application firewall rules in front of CloudFront.
```

For this guide:

```text
Use OAC.
Skip WAF initially unless you have a specific reason to enable it.
```

WAF can be useful, but it adds cost, complexity, and more settings. For a tiny static calculator site, it is reasonable to skip WAF initially and add it later if needed.

---

### Step 6: Configure behavior settings

Some behavior settings may not be fully visible on the **Behaviors** tab until you edit the behavior.

After creating or editing the distribution, you can check them here:

```text
CloudFront > Distributions > select distribution > Behaviors > select default behavior > Edit
```

Recommended behavior values:

```text
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed HTTP methods: GET, HEAD
Cache policy: CachingOptimized or the current AWS managed static-content default
```

What these mean:

- **Viewer protocol policy** controls whether visitors can use HTTP, HTTPS, or whether HTTP automatically redirects to HTTPS.
- **Allowed HTTP methods** controls which request methods CloudFront accepts. For a static calculator site, only `GET` and `HEAD` are needed.
- **Cache policy** controls how CloudFront caches files. Static files are a good fit for caching.

If you are changing files frequently during early testing, caching can make updates look delayed. That is normal. You can invalidate the cache after uploading updates.

---

### Step 7: Choose TLS / security policy settings

If CloudFront asks for a TLS or security policy, choose the newest AWS-recommended TLS 1.2-or-newer policy shown in the console, unless you have a specific need to support very old browsers.

Beginner recommendation:

```text
Security policy / TLS policy: TLSv1.2_2021 or the current AWS recommended TLSv1.2+ policy
Supported HTTP versions: HTTP/2 enabled if available
SSL support method: SNI only
```

Choose **SNI** if CloudFront asks how to serve HTTPS. SNI is the normal modern option and is recommended for most sites. Dedicated IP SSL is generally unnecessary and can add cost.

---

### Step 8: Set the default root object

Set:

```text
index.html
```

Important:

Do not include a leading slash.

Correct:

```text
index.html
```

Incorrect:

```text
/index.html
```

---

### Step 9: Add Alternate domain name (CNAME) values if available

If the create-distribution flow lets you add custom domains now, look for the AWS UI field named:

```text
Alternate domain name (CNAME)
```

Add:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

Then choose the issued ACM certificate from `us-east-1`.

AWS often calls these **alternate domain names** or **CNAMEs**. Casually, they are your custom domain names.

If the create flow does not ask for alternate domain names yet, create the distribution first and add them afterward in the distribution settings. Both flows are fine.

---

### Step 10: Review and finish creating the distribution

1. Review the settings.
2. Choose **Create distribution**.
3. Wait for deployment.

When you choose **Create distribution**, CloudFront creates the distribution configuration and begins deploying it globally. The distribution may show a status like:

```text
Deploying
```

CloudFront deployment can take several minutes. Sometimes it is quick; sometimes it takes longer. Wait until the distribution is deployed before assuming final behavior is ready.

CloudFront will give you a distribution domain name like:

```text
d111111abcdef8.cloudfront.net
```

Copy it. You will need it in Cloudflare.

Do not use the example value above. Use your real CloudFront distribution domain name.

---

### Step 11: Test the CloudFront distribution domain

After the distribution finishes deploying, test:

```text
https://<DISTRIBUTION_DOMAIN>
```

Example only:

```text
https://d111111abcdef8.cloudfront.net
```

You should see the calculator site.

If the root URL does not work, try:

```text
https://<DISTRIBUTION_DOMAIN>/index.html
```

If `/index.html` works but `/` does not, check the CloudFront default root object.

Yes, by default the CloudFront distribution domain usually works directly. That is normal. It is useful for testing before DNS is ready. If you want only your custom hostnames to work, see the optional advanced section on restricting the default CloudFront domain.

---

## 18. Track B: Make S3 private again and allow only CloudFront

If you completed Track A, your bucket is currently public.

For the recommended Track B final setup, change it so:

```text
Public internet → CloudFront → S3
```

not:

```text
Public internet → S3
```

### Before and after

Before Track B cleanup, your temporary Track A setup may look like this:

```text
Block all public access: Off
Bucket policy: includes Principal "*" public-read statement
S3 website endpoint: works
CloudFront URL: may work
Custom domain: may or may not work yet
```

After Track B cleanup, the recommended final setup should look like this:

```text
Block all public access: On
Bucket policy: no Principal "*" public-read statement
Bucket policy: may include CloudFront OAC allow statement
S3 website endpoint: should not work publicly
CloudFront distribution domain: works unless separately blocked
Custom domains: work
```

### Step 1: Remove the public-read bucket policy statement

1. Open S3.
2. Open the bucket:

```text
jefsko-calculator-site
```

3. Go to **Permissions**.
4. Find **Bucket policy**.
5. Choose **Edit**.
6. Remove the entire public-read statement, not just one line.

The public-read statement looks like this:

```json
{
  "Sid": "PublicReadForStaticWebsiteFiles",
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::jefsko-calculator-site/*"
}
```

If that statement is inside a larger policy, remove the whole statement object and make sure the remaining JSON is still valid.

If that was the only statement in the policy, the final policy may either be blank or replaced by the CloudFront OAC policy shown below.

Important:

The final bucket policy should not contain a public-read statement with:

```json
"Principal": "*"
```

for `s3:GetObject` access to your website files.

### Step 2: Keep or add the CloudFront OAC allow statement

The final bucket policy might be blank only if CloudFront/AWS is managing access another way or if no policy is currently needed.

In the typical OAC setup, the bucket policy is not blank. It contains a statement that allows the CloudFront service principal to read objects from this bucket for this specific distribution.

If CloudFront already added an OAC policy, leave that policy in place.

If CloudFront did not automatically update the bucket policy, add a policy like this.

Replace:

```text
<ACCOUNT_ID>
<DISTRIBUTION_ID>
```

with your real AWS account ID and CloudFront distribution ID.

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

Example only:

```text
arn:aws:cloudfront::123456789012:distribution/E1ABCDEF234567
```

Do not use the example values.

### Step 3: Turn Block Public Access back on

1. Stay on the S3 bucket **Permissions** tab.
2. Find **Block public access (bucket settings)**.
3. Choose **Edit**.
4. Turn on:

```text
Block all public access
```

5. Save changes.

The old S3 website endpoint should stop working publicly. That is expected.

The CloudFront URL and custom domains should keep working because CloudFront is allowed through OAC.

### Common point of confusion

Turning Block Public Access back on does not automatically mean the CloudFront custom domain stops working. That is the goal. CloudFront should still work because it is allowed through OAC.

What should stop working is direct public access to S3.

If the S3 website endpoint still works after Track B cleanup, check whether the public-read bucket policy statement is still present.

---

## 19. Track B: Add custom domains to CloudFront

If you did not add custom domains during distribution creation, add them now.

### Step 1: Open the distribution

1. Open **CloudFront**.
2. Choose your distribution.
3. Go to the distribution settings.

---

### Step 2: Edit Alternate domain name (CNAME) values

Look for:

```text
Alternate domain name (CNAME)
Alternate domain names
CNAMEs
Custom domain names
```

Add:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

CloudFront often refers to these as **alternate domain names**, but people commonly call them CNAMEs or aliases.

---

### Step 3: Choose the ACM certificate

Choose the ACM certificate that covers:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

If you do not see the certificate:

1. Confirm the certificate status is:

```text
Issued
```

2. Confirm the certificate is in:

```text
us-east-1
```

3. Confirm the certificate includes the exact domain names.

---

### Step 4: Save changes

Save the CloudFront distribution changes.

Wait for the distribution to deploy.

---


## 20. Track B: Point Cloudflare DNS to CloudFront

Now create DNS records in Cloudflare.

### Step 1: Open Cloudflare DNS

1. Sign in to Cloudflare.
2. Choose:

```text
jeffskone.com
```

3. Go to:

```text
DNS > Records
```

---

### Step 2: Create the primary CNAME record

Choose **Add record**.

Use:

```text
Type: CNAME
Name: calculator
Target: <your-cloudfront-domain-name>
Proxy status: DNS only
TTL: Auto
```

Example only:

```text
Type: CNAME
Name: calculator
Target: d111111abcdef8.cloudfront.net
Proxy status: DNS only
TTL: Auto
```

Do not use the example CloudFront value.

---

### Step 3: Create the short alias CNAME record

Choose **Add record** again.

Use:

```text
Type: CNAME
Name: calc
Target: <your-cloudfront-domain-name>
Proxy status: DNS only
TTL: Auto
```

Example only:

```text
Type: CNAME
Name: calc
Target: d111111abcdef8.cloudfront.net
Proxy status: DNS only
TTL: Auto
```

---

### Step 4: Why DNS only at first?

Use **DNS only** for the website CNAME records at first.

Reason:

- It keeps the setup simpler.
- CloudFront handles HTTPS.
- CloudFront handles caching.
- Cloudflare only handles DNS.
- Troubleshooting is much easier.

Important distinction:

| Record type | Recommended proxy status |
|---|---|
| ACM certificate validation CNAME | DNS only |
| AWS domain ownership validation CNAME | DNS only |
| Website CNAME pointing to CloudFront | DNS only at first; proxying is optional later |

Later, you can experiment with Cloudflare proxying if you want. If you turn on the orange cloud, you are putting Cloudflare in front of CloudFront, which means you are stacking one CDN/proxy in front of another CDN. That can work, but it adds more moving parts.

If Cloudflare proxying is enabled later, the path becomes:

```text
Visitor → Cloudflare proxy → CloudFront → S3
```

If you proxy through Cloudflare later, use Cloudflare SSL/TLS mode:

```text
Full (strict)
```

and make sure CloudFront still has a valid certificate for the hostname.

### Optional: apex/root domain in Cloudflare

For this guide, the final hostnames are subdomains:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

If you later want the apex/root domain, such as:

```text
jeffskone.com
```

Cloudflare supports CNAME flattening, so you can usually create a CNAME at the root using:

```text
Type: CNAME
Name: @
Target: <DISTRIBUTION_DOMAIN>
Proxy status: DNS only at first
TTL: Auto
```

Use the CloudFront distribution domain name as the target, not the S3 bucket name, not the S3 website endpoint, not the CloudFront distribution ID, and not CloudFront IP addresses.

---


## 21. Track B: Test the final HTTPS custom domains

Test these in a browser:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

Also test HTTP redirects:

```text
http://calculator.jeffskone.com
http://calc.jeffskone.com
```

They should redirect to HTTPS.

Optional command-line tests:

```bash
curl -I https://calculator.jeffskone.com
curl -I https://calc.jeffskone.com
```

You want to see a successful HTTP status such as:

```text
HTTP/2 200
```

or possibly:

```text
HTTP/2 304
```

if the browser/cache already has the file.

To check DNS:

```bash
nslookup calculator.jeffskone.com
nslookup calc.jeffskone.com
```

You should see the names resolving toward CloudFront.

---


## 22. Final access expectations and URL behavior

For the final recommended Track B setup, your intended public URLs are:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

Use this matrix to understand what should and should not work:

| URL | Should work in final recommended Track B? | Notes |
|---|---:|---|
| `https://calculator.jeffskone.com` | Yes | Primary final URL. |
| `https://calc.jeffskone.com` | Yes | Short alias final URL. |
| `https://<DISTRIBUTION_DOMAIN>` | Yes by default | CloudFront provides this domain for every distribution. You can optionally block it with a CloudFront Function. |
| `http://<bucket>.s3-website-us-west-2.amazonaws.com` | No | For the recommended final setup, S3 Static website hosting should be disabled or direct website endpoint access should be blocked. |
| `https://<bucket>.s3.us-west-2.amazonaws.com/index.html` | No | Direct S3 object URLs should not be publicly readable when the bucket is private. |
| CloudFront to S3 through OAC | Yes | CloudFront should be the authorized reader of the private bucket. |

For the final recommended setup, the S3 website endpoint should not be the public site anymore.

The CloudFront distribution domain may still work by default. That is normal. If you want only your custom domains to work, that requires an optional advanced restriction.

### Optional advanced hardening: restrict the default CloudFront domain

Beginner recommendation:

```text
Leave the CloudFront distribution domain working.
```

Reason:

- It is normal CloudFront behavior.
- It helps with testing.
- It is usually not a problem for a small static site.

If you have a strong reason to allow only these hostnames:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

and reject this hostname:

```text
d111111abcdef8.cloudfront.net
```

you can add optional Host-header logic with CloudFront Functions or AWS WAF.

| Option | Result | Recommendation |
|---|---|---|
| Leave it alone | Both custom domains and CloudFront distribution domain work | Best beginner default. |
| CloudFront Function | Reject requests unless Host is approved | Good optional advanced option. |
| AWS WAF rule | More policy-style control | Usually more than this small site needs. |

This guide does not make that restriction part of the main path because it adds complexity. The main security goal is to block direct public S3 access and use CloudFront as the public front door.

---

## 23. Access control deep dive: S3, CloudFront, and custom domains

This section explains the access-control model in more detail. It is especially useful if you are trying to answer questions like:

```text
Should S3 Static website hosting be enabled or disabled?
Should the S3 website endpoint work?
Should the CloudFront standard domain work?
Should the bucket policy be blank?
Where do I set the default page?
How do I block direct CloudFront-domain access if I want only my custom domain to work?
```

### Terminology: four different URLs or access paths

There are several similar-sounding access paths. Keep them separate:

| Name | Example | What it means |
|---|---|---|
| Custom domain | `https://calculator.jeffskone.com` | The final public URL you want visitors to use. |
| CloudFront standard domain | `https://d111111abcdef8.cloudfront.net/` | The default `*.cloudfront.net` hostname AWS gives every distribution. |
| S3 website endpoint | `http://jefsko-calculator-site.s3-website-us-west-2.amazonaws.com/` | The public website endpoint created when S3 Static website hosting is enabled. |
| Direct S3 object URL | `https://jefsko-calculator-site.s3.us-west-2.amazonaws.com/index.html` | A direct request to an S3 object through the normal S3 REST endpoint. |

When the goal is “block direct S3 access,” you usually mean both of these should not be publicly usable:

- S3 website endpoint.
- Direct S3 object URL.

CloudFront should be the only authorized public path to the objects.

### Recommended final state

For the secure, modern CloudFront + private S3 setup, use this final state:

| Area | Recommended state |
|---|---|
| S3 Static website hosting | Disabled, unless intentionally needed |
| S3 Block Public Access | Enabled |
| Public-read bucket policy | Removed |
| OAC | Enabled |
| Bucket policy | Allows only the CloudFront distribution through OAC |
| CloudFront origin | Normal S3 bucket origin |
| CloudFront Default root object | `index.html` |
| Cloudflare DNS | Website CNAME points to CloudFront |
| CloudFront standard domain | Works by default; optionally blocked with a CloudFront Function |

Plain-English version:

```text
S3 stores the files privately.
CloudFront is the public web server/CDN.
OAC is CloudFront's permission pass to read the private S3 bucket.
Cloudflare DNS points the custom domain to CloudFront.
```

### S3 website endpoint vs. S3 bucket origin

An **S3 website endpoint** is created when S3 Static website hosting is enabled.

Example:

```text
http://jefsko-calculator-site.s3-website-us-west-2.amazonaws.com/
```

An **S3 bucket origin** is the normal S3 bucket endpoint CloudFront uses for private-bucket access.

Example:

```text
jefsko-calculator-site.s3.us-west-2.amazonaws.com
```

| Feature | S3 website endpoint | S3 bucket origin |
|---|---:|---:|
| Requires S3 Static website hosting | Yes | No |
| Supports direct S3 website index/error behavior | Yes | No |
| Supports direct HTTPS from S3 website endpoint | No | Not a website endpoint; CloudFront handles HTTPS |
| Can use CloudFront OAC/OAI | No | Yes |
| Bucket can stay private | No, not for public website endpoint access | Yes |
| Recommended for this guide's final setup | No | Yes |

Quick origin check:

```text
s3-website in the origin domain = S3 website endpoint
s3.<region>.amazonaws.com = normal S3 bucket origin
```

For this guide's recommended setup, choose the normal S3 bucket origin and OAC.

### OAC vs. OAI

OAC means **Origin Access Control**.

OAI means **Origin Access Identity**.

| Feature | OAC | OAI |
|---|---:|---:|
| Modern recommended method | Yes | No |
| Legacy method | No | Yes |
| Works with normal S3 bucket origin | Yes | Yes |
| Works with S3 website endpoint | No | No |
| Supports newer S3 capabilities better | Yes | Limited |
| Recommended for new setup | Yes | No |

Use OAC for new or cleaned-up setups.

### How to confirm or fix the CloudFront origin

For an existing distribution:

1. Go to **AWS Console → CloudFront**.
2. Choose **Distributions**.
3. Open the distribution.
4. Go to **Origins**.
5. Select the S3 origin.
6. Choose **Edit**.
7. Confirm the **Origin domain** is a normal S3 bucket origin, not an `s3-website` endpoint.
8. Under **Origin access**, choose **Origin access control settings** or the closest current UI label.
9. Select an existing OAC or create a new one.
10. Use signing behavior similar to:

```text
Sign requests / Sign requests (recommended)
```

11. Save changes.
12. Update the S3 bucket policy with the OAC allow statement.
13. Wait for CloudFront deployment to complete.

To create an OAC separately:

1. Go to **AWS Console → CloudFront**.
2. In the left navigation, choose **Origin access**.
3. Choose **Create control setting**.
4. Name it clearly, for example:

```text
oac-calculator-site-s3
```

5. Set origin type to:

```text
S3
```

6. Keep signing behavior as:

```text
Sign requests (recommended)
```

7. Create the OAC.
8. Attach it to the S3 origin in the CloudFront distribution.
9. Apply the corresponding S3 bucket policy.

### Bucket policy: blank or not?

For the recommended OAC setup, the bucket policy usually should **not** be blank.

It should contain a policy that allows only the specific CloudFront distribution to read objects.

Example read-only OAC bucket policy:

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

with your actual AWS account ID and CloudFront distribution ID.

For a read-only static site, `s3:GetObject` is usually sufficient.

A blank bucket policy may work only if another mechanism is granting access, such as public access, ACLs, or a legacy OAI policy. For a clean modern OAC setup, the policy should explicitly allow CloudFront and should not allow public access.

### CloudFront Default root object vs. S3 index document

When S3 Static website hosting is disabled, configure the default page in CloudFront.

CloudFront calls this the **Default root object**.

This controls what CloudFront returns when someone requests the root path:

```text
https://calculator.jeffskone.com/
```

Recommended value:

```text
index.html
```

To set it:

1. Go to **AWS Console → CloudFront**.
2. Choose **Distributions**.
3. Open the distribution.
4. Go to the **General** tab.
5. Choose **Edit**.
6. Find **Default root object**.
7. Enter:

```text
index.html
```

8. Save changes and wait for deployment.

You can use a different default file if you really want to, such as:

```text
home.html
main.html
app.html
```

But if you do, the file must actually exist in S3 and the guide examples should be adjusted. For this guide, keep `index.html`.

Important difference:

| Request | CloudFront Default root object behavior | S3 website index document behavior |
|---|---|---|
| `/` | Can return `/index.html` | Can return `/index.html` |
| `/about/` | Does not automatically return `/about/index.html` by default | Can return `/about/index.html` if that object exists |
| `/docs/` | Does not automatically return `/docs/index.html` by default | Can return `/docs/index.html` if that object exists |

CloudFront Default root object handles the distribution root. It does not automatically reproduce all S3 website folder-index behavior.

### Optional: clean subdirectory URL rewrite

If your site uses clean URLs such as:

```text
/about/
/docs/
/contact/
```

and you want those to map to:

```text
/about/index.html
/docs/index.html
/contact/index.html
```

you can use a CloudFront Function on the **viewer request** event.

For the simple calculator site in this guide, you probably do not need this. You only need it if the site has subdirectory `index.html` files or single-page-app routing needs.

Example rewrite-only CloudFront Function:

```javascript
function handler(event) {
    var request = event.request;
    var uri = request.uri;

    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }

    return request;
}
```

### CloudFront custom error responses

When S3 Static website hosting is disabled, S3 website error document settings no longer apply. Use CloudFront **Custom error responses** instead.

For a normal static site with a custom 404 page:

1. Upload `404.html` to the S3 bucket.
2. Go to **AWS Console → CloudFront**.
3. Choose **Distributions**.
4. Open the distribution.
5. Go to **Error Pages**.
6. Choose **Create Custom Error Response**.
7. Set **HTTP error code** to:

```text
404: Not Found
```

8. Set **Customize error response** to **Yes**.
9. Set **Response page path** to:

```text
/404.html
```

10. Set **HTTP response code** to:

```text
404: Not Found
```

11. Set **Error caching minimum TTL** to a modest value, such as:

```text
10
```

or:

```text
60
```

12. Create/save the custom error response.

For a single-page app where paths like `/dashboard` should return `index.html`, create custom error responses for both `403` and `404`:

```text
HTTP error code: 403
Customize error response: Yes
Response page path: /index.html
HTTP response code: 200
Error caching minimum TTL: 0 or 10
```

Repeat for:

```text
HTTP error code: 404
```

Why include `403`? With private S3 origins, missing objects can sometimes surface as `403 Access Denied` rather than `404 Not Found`, especially if the requester does not have permission to list the bucket. Handling both `403` and `404` is common for SPA fallback behavior.

### CloudFront standard domain: should it be blocked?

By default, this usually works:

```text
https://<DISTRIBUTION_DOMAIN>/
```

Example:

```text
https://d111111abcdef8.cloudfront.net/
```

That does not mean S3 is public. It means CloudFront is serving the site through its standard AWS-provided hostname.

Blocking the CloudFront standard domain is optional.

| Question | Answer |
|---|---|
| Is blocking the CloudFront standard domain required? | No. |
| Is it standard for small static sites? | Usually no. |
| Is it sometimes done? | Yes, when you want only approved custom hostnames to work. |
| Does the ACM certificate block the standard domain? | No. |
| Does pointing DNS to CloudFront block the standard domain? | No. |

Reasons someone might block it:

- Branding: users should only see the custom domain.
- Canonical URL control: avoid duplicate access paths.
- Cleanup/security preference: reduce unexpected public hostnames.
- Policy requirement: only approved hostnames should work.

For this guide, leave it alone unless you specifically want the extra restriction.

### CloudFront Function: block the standard domain only

Use this if you want only the custom hostnames to work.

Replace the example hostnames with your real allowed custom domains.

```javascript
function handler(event) {
    var request = event.request;
    var host = request.headers.host.value.toLowerCase();

    var allowedHosts = {
        "calculator.jeffskone.com": true,
        "calc.jeffskone.com": true
    };

    if (!allowedHosts[host]) {
        return {
            statusCode: 403,
            statusDescription: "Forbidden"
        };
    }

    return request;
}
```

Expected behavior:

```text
https://calculator.jeffskone.com/      → works
https://calc.jeffskone.com/            → works
https://d111111abcdef8.cloudfront.net/ → 403 Forbidden
```

### CloudFront Function: block standard domain and rewrite clean URLs

Use this if both are needed:

1. Block direct use of the CloudFront standard domain.
2. Rewrite `/about/` to `/about/index.html`.
3. Rewrite `/about` to `/about/index.html`.

```javascript
function handler(event) {
    var request = event.request;
    var host = request.headers.host.value.toLowerCase();

    var allowedHosts = {
        "calculator.jeffskone.com": true,
        "calc.jeffskone.com": true
    };

    // Block direct access through the CloudFront standard domain.
    if (!allowedHosts[host]) {
        return {
            statusCode: 403,
            statusDescription: "Forbidden"
        };
    }

    // Optional clean-URL rewrite:
    // /about/ -> /about/index.html
    // /about  -> /about/index.html
    var uri = request.uri;

    if (uri.endsWith('/')) {
        request.uri += 'index.html';
    } else if (!uri.includes('.')) {
        request.uri += '/index.html';
    }

    return request;
}
```

Do not attach separate competing viewer-request functions for the same behavior. If you need both host blocking and clean-URL rewriting, combine the logic into one CloudFront Function.

### Step-by-step: create and associate the CloudFront Function

1. Identify the allowed hostnames:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

2. Go to **AWS Console → CloudFront**.
3. In the left navigation, choose **Functions**.
4. Choose **Create function**.
5. Give it a clear name, such as:

```text
allow-only-calculator-custom-domains
```

6. Paste the function code.
7. Save the function.
8. Test the function in the CloudFront console if the UI provides a test option.
9. Publish the function.
10. Open the CloudFront distribution.
11. Go to **Behaviors**.
12. Select the default behavior.
13. Choose **Edit**.
14. Find **Function associations**.
15. Associate the function with:

```text
Viewer request
```

16. Save changes.
17. Wait for deployment.
18. Test both custom domains and the CloudFront standard domain.

### Verification commands

Replace example values with your real values.

Test the custom domains:

```bash
curl -I https://calculator.jeffskone.com/
curl -I https://calc.jeffskone.com/
```

Expected:

```text
HTTP/2 200
```

Test the CloudFront standard domain:

```bash
curl -I https://<DISTRIBUTION_DOMAIN>/
```

Expected if not blocked:

```text
HTTP/2 200
```

Expected if blocked with a CloudFront Function:

```text
HTTP/2 403
```

Test the S3 website endpoint:

```bash
curl -I http://jefsko-calculator-site.s3-website-us-west-2.amazonaws.com/
```

Expected for the final private setup:

```text
403 Forbidden
```

or another failure/disabled result, depending on whether S3 Static website hosting is disabled.

Test a direct S3 object URL:

```bash
curl -I https://jefsko-calculator-site.s3.us-west-2.amazonaws.com/index.html
```

Expected for the private bucket:

```text
403 Forbidden
```

Test a missing page:

```bash
curl -I https://calculator.jeffskone.com/does-not-exist
```

Expected depends on your choice:

- Normal static site: `404`.
- SPA fallback: `200` with `/index.html`.
- Private S3 without custom error mapping: possibly `403`.

### Common misunderstandings

| Misunderstanding | Correction |
|---|---|
| Static website means S3 Static website hosting must be enabled. | A site can be static while S3 Static website hosting is disabled. CloudFront can serve static files from a private S3 bucket origin. |
| Disabling S3 Static website hosting disables my CloudFront website. | It disables the S3 website endpoint behavior only. CloudFront can still serve the site from the normal S3 bucket origin. |
| Block Public Access alone controls everything. | Block Public Access prevents public S3 access, but CloudFront still needs permission through OAC and the bucket policy. |
| The bucket policy should always be blank. | For OAC, the bucket policy usually should contain a specific allow statement for the CloudFront distribution. It should not contain public-read access. |
| The ACM certificate blocks the CloudFront standard domain. | The ACM certificate enables HTTPS for the custom domain. It does not block `*.cloudfront.net` access. |
| DNS pointing to CloudFront means the CloudFront standard domain must remain usable. | DNS can point to CloudFront while a CloudFront Function blocks direct requests to the CloudFront standard hostname based on the `Host` header. |
| CloudFront Default root object behaves exactly like S3 website index document. | CloudFront Default root object handles `/`, but not subdirectories like `/about/`. Use a CloudFront Function if you need folder-style index behavior. |

---

## 24. Updating the website after setup

When you change your website files, upload them again.

### Console method

1. Open S3.
2. Open:

```text
jefsko-calculator-site
```

3. Upload the changed files.
4. Overwrite the old files.

### AWS CLI method

From the local folder containing the website files:

```bash
aws s3 sync . s3://jefsko-calculator-site --exclude "*" --include "index.html" --include "style.css" --include "calculator.js"
```

After CloudFront is enabled, you may need to invalidate the cache.

Example:

```bash
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/*"
```

Example with a fake distribution ID:

```bash
aws cloudfront create-invalidation --distribution-id E1ABCDEF234567 --paths "/*"
```

Do not use the fake ID. Use your real CloudFront distribution ID.

For only one changed file:

```bash
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/calculator.js"
```

---


## 25. Troubleshooting

### Problem: S3 website endpoint says 403 Access Denied

Common causes:

1. Block Public Access is still on.
2. The public-read bucket policy is missing.
3. The bucket policy has the wrong bucket name.
4. `index.html` is missing.
5. Files were uploaded inside a folder instead of the root.

Check:

```text
S3 > bucket > Permissions > Block public access
S3 > bucket > Permissions > Bucket policy
S3 > bucket > Objects
S3 > bucket > Properties > Static website hosting
```

---

### Problem: S3 website endpoint says 404 Not Found

Common causes:

1. `index.html` was not uploaded.
2. The index document name is wrong.
3. The file has the wrong case, like `Index.html`.

Correct filename:

```text
index.html
```

---

### Problem: CSS or JavaScript does not load

Check your HTML references.

Good:

```html
<link rel="stylesheet" href="style.css">
<script src="calculator.js" defer></script>
```

Bad, unless you actually created folders:

```html
<link rel="stylesheet" href="/css/style.css">
<script src="/js/calculator.js"></script>
```

Also check capitalization.

---

### Problem: ACM certificate stays Pending validation

Check:

1. You created the CNAME records in Cloudflare.
2. The records are **DNS only**, not proxied.
3. The CNAME names and values exactly match ACM.
4. You are looking at ACM in `us-east-1`.
5. Cloudflare did not accidentally duplicate the domain name in the record.

For example, if ACM gives:

```text
_abc123.calculator.jeffskone.com
```

Cloudflare might want the name entered as:

```text
_abc123.calculator
```

because Cloudflare already knows the zone is:

```text
jeffskone.com
```

Look at the final displayed hostname in Cloudflare and make sure it is correct.

---


### Problem: I requested the ACM certificate in `us-west-2`

That is a common mistake.

For CloudFront viewer HTTPS, request the certificate again in:

```text
us-east-1
```

Then validate the new `us-east-1` certificate in Cloudflare and attach that certificate to the CloudFront distribution.

The `us-west-2` certificate is not harmful, but it will not be the CloudFront viewer certificate. After the correct certificate is issued and working, you can delete the extra regional certificate to keep ACM tidy.

### Problem: Certificate is issued but not visible in CloudFront

Check:

1. The certificate is in `us-east-1`.
2. The certificate is issued.
3. The certificate includes the CloudFront alternate domain names.
4. You are in the same AWS account.

---

### Problem: CloudFront URL works, custom domain does not

Check:

1. Cloudflare CNAME records point to the correct CloudFront distribution domain.
2. CloudFront has the custom names added as alternate domain names.
3. The certificate covers the custom names.
4. CloudFront distribution status is deployed.
5. DNS propagation has completed.

---

### Problem: CloudFront returns 403 Access Denied

Common causes:

1. S3 bucket policy does not allow CloudFront OAC.
2. The files are not in the bucket root.
3. The default root object is missing or wrong.
4. The OAC policy has the wrong distribution ID.
5. The object key is wrong due to capitalization.

Check:

```text
CloudFront > Distribution > Origins
CloudFront > Distribution > General > Default root object
S3 > Bucket > Permissions > Bucket policy
S3 > Bucket > Objects
```

---

### Problem: The S3 website endpoint stopped working after Track B

That is expected if you made the bucket private again.

The final public URL should be CloudFront/custom domain:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

not the S3 website endpoint.

---

### Problem: Updates do not appear immediately

CloudFront may be serving cached content.

Fix:

```bash
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/*"
```

For a tiny site, invalidating everything is simple and fine.

---


### Problem: I cannot find Viewer protocol policy or Allowed HTTP methods

These settings are part of a CloudFront behavior.

Try this path:

```text
CloudFront > Distributions > select distribution > Behaviors > select default behavior > Edit
```

Set:

```text
Viewer protocol policy: Redirect HTTP to HTTPS
Allowed HTTP methods: GET, HEAD
```

The Behaviors tab may show only a summary. To confirm the exact settings, select the default behavior and choose **Edit**.

---

### Problem: I cannot find Origin Access Control

Look for wording such as:

```text
Origin access
Origin Access Control
OAC
Origin access control settings
Use recommended origin settings
Sign requests
```

For this guide, the important idea is:

```text
Use a regular S3 bucket origin.
Use OAC.
Use Sign requests / Sign requests (recommended).
```

Do not use the S3 website endpoint as the origin if you want OAC.

---

### Problem: The S3 website endpoint still works after I turned Block Public Access back on

Check the bucket policy.

If the policy still contains this type of statement, direct public S3 access may still be allowed:

```json
{
  "Principal": "*",
  "Action": "s3:GetObject"
}
```

For final Track B, remove the entire public-read statement. Leave only the CloudFront OAC allow statement, if needed.

---

### Problem: The CloudFront distribution domain still works directly

That is normal.

CloudFront gives every distribution a domain like:

```text
d111111abcdef8.cloudfront.net
```

By default, that domain works. It is useful for testing before your custom DNS records are ready.

If you want only the custom hostnames to work, use optional advanced Host-header logic with CloudFront Functions or AWS WAF.

---


### Problem: I disabled S3 Static website hosting and the custom domain still works

That is expected if CloudFront is using the normal S3 bucket origin with OAC.

Disabling S3 Static website hosting disables the S3 website endpoint. It does not delete files and does not disable CloudFront.

Check:

```text
CloudFront > Distribution > Origins > Origin domain
```

If the origin looks like this, CloudFront is using the normal S3 bucket origin:

```text
jefsko-calculator-site.s3.us-west-2.amazonaws.com
```

That is the recommended final pattern.

---

### Problem: Direct S3 object URLs still work publicly

A direct S3 object URL looks like this:

```text
https://jefsko-calculator-site.s3.us-west-2.amazonaws.com/index.html
```

For the final recommended setup, this should not be publicly readable.

Check:

1. S3 Block Public Access is enabled.
2. The bucket policy does not contain a public `Principal: "*"` read statement.
3. ACLs are disabled or not granting public read.
4. The bucket policy contains only the CloudFront OAC allow statement, if needed.

---

### Problem: `/about/` does not load even though `/` works

CloudFront Default root object handles the distribution root:

```text
/
```

It does not automatically map every subdirectory to an `index.html` file.

If your site needs clean subdirectory URLs such as:

```text
/about/
/docs/
```

use a CloudFront Function to rewrite those paths to:

```text
/about/index.html
/docs/index.html
```

For the simple calculator site, this is usually not needed.

---

### Problem: Missing pages show `403` instead of `404`

With a private S3 bucket origin, missing objects can sometimes appear as `403 Access Denied` instead of `404 Not Found` because public callers are not allowed to list or inspect the bucket.

For a normal website, configure a CloudFront custom error response if you want a friendly `404.html` page.

For a single-page app, configure CloudFront custom error responses for both `403` and `404` to return `/index.html` with HTTP status `200`.

---

### Problem: I cannot decide whether to block the CloudFront standard domain

Blocking it is optional.

Leave it alone if:

- You want the simplest setup.
- You want an easy CloudFront testing URL.
- Duplicate access through `*.cloudfront.net` does not matter.

Block it with a CloudFront Function if:

- You only want approved custom hostnames to work.
- You want cleaner canonical-domain behavior.
- You are comfortable adding CloudFront Function logic.

---

## 26. Optional alternatives outside AWS

This guide uses AWS because the goal is to host on AWS.

That said, for a simple three-file static website, there are easier alternatives.

### Alternative 1: Cloudflare Pages

Cloudflare Pages can host static sites directly.

Pros:

- Very simple.
- Great fit if Cloudflare already manages your DNS.
- HTTPS is built in.
- Git integration is easy.

Cons:

- Not AWS.
- Different deployment workflow.
- Wildcard custom domain behavior may have limitations depending on the use case.

This is probably the strongest non-AWS alternative for this exact project.

### Alternative 2: GitHub Pages

Pros:

- Simple.
- Good for Git repos.
- Free for many public/static use cases.

Cons:

- Less AWS-like.
- Custom-domain and HTTPS setup is different.
- Not as flexible as CloudFront for CDN behavior.

### Alternative 3: Netlify or Vercel

Pros:

- Very easy static hosting.
- Git deployment.
- Built-in HTTPS.

Cons:

- Not AWS.
- More platform-specific behavior.
- May be more than you need for a tiny calculator site.

### Alternative 4: Cloudflare DNS wildcard plus AWS CloudFront

If you want many subdomains like:

```text
alpha.calc.jeffskone.com
beta.calc.jeffskone.com
demo.calc.jeffskone.com
```

you can combine:

```text
Cloudflare wildcard DNS record
CloudFront wildcard alternate domain name
ACM wildcard certificate
```

Example DNS:

```text
Type: CNAME
Name: *.calc
Target: <your-cloudfront-domain-name>
Proxy status: DNS only
```

Example CloudFront alternate domain name:

```text
*.calc.jeffskone.com
```

Example ACM certificate names:

```text
calc.jeffskone.com
*.calc.jeffskone.com
```

Use this only if you actually need wildcard subdomains.

For the calculator project, exact names are simpler.

### Alternative 5: Import an outside certificate into ACM

You can buy or obtain a certificate from another certificate authority and import it into ACM for CloudFront.

This is usually unnecessary for this guide.

Reasons to avoid it for now:

- ACM public certificates are free.
- ACM DNS validation is straightforward with Cloudflare.
- Imported certificates must be renewed and re-imported by you.
- Wildcard depth rules still apply to normal public TLS certificates.

Use ACM unless you have a specific reason not to.

---


## 27. Reference commands

These commands are optional. The AWS Console steps above are friendlier for beginners.

### Set variables

Linux/macOS/Git Bash:

```bash
BUCKET_NAME="jefsko-calculator-site"
AWS_REGION="us-west-2"
CERT_REGION="us-east-1"
PRIMARY_DOMAIN="calculator.jeffskone.com"
ALIAS_DOMAIN="calc.jeffskone.com"
```

PowerShell:

```powershell
$BUCKET_NAME = "jefsko-calculator-site"
$AWS_REGION = "us-west-2"
$CERT_REGION = "us-east-1"
$PRIMARY_DOMAIN = "calculator.jeffskone.com"
$ALIAS_DOMAIN = "calc.jeffskone.com"
```

Windows Command Prompt:

```cmd
set BUCKET_NAME=jefsko-calculator-site
set AWS_REGION=us-west-2
set CERT_REGION=us-east-1
set PRIMARY_DOMAIN=calculator.jeffskone.com
set ALIAS_DOMAIN=calc.jeffskone.com
```

---

### Create the S3 bucket

For `us-west-2`:

```bash
aws s3api create-bucket \
  --bucket jefsko-calculator-site \
  --region us-west-2 \
  --create-bucket-configuration LocationConstraint=us-west-2
```

PowerShell single-line version:

```powershell
aws s3api create-bucket --bucket jefsko-calculator-site --region us-west-2 --create-bucket-configuration LocationConstraint=us-west-2
```

---

### Upload the website files

```bash
aws s3 cp index.html s3://jefsko-calculator-site/index.html
aws s3 cp style.css s3://jefsko-calculator-site/style.css
aws s3 cp calculator.js s3://jefsko-calculator-site/calculator.js
```

Or:

```bash
aws s3 sync . s3://jefsko-calculator-site --exclude "*" --include "index.html" --include "style.css" --include "calculator.js"
```

---

### Request the recommended ACM certificate

Remember:

Use `us-east-1` for CloudFront certificates.

```bash
aws acm request-certificate \
  --region us-east-1 \
  --domain-name calculator.jeffskone.com \
  --subject-alternative-names calc.jeffskone.com \
  --validation-method DNS \
  --idempotency-token calcsite20260630 \
  --query CertificateArn \
  --output text
```

PowerShell single-line version:

```powershell
aws acm request-certificate --region us-east-1 --domain-name calculator.jeffskone.com --subject-alternative-names calc.jeffskone.com --validation-method DNS --idempotency-token calcsite20260630 --query CertificateArn --output text
```

Save the returned certificate ARN.

Example only:

```text
arn:aws:acm:us-east-1:123456789012:certificate/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee
```

---

### Request a wildcard certificate for `*.calc.jeffskone.com`

Only use this if you need subdomains like `alpha.calc.jeffskone.com`.

```bash
aws acm request-certificate \
  --region us-east-1 \
  --domain-name calc.jeffskone.com \
  --subject-alternative-names "*.calc.jeffskone.com" calculator.jeffskone.com \
  --validation-method DNS \
  --idempotency-token calcwild20260630 \
  --query CertificateArn \
  --output text
```

PowerShell single-line version:

```powershell
aws acm request-certificate --region us-east-1 --domain-name calc.jeffskone.com --subject-alternative-names "*.calc.jeffskone.com" calculator.jeffskone.com --validation-method DNS --idempotency-token calcwild20260630 --query CertificateArn --output text
```

---

### Get ACM DNS validation records

Replace `<CERTIFICATE_ARN>` with the real ARN:

```bash
aws acm describe-certificate \
  --region us-east-1 \
  --certificate-arn "<CERTIFICATE_ARN>" \
  --query "Certificate.DomainValidationOptions[].ResourceRecord" \
  --output table
```

PowerShell single-line version:

```powershell
aws acm describe-certificate --region us-east-1 --certificate-arn "<CERTIFICATE_ARN>" --query "Certificate.DomainValidationOptions[].ResourceRecord" --output table
```

---

### Check certificate status

```bash
aws acm describe-certificate \
  --region us-east-1 \
  --certificate-arn "<CERTIFICATE_ARN>" \
  --query "Certificate.Status" \
  --output text
```

Expected final result:

```text
ISSUED
```

---

### Invalidate CloudFront cache

```bash
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/*"
```

For one file:

```bash
aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/calculator.js"
```

---


## 28. Variables, ARNs, and placeholders reference

Use this table when replacing placeholders in commands, policies, or examples.

| Placeholder / value | Meaning | Example | Where to find it |
|---|---|---|---|
| `<BUCKET_NAME>` | Your S3 bucket name | `jefsko-calculator-site` | S3 > Buckets |
| `<BUCKET_ARN>` | ARN for the bucket itself | `arn:aws:s3:::jefsko-calculator-site` | S3 bucket Properties or construct from bucket name |
| `<OBJECTS_ARN>` | ARN pattern for all objects in the bucket | `arn:aws:s3:::jefsko-calculator-site/*` | Construct from bucket ARN by adding `/*` |
| `<AWS_REGION>` | S3 bucket region | `us-west-2` | S3 bucket Properties |
| `<CERT_REGION>` | ACM certificate region for CloudFront | `us-east-1` | AWS Console region selector in ACM |
| `<CERTIFICATE_ARN>` | ARN of ACM certificate | `arn:aws:acm:us-east-1:...` | ACM certificate details |
| `<DISTRIBUTION_ID>` | CloudFront distribution ID | `E1ABCDEF234567` | CloudFront distribution list/details |
| `<DISTRIBUTION_DOMAIN>` | CloudFront assigned domain | `d111111abcdef8.cloudfront.net` | CloudFront distribution details |
| `<ACCOUNT_ID>` | AWS account ID | `123456789012` | AWS account menu or `aws sts get-caller-identity` |
| `<PRIMARY_DOMAIN>` | Main custom domain | `calculator.jeffskone.com` | User-defined |
| `<ALIAS_DOMAIN>` | Short alias hostname | `calc.jeffskone.com` | User-defined |

### Bucket ARN vs. object ARN

The bucket ARN refers to the bucket itself:

```text
arn:aws:s3:::jefsko-calculator-site
```

The object ARN pattern refers to every object inside the bucket:

```text
arn:aws:s3:::jefsko-calculator-site/*
```

The `/*` means “all objects inside this bucket.”

When granting read access to website files, the object ARN pattern is usually the value you need.

---

## 29. Recommended AWS tags

Tags are optional for this small project, but they are a good habit.

Tags help you identify and organize resources later, especially when your AWS account starts to contain more buckets, distributions, certificates, or test resources.

Recommended simple tags:

| Key | Value |
|---|---|
| `Project` | `CalculatorSite` |
| `Environment` | `Production` or `Test` |
| `Owner` | `Jeff` |
| `ManagedBy` | `Manual` |
| `Purpose` | `StaticWebsite` |

You can use tags on resources such as:

- S3 buckets.
- CloudFront distributions.
- ACM certificates, if the console offers tag fields.

Do not put secrets, passwords, API keys, or sensitive personal information in tags. Tags are metadata and may be visible in places you do not expect.

---

## 30. AWS and Cloudflare console UI drift note

AWS and Cloudflare console labels may change over time.

If the exact field name differs, look for the closest equivalent. This guide tries to include the visible label, the AWS/Cloudflare concept name, and the reason for the setting.

| Concept | Possible UI labels |
|---|---|
| Alternate domain names | Alternate domain name (CNAME), CNAMEs, Custom domain names |
| OAC | Origin Access Control, Origin access, Origin access control settings, Use recommended origin settings |
| Viewer protocol policy | Redirect HTTP to HTTPS, Viewer protocol policy |
| WAF/security | Enable security, Security protections, Web Application Firewall |
| TLS policy | Security policy, TLS policy, Minimum TLS version |
| Distribution name | Distribution name, Name tag, Name |

When in doubt, match the concept and reason, not only the exact label.

---

## 31. Official references

AWS S3 static website hosting:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/HostingWebsiteOnS3Setup.html
```

AWS S3 website endpoints and HTTPS limitation:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteEndpoints.html
```

AWS S3 custom domain walkthrough:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/website-hosting-custom-domain-walkthrough.html
```

AWS S3 virtual-hosted bucket/CNAME behavior:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html
```

AWS S3 bucket naming rules:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucketnamingrules.html
```

AWS S3 Bucket Keys:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-key.html
```

AWS S3 Block Public Access:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html
```

AWS CloudFront create a distribution:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-web-creating-console.html
```

AWS CloudFront getting started with S3 and OAC:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.SimpleDistribution.html
```

AWS CloudFront testing a distribution:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/distribution-web-testing.html
```

AWS CloudFront alternate domain names:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/CNAMEs.html
```

AWS CloudFront wildcard alternate domain names:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/alternate-domain-names-wildcard.html
```

AWS CloudFront certificate requirements:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-requirements.html
```

AWS CloudFront configure alternate domain names and HTTPS:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-and-https-procedures.html
```

AWS CloudFront Origin Access Control for S3:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html
```

AWS CloudFront HTTPS from viewers:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https-viewers-to-cloudfront.html
```

AWS CloudFront cache behavior settings:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistValuesCacheBehavior.html
```

AWS CloudFront invalidations:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html
```

AWS CloudFront SNI vs. dedicated IP SSL:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cnames-https-dedicated-ip-or-sni.html
```

AWS CloudFront Functions:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-functions.html
```

AWS CloudFront Functions event structure:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-event-structure.html
```

AWS Certificate Manager overview:

```text
https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html
```

AWS Certificate Manager overview and CloudFront region behavior:

```text
https://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html#concepts-regions
```

AWS Certificate Manager best practices:

```text
https://docs.aws.amazon.com/acm/latest/userguide/acm-bestpractices.html
```

AWS wildcard certificate behavior:

```text
https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DomainNameFormat.html
```

AWS Tag Editor and resource tagging:

```text
https://docs.aws.amazon.com/tag-editor/latest/userguide/tagging.html
```

Cloudflare DNS record management:

```text
https://developers.cloudflare.com/dns/manage-dns-records/how-to/create-dns-records/
```

Cloudflare DNS proxy status:

```text
https://developers.cloudflare.com/dns/proxy-status/
```

Cloudflare wildcard DNS records:

```text
https://developers.cloudflare.com/dns/manage-dns-records/reference/wildcard-dns-records/
```

Cloudflare Universal SSL:

```text
https://developers.cloudflare.com/ssl/edge-certificates/universal-ssl/
```

Cloudflare advanced certificate wildcard coverage:

```text
https://developers.cloudflare.com/ssl/edge-certificates/advanced-certificate-manager/
```

Cloudflare Pages custom domains:

```text
https://developers.cloudflare.com/pages/configuration/custom-domains/
```

AWS S3 enabling website hosting:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/EnableWebsiteHosting.html
```

AWS S3 index document support:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/IndexDocumentSupport.html
```

AWS S3 website access permissions:

```text
https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteAccessPermissionsReqd.html
```

AWS CloudFront origins: S3 and custom origins:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html
```

AWS CloudFront default root object:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DefaultRootObject.html
```

AWS CloudFront custom error responses:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GeneratingCustomErrorResponses.html
```

AWS CloudFront custom error pages procedure:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/custom-error-pages-procedure.html
```

AWS CloudFront Function URL rewrite example:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/example_cloudfront_functions_url_rewrite_single_page_apps_section.html
```

Cloudflare CNAME flattening:

```text
https://developers.cloudflare.com/dns/cname-flattening/
```

Cloudflare CNAME/domain verification troubleshooting:

```text
https://developers.cloudflare.com/dns/manage-dns-records/troubleshooting/cname-domain-verification/
```


---

## 32. Changelog

### v1.2.0 - 2026-07-01

Additive access-control and CloudFront behavior update.

Added:

- Stronger final recommended access-control target state.
- Clearer terminology for S3 website endpoint, S3 bucket origin, direct S3 object URL, and CloudFront standard distribution domain.
- Expanded DNS guidance explaining that public website records should point to CloudFront, not S3.
- Clarification that ACM validation CNAMEs should be DNS-only, while website CNAME proxying is optional after initial testing.
- Guidance for Cloudflare apex/root CNAME flattening.
- Expanded S3 Static website hosting enable/disable guidance for the private CloudFront + OAC pattern.
- OAC vs. OAI explanation.
- Step-by-step guidance for confirming or creating a regular S3 bucket origin with OAC.
- CloudFront Default root object vs. S3 website index-document behavior.
- Optional CloudFront Function examples for host blocking and clean-URL rewriting.
- CloudFront custom error response guidance for normal 404 pages and SPA fallbacks.
- Additional verification commands for custom domains, CloudFront standard domain, S3 website endpoint, direct S3 object URL, and missing-page behavior.
- Additional troubleshooting entries.
- Additional official AWS and Cloudflare references.

Changed:

- Cleaned repeated horizontal-rule formatting from the previous guide version.
- Clarified that the clean final recommended setup disables S3 Static website hosting unless S3 website endpoint features are intentionally needed.

### v1.1.0 - 2026-07-01

Additive clarification update based on initial reader feedback.

Added:

- Clear “How to Use This Guide” section.
- Shared setup explanation for Track A and Track B.
- Table showing which topics apply to Track A, Track B, or both.
- Clarification that Track B needs the S3 bucket and uploaded files, but not the public S3 website endpoint steps.
- Expanded dashes vs. periods bucket naming guidance.
- AWS tag guidance.
- S3 Bucket Key explanation.
- Drag-and-drop upload option.
- Expanded static website hosting enable/disable explanation.
- Variables, placeholders, ARNs, and where to find values.
- Expanded CloudFront distribution name, description, origin, OAC, behavior, TLS, and deployment guidance.
- Clearer “Alternate domain name (CNAME)” terminology.
- Explicit public S3 bucket policy cleanup guidance.
- Final access expectations and URL behavior matrix.
- Optional advanced note about restricting the default CloudFront distribution domain.
- Additional troubleshooting entries.
- Additional official references.

### v1.0.0 - 2026-06-30

Initial complete version.

Added:

- Summary.
- Prerequisites.
- Track A and Track B explanation.
- Bucket naming guidance.
- S3 Console setup steps.
- Upload steps.
- Static website hosting steps.
- Public-read bucket policy.
- Testing steps.
- Troubleshooting.
- Optional AWS CLI upload commands.
- CloudFront, ACM, and Cloudflare setup.
- Wildcard certificate and subdomain-level guidance.
- Origin Access Control guidance.
- Official references.

---
