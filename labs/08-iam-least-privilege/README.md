# Lab 07: IAM Least Privilege

## Lab Summary

In this lab, I practiced the principle of least privilege by creating an IAM user with read-only access to one specific S3 bucket.

I created an S3 bucket with a test file, created an IAM user with no permissions by default, wrote a custom inline policy that only allows listing and reading from that bucket, created access keys for the restricted user, and tested that reading worked while writing and deleting were denied.

This lab is important because least privilege is one of the most important security concepts in AWS and cloud security.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 3A — IAM Least Privilege: Create a Read-Only User
- Session: 3 — Cloud Security Fundamentals
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Create an S3 bucket for testing
- Upload a test file to the S3 bucket
- Create an IAM user with no permissions
- Write a custom S3 read-only policy
- Attach the policy directly to the IAM user
- Create access keys for the restricted user
- Temporarily switch the AWS CLI to the restricted user's credentials
- Test list access to the bucket
- Test read access to the object
- Confirm write access is denied
- Confirm delete access is denied
- Switch back to the admin profile
- Verify the user and policy in the AWS Console
- Clean up access keys, policy, user, bucket, and local files

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS IAM | Creates the restricted user and controls permissions |
| IAM User | Identity used to test limited access |
| IAM Inline Policy | Custom policy attached directly to the user |
| IAM Access Keys | Programmatic credentials used to test the restricted user's permissions |
| Amazon S3 | Stores the test file used for permission testing |
| AWS CLI | Creates, tests, and deletes resources |
| PowerShell | Terminal used to run AWS CLI commands |
| JSON | Used to define the custom IAM policy |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- Text editor such as VS Code or Notepad
- Basic understanding of S3 buckets and objects
- Basic understanding of IAM users and policies

## Cost Notice

Estimated cost: `$0.00`

Notes:

- IAM is always free.
- S3 usage in this lab should remain within the free tier because only a small test file is used.
- Cleanup is still required so access keys, IAM users, and test buckets do not remain in the account unnecessarily.

## Key Concepts

| Concept | Meaning |
|---|---|
| IAM | AWS Identity and Access Management. It controls who can do what in an AWS account. |
| IAM User | An identity inside an AWS account with its own credentials and permissions. |
| IAM Policy | A JSON document that defines allowed or denied actions on AWS resources. |
| Inline Policy | A policy embedded directly into one user, group, or role. |
| Least Privilege | Giving an identity only the permissions it needs and nothing more. |
| Access Keys | Long-term credentials used for programmatic AWS CLI/API access. |
| `s3:ListBucket` | Permission that allows listing objects inside a bucket. |
| `s3:GetObject` | Permission that allows reading/downloading objects from a bucket. |
| `s3:PutObject` | Permission that allows uploading/writing objects to a bucket. |
| `s3:DeleteObject` | Permission that allows deleting objects from a bucket. |
| Bucket ARN | ARN format for the bucket itself, such as `arn:aws:s3:::bucket-name`. |
| Object ARN | ARN format for objects inside the bucket, such as `arn:aws:s3:::bucket-name/*`. |

## Security Notes

| Topic | Explanation |
|---|---|
| Least privilege | The user only receives the permissions required to list and read files from one specific bucket. |
| No write/delete permissions | The policy intentionally does not include `s3:PutObject`, `s3:DeleteObject`, or `s3:DeleteBucket`. |
| Access key handling | Access keys should never be committed to GitHub, shared publicly, or stored in screenshots. |
| Temporary lab credentials | The restricted user's access keys are only used temporarily for testing and deleted during cleanup. |
| IAM users vs Identity Center | Human users should generally use IAM Identity Center. IAM users and access keys are often used for legacy apps, automation, or controlled demonstrations. |
| Environment variables | The lab temporarily uses environment variables to make the AWS CLI act as the restricted user. |

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.

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

- AWS CLI returned my active admin identity successfully.

**Notes:**

- If the SSO token is expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

