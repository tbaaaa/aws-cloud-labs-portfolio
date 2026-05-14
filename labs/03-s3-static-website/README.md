# Lab 03: S3 Static Website

## Lab Summary

In this lab, I hosted a static website using Amazon S3. I created an S3 bucket, uploaded an `index.html` homepage and an `error.html` error page, enabled static website hosting, configured public access, and tested the live website endpoint.

This lab is important because S3 static website hosting is a common beginner-friendly cloud project and introduces key concepts such as buckets, objects, website endpoints, bucket policies, and public access controls.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 1C — Host a Static Website on Amazon S3
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Create a globally unique S3 bucket
- Create website files locally
- Upload website files to S3
- Enable S3 static website hosting
- Disable Block Public Access for the lab bucket
- Add a bucket policy for public read access
- Visit and test the website endpoint
- Test the custom error page
- Clean up all lab resources

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| Amazon S3 | Stores the static website files |
| S3 Static Website Hosting | Serves HTML files as a public website |
| S3 Bucket Policy | Allows public read access to website files |
| AWS CLI | Creates, uploads, configures, and verifies resources |
| PowerShell | Terminal used to run AWS CLI commands |
| HTML | Used to create the website homepage and error page |
| Web Browser | Used to test the live website URL |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- Text editor installed, such as VS Code or Notepad
- Web browser
- Basic understanding of files and folders

## Cost Notice

Estimated cost: `$0.00`

Notes:

- Amazon S3 has a free tier that includes limited storage and requests.
- This lab uses small HTML files, so the expected cost is `$0.00`.
- Cleanup is still required because public buckets and hosted files should not be left running unnecessarily.
- Public access is enabled only for this learning lab.

## Key Concepts

| Concept | Meaning |
|---|---|
| Amazon S3 | AWS object storage service used to store files in buckets. |
| Bucket | A storage container in S3. Bucket names must be globally unique. |
| Object | A file stored in S3, such as `index.html` or `error.html`. |
| Static website | A website made of files like HTML, CSS, and JavaScript without server-side processing. |
| Website endpoint | The public URL used to access an S3 static website. |
| Block Public Access | S3 safety setting that prevents accidental public exposure. |
| Bucket policy | JSON-based permissions document that controls access to bucket objects. |
| `s3:GetObject` | Permission that allows users to read/download objects from an S3 bucket. |

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.

**Command used:**

```powershell
$env:AWS_PROFILE="<PROFILE_NAME>"
```

**Result:**

- AWS CLI used my selected profile for commands in the current terminal session.

**Notes:**

- This avoids typing `--profile <PROFILE_NAME>` on every AWS CLI command.
- This setting only applies to the current PowerShell session.

---

### Step 2: Verify AWS CLI Identity

**What I did:**

- Verified that my terminal was authenticated to AWS.

**Command used:**

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

- Account ID and ARN details should be redacted before uploading screenshots.
- If the SSO session expires, run:

```bash
aws sso login --profile <PROFILE_NAME>
```

---

### Step 3: Create Local Project Folder

**What I did:**

- Created a local folder for the lab files.
- Changed into the folder before creating website files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-1c
cd ~\Desktop\workshop-lab-1c
pwd
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-1c`.

**Notes:**

- This folder stores the website files and bucket policy file for the lab.

---

### Step 4: Create an S3 Bucket

**What I did:**

- Created a globally unique S3 bucket in `us-east-1`.

**Command used:**

```bash
aws s3 mb s3://<YOUR_UNIQUE_BUCKET_NAME> --region us-east-1
```

**Expected result:**

```text
make_bucket: <YOUR_UNIQUE_BUCKET_NAME>
```

**Result:**

- S3 bucket was created successfully.

**Notes:**

- S3 bucket names must be globally unique.
- Bucket names should use lowercase letters, numbers, and hyphens.
- Avoid uppercase letters, spaces, and underscores.
- If `BucketAlreadyExists` appears, choose another bucket name.

---

### Step 5: Verify Bucket in AWS Console

**What I did:**

- Opened the AWS Console.
- Went to Amazon S3.
- Confirmed the new bucket appeared in the bucket list.

**Result:**

- Bucket was visible in the S3 console.

**Notes:**

- Console verification confirms the bucket was created outside of the CLI output.

---

### Step 6: Create Website Folder

**What I did:**

- Created a folder named `my-website` inside the local lab folder.

**Command used:**

```powershell
mkdir my-website
```

**Result:**

- Local website folder was created.

**Notes:**

- This folder stores the website files before they are uploaded to S3.

---

### Step 7: Create `index.html`

**What I did:**

- Created the homepage file for the static website.

**File path:**

```text
my-website/index.html
```

**File content:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My First Cloud Website</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background: #f0f4f8;
        }

        h1 {
            color: #232f3e;
        }

        p {
            color: #545b64;
            font-size: 18px;
        }

        .badge {
            background: #ff9900;
            color: white;
            padding: 10px 20px;
            border-radius: 5px;
            display: inline-block;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <h1>Hello from AWS S3!</h1>
    <p>You just deployed your first static website on the cloud.</p>
    <div class="badge">Cloud Engineer in Training</div>
</body>
</html>
```

