# Lab 04: CloudFront HTTPS Static Website

## Lab Summary

In this lab, I upgraded my S3 static website from Lab 1C by placing Amazon CloudFront in front of it. This added HTTPS support, global content delivery, and a more secure architecture where the S3 bucket is no longer publicly accessible.

I created an Origin Access Control, deployed a CloudFront distribution, updated the S3 bucket policy so only CloudFront can read the website files, re-enabled S3 Block Public Access, and tested that the CloudFront HTTPS URL works while the old S3 website endpoint is blocked.

## Source Lab

- Repository: AICloudFusion
- Original lab: Side Quest Lab 1D — HTTPS for Your Static Website with CloudFront
- Session: 1 — Cloud Concepts
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Confirm the S3 static website bucket from Lab 1C still exists
- Create an Origin Access Control
- Create a CloudFront distribution
- Wait for CloudFront deployment
- Remove the old public S3 bucket policy
- Add a CloudFront-only S3 bucket policy
- Re-enable S3 Block Public Access
- Verify the CloudFront distribution in the AWS Console
- Test HTTPS access through CloudFront
- Test HTTP-to-HTTPS redirect
- Confirm the old S3 website URL is blocked
- Test the custom error page
- Understand what is needed for a custom domain
- Clean up CloudFront resources if not keeping the site live

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| Amazon S3 | Stores the static website files |
| Amazon CloudFront | Provides CDN caching and HTTPS access |
| Origin Access Control | Allows CloudFront to securely access a private S3 bucket |
| S3 Bucket Policy | Allows only the CloudFront distribution to read website files |
| S3 Block Public Access | Makes the S3 bucket private again |
| AWS CLI | Creates and configures CloudFront and S3 resources |
| PowerShell | Terminal used to run AWS CLI commands |
| Web Browser | Used to test HTTPS, redirect behavior, and error pages |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- Completed Lab 03: S3 Static Website
- Existing S3 bucket from Lab 1C
- `index.html` and `error.html` uploaded to the S3 bucket
- AWS CLI installed and configured
- AWS CLI authentication working
- AWS account ID available
- S3 bucket region known, such as `us-east-1`

## Cost Notice

Estimated cost: `$0.00`

Notes:

- CloudFront has a generous free tier for small portfolio-style websites.
- S3 storage should already be within the free tier from Lab 1C.
- Cleanup is optional if I want to keep the HTTPS website live.
- If I clean up, CloudFront distributions must be disabled before they can be deleted.

## Key Concepts

| Concept | Meaning |
|---|---|
| CloudFront | AWS content delivery network that caches and serves content from edge locations. |
| CDN | Content Delivery Network. It improves performance by serving content closer to users. |
| HTTPS | Encrypted HTTP traffic using TLS. |
| TLS Certificate | Certificate used to secure HTTPS traffic. CloudFront provides one for the default `cloudfront.net` domain. |
| Origin | The backend source CloudFront retrieves files from. In this lab, the S3 bucket is the origin. |
| Origin Access Control | Secure method that lets CloudFront sign requests to S3 using SigV4. |
| OAI | Legacy CloudFront-to-S3 access method. OAC is the newer recommended method. |
| Viewer Protocol Policy | Controls whether CloudFront redirects HTTP users to HTTPS. |
| Price Class | Controls which CloudFront edge locations serve content. |
| Cache Policy | Controls how CloudFront caches content. |
| Custom Error Response | Lets CloudFront show `error.html` instead of raw S3 access errors. |

## Security Notes

| Topic | Explanation |
|---|---|
| S3 bucket privacy | The S3 bucket becomes private again after CloudFront is configured. |
| CloudFront-only access | The bucket policy only allows requests from the specific CloudFront distribution. |
| OAC over OAI | OAC is the newer recommended approach for securing S3 origins behind CloudFront. |
| HTTPS | Visitors access the site securely using the CloudFront HTTPS URL. |
| Old S3 URL blocked | Direct access to the old S3 website endpoint should no longer work. |
| Public bucket removed | This is more secure than leaving the S3 bucket publicly readable. |

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my current AWS identity.
- Recorded my AWS account ID.

