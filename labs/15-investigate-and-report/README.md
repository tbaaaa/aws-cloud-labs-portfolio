# Lab 15: Investigate and Report

## Lab Summary

In this lab, I practiced investigating a simulated AWS security incident and writing an incident report.

I configured CloudTrail to deliver logs to both an S3 bucket and CloudWatch Logs, enabled S3 data event logging for an evidence bucket, created a simulated attacker IAM user, used the attacker credentials to list buckets and download a sensitive file, investigated the activity with CloudTrail and CloudWatch Logs Insights, revoked the attacker’s credentials, deleted the attacker user, and documented the incident.

This lab is important because it shows the difference between basic CloudTrail Event History and a more complete forensic logging setup that captures S3 object-level activity.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 5B — Investigate an Incident & Write a Report
- Session: 5 — Incident Response on AWS
- My purpose: Document my own process, commands, validation steps, investigation notes, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Create an evidence S3 bucket
- Create a CloudTrail logging S3 bucket
- Create a CloudTrail bucket policy
- Create a CloudWatch log group
- Create an IAM role that allows CloudTrail to write to CloudWatch Logs
- Create a CloudTrail trail
- Enable S3 data event logging for the evidence bucket
- Start CloudTrail logging
- Upload a sensitive test file
- Create a simulated attacker IAM user
- Attach S3 read permissions to the attacker user
- Create attacker access keys
- Switch to attacker credentials
- Simulate reconnaissance and data exfiltration
- Switch back to admin credentials
- Investigate management events with CloudTrail `lookup-events`
- Investigate full activity with CloudWatch Logs Insights
- Isolate the `GetObject` data exfiltration event
- Deactivate the attacker access key
- Delete the attacker access key, policy, and user
- Write an incident report
- Clean up all lab resources

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS CloudTrail | Records AWS account activity |
| CloudTrail Data Events | Records S3 object-level actions such as `GetObject` |
| Amazon CloudWatch Logs | Stores CloudTrail logs for querying |
| CloudWatch Logs Insights | Queries the attacker timeline and exfiltration event |
| Amazon S3 | Stores the sensitive test file and CloudTrail logs |
| AWS IAM | Creates the attacker user and CloudTrail logging role |
| AWS STS | Verifies active identity with `get-caller-identity` |
| AWS CLI | Creates resources, simulates activity, investigates events, and cleans up |
| PowerShell | Terminal used to run commands |
| JSON | Used to define policies and event selectors |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- Completed Lab 5A: Credential Revocation
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- Text editor such as VS Code or Notepad
- Basic understanding of IAM users, access keys, S3, CloudTrail, and incident response

## Cost Notice

Estimated cost: `$0.00`

Notes:

- IAM is free.
- S3 costs should be near `$0.00` because this lab uses very small files.
- CloudTrail management events for the first trail are generally free.
- S3 data events have a small cost, but this lab generates very few events.
- CloudWatch Logs and Logs Insights should remain near `$0.00` because the log volume is tiny.
- Cleanup is required so CloudTrail, CloudWatch Logs, IAM, and S3 resources do not remain unnecessarily.

## Key Concepts

| Concept | Meaning |
|---|---|
| Incident Response | The process of preparing for, detecting, containing, eradicating, recovering from, and learning from security incidents. |
| NIST IR Lifecycle | Common incident response model: Preparation, Detection & Analysis, Containment, Eradication, Recovery, and Lessons Learned. |
| Management Event | CloudTrail event for control-plane activity, such as creating users or buckets. |
| Data Event | CloudTrail event for data-plane activity, such as reading or downloading an S3 object. |
| `lookup-events` | AWS CLI command that queries CloudTrail Event History for management events. |
| CloudWatch Logs Insights | Query tool used to search CloudTrail logs streamed into CloudWatch. |
| Forensic Timeline | Chronological reconstruction of attacker activity. |
| Data Exfiltration | Unauthorized copying or downloading of data. |
| Access Key ID Pivot | Using a suspicious access key ID to find all activity performed with that key. |
| Incident Report | Formal document explaining what happened, evidence found, actions taken, and recommendations. |

