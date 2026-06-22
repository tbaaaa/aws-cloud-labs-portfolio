# Lab 17: S3 Security & Reliability

## Lab Summary

In this lab, I applied the Security and Reliability pillars of the AWS Well-Architected Framework to an Amazon S3 bucket containing confidential HR salary data.

I created separate Lambda roles for an HR team and an Analytics team. Both roles initially had broad S3 read permissions, allowing both teams to access the confidential salary report. I then applied a restrictive S3 bucket policy that allowed the HR role to read the file while explicitly denying the Analytics role.

I also demonstrated the effect of deleting a file without S3 versioning enabled, then enabled versioning and recovered a deleted file from its previous version.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 6A — Architecting S3 for Security & Reliability
* Session: 6 — AWS Well-Architected Framework
* Track: Solutions Architecture
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Set the active AWS CLI profile
* Verify AWS CLI authentication
* Create a local project folder
* Create an S3 bucket for confidential HR data
* Upload a confidential salary report
* Create separate IAM roles for HR and Analytics Lambda functions
* Give both roles initial S3 read access to demonstrate an overly permissive configuration
* Deploy two Lambda functions that attempt to read the confidential file
* Verify that both teams can initially access the file
* Apply an S3 bucket policy enforcing least privilege
* Verify that HR retains access while Analytics is denied
* Demonstrate permanent deletion without versioning
* Enable S3 versioning
* Delete the file again after versioning is enabled
* Recover the deleted file from a previous object version
* Clean up all lab resources

## Services / Tools Used

| Service / Tool                 | Purpose                                                             |
| ------------------------------ | ------------------------------------------------------------------- |
| Amazon S3                      | Stores the confidential salary report and supports version recovery |
| AWS Lambda                     | Simulates HR and Analytics teams reading the S3 object              |
| AWS IAM                        | Creates the Lambda execution roles and controls permissions         |
| S3 Bucket Policy               | Applies an explicit deny to enforce least privilege                 |
| AWS CLI                        | Creates, tests, verifies, and deletes resources                     |
| PowerShell                     | Runs AWS CLI commands and packages Lambda code                      |
| Python                         | Lambda runtime language                                             |
| AWS Well-Architected Framework | Provides the Security and Reliability design principles             |

## Prerequisites

* Completed Lab 01: AWS Account & CLI Setup
* AWS CLI installed and configured
* Working AWS IAM Identity Center / SSO profile
* Access to the AWS Management Console
* Text editor such as VS Code or Notepad
* Basic understanding of S3, IAM roles, Lambda, and bucket policies

## Cost Notice

Estimated cost: `$0.00`

Notes:

* IAM is free.
* This lab uses a small S3 object and two short Lambda tests.
* S3 versioning can retain object versions, so cleanup is required.
* Lambda usage should remain within the AWS free tier for this lab.

## Key Concepts

| Concept                        | Meaning                                                                                                      |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| AWS Well-Architected Framework | AWS design guidance for building secure, reliable, efficient, sustainable, and cost-aware workloads.         |
| Security Pillar                | Protecting data, systems, and assets through controlled access and least privilege.                          |
| Reliability Pillar             | Designing systems that recover from failures and protect against data loss.                                  |
| Least Privilege                | Giving an identity only the permissions it needs to complete its task.                                       |
| IAM Role                       | An AWS identity assumed by a service, such as Lambda, to receive permissions.                                |
| Bucket Policy                  | A resource-based policy attached directly to an S3 bucket.                                                   |
| Explicit Deny                  | A deny rule that overrides any matching allow permission.                                                    |
| S3 Versioning                  | A feature that keeps prior object versions and enables recovery after deletion or overwrite.                 |
| Delete Marker                  | A marker created when a versioned object is deleted; it hides the object without removing its prior version. |

## Security Notes

| Topic                                 | Explanation                                                                                                                                                                |
| ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Broad permissions are risky           | Both roles initially receive S3 read access to demonstrate why broad permissions should not be used by default.                                                            |
| Explicit deny overrides allow         | Even if Analytics has `AmazonS3ReadOnlyAccess`, an explicit bucket-policy deny prevents access to the salary file.                                                         |
| Bucket policies protect data directly | IAM permissions alone may be too broad; a bucket policy can enforce restrictions at the data layer.                                                                        |
| Admin exception is necessary          | The administrator role is excluded from the deny rule so the bucket can still be managed.                                                                                  |
| Versioning is not backup by itself    | Versioning protects against accidental deletion and overwrite, but production environments may also need replication, backups, retention controls, and lifecycle policies. |
| Sensitive data should be protected    | Real salary data should be encrypted, access logged, and limited to approved identities.                                                                                   |