**Command used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

```json
{
  "UserId": "REDACTED",
  "Account": "REDACTED",
  "Arn": "REDACTED"
}
```

**Result:**

- AWS CLI returned my active identity successfully.

**Notes:**

- If the SSO token is expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

- Created a local folder for this lab.
- Changed into the folder before creating JSON files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-1d
cd ~\Desktop\workshop-lab-1d
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-1d
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-1d`.

---

### Step 3: Confirm S3 Bucket Still Exists

**What I did:**

- Checked the S3 bucket from Lab 1C.
- Confirmed that `index.html` and `error.html` were still uploaded.

**Command used:**

```bash
aws s3 ls s3://<YOUR_BUCKET_NAME>
```

**Expected result:**

```text
index.html
error.html
```

**Result:**

- The Lab 1C website files were present in the S3 bucket.

**Notes:**

- This lab depends on the Lab 1C S3 bucket still existing.
- If the bucket does not exist, Lab 1C must be recreated first.

---

### Step 4: Create Origin Access Control Config

**What I did:**

- Created a file named `oac-config.json`.
- Added the Origin Access Control configuration for S3.

**File created:**

```text
oac-config.json
```

**File content:**

```json
{
  "Name": "workshop-lab-1d-oac",
  "Description": "OAC for Lab 1D S3 static website",
  "SigningProtocol": "sigv4",
  "SigningBehavior": "always",
  "OriginAccessControlOriginType": "s3"
}
```

**Result:**

- OAC configuration file was created locally.

**Notes:**

- `SigningProtocol: sigv4` means CloudFront signs requests to S3 using AWS Signature Version 4.
- `SigningBehavior: always` means every request is signed.
- This lets S3 verify that the request came from CloudFront.

---

### Step 5: Create Origin Access Control

**What I did:**

- Created the OAC using the configuration file.

**Command used:**

```bash
aws cloudfront create-origin-access-control --origin-access-control-config file://oac-config.json
```

**Expected result:**

```json
{
  "OriginAccessControl": {
    "Id": "EABCDEF1234567"
  }
}
```

**Result:**

- Origin Access Control was created successfully.

**My OAC ID:**

```text
REDACTED_OR_RECORDED_HERE
```

**Notes:**

- The OAC ID is needed in the CloudFront distribution configuration.

---

### Step 6: Create CloudFront Distribution Config

**What I did:**

- Created a file named `distribution-config.json`.
- Added the CloudFront distribution configuration.
- Replaced placeholders with my bucket name, region, account ID, and OAC ID.

**File created:**

```text
distribution-config.json
```

**File content:**

```json
{
  "CallerReference": "workshop-lab-1d-dist",
  "DefaultRootObject": "index.html",
  "Comment": "Workshop Lab 1D: Static website with HTTPS",
  "Enabled": true,
  "HttpVersion": "http2",
  "PriceClass": "PriceClass_100",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "S3Origin",
        "DomainName": "<YOUR_BUCKET_NAME>.s3.<YOUR_REGION>.amazonaws.com",
        "OriginAccessControlId": "<YOUR_OAC_ID>",
        "S3OriginConfig": {
          "OriginAccessIdentity": ""
        }
      }
    ]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3Origin",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "AllowedMethods": {
      "Quantity": 2,
      "Items": ["GET", "HEAD"],
      "CachedMethods": {
        "Quantity": 2,
        "Items": ["GET", "HEAD"]
      }
    },
    "Compress": true
  },
  "CustomErrorResponses": {
    "Quantity": 1,
    "Items": [
      {
        "ErrorCode": 403,
        "ResponsePagePath": "/error.html",
        "ResponseCode": "404",
        "ErrorCachingMinTTL": 10
      }
    ]
  }
}
```

**Result:**

- Distribution configuration file was created locally.

**Notes:**

- `DefaultRootObject` tells CloudFront to serve `index.html` when visitors go to the root URL.
- `ViewerProtocolPolicy: redirect-to-https` redirects HTTP requests to HTTPS.
- `PriceClass_100` uses a lower-cost CloudFront edge location set.
- The custom error response maps S3 `403` responses to the custom `error.html` page.

---

### Step 7: Deploy CloudFront Distribution

**What I did:**

- Created the CloudFront distribution using the JSON file.

**Command used:**

```bash
aws cloudfront create-distribution --distribution-config file://distribution-config.json
```

**Expected result:**

```json
{
  "Distribution": {
    "Id": "EXXXXXXXXXXXXX",
    "DomainName": "dxxxxxxxxxxxxx.cloudfront.net",
    "Status": "InProgress"
  }
}
```

**Result:**

- CloudFront distribution was created successfully.

**My Distribution ID:**

```text
REDACTED_OR_RECORDED_HERE
```

**My Distribution Domain:**

```text
REDACTED_OR_RECORDED_HERE.cloudfront.net
```

**Notes:**

- The distribution starts as `InProgress`.
- CloudFront deployment can take 5–20 minutes.

---

### Step 8: Wait for CloudFront Deployment

**What I did:**

- Checked the distribution status until it changed to `Deployed`.

**Command used:**

```bash
aws cloudfront get-distribution --id <YOUR_DIST_ID> --query 'Distribution.Status' --output text
```

**Expected result after waiting:**

```text
Deployed
```

**Result:**

- CloudFront distribution eventually reached the `Deployed` state.

**Notes:**

- CloudFront must propagate the configuration globally.
- It is normal for deployment to take several minutes.

---

### Step 9: Remove Old Public S3 Bucket Policy

**What I did:**

- Removed the public-read bucket policy from Lab 1C.

**Command used:**

```bash
aws s3api delete-bucket-policy --bucket <YOUR_BUCKET_NAME>
```

**Expected result:**

```text
No output
```

**Result:**

- Old public bucket policy was deleted.

**Notes:**

- This prepares the bucket to be private and CloudFront-only.

---

### Step 10: Create CloudFront-Only Bucket Policy

**What I did:**

- Created a file named `cloudfront-bucket-policy.json`.
- Added a policy that allows only my CloudFront distribution to read objects from the S3 bucket.

**File created:**

```text
cloudfront-bucket-policy.json
```

**File content:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<YOUR_ACCOUNT_ID>:distribution/<YOUR_DIST_ID>"
        }
      }
    }
  ]
}
```