## Security Notes

| Topic | Explanation |
|---|---|
| Logging must exist before the attack | If data event logging is not enabled before the S3 download, the `GetObject` evidence may not exist. |
| Event History is limited | CloudTrail Event History and `lookup-events` do not show S3 object downloads. |
| CloudWatch Logs gives fuller evidence | CloudTrail streaming to CloudWatch Logs allows querying management and data events together. |
| Attacker credentials are temporary for the lab | The attacker access key is created only for simulation and must be deleted. |
| Do not commit secrets | Access keys, secret keys, and account IDs should be redacted before screenshots or GitHub commits. |
| Evidence source matters | The incident report should distinguish between CLI Event History evidence and CloudWatch Logs Insights evidence. |

## Architecture Overview

```text
Attacker IAM User
      |
      | uses access key
      v
S3 Evidence Bucket
      |
      | GetObject data event
      v
CloudTrail Trail
      |
      +--> S3 CloudTrail Log Bucket
      |
      +--> CloudWatch Logs
              |
              v
      CloudWatch Logs Insights
              |
              v
      Forensic Timeline + Incident Report
```

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my current admin identity.

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
- Changed into the folder before creating JSON files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-5b
cd ~\Desktop\workshop-lab-5b
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-5b
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-5b`.

---

### Step 3: Get AWS Account ID

**What I did:**

- Retrieved my AWS account ID for later policy files.

**Command used:**

```bash
aws sts get-caller-identity --query Account --output text
```

**Expected result:**

```text
123456789012
```

**Result:**

- Account ID was recorded.

---

### Step 4: Create Evidence Bucket

**What I did:**

- Created an S3 bucket that will store the simulated sensitive file.

**Command used:**

```bash
aws s3 mb s3://<STUDENT>-incident-evidence --region us-east-1
```

**Expected result:**

```text
make_bucket: <STUDENT>-incident-evidence
```

**Result:**

- Evidence bucket was created successfully.

---

### Step 5: Create CloudTrail Logging Bucket

**What I did:**

- Created a dedicated S3 bucket for CloudTrail logs.

**Command used:**

```bash
aws s3 mb s3://<STUDENT>-cloudtrail-logs --region us-east-1
```

**Expected result:**

```text
make_bucket: <STUDENT>-cloudtrail-logs
```

**Result:**

- CloudTrail log bucket was created successfully.

---

### Step 6: Create CloudTrail Bucket Policy

**What I did:**

- Created a file named `trail-policy.json`.
- Added a policy allowing CloudTrail to check the bucket ACL and write logs.

**File created:**

```text
trail-policy.json
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
            "Resource": "arn:aws:s3:::<STUDENT>-cloudtrail-logs"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<STUDENT>-cloudtrail-logs/AWSLogs/<ACCOUNT_ID>/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}
```

**Apply policy command:**

```bash
aws s3api put-bucket-policy --bucket <STUDENT>-cloudtrail-logs --policy file://trail-policy.json
```

**Expected result:**

```text
No output
```

**Result:**

- CloudTrail bucket policy was applied successfully.

---

### Step 7: Create CloudWatch Log Group

**What I did:**

- Created a CloudWatch log group for CloudTrail logs.
- Set a 30-day retention policy.

**Commands used:**

```bash
aws logs create-log-group --log-group-name lab5b-cloudtrail-logs --region us-east-1
```

```bash
aws logs put-retention-policy --log-group-name lab5b-cloudtrail-logs --retention-in-days 30 --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

- Log group and retention policy were created successfully.

---

### Step 8: Create CloudTrail IAM Role

**What I did:**

- Created a trust policy file named `cloudtrail-trust-policy.json`.
- Created an IAM role that CloudTrail can assume.

**File created:**

