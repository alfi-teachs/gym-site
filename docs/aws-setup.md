# AWS Setup

All resources were created in the AWS Console in region **ap-south-1 (Mumbai)**.
Replace the placeholders below with your own values:

| Placeholder | Example |
|---|---|
| `<ACCOUNT_ID>` | 12-digit AWS account ID |
| `<BUCKET_NAME>` | `gym-site-1` |
| `<DISTRIBUTION_ID>` | `E2UNHZ1KSDAVOK` |
| `<GITHUB_OWNER>` | `alfi-teachs` |

Order matters: each step depends on the one before it.

---

## 0. Billing alert

**Billing and Cost Management → Budgets → Create budget → Zero spend budget**, add an email, create.
This sends an alert as soon as anything starts costing money.

---

## 1. S3 bucket

**S3 → Create bucket**

| Setting | Value | Reason |
|---|---|---|
| Bucket name | `<BUCKET_NAME>` | Must be globally unique |
| Region | ap-south-1 | Closest to target users |
| Object Ownership | ACLs disabled (Bucket owner enforced) | AWS best practice; access controlled by policy only |
| Block all public access | **On** | Bucket is never public — CloudFront reads it via OAC |
| Bucket Versioning | Enabled | Roll back after a bad deploy |
| Default encryption | SSE-S3 | Encryption at rest (SSE-KMS would need extra KMS permissions for OAC) |

No files need to be uploaded manually — the pipeline does it. (During the build, the contents of `site/` were uploaded once by hand to verify CloudFront before adding the pipeline.)

> **Note:** S3 has no real folders. `images/logo.png` is a single object whose key contains a `/`. The console just displays prefixes as folders.

---

## 2. CloudFront distribution

**CloudFront → Create distribution**

| Setting | Value |
|---|---|
| Origin domain | `<BUCKET_NAME>.s3.ap-south-1.amazonaws.com` (the REST endpoint — **not** `s3-website`) |
| Origin access | **Origin access control settings** → create new OAC, signing behavior **Sign requests** |
| Viewer protocol policy | Redirect HTTP to HTTPS |
| Allowed methods | GET, HEAD |
| Default root object | `index.html` |
| WAF | Disabled (costs money; not needed for this project) |

After creation, note the **Distribution ID** and **distribution domain name** (`dxxxx.cloudfront.net`).

### Bucket policy

CloudFront offers to update the bucket policy automatically. If it doesn't, add this under **S3 → bucket → Permissions → Bucket policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontRead",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<BUCKET_NAME>/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
        }
      }
    }
  ]
}
```

This allows **only this distribution** to read objects. Anyone hitting S3 directly gets `AccessDenied`.

**Verify:** open `https://dxxxx.cloudfront.net/` — the site should load.

---

## 3. GitHub OIDC identity provider

**IAM → Identity providers → Add provider**

| Setting | Value |
|---|---|
| Provider type | OpenID Connect |
| Provider URL | `https://token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |

This tells AWS to trust tokens signed by GitHub Actions. It is created once per AWS account.

---

## 4. IAM deploy role

**IAM → Roles → Create role → Web identity**, select the GitHub provider and audience `sts.amazonaws.com`, name it `gym-site-github-deploy`.

### Trust policy

Who is allowed to assume the role. **IAM → Roles → gym-site-github-deploy → Trust relationships → Edit**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<GITHUB_OWNER>@<OWNER_ID>/gym-site@<REPO_ID>:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

> GitHub may send the subject claim in its **immutable-ID format** (`repo:owner@ownerId/repo@repoId:...`) instead of `repo:owner/repo:...`. The `sub` value must match exactly what GitHub sends — see [troubleshooting](troubleshooting.md#3-not-authorized-to-perform-stsassumerolewithwebidentity) for how to read it.

### Permissions policy

What the role can do. **Add permissions → Create inline policy → JSON**, name it `deploy-site`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListBucket",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::<BUCKET_NAME>"
    },
    {
      "Sid": "ManageObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::<BUCKET_NAME>/*"
    },
    {
      "Sid": "CloudFront",
      "Effect": "Allow",
      "Action": ["cloudfront:CreateInvalidation", "cloudfront:GetDistribution"],
      "Resource": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
    }
  ]
}
```

> **Bucket vs object ARNs:** `s3:ListBucket` acts on the bucket (`arn:aws:s3:::bucket`). `GetObject` / `PutObject` / `DeleteObject` act on objects (`arn:aws:s3:::bucket/*`). Mixing them up causes `AccessDenied` on `aws s3 sync`.

Copy the **role ARN** — it goes into GitHub next.

---

Next: **[GitHub setup](github-setup.md)**