**Result:**

- Homepage file was created locally.

**Notes:**

- `index.html` is the default page visitors see when they open the website URL.

---

### Step 8: Create `error.html`

**What I did:**

- Created a custom error page for the static website.

**File path:**

```text
my-website/error.html
```

**File content:**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Page Not Found</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background: #f0f4f8;
        }

        h1 {
            color: #d13212;
        }

        p {
            color: #545b64;
            font-size: 18px;
        }
    </style>
</head>
<body>
    <h1>404 - Page Not Found</h1>
    <p>The page you are looking for does not exist.</p>
</body>
</html>
```

**Result:**

- Error page file was created locally.

**Notes:**

- `error.html` is shown when a visitor tries to access a page that does not exist.

---

### Step 9: Verify Local Website Files

**What I did:**

- Confirmed the website folder contained both required files.

**Expected structure:**

```text
my-website/
├── index.html
└── error.html
```

**Result:**

- Both website files were present.

---

### Step 10: Upload Website Files to S3

**What I did:**

- Uploaded `index.html` and `error.html` to the S3 bucket.
- Set the content type to `text/html`.

**Commands used:**

```bash
aws s3 cp my-website/index.html s3://<YOUR_UNIQUE_BUCKET_NAME>/index.html --content-type "text/html"
```

```bash
aws s3 cp my-website/error.html s3://<YOUR_UNIQUE_BUCKET_NAME>/error.html --content-type "text/html"
```

**Expected result:**

```text
upload: my-website/index.html to s3://<YOUR_UNIQUE_BUCKET_NAME>/index.html
upload: my-website/error.html to s3://<YOUR_UNIQUE_BUCKET_NAME>/error.html
```

**Result:**

- Both HTML files were uploaded successfully.

**Notes:**

- The `--content-type "text/html"` option helps browsers display the files as web pages.

---

### Step 11: Verify Uploaded Objects

**What I did:**

- Checked the S3 bucket in the AWS Console.
- Confirmed that `index.html` and `error.html` appeared as objects.

**Result:**

- Both files were visible in the S3 bucket.

---

### Step 12: Enable Static Website Hosting

**What I did:**

- Enabled static website hosting on the S3 bucket.
- Set `index.html` as the index document.
- Set `error.html` as the error document.

**Command used:**

```bash
aws s3 website s3://<YOUR_UNIQUE_BUCKET_NAME> --index-document index.html --error-document error.html
```

**Result:**

- Static website hosting was enabled.

**Notes:**

- This command may return no output when successful.
- The bucket website endpoint follows this format:

```text
http://<YOUR_UNIQUE_BUCKET_NAME>.s3-website-us-east-1.amazonaws.com
```

- At this stage, the website may still show `403 Forbidden` because the bucket is not publicly readable yet.

---

### Step 13: Disable Block Public Access for the Lab Bucket

**What I did:**

- Disabled the S3 Block Public Access settings for the lab bucket.

**Command used:**

```bash
aws s3api put-public-access-block --bucket <YOUR_UNIQUE_BUCKET_NAME> --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**Result:**

- Public access block settings were updated.

**Notes:**

- S3 blocks public access by default as a safety feature.
- For this lab, public access was intentionally allowed so the website could be viewed publicly.
- In production, a more secure design would usually use CloudFront in front of S3 rather than exposing the bucket directly.

---

### Step 14: Create Bucket Policy File

**What I did:**

- Created a file named `bucket-policy.json`.
- Added a policy allowing public read access to objects in the bucket.

**File path:**

```text
bucket-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<YOUR_UNIQUE_BUCKET_NAME>/*"
        }
    ]
}
```

**Result:**

- Bucket policy file was created locally.

**Notes:**

- Replace `<YOUR_UNIQUE_BUCKET_NAME>` with the actual bucket name.
- The `/*` at the end means the policy applies to objects inside the bucket.
- This policy allows anyone on the internet to read the website files.
- Do not use this type of public policy for private or sensitive files.

---

### Step 15: Apply Bucket Policy

**What I did:**

- Applied the bucket policy to the S3 bucket.

**Command used:**

```bash
aws s3api put-bucket-policy --bucket <YOUR_UNIQUE_BUCKET_NAME> --policy file://bucket-policy.json
```

**Result:**

- Bucket policy was applied successfully.

**Notes:**

- This command may return no output when successful.

---

### Step 16: Visit Website Endpoint

**What I did:**

- Opened the S3 static website endpoint in a web browser.

**Website endpoint format:**

```text
http://<YOUR_UNIQUE_BUCKET_NAME>.s3-website-us-east-1.amazonaws.com
```

**Result:**

- The `index.html` homepage loaded successfully.

**Notes:**

- Use the S3 website endpoint, not the regular S3 object URL.
- The website endpoint includes `s3-website-us-east-1`.

---

### Step 17: Test Custom Error Page

**What I did:**

- Added `/nonexistent` to the website URL to test the error page.

