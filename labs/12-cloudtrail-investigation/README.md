# Lab 12: CloudTrail Investigation

## Lab Summary

In this lab, I used AWS CloudTrail to investigate suspicious activity in an AWS account.

I created a CloudTrail trail, delivered logs to an S3 bucket, simulated suspicious activity by creating an evidence bucket, uploading sensitive data, creating an IAM user, creating access keys, deleting the access keys, and deleting the user. Then I used CloudTrail Event History and `lookup-events` to prove that the actions were still recorded even after the user and access key were deleted.

This lab is important because CloudTrail is one of the main services used for cloud auditing, incident investigation, and forensic analysis in AWS.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 4B — Investigate Activity with CloudTrail
- Session: 4 — Threat Detection & AWS Security Services
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Confirm any previous CloudTrail trails are deleted
- Create a local project folder
- Create an S3 bucket for CloudTrail logs
- Create and apply a CloudTrail bucket policy
- Create and start a multi-region CloudTrail trail
- Simulate suspicious activity
- Create an evidence S3 bucket
- Upload a sensitive test file
- Create a suspicious IAM user
- Create access keys for the suspicious user
- Delete the access keys
- Delete the suspicious user
- Wait for CloudTrail events to appear
- Investigate recent account activity with CloudTrail
- Filter events by event name
- Prove deleted resources still leave an audit trail
- Review CloudTrail Event History in the AWS Console
- Clean up all lab resources

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS CloudTrail | Records AWS API activity and account events |
| Amazon S3 | Stores CloudTrail logs and the evidence test file |
| AWS IAM | Creates and deletes the suspicious test user and access key |
| AWS CLI | Creates resources, simulates activity, and investigates CloudTrail events |
| PowerShell | Terminal used to run AWS CLI commands |
| JSON | Used to define the CloudTrail bucket policy |
| AWS Console | Used to visually review CloudTrail Event History |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- Text editor such as VS Code or Notepad
- Basic understanding of S3, IAM, and CloudTrail
- Active CloudTrail trail from Lab 3C should be deleted before starting this lab

## Cost Notice

Estimated cost: `$0.00`

Notes:

- The first active CloudTrail trail is generally free for management events.
- IAM is free.
- S3 storage for this lab should be very small.
- Cleanup is required so the CloudTrail trail and S3 buckets do not remain unnecessarily.

## Key Concepts

| Concept | Meaning |
|---|---|
| CloudTrail | AWS service that records API activity in an AWS account. |
| Trail | A CloudTrail configuration that sends log files to an S3 bucket. |
| Event History | CloudTrail view of recent management events. |
| Lookup Events | CloudTrail API used to search event history from the CLI. |
| Management Event | An event related to control-plane actions such as creating users, buckets, keys, or policies. |
| Immutable Audit Trail | Evidence that remains even after a resource is deleted. |
| Forensic Investigation | Reviewing evidence after a security incident to understand what happened. |
| Suspicious User | A test IAM user created in this lab to simulate attacker behavior. |
| Evidence Bucket | S3 bucket used to simulate sensitive data staging or exfiltration. |

## Security Notes

| Topic | Explanation |
|---|---|
| Deleted resources still leave records | Even after the suspicious IAM user and access key were deleted, CloudTrail still recorded their creation and deletion. |
| CloudTrail is not instant | Events can take 5–15 minutes to appear in Event History. |
| Event actor matters | In this lab, the admin identity creates and deletes the suspicious user, so the events are logged under the admin identity. |
| Suspicious user may show no actor events | Since the suspicious user's credentials are created but not used to make API calls, filtering by `Username=suspicious-user` may return no results. |
| Logs support investigations | CloudTrail can help answer who did what, when, from where, and against which resource. |
| Cleanup matters | CloudTrail trails and S3 buckets should be removed after the lab if they are no longer needed. |

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my current AWS identity.
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

- AWS CLI returned my active identity successfully.

**Notes:**

- The account ID is needed for the CloudTrail bucket policy.
- If the SSO token is expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Check for Existing CloudTrail Trails

