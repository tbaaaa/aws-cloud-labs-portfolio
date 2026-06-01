# Lab 08: IAM Groups, Custom Policies & MFA

## Lab Summary

In this lab, I practiced managing AWS permissions at scale using IAM groups. I created an S3 bucket with a test file, created an IAM group, wrote a custom policy that allows read access but explicitly denies destructive delete actions, added a user to the group, and tested that the user inherited the group permissions.

I also enabled MFA on the AWS root account to add a second layer of protection.

This lab is important because it introduces three major cloud security concepts: group-based access management, explicit Deny, and multi-factor authentication.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 3B — IAM Groups, Custom Policies & MFA
- Session: 3 — Cloud Security Fundamentals
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Create an S3 bucket and upload a test file
- Create an IAM group
- Write a custom S3 policy with an explicit Deny
- Attach the policy to the IAM group
- Create an IAM user
- Add the user to the group
- Create access keys for the group member
- Temporarily switch the AWS CLI to the group member credentials
- Test that list access works
- Test that delete access is denied
- Switch back to admin credentials
- Verify the group, user, and policy in the AWS Console
- Enable MFA on the AWS root account
- Clean up temporary IAM and S3 resources

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS IAM | Manages users, groups, and permissions |
| IAM Group | Holds permissions that users inherit |
| IAM User | Test user added to the group |
| IAM Inline Group Policy | Custom policy attached directly to the group |
| IAM Access Keys | Temporary programmatic credentials used to test the group member |
| Amazon S3 | Stores the test file used for permission testing |
| AWS CLI | Creates, tests, and deletes resources |
| PowerShell | Terminal used to run AWS CLI commands |
| JSON | Used to define the custom group policy |
| MFA Authenticator App | Provides time-based codes for root account MFA |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- Completed Lab 3A: IAM Least Privilege
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- Text editor such as VS Code or Notepad
- Smartphone with an authenticator app, such as Microsoft Authenticator, Google Authenticator, or Authy
- Basic understanding of S3, IAM users, IAM groups, and policies

## Cost Notice

Estimated cost: about `$0.023` if using a very small S3 test file.

Notes:

- IAM is free.
- MFA is free.
- S3 may create very small storage/request costs.
- Cleanup is required for the temporary S3 bucket, IAM user, access key, group, and policy.
- Do **not** remove MFA from the root account after the lab. MFA should stay enabled permanently.

## Key Concepts

| Concept | Meaning |
|---|---|
| IAM Group | A collection of IAM users that can share the same permissions. |
| Group Policy | A policy attached to a group. Users in the group inherit the permissions. |
| Inline Policy | A policy embedded directly into one IAM identity, such as a user, group, or role. |
| Explicit Deny | A policy statement with `"Effect": "Deny"`. It overrides Allow permissions. |
| Implicit Deny | The default deny that happens when no Allow policy exists. |
| Least Privilege | Granting only the permissions needed and nothing more. |
| MFA | Multi-Factor Authentication, which requires a password plus a second factor such as a phone code. |
| Defense in Depth | Using multiple layers of security controls to reduce risk. |
| `s3:ListBucket` | Allows listing objects in a bucket. |
| `s3:GetObject` | Allows reading objects from a bucket. |
| `s3:DeleteObject` | Allows deleting objects from a bucket. |
| `s3:DeleteBucket` | Allows deleting the bucket itself. |

## Security Notes

