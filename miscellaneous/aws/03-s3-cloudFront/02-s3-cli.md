# AWS S3 & CloudFront — Command Reference

Practical reference for manipulating S3 buckets from the CLI, plus CloudFront
commands — the two come paired together whenever S3 serves as a CDN origin.


## Bucket creation & listing

```bash
# Create a bucket
aws s3 mb s3://<bucket-name> --region <region>

# List all buckets in the account
aws s3 ls

# List a bucket's contents
aws s3 ls s3://<bucket-name>
aws s3 ls s3://<bucket-name> --recursive   # include contents of all "subfolders"
```

*Example:*
```bash
aws s3 mb s3://my-app-frontend --region us-east-1

aws s3 ls s3://my-app-frontend --profile my-profile
aws s3 ls s3://my-app-media --recursive --profile my-profile
```


## Public access control

```bash
# Block all public access (default/safe posture — the "bouncer")
aws s3api put-public-access-block \
  --bucket <bucket-name> \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Allow public access to pass through (required before a public bucket policy can take effect)
aws s3api put-public-access-block \
  --bucket <bucket-name> \
  --public-access-block-configuration \
  BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false

# Check current public access block settings
aws s3api get-public-access-block --bucket <bucket-name>
```

*Example (bucket kept private — access happens only via CloudFront OAC):*
```bash
aws s3api put-public-access-block \
  --bucket my-app-frontend \
  --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

## Static website hosting

```bash
# Check the website hosting configuration of a bucket
aws s3api get-bucket-website --bucket <bucket-name>

# Enable static website hosting (index/error document) — same result as
# the console's "Static website hosting" screen
aws s3api put-bucket-website --bucket <bucket-name> --website-configuration '{
  "IndexDocument": {"Suffix": "index.html"},
  "ErrorDocument": {"Key": "index.html"}
}'
```

Resulting URL pattern: `http://<bucket-name>.s3-website-<region>.amazonaws.com`

*Example (plain S3 website hosting, no CloudFront):*
```bash
aws s3api get-bucket-website --bucket my-app-frontend --profile my-profile
# → resulting URL: http://my-app-frontend.s3-website-us-east-1.amazonaws.com
```


## Bucket policy

```bash
# View the current bucket policy
aws s3api get-bucket-policy --bucket <bucket-name>

# Attach/replace a bucket policy from a local JSON file
aws s3api put-bucket-policy --bucket <bucket-name> --policy file://policy.json

# Remove a bucket policy entirely
aws s3api delete-bucket-policy --bucket <bucket-name>
```

## Syncing files (deployment)

```bash
# Upload/update changed files only (compares local vs bucket)
aws s3 sync <local-folder>/ s3://<bucket-name>/

# Same, but also delete from the bucket anything no longer present locally
# (prevents old build artifacts from accumulating)
aws s3 sync <local-folder>/ s3://<bucket-name>/ --delete

# Copy a single file
aws s3 cp <local-file> s3://<bucket-name>/<key>

# Copy a whole folder (does NOT delete removed files — prefer sync for deployments)
aws s3 cp <local-folder>/ s3://<bucket-name>/ --recursive
```

**Typical full deployment sequence** (build a static frontend, then ship it):

```bash
cd <project-folder>
npm run build
aws s3 sync dist/ s3://<bucket-name>/ --delete --profile <profile-name>
```

*Example:*
```bash
cd ~/projects/my-app/ui
npm run build
aws s3 sync dist/ s3://my-app-frontend --delete --profile my-profile
```


## Object management

```bash
# Delete a single object
aws s3 rm s3://<bucket-name>/<key>

# Delete every object under a prefix
aws s3 rm s3://<bucket-name>/<prefix>/ --recursive

# Delete an entire bucket AND all its contents (irreversible)
aws s3 rb s3://<bucket-name> --force
```


## CloudFront

```bash
# List all distributions (id + domain name)
aws cloudfront list-distributions \
  --query "DistributionList.Items[*].[Id,DomainName]" \
  --output table

# List with aliases (custom domains) included
aws cloudfront list-distributions \
  --query "DistributionList.Items[].{Id:Id,Domain:DomainName,Aliases:Aliases.Items}" \
  --output table

# Invalidate the cache after a deployment (mandatory — otherwise CloudFront
# keeps serving the previous version)
aws cloudfront create-invalidation --distribution-id <distribution-id> --paths "/*"
```

**Full redeploy sequence for a static site behind CloudFront:**

```bash
npm run build
aws s3 sync dist/ s3://<bucket-name>/ --delete --profile <profile-name>
aws cloudfront create-invalidation --distribution-id <distribution-id> --paths "/*" --profile <profile-name>
```

*Example:*
```bash
cd ~/projects/my-app/ui
npm run build
aws s3 sync dist/ s3://my-app-frontend --delete --profile my-profile
aws cloudfront create-invalidation --distribution-id E1A2B3C4D5E6F7 --paths "/*" --profile my-profile
```


## Identity check (relevant before any deploy/sync)

```bash
aws sts get-caller-identity --profile <profile-name>
```

*Example:*
```bash
aws sts get-caller-identity --profile my-profile
```


## Annex — The 3 common ways to host a static site with S3

**1. S3 Static Website Hosting alone**

- "Static website hosting" enabled on the bucket, with `index.html` and `error.html` defined.
- Served through the website endpoint: `http://<bucket>.s3-website-<region>.amazonaws.com`.
- The bucket must be public: Block Public Access disabled, plus a bucket policy granting `s3:GetObject`.
- HTTP only — no HTTPS.
- Handles redirects and index-in-subfolder resolution natively.

**2. S3 + CloudFront (the recommended production approach)**

- The bucket stays private. CloudFront reaches it via OAC (Origin Access Control, which replaced the older OAI).
- Gives HTTPS, a custom domain (ACM certificate required in us-east-1), caching, and a global distribution.
- The origin is the bucket's REST endpoint, not the website endpoint — so index-in-subfolder resolution doesn't work out of the box; a CloudFront Function is needed if that behavior is required.
- Variant: using the website endpoint as a custom origin instead. This keeps S3's native redirect handling, but the bucket must then be public and OAC no longer applies.

**3. AWS Amplify Hosting**

- Managed S3 + CloudFront under the hood, with CI/CD wired directly to a Git repo.
- Simplest option for a frontend (Next.js, React...), at the cost of less manual control.

**Deploying the files themselves** — several valid options:

```bash
aws s3 sync ./build s3://<bucket-name> --delete
```

Then, if serving through CloudFront:

```bash
aws cloudfront create-invalidation --distribution-id <distribution-id> --paths "/*"
```

Other valid options: manual upload via the console, a CI/CD pipeline (CodePipeline, GitHub Actions), or Infrastructure as Code (CloudFormation/SAM/CDK).


## References

- [aws s3 command reference](https://docs.aws.amazon.com/cli/latest/reference/s3/index.html)
- [aws s3api command reference](https://docs.aws.amazon.com/cli/latest/reference/s3api/index.html)
- [S3 static website hosting](https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html)
- [CloudFront CLI reference](https://docs.aws.amazon.com/cli/latest/reference/cloudfront/index.html)