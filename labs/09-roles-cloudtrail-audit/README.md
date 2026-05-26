# Lab 09: Roles, CloudTrail & Audit Logging

## Lab Summary

In this lab, I built a secure multi-user AWS environment using IAM roles and CloudTrail audit logging.

I created a CloudTrail trail to record account activity, created a developer role and an auditor role, tested what each role could and could not do, and used CloudTrail to review the actions performed in the account.

This lab is important because it demonstrates real-world cloud security practices such as role-based access control, temporary credentials, separation of duties, privilege escalation prevention, and audit logging.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 3C — Build a Secure Multi-User Environment with Audit Logging
- Session: 3 — Cloud Security Fundamentals
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Create an S3 bucket for CloudTrail logs
- Apply a bucket policy that allows CloudTrail to write logs
- Create a multi-region CloudTrail trail
- Start CloudTrail logging
- Create a trust policy for IAM roles
- Create a developer role
- Attach a developer policy that allows S3/Lambda activity but blocks IAM changes
- Create an auditor role
- Attach an auditor policy that allows audit read access but blocks modifications
- Assume the developer role using temporary credentials
- Test that the developer can write to S3
- Test that the developer cannot create IAM users
- Switch back to admin credentials
- Assume the auditor role using temporary credentials
- Query CloudTrail events as the auditor
- Test that the auditor cannot make changes
- Review CloudTrail Event history in the AWS Console
- Clean up CloudTrail, IAM roles, S3 bucket, and local files

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS CloudTrail | Records API activity and account events |
| Amazon S3 | Stores CloudTrail log files |
| AWS IAM | Creates roles, policies, and trust relationships |
| IAM Role | Temporary identity used by developer and auditor sessions |
| IAM Trust Policy | Defines who can assume a role |
| IAM Inline Role Policy | Defines what each role can and cannot do |
| AWS STS | Provides temporary credentials through `assume-role` |
| AWS CLI | Creates, tests, audits, and deletes AWS resources |
| PowerShell | Terminal used to run commands |
| JSON | Used to define bucket policies, trust policies, and role policies |
| AWS Console | Used to verify CloudTrail events visually |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- Completed Lab 3A: IAM Least Privilege
- Completed Lab 3B: IAM Groups, Custom Policies & MFA
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- Text editor such as VS Code or Notepad
- Basic understanding of IAM users, groups, policies, roles, and S3

## Cost Notice

Estimated cost: about `$0.023`.

Notes:

- The first CloudTrail trail is generally free.
- IAM is free.
- S3 may create small storage costs for CloudTrail logs.
- Cleanup is required so the CloudTrail trail, IAM roles, and S3 bucket do not remain unnecessarily.

## Key Concepts

| Concept | Meaning |
|---|---|
| CloudTrail | AWS audit logging service that records API calls and account activity. |
| Audit logging | Recording actions so administrators and security teams can investigate what happened. |
| IAM Role | Temporary AWS identity that can be assumed to receive specific permissions. |
| AssumeRole | AWS STS action used to switch into a role and receive temporary credentials. |
| Temporary credentials | Short-lived access key, secret access key, and session token issued by AWS STS. |
| Session token | Extra credential required when using assumed-role credentials. |
| Trust policy | Defines who is allowed to assume a role. |
| Role policy | Defines what actions the assumed role is allowed or denied to perform. |
| Separation of duties | Dividing responsibilities so one person or role does not have too much power. |
| Explicit Deny | A deny statement that overrides allowed permissions. |
| Privilege escalation | Gaining more permissions than intended, such as creating a new admin user. |
| Multi-region trail | A CloudTrail trail that records events from all AWS regions. |

## Security Notes

| Topic | Explanation |
|---|---|
| Developer role | Can perform development-related actions such as S3 writes but cannot modify IAM. |
| Auditor role | Can review CloudTrail and read logs but cannot modify resources. |
| Explicit Deny | Used to prevent privilege escalation and destructive actions. |
| Temporary role credentials | Safer than long-term access keys because they expire. |
| CloudTrail logging | Provides accountability by recording successful and failed actions. |
| Failed actions are logged | Denied attempts, such as blocked IAM user creation, can still appear in CloudTrail. |
| Separation of duties | Developers should not be able to erase evidence, and auditors should not be able to change production resources. |

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my admin identity.
- Recorded my AWS account ID for later policy files.

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