```text
cloudtrail-trust-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Create role command:**

```bash
aws iam create-role --role-name lab5b-cloudtrail-role --assume-role-policy-document file://cloudtrail-trust-policy.json
```

**Result:**

- IAM role for CloudTrail was created.

---

### Step 9: Attach CloudWatch Logs Permission to Role

**What I did:**

- Created a file named `cloudtrail-logs-policy.json`.
- Gave the CloudTrail role permission to create log streams and put log events.

**File created:**

```text
cloudtrail-logs-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:lab5b-cloudtrail-logs:*"
        }
    ]
}
```

**Attach policy command:**

```bash
aws iam put-role-policy --role-name lab5b-cloudtrail-role --policy-name CloudWatchLogsWrite --policy-document file://cloudtrail-logs-policy.json
```

**Expected result:**

```text
No output
```

**Result:**

- CloudTrail role received CloudWatch Logs write permissions.

---

### Step 10: Create CloudTrail Trail

**What I did:**

- Checked for existing trails.
- Created a CloudTrail trail named `lab5b-trail`.
- Configured it to deliver logs to S3 and CloudWatch Logs.

**Check existing trails:**

```bash
aws cloudtrail describe-trails --query "trailList[].Name" --output text
```

**Create trail command:**

```bash
aws cloudtrail create-trail --name lab5b-trail --s3-bucket-name <STUDENT>-cloudtrail-logs --cloud-watch-logs-log-group-arn arn:aws:logs:us-east-1:<ACCOUNT_ID>:log-group:lab5b-cloudtrail-logs:* --cloud-watch-logs-role-arn arn:aws:iam::<ACCOUNT_ID>:role/lab5b-cloudtrail-role --region us-east-1
```

**Result:**

- CloudTrail trail was created successfully.

---

### Step 11: Enable S3 Data Event Logging

**What I did:**

- Created a file named `event-selectors.json`.
- Configured CloudTrail to capture object-level activity on the evidence bucket.

**File created:**

```text
event-selectors.json
```

**File content:**

```json
[
    {
        "ReadWriteType": "All",
        "IncludeManagementEvents": true,
        "DataResources": [
            {
                "Type": "AWS::S3::Object",
                "Values": ["arn:aws:s3:::<STUDENT>-incident-evidence/"]
            }
        ]
    }
]
```

**Apply event selector command:**

```bash
aws cloudtrail put-event-selectors --trail-name lab5b-trail --event-selectors file://event-selectors.json
```

**Result:**

- S3 data event logging was enabled for the evidence bucket.

**Notes:**

- This is the key step that allows the `GetObject` file download to be recorded.

---

### Step 12: Start CloudTrail Logging

**What I did:**

- Started the CloudTrail trail.
- Verified that logging was active.

**Commands used:**

```bash
aws cloudtrail start-logging --name lab5b-trail
```

```bash
aws cloudtrail get-trail-status --name lab5b-trail --query "{IsLogging:IsLogging,LatestDeliveryError:LatestDeliveryError}"
```

**Expected result:**

```json
{
  "IsLogging": true,
  "LatestDeliveryError": ""
}
```

**Result:**

- CloudTrail logging was active.

---

### Step 13: Upload Sensitive Test File

**What I did:**

- Created a local file named `sensitive-data.txt`.
- Uploaded it to the evidence bucket.

**PowerShell commands used:**

```powershell
"CONFIDENTIAL: Employee salary data - Q4 2024 report. SSN: XXX-XX-XXXX" | Out-File sensitive-data.txt
aws s3 cp sensitive-data.txt s3://<STUDENT>-incident-evidence/sensitive-data.txt
```

**Result:**

- Sensitive test file was uploaded successfully.

---

### Step 14: Create Attacker IAM User

**What I did:**

- Created a simulated attacker user named `attacker-simulation`.

**Command used:**

```bash
aws iam create-user --user-name attacker-simulation
```

**Result:**

- Attacker simulation user was created.

---

### Step 15: Attach S3 Read Policy to Attacker

**What I did:**

- Created a file named `attacker-policy.json`.
- Gave the attacker user read access to S3 for the lab.

**File created:**

```text
attacker-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3Read",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject",
                "s3:ListAllMyBuckets"
            ],
            "Resource": "*"
        }
    ]
}
```

**Apply policy command:**

```bash
aws iam put-user-policy --user-name attacker-simulation --policy-name S3ReadAccess --policy-document file://attacker-policy.json
```

**Result:**

- Attacker user received S3 read access.

---

### Step 16: Create Attacker Access Key

**What I did:**

- Created an access key for the attacker user.

**Command used:**

```bash
aws iam create-access-key --user-name attacker-simulation
```

**Expected result:**

```json
{
  "AccessKey": {
    "AccessKeyId": "REDACTED",
    "SecretAccessKey": "REDACTED",
    "Status": "Active",
    "UserName": "attacker-simulation"
  }
}
```

**Result:**

- Attacker access key was created.

**Notes:**

- The secret access key is shown only once.
- Do not upload the access key or secret key to GitHub.
- Record the access key ID temporarily for investigation and cleanup.

---

### Step 17: Switch to Attacker Credentials

**What I did:**

- Temporarily set the attacker access key credentials in PowerShell.
- Removed `AWS_PROFILE` so the AWS CLI would use the attacker access key.

**Commands used:**

```powershell
$env:AWS_ACCESS_KEY_ID="<KEY_ID>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_KEY>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- The output should show `attacker-simulation`.

