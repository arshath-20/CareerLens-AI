# CareerLens-AI: AWS Free Tier deployment

This guide sets up static frontend deployment from the `production` branch using GitHub Actions and AWS S3. It intentionally uses GitHub OIDC instead of long-lived AWS access keys.

## Architecture (first deployment milestone)

- Frontend: Vite production build hosted as an S3 static website.
- CI: `.github/workflows/ci.yml` validates backend imports/tests and builds the frontend.
- CD: `.github/workflows/deploy-frontend.yml` builds the frontend and syncs `frontend/dist` to S3 after changes on `production`.
- Backend: remains on its current host until it is adapted for stateless/serverless execution and persistent resume/session storage.

## AWS account and cost guardrails

Choose the AWS **Free account plan** during signup if the account is eligible and the goal is to avoid charges while experimenting. AWS currently describes this plan as limited to six months or until credits are exhausted, whichever comes first; access to services is restricted. Confirm the account plan and service eligibility in the AWS console before creating resources.

Do not create NAT Gateways, load balancers, EC2 instances, ECS services, paid support plans, or a Route 53 hosted zone for this first milestone. Set an AWS Budget / billing alerts before deploying. S3 requests, storage, and data transfer have limits and can incur charges on a paid plan or after upgrading/exceeding allowances. A public S3 website endpoint serves HTTP, not HTTPS. CloudFront can add HTTPS but must be reviewed against the account's current Free Tier allowance and usage.

## 1. Create the S3 website bucket

1. Open the S3 console in your selected AWS region.
2. Create a globally unique bucket name, for example `careerlens-ai-frontend-<unique-suffix>`.
3. Keep Object Ownership as **Bucket owner enforced**.
4. In bucket Properties, enable **Static website hosting**.
5. Set Index document to `index.html`. Set Error document to `index.html` only if the frontend is a client-side SPA and its routes need fallback.
6. In Permissions, turn off **Block all public access** for this website bucket only. A public website bucket is intentionally readable by anyone; never store secrets or user uploads in it.
7. Add this bucket policy, replacing the bucket name:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadForStaticWebsiteOnly",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    }
  ]
}
```

8. Save the website endpoint shown under Static website hosting. It is HTTP-only.

## 2. Create GitHub OIDC identity in AWS

Use the IAM console:

1. Add an identity provider with provider URL `https://token.actions.githubusercontent.com` and audience `sts.amazonaws.com` if it does not already exist.
2. Create an IAM role for web identity using that provider.
3. Restrict the trust policy to this exact repository and production environment. Replace `OWNER` and `REPO`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::YOUR_ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:environment:production"
        }
      }
    }
  ]
}
```

4. Attach a least-privilege permissions policy to this role, replacing the bucket name:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListFrontendBucket",
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME"
    },
    {
      "Sid": "DeployFrontendObjects",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    }
  ]
}
```

5. Copy the role ARN. Do not create or store IAM user access keys for this workflow.

## 3. Configure GitHub repository variables

In GitHub: Settings -> Secrets and variables -> Actions -> Variables. Add:

| Variable | Value |
|---|---|
| `AWS_ROLE_TO_ASSUME` | IAM role ARN created above |
| `AWS_REGION` | Region where the bucket was created |
| `AWS_S3_BUCKET` | Exact S3 bucket name |
| `VITE_API_BASE_URL` | Current backend base URL, e.g. the Render service URL |

The backend URL must be a base URL only (no trailing `/api`) because frontend requests already include `/api/...`.

No AWS access key or secret key is required. The workflow exchanges the GitHub OIDC token for short-lived AWS credentials.

## 4. Deploy

1. Push this workflow to `production`.
2. Open GitHub -> Actions -> Deploy CareerLens AI frontend to S3.
3. Confirm the workflow succeeds.
4. Open the S3 static website endpoint from the bucket Properties page.
5. Test login, API health, resume upload, and the main user flows.

The deployment workflow runs on changes under `frontend/` or to the deployment workflow itself, and can also be started manually.

## Backend production blockers (do not skip)

The current FastAPI backend is not ready to move directly to Lambda:

- `active_sessions` and resume analysis state are held in process memory. Lambda execution environments are temporary and not shared across concurrent instances.
- Uploaded PDFs are written to the local `backend/uploads` directory. Lambda local storage is temporary; use S3 for durable uploads.
- The Render keep-alive thread starts on application startup and should be removed or disabled outside Render.
- CORS currently allows every origin. Restrict it to the actual frontend website origin.
- The current API has long-running analysis operations. Check API Gateway / Lambda timeout and payload constraints before choosing an API front door.

Before backend CD, migrate upload storage to S3 and session/evaluation state to durable storage (MongoDB is already integrated; DynamoDB is another option), make the app stateless, add a Lambda ASGI adapter, and test the actual deployment package. Do not claim backend deployment is complete until those checks pass.

## Rollback

S3 sync with `--delete` replaces the current site contents. For safer releases, enable S3 Versioning before production use and retain previous build artifacts in GitHub Actions or a separate release prefix. To roll back, re-upload the previous known-good build.