- The account ID is needed for the CloudTrail bucket policy and IAM trust policy.
- If the SSO token is expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

- Created a local folder for this lab.
- Changed into that folder before creating JSON files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-3c
cd ~\Desktop\workshop-lab-3c
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-3c
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-3c`.

**Notes:**

- All JSON files for this lab should be saved in this folder.

---

### Step 3: Create S3 Bucket for CloudTrail Logs

**What I did:**

- Created a globally unique S3 bucket to store CloudTrail logs.

**Command used:**

```bash
aws s3 mb s3://<YOUR_CLOUDTRAIL_BUCKET> --region us-east-1
```

**Expected result:**

```text
make_bucket: <YOUR_CLOUDTRAIL_BUCKET>
```

**Result:**

- CloudTrail log bucket was created successfully.

**Notes:**

- S3 bucket names must be globally unique.
- This bucket stores CloudTrail log files under an `AWSLogs/<ACCOUNT_ID>/` prefix.

---

### Step 4: Create CloudTrail Bucket Policy

**What I did:**

- Created a file named `cloudtrail-bucket-policy.json`.
- Added a policy allowing the CloudTrail service to check the bucket ACL and write log files to the bucket.

**File path:**

```text
cloudtrail-bucket-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSCloudTrailAclCheck",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::<YOUR_CLOUDTRAIL_BUCKET>"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<YOUR_CLOUDTRAIL_BUCKET>/AWSLogs/<YOUR_ACCOUNT_ID>/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}
```

**Result:**

- Bucket policy file was created locally.

**Notes:**

- Replace `<YOUR_CLOUDTRAIL_BUCKET>` in both resource lines.
- Replace `<YOUR_ACCOUNT_ID>` with the 12-digit AWS account ID.
- This policy allows CloudTrail to write logs while ensuring the bucket owner has full control.

---

### Step 5: Apply CloudTrail Bucket Policy

**What I did:**

- Applied the CloudTrail bucket policy to the S3 bucket.

**Command used:**

```bash
aws s3api put-bucket-policy --bucket <YOUR_CLOUDTRAIL_BUCKET> --policy file://cloudtrail-bucket-policy.json
```

**Expected result:**

```text
No output
```

**Result:**

- Bucket policy was applied successfully.

**Notes:**

- No output usually means the command succeeded.

---

### Step 6: Create CloudTrail Trail

**What I did:**

- Created a multi-region CloudTrail trail named `workshop-audit-trail`.

**Command used:**

```bash
aws cloudtrail create-trail --name workshop-audit-trail --s3-bucket-name <YOUR_CLOUDTRAIL_BUCKET> --is-multi-region-trail
```

**Expected result:**

- JSON output showing trail details.

**Result:**

- CloudTrail trail was created successfully.

**Notes:**

- A multi-region trail records activity from all AWS regions, not only `us-east-1`.

---

### Step 7: Start CloudTrail Logging

**What I did:**

- Started logging for the CloudTrail trail.

**Command used:**

```bash
aws cloudtrail start-logging --name workshop-audit-trail
```

**Expected result:**

```text
No output
```

**Result:**

- CloudTrail logging was started.

**Console checkpoint:**

- Opened CloudTrail in the AWS Console.
- Confirmed that `workshop-audit-trail` appeared with logging enabled.

**Notes:**

- CloudTrail events are not always immediate.
- Events can take several minutes to appear in Event history.

---

### Step 8: Create Role Trust Policy

**What I did:**

- Created a file named `role-trust-policy.json`.
- Added a trust policy allowing identities in my AWS account to assume the roles.

**File path:**

```text
role-trust-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAccountToAssume",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<YOUR_ACCOUNT_ID>:root"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Result:**

- Trust policy file was created locally.

**Notes:**