- Created a local folder for this lab.
- Changed into that folder before creating files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-3a
cd ~\Desktop\workshop-lab-3a
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-3a
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-3a`.

**Notes:**

- All local files for this lab should be saved in this folder.

---

### Step 3: Create an S3 Bucket

**What I did:**

- Created a globally unique S3 bucket for testing permissions.

**Command used:**

```bash
aws s3 mb s3://<YOUR_BUCKET_NAME> --region us-east-1
```

**Expected result:**

```text
make_bucket: <YOUR_BUCKET_NAME>
```

**Result:**

- S3 bucket was created successfully.

**Notes:**

- S3 bucket names must be globally unique.
- Use lowercase letters, numbers, and hyphens.
- Do not include angle brackets when running the command.

---

### Step 4: Create and Upload Test File

**What I did:**

- Created a local test file.
- Uploaded it to the S3 bucket.

**PowerShell commands used:**

```powershell
"This is a secret document. Only authorized readers should see this." | Out-File test-file.txt
aws s3 cp test-file.txt s3://<YOUR_BUCKET_NAME>/test-file.txt
```

**Expected result:**

```text
upload: ./test-file.txt to s3://<YOUR_BUCKET_NAME>/test-file.txt
```

**Result:**

- Test file uploaded successfully.

**Notes:**

- This file will be used to test read access later.

---

### Step 5: Create IAM User

**What I did:**

- Created an IAM user named `workshop-readonly-user`.

**Command used:**

```bash
aws iam create-user --user-name workshop-readonly-user
```

**Expected result:**

- JSON output showing the new IAM user.

**Result:**

- IAM user was created successfully.

**Notes:**

- A new IAM user has no permissions by default.
- This is useful for demonstrating how permissions are added intentionally.

---

### Step 6: Create Read-Only IAM Policy File

**What I did:**

- Created a file named `s3-readonly-policy.json`.
- Wrote a custom policy that allows only listing and reading from my specific S3 bucket.

**File path:**

```text
s3-readonly-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowListBucket",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>"
        },
        {
            "Sid": "AllowGetObject",
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
        }
    ]
}
```

**Result:**

- Custom read-only policy file was created locally.

**Notes:**

- Replace `<YOUR_BUCKET_NAME>` in both resource lines.
- The first resource line applies to the bucket itself.
- The second resource line applies to objects inside the bucket.
- Make sure the file is saved as `.json`, not `.json.txt`.

---

### Step 7: Attach Inline Policy to User

**What I did:**

- Attached the custom read-only policy directly to the IAM user as an inline policy.

**Command used:**

```bash
aws iam put-user-policy --user-name workshop-readonly-user --policy-name S3ReadOnlyPolicy --policy-document file://s3-readonly-policy.json
```

**Expected result:**

```text
No output
```

**Result:**

- Inline policy was attached successfully.

**Notes:**

- No output usually means the command succeeded.
- This policy is directly attached to `workshop-readonly-user`.

---

### Step 8: Create Access Keys for Restricted User

**What I did:**

- Created access keys for `workshop-readonly-user`.

**Command used:**

```bash
aws iam create-access-key --user-name workshop-readonly-user
```

**Expected result:**

```json
{
  "AccessKey": {
    "UserName": "workshop-readonly-user",
    "AccessKeyId": "REDACTED",
    "SecretAccessKey": "REDACTED",
    "Status": "Active"
  }
}
```

**Result:**

- Access key was created successfully.

**Notes:**

- The secret access key is only shown once.
- Do not upload the access key ID or secret access key to GitHub.
- Do not include these values in screenshots.
- These credentials must be deleted during cleanup.

---

### Step 9: Switch to Restricted User Credentials

**What I did:**

- Temporarily configured PowerShell to use the restricted user's access keys.
- Removed `AWS_PROFILE` so the CLI would use the access keys instead of my admin SSO profile.

**PowerShell commands used:**

```powershell
$env:AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID_FROM_STEP_8>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY_FROM_STEP_8>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- The identity output should show `workshop-readonly-user`, not the admin role.

**Result:**

- AWS CLI switched to the restricted IAM user.

**Notes:**

- This step is important because `AWS_PROFILE` can override the access key environment variables.
- If the identity still shows the admin role, check that `AWS_PROFILE` was removed.

---

### Step 10: Test LIST Access

**What I did:**

- Tested whether the restricted user could list the contents of the S3 bucket.

**Command used:**

```bash
aws s3 ls s3://<YOUR_BUCKET_NAME>/
```

**Expected result:**

```text
test-file.txt
```

**Result:**

- The restricted user could list the bucket contents.

**Why this worked:**

- The policy allowed `s3:ListBucket` on the bucket ARN.

---

### Step 11: Test READ Access

**What I did:**

- Tested whether the restricted user could read the uploaded file.

**Command used:**

```bash
aws s3 cp s3://<YOUR_BUCKET_NAME>/test-file.txt -
```

**Expected result:**

```text
This is a secret document. Only authorized readers should see this.
```

**Result:**

- The restricted user could read the file.