## Architecture Overview

```text
HR Lambda Role -----------\
                           \
                            > S3 Bucket: Confidential Salary File
                           /
Analytics Lambda Role ----/

Before policy:
- HR role: Access granted
- Analytics role: Access granted

After restrictive bucket policy:
- HR role: Access granted
- Analytics role: Access denied

Reliability protection:
- S3 versioning retains prior object versions
- Deleted file can be recovered
```

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

* Set my AWS CLI profile for the current PowerShell session.
* Verified the active AWS identity.
* Recorded the AWS account ID for later role ARN and policy configuration.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
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

**Notes:**

* If the SSO session is expired, use:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

* Created a working folder for this lab.

**Commands used:**

```powershell
mkdir ~\Desktop\workshop-lab-6a
cd ~\Desktop\workshop-lab-6a
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-6a
```

---

### Step 3: Set Bucket Name Variable

**What I did:**

* Set a globally unique bucket name variable for the current PowerShell session.

**Command used:**

```powershell
$BUCKET="<YOUR_UNIQUE_BUCKET_NAME>"
echo $BUCKET
```

**Example:**

```powershell
$BUCKET="ayden-waf-lab6a-2026"
```

**Notes:**

* Bucket names must use lowercase letters, numbers, and hyphens.
* The variable is lost if the PowerShell window closes.
* Re-run this step if later commands report that a bucket name is missing.

---

### Step 4: Create S3 Bucket and Upload Confidential File

#### Step 4A: Create the Bucket

```powershell
aws s3 mb s3://$BUCKET --region us-east-1
```

**Expected result:**

```text
make_bucket: <YOUR_UNIQUE_BUCKET_NAME>
```

#### Step 4B: Create the Confidential File

Create a file named `employee-salaries.txt` with the following content:

```text
CONFIDENTIAL - HR Department
Employee Salary Report Q4 2025

Name            | Role              | Annual Salary
----------------|-------------------|-------------
Jane Smith      | Cloud Engineer    | $125,000
John Doe        | Security Analyst  | $115,000
Alex Johnson    | DevOps Engineer   | $130,000

This document is classified INTERNAL ONLY.
```

#### Step 4C: Upload the File

```powershell
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**Expected result:**

```text
upload: .\employee-salaries.txt to s3://<YOUR_BUCKET>/employee-salaries.txt
```

---

### Step 5: Create HR and Analytics Lambda Roles

#### Step 5A: Create Lambda Trust Policy

Create `trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Step 5B: Create the HR Role

```powershell
aws iam create-role `
  --role-name workshop-hr-team-role `
  --assume-role-policy-document file://trust-policy.json
```

#### Step 5C: Create the Analytics Role

```powershell
aws iam create-role `
  --role-name workshop-analytics-team-role `
  --assume-role-policy-document file://trust-policy.json
```

#### Step 5D: Attach Initial Permissions

```powershell
aws iam attach-role-policy `
  --role-name workshop-hr-team-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy `
  --role-name workshop-hr-team-role `
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam attach-role-policy `
  --role-name workshop-analytics-team-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy `
  --role-name workshop-analytics-team-role `
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

**Result:**

* Both roles initially had general S3 read permissions.
* This intentionally created the overly permissive state used for the security demonstration.

---

### Step 6: Create and Deploy Lambda Functions

#### Step 6A: Create Lambda Code

Create `s3_reader.py`:

```python
import boto3

def lambda_handler(event, context):
    s3 = boto3.client("s3")
    bucket = event.get("bucket")
    key = event.get("key", "employee-salaries.txt")

    try:
        response = s3.get_object(Bucket=bucket, Key=key)
        content = response["Body"].read().decode("utf-8")

        return {
            "statusCode": 200,
            "body": f"ACCESS GRANTED - File contents:\n{content}"
        }

    except Exception as error:
        return {
            "statusCode": 403,
            "body": f"ACCESS DENIED - {str(error)}"
        }