- In this context, `root` means authenticated identities in the account can assume the role if they have permission.
- In production, the trust policy should be more restrictive.

---

### Step 9: Create Developer Policy

**What I did:**

- Created a file named `developer-policy.json`.
- Added permissions allowing S3 actions and Lambda invocation.
- Added an explicit Deny to block IAM modifications.

**File path:**

```text
developer-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowDeveloperActions",
            "Effect": "Allow",
            "Action": [
                "s3:*",
                "lambda:InvokeFunction"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DenyIAMModifications",
            "Effect": "Deny",
            "Action": [
                "iam:CreateUser",
                "iam:DeleteUser",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:AttachUserPolicy",
                "iam:AttachRolePolicy",
                "iam:PutUserPolicy",
                "iam:PutRolePolicy",
                "iam:CreateAccessKey"
            ],
            "Resource": "*"
        }
    ]
}
```

**Result:**

- Developer policy file was created locally.

**Notes:**

- The developer can work with S3 and invoke Lambda.
- The explicit Deny blocks IAM changes that could lead to privilege escalation.

---

### Step 10: Create Developer Role and Attach Policy

**What I did:**

- Created an IAM role named `workshop-developer-role`.
- Attached the developer policy as an inline role policy.

**Commands used:**

```bash
aws iam create-role --role-name workshop-developer-role --assume-role-policy-document file://role-trust-policy.json
```

```bash
aws iam put-role-policy --role-name workshop-developer-role --policy-name DeveloperPolicy --policy-document file://developer-policy.json
```

**Expected result:**

- Role creation returns JSON output.
- Policy attachment returns no output.

**Result:**

- Developer role was created and configured successfully.

---

### Step 11: Create Auditor Policy

**What I did:**

- Created a file named `auditor-policy.json`.
- Added permissions allowing audit read access.
- Added explicit Deny statements to block modifications.

**File path:**

```text
auditor-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowAuditReadAccess",
            "Effect": "Allow",
            "Action": [
                "cloudtrail:LookupEvents",
                "cloudtrail:GetTrailStatus",
                "cloudtrail:DescribeTrails",
                "cloudtrail:ListTrails",
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": "*"
        },
        {
            "Sid": "DenyAllModifications",
            "Effect": "Deny",
            "Action": [
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:DeleteBucket",
                "iam:*",
                "lambda:*",
                "ec2:*"
            ],
            "Resource": "*"
        }
    ]
}
```

**Result:**

- Auditor policy file was created locally.

**Notes:**

- The auditor can review CloudTrail and read S3 data.
- The auditor cannot make infrastructure or security changes.

---

### Step 12: Create Auditor Role and Attach Policy

**What I did:**

- Created an IAM role named `workshop-auditor-role`.
- Attached the auditor policy as an inline role policy.

**Commands used:**

```bash
aws iam create-role --role-name workshop-auditor-role --assume-role-policy-document file://role-trust-policy.json
```

```bash
aws iam put-role-policy --role-name workshop-auditor-role --policy-name AuditorPolicy --policy-document file://auditor-policy.json
```

**Expected result:**

- Role creation returns JSON output.
- Policy attachment returns no output.

**Result:**

- Auditor role was created and configured successfully.

---

### Step 13: Assume Developer Role

**What I did:**

- Used AWS STS to assume the developer role.
- Copied the temporary credentials from the output.
- Set the temporary credentials as environment variables.
- Removed the admin AWS profile environment variable.
- Verified that the CLI identity changed to the developer role.

**Command used to assume role:**

```bash
aws sts assume-role --role-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-developer-role --role-session-name dev-session
```

**PowerShell commands used to set developer credentials:**