**Result:**

- CLI switched to the attacker identity.

---

### Step 18: Simulate Reconnaissance

**What I did:**

- Listed all visible S3 buckets.
- Listed the contents of the evidence bucket.

**Commands used:**

```bash
aws s3 ls
```

```bash
aws s3 ls s3://<STUDENT>-incident-evidence/
```

**Expected result:**

```text
sensitive-data.txt
```

**Result:**

- Attacker user was able to discover the evidence bucket and sensitive file.

---

### Step 19: Simulate Data Exfiltration

**What I did:**

- Downloaded the sensitive file from S3 using the attacker credentials.

**Command used:**

```bash
aws s3 cp s3://<STUDENT>-incident-evidence/sensitive-data.txt stolen-data.txt
```

**Expected result:**

```text
download: s3://<STUDENT>-incident-evidence/sensitive-data.txt to ./stolen-data.txt
```

**Result:**

- Sensitive file was downloaded.

**Notes:**

- This `GetObject` action is the key data exfiltration event.
- It should appear in CloudWatch Logs Insights because S3 data event logging was enabled.

---

### Step 20: Switch Back to Admin Credentials

**What I did:**

- Cleared attacker access key environment variables.
- Restored my admin SSO profile.
- Verified admin identity.

**Commands used:**

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

**Result:**

- CLI returned to admin identity.

**Notes:**

- If the SSO token expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 21: Wait for CloudTrail Events

**What I did:**

- Waited for CloudTrail events to arrive in CloudWatch Logs.

**Expected wait time:**

```text
5–15 minutes
```

**Result:**

- Waited before beginning investigation.

---

### Step 22: Investigate with CloudTrail `lookup-events`

**What I did:**

- Used CloudTrail Event History from the CLI to check management activity by the attacker.

**Command used:**

```bash
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=attacker-simulation --query "Events[].{Time:EventTime,Name:EventName,Source:EventSource}" --output table
```

**Expected result:**

- Events such as:
  - `GetCallerIdentity`
  - `ListBuckets`

**Result:**

- Management events were visible.

**Important finding:**

- `GetObject` did **not** appear here because `lookup-events` shows management events, not S3 data events.

---

### Step 23: Investigate with CloudWatch Logs Insights

**What I did:**

- Opened CloudWatch Logs Insights.
- Selected the log group `lab5b-cloudtrail-logs`.
- Set the time range to Last 1 hour.
- Ran a query to reconstruct the attacker timeline.

**Logs Insights query:**

```sql
fields @timestamp, eventName, userIdentity.userName, userIdentity.accessKeyId,
       requestParameters.bucketName, requestParameters.key,
       sourceIPAddress, errorCode, errorMessage
| filter userIdentity.userName = "attacker-simulation"
| sort @timestamp asc
```

