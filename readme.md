# Personal website

This repository contains the source code for my personal website, developed with [Hugo](https://gohugo.io/) and hosted on AWS using **S3** and **CloudFront**. Deployment is automated via **GitHub Actions**.

## Technologies Used

- **[Hugo](https://gohugo.io/)** - Static site generator
- **[hugo-coder theme](https://github.com/luizdepra/hugo-coder)** - Clean and minimalist theme for Hugo
- **AWS S3** - Storage for static files
- **AWS CloudFront** - CDN for fast content delivery
- **GitHub Actions** - CI/CD for automatic deployment

## Hosting
The site is hosted on AWS with an **S3** bucket serving static files and a **CloudFront Distribution** as a CDN to optimize global delivery.

## Automatic Deployment
Deployment is automatically triggered whenever there is a push to the main branch. GitHub Actions executes the following steps:

1. Checkout the repository
2. Setup Hugo
3. Authenticate with AWS via OIDC
4. Generate static files with Hugo
5. Upload files to S3
6. Invalidate the CloudFront cache to ensure changes are immediately reflected.

## Manually Running the Deployment
If you need to force a manual deployment, run the following commands:

```bash
hugo --minify
hugo deploy --maxDeletes -1 --invalidateCDN
aws cloudfront create-invalidation --distribution-id <id> --paths "/*"
```

## Environment Configuration
Sensitive credentials and configurations are stored as GitHub secrets. The environment variables used in the deployment are:

- `AWS_S3_BUCKET` - Name of the S3 bucket
- `CLOUDFRONT_DISTRIBUTION_ID` - CloudFront distribution ID
- `ROLE_ARN` - IAM Role for OIDC