| Topic | Explanation |
|---|---|
| Groups scale better than user policies | Instead of attaching the same policy to many users individually, permissions can be attached to a group. |
| Explicit Deny is strongest | If a policy explicitly denies an action, the action is blocked even if another policy allows it. |
| Root MFA is critical | The root account has full account-level power, so MFA helps protect it from password-only compromise. |
| Access key handling | Access keys must never be committed to GitHub, shared, or left active after the lab. |
| Temporary test user | The IAM user in this lab is only for testing and should be deleted during cleanup. |
| Do not remove root MFA | MFA should stay enabled after the lab because it protects the AWS account. |

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
mkdir ~\Desktop\workshop-lab-3b
cd ~\Desktop\workshop-lab-3b
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-3b
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-3b`.

**Notes:**

- All local files for this lab should be saved in this folder.

---

### Step 3: Create S3 Bucket and Upload Test File

**What I did:**

- Created a globally unique S3 bucket.
- Created a test file.
- Uploaded the test file to the bucket.

**Commands used:**

```bash
aws s3 mb s3://<YOUR_BUCKET_NAME> --region us-east-1
```

```powershell
"Confidential HR data - do not delete" | Out-File test-file.txt
aws s3 cp test-file.txt s3://<YOUR_BUCKET_NAME>/test-file.txt
```

**Expected result:**

```text
make_bucket: <YOUR_BUCKET_NAME>
upload: ./test-file.txt to s3://<YOUR_BUCKET_NAME>/test-file.txt
```

**Result:**

- S3 bucket and test file were created successfully.

**Notes:**

- S3 bucket names must be globally unique.
- Do not include angle brackets when running commands.
- This file will be used to test group permissions.

---

### Step 4: Create IAM Group

**What I did:**

- Created an IAM group named `workshop-s3-readers`.

**Command used:**

```bash
aws iam create-group --group-name workshop-s3-readers
```

**Expected result:**

- JSON output showing `"GroupName": "workshop-s3-readers"`.

**Result:**

- IAM group was created successfully.

**Notes:**

- Groups are useful when multiple users need the same permissions.
- Users added to this group inherit policies attached to the group.

---

### Step 5: Create Custom S3 Policy with Explicit Deny

**What I did:**

- Created a file named `custom-s3-policy.json`.
- Wrote a policy that allows read access to the bucket but explicitly denies delete actions.

**File path:**

```text
custom-s3-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowReadAccess",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::<YOUR_BUCKET_NAME>",
                "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
            ]
        },
        {
            "Sid": "ExplicitDenyDestructiveActions",
            "Effect": "Deny",
            "Action": [
                "s3:DeleteObject",
                "s3:DeleteBucket"
            ],
            "Resource": [
                "arn:aws:s3:::<YOUR_BUCKET_NAME>",
                "arn:aws:s3:::<YOUR_BUCKET_NAME>/*"
            ]
        }
    ]
}
```

**Result:**

- Custom group policy file was created locally.

**Notes:**

- Replace `<YOUR_BUCKET_NAME>` in all four places.
- Make sure the file is saved as `.json`, not `.json.txt`.
- The Allow statement permits listing and reading.
- The Deny statement blocks object and bucket deletion.

---

### Step 6: Attach Policy to IAM Group

**What I did:**

- Attached the custom inline policy to the IAM group.

**Command used:**

```bash
aws iam put-group-policy --group-name workshop-s3-readers --policy-name CustomS3ReadPolicy --policy-document file://custom-s3-policy.json
```

**Expected result:**

```text
No output
```

**Result:**

- Group policy was attached successfully.

**Notes:**

- No output usually means the command succeeded.
- Anyone added to the group inherits this policy.

---

### Step 7: Create User and Add User to Group

**What I did:**

- Created an IAM user named `workshop-group-member`.
- Added the user to the `workshop-s3-readers` group.

**Commands used:**

```bash
aws iam create-user --user-name workshop-group-member
```

```bash
aws iam add-user-to-group --user-name workshop-group-member --group-name workshop-s3-readers
```

**Expected result:**

- User creation returns JSON output.
- Adding the user to the group returns no output.

**Result:**

- IAM user was created and added to the group successfully.

**Notes:**

- The user inherits permissions from the group.
- No policy was attached directly to the user.

---

### Step 8: Create Access Keys and Test Group Permissions

#### Step 8a: Create Access Keys

**What I did:**

- Created access keys for `workshop-group-member`.

**Command used:**

```bash
aws iam create-access-key --user-name workshop-group-member
```

**Expected result:**

```json
{
  "AccessKey": {
    "UserName": "workshop-group-member",
    "AccessKeyId": "REDACTED",
    "SecretAccessKey": "REDACTED",
    "Status": "Active"
  }
}
```

**Result:**

- Access keys were created successfully.

**Notes:**

- The secret access key is only shown once.
- Do not upload access keys to GitHub.
- Do not show access keys in screenshots.
- These keys must be deleted during cleanup.

#### Step 8b: Switch CLI to Group Member Credentials

**What I did:**

- Set temporary environment variables for the group member access keys.
- Removed the admin AWS profile so the CLI would use the group member credentials.

**PowerShell commands used:**

```powershell
$env:AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- The output should show `workshop-group-member`.