```powershell
$env:AWS_ACCESS_KEY_ID="<DEVELOPER_ACCESS_KEY_ID>"
$env:AWS_SECRET_ACCESS_KEY="<DEVELOPER_SECRET_ACCESS_KEY>"
$env:AWS_SESSION_TOKEN="<DEVELOPER_SESSION_TOKEN>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- Output should show `workshop-developer-role/dev-session`.

**Result:**

- AWS CLI switched to the developer role.

**Notes:**

- Assumed role credentials include three values:
  - Access key ID
  - Secret access key
  - Session token
- The session token is required.
- These temporary credentials expire.

---

### Step 14: Test Developer S3 Write Access

**What I did:**

- Created a test file.
- Uploaded it to the CloudTrail log bucket as the developer role.

**PowerShell commands used:**

```powershell
"developer was here" | Out-File dev-test.txt
aws s3 cp dev-test.txt s3://<YOUR_CLOUDTRAIL_BUCKET>/dev-test.txt
```

**Expected result:**

```text
upload: ./dev-test.txt to s3://<YOUR_CLOUDTRAIL_BUCKET>/dev-test.txt
```

**Result:**

- Developer role successfully uploaded a file to S3.

**Why this worked:**

- The developer policy allows `s3:*`.

**Notes:**

- This action should be recorded by CloudTrail.

---

### Step 15: Test Developer Privilege Escalation Attempt

**What I did:**

- Tried to create a new IAM user while using the developer role.

**Command used:**

```bash
aws iam create-user --user-name hacker-attempt 2>&1
```

**Expected result:**

```text
An error occurred (AccessDenied)
```

**Result:**

- Developer role was denied when trying to create an IAM user.

**Why this was expected:**

- The developer policy explicitly denies IAM modification actions.
- This blocks privilege escalation attempts.

---

### Step 16: Switch Back to Admin Credentials

**What I did:**

- Cleared all developer temporary credential environment variables.
- Restored my admin AWS CLI profile.
- Verified that the CLI was using the admin role again.

**PowerShell commands used:**

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_SESSION_TOKEN
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- Output should show the admin role.

**Result:**

- AWS CLI returned to admin credentials.

**Notes:**

- `AWS_SESSION_TOKEN` must be cleared.
- If the SSO token expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 17: Assume Auditor Role

**What I did:**

- Used AWS STS to assume the auditor role.
- Copied the temporary credentials from the output.
- Set the temporary credentials as environment variables.
- Removed the admin AWS profile environment variable.
- Verified that the CLI identity changed to the auditor role.

**Command used to assume role:**

```bash
aws sts assume-role --role-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-auditor-role --role-session-name auditor-session
```

**PowerShell commands used to set auditor credentials:**

```powershell
$env:AWS_ACCESS_KEY_ID="<AUDITOR_ACCESS_KEY_ID>"
$env:AWS_SECRET_ACCESS_KEY="<AUDITOR_SECRET_ACCESS_KEY>"
$env:AWS_SESSION_TOKEN="<AUDITOR_SESSION_TOKEN>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- Output should show `workshop-auditor-role/auditor-session`.

**Result:**

- AWS CLI switched to the auditor role.

---

### Step 18: Query CloudTrail Events as Auditor

**What I did:**

- Queried recent CloudTrail events while using the auditor role.

**Command used:**

```bash
aws cloudtrail lookup-events --max-results 5 --query "Events[].{Time:EventTime,Name:EventName,User:Username}" --output table
```

**Expected result:**

- A table of recent CloudTrail events.

**Result:**

- CloudTrail events were viewable by the auditor role.

**Notes:**

- CloudTrail events can take 5–15 minutes to appear.
- Successful and failed actions can both be logged.
- The developer `PutObject` and failed `CreateUser` attempt may appear after a delay.

---

### Step 19: Test Auditor Modification Attempt

**What I did:**

- Tried to create an IAM user while using the auditor role.

**Command used:**

```bash
aws iam create-user --user-name auditor-hack 2>&1
```

**Expected result:**

```text
An error occurred (AccessDenied)
```

**Result:**

- Auditor role was denied when trying to create an IAM user.

**Why this was expected:**

- The auditor policy explicitly denies IAM modifications.
- This confirms that the auditor can investigate but cannot make changes.

---

### Step 20: Switch Back to Admin Credentials

**What I did:**

- Cleared all auditor temporary credential environment variables.
- Restored my admin AWS CLI profile.
- Verified that the CLI was using the admin role again.

