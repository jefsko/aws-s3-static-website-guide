# AWS S3 Static Website Quick Start

**Filename:** `aws-s3-static-website-quick-start.md`  
**Version:** `v1.0.0`  
**Last updated:** 2026-06-30  
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
2. [Recommended setup](#2-recommended-setup)
3. [Prerequisites](#3-prerequisites)
4. [Project file structure](#4-project-file-structure)
5. [Track A vs. Track B](#5-track-a-vs-track-b)
6. [Bucket naming rules and domain-name matching](#6-bucket-naming-rules-and-domain-name-matching)
7. [Custom domain options: S3-only vs. CloudFront](#7-custom-domain-options-s3-only-vs-cloudfront)
8. [Certificates, wildcard certificates, and subdomain levels](#8-certificates-wildcard-certificates-and-subdomain-levels)
9. [Track A: Create the S3 bucket](#9-track-a-create-the-s3-bucket)
10. [Track A: Upload the website files](#10-track-a-upload-the-website-files)
11. [Track A: Enable S3 static website hosting](#11-track-a-enable-s3-static-website-hosting)
12. [Track A: Make the bucket publicly readable](#12-track-a-make-the-bucket-publicly-readable)
13. [Track A: Test the S3 website endpoint](#13-track-a-test-the-s3-website-endpoint)
14. [Track B: Request an ACM certificate for CloudFront](#14-track-b-request-an-acm-certificate-for-cloudfront)
15. [Track B: Validate the ACM certificate using Cloudflare DNS](#15-track-b-validate-the-acm-certificate-using-cloudflare-dns)
16. [Track B: Create the CloudFront distribution](#16-track-b-create-the-cloudfront-distribution)
17. [Track B: Make S3 private again and allow only CloudFront](#17-track-b-make-s3-private-again-and-allow-only-cloudfront)
18. [Track B: Add custom domains to CloudFront](#18-track-b-add-custom-domains-to-cloudfront)
19. [Track B: Point Cloudflare DNS to CloudFront](#19-track-b-point-cloudflare-dns-to-cloudfront)
20. [Track B: Test the final HTTPS custom domains](#20-track-b-test-the-final-https-custom-domains)
21. [Updating the website after setup](#21-updating-the-website-after-setup)
22. [Troubleshooting](#22-troubleshooting)
23. [Optional alternatives outside AWS](#23-optional-alternatives-outside-aws)
24. [Reference commands](#24-reference-commands)
25. [Official references](#25-official-references)
26. [Changelog](#26-changelog)

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

## 2. Recommended setup

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

## 3. Prerequisites

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

## 4. Project file structure

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

## 5. Track A vs. Track B

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

## 6. Bucket naming rules and domain-name matching

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

## 7. Custom domain options: S3-only vs. CloudFront

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

## 8. Certificates, wildcard certificates, and subdomain levels

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

For v1.0.0 of this guide, use:

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

## 9. Track A: Create the S3 bucket

Track A creates the bucket and tests the site directly from S3.

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

Skip tags for now.

Optional future tags:

```text
Project = CalculatorSite
Environment = Test
```

---

### Step 8: Default encryption

Use the default:

```text
Server-side encryption with Amazon S3 managed keys (SSE-S3)
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

## 10. Track A: Upload the website files

### Step 1: Open the bucket

1. In S3, choose **Buckets**.
2. Choose:

```text
jefsko-calculator-site
```

---

### Step 2: Upload files

1. Choose **Upload**.
2. Choose **Add files**.
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

## 11. Track A: Enable S3 static website hosting

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

## 12. Track A: Make the bucket publicly readable

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
4. Paste this policy:

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

## 13. Track A: Test the S3 website endpoint

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

## 14. Track B: Request an ACM certificate for CloudFront

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

For v1.0.0, use the simpler exact-name certificate.

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

## 15. Track B: Validate the ACM certificate using Cloudflare DNS

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

## 16. Track B: Create the CloudFront distribution

For the recommended setup, CloudFront should use the regular S3 bucket origin with Origin Access Control.

This means:

- CloudFront can read the bucket.
- The public internet cannot directly read the bucket.
- Visitors access the site through CloudFront, not directly through S3.

This is better than using the S3 website endpoint as the CloudFront origin.

Important:

For Track B, CloudFront does not need S3 static website hosting. It uses the regular S3 bucket origin.

---

### Step 1: Open CloudFront

1. In the AWS Console search box, search for:

```text
CloudFront
```

2. Open **CloudFront**.
3. Choose **Create distribution**.

---

### Step 2: Choose distribution type or app type

If the console asks what kind of distribution/app you are creating, choose something like:

```text
Single website or app
```

Continue to the next step.

---

### Step 3: Choose the origin

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

The S3 website endpoint is public and HTTP-only. The regular S3 origin can be private and protected with Origin Access Control.

---

### Step 4: Use recommended origin settings

If CloudFront offers:

```text
Use recommended origin settings
```

choose it.

This should create or use Origin Access Control.

Look for language like:

```text
Origin Access Control
OAC
Sign requests
Recommended
```

Use:

```text
Sign requests: Yes / recommended
```

If CloudFront offers to update the S3 bucket policy automatically, allow it.

If it does not, this guide includes a manual bucket policy later.

---

### Step 5: Viewer protocol policy

Set viewer protocol policy to:

```text
Redirect HTTP to HTTPS
```

This means:

```text
http://calculator.jeffskone.com
```

will redirect to:

```text
https://calculator.jeffskone.com
```

---

### Step 6: Allowed HTTP methods

For a simple static website, use:

```text
GET, HEAD
```

You do not need POST, PUT, PATCH, or DELETE.

---

### Step 7: Cache policy

Use a managed/default cache policy suitable for static content.

A common choice is:

```text
CachingOptimized
```

If you are changing files frequently during early testing, caching can make updates look delayed. That is normal. You can invalidate the cache after uploading updates.

---

### Step 8: Web Application Firewall

If CloudFront asks about AWS WAF security protections, for this simple calculator site choose:

```text
Do not enable security protections
```

Reason:

WAF can be useful, but it can also add cost and complexity. You do not need it for the first version of a tiny static calculator site.

---

### Step 9: Default root object

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

### Step 10: Custom domain names

If the create-distribution flow lets you add custom domain names now, add:

```text
calculator.jeffskone.com
calc.jeffskone.com
```

Then choose the issued ACM certificate from `us-east-1`.

If the create flow does not ask for custom domains yet, create the distribution first and add custom domains afterward in the distribution settings.

Both flows are fine.

---

### Step 11: Create the distribution

1. Review the settings.
2. Choose **Create distribution**.
3. Wait for deployment.

CloudFront will give you a distribution domain name like:

```text
d111111abcdef8.cloudfront.net
```

Copy it. You will need it in Cloudflare.

Do not use the example value above. Use your real CloudFront distribution domain name.

---

### Step 12: Test the CloudFront distribution domain

After the distribution finishes deploying, test:

```text
https://<your-cloudfront-domain-name>
```

Example only:

```text
https://d111111abcdef8.cloudfront.net
```

You should see the calculator site.

If the root URL does not work, try:

```text
https://<your-cloudfront-domain-name>/index.html
```

If `/index.html` works but `/` does not, check the CloudFront default root object.

---

## 17. Track B: Make S3 private again and allow only CloudFront

If you completed Track A, your bucket is currently public.

For the recommended Track B final setup, change it so:

```text
Public internet → CloudFront → S3
```

not:

```text
Public internet → S3
```

### Step 1: Remove the public-read bucket policy

1. Open S3.
2. Open the bucket:

```text
jefsko-calculator-site
```

3. Go to **Permissions**.
4. Find **Bucket policy**.
5. Choose **Edit**.
6. Remove the public-read statement:

```json
{
  "Sid": "PublicReadForStaticWebsiteFiles",
  "Effect": "Allow",
  "Principal": "*",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::jefsko-calculator-site/*"
}
```

7. Save the policy.

If CloudFront already added an OAC policy, leave that policy in place.

---

### Step 2: Turn Block Public Access back on

1. Stay on the S3 bucket **Permissions** tab.
2. Find **Block public access (bucket settings)**.
3. Choose **Edit**.
4. Turn on:

```text
Block all public access
```

5. Save changes.

The old S3 website endpoint may stop working. That is expected.

The CloudFront URL should keep working.

---

### Step 3: Manual OAC bucket policy fallback

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

---

## 18. Track B: Add custom domains to CloudFront

If you did not add custom domains during distribution creation, add them now.

### Step 1: Open the distribution

1. Open **CloudFront**.
2. Choose your distribution.
3. Go to the distribution settings.

---

### Step 2: Edit alternate domain names

Look for:

```text
Alternate domain names
Custom domain names
CNAMEs
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

## 19. Track B: Point Cloudflare DNS to CloudFront

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

### Step 4: Why DNS only?

Use **DNS only** at first.

Reason:

- It keeps the setup simpler.
- CloudFront handles HTTPS.
- CloudFront handles caching.
- Cloudflare only handles DNS.
- Troubleshooting is much easier.

Later, you can experiment with Cloudflare proxying if you want. If you turn on the orange cloud, you are putting Cloudflare in front of CloudFront, which means you are stacking one CDN/proxy in front of another CDN. That can work, but it adds more moving parts.

If you proxy through Cloudflare later, use Cloudflare SSL/TLS mode:

```text
Full (strict)
```

and make sure CloudFront still has a valid certificate for the hostname.

---

## 20. Track B: Test the final HTTPS custom domains

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

## 21. Updating the website after setup

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

## 22. Troubleshooting

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

## 23. Optional alternatives outside AWS

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

## 24. Reference commands

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

## 25. Official references

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

AWS CloudFront getting started with S3 and OAC:

```text
https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.SimpleDistribution.html
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

---

## 26. Changelog

### v1.0.0 - 2026-06-30

Additive update from the initial draft.

Added:

- Versioned document header.
- Table of contents.
- Expanded summary.
- Clear Track A vs. Track B distinction.
- Direct explanation of whether an S3 bucket must match a custom domain.
- Explanation that S3-only custom domains require matching bucket names.
- Explanation that CloudFront custom domains do not require matching bucket names.
- Expanded custom-domain examples for:
  - `calculator.jeffskone.com`
  - `calc.jeffskone.com`
- Expanded AWS ACM certificate guidance.
- Wildcard certificate examples.
- Subdomain-level examples.
- Explanation for `*.calc.jeffskone.com`.
- Explanation of why `*.*.jeffskone.com` is not the right solution.
- CloudFront Track B setup.
- ACM certificate request steps in `us-east-1`.
- Added a plain-English explanation of why CloudFront viewer certificates must be requested in `us-east-1` even when the S3 bucket is in `us-west-2`.
- Cloudflare DNS validation steps.
- CloudFront Origin Access Control guidance.
- Steps to make the S3 bucket private again after Track A.
- Cloudflare custom-domain DNS steps.
- Optional outside-AWS alternatives.
- Reference commands.
- Official reference section.

Recommended final setup:

```text
Cloudflare DNS
  ↓
CloudFront with HTTPS
  ↓
Private S3 bucket in us-west-2
```