```

#### Step 6B: Create Deployment Package

```powershell
Compress-Archive -Path s3_reader.py -DestinationPath s3reader.zip -Force
```

#### Step 6C: Deploy HR Reader Function

```powershell
aws lambda create-function `
  --function-name workshop-hr-reader `
  --runtime python3.12 `
  --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-hr-team-role" `
  --handler s3_reader.lambda_handler `
  --zip-file fileb://s3reader.zip `
  --timeout 10 `
  --region us-east-1
```

#### Step 6D: Deploy Analytics Reader Function

```powershell
aws lambda create-function `
  --function-name workshop-analytics-reader `
  --runtime python3.12 `
  --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-analytics-team-role" `
  --handler s3_reader.lambda_handler `
  --zip-file fileb://s3reader.zip `
  --timeout 10 `
  --region us-east-1
```

**Notes:**

* IAM role propagation may take a short time.
* Wait briefly and retry if Lambda reports that the role cannot be assumed.

---

## Part 1: Security Pillar

### Step 7: Demonstrate the Overly Permissive State

#### Step 7A: Create Test Payload

Create `read-payload.json`:

```json
{
  "bucket": "<YOUR_BUCKET_NAME>",
  "key": "employee-salaries.txt"
}
```

#### Step 7B: Test HR Access

```powershell
aws lambda invoke `
  --function-name workshop-hr-reader `
  --payload file://read-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  hr-response.json

Write-Output "=== HR TEAM RESULT ==="
(Get-Content hr-response.json | ConvertFrom-Json).body
```

**Expected result:**

```text
ACCESS GRANTED
```

#### Step 7C: Test Analytics Access

```powershell
aws lambda invoke `
  --function-name workshop-analytics-reader `
  --payload file://read-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  analytics-response.json

Write-Output "=== ANALYTICS TEAM RESULT ==="
(Get-Content analytics-response.json | ConvertFrom-Json).body
```

**Expected result:**

```text
ACCESS GRANTED
```

**Finding:**

* Analytics could read confidential HR salary data.
* This demonstrated a least-privilege failure.

---

### Step 8: Apply Restrictive Bucket Policy

Create `restrict-policy.json`.

Replace:

* `BUCKET_NAME_HERE` with the S3 bucket name.
* `ACCOUNT_ID_HERE` with the AWS account ID.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonHRAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::BUCKET_NAME_HERE/*",
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::ACCOUNT_ID_HERE:role/workshop-hr-team-role",
            "arn:aws:iam::ACCOUNT_ID_HERE:role/AWSReservedSSO_AdministratorAccess_*"
          ]
        }
      }
    }
  ]
}
```

Apply the policy:

```powershell
aws s3api put-bucket-policy `
  --bucket $BUCKET `
  --policy file://restrict-policy.json
```

**Result:**

* The policy denied `s3:GetObject` to all identities except the HR role and the administrator role.

---

### Step 9: Verify Least Privilege

#### Step 9A: Retest HR Access

```powershell
aws lambda invoke `
  --function-name workshop-hr-reader `
  --payload file://read-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  hr-response2.json

Write-Output "=== HR TEAM AFTER POLICY ==="
(Get-Content hr-response2.json | ConvertFrom-Json).body
```

**Expected result:**

```text
ACCESS GRANTED
```

#### Step 9B: Retest Analytics Access

```powershell
aws lambda invoke `
  --function-name workshop-analytics-reader `
  --payload file://read-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  analytics-response2.json

Write-Output "=== ANALYTICS TEAM AFTER POLICY ==="
(Get-Content analytics-response2.json | ConvertFrom-Json).body
```

**Expected result:**

```text
ACCESS DENIED
```

**Result:**

| Team           | Before Bucket Policy | After Bucket Policy |
| -------------- | -------------------- | ------------------- |
| HR Team        | Access granted       | Access granted      |
| Analytics Team | Access granted       | Access denied       |

**What this proved:**

* The bucket policy’s explicit deny overrode the Analytics role’s broad S3 read permission.
* Least privilege was enforced at the data layer.

---

## Part 2: Reliability Pillar

### Step 10: Delete File Without Versioning

#### Step 10A: Confirm File Exists

```powershell
aws s3 ls s3://$BUCKET/
```

#### Step 10B: Delete the File

```powershell
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

#### Step 10C: Confirm File Is Gone

```powershell
aws s3 ls s3://$BUCKET/
```

**Result:**

* The object was no longer listed.
* Because versioning was not enabled at the time of deletion, the original object could not be recovered through S3 version history.

---

### Step 11: Enable S3 Versioning

#### Step 11A: Enable Versioning

```powershell
aws s3api put-bucket-versioning `
  --bucket $BUCKET `
  --versioning-configuration Status=Enabled
```

#### Step 11B: Verify Versioning

```powershell
aws s3api get-bucket-versioning `
  --bucket $BUCKET `
  --query "Status" `
  --output text
```

**Expected result:**

```text
Enabled
```

#### Step 11C: Re-upload the Salary File

```powershell
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

---

### Step 12: Delete and Recover File With Versioning

#### Step 12A: Delete File Again

```powershell
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

#### Step 12B: Confirm Normal Listing Appears Empty

```powershell
aws s3 ls s3://$BUCKET/
```

#### Step 12C: List Object Versions

```powershell
aws s3api list-object-versions `
  --bucket $BUCKET `
  --query "Versions[].{Key:Key,VersionId:VersionId}" `
  --output table
```

**Result:**

* The previous version of `employee-salaries.txt` remained available.
* A delete marker hid the object from normal listings.

#### Step 12D: Recover the Previous Version

Replace `<VERSION_ID>` with the version ID displayed in the previous step.

```powershell
aws s3api copy-object `
  --bucket $BUCKET `
  --copy-source "$BUCKET/employee-salaries.txt?versionId=<VERSION_ID>" `
  --key employee-salaries.txt
```

#### Step 12E: Confirm Recovery

```powershell
aws s3 ls s3://$BUCKET/
```

**Expected result:**

```text
employee-salaries.txt
```

**Result:**

* The deleted salary report was restored successfully.

## Validation / Checkpoints

| Checkpoint                                    | Result |
| --------------------------------------------- | ------ |
| AWS profile set                               | Passed |
| AWS identity verified                         | Passed |
| Local project folder created                  | Passed |
| Unique bucket variable set                    | Passed |
| S3 bucket created                             | Passed |
| Confidential salary file uploaded             | Passed |
| HR IAM role created                           | Passed |
| Analytics IAM role created                    | Passed |
| Initial S3 read access attached to both roles | Passed |
| HR Lambda function deployed                   | Passed |
| Analytics Lambda function deployed            | Passed |
| HR initially accessed file                    | Passed |
| Analytics initially accessed file             | Passed |
| Restrictive bucket policy created             | Passed |
| Restrictive bucket policy applied             | Passed |
| HR retained access after policy               | Passed |
| Analytics denied access after policy          | Passed |
| File deleted without versioning               | Passed |
| S3 versioning enabled                         | Passed |
| File re-uploaded                              | Passed |
| File deleted with versioning enabled          | Passed |
| Previous version located                      | Passed |
| Salary file recovered successfully            | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                   | What It Means                                                            | How to Fix                                                                                 |
| --------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| `$BUCKET` is empty                      | PowerShell session was restarted or variable was never set               | Re-run Step 3                                                                              |
| `BucketAlreadyExists`                   | The selected bucket name belongs to another AWS account                  | Choose a different globally unique name and set `$BUCKET` again                            |
| Lambda role cannot be assumed           | IAM changes have not propagated yet                                      | Wait 10–30 seconds, then retry deployment                                                  |
| Analytics still has access after policy | Bucket policy contains an incorrect bucket name, account ID, or role ARN | Recheck all placeholders in `restrict-policy.json`                                         |
| HR access is denied after policy        | HR role ARN does not exactly match the deployed role                     | Confirm role name is `workshop-hr-team-role`                                               |
| `MalformedPolicy`                       | JSON policy contains invalid syntax                                      | Check commas, brackets, quotation marks, and placeholders                                  |
| No object versions appear               | Versioning was enabled after the deletion                                | Re-upload the object after enabling versioning, then repeat the deletion and recovery test |
| Bucket cannot be deleted                | Versioned objects or delete markers still remain                         | Delete every object version and delete marker before deleting the bucket                   |

## Cleanup

### Step 1: Delete Lambda Functions

```powershell
aws lambda delete-function `
  --function-name workshop-hr-reader `
  --region us-east-1

aws lambda delete-function `
  --function-name workshop-analytics-reader `
  --region us-east-1
```

### Step 2: Delete Lambda Log Groups

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-hr-reader `
  --region us-east-1

aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-analytics-reader `
  --region us-east-1
```

### Step 3: Detach Policies and Delete IAM Roles

```powershell
aws iam detach-role-policy `
  --role-name workshop-hr-team-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam detach-role-policy `
  --role-name workshop-hr-team-role `
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam delete-role `
  --role-name workshop-hr-team-role

aws iam detach-role-policy `
  --role-name workshop-analytics-team-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam detach-role-policy `
  --role-name workshop-analytics-team-role `
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam delete-role `
  --role-name workshop-analytics-team-role
```

### Step 4: Remove S3 Object Versions and Delete Markers

List versions and delete markers:

```powershell
aws s3api list-object-versions `
  --bucket $BUCKET `
  --query "[Versions[].{Key:Key,VersionId:VersionId},DeleteMarkers[].{Key:Key,VersionId:VersionId}]" `
  --output table
```

For every version ID and delete-marker version ID returned, run:

```powershell
aws s3api delete-object `
  --bucket $BUCKET `
  --key employee-salaries.txt `
  --version-id <VERSION_ID>
```

### Step 5: Delete Bucket

```powershell
aws s3api delete-bucket `
  --bucket $BUCKET `
  --region us-east-1
```

### Step 6: Delete Local Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-6a
```

## Cleanup Verification

| Resource                    | Cleanup Action | Verified |
| --------------------------- | -------------- | -------- |
| HR Lambda function          | Deleted        | Yes      |
| Analytics Lambda function   | Deleted        | Yes      |
| HR Lambda log group         | Deleted        | Yes      |
| Analytics Lambda log group  | Deleted        | Yes      |
| HR IAM role policies        | Detached       | Yes      |
| HR IAM role                 | Deleted        | Yes      |
| Analytics IAM role policies | Detached       | Yes      |
| Analytics IAM role          | Deleted        | Yes      |
| S3 object versions          | Deleted        | Yes      |
| S3 delete markers           | Deleted        | Yes      |
| S3 bucket                   | Deleted        | Yes      |
| Local lab folder            | Deleted        | Yes      |

## What I Learned

* Least privilege is not limited to IAM policies; S3 bucket policies can enforce access directly at the data layer.
* An explicit deny overrides an allow permission in AWS.
* Broad permissions such as `AmazonS3ReadOnlyAccess` may be inappropriate for confidential data.
* IAM roles make it possible to give separate workloads separate access levels.
* Bucket-policy conditions can restrict access to specific IAM role ARNs.
* S3 versioning protects against accidental deletion and object overwrite.
* Deleting a versioned object creates a delete marker instead of permanently removing the underlying version.
* A versioned S3 bucket requires all object versions and delete markers to be removed before the bucket can be deleted.
* The Security and Reliability pillars can be demonstrated through measurable configuration changes.

## Screenshots

| Screenshot                                        | Description                                             |
| ------------------------------------------------- | ------------------------------------------------------- |
| `screenshots/project-folder-created.png`          | Local lab folder created                                |
| `screenshots/s3-bucket-created.png`               | S3 bucket created                                       |
| `screenshots/confidential-file-uploaded.png`      | Salary report uploaded to S3                            |
| `screenshots/hr-role-created.png`                 | HR Lambda role created                                  |
| `screenshots/analytics-role-created.png`          | Analytics Lambda role created                           |
| `screenshots/lambda-functions-created-1.png`        | Both Lambda functions deployed                          |
| `screenshots/lambda-functions-created-2.png`        | Both Lambda functions deployed                          |
| `screenshots/hr-access-before-policy.png`         | HR function accessed salary report before policy        |
| `screenshots/analytics-access-before-policy.png`  | Analytics function accessed salary report before policy |
| `screenshots/restrictive-policy-created.png`      | Restrictive S3 bucket policy created                    |
| `screenshots/restrictive-policy-applied.png`      | Restrictive S3 bucket policy applied                    |
| `screenshots/hr-access-after-policy.png`          | HR retained access after policy                         |
| `screenshots/analytics-access-denied.png`         | Analytics denied access after policy                    |
| `screenshots/file-deleted-without-versioning.png` | File deleted before versioning was enabled              |
| `screenshots/versioning-enabled.png`              | S3 versioning enabled                                   |
| `screenshots/object-versions-listed.png`          | Previous object versions displayed                      |
| `screenshots/delete-marker-console.png`           | Delete marker visible in S3 console                     |
| `screenshots/file-recovered.png`                  | Salary report recovered successfully                    |
| `screenshots/cleanup-verified-1.png`                | Lab cleanup completed                                   |
| `screenshots/cleanup-verified-2.png`                | Lab cleanup completed                                   |
