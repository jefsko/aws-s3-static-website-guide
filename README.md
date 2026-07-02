# AWS S3 Static Website Quick Start

A beginner-friendly documentation set for hosting a simple static website on AWS using Amazon S3, CloudFront, AWS Certificate Manager, and Cloudflare DNS.

The example site is a small calculator website with three root-level files:

```text
index.html
style.css
calculator.js
```

## Current documentation set version

`v1.2.1`

## Full guide version

`v1.2.0`

The main full guide was not changed for `v1.2.1`. This patch release adds supplemental quick-start and cheat-sheet documentation.

## Which file should I open first?

| Goal | Start here |
|---|---|
| I want the shortest practical walkthrough. | [`aws-s3-static-website-quick-start-guide-v1.2.1.md`](aws-s3-static-website-quick-start-guide-v1.2.1.md) |
| I want a compact repeat checklist. | [`aws-s3-static-website-cheat-sheet-v1.2.1.md`](aws-s3-static-website-cheat-sheet-v1.2.1.md) |
| I need the full explanation and troubleshooting. | [`aws-s3-static-website-quick-start.md`](aws-s3-static-website-quick-start.md) |

## Main guide

Full guide:

[`aws-s3-static-website-quick-start.md`](aws-s3-static-website-quick-start.md)

The guide uses shared S3 setup plus two paths:

| Track | Name | Purpose |
|---|---|---|
| Shared setup | Required baseline | Create the S3 bucket and upload the static files |
| Track A | Minimal S3-only setup | Optional quick HTTP test using an S3 static website endpoint |
| Track B | Recommended setup | HTTPS custom-domain setup using CloudFront, ACM, Cloudflare DNS, and private S3 bucket access |

## Supplemental files added in v1.2.1

| File | Purpose |
|---|---|
| `aws-s3-static-website-quick-start-guide-v1.2.1.md` | Shorter sequential setup guide for the recommended workflow |
| `aws-s3-static-website-cheat-sheet-v1.2.1.md` | Compact checklist for repeat use after learning the workflow |

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
├── aws-s3-static-website-quick-start-guide-v1.2.1.md
├── aws-s3-static-website-cheat-sheet-v1.2.1.md
├── index.html
├── style.css
└── calculator.js
```

Current documentation files:

| File | Purpose |
|---|---|
| `README.md` | Repo overview and starting point |
| `CHANGELOG.md` | Version history and notable changes |
| `aws-s3-static-website-quick-start.md` | Full setup guide |
| `aws-s3-static-website-quick-start-guide-v1.2.1.md` | Quick-start companion guide |
| `aws-s3-static-website-cheat-sheet-v1.2.1.md` | Compact cheat sheet |

Website files expected by the guide:

| File | Purpose |
|---|---|
| `index.html` | Main web page |
| `style.css` | Site styling |
| `calculator.js` | Calculator behavior |

## What this documentation is for

This documentation is intended for a novice-to-advanced audience. It is written casually, but with explicit technical steps and examples.

Use the quick-start guide when you want the fastest practical path.
Use the cheat sheet when you already understand the flow and want a compact checklist.
Use the full guide when you need the detailed explanations, reference sections, troubleshooting, or optional advanced topics.

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

This release:

```text
1.2.1 = supplemental quick-start guide and cheat sheet added; full guide remains v1.2.0
```

See [`CHANGELOG.md`](CHANGELOG.md) for notable changes.

## Recommended first setup path

For the first run, follow this order:

1. Open the quick-start guide.
2. Complete the shared setup: create the S3 bucket and upload the files.
3. Optionally complete Track A if you want a quick HTTP-only S3 website endpoint test.
4. Complete Track B to add CloudFront, HTTPS, the ACM certificate, and Cloudflare DNS.
5. Make sure the S3 bucket is private once CloudFront is working.
6. Disable S3 Static website hosting unless you intentionally need the S3 website endpoint.
7. Use CloudFront/custom domains as the final public URLs.
8. Use the cheat sheet later as a repeat checklist.

Final intended URLs:

```text
https://calculator.jeffskone.com
https://calc.jeffskone.com
```

## Notes

This documentation assumes:

- The AWS account already exists.
- The domain is managed in Cloudflare.
- The static site has no server-side code.
- The site files are stored at the root of the S3 bucket.

For this type of small static website, S3 is a simple and low-cost file host. CloudFront is recommended for the final public setup because S3 static website endpoints do not support HTTPS.
