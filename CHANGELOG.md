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

- Nothing yet.

## [1.2.2] - 2026-07-01

### Changed

- Renamed the full setup guide reference from `aws-s3-static-website-quick-start.md` to `aws-s3-static-website-setup.md` after the repository file was renamed.
- Updated README navigation and supplemental guide links to point to `aws-s3-static-website-setup.md`.
- Updated supplemental quick-start guide and cheat-sheet version references from `v1.2.1` to `v1.2.2`.

### Notes

- This is a supplemental documentation patch release from `v1.2.1`.
- The full guide content was not materially changed for this release.
- The rename avoids confusion with the separate quick-start guide and cheat sheet.

## [1.2.1] - 2026-07-01

### Added

- Added `aws-s3-static-website-quick-start-guide-v1.2.1.md` as a quick-start companion guide for the recommended AWS S3 + CloudFront + ACM + Cloudflare setup.
- Added `aws-s3-static-website-cheat-sheet-v1.2.1.md` as a compact repeat-use checklist for the same workflow.
- Added README navigation showing which file to open first: quick-start guide, cheat sheet, or full guide.

### Changed

- Updated README repository contents to include the new supplemental documentation files.
- Updated README version context to distinguish the `v1.2.1` documentation set from the unchanged `v1.2.0` full guide.

### Notes

- This is a supplemental patch release from `v1.2.0`.
- The main full guide content was not changed for this release.
- The supplemental files are intended to be tagged as `v1.2.1` in the repository.

## [1.2.0] - 2026-07-01

### Added

- Expanded access-control guidance for the recommended CloudFront + private S3 setup.
- Clear terminology for S3 website endpoint, S3 bucket origin, direct S3 object URL, and CloudFront standard distribution domain.
- Stronger final recommended state: S3 Static website hosting disabled unless intentionally needed, S3 Block Public Access enabled, and OAC bucket policy allowing only CloudFront.
- Expanded Cloudflare DNS guidance explaining website CNAMEs, apex/root CNAME flattening, validation CNAMEs, DNS-only records, and optional Cloudflare proxying.
- ACM certificate behavior clarification: the certificate enables HTTPS for custom domains but does not block the CloudFront standard domain.
- OAC vs. OAI explanation.
- Guidance for confirming or creating a regular S3 bucket origin with OAC.
- CloudFront Default root object vs. S3 website folder-index behavior explanation.
- Optional CloudFront Function examples for blocking the standard CloudFront domain and rewriting clean subdirectory URLs.
- CloudFront custom error response guidance for normal `404.html` pages and single-page-app fallback behavior.
- Additional verification commands for custom domains, CloudFront standard domain, S3 website endpoint, direct S3 object URLs, clean URLs, and missing pages.
- Additional troubleshooting entries and official references.

### Changed

- Cleaned repeated horizontal-rule formatting from the prior guide file.
- Clarified that public website DNS should point to CloudFront only for the recommended architecture.
- Clarified that the final bucket policy should not contain public-read access and usually should contain the OAC allow statement.

### Notes

- This is an additive update from `v1.1.0`.
- No existing version history was removed.

## [1.1.0] - 2026-07-01

### Added

- Clear shared setup explanation for Track A and Track B.
- Section guidance showing which steps apply to Track A, Track B, or both.
- Clarification that Track B needs the S3 bucket and uploaded files, but does not require the public S3 website endpoint steps.
- Expanded dashes vs. periods bucket naming guidance.
- Optional AWS tags guidance.
- S3 Bucket Key explanation.
- Drag-and-drop upload guidance.
- Static website hosting enable/disable explanation.
- Variables, placeholders, ARNs, and where to find values.
- Expanded CloudFront distribution guidance for name, description, origin choice, OAC, behaviors, TLS policy, deployment status, and the default CloudFront distribution domain.
- Clearer Alternate domain name (CNAME) terminology.
- Final access expectations and URL behavior matrix.
- Optional advanced note about restricting the default CloudFront domain.
- Additional troubleshooting entries and official references.

### Changed

- Reframed the guide flow as shared setup plus optional Track A and recommended Track B.
- Renamed the S3 bucket creation and upload sections as shared setup because both tracks use them.
- Clarified public-read bucket policy cleanup for the final CloudFront/OAC/private-bucket setup.

### Notes

- This is an additive update from `v1.0.0`.
- No existing version history was removed.

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