**PowerShell commands used:**

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_SESSION_TOKEN
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- Output should show the admin role.

**Result:**

- AWS CLI returned to admin credentials.

---

### Step 21: Review CloudTrail Event History in AWS Console

**What I did:**

- Opened the CloudTrail console.
- Went to Event history.
- Looked for events such as:
  - `AssumeRole`
  - `PutObject`
  - `CreateUser`

**Result:**

- CloudTrail showed account activity and role activity.

**Notes:**

- CloudTrail is useful for security investigations.
- Failed access attempts can help reveal suspicious activity.
- Event visibility may be delayed.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| Admin identity verified | Passed |
| Local project folder created | Passed |
| CloudTrail S3 bucket created | Passed |
| CloudTrail bucket policy created | Passed |
| Bucket policy applied | Passed |
| CloudTrail trail created | Passed |
| CloudTrail logging started | Passed |
| Trail verified in AWS Console | Passed |
| Role trust policy created | Passed |
| Developer policy created | Passed |
| Developer role created | Passed |
| Developer policy attached | Passed |
| Auditor policy created | Passed |
| Auditor role created | Passed |
| Auditor policy attached | Passed |
| Developer role assumed | Passed |
| Developer S3 write test succeeded | Passed |
| Developer IAM create-user attempt denied | Passed |
| Admin profile restored after developer test | Passed |
| Auditor role assumed | Passed |
| Auditor CloudTrail lookup worked | Passed |
| Auditor IAM create-user attempt denied | Passed |
| Admin profile restored after auditor test | Passed |
| CloudTrail Event history reviewed | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `InsufficientS3BucketPolicyException` when creating the trail | CloudTrail does not have permission to write to the S3 bucket | Check the bucket policy, bucket name, account ID, and apply the policy again |
| `TrailAlreadyExistsException` | The trail already exists from a previous attempt | Delete the existing trail or use a different trail name |
| `MalformedPolicyDocument` | JSON syntax error or placeholder not replaced | Check commas, brackets, quotes, account ID, and bucket name |
| `AccessDenied` when assuming a role | The current admin identity may not have permission, or trust policy is wrong | Verify admin identity and check `role-trust-policy.json` |
| `get-caller-identity` still shows previous role | Temporary environment variables were not fully cleared | Remove `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`, and `AWS_DEFAULT_REGION` |
| `The security token included in the request is expired` | Assumed role credentials expired | Switch back to admin and assume the role again |
| CloudTrail lookup shows few or no events | CloudTrail is not real-time | Wait 5–15 minutes and try again |
| Cleanup fails with AccessDenied | CLI may still be using developer or auditor credentials | Clear all temporary credentials and restore admin profile |
| SSO token expired after switching back to admin | Admin profile uses IAM Identity Center and the token expired | Run `aws sso login --profile <YOUR_PROFILE_NAME>` |

## Cleanup

### Step 1: Confirm Admin Identity

**What I did:**

- Confirmed that cleanup was being performed with admin credentials.

**Command used:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- Output should show the admin role.

**Notes:**

- Do not clean up while using the developer or auditor role.

---

### Step 2: Stop CloudTrail Logging

**What I did:**

- Stopped CloudTrail logging.

**Command used:**

```bash
aws cloudtrail stop-logging --name workshop-audit-trail
```

**Expected result:**

```text
No output
```

**Result:**

- CloudTrail logging stopped.

---

### Step 3: Delete CloudTrail Trail

**What I did:**

- Deleted the CloudTrail trail.

**Command used:**

```bash
aws cloudtrail delete-trail --name workshop-audit-trail
```

**Expected result:**

```text
No output
```

**Result:**

- CloudTrail trail was deleted.

---

### Step 4: Delete Developer Role Policy and Role

**What I did:**

- Deleted the inline policy attached to the developer role.
- Deleted the developer role.

**Commands used:**

```bash
aws iam delete-role-policy --role-name workshop-developer-role --policy-name DeveloperPolicy
```

```bash
aws iam delete-role --role-name workshop-developer-role
```

**Expected result:**

```text
No output
```

**Result:**