**Why this worked:**

- The policy allowed `s3:GetObject` on objects inside the bucket.
- The `-` at the end prints the file contents to the terminal instead of downloading it.

---

### Step 12: Test WRITE Access

**What I did:**

- Tried to upload a new file to the bucket using the restricted user's credentials.

**PowerShell commands used:**

```powershell
"hack attempt" | Out-File test-write.txt
aws s3 cp test-write.txt s3://<YOUR_BUCKET_NAME>/unauthorized.txt 2>&1
```

**Expected result:**

```text
upload failed
An error occurred (AccessDenied)
```

**Result:**

- Write access was denied.

**Why this was expected:**

- The policy did not allow `s3:PutObject`.
- This confirms least privilege is working.

---

### Step 13: Test DELETE Access

**What I did:**

- Tried to delete the original test file using the restricted user's credentials.

**Command used:**

```bash
aws s3 rm s3://<YOUR_BUCKET_NAME>/test-file.txt 2>&1
```

**Expected result:**

```text
delete failed
An error occurred (AccessDenied)
```

**Result:**

- Delete access was denied.

**Why this was expected:**

- The policy did not allow `s3:DeleteObject`.
- The restricted user can read the file but cannot delete it.

---

### Step 14: Switch Back to Admin Credentials

**What I did:**

- Cleared the restricted user's access key environment variables.
- Restored my admin AWS CLI profile.

**PowerShell commands used:**

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- The identity should show the admin role again, not `workshop-readonly-user`.

**Result:**

- AWS CLI returned to my admin profile.

**Notes:**

- This step is required before cleanup.
- Cleanup will fail if the CLI is still using the restricted user's credentials.

---

### Step 15: Verify in AWS Console

**What I did:**

- Opened the AWS Console.
- Went to IAM.
- Opened the `workshop-readonly-user`.
- Checked the Permissions tab.
- Verified that `S3ReadOnlyPolicy` appeared as an inline policy.

**Result:**

- IAM user and inline policy were visible in the AWS Console.

**Notes:**

- This is similar to how an auditor or cloud security analyst might review permissions.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| Admin identity verified | Passed |
| Local project folder created | Passed |
| S3 bucket created | Passed |
| Test file uploaded | Passed |
| IAM user created | Passed |
| Read-only policy file created | Passed |
| Inline policy attached to user | Passed |
| Access keys created | Passed |
| CLI switched to restricted user | Passed |
| Restricted user could list bucket | Passed |
| Restricted user could read object | Passed |
| Restricted user could not upload object | Passed |
| Restricted user could not delete object | Passed |
| CLI switched back to admin user | Passed |
| IAM user and policy verified in console | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| `Error when retrieving token from sso: Token has expired and refresh failed` after switching back to admin credentials | The admin AWS CLI profile uses IAM Identity Center/SSO. While testing the restricted IAM user, the cached SSO token expired, so AWS CLI could not refresh the admin session automatically. | Ran `aws sso login --profile <YOUR_PROFILE_NAME>` to renew the SSO session, restored `$env:AWS_PROFILE`, then verified the admin identity with `aws sts get-caller-identity`. |

## Troubleshooting Notes

| Issue | Possible Cause | Fix |
|---|---|---|
| `EntityAlreadyExists` when creating the user | The IAM user already exists from a previous attempt | Delete the old user first or use a different user name |
| `MalformedPolicyDocument` when attaching the policy | JSON syntax error or bucket placeholder was not replaced | Check commas, brackets, quotation marks, and both bucket ARN lines |
| `get-caller-identity` still shows admin after setting access keys | `AWS_PROFILE` is still overriding the access keys | Run `Remove-Item Env:\AWS_PROFILE` |
| `get-caller-identity` still shows readonly user during cleanup | Restricted access key variables were not cleared | Remove `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION`, then restore `AWS_PROFILE` |
| `AccessDenied` during list/read test | Policy is missing `s3:ListBucket` or `s3:GetObject`, or bucket ARN is wrong | Review the policy and confirm the bucket name is correct |
| Write/delete test does not show `AccessDenied` clearly | Error output may not be redirected | Use `2>&1` at the end of the command |
| `NoSuchBucket` | Bucket name was typed incorrectly or deleted | Confirm the bucket name with `aws s3 ls` |
| SSO token expired after switching back to admin | The cached IAM Identity Center token expired while using the restricted IAM user credentials | Run `aws sso login --profile <YOUR_PROFILE_NAME>`, then run `aws sts get-caller-identity` again |

