# Frontend Hosting Architecture using AWS

> **Repository:** Frontend-Hosting-Architecture-using-AWS

## Table of Contents

1. [Overview](#overview)
2. [Architecture Diagram (logical)](#architecture-diagram-logical)
3. [AWS Services Used](#aws-services-used)
4. [Deployment Options & Flow](#deployment-options--flow)
5. [Prerequisites](#prerequisites)
6. [Step-by-step Manual Deployment](#step-by-step-manual-deployment)
7. [CI/CD — Recommended GitHub Actions Workflow Example](#cicd---recommended-github-actions-workflow-example)
8. [DNS & HTTPS (Route 53 + ACM)](#dns--https-route-53--acm)
9. [Security & Best Practices](#security--best-practices)
10. [Cost Considerations](#cost-considerations)
11. [Testing & Validation](#testing--validation)
12. [Troubleshooting](#troubleshooting)
13. [References & Further Reading](#references--further-reading)
14. [License](#license)

---

## Overview

This README documents a robust, production-ready architecture for hosting a static or single-page frontend application (React/Vue/Angular/Static HTML) on AWS. The goal is a fast, secure, and highly available setup using S3 + CloudFront, with optional automation through GitHub Actions or AWS CodePipeline.

Use cases:

* Static websites / SPAs
* Marketing sites
* Documentation portals

---

## Architecture Diagram (logical)

```
GitHub (repo) -> CI (GitHub Actions / CodeBuild) -> build artifacts (.zip / dist)
     ↓
  S3 (website / object storage)
     ↓
CloudFront Distribution (edge cache, TLS)
     ↓
Route 53 (optional) -> User

Optional: WAF in front of CloudFront for additional security.
Optional: Lambda@Edge for advanced header manipulations, A/B testing, SSR-ish tricks.
```

---

## AWS Services Used

* **Amazon S3** — Host static files (object storage)
* **Amazon CloudFront** — CDN, caching, TLS termination
* **AWS Certificate Manager (ACM)** — TLS certificates (for CloudFront use us-east-1)
* **Amazon Route 53** — DNS management (optional)
* **AWS WAF** — Web Application Firewall (optional)
* **AWS Lambda@Edge** — Optional edge logic (redirects, A/B testing, SSR tweaks)
* **CodeBuild / CodePipeline** or **GitHub Actions** — CI/CD pipeline

---

## Deployment Options & Flow

1. **Manual (Quick test)**

   * Build locally, upload `dist/` to an S3 bucket configured for static hosting.
   * Configure CloudFront to use the S3 bucket as origin.
   * Attach ACM certificate and point DNS to CloudFront.

2. **Automated (recommended)**

   * Use GitHub Actions to build and upload artifacts to S3, then optionally issue a CloudFront invalidation.
   * Alternatively, use AWS CodePipeline with CodeBuild for fully managed AWS-native CI/CD.

---

## Prerequisites

* An AWS account with permissions to create S3, CloudFront, ACM, Route 53, IAM roles, and optionally WAF.
* `aws` CLI configured locally (for manual steps): `aws configure`.
* Access to your DNS provider (or Route 53 hosted zone) to add records.
* Domain name (optional) for friendly URL and TLS.

---

## Step-by-step Manual Deployment

> These are the minimal manual steps to get a static SPA online.

### 1. Build your app

```bash
# example for a React app
npm ci
npm run build
# build output typically in `build/` or `dist/`
```

### 2. Create an S3 bucket

* Create an S3 bucket (choose a globally-unique name). Disable public ACLs but allow public access via CloudFront OAI or Origin Access Control (recommended).
* Upload build output to the bucket.

Example `aws` commands (simple):

```bash
aws s3 mb s3://my-frontend-bucket --region us-east-1
aws s3 sync build/ s3://my-frontend-bucket --delete
```

> **Important:** Do not enable S3 static website hosting with public objects when using CloudFront + OAC/OAI. Instead, use CloudFront origin access so the bucket remains private.

### 3. Configure Origin Access (OAC / OAI)

* Create a CloudFront Origin Access Control (OAC) or Origin Access Identity (OAI) and attach it to CloudFront to restrict S3 bucket access so only CloudFront can fetch objects.
* Update the S3 bucket policy to allow `s3:GetObject` from the CloudFront origin identity.

### 4. Create a CloudFront Distribution

* Origin: your S3 bucket.
* Viewer Protocol Policy: Redirect HTTP to HTTPS (or HTTPS only).
* Default Root Object: `index.html`.
* Behavior: Cache static assets (long TTL) and HTML (short TTL). Configure `Cache-Control` headers at upload-time for fine control.
* Attach ACM certificate for your domain (in `us-east-1`).
* (Optional) Enable HTTP/2 and IPv6.

### 5. Set up DNS

* In Route 53 (or your DNS provider) create an alias/A record pointing to the CloudFront distribution domain name.
* Use `www` and root (apex) handling via alias records where supported.

### 6. Cache invalidation after deploy

* After uploading new build files, create a CloudFront invalidation for `/*` or a more targeted path so users get fresh content.

Example CLI invalidation:

```bash
aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"
```

---

## CI/CD — Recommended GitHub Actions Workflow Example

Create `.github/workflows/deploy.yml` in your repo (example for Node.js + S3 + CloudFront):

```yaml
name: Deploy to S3 & CloudFront

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  id-token: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Use Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
      - name: Install dependencies
        run: npm ci
      - name: Run build
        run: npm run build
      - name: Sync to S3
        uses: jakejarvis/s3-sync-action@v0.5.2
        with:
          args: --delete
        env:
          AWS_S3_BUCKET: ${{ secrets.AWS_S3_BUCKET }}
          AWS_REGION: ${{ secrets.AWS_REGION }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      - name: Create CloudFront invalidation
        uses: chetan/invalidate-cloudfront-action@v2
        with:
          distribution: ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }}
          paths: '/*'
        env:
          AWS_REGION: ${{ secrets.AWS_REGION }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

> Store AWS credentials and bucket/distribution IDs in GitHub Secrets.

---

## DNS & HTTPS (Route 53 + ACM)

* Request a public certificate in **us-east-1** (N. Virginia) for CloudFront if you plan to use a custom domain.
* Validate via DNS (Route 53) or email.
* Add Route 53 Alias record that points your domain to the CloudFront distribution.

---

## Security & Best Practices

* **Use Origin Access Control or OAI** — keep S3 bucket private.
* **Least privilege IAM** — create an IAM user/role for CI with only the permissions needed (PutObject, ListBucket, CloudFront:CreateInvalidation).
* **Use HTTPS** everywhere (CloudFront + ACM).
* **Set strict cache-control** headers for assets, and short TTL for HTML to allow quick rollouts.
* **Enable AWS WAF** for OWASP rules if you expect traffic from varied sources.
* **Enable logging**: CloudFront access logs + S3 access logs for troubleshooting.
* **Use versioned object keys or hashed filenames** (e.g., `app.123abc.js`) to avoid invalidation on every deploy.

---

## Cost Considerations

* S3 storage (small for static sites) + GET requests.
* CloudFront is billed by data transfer and requests (in general small cost for typical websites, but can grow with traffic).
* ACM certificates are free when used with CloudFront.
* WAF adds additional monthly costs if enabled.

Check the AWS Pricing Calculator for accurate estimates.

---

## Testing & Validation

* Validate CloudFront uses the correct origin and certificate.
* Test both `https://your-domain` and the CloudFront domain directly.
* Use `curl -I` to inspect headers and confirm `Cache-Control`, `Content-Type`, and TLS.