- Developer policy and role were deleted.

---

### Step 5: Delete Auditor Role Policy and Role

**What I did:**

- Deleted the inline policy attached to the auditor role.
- Deleted the auditor role.

**Commands used:**

```bash
aws iam delete-role-policy --role-name workshop-auditor-role --policy-name AuditorPolicy
```

```bash
aws iam delete-role --role-name workshop-auditor-role
```

**Expected result:**

```text
No output
```

**Result:**

- Auditor policy and role were deleted.

---

### Step 6: Empty and Delete CloudTrail S3 Bucket

**What I did:**

- Removed all objects from the CloudTrail log bucket.
- Deleted the empty bucket.

**Commands used:**

```bash
aws s3 rm s3://<YOUR_CLOUDTRAIL_BUCKET> --recursive
```

```bash
aws s3 rb s3://<YOUR_CLOUDTRAIL_BUCKET>
```

**Expected result:**

```text
delete: ...
remove_bucket: <YOUR_CLOUDTRAIL_BUCKET>
```

**Result:**

- CloudTrail bucket was emptied and deleted.

---

### Step 7: Delete Local Project Folder

**What I did:**

- Deleted the local lab folder from my desktop.

**PowerShell commands used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-3c
```

**Result:**

- Local lab folder was removed.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| CloudTrail logging | Stopped | Yes |
| CloudTrail trail | Deleted | Yes |
| Developer role policy | Deleted | Yes |
| Developer role | Deleted | Yes |
| Auditor role policy | Deleted | Yes |
| Auditor role | Deleted | Yes |
| CloudTrail S3 objects | Deleted | Yes |
| CloudTrail S3 bucket | Deleted | Yes |
| Local lab folder | Deleted | Yes |

## What I Learned

- CloudTrail records AWS account activity for auditing and investigations.
- CloudTrail can log both successful actions and failed attempts.
- IAM roles provide temporary credentials through AWS STS.
- Assumed role credentials require an access key, secret access key, and session token.
- Developer and auditor roles should have different permissions.
- Developers can be allowed to perform technical work while being blocked from IAM changes.
- Auditors can be allowed to review logs without being allowed to modify resources.
- Explicit Deny is useful for preventing privilege escalation.
- Separation of duties reduces risk because no single role has all permissions.
- CloudTrail events may take several minutes to appear.
- It is important to confirm the active AWS identity before testing or cleanup.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/project-folder-created.png` | Local lab folder created |
| `screenshots/cloudtrail-bucket-created.png` | S3 bucket created for CloudTrail logs |
| `screenshots/cloudtrail-bucket-policy-created.png` | CloudTrail bucket policy file created |
| `screenshots/cloudtrail-bucket-policy-applied.png` | Bucket policy applied successfully |
| `screenshots/cloudtrail-trail-created.png` | CloudTrail trail created |
| `screenshots/cloudtrail-logging-enabled.png` | CloudTrail logging started |
| `screenshots/role-trust-policy-created.png` | IAM role trust policy created |
| `screenshots/developer-role-created.png` | Developer role created |
| `screenshots/developer-policy-attached.png` | Developer policy attached |
| `screenshots/auditor-role-created.png` | Auditor role created |
| `screenshots/auditor-policy-attached.png` | Auditor policy attached |
| `screenshots/developer-role-assumed.png` | CLI switched to developer role |
| `screenshots/developer-s3-upload-success.png` | Developer successfully uploaded to S3 |
| `screenshots/developer-iam-denied.png` | Developer denied when attempting IAM user creation |
| `screenshots/admin-restored-after-dev.png` | Admin credentials restored after developer test |
| `screenshots/auditor-role-assumed.png` | CLI switched to auditor role |
| `screenshots/auditor-cloudtrail-lookup.png` | Auditor successfully queried CloudTrail events |
| `screenshots/auditor-iam-denied.png` | Auditor denied when attempting IAM user creation |
| `screenshots/cloudtrail-event-history.png` | CloudTrail Event history reviewed in the console |
| `screenshots/cleanup-verified.png` | Temporary lab resources cleaned up |