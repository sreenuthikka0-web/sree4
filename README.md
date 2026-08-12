# Sreenu's Project — CI/CD Pipeline (GitHub → AWS)

This project auto-deploys to **AWS S3 + CloudFront** every time you push to the `main` branch via **GitHub Actions**.

```
GitHub push → GitHub Actions → S3 sync → CloudFront invalidation → Live site
```

---

## Project Structure

```
projet 4/
├── index.html                        # Your website
├── .github/
│   └── workflows/
│       └── deploy.yml                # CI/CD pipeline
├── infra/
│   └── cloudformation.yml            # AWS infrastructure (S3 + CloudFront)
└── README.md
```

---

## One-Time Setup

### Step 1 — Prerequisites

Make sure you have:
- An **AWS account**
- **AWS CLI** installed and configured (`aws configure`)
- A **GitHub account** with this project pushed to a repo

---

### Step 2 — Deploy AWS Infrastructure

Run this command once to create your S3 bucket and CloudFront distribution:

```bash
aws cloudformation deploy \
  --template-file infra/cloudformation.yml \
  --stack-name sreenu-website \
  --parameter-overrides ProjectName=sreenu-website \
  --region us-east-1
```

After it completes, get your resource values:

```bash
aws cloudformation describe-stacks \
  --stack-name sreenu-website \
  --query "Stacks[0].Outputs" \
  --output table
```

Note down:
| Output Key | What it is |
|---|---|
| `BucketName` | Your S3 bucket name |
| `CloudFrontDistributionId` | Your CloudFront distribution ID |
| `CloudFrontURL` | Your live website URL |

---

### Step 3 — Create an IAM User for GitHub Actions

Create a dedicated IAM user with least-privilege permissions:

```bash
# Create user
aws iam create-user --user-name github-actions-deployer

# Attach inline policy
aws iam put-user-policy \
  --user-name github-actions-deployer \
  --policy-name DeployPolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:PutObject","s3:DeleteObject","s3:ListBucket","s3:GetObject"],
        "Resource": [
          "arn:aws:s3:::YOUR_BUCKET_NAME",
          "arn:aws:s3:::YOUR_BUCKET_NAME/*"
        ]
      },
      {
        "Effect": "Allow",
        "Action": ["cloudfront:CreateInvalidation","cloudfront:GetDistribution"],
        "Resource": "*"
      }
    ]
  }'

# Generate access keys
aws iam create-access-key --user-name github-actions-deployer
```

> Replace `YOUR_BUCKET_NAME` with the bucket name from Step 2.

Save the `AccessKeyId` and `SecretAccessKey` — you will need them in the next step.

---

### Step 4 — Add GitHub Secrets

Go to your GitHub repo → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add these three secrets:

| Secret Name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Access key from Step 3 |
| `AWS_SECRET_ACCESS_KEY` | Secret key from Step 3 |
| `S3_BUCKET_NAME` | Bucket name from Step 2 |
| `CLOUDFRONT_DISTRIBUTION_ID` | Distribution ID from Step 2 |

---

### Step 5 — Push and Deploy

```bash
git add .
git commit -m "Initial deployment"
git push origin main
```

GitHub Actions will automatically:
1. Checkout your code
2. Authenticate with AWS
3. Sync all files to S3
4. Invalidate the CloudFront cache
5. Print your live URL in the workflow logs

---

## How the Pipeline Works

```yaml
Trigger:   push to main  (or manual via workflow_dispatch)
Runner:    ubuntu-latest
Steps:
  1. Checkout code
  2. Configure AWS credentials (from GitHub Secrets)
  3. aws s3 sync → uploads changed files, deletes removed ones
  4. aws cloudfront create-invalidation → clears CDN cache
  5. Print live URL
```

---

## Monitoring Deployments

- View pipeline runs: `https://github.com/<your-username>/<repo>/actions`
- Each run shows logs for every step
- Failed deployments send a notification via GitHub

---

## Updating the Site

Just push to `main` — the pipeline handles the rest:

```bash
# Edit index.html, then:
git add index.html
git commit -m "Update welcome page"
git push origin main
```

Changes are live within ~1-2 minutes.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `AccessDenied` on S3 sync | Check IAM policy and secret key values in GitHub |
| CloudFront still shows old content | Invalidation may take 1-2 min; check the Actions log |
| CloudFormation deploy fails | Ensure your AWS CLI has `cloudformation:*` and `s3:*` and `cloudfront:*` permissions |
| Bucket name already taken | S3 bucket names are global; change `ProjectName` to something unique |