**Test URL format:**

```text
http://<YOUR_UNIQUE_BUCKET_NAME>.s3-website-us-east-1.amazonaws.com/nonexistent
```

**Result:**

- The custom `404 - Page Not Found` page loaded successfully.

**Notes:**

- This confirms that the configured error document works.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| `aws sts get-caller-identity` worked | Passed |
| Local lab folder created | Passed |
| S3 bucket created | Passed |
| Bucket visible in AWS Console | Passed |
| `index.html` created | Passed |
| `error.html` created | Passed |
| Website files uploaded to S3 | Passed |
| Objects visible in S3 bucket | Passed |
| Static website hosting enabled | Passed |
| Block Public Access disabled for lab bucket | Passed |
| Bucket policy created | Passed |
| Bucket policy applied | Passed |
| Website endpoint loaded homepage | Passed |
| Custom error page loaded | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | Possible Cause | Fix |
|---|---|---|
| `BucketAlreadyExists` | The bucket name is already used by another AWS account | Choose a different globally unique bucket name |
| `403 Forbidden` when visiting website | Public access or bucket policy is not configured correctly | Confirm Block Public Access was disabled and bucket policy was applied |
| Website shows XML instead of HTML | Wrong URL format was used | Use the S3 static website endpoint, not the regular S3 object URL |
| `NoSuchBucket` | Bucket name was typed incorrectly | Check the bucket name carefully |
| Website shows old content | Browser cache | Hard refresh with `Ctrl + Shift + R` |

## Cleanup

### Step 1: Empty the S3 Bucket

**What I did:**

- Removed all files from the S3 bucket.

**Command used:**

```bash
aws s3 rm s3://<YOUR_UNIQUE_BUCKET_NAME> --recursive
```

**Expected result:**

```text
delete: s3://<YOUR_UNIQUE_BUCKET_NAME>/index.html
delete: s3://<YOUR_UNIQUE_BUCKET_NAME>/error.html
```

**Result:**

- Bucket objects were deleted.

---

### Step 2: Delete the S3 Bucket

**What I did:**

- Deleted the empty S3 bucket.

**Command used:**

```bash
aws s3 rb s3://<YOUR_UNIQUE_BUCKET_NAME>
```

**Expected result:**

```text
remove_bucket: <YOUR_UNIQUE_BUCKET_NAME>
```

**Result:**

- S3 bucket was deleted.

---

### Step 3: Verify Bucket Deletion

**What I did:**

- Checked whether the bucket still existed.

**Command used:**

```bash
aws s3 ls s3://<YOUR_UNIQUE_BUCKET_NAME>
```

**Expected result:**

```text
An error occurred (NoSuchBucket) when calling the ListObjectsV2 operation: The specified bucket does not exist
```

**Result:**

- Bucket deletion was verified.

---

### Step 4: Verify in AWS Console

**What I did:**

- Opened the S3 console.
- Confirmed the deleted bucket no longer appeared in the bucket list.

**Result:**

- Bucket was no longer visible.

---

### Step 5: Test Website URL After Cleanup

**What I did:**

- Refreshed the website endpoint after deleting the bucket.

**Result:**

- Website was no longer available.

**Notes:**

- This confirms the public website was removed.

---

### Step 6: Delete Local Lab Folder

**What I did:**

- Deleted the local project folder from my desktop.

**Command used:**

```powershell
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-1c
```

**Result:**

- Local lab folder was removed.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| `index.html` object | Deleted from S3 | Yes |
| `error.html` object | Deleted from S3 | Yes |
| S3 bucket | Deleted | Yes |
| Public website endpoint | No longer available | Yes |
| Local lab folder | Deleted | Yes |

## What I Learned

- Amazon S3 can be used to host a static website.
- S3 buckets store files as objects.
- Bucket names must be globally unique.
- Static website hosting uses a special website endpoint.
- `index.html` is the default homepage.
- `error.html` can be used as a custom error page.
- S3 blocks public access by default to protect data.
- A bucket policy can allow public read access to website files.
- Public access should be used carefully and only when intended.
- Cleanup is important to avoid leaving public resources active.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/s3-bucket-created.png` | S3 bucket created successfully |
| `screenshots/local-website-files.png` | Local `index.html` and `error.html` files created |
| `screenshots/s3-objects-uploaded.png` | Website files uploaded to S3 |
| `screenshots/static-website-hosting-enabled.png` | Static website hosting enabled |
| `screenshots/block-public-access-disabled.png` | Public access settings changed for lab bucket |
| `screenshots/bucket-policy-applied.png` | Bucket policy applied successfully |
| `screenshots/live-website-homepage.png` | Website homepage loaded in browser |
| `screenshots/custom-error-page.png` | Custom error page loaded in browser |
| `screenshots/cleanup-verified-terminal.png` | S3 bucket cleanup verified from terminal|
| `screenshots/cleanup-verified-aws-console.png` | S3 bucket cleanup verified from AWS Console|
| `screenshots/cleanup-verified-website-url.png` | S3 bucket cleanup verified from Website URL|