**Expected result:**

- Timeline showing:
  - `GetCallerIdentity`
  - `ListBuckets`
  - `ListObjects`
  - `GetObject`

**Result:**

- Full attacker timeline was visible.

---

### Step 24: Query by Access Key ID

**What I did:**

- Queried the timeline using the attacker access key ID instead of username.

**Logs Insights query:**

```sql
fields @timestamp, eventName, userIdentity.userName, userIdentity.accessKeyId,
       requestParameters.bucketName, requestParameters.key,
       sourceIPAddress
| filter userIdentity.accessKeyId = "<KEY_ID>"
| sort @timestamp asc
```

**Result:**

- Same attacker timeline was visible using the access key ID.

**Notes:**

- This is important because GuardDuty and other alerts often provide access key IDs.

---

### Step 25: Isolate the Exfiltration Event

**What I did:**

- Filtered specifically for the `GetObject` event.

**Logs Insights query:**

```sql
fields @timestamp, userIdentity.userName, userIdentity.accessKeyId,
       requestParameters.bucketName, requestParameters.key,
       sourceIPAddress, responseElements
| filter eventName = "GetObject" and userIdentity.userName = "attacker-simulation"
| sort @timestamp asc
```

**Expected result:**

- One row showing:
  - Who downloaded the file
  - Which bucket was accessed
  - Which object was downloaded
  - What time it happened
  - Source IP address

**Result:**

- Data exfiltration event was identified.

---

### Step 26: Contain the Incident

**What I did:**

- Deactivated the attacker access key.

**Command used:**

```bash
aws iam update-access-key --user-name attacker-simulation --access-key-id <KEY_ID> --status Inactive
```

**Expected result:**

```text
No output
```

**Result:**

- Attacker credentials were revoked.

---

### Step 27: Eradicate the Attacker

**What I did:**

- Deleted the attacker access key.
- Deleted the attacker inline policy.
- Deleted the attacker user.

**Commands used:**

```bash
aws iam delete-access-key --user-name attacker-simulation --access-key-id <KEY_ID>
```

```bash
aws iam delete-user-policy --user-name attacker-simulation --policy-name S3ReadAccess
```

```bash
aws iam delete-user --user-name attacker-simulation
```

**Expected result:**

```text
No output
```

**Result:**

- Attacker user and credentials were removed.

---

### Step 28: Write Incident Report

**What I did:**

- Created a file named `incident-report.md`.
- Filled in the incident report using evidence from CloudTrail and CloudWatch Logs Insights.

**Report sections included:**

- Incident Summary
- Detection
- Timeline of Events
- Affected Resources
- Key Forensic Evidence
- Containment Actions
- Eradication Actions
- Root Cause
- Lessons Learned & Recommendations

**Notes:**

- The timeline should clearly separate management events from data events.
- The `GetObject` event is the strongest evidence of data exfiltration.

---

### Step 29: Console Checkpoint

**What I did:**

- Opened CloudTrail Event History and filtered by `attacker-simulation`.
- Opened CloudTrail trail settings and verified CloudWatch Logs and data events.
- Opened CloudWatch Logs Insights and reran the timeline query.

**Result:**