**Result:**

- AWS CLI switched to the group member identity.

**Notes:**

- If `get-caller-identity` still shows the admin role, `AWS_PROFILE` may still be active.

#### Step 8c: Test List Access

**What I did:**

- Tested whether the group member could list objects in the S3 bucket.

**Command used:**

```bash
aws s3 ls s3://<YOUR_BUCKET_NAME>/
```

**Expected result:**

```text
test-file.txt
```

**Result:**

- Listing worked successfully.

**Why this worked:**

- The group policy allowed `s3:ListBucket`.

#### Step 8d: Test Delete Access

**What I did:**

- Tried to delete the test file as the group member.

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

- The custom group policy contains an explicit Deny for `s3:DeleteObject`.
- Explicit Deny overrides any Allow.

---

### Step 9: Switch Back to Admin Credentials

**What I did:**

- Cleared the group member access key environment variables.
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

- The identity should show the admin role again.

**Result:**

- AWS CLI returned to my admin profile.

**Notes:**

- If the SSO token expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 10: Verify Group Setup in AWS Console

**What I did:**

- Opened the AWS Console.
- Went to IAM.
- Opened **User groups**.
- Selected `workshop-s3-readers`.
- Verified that `workshop-group-member` appeared under the Users tab.
- Verified that `CustomS3ReadPolicy` appeared under the Permissions tab.
- Reviewed the Allow and Deny statements.

**Result:**

- The group, user membership, and inline policy were visible in the AWS Console.

**Notes:**

- This is how administrators can review group-based permissions in a real environment.

---

### Step 11: Enable MFA on Root Account

**What I did:**

- Signed in to the AWS Console as the root user.
- Opened **Security credentials**.
- Went to the MFA section.
- Assigned an authenticator app as the MFA device.
- Scanned the QR code with my authenticator app.
- Entered two consecutive MFA codes.
- Confirmed MFA was enabled.

**Result:**

- MFA was enabled on the AWS root account.

**Notes:**

- This is a console-only step.
- MFA should remain enabled permanently.
- Do not delete the MFA device after the lab.
- Keep backup/recovery options in mind so root account access is not lost.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| Admin identity verified | Passed |
| Local project folder created | Passed |
| S3 bucket created | Passed |
| Test file uploaded | Passed |
| IAM group created | Passed |
| Custom group policy file created | Passed |
| Policy attached to group | Passed |
| IAM user created | Passed |
| User added to group | Passed |
| Access keys created | Passed |
| CLI switched to group member | Passed |
| Group member could list bucket | Passed |
| Group member could not delete object | Passed |
| CLI switched back to admin | Passed |
| Group and policy verified in console | Passed |
| Root MFA enabled | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | Possible Cause | Fix |
|---|---|---|
| `EntityAlreadyExists` when creating group or user | The resource already exists from a previous attempt | Delete the old resource first or use a different name |
| `MalformedPolicyDocument` when attaching policy | JSON syntax error or bucket placeholder was not replaced in all four places | Check the JSON formatting and replace all `<YOUR_BUCKET_NAME>` placeholders |
| `AccessDenied` during delete test | This is expected because explicit Deny is working | No fix needed if this happens during the delete test |
| `get-caller-identity` still shows admin after setting access keys | `AWS_PROFILE` is still overriding the access key environment variables | Run `Remove-Item Env:\AWS_PROFILE` and test again |
| `get-caller-identity` still shows group member during cleanup | Restricted credential environment variables were not cleared | Remove `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION`, then restore `AWS_PROFILE` |
| SSO token expired after switching back to admin | Admin profile uses IAM Identity Center and the token expired | Run `aws sso login --profile <YOUR_PROFILE_NAME>` |
| Cannot find Security credentials | You may be logged in as an Identity Center user instead of root | Sign out and sign in as the root user |
| MFA code rejected | Code expired or phone time is out of sync | Wait for a new code and make sure the phone clock is set automatically |

## Cleanup

### Step 1: Delete Access Key

**What I did:**

- Deleted the access key created for `workshop-group-member`.

**Command used:**

```bash
aws iam delete-access-key --user-name workshop-group-member --access-key-id <ACCESS_KEY_ID>
```

**Expected result:**

