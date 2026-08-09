# DevOps Accelerator — Step-by-Step Deployment Tutorial

A practical, corrected walkthrough for deploying the `pwSkills-CapstoneProject` (DevOps Accelerator) into **your own AWS account**.

This guide is based on the actual Terraform, Lambda, and GitHub Actions code in
the repo — not just the README — and fixes a few hardcoded values and ordering
issues that will otherwise break the deployment.

---

## 1. What you are building (architecture)

```
                 ┌─────────────┐
Browser ───────► │ CloudFront  │ ──► S3 (frontend bucket: static HTML)
                 └─────────────┘

Browser ──POST──► API Gateway ──► presign Lambda ──► returns pre-signed S3 URL
                                       (generate-presigned-url)

Browser ──PUT file──► S3 (upload bucket)
                           │
                           │  s3:ObjectCreated event
                           ▼
                      process Lambda ──► CloudWatch logs
                      (process-uploaded-file) ──► SNS ──► email to you
```

**Provisioning:** Terraform, with a remote backend (S3 for state + DynamoDB for locking). **CI/CD:** 3 GitHub Actions workflows — `terraform.yml`, `backend-deploy.yml`, `frontend.yml`.

---

## 2. Prerequisites

Install / have ready:

- An **AWS account** with admin access
- **AWS CLI v2** — verify: `aws --version`
- **Terraform v1.5+** — verify: `terraform -version`
- **Git** + a **GitHub account**
- `zip` available on your machine (Linux/macOS have it; on Windows use WSL or Git Bash)

> 💡 This project creates billable resources (CloudFront, API Gateway, Lambda, S3,
> SNS). Costs are small but **not zero**. See **Section 14 (Teardown)** to remove
> everything when you're done.

---

## 3. Get the code into your own GitHub repo

You need the code in **your** GitHub account so GitHub Actions can run with your secrets.

The cleanest path is to **fork** the original repo on GitHub
(`sidoncode/pwSkills-CapstoneProject`) using the "Fork" button, then clone your fork:

```
git clone https://github.com/<your-username>/pwSkills-CapstoneProject.git
cd pwSkills-CapstoneProject
```

> ⚠️ The README's `git clone git@github.com:Anees-DevOps/devops-accelerator.git` line points at an unrelated repo — ignore it. Use your fork instead.

---

## 4. Set up AWS credentials

1. AWS Console → **IAM** → **Users** → **Create user**.
2. Open the user → **Security credentials** → **Create access key** →
   choose "Command Line Interface (CLI)" → **download the `.csv`**.
3. **Permissions** → Add permissions → Attach policies directly →
   attach **`AdministratorAccess`** (fine for a learning project).
4. Configure the CLI locally:

```
aws configure
# AWS Access Key ID:     <from the CSV>
# AWS Secret Access Key: <from the CSV>
# Default region name:   us-east-1
# Default output format: json
```

5. Confirm it works:

```
aws sts get-caller-identity
```

---

## 5. Customize the hardcoded values (the most important step)

The repo ships with the author's own names baked in. **S3 bucket names are globally
unique across all AWS accounts**, so you MUST change them or `apply` will fail.

### 5a. Pick three unique bucket names

Decide on three globally-unique names now (add random characters to be safe):

| Purpose                 | Example name                      |
| ------------------------ | ---------------------------------- |
| Terraform state bucket   | `myname-devops-acc-tfstate-7x9k`  |
| Upload bucket            | `myname-devops-acc-upload-7x9k`   |
| Frontend hosting bucket  | `myname-devops-acc-frontend-7x9k` |

### 5b. Edit `infra/terraform/main.tf` — the backend block

Open `infra/terraform/main.tf` and update the **backend "s3"** block at the top so
the `bucket` matches the state-bucket name you chose. Keep the DynamoDB table name
or change it (just stay consistent with Step 6):

```hcl
backend "s3" {
  bucket         = "myname-devops-acc-tfstate-7x9k"   # <-- your state bucket
  key            = "global/devops-accelerator/terraform.tfstate"
  region         = "us-east-1"
  dynamodb_table = "devops-accelerator-tf-locker"     # <-- match Step 6
  encrypt        = true
}
```

### 5c. Edit `infra/terraform/terraform.tfvars`

Replace every value with your own:

```hcl
aws_region             = "us-east-1"
upload_bucket_name     = "myname-devops-acc-upload-7x9k"
frontend_bucket_name   = "myname-devops-acc-frontend-7x9k"
cloudfront_price_class = "PriceClass_100"
notification_email     = "you@example.com"   # <-- YOUR email for SNS alerts
```

> The `notification_email` is a **required** Terraform variable. AWS will send a
> subscription-confirmation email here — you must click it later (Step 11).

---

## 6. Create the Terraform remote-backend resources manually

The backend (state bucket + lock table) must exist **before** `terraform init`.
Terraform can't create the bucket that stores its own state, so do it by hand once:

```bash
# State bucket — name MUST match the backend block in main.tf (Step 5b)
aws s3api create-bucket \
  --bucket myname-devops-acc-tfstate-7x9k \
  --region us-east-1

# DynamoDB lock table — name MUST match dynamodb_table in main.tf
aws dynamodb create-table \
  --table-name devops-accelerator-tf-locker \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

> Note: for regions other than `us-east-1`, `create-bucket` needs `--create-bucket-configuration LocationConstraint=<region>`. Staying in `us-east-1` keeps things simplest.

---

## 7. Package the Lambda functions

Terraform references a `lambda.zip` inside each Lambda folder. Re-create them so
they contain your current code:

```bash
cd backend/lambda/process-uploaded-file
rm -f lambda.zip
zip -r lambda.zip main.py

cd ../generate-presigned-url
rm -f lambda.zip
zip -r lambda.zip main.py

