# Changelog

All notable changes to this project will be documented in this file.

This project loosely follows documentation-oriented semantic versioning:

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

## [Unreleased]

### Added

- `README.md` for repository overview, setup context, and links to the main guide.
- `CHANGELOG.md` for version history and notable documentation changes.

## [1.0.0] - 2026-06-30

### Added

- Initial complete guide: `aws-s3-static-website-quick-start.md`.
- Track A: minimal S3-only static website setup.
- Track B: recommended HTTPS/custom-domain setup using CloudFront, AWS Certificate Manager, Cloudflare DNS, and S3.
- Guidance for using `us-west-2` as the S3 bucket region.
- Explanation that CloudFront viewer certificates from ACM must be requested in `us-east-1`.
- Explanation of why the CloudFront certificate region differs from the S3 bucket region.
- S3 bucket naming guidance.
- Explanation of when an S3 bucket must match a custom domain or subdomain.
- Explanation that CloudFront allows custom domains without requiring the S3 bucket name to match the domain.
- Guidance for `calculator.jeffskone.com` and `calc.jeffskone.com`.
- AWS ACM certificate guidance.
- Wildcard certificate examples, including:
  - `*.jeffskone.com`
  - `*.calc.jeffskone.com`
- Explanation of wildcard certificate subdomain-level limits.
- Cloudflare DNS validation steps for ACM.
- CloudFront distribution setup steps.
- CloudFront Origin Access Control guidance.
- Steps for making the S3 bucket private after CloudFront is configured.
- Cloudflare DNS records for custom domains.
- Website update and CloudFront cache invalidation steps.
- Troubleshooting section.
- Optional alternatives outside AWS.
- Reference commands.
- Official references section.

### Recommended setup

```text
Cloudflare DNS
  ↓
CloudFront with HTTPS
  ↓
Private S3 bucket in us-west-2
```

### Example project values

```text
S3 bucket: jefsko-calculator-site
S3 bucket region: us-west-2
CloudFront certificate region: us-east-1
Primary domain: calculator.jeffskone.com
Short alias domain: calc.jeffskone.com
```
