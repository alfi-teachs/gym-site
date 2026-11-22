# Troubleshooting

Real issues hit while building this project, how each was diagnosed, and the fix.

The general approach: **verify one layer at a time** (S3 → CloudFront → IAM → pipeline). The site was uploaded by hand and confirmed working through CloudFront *before* the pipeline was added, so every later failure could only be in IAM or the workflow.

---

## 1. CloudFront returns `AccessDenied` (403)

**Symptom:** the CloudFront URL shows an XML `AccessDenied` error.

**Diagnosis:** with a private bucket, S3 returns **403 instead of 404** when an object doesn't exist, because the caller isn't allowed to list the bucket. So a 403 can mean either "not permitted" or "not there".

Checked in order:

| Cause | Check | Fix |
|---|---|---|
| Bucket empty / files in a subfolder | S3 → Objects: is `index.html` at the root? | Upload the *contents* of `site/`, not the folder |
| Default root object missing | Does `/index.html` work but `/` doesn't? | CloudFront → General → Default root object = `index.html` |
| Bucket policy missing or wrong ARN | Policy must name this distribution's ARN | Replace with the policy in [AWS setup](aws-setup.md#bucket-policy) |
| Wrong or non-signing OAC attached | Origin → Origin access control; OAC → Signing behavior | Attach an OAC set to **Sign requests** |
| Error response cached | — | Test in incognito or create an invalidation for `/*` |

**What it actually was:** first an empty bucket, then a missing default root object (`/index.html` loaded, `/` didn't). Duplicate OACs had also been created; the one attached was replaced with a correctly configured one.

---

## 2. `Input required and not supplied: aws-region`

**Symptom:** the `configure-aws-credentials` step fails immediately.

**Diagnosis:** the step log showed **neither** `role-to-assume` nor `aws-region` under `with:` — so every `vars.*` value was resolving to empty.

**Cause:** all four values had been saved as **one** variable (`GYMSITE`) containing the whole block of text. The workflow looks up four separately named variables.

**Fix:** delete it and create `AWS_REGION`, `S3_BUCKET`, `CLOUDFRONT_DISTRIBUTION_ID` and `AWS_ROLE_ARN` as separate variables.

**Follow-up:** the next run failed with `Could not load credentials from any providers` — `AWS_ROLE_ARN` had been missed. With no role to assume, the action fell back to looking for static credentials and found none.

---

## 3. `Not authorized to perform sts:AssumeRoleWithWebIdentity`

**Symptom:** OIDC role assumption fails, even though:
- the OIDC provider existed with audience `sts.amazonaws.com`
- the role ARN variable matched exactly
- the trust policy allowed `repo:alfi-teachs/gym-site:ref:refs/heads/main`

**Diagnosis:** instead of guessing, a temporary step printed the claims GitHub actually puts in the token:

```yaml
- name: Debug OIDC claims
  uses: actions/github-script@v7
  with:
    script: |
      const token = await core.getIDToken('sts.amazonaws.com');
      const claims = JSON.parse(Buffer.from(token.split('.')[1], 'base64').toString());
      core.info(`sub: ${claims.sub}`);
      core.info(`aud: ${claims.aud}`);
```

Output:

```
sub: repo:alfi-teachs@262962106/gym-site@1462281036:ref:refs/heads/main
aud: sts.amazonaws.com
```

**Cause:** GitHub was sending the subject claim in its **immutable-ID format** — owner and repository names suffixed with their numeric IDs. The trust policy's `StringEquals` condition expected the plain `repo:owner/repo` format, so the strings never matched.

**Fix:** set the trust policy's `sub` condition to the exact value GitHub sends. The debug step was then removed (it prints only the decoded claims, never the token, but has no reason to stay).

**Why this format is better:** repository names can be reused. If the repo were deleted and someone created a new one with the same name, a name-based trust policy would accept it. A new repo gets a new numeric ID, so the immutable subject rejects it.

---

## 4. Uploading folders to S3 shows individual files

**Symptom:** uploading a folder in the S3 console lists every file separately.

**Cause:** not a problem — S3 is flat object storage. `images/logo.png` is one object whose key contains `/`. The upload screen lists each key; the Objects tab groups keys by prefix and displays them as folders.

**Real gotcha:** uploading from *inside a `.zip`* in File Explorer does flatten the structure. Extract the archive first.

---

## 5. `s3:ListBucket` permission and `aws s3 sync`

**Mistake caught in review:** the deploy policy originally granted `s3:ListBucket` on `arn:aws:s3:::bucket/*`.

**Why it's wrong:** `ListBucket` is a bucket-level action and needs `arn:aws:s3:::bucket`. Object actions (`GetObject`, `PutObject`, `DeleteObject`) need `arn:aws:s3:::bucket/*`. `aws s3 sync` calls `ListObjectsV2` to compare files, so the wrong ARN causes `AccessDenied`.