**What I did:**

- Checked whether any previous CloudTrail trails still existed.

**Command used:**

```bash
aws cloudtrail describe-trails --query "trailList[].{Name:Name,Status:Status}" --region us-east-1
```

**Expected result if no trail exists:**

```json
[]
```

**Result:**

- Existing trails were checked before continuing.

**Notes:**

- The source lab recommends making sure the active trail from Lab 3C has been deleted before starting this lab.
- If a previous trail still exists, clean it up before continuing.

---

### Step 3: Create Local Project Folder

**What I did:**

- Created a local folder for this lab.
- Changed into the folder before creating JSON files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-4b
cd ~\Desktop\workshop-lab-4b
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-4b
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-4b`.

---

### Step 4: Create S3 Bucket for CloudTrail Logs

**What I did:**

- Created a globally unique S3 bucket for CloudTrail logs.

**Command used:**

```bash
aws s3 mb s3://<YOUR_TRAIL_LOG_BUCKET> --region us-east-1
```

**Expected result:**

```text
make_bucket: <YOUR_TRAIL_LOG_BUCKET>
```

**Result:**

- CloudTrail log bucket was created successfully.

**Notes:**

- S3 bucket names must be globally unique.
- Do not include angle brackets when running the command.

---

### Step 5: Create CloudTrail Bucket Policy

**What I did:**

- Created a file named `cloudtrail-bucket-policy.json`.
- Added a policy that allows CloudTrail to check the bucket ACL and write log files to the bucket.

**File created:**

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
            "Resource": "arn:aws:s3:::<YOUR_TRAIL_LOG_BUCKET>"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<YOUR_TRAIL_LOG_BUCKET>/AWSLogs/<YOUR_ACCOUNT_ID>/*",
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

- CloudTrail bucket policy file was created locally.

**Notes:**

- Replace `<YOUR_TRAIL_LOG_BUCKET>` in both resource lines.
- Replace `<YOUR_ACCOUNT_ID>` with the 12-digit account ID from `aws sts get-caller-identity`.
- Make sure the file is saved as `.json`, not `.json.txt`.

---

### Step 6: Apply CloudTrail Bucket Policy

**What I did:**

- Applied the bucket policy to the CloudTrail log bucket.

**Command used:**

```bash
aws s3api put-bucket-policy --bucket <YOUR_TRAIL_LOG_BUCKET> --policy file://cloudtrail-bucket-policy.json
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

### Step 7: Create CloudTrail Trail

**What I did:**

- Created a multi-region CloudTrail trail named `workshop-investigation-trail`.

**Command used:**

```bash
aws cloudtrail create-trail --name workshop-investigation-trail --s3-bucket-name <YOUR_TRAIL_LOG_BUCKET> --is-multi-region-trail
```

**Expected result:**

- JSON output showing the trail details.

**Result:**

- CloudTrail trail was created successfully.

**Notes:**

- A multi-region trail records activity from all AWS regions.

---

### Step 8: Start CloudTrail Logging

**What I did:**

- Started logging for the trail.

**Command used:**

```bash
aws cloudtrail start-logging --name workshop-investigation-trail
```

**Expected result:**

```text
No output
```

**Result:**

- CloudTrail logging started successfully.

---

### Step 9: Create Evidence S3 Bucket

**What I did:**

- Created a second S3 bucket to simulate an evidence/data staging bucket.

**Command used:**

```bash
aws s3 mb s3://<YOUR_EVIDENCE_BUCKET> --region us-east-1
```

**Expected result:**

```text
make_bucket: <YOUR_EVIDENCE_BUCKET>
```

**Result:**

- Evidence bucket was created successfully.

---

### Step 10: Upload Sensitive Test File

**What I did:**

- Created a local file named `sensitive-data.txt`.
- Uploaded it to the evidence bucket.

**PowerShell commands used:**

```powershell
"CONFIDENTIAL: Employee salary data Q4 2025" | Out-File sensitive-data.txt
aws s3 cp sensitive-data.txt s3://<YOUR_EVIDENCE_BUCKET>/sensitive-data.txt
```

