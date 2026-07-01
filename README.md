# AWS S3 Static Website Quick Start

A beginner-friendly quick start guide for hosting a simple static website on AWS using Amazon S3, CloudFront, AWS Certificate Manager, and Cloudflare DNS.

The example site is a small calculator website with three root-level files:

```text
index.html
style.css
calculator.js
```

## Current guide version

`v1.2.0`

## Main guide

Start here:

[`aws-s3-static-website-quick-start.md`](aws-s3-static-website-quick-start.md)

The guide uses shared S3 setup plus two paths:

| Track | Name | Purpose |
|---|---|---|
| Shared setup | Required baseline | Create the S3 bucket and upload the static files |
| Track A | Minimal S3-only setup | Optional quick HTTP test using an S3 static website endpoint |
| Track B | Recommended setup | HTTPS custom-domain setup using CloudFront, ACM, Cloudflare DNS, and private S3 bucket access |

## Recommended final architecture

```text
Visitor
  ↓
calculator.jeffskone.com or calc.jeffskone.com
  ↓
Cloudflare DNS
  ↓
Amazon CloudFront
  ↓
Private S3 bucket
```

## Recommended final access-control state

```text
Custom domain → Cloudflare DNS → CloudFront → OAC → private S3 bucket
```

Recommended final state:

| Area | Setting |
|---|---|
| S3 Static website hosting | Disabled unless intentionally needed |
| S3 Block Public Access | Enabled |
| S3 bucket policy | Allow only CloudFront OAC; no public-read statement |
| CloudFront origin | Normal S3 bucket origin, not S3 website endpoint |
| CloudFront default root object | `index.html` |
| CloudFront standard domain | Works by default; optionally block with CloudFront Function |

## Example project values

```text
S3 bucket: jefsko-calculator-site
S3 bucket region: us-west-2
CloudFront certificate region: us-east-1
Primary domain: calculator.jeffskone.com
Short alias domain: calc.jeffskone.com
```

Important note:

The S3 bucket can be in `us-west-2`, which is a good West Coast / Seattle-area default. However, the ACM certificate used by CloudFront must be requested in `us-east-1`.


## What changed in v1.2.0

Version `v1.2.0` is an additive access-control and CloudFront behavior update.

It clarifies:

- The recommended final setup uses a normal private S3 bucket origin with CloudFront OAC.
- S3 Static website hosting should be disabled unless the S3 website endpoint is intentionally needed.
- Public website DNS records in Cloudflare should point to CloudFront, not directly to S3.
- ACM/domain validation CNAMEs should be DNS-only, while the website CNAME can stay DNS-only initially and optionally be proxied later.
- The CloudFront standard distribution domain works by default and can optionally be blocked with a CloudFront Function.
- CloudFront Default root object is not identical to S3 website folder-index behavior.
- CloudFront custom error responses are the right place to configure `404.html` or SPA fallback behavior when using a private S3 bucket origin.

## Repository name and description

Recommended repository name:

```text
aws-s3-static-website-guide
```

Recommended GitHub repository description:

```text
Beginner-friendly guide for hosting a static website on AWS using S3, CloudFront, ACM, and Cloudflare DNS.
```

## Repository contents

Recommended repo structure:

```text
/
├── README.md
├── CHANGELOG.md
├── aws-s3-static-website-quick-start.md
├── index.html
├── style.css
└── calculator.js
```

Current documentation files:

| File | Purpose |
|---|---|
| `README.md` | Repo overview and starting point |
| `CHANGELOG.md` | Version history and notable changes |
| `aws-s3-static-website-quick-start.md` | Main setup guide |

Website files expected by the guide:

| File | Purpose |
|---|---|
| `index.html` | Main web page |
| `style.css` | Site styling |
| `calculator.js` | Calculator behavior |

## What this guide is for

This guide is intended for a novice-to-advanced audience. It is written casually, but with explicit technical steps and examples.

It explains:

- How to create an S3 bucket.
- How to upload static website files.
- How to enable S3 static website hosting for quick testing.
- How S3 bucket names work.
- When an S3 bucket must match a custom domain.
- Why CloudFront is recommended for HTTPS.
- Why CloudFront certificates must be requested in `us-east-1`.
- How to request and validate an ACM certificate using Cloudflare DNS.
- How to configure CloudFront for custom domains.
- How to use CloudFront Origin Access Control so the S3 bucket can be private.
- How to troubleshoot common setup problems.

## Versioning

This project uses simple documentation-oriented semantic versioning:

```text
MAJOR.MINOR.PATCH
```

Suggested meaning:

```text
1.0.0 = first complete guide
1.1.0 = additive guide expansion or new major section
1.1.1 = correction, clarification, typo fix, or small accuracy update
2.0.0 = major restructure or change in recommended approach
```

See [`CHANGELOG.md`](CHANGELOG.md) for notable changes.

## Recommended first setup path

For the first run, follow the guide in this order:

1. Complete the shared setup: create the S3 bucket and upload the files.
2. Optionally complete Track A if you want a quick HTTP-only S3 website endpoint test.
3. Complete Track B to add CloudFront, HTTPS, the ACM certificate, and Cloudflare DNS.
4. Make sure the S3 bucket is private once CloudFront is working.
5. Disable S3 Static website hosting unless you intentionally need the S3 website endpoint.
6. Use CloudFront/custom domains as the final public URLs.

Final intended URLs:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

## Notes

This guide assumes:

- The AWS account already exists.
- The domain is managed in Cloudflare.
- The static site has no server-side code.
- The site files are stored at the root of the S3 bucket.

For this type of small static website, S3 is a simple and low-cost file host. CloudFront is recommended for the final public setup because S3 static website endpoints do not support HTTPS.