**Result:**

- CloudFront-only bucket policy file was created locally.

**Notes:**

- This policy allows `s3:GetObject` only when the request comes from my specific CloudFront distribution.
- Direct public S3 access should be blocked after this.

---

### Step 11: Apply CloudFront-Only Bucket Policy

**What I did:**

- Applied the new CloudFront-only bucket policy to the S3 bucket.

**Command used:**

```bash
aws s3api put-bucket-policy --bucket <YOUR_BUCKET_NAME> --policy file://cloudfront-bucket-policy.json
```

**Expected result:**

```text
No output
```

**Result:**

- New bucket policy was applied successfully.

---

### Step 12: Re-enable S3 Block Public Access

**What I did:**

- Re-enabled all S3 Block Public Access settings on the bucket.

**Command used:**

```bash
aws s3api put-public-access-block --bucket <YOUR_BUCKET_NAME> --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

**Expected result:**

```text
No output
```

**Result:**

- S3 Block Public Access was re-enabled.

**Notes:**

- The bucket is now private.
- CloudFront should be the only valid way to access the website.

---

### Step 13: Verify Setup in AWS Console

**What I did:**

- Opened the AWS Console.
- Checked the CloudFront distribution.
- Checked the S3 bucket permissions.

**CloudFront checks:**

- Distribution exists
- Distribution status is enabled/deployed
- Distribution domain name is visible
- Default root object is `index.html`
- Origin points to the S3 bucket
- Origin access uses the OAC
- Viewer protocol policy redirects HTTP to HTTPS

**S3 checks:**

- Block Public Access settings are on
- Bucket policy contains the CloudFront service principal
- Bucket policy references the CloudFront distribution ARN

**Result:**

- CloudFront and S3 configuration were verified.

---

### Step 14: Test HTTPS Website

**What I did:**

- Opened the CloudFront HTTPS URL in a browser.

**URL format:**

```text
https://<YOUR_DIST_DOMAIN>
```

**Expected result:**

- Website loads successfully.
- Browser shows HTTPS/padlock.

**Result:**

- The static website loaded over HTTPS.

---

### Step 15: Test HTTP Redirect to HTTPS

**What I did:**

- Opened the HTTP version of the CloudFront domain.

**URL format:**

```text
http://<YOUR_DIST_DOMAIN>
```

**Expected result:**

- Browser redirects to:

```text
https://<YOUR_DIST_DOMAIN>
```

**Result:**

- HTTP redirected to HTTPS successfully.

---

### Step 16: Test Old S3 Website URL Is Blocked

**What I did:**

- Tried to access the original S3 website endpoint from Lab 1C.

**URL format:**

```text
http://<YOUR_BUCKET_NAME>.s3-website-<YOUR_REGION>.amazonaws.com
```

**Expected result:**

```text
Access Denied
```

or:

```text
403 Forbidden
```

**Result:**

- The old S3 website URL was blocked.

**Notes:**

- This confirms the bucket is no longer publicly accessible.
- CloudFront is now the proper entry point.

---

### Step 17: Test Custom Error Page

**What I did:**

- Visited a page that does not exist through the CloudFront domain.

**URL format:**

```text
https://<YOUR_DIST_DOMAIN>/this-page-does-not-exist
```

**Expected result:**

- Custom `error.html` page appears.

**Result:**

- Custom error page loaded successfully.

**Notes:**

- Because the bucket is private, S3 may return `403` instead of `404`.
- CloudFront maps the `403` response to the custom error page.

---

### Step 18: Understand Custom Domains

**What I learned:**

To use a custom domain such as `www.myportfolio.com`, I would need:

- A registered domain name
- An ACM certificate in `us-east-1`
- The custom domain added as an Alternate Domain Name in CloudFront
- DNS configured to point the domain to CloudFront

**Notes:**

- This is outside the lab unless I already own a domain.
- CloudFront requires ACM certificates for custom domains to be created in `us-east-1`.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| Local project folder created | Passed |
| Existing S3 bucket confirmed | Passed |
| `index.html` and `error.html` confirmed | Passed |
| OAC config file created | Passed |
| Origin Access Control created | Passed |
| Distribution config file created | Passed |
| CloudFront distribution created | Passed |
| Distribution reached `Deployed` status | Passed |
| Old public bucket policy deleted | Passed |
| CloudFront-only bucket policy created | Passed |
| CloudFront-only bucket policy applied | Passed |
| S3 Block Public Access re-enabled | Passed |
| CloudFront console configuration verified | Passed |
| S3 bucket permissions verified | Passed |
| HTTPS CloudFront URL loaded | Passed |
| HTTP redirected to HTTPS | Passed |
| Old S3 website endpoint blocked | Passed |
| Custom error page worked | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| CloudFront `update-distribution` failed with `Expected: '=', received: 'ÿ'` | The `current-dist-config.json` file was saved as UTF-16LE because Windows PowerShell redirection created the file with that encoding. AWS CLI could not parse the hidden characters at the start of the JSON file. | Re-saved/recreated the distribution config as UTF-8 without BOM, then reran the CloudFront `update-distribution` command. |
| CloudFront `update-distribution` failed again with `Expected: '=', received: 'ï'` | The file was converted to UTF-8, but it still included a BOM marker. AWS CLI still could not parse the hidden BOM characters before the opening `{`. | Rewrote the JSON file as UTF-8 without BOM using PowerShell/.NET, then updated the distribution successfully. |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `NoSuchDistribution` | Distribution ID is wrong or does not exist | Run `aws cloudfront list-distributions --query 'DistributionList.Items[].{Id:Id,Domain:DomainName}'` |
| Website shows `AccessDenied` through CloudFront | Bucket policy may be wrong, OAC ID may be wrong, or distribution may not be deployed | Check bucket policy, OAC ID, and distribution status |
| Website shows stale content | CloudFront is serving cached content | Create an invalidation using `aws cloudfront create-invalidation --distribution-id <ID> --paths "/*"` |
| Error page shows XML instead of custom page | `error.html` may be missing or custom error response is not configured | Confirm `error.html` exists and review distribution config |
| Old S3 URL still works | Public access may not be fully blocked | Re-run the Block Public Access command and check bucket policy |
| Distribution stuck on `InProgress` | CloudFront propagation delay | Wait longer and check the CloudFront console |
| `MalformedXML` or `InvalidArgument` | JSON file has a syntax error or placeholder was not replaced | Recheck the JSON file carefully |
| `Expected: '=', received: 'ÿ'` when using `--distribution-config file://...` | The JSON file is likely saved as UTF-16LE | Rewrite the file as UTF-8 without BOM before running the AWS CLI command |
| `Expected: '=', received: 'ï'` when using `--distribution-config file://...` | The JSON file is likely saved as UTF-8 with BOM | Remove the BOM and save the file as UTF-8 without BOM |
| CloudFront config JSON looks correct but AWS CLI cannot parse it | The issue may be hidden file encoding characters, not the JSON content | Check the first bytes of the file and regenerate it as UTF-8 without BOM |

## Cleanup

### Optional Note

If I want to keep the HTTPS website live, I can skip cleanup. CloudFront should remain within the free tier for a small portfolio website.

If I want to remove the lab resources, CloudFront must be disabled before it can be deleted.

---

### Step 1: Get Current Distribution ETag

**Command used:**

```bash
aws cloudfront get-distribution-config --id <YOUR_DIST_ID> --query 'ETag' --output text
```

**Result:**

- ETag was retrieved.

---

### Step 2: Save Current Distribution Config

**Command used:**

```bash
aws cloudfront get-distribution-config --id <YOUR_DIST_ID> --query 'DistributionConfig' > current-dist-config.json
```

**Result:**

- Current distribution config was saved locally.

---

### Step 3: Disable Distribution

**What I did:**

- Opened `current-dist-config.json`.
- Changed:

```json
"Enabled": true
```

to:

```json
"Enabled": false
```

- Saved the file.

**Command used:**

```bash
aws cloudfront update-distribution --id <YOUR_DIST_ID> --distribution-config file://current-dist-config.json --if-match <ETAG>
```

**Result:**

- Distribution disable process did not start.

### Encoding Issue During Distribution Update (First Fix Attempt)

While disabling the CloudFront distribution, I ran into an AWS CLI parsing error when using the local `current-dist-config.json` file.

The first error showed:

```text
Expected: '=', received: 'ÿþ' (UTF-16LE)
```

**What I did:**
```bash
Get-Content .\current-dist-config.json | Set-Content .\current-dist-config-utf8.json -Encoding utf8
aws cloudfront update-distribution `
  --id <YOUR_DISTRIBUTION_ID> `
  --distribution-config file://current-dist-config-utf8.json `
  --if-match <YOUR_ETAG>
```

**Result:**
- Problem persisted with encoding problem.

### Encoding Issue During Distribution Update (Second Fix Attempt)

It was later found out that the first hidden character had changed again.

Similar to the first error, it showed:

```text
Expected: '=', received: 'ï»¿' (UTF-8 with BOM)
```

**What I did**
```bash
$raw = Get-Content .\current-dist-config.json -Raw
$raw = $raw.TrimStart([char]0xFEFF)

$utf8NoBom = New-Object System.Text.UTF8Encoding $false
[System.IO.File]::WriteAllText(
    (Join-Path (Get-Location) "current-dist-config-nobom.json"),
    $raw,
    $utf8NoBom
)
aws cloudfront update-distribution `
  --id <YOUR_DISTRIBUTION_ID> `
  --distribution-config file://current-dist-config-nobom.json `
  --if-match <YOUR_ETAG>
```

**Result**
- Problem was rectified.
- Distribution disable process started successfully.

---

### Step 4: Wait for Disabled Distribution to Deploy

**Command used:**

```bash
aws cloudfront get-distribution --id <YOUR_DIST_ID> --query 'Distribution.Status' --output text
```

**Expected result:**

```text
Deployed
```

**Notes:**

- The distribution must finish deploying the disabled state before deletion.

---

### Step 5: Delete Distribution

**Command used to get new ETag:**

```bash
aws cloudfront get-distribution --id <YOUR_DIST_ID> --query 'ETag' --output text
```

**Delete command:**

```bash
aws cloudfront delete-distribution --id <YOUR_DIST_ID> --if-match <NEW_ETAG>
```

**Result:**

- CloudFront distribution was deleted.

---

### Step 6: Delete Origin Access Control

**Command used to get OAC ETag:**

```bash
aws cloudfront get-origin-access-control --id <YOUR_OAC_ID> --query 'ETag' --output text
```

**Delete command:**

```bash
aws cloudfront delete-origin-access-control --id <YOUR_OAC_ID> --if-match <OAC_ETAG>
```

**Result:**

- Origin Access Control was deleted.

---

### Step 7: Delete Local Lab Folder

**PowerShell commands used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-1d
```

**Result:**

- Local lab folder was deleted.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| CloudFront distribution | Disabled and deleted | Yes |
| Origin Access Control | Deleted | Yes |
| Local lab folder | Deleted | Yes |
| S3 bucket | Kept from Lab 1C unless intentionally deleted | Yes |

## What I Learned

- CloudFront can add HTTPS to an S3-hosted static website.
- CloudFront is a CDN that serves content from edge locations closer to users.
- OAC is the modern recommended way to let CloudFront access a private S3 bucket.
- OAI is the older/legacy method, while OAC uses SigV4 request signing.
- The S3 bucket does not need to stay public when CloudFront is used.
- Bucket policies can restrict access to a specific CloudFront distribution.
- Viewer protocol policy can redirect HTTP traffic to HTTPS.
- CloudFront deployments take time because configuration must propagate globally.
- Custom domains require a registered domain, DNS records, and an ACM certificate in `us-east-1`.
- CloudFront resources require a careful cleanup process because distributions must be disabled before deletion.
- AWS CLI JSON input files must be saved in a format the CLI can parse correctly.
- Windows PowerShell redirection can create files with encoding markers that break AWS CLI JSON parsing.
- The errors `Expected: '=', received: 'ÿ'` and `Expected: '=', received: 'ï'` can indicate hidden encoding characters at the start of the file.
- UTF-16LE and UTF-8 with BOM can both cause parsing issues with AWS CLI input files.
- Saving the CloudFront distribution config as UTF-8 without BOM fixed the issue.
- The JSON content was valid; the problem was the file encoding.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/s3-bucket-files-confirmed.png` | Existing `index.html` and `error.html` confirmed in S3 |
| `screenshots/oac-created.png` | Origin Access Control created |
| `screenshots/distribution-created.png` | CloudFront distribution created |
| `screenshots/distribution-deployed.png` | CloudFront distribution reached deployed status |
| `screenshots/s3-old-policy-deleted.png` | Old public S3 bucket policy removed |
| `screenshots/cloudfront-bucket-policy-applied.png` | CloudFront-only bucket policy applied |
| `screenshots/block-public-access-enabled.png` | S3 Block Public Access re-enabled |
| `screenshots/cloudfront-origin-oac.png` | CloudFront origin using OAC |
| `screenshots/https-website-loaded.png` | Website loaded using HTTPS CloudFront URL |
| `screenshots/old-s3-url-blocked.png` | Old S3 website endpoint blocked |
| `screenshots/custom-error-page.png` | Custom error page loaded through CloudFront |
| `screenshots/cleanup-verified.png` | CloudFront/OAC cleanup verified |
| `screenshots/encoding-issue-1.png` | Encoding Issue during Distribution Update |
| `screenshots/encoding-issue-2.png` | Encoding Issue during Distribution Update |
| `screenshots/encoding-issue-workaround.png` | Workaround for Encoding Issue during Distribution Update |