**Expected result:**

```text
upload: ./sensitive-data.txt to s3://<YOUR_EVIDENCE_BUCKET>/sensitive-data.txt
```

**Result:**

- Sensitive test file was uploaded successfully.

**Notes:**

- This simulates suspicious data staging or exfiltration preparation.

---

### Step 11: Create Suspicious IAM User

**What I did:**

- Created a test IAM user named `suspicious-user`.

**Command used:**

```bash
aws iam create-user --user-name suspicious-user
```

**Expected result:**

- JSON output showing `"UserName": "suspicious-user"`.

**Result:**

- Suspicious test user was created successfully.

---

### Step 12: Create Access Key for Suspicious User

**What I did:**

- Created an access key for `suspicious-user`.

**Command used:**

```bash
aws iam create-access-key --user-name suspicious-user
```

**Expected result:**

```json
{
  "AccessKey": {
    "UserName": "suspicious-user",
    "AccessKeyId": "REDACTED",
    "SecretAccessKey": "REDACTED",
    "Status": "Active"
  }
}
```

**Result:**

- Access key was created successfully.

**Notes:**

- Record the Access Key ID for cleanup.
- Do not commit access keys to GitHub.
- Do not include access keys in screenshots.

---

### Step 13: Delete Suspicious User Access Key

**What I did:**

- Deleted the access key for `suspicious-user`.

**Command used:**

```bash
aws iam delete-access-key --user-name suspicious-user --access-key-id <SUSPICIOUS_USER_ACCESS_KEY_ID>
```

**Expected result:**

```text
No output
```

**Result:**

- Suspicious user's access key was deleted.

**Notes:**

- This simulates an attacker trying to remove evidence.

---

### Step 14: Delete Suspicious IAM User

**What I did:**

- Deleted the suspicious IAM user.

**Command used:**

```bash
aws iam delete-user --user-name suspicious-user
```

**Expected result:**

```text
No output
```

**Result:**

- Suspicious user was deleted.

**Notes:**

- The user is gone, but CloudTrail should still record the creation and deletion events.

---

### Step 15: Wait for CloudTrail Events

**What I did:**

- Waited for CloudTrail Event History to update.

**Expected wait time:**

```text
5–15 minutes
```

**Result:**

- Waited before querying CloudTrail.

**Notes:**

- CloudTrail is not always immediate.
- Empty results can simply mean the events have not appeared yet.

---

### Step 16: List Recent CloudTrail Events

**What I did:**

- Queried recent CloudTrail events.

**Command used:**

```bash
aws cloudtrail lookup-events --max-results 10 --query "Events[].{Time:EventTime,Name:EventName,User:Username,Source:EventSource}" --output table
```

**Expected result:**

- Recent API calls such as:
  - `CreateUser`
  - `CreateAccessKey`
  - `DeleteAccessKey`
  - `DeleteUser`
  - `CreateBucket`
  - `PutObject`

**Result:**

- Recent CloudTrail events were displayed.

---

### Step 17: Filter for User Creation Events

**What I did:**

- Searched CloudTrail for `CreateUser` events.

**Command used:**

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser --query "Events[].{Time:EventTime,User:Username,Affected:Resources[0].ResourceName}" --output table
```

**Expected result:**

- A `CreateUser` event showing that `suspicious-user` was created.

**Result:**

- CloudTrail showed the user creation event.

**Notes:**

- This proves the user creation was logged even if the user no longer exists.

---

### Step 18: Filter by Suspicious Username

**What I did:**

- Searched CloudTrail for events where `suspicious-user` was the actor.

**Command used:**

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=suspicious-user --query "Events[].{Time:EventTime,Name:EventName,Source:EventSource}" --output table
```

**Expected result:**

```text
Empty or no matching events
```

**Result:**

- This search may return no events.

**Notes:**

- This is expected because `suspicious-user` was created but its credentials were not used to perform actions.
- The creation and deletion of `suspicious-user` are logged under the admin identity that performed those actions.