```text
No output
```

**Result:**

- Access key was deleted successfully.

**Notes:**

- This is important because access keys are long-term credentials.

---

### Step 2: Remove User from Group

**What I did:**

- Removed `workshop-group-member` from `workshop-s3-readers`.

**Command used:**

```bash
aws iam remove-user-from-group --user-name workshop-group-member --group-name workshop-s3-readers
```

**Expected result:**

```text
No output
```

**Result:**

- User was removed from the group.

---

### Step 3: Delete IAM User

**What I did:**

- Deleted the IAM user.

**Command used:**

```bash
aws iam delete-user --user-name workshop-group-member
```

**Expected result:**

```text
No output
```

**Result:**

- IAM user was deleted.

---

### Step 4: Delete Group Inline Policy

**What I did:**

- Deleted the inline policy attached to the group.

**Command used:**

```bash
aws iam delete-group-policy --group-name workshop-s3-readers --policy-name CustomS3ReadPolicy
```

**Expected result:**

```text
No output
```

**Result:**

- Group inline policy was deleted.

---

### Step 5: Delete IAM Group

**What I did:**

- Deleted the IAM group.

**Command used:**

```bash
aws iam delete-group --group-name workshop-s3-readers
```

**Expected result:**

```text
No output
```

**Result:**

- IAM group was deleted.

---

### Step 6: Empty and Delete S3 Bucket

**What I did:**

- Removed all objects from the S3 bucket.
- Deleted the empty bucket.

**Commands used:**

```bash
aws s3 rm s3://<YOUR_BUCKET_NAME> --recursive
```

```bash
aws s3 rb s3://<YOUR_BUCKET_NAME>
```

**Expected result:**

```text
delete: s3://<YOUR_BUCKET_NAME>/test-file.txt
remove_bucket: <YOUR_BUCKET_NAME>
```

**Result:**

- S3 bucket was emptied and deleted.

---

### Step 7: Delete Local Project Folder

**What I did:**

- Deleted the local project folder from my desktop.

**PowerShell commands used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-3b
```

**Result:**

- Local lab folder was removed.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| IAM access key | Deleted | Yes |
| IAM user | Removed from group and deleted | Yes |
| IAM group inline policy | Deleted | Yes |
| IAM group | Deleted | Yes |
| S3 test object | Deleted | Yes |
| S3 bucket | Deleted | Yes |
| Local lab folder | Deleted | Yes |
| Root MFA | Kept enabled | Yes |

## What I Learned

- IAM groups make permission management easier when multiple users need the same access.
- Users inherit permissions from the groups they belong to.
- Inline group policies can define custom permissions for a group.
- Explicit Deny overrides Allow.
- Implicit Deny means access is denied because no Allow exists.
- Explicit Deny is stronger than implicit Deny because it cannot be overridden by adding another Allow policy.
- Access keys should be treated as sensitive credentials and deleted when no longer needed.
- `aws sts get-caller-identity` is useful for confirming which identity the CLI is using.
- MFA adds a second layer of protection to the root account.
- Root MFA should stay enabled after the lab.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/project-folder-created.png` | Local lab folder created |
| `screenshots/s3-bucket-created.png` | S3 bucket created successfully |
| `screenshots/test-file-uploaded.png` | Test file uploaded to S3 |
| `screenshots/iam-group-created.png` | IAM group created |
| `screenshots/custom-policy-file-created.png` | Custom S3 group policy file created |
| `screenshots/group-policy-attached.png` | Inline policy attached to the IAM group |
| `screenshots/iam-user-created.png` | IAM user created |
| `screenshots/user-added-to-group.png` | User added to IAM group |
| `screenshots/access-key-created-redacted.png` | Access key created with sensitive values redacted |
| `screenshots/group-member-identity.png` | CLI switched to group member identity |
| `screenshots/list-access-success.png` | Group member successfully listed bucket contents |
| `screenshots/delete-access-denied.png` | Group member denied when trying to delete object |
| `screenshots/admin-profile-restored.png` | CLI switched back to admin profile |
| `screenshots/iam-console-group-check.png` | IAM group, user membership, and inline policy verified |
| `screenshots/root-mfa-enabled.png` | Root MFA enabled successfully |
| `screenshots/cleanup-verified.png` | Temporary lab resources cleaned up |