- Event History showed partial activity.
- Logs Insights showed the full forensic timeline, including `GetObject`.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| Admin identity verified | Passed |
| Local project folder created | Passed |
| Evidence bucket created | Passed |
| CloudTrail logging bucket created | Passed |
| Trail bucket policy created and applied | Passed |
| CloudWatch log group created | Passed |
| CloudTrail IAM role created | Passed |
| CloudTrail Logs policy attached | Passed |
| CloudTrail trail created | Passed |
| S3 data event logging enabled | Passed |
| CloudTrail logging started | Passed |
| Sensitive file uploaded | Passed |
| Attacker user created | Passed |
| Attacker S3 read policy attached | Passed |
| Attacker access key created | Passed |
| CLI switched to attacker credentials | Passed |
| Attacker listed S3 buckets | Passed |
| Attacker listed sensitive bucket contents | Passed |
| Attacker downloaded sensitive file | Passed |
| CLI switched back to admin credentials | Passed |
| CloudTrail `lookup-events` reviewed | Passed |
| Logs Insights timeline query completed | Passed |
| `GetObject` exfiltration event found | Passed |
| Attacker key deactivated | Passed |
| Attacker key, policy, and user deleted | Passed |
| Incident report written | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `InsufficientS3BucketPolicyException` when creating trail | CloudTrail cannot write to the logging bucket | Check `trail-policy.json`, confirm bucket name/account ID, and reapply the policy |
| CloudTrail cannot deliver to CloudWatch Logs | IAM role or CloudWatch Logs policy may be wrong | Check `cloudtrail-trust-policy.json`, `cloudtrail-logs-policy.json`, account ID, and role ARN |
| `get-trail-status` shows delivery errors | CloudTrail cannot deliver to S3 or CloudWatch Logs | Review bucket policy, log group ARN, and role permissions |
| Logs Insights query returns no results | Events may not have arrived yet or the time range is wrong | Wait 5–15 minutes and set time range to Last 1 hour |
| `GetObject` does not appear in Logs Insights | Data events were not enabled, trail was not active, or attack happened before logging | Verify event selectors, start logging, rerun attack steps, and wait |
| `lookup-events` does not show `GetObject` | This is expected because `lookup-events` shows management events only | Use CloudWatch Logs Insights for data events |
| CLI still shows attacker after switching back | Environment variables were not cleared | Remove `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION`, then reset `AWS_PROFILE` |
| SSO token expired after switching back | Admin profile uses IAM Identity Center and the cached token expired | Run `aws sso login --profile <YOUR_PROFILE_NAME>` |
| Cannot delete attacker user | User still has an access key or inline policy | Delete access key, delete user policy, then delete user |

## Cleanup

### Step 1: Stop and Delete CloudTrail Trail

**Commands used:**

```bash
aws cloudtrail stop-logging --name lab5b-trail
```

```bash
aws cloudtrail delete-trail --name lab5b-trail
```

**Result:**

- CloudTrail trail was stopped and deleted.

---

### Step 2: Delete CloudWatch Log Group

**Command used:**

```bash
aws logs delete-log-group --log-group-name lab5b-cloudtrail-logs --region us-east-1
```

**Result:**

- CloudWatch log group was deleted.

---

### Step 3: Delete CloudTrail IAM Role

**Commands used:**

```bash
aws iam delete-role-policy --role-name lab5b-cloudtrail-role --policy-name CloudWatchLogsWrite
```

```bash
aws iam delete-role --role-name lab5b-cloudtrail-role
```

**Result:**

- CloudTrail IAM role and inline policy were deleted.

---

### Step 4: Delete CloudTrail Log Bucket

**Commands used:**

```bash
aws s3 rm s3://<STUDENT>-cloudtrail-logs --recursive
```

```bash
aws s3 rb s3://<STUDENT>-cloudtrail-logs
```

**Result:**

- CloudTrail log bucket was emptied and deleted.

---

### Step 5: Delete Evidence Bucket

**Commands used:**

```bash
aws s3 rm s3://<STUDENT>-incident-evidence --recursive
```

```bash
aws s3 rb s3://<STUDENT>-incident-evidence
```

**Result:**

- Evidence bucket was emptied and deleted.

---

### Step 6: Verify Attacker User Is Deleted

**Command used:**

```bash
aws iam get-user --user-name attacker-simulation
```

**Expected result:**

```text
NoSuchEntity
```

**Result:**

- Attacker user was confirmed deleted.

---

### Step 7: Delete Local Lab Folder