---

### Step 19: Find User Deletion Event

**What I did:**

- Searched CloudTrail for `DeleteUser` events.

**Command used:**

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteUser --query "Events[].{Time:EventTime,User:Username,Affected:Resources[0].ResourceName,Action:EventName}" --output table
```

**Expected result:**

- A `DeleteUser` event showing that `suspicious-user` was deleted.

**Result:**

- CloudTrail showed the user deletion event.

---

### Step 20: Find Access Key Deletion Event

**What I did:**

- Searched CloudTrail for `DeleteAccessKey` events.

**Command used:**

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteAccessKey --query "Events[].{Time:EventTime,Actor:Username,Affected:Resources[0].ResourceName,Action:EventName}" --output table
```

**Expected result:**

- A `DeleteAccessKey` event showing the key deletion activity.

**Result:**

- CloudTrail showed the access key deletion event.

**Notes:**

- Even though the access key was deleted, CloudTrail still recorded the action.

---

### Step 21: Review CloudTrail Event History in Console

**What I did:**

- Opened the AWS Console.
- Went to CloudTrail.
- Opened Event history.
- Filtered by event names:
  - `CreateUser`
  - `DeleteUser`
  - `DeleteAccessKey`
- Opened events to review details such as:
  - Event time
  - Event source
  - Username
  - Source IP address
  - Request parameters
  - Affected resources

**Result:**

- CloudTrail Event History showed a complete audit trail.

**Notes:**

- This is how security teams investigate suspicious activity after an alert.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| Previous trail check completed | Passed |
| Local project folder created | Passed |
| CloudTrail log bucket created | Passed |
| CloudTrail bucket policy created | Passed |
| CloudTrail bucket policy applied | Passed |
| CloudTrail trail created | Passed |
| CloudTrail logging started | Passed |
| Evidence bucket created | Passed |
| Sensitive test file uploaded | Passed |
| Suspicious IAM user created | Passed |
| Suspicious user access key created | Passed |
| Suspicious user access key deleted | Passed |
| Suspicious IAM user deleted | Passed |
| Recent CloudTrail events queried | Passed |
| `CreateUser` event found | Passed |
| Suspicious username filter tested | Passed |
| `DeleteUser` event found | Passed |
| `DeleteAccessKey` event found | Passed |
| Event history reviewed in AWS Console | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `InsufficientS3BucketPolicyException` when creating the trail | CloudTrail does not have permission to write to the S3 bucket | Check `cloudtrail-bucket-policy.json`, verify bucket name and account ID, then reapply the bucket policy |
| `TrailAlreadyExistsException` | A trail with the same name already exists | Delete the old trail or use a different trail name |
| `MalformedPolicyDocument` | JSON syntax error or placeholder was not replaced | Check commas, brackets, quotes, bucket name, and account ID |
| `EntityAlreadyExists` when creating `suspicious-user` | The user exists from a previous attempt | Delete the old user or use a different test username |
| `NoSuchEntity` when deleting the user | The user was already deleted or never created | Continue if cleanup is already complete |
| `lookup-events` returns empty results | CloudTrail events may not have appeared yet | Wait 5–15 minutes and try again |
| Filtering by `Username=suspicious-user` returns no events | The suspicious user was created but never used to make API calls | Look for `CreateUser` and `DeleteUser` events under the admin actor instead |
| SSO token expired | Admin SSO session expired | Run `aws sso login --profile <YOUR_PROFILE_NAME>` |

## Cleanup

### Step 1: Stop CloudTrail Logging

**What I did:**

- Stopped logging for the CloudTrail trail.

**Command used:**

```bash
aws cloudtrail stop-logging --name workshop-investigation-trail
```

**Expected result:**

```text
No output
```

**Result:**

- CloudTrail logging stopped.

---

### Step 2: Delete CloudTrail Trail

**What I did:**

- Deleted the CloudTrail trail.

**Command used:**

```bash
aws cloudtrail delete-trail --name workshop-investigation-trail
```

**Expected result:**