## Cleanup

### Step 1: Delete Access Key

**What I did:**

- Deleted the access key created for the restricted user.

**Command used:**

```bash
aws iam delete-access-key --user-name workshop-readonly-user --access-key-id <ACCESS_KEY_ID_FROM_STEP_8>
```

**Expected result:**

```text
No output
```

**Result:**

- Access key was deleted successfully.

**Notes:**

- This is important because access keys are long-term credentials.
- Leaving unused access keys active is a security risk.

---

### Step 2: Delete User Inline Policy

**What I did:**

- Deleted the inline policy attached to the IAM user.

**Command used:**

```bash
aws iam delete-user-policy --user-name workshop-readonly-user --policy-name S3ReadOnlyPolicy
```

**Expected result:**

```text
No output
```

**Result:**

- Inline policy was deleted successfully.

---

### Step 3: Delete IAM User

**What I did:**

- Deleted the restricted IAM user.

**Command used:**

```bash
aws iam delete-user --user-name workshop-readonly-user
```

**Expected result:**

```text
No output
```

**Result:**

- IAM user was deleted successfully.

---

### Step 4: Empty S3 Bucket

**What I did:**

- Removed all objects from the S3 bucket.

**Command used:**

```bash
aws s3 rm s3://<YOUR_BUCKET_NAME> --recursive
```

**Expected result:**

```text
delete: s3://<YOUR_BUCKET_NAME>/test-file.txt
```

**Result:**

- Bucket objects were removed.

---

### Step 5: Delete S3 Bucket

**What I did:**

- Deleted the empty S3 bucket.

**Command used:**

```bash
aws s3 rb s3://<YOUR_BUCKET_NAME>
```

**Expected result:**

```text
remove_bucket: <YOUR_BUCKET_NAME>
```

**Result:**

- S3 bucket was deleted.

---

### Step 6: Delete Local Project Folder

**What I did:**

- Deleted the local project folder.

**PowerShell commands used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-3a
```

**Result:**

- Local lab folder was removed.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| IAM access key | Deleted | Yes |
| IAM inline policy | Deleted | Yes |
| IAM user | Deleted | Yes |
| S3 test object | Deleted | Yes |
| S3 bucket | Deleted | Yes |
| Local lab folder | Deleted | Yes |

## What I Learned

- IAM users start with no permissions by default.
- Least privilege means giving only the access required for the task.
- A user can be allowed to read from S3 without being allowed to upload or delete files.
- `s3:ListBucket` applies to the bucket itself.
- `s3:GetObject` applies to objects inside the bucket.
- Bucket ARNs and object ARNs use different formats.
- Inline policies are useful for one-off permissions but are not reusable like managed policies.
- Access keys are powerful and should be protected, avoided when possible, and deleted when no longer needed.
- Environment variables can temporarily change which AWS identity the CLI uses.
- It is important to confirm identity with `aws sts get-caller-identity` before testing or cleaning up.
- IAM Identity Center/SSO sessions are temporary and can expire during a lab.
- If the AWS CLI says the SSO token expired, I need to renew the session with `aws sso login --profile <PROFILE_NAME>`.
- `aws sts get-caller-identity` is useful for confirming whether the CLI is using the admin profile or the restricted IAM user.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/project-folder-created.png` | Local lab folder created |
| `screenshots/s3-bucket-created.png` | S3 bucket created successfully |
| `screenshots/test-file-uploaded.png` | Test file uploaded to S3 |
| `screenshots/iam-user-created.png` | IAM read-only user created |
| `screenshots/readonly-policy-file-created.png` | Custom read-only policy file created |
| `screenshots/policy-attached.png` | Inline policy attached to IAM user |
| `screenshots/access-key-created-redacted.png` | Access key created with sensitive values redacted |
| `screenshots/restricted-user-identity.png` | CLI switched to restricted IAM user |
| `screenshots/list-access-success.png` | Restricted user successfully listed bucket contents |
| `screenshots/read-access-success.png` | Restricted user successfully read test file |
| `screenshots/write-access-denied.png` | Restricted user denied when trying to upload |
| `screenshots/delete-access-denied.png` | Restricted user denied when trying to delete |
| `screenshots/sso-token-expired.png` | Cached SSO token has expired when switching back |
| `screenshots/admin-profile-restored.png` | CLI switched back to admin profile |
| `screenshots/iam-console-policy-check.png` | IAM user and inline policy verified in AWS Console |
| `screenshots/cleanup-verified.png` | Lab resources cleaned up |