**PowerShell command used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-5b
```

**Result:**

- Local project folder was deleted.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| CloudTrail trail | Stopped and deleted | Yes |
| CloudWatch log group | Deleted | Yes |
| CloudTrail IAM role policy | Deleted | Yes |
| CloudTrail IAM role | Deleted | Yes |
| CloudTrail log bucket objects | Deleted | Yes |
| CloudTrail log bucket | Deleted | Yes |
| Evidence bucket objects | Deleted | Yes |
| Evidence bucket | Deleted | Yes |
| Attacker access key | Deleted | Yes |
| Attacker inline policy | Deleted | Yes |
| Attacker IAM user | Deleted | Yes |
| Local lab folder | Deleted | Yes |

## Incident Report Notes

The incident report should include:

| Section | What to Include |
|---|---|
| Incident Summary | Date, severity, affected identity, and status |
| Detection | How the incident was discovered |
| Timeline | Chronological events from CloudTrail and Logs Insights |
| Affected Resources | S3 bucket, sensitive object, attacker identity |
| Key Evidence | `GetObject` event showing data exfiltration |
| Containment | Access key deactivated |
| Eradication | Access key, policy, and user deleted |
| Root Cause | Simulated leaked/compromised access key |
| Recommendations | GuardDuty, secret scanning, IAM roles, least privilege, CloudTrail data events |

## What I Learned

- CloudTrail Event History is useful, but it does not show everything.
- `aws cloudtrail lookup-events` is useful for management events.
- S3 object downloads require CloudTrail data event logging.
- CloudWatch Logs Insights can reconstruct a full forensic timeline.
- Querying by access key ID is useful because security alerts often include key IDs.
- The `GetObject` event is the key evidence of S3 data exfiltration.
- Logging must be configured before an incident occurs.
- If logging is not enabled before the attack, some evidence may not exist.
- Incident reports should clearly separate what happened, how it was proven, what was done, and what should improve.
- This lab follows the NIST incident response lifecycle: preparation, detection and analysis, containment, eradication, recovery, and lessons learned.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/project-folder-created.png` | Local project folder created |
| `screenshots/evidence-bucket-created.png` | S3 evidence bucket created |
| `screenshots/cloudtrail-log-bucket-created.png` | CloudTrail logging bucket created |
| `screenshots/trail-policy-created-1.png` | CloudTrail bucket policy file created |
| `screenshots/trail-policy-created-2.png` | CloudTrail bucket policy file created |
| `screenshots/cloudwatch-log-group-created.png` | CloudWatch log group created |
| `screenshots/cloudtrail-role-created.png` | IAM role for CloudTrail created |
| `screenshots/cloudtrail-trail-created.png` | CloudTrail trail created |
| `screenshots/data-event-logging-enabled.png` | S3 data event logging enabled |
| `screenshots/cloudtrail-logging-active.png` | CloudTrail logging verified |
| `screenshots/sensitive-file-uploaded.png` | Sensitive test file uploaded |
| `screenshots/attacker-user-created.png` | Attacker IAM user created |
| `screenshots/attacker-policy-attached.png` | Attacker S3 read policy attached |
| `screenshots/attacker-access-key-created-redacted.png` | Attacker access key created with sensitive values redacted |
| `screenshots/attacker-identity-verified.png` | CLI switched to attacker identity |
| `screenshots/attacker-s3-recon.png` | Attacker listed buckets and evidence bucket contents |
| `screenshots/data-exfiltration-download.png` | Sensitive file downloaded by attacker |
| `screenshots/admin-profile-restored.png` | CLI switched back to admin |
| `screenshots/cloudtrail-lookup-events.png` | Management events reviewed with `lookup-events` |
| `screenshots/logs-insights-full-timeline.png` | Full attacker timeline from Logs Insights |
| `screenshots/logs-insights-access-key-query.png` | Timeline queried by access key ID |
| `screenshots/logs-insights-getobject-event.png` | `GetObject` exfiltration event isolated |
| `screenshots/key-deactivated.png` | Attacker key deactivated |
| `screenshots/attacker-deleted.png` | Attacker user and credentials removed |
| `screenshots/incident-report-created.png` | Incident report created |
| `screenshots/cleanup-verified-1.png` | Lab cleanup verified |
| `screenshots/cleanup-verified-2.png` | Lab cleanup verified |