```text
No output
```

**Result:**

- CloudTrail trail was deleted.

---

### Step 3: Empty and Delete Trail Log Bucket

**What I did:**

- Removed all CloudTrail log objects.
- Deleted the trail log bucket.

**Commands used:**

```bash
aws s3 rm s3://<YOUR_TRAIL_LOG_BUCKET> --recursive
```

```bash
aws s3 rb s3://<YOUR_TRAIL_LOG_BUCKET>
```

**Expected result:**

```text
remove_bucket: <YOUR_TRAIL_LOG_BUCKET>
```

**Result:**

- Trail log bucket was emptied and deleted.

---

### Step 4: Empty and Delete Evidence Bucket

**What I did:**

- Removed the sensitive test file.
- Deleted the evidence bucket.

**Commands used:**

```bash
aws s3 rm s3://<YOUR_EVIDENCE_BUCKET> --recursive
```

```bash
aws s3 rb s3://<YOUR_EVIDENCE_BUCKET>
```

**Expected result:**

```text
remove_bucket: <YOUR_EVIDENCE_BUCKET>
```

**Result:**

- Evidence bucket was emptied and deleted.

---

### Step 5: Delete Local Project Folder

**What I did:**

- Deleted the local lab folder from my desktop.

**PowerShell command used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-4b
```

**Result:**

- Local lab folder was deleted.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| CloudTrail logging | Stopped | Yes |
| CloudTrail trail | Deleted | Yes |
| Trail log bucket objects | Deleted | Yes |
| Trail log bucket | Deleted | Yes |
| Evidence bucket objects | Deleted | Yes |
| Evidence bucket | Deleted | Yes |
| Suspicious IAM user | Deleted during lab | Yes |
| Suspicious access key | Deleted during lab | Yes |
| Local lab folder | Deleted | Yes |

## What I Learned

- CloudTrail records API activity in an AWS account.
- CloudTrail can help answer who did what, when, from where, and against which resource.
- A deleted IAM user can still have creation and deletion events recorded in CloudTrail.
- A deleted access key can still have creation and deletion events recorded in CloudTrail.
- CloudTrail events may take several minutes to appear.
- Filtering by event name is useful for investigations.
- Filtering by username only shows events where that username was the actor.
- In this lab, `suspicious-user` was created and deleted by the admin identity, so the events appear under the admin actor.
- CloudTrail is a key service for cloud forensics, auditing, compliance, and incident response.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/existing-trails-checked.png` | Existing CloudTrail trails checked before starting |
| `screenshots/project-folder-created.png` | Local project folder created |
| `screenshots/trail-log-bucket-created.png` | CloudTrail log bucket created |
| `screenshots/cloudtrail-bucket-policy-created.png` | Bucket policy file created |
| `screenshots/cloudtrail-bucket-policy-applied.png` | Bucket policy applied |
| `screenshots/cloudtrail-trail-created.png` | CloudTrail investigation trail created |
| `screenshots/cloudtrail-logging-started.png` | CloudTrail logging started |
| `screenshots/evidence-bucket-created.png` | Evidence S3 bucket created |
| `screenshots/sensitive-file-uploaded.png` | Sensitive test file uploaded |
| `screenshots/suspicious-user-created.png` | Suspicious IAM user created |
| `screenshots/access-key-created-redacted.png` | Access key created with sensitive values redacted |
| `screenshots/access-key-deleted.png` | Suspicious access key deleted |
| `screenshots/suspicious-user-deleted.png` | Suspicious IAM user deleted |
| `screenshots/recent-events-cli.png` | Recent CloudTrail events listed from CLI |
| `screenshots/createuser-event-found.png` | `CreateUser` event found |
| `screenshots/deleteuser-event-found.png` | `DeleteUser` event found |
| `screenshots/deleteaccesskey-event-found.png` | `DeleteAccessKey` event found |
| `screenshots/cloudtrail-console-event-history.png` | CloudTrail Event History reviewed in console |
| `screenshots/cleanup-verified.png` | Lab cleanup verified |