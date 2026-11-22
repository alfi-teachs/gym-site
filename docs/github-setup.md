# GitHub Setup

Prerequisite: everything in **[AWS setup](aws-setup.md)** is done.

---

## 1. Local project and repository

```bash
# project layout: website files live in site/
mkdir -p gym-site/site gym-site/.github/workflows
cd gym-site
# copy index.html, contact.html, trainer.html, why.html, css/, js/, images/ into site/

git init
git add .
git commit -m "Initial commit: gym website"
git branch -M main
```

Create an **empty public** repository named `gym-site` on GitHub (no README, .gitignore or license — those cause a conflict on the first push), then:

```bash
git remote add origin https://github.com/<GITHUB_OWNER>/gym-site.git
git push -u origin main
```

> **Tip:** Keep Git repositories out of OneDrive/Dropbox-synced folders, or pause syncing while running Git. Sync clients can lock or corrupt files inside `.git`.

---

## 2. Actions variables

**Repo → Settings → Secrets and variables → Actions → Variables tab → New repository variable.**
Create **four separate variables** (one name, one value each):

| Name | Value |
|---|---|
| `AWS_REGION` | `ap-south-1` |
| `S3_BUCKET` | `<BUCKET_NAME>` |
| `CLOUDFRONT_DISTRIBUTION_ID` | `<DISTRIBUTION_ID>` |
| `AWS_ROLE_ARN` | `arn:aws:iam::<ACCOUNT_ID>:role/gym-site-github-deploy` |

**Why variables, not secrets?** None of these values are sensitive — the role ARN is useless without passing the OIDC trust policy. Secrets are for credentials, and thanks to OIDC this project has none.

**Why variables, not hardcoded values?** Configuration stays separate from code: if the bucket or distribution changes, only the variable changes, and the same workflow could deploy to other environments.

---

## 3. Deploy workflow

File: [`.github/workflows/deploy.yml`](../.github/workflows/deploy.yml)

| Part | Purpose |
|---|---|
| `on: push` to `main`, paths `site/**` | Deploy only when site files (or the workflow) change |
| `workflow_dispatch` | Allow manual runs from the Actions tab |
| `permissions: id-token: write` | Lets the job request an OIDC token from GitHub — **required** |
| `concurrency` | Prevents two deploys running at once |
| `configure-aws-credentials` | Exchanges the OIDC token for temporary AWS credentials |
| `aws s3 sync site/ ... --delete` | Uploads changed files and removes deleted ones |
| `aws cloudfront create-invalidation --paths "/*"` | Clears the CDN cache so changes appear immediately |
| Smoke test | Fails the pipeline if the live site doesn't return HTTP 200 |

---

## 4. Deploy and verify

```bash
# edit something visible in site/index.html
git add .
git commit -m "Update homepage"
git push
```

1. **Actions** tab → **Deploy site** → every step green; the smoke test prints `returned HTTP 200`.
2. Open the CloudFront URL in an incognito window — the change is live.

To re-run a failed deploy without a new commit: open the run → **Re-run all jobs**.

---

If something fails, see **[Troubleshooting](troubleshooting.md)**.