cd ../../..   # back to repo root
```

> 🔁 Any time you edit a Lambda's `main.py`, delete the old `lambda.zip` and re-zip
> before pushing — Terraform deploys whatever is in the zip.

---

## 8. Validate Terraform locally (do NOT apply locally)

This project's rule: **never run `terraform apply` from your laptop** — all applies
go through the CI/CD pipeline. Locally you only initialize, validate, and preview:

```bash
cd infra/terraform
terraform init        # connects to your remote backend (Step 6)
terraform validate    # checks syntax/config
terraform plan        # previews what will be created
cd ../..
```

If `plan` runs cleanly and lists the resources to be created, you're good.

---

## 9. Add GitHub repository secrets

In **GitHub → your repo → Settings → Secrets and variables → Actions →
New repository secret**, add:

| Secret name              | Value                                    |
| ------------------------- | ----------------------------------------- |
| `AWS_ACCESS_KEY_ID`      | from your IAM CSV                        |
| `AWS_SECRET_ACCESS_KEY`  | from your IAM CSV                        |
| `AWS_REGION`             | `us-east-1`                              |
| `LAMBDA_FUNCTION_NAME`   | `process-uploaded-file`                  |
| `UPLOAD_BUCKET_NAME`     | your upload bucket (matches `tfvars`)    |
| `FRONTEND_BUCKET_NAME`   | your frontend bucket (matches `tfvars`)  |

> `CLOUDFRONT_DIST_ID` is added **later** (Step 12) — it doesn't exist until after
> the first Terraform apply.

---

## 10. First push → run the Terraform pipeline

Commit your customized files and push to `main`:

```bash
git add .
git commit -m "Customize for my AWS account"
git push origin main
```

Go to **GitHub → Actions**. Three workflows can trigger:

- **Terraform Deploy** — runs on every push; this is the one that builds all infra.
- **Deploy Backend Lambda** — runs only when `backend/lambda/process-uploaded-file/**` changes.
- **Deploy Frontend** — runs only when `frontend/**` changes.

On this first push, let the **Terraform Deploy** job finish successfully. It runs `init -reconfigure`, `plan`, then `apply -auto-approve` and creates everything:
buckets, both Lambdas, API Gateway, CloudFront, SNS, IAM, CloudWatch.

> ⚠️ The **Deploy Frontend** workflow will likely **fail on this first push** because `CLOUDFRONT_DIST_ID` isn't set yet. That's expected — you'll fix it in
> the next step and re-run it.

### Push troubleshooting (`.terraform/` too large)

If git rejects the push due to a large `.terraform/` directory:

```bash
echo ".terraform/" >> .gitignore
git rm -r --cached infra/terraform/.terraform
git commit -m "Ignore .terraform directory"
git push origin main
```

---

## 11. Read the Terraform outputs & confirm SNS

After the Terraform job succeeds, get the outputs. Either open the **Terraform Apply** step log in the Actions run, or run locally:

```bash
cd infra/terraform
terraform output
cd ../..
```

You'll see something like:

```
cloudfront_url             = "dxxxxxxxxxxxxx.cloudfront.net"
frontend_bucket_name       = "myname-devops-acc-frontend-7x9k"
presigned_url_api_endpoint = "https://xxxxxxxx.execute-api.us-east-1.amazonaws.com"
lambda_function_name       = "process-uploaded-file"
```

Now **check your email inbox** for an *AWS Notification — Subscription Confirmation* message and click **Confirm subscription**. Without this you won't get upload alerts.

---

## 12. Wire up the frontend (CloudFront ID + API endpoint)

Two things must be set now that the infra exists:

**12a. Add the CloudFront secret.** In AWS Console → CloudFront → Distributions,
copy the **ID** (e.g. `E1ABCDEF2GHIJ`) of the new distribution. Add it as a GitHub
secret:

| Secret name           | Value                 |
| ----------------------- | ---------------------- |
| `CLOUDFRONT_DIST_ID`  | your distribution ID  |

**12b. Point the frontend at your real API.** Open `frontend/index.html` and find
the placeholder API URL used for the upload request. Replace it with your real `presigned_url_api_endpoint` from Step 11, appending the route, e.g.:

```
https://xxxxxxxx.execute-api.us-east-1.amazonaws.com/generate-presigned-url
```

**12c. Re-push to deploy the frontend:**

```bash
git add frontend/index.html
git commit -m "Set real API endpoint in frontend"
git push origin main
```

This time the **Deploy Frontend** workflow should run cleanly: it syncs `frontend/` to your S3 bucket and invalidates the CloudFront cache.

**Access the site via CloudFront, not the raw S3 URL.** Use the `cloudfront_url`
output (e.g. `https://dxxxxxxxxxxxxx.cloudfront.net`) as your entry point —
not `https://<bucket>.s3.<region>.amazonaws.com`. Testing from the raw S3 URL can
surface CORS/permission behavior that doesn't reflect the real deployment.

---

## 13. Test the full flow

1. Open `https://<your-cloudfront_url>` in a browser.
2. (Optional) In AWS Console → **API Gateway** → your API → **Throttling**, set the
   default-stage **Burst = 100**, **Rate = 200**, save, wait ~2 min. This avoids the
   default very-low throttle blocking your first upload.
3. On the site, fill in the form and **upload a JPG / PNG / PDF**.
4. Verify it landed: AWS Console → **S3** → your upload bucket → you should see a
   new object named `<uuid>_<filename>`.
5. Check the processing Lambda's logs: AWS Console → **Lambda** →
   `process-uploaded-file` → **Monitor** → **View CloudWatch logs**. You should see
   "New file uploaded" and "SNS notification sent successfully."
6. Check your **email** — you should receive the "New File Uploaded" SNS alert.

> The `process-uploaded-file` Lambda only accepts `.jpg`, `.jpeg`, `.png`, `.pdf`.
> Other extensions are logged and rejected by design.

---

## 14. Known deployment issues & fixes

These two issues aren't hypothetical — they were hit in a real deployment of this
repo and are documented here so future users don't have to re-diagnose them.

### 14.1 — `terraform apply` fails: `PutBucketPolicy ... AccessDenied ... BlockPublicPolicy`

**Symptom:**

```
Error: putting S3 Bucket (frontendbucket-xxxxxxx) Policy: operation error S3: PutBucketPolicy,
https response error StatusCode: 403, ...
api error AccessDenied: User: arn:aws:iam::***:user/user1 is not authorized to perform:
s3:PutBucketPolicy on resource: "arn:aws:s3:::frontendbucket-xxxxxxx" because public policies
are prevented by the BlockPublicPolicy setting in S3 Block Public Access.

  with aws_s3_bucket_policy.frontend_bucket_policy,
  on main.tf line 126, in resource "aws_s3_bucket_policy" "frontend_bucket_policy":
```

**Cause:** AWS accounts have account-level **S3 Block Public Access** enabled by
default. It overrides bucket policies that grant public access — which is exactly
what `aws_s3_bucket_policy.frontend_bucket_policy` does, since the static frontend
needs to be publicly readable. This has nothing to do with your IAM user's
permissions, even with `AdministratorAccess`.

**Immediate fix — disable Block Public Access on the frontend bucket:**

```bash
aws s3api put-public-access-block \
  --bucket <YOUR_FRONTEND_BUCKET_NAME> \
  --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false
```

Then re-run the failed GitHub Actions workflow (Actions tab → failed run → **Re-run all jobs**).

**Permanent fix — manage it in Terraform** so it's not a manual, easy-to-forget step.
Add this to `infra/terraform/main.tf`, and make the bucket policy depend on it:

```hcl
resource "aws_s3_bucket_public_access_block" "frontend_bucket_pab" {
  bucket = aws_s3_bucket.frontend_bucket.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_policy" "frontend_bucket_policy" {
  # ...existing config...
  depends_on = [aws_s3_bucket_public_access_block.frontend_bucket_pab]
}
```

### 14.2 — Upload fails: `Failed to fetch` / CORS error on the S3 **upload** bucket

**Symptom (browser console):**

```
Access to XMLHttpRequest at 'https://<upload-bucket>.s3.amazonaws.com/<uuid>_<filename>?...'
from origin 'https://<frontend-bucket-or-cloudfront>' has been blocked by CORS policy:
Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin'
header is present on the requested resource.
```

This appears **after** the presigned-URL API call already succeeded — i.e. the
frontend got a valid signed URL back, but the direct `PUT` of the file to S3 is
what fails.

**Cause:** the repo's Terraform only attaches a CORS configuration to the
**frontend** bucket (`aws_s3_bucket_cors_configuration` for `frontend_bucket`).
The **upload** bucket has no CORS rule at all, so the browser blocks the
cross-origin `PUT` request that uploads the file.

**Immediate fix — add CORS to the upload bucket directly:**

```bash
aws s3api put-bucket-cors \
  --bucket <YOUR_UPLOAD_BUCKET_NAME> \
  --cors-configuration '{
    "CORSRules": [
      {
        "AllowedHeaders": ["*"],
        "AllowedMethods": ["PUT", "GET", "HEAD"],
        "AllowedOrigins": ["*"],
        "ExposeHeaders": [],
        "MaxAgeSeconds": 3000
      }
    ]
  }' --no-cli-pager

# verify it stuck:
aws s3api get-bucket-cors --bucket <YOUR_UPLOAD_BUCKET_NAME> --no-cli-pager
```

**Permanent fix — add the same rule in Terraform** so a future `terraform apply`
doesn't silently remove the manual fix (Terraform will otherwise treat the bucket's
CORS config as "not managed" and leave it as-is, but if the bucket is ever
recreated, the manual rule is lost). Add to `infra/terraform/main.tf`:

```hcl
resource "aws_s3_bucket_cors_configuration" "upload_cors" {
  bucket = aws_s3_bucket.upload_bucket.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["PUT", "GET", "HEAD"]
    allowed_origins = ["*"]
    expose_headers  = []
    max_age_seconds = 3000
  }
}
```

Commit and push so CI/CD applies it:

```bash
git add infra/terraform/main.tf
git commit -m "Add CORS config to upload bucket"
git push origin main
```

**After applying either fix:** hard-refresh the browser (**Cmd/Ctrl + Shift + R**)
before retrying — browsers cache both the page and failed CORS preflights.

### 14.3 — General CORS debugging approach

If you hit a *different* `Failed to fetch` / CORS error than the two above, the
upload flow makes **two** separate browser→cloud calls, and either can fail:

1. **`POST .../generate-presigned-url`** (the API call) — test the preflight
   directly with curl, which bypasses CORS and shows the raw HTTP status:

   ```bash
   curl -i -X OPTIONS https://<API_ID>.execute-api.us-east-1.amazonaws.com/generate-presigned-url \
     -H "Origin: https://<your-cloudfront-domain>" \
     -H "Access-Control-Request-Method: POST" \
     -H "Access-Control-Request-Headers: content-type" --no-cli-pager
   ```

   - `429` → API Gateway throttle is near-zero. Raise it:
     ```bash
     API_ID=$(aws apigatewayv2 get-apis \
       --query "Items[?Name=='DevOps-Accelerator-Presign-API'].ApiId" \
       --output text --no-cli-pager)

     aws apigatewayv2 update-stage \
       --api-id "$API_ID" \
       --stage-name '$default' \
       --default-route-settings ThrottlingBurstLimit=100,ThrottlingRateLimit=200 \
       --no-cli-pager
     ```
   - `404` / no `access-control-allow-*` headers → CORS isn't applied to the API:
     ```bash
     aws apigatewayv2 update-api \
       --api-id "$API_ID" \
       --cors-configuration '{"AllowOrigins":["*"],"AllowMethods":["GET","POST","OPTIONS"],"AllowHeaders":["*"]}' \
       --no-cli-pager
     ```
   - `access-control-allow-origin` appears **twice** → both API Gateway and the
     Lambda are setting CORS headers. Keep it on one layer only.

2. **`PUT` to the upload bucket** — see Section 14.2 above.

> ⚠️ "Blocked by CORS policy" in the browser console does **not** always mean CORS
> itself is misconfigured — it's often a non-2xx status (e.g. a 429 throttle) that
> the browser reports *as* a CORS failure. Always check the real status with curl
> before assuming the CORS config is wrong.

---

## 15. Teardown (do this to stop charges)

Because applies run through CI/CD, the simplest clean teardown is locally with the
same backend, then delete the manual backend resources:

```bash
cd infra/terraform
terraform destroy        # type 'yes' to confirm
cd ../..

# Then remove the manually-created backend resources:
aws s3 rb s3://myname-devops-acc-tfstate-7x9k --force
aws dynamodb delete-table --table-name devops-accelerator-tf-locker --region us-east-1
```

> If `destroy` complains about non-empty S3 buckets, the upload/frontend buckets use `force_destroy = true`, so Terraform should empty them automatically. If a bucket
> still blocks, empty it in the console and re-run destroy.

---

## 16. Quick troubleshooting reference

| Symptom                                                                        | Likely cause / fix                                                                                                                  |
| -------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `BucketAlreadyExists` / `BucketAlreadyOwnedByYou` on apply                     | Bucket name not globally unique — change it in `tfvars` (and state bucket in `main.tf`).                                            |
| `terraform init`: "Backend configuration changed" / provider checksum mismatch | Stale `.terraform/` shipped in the repo — `rm -rf .terraform .terraform.lock.hcl` then `terraform init -reconfigure`.               |
| `terraform init` fails on backend                                              | State bucket / DynamoDB table from Step 6 missing or names don't match `main.tf`.                                                   |
| `apply` fails: `PutBucketPolicy ... AccessDenied ... BlockPublicPolicy`        | Account-level S3 Block Public Access is on — see **Section 14.1**.                                                                    |
| Frontend workflow fails at "Invalidate CloudFront"                             | `CLOUDFRONT_DIST_ID` secret not set yet (Step 12a).                                                                                 |
| `Upload failed: Failed to fetch` on the presign API call                       | See **Section 14.3(1)** — most often a 429 throttle or missing CORS on the API.                                                      |
| `Failed to fetch` / CORS error on the S3 `PUT` (after presign succeeded)       | Missing CORS on the **upload bucket** — see **Section 14.2**.                                                                         |
| Preflight returns `429` (looks like a CORS error)                              | API Gateway throttle set to 0 — raise burst/rate (Section 14.3, Case A).                                                              |
| No email received                                                              | SNS subscription not confirmed (Step 11), or wrong `notification_email` in `tfvars`.                                                |
| Lambda not triggering on upload                                                | Confirm the S3 event notification + `aws_lambda_permission.allow_s3` applied; check the upload bucket is the one Terraform created. |
| Push rejected: file too large                                                  | `.terraform/` got committed — see Step 10 troubleshooting.                                                                          |

---

## 17. What you'll have learned

By the end you've practiced the core DevOps loop end to end: **IaC** (Terraform with
a remote, locked backend), **serverless app design** (API Gateway + two Lambdas +
S3 events + SNS), **CI/CD** (three independent GitHub Actions pipelines triggered by
path-scoped pushes), and **observability** (CloudWatch logs + email alerting) — all
wired together so a single `git push` ships infrastructure, backend, and frontend.
Debugging the two real-world issues in Section 14 is also good practice in reading
raw AWS error messages and browser network traces rather than guessing.

*Happy DevOps-ing!*
