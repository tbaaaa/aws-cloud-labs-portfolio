# Lab 16: Automated Remediation

## Lab Summary

In this lab, I built an automated incident response pipeline that deactivates newly created IAM access keys.

The pipeline uses CloudTrail to record the `CreateAccessKey` API call, EventBridge to detect the event, and Lambda to automatically deactivate the new access key. This simulates an organization enforcing a policy where long-lived IAM access keys are not allowed.

This lab is important because it introduces automated remediation and SOAR: Security Orchestration, Automation, and Response.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 5C — Build Automated Incident Response with Lambda
* Session: 5 — Incident Response on AWS
* My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

* Set the active AWS CLI profile
* Verify AWS CLI authentication
* Create a local project folder
* Check for an existing CloudTrail trail
* Create an S3 bucket for CloudTrail logs
* Create and apply a CloudTrail bucket policy
* Create and start a CloudTrail trail
* Create an IAM role for Lambda
* Attach IAM permissions to the Lambda role
* Write a Python Lambda function that deactivates IAM access keys
* Package and deploy the Lambda function
* Manually test the Lambda function with a simulated event
* Verify that the test access key becomes inactive
* Create an EventBridge rule for `CreateAccessKey`
* Add the Lambda function as the EventBridge target
* Grant EventBridge permission to invoke Lambda
* Create a new access key to trigger the full pipeline
* Verify that the key is automatically deactivated
* Review Lambda logs
* Clean up all lab resources

## Services / Tools Used

| Service / Tool         | Purpose                                                      |
| ---------------------- | ------------------------------------------------------------ |
| AWS CloudTrail         | Records AWS API activity and sends API events to EventBridge |
| Amazon S3              | Stores CloudTrail log files                                  |
| Amazon EventBridge     | Detects `CreateAccessKey` events and triggers Lambda         |
| AWS Lambda             | Runs remediation code to deactivate access keys              |
| AWS IAM                | Creates users, roles, policies, and access keys              |
| Amazon CloudWatch Logs | Stores Lambda execution logs                                 |
| AWS CLI                | Creates, tests, verifies, and deletes lab resources          |
| PowerShell             | Terminal used to run AWS CLI commands                        |
| Python                 | Lambda function runtime                                      |
| JSON                   | Used for policies, test events, and EventBridge patterns     |

## Prerequisites

* Completed Lab 01: AWS Account & CLI Setup
* Completed Lab 5A: Credential Revocation
* AWS CLI installed and configured
* AWS SSO profile working
* AWS CLI authentication verified
* Text editor such as VS Code or Notepad
* Basic understanding of IAM access keys, Lambda, EventBridge, and CloudTrail

## Cost Notice

Estimated cost: `$0.00`

Notes:

* IAM is free.
* Lambda has a generous free tier.
* EventBridge is free for AWS service events in this lab.
* CloudTrail management events for the first trail are generally free.
* S3 cost should remain near `$0.00` because the log volume is tiny.
* Cleanup is required so resources do not remain active unnecessarily.

## Key Concepts

| Concept                 | Meaning                                                                                          |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| SOAR                    | Security Orchestration, Automation, and Response.                                                |
| Automated Remediation   | Automatically fixing or containing a security issue without manual action.                       |
| CloudTrail Trail        | A CloudTrail configuration that records API activity and allows events to flow into EventBridge. |
| EventBridge Rule        | A rule that watches for matching AWS events and sends them to a target.                          |
| Lambda Function         | Serverless code that runs when triggered.                                                        |
| `CreateAccessKey`       | IAM API call that creates a new long-lived access key.                                           |
| Access Key Deactivation | Changing an access key status from `Active` to `Inactive`.                                       |
| Least Privilege         | Giving only the permissions needed to perform a task.                                            |
| CloudWatch Logs         | Logs created by Lambda showing what the function did.                                            |

## Security Notes

| Topic                                             | Explanation                                                                                |
| ------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Long-lived access keys are risky                  | Access keys can be leaked, copied, or committed to public repositories.                    |
| Automated response is faster than manual response | Lambda can deactivate a key within minutes of creation.                                    |
| CloudTrail trail is required                      | CloudTrail Event History alone does not feed EventBridge. An active trail is required.     |
| EventBridge delay                                 | CloudTrail to EventBridge can take a few minutes.                                          |
| Lab permission is broad                           | This lab uses `IAMFullAccess` for simplicity. In production, use a least-privilege policy. |
| Lambda logs help with troubleshooting             | CloudWatch Logs show whether the function ran and what key was deactivated.                |
| Cleanup order matters                             | Remove EventBridge first so Lambda does not interfere with cleanup actions.                |

## Architecture Overview

```text
Someone creates an IAM access key
        |
        v
CloudTrail records CreateAccessKey
        |
        v
EventBridge matches the API event
        |
        v
Lambda function is triggered
        |
        v
Lambda deactivates the new access key
```

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

* Set my AWS CLI profile for the current PowerShell session.
* Verified my current admin identity.

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

* AWS CLI returned my active admin identity successfully.

**Notes:**

* If the SSO token is expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

* Created a local folder for this lab.
* Changed into the folder before creating files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-5c
cd ~\Desktop\workshop-lab-5c
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-5c
```

**Result:**

* PowerShell showed a path ending in `workshop-lab-5c`.

---

### Step 3: Check for Existing CloudTrail Trails

**What I did:**

* Checked whether a CloudTrail trail already existed.

**Command used:**

```bash
aws cloudtrail describe-trails --region us-east-1
```

**Expected result if no trail exists:**

```json
{
  "trailList": []
}
```

**Result:**

* Existing CloudTrail trails were checked.

**Notes:**

* CloudTrail Event History is not enough for this lab.
* EventBridge needs an active CloudTrail trail to receive AWS API call events.

---

### Step 4: Create S3 Bucket for CloudTrail Logs

**What I did:**

* Created an S3 bucket to store CloudTrail logs.

**Command used:**

```bash
aws s3api create-bucket --bucket <STUDENT>-workshop-cloudtrail --region us-east-1
```

**Expected result:**

```json
{
  "Location": "/<STUDENT>-workshop-cloudtrail"
}
```

**Result:**

* CloudTrail log bucket was created successfully.

**Notes:**

* S3 bucket names must be globally unique.
* Do not include angle brackets when running the command.

---

### Step 5: Create CloudTrail Bucket Policy

**What I did:**

* Created a file named `bucket-policy.json`.
* Added permissions allowing CloudTrail to write logs to the S3 bucket.

**File created:**

```text
bucket-policy.json
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
            "Resource": "arn:aws:s3:::<STUDENT>-workshop-cloudtrail"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<STUDENT>-workshop-cloudtrail/AWSLogs/<ACCOUNT_ID>/*",
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
aws s3api put-bucket-policy --bucket <STUDENT>-workshop-cloudtrail --policy file://bucket-policy.json
```

**Expected result:**

```text
No output
```

**Result:**

* CloudTrail bucket policy was applied successfully.

---

### Step 6: Create CloudTrail Trail

**What I did:**

* Created a multi-region CloudTrail trail named `workshop-trail`.

**Command used:**

```bash
aws cloudtrail create-trail --name workshop-trail --s3-bucket-name <STUDENT>-workshop-cloudtrail --is-multi-region-trail
```

**Expected result:**

* JSON output showing the trail details.

**Result:**

* CloudTrail trail was created successfully.

---

### Step 7: Start CloudTrail Logging

**What I did:**

* Started CloudTrail logging.
* Verified logging was active.

**Commands used:**

```bash
aws cloudtrail start-logging --name workshop-trail
```

```bash
aws cloudtrail get-trail-status --name workshop-trail --region us-east-1
```

**Expected result:**

```json
{
  "IsLogging": true
}
```

**Result:**

* CloudTrail logging was active.

**Notes:**

* This is required so IAM API calls can flow into EventBridge.

---

### Step 8: Create Lambda Trust Policy

**What I did:**

* Created a file named `trust-policy.json`.
* Allowed Lambda to assume the IAM role.

**File created:**

```text
trust-policy.json
```

**File content:**

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

---

### Step 9: Create Lambda IAM Role

**What I did:**

* Created an IAM role named `workshop-key-revoker-role`.

**Command used:**

```bash
aws iam create-role --role-name workshop-key-revoker-role --assume-role-policy-document file://trust-policy.json
```

**Expected result:**

* JSON output showing role details and role ARN.

**Result:**

* Lambda IAM role was created successfully.

---

### Step 10: Attach Permissions to Lambda Role

**What I did:**

* Attached IAM permissions so Lambda can deactivate access keys.
* Attached Lambda basic execution permissions so Lambda can write logs to CloudWatch.

**Commands used:**

```bash
aws iam attach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/IAMFullAccess
```

```bash
aws iam attach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**Expected result:**

```text
No output
```

**Result:**

* Policies were attached successfully.

**Notes:**

* `IAMFullAccess` is used for lab simplicity.
* A production version should use a custom least-privilege policy allowing only the required IAM actions.

---

### Step 11: Create Lambda Function Code

**What I did:**

* Created a Python file named `revoke_key.py`.
* Added logic to extract the username and access key ID from a CloudTrail event.
* Added logic to deactivate the access key.

**File created:**

```text
revoke_key.py
```

**File content:**

```python
import json
import boto3

iam_client = boto3.client('iam')

def lambda_handler(event, context):
    detail = event.get('detail', {})
    request_params = detail.get('requestParameters', {})
    response_elements = detail.get('responseElements', {})

    username = request_params.get('userName', 'unknown')
    access_key_info = response_elements.get('accessKey', {})
    access_key_id = access_key_info.get('accessKeyId', '')

    if not access_key_id:
        print(f"No access key ID found in event for user {username}")
        return {'statusCode': 400, 'body': 'No access key ID found'}

    try:
        iam_client.update_access_key(
            UserName=username,
            AccessKeyId=access_key_id,
            Status='Inactive'
        )

        message = f"SECURITY ALERT: Deactivated access key {access_key_id} for user {username}"
        print(message)

        return {'statusCode': 200, 'body': message}

    except Exception as e:
        error_msg = f"Failed to deactivate key {access_key_id} for user {username}: {str(e)}"
        print(error_msg)

        return {'statusCode': 500, 'body': error_msg}
```

**Result:**

* Lambda function code was created locally.

---

### Step 12: Package Lambda Function

**What I did:**

* Compressed the Python file into a deployment zip file.

**PowerShell command used:**

```powershell
Compress-Archive -Path revoke_key.py -DestinationPath revoke_key.zip -Force
```

**Expected result:**

```text
revoke_key.zip
```

**Result:**

* Lambda deployment package was created.

---

### Step 13: Deploy Lambda Function

**What I did:**

* Created a Lambda function named `workshop-key-revoker`.

**Command used:**

```bash
aws lambda create-function --function-name workshop-key-revoker --runtime python3.12 --role arn:aws:iam::<ACCOUNT_ID>:role/workshop-key-revoker-role --handler revoke_key.lambda_handler --zip-file fileb://revoke_key.zip --region us-east-1 --timeout 30
```

**Expected result:**

* JSON output showing the Lambda function details and Function ARN.

**Result:**

* Lambda function was deployed successfully.

**Notes:**

* If the command fails because the role cannot be assumed yet, wait 10–30 seconds and try again.
* IAM role propagation can take a short time.

---

### Step 14: Create Test IAM User

**What I did:**

* Created a test IAM user named `auto-revoke-test`.

**Command used:**

```bash
aws iam create-user --user-name auto-revoke-test
```

**Expected result:**

* JSON output showing the created user.

**Result:**

* Test user was created successfully.

---

### Step 15: Create Test Access Key

**What I did:**

* Created an access key for the test user.

**Command used:**

```bash
aws iam create-access-key --user-name auto-revoke-test
```

**Expected result:**

```json
{
  "AccessKey": {
    "UserName": "auto-revoke-test",
    "AccessKeyId": "REDACTED",
    "SecretAccessKey": "REDACTED",
    "Status": "Active"
  }
}
```

**Result:**

* Access key was created successfully.

**Notes:**

* Record the Access Key ID temporarily.
* Do not commit access keys or secret keys to GitHub.

---

### Step 16: Verify Test Key Is Active

**What I did:**

* Checked the test user's access key status.

**Command used:**

```bash
aws iam list-access-keys --user-name auto-revoke-test --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**Expected result:**

```text
Id        Status
--------  ------
AKIA...   Active
```

**Result:**

* Test access key was active.

---

### Step 17: Create Manual Lambda Test Event

**What I did:**

* Created a file named `test-event.json`.
* Added a simulated CloudTrail `CreateAccessKey` event.

**File created:**

```text
test-event.json
```

**File content:**

```json
{
    "detail": {
        "eventName": "CreateAccessKey",
        "requestParameters": {
            "userName": "auto-revoke-test"
        },
        "responseElements": {
            "accessKey": {
                "accessKeyId": "<ACCESS_KEY_ID>",
                "status": "Active",
                "userName": "auto-revoke-test"
            }
        }
    }
}
```

**Result:**

* Test event file was created.

**Notes:**

* Replace `<ACCESS_KEY_ID>` with the access key ID created in Step 15.

---

### Step 18: Manually Invoke Lambda

**What I did:**

* Invoked the Lambda function manually using the test event.

**Command used:**

```bash
aws lambda invoke --function-name workshop-key-revoker --payload file://test-event.json --cli-binary-format raw-in-base64-out response.json --region us-east-1
```

**Expected result:**

```json
{
  "StatusCode": 200
}
```

**Result:**

* Lambda function returned status code `200`.

---

### Step 19: Check Lambda Response

**What I did:**

* Opened the Lambda response file.

**PowerShell command used:**

```powershell
Get-Content response.json
```

**Expected result:**

```json
{
  "statusCode": 200,
  "body": "SECURITY ALERT: Deactivated access key AKIA... for user auto-revoke-test"
}
```

**Result:**

* Lambda response confirmed that the key was deactivated.

---

### Step 20: Verify Test Key Is Inactive

**What I did:**

* Checked the key status again.

**Command used:**

```bash
aws iam list-access-keys --user-name auto-revoke-test --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**Expected result:**

```text
Id        Status
--------  --------
AKIA...   Inactive
```

**Result:**

* The test access key was inactive.

---

### Step 21: Delete Manual Test Key

**What I did:**

* Deleted the inactive test key before wiring up EventBridge.

**Command used:**

```bash
aws iam delete-access-key --user-name auto-revoke-test --access-key-id <ACCESS_KEY_ID>
```

**Expected result:**

```text
No output
```

**Result:**

* Manual test key was deleted.

**Notes:**

* This prevents confusion when testing the automated pipeline.

---

### Step 22: Create EventBridge Event Pattern

**What I did:**

* Created a file named `eventbridge-pattern.json`.
* Added a pattern that matches IAM `CreateAccessKey` API calls.

**File created:**

```text
eventbridge-pattern.json
```

**File content:**

```json
{
    "source": ["aws.iam"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
        "eventSource": ["iam.amazonaws.com"],
        "eventName": ["CreateAccessKey"]
    }
}
```

**Result:**

* EventBridge pattern file was created.

---

### Step 23: Create EventBridge Rule

**What I did:**

* Created an EventBridge rule named `workshop-auto-revoke`.

**Command used:**

```bash
aws events put-rule --name workshop-auto-revoke --event-pattern file://eventbridge-pattern.json --state ENABLED --region us-east-1
```

**Expected result:**

* JSON output showing the Rule ARN.

**Result:**

* EventBridge rule was created successfully.

---

### Step 24: Add Lambda as EventBridge Target

**What I did:**

* Added the Lambda function as the EventBridge rule target.

**Command used:**

```bash
aws events put-targets --rule workshop-auto-revoke --targets "Id=lambda-target,Arn=<LAMBDA_FUNCTION_ARN>" --region us-east-1
```

**Expected result:**

```json
{
  "FailedEntryCount": 0,
  "FailedEntries": []
}
```

**Result:**

* Lambda function was added as the EventBridge target.

---

### Step 25: Grant EventBridge Permission to Invoke Lambda

**What I did:**

* Added Lambda permission allowing EventBridge to invoke the function.

**Command used:**

```bash
aws lambda add-permission --function-name workshop-key-revoker --statement-id eventbridge-invoke --action lambda:InvokeFunction --principal events.amazonaws.com --source-arn <EVENTBRIDGE_RULE_ARN> --region us-east-1
```

**Expected result:**

* JSON output containing a `Statement`.

**Result:**

* EventBridge was granted permission to invoke Lambda.

---

### Step 26: Trigger the Automated Pipeline

**What I did:**

* Created a new access key for `auto-revoke-test`.
* Waited for CloudTrail and EventBridge to process the event.

**Command used:**

```bash
aws iam create-access-key --user-name auto-revoke-test
```

**Expected result:**

* A new active access key is created initially.
* After 1–5 minutes, Lambda should deactivate it.

**Result:**

* New key was created and the pipeline was triggered.

---

### Step 27: Verify Automatic Deactivation

**What I did:**

* Checked whether the newly created access key was automatically deactivated.

**Command used:**

```bash
aws iam list-access-keys --user-name auto-revoke-test --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**Expected result:**

```text
Id        Status
--------  --------
AKIA...   Inactive
```

**Result:**

* The key was automatically deactivated by Lambda.

---

### Step 28: Review Lambda Logs

**What I did:**

* Checked Lambda logs in CloudWatch.

**Command used:**

```bash
aws logs tail /aws/lambda/workshop-key-revoker --region us-east-1 --since 10m
```

**Expected result:**

```text
SECURITY ALERT: Deactivated access key AKIA... for user auto-revoke-test
```

**Result:**

* Lambda logs confirmed that the function ran.

---

### Step 29: Console Checkpoint

**What I did:**

* Opened the AWS Console.
* Checked the Lambda function.
* Checked EventBridge rule configuration.
* Checked CloudTrail trail status.
* Checked the test IAM user's access key status.

**Expected result:**

* Lambda function exists.
* EventBridge rule is enabled.
* Lambda is configured as the target.
* CloudTrail trail is active.
* Test user's new access key is inactive.

**Result:**

* Automated remediation pipeline was verified.

## Validation / Checkpoints

| Checkpoint                                      | Result |
| ----------------------------------------------- | ------ |
| AWS CLI profile set                             | Passed |
| Admin identity verified                         | Passed |
| Local project folder created                    | Passed |
| Existing CloudTrail trails checked              | Passed |
| CloudTrail S3 bucket created                    | Passed |
| CloudTrail bucket policy created and applied    | Passed |
| CloudTrail trail created                        | Passed |
| CloudTrail logging started                      | Passed |
| Lambda trust policy created                     | Passed |
| Lambda IAM role created                         | Passed |
| IAM permissions attached to Lambda role         | Passed |
| Lambda code written                             | Passed |
| Lambda zip package created                      | Passed |
| Lambda function deployed                        | Passed |
| Test IAM user created                           | Passed |
| Test access key created                         | Passed |
| Manual Lambda test event created                | Passed |
| Lambda manually invoked                         | Passed |
| Test key deactivated manually by Lambda         | Passed |
| Manual test key deleted                         | Passed |
| EventBridge event pattern created               | Passed |
| EventBridge rule created                        | Passed |
| Lambda added as EventBridge target              | Passed |
| EventBridge granted permission to invoke Lambda | Passed |
| New access key created to trigger pipeline      | Passed |
| New key automatically deactivated               | Passed |
| Lambda logs reviewed                            | Passed |
| Console verification completed                  | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                          | What It Means                                        | How to Fix                                                                                        |
| ---------------------------------------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Lambda never triggers                          | No active CloudTrail trail exists                    | Create and start `workshop-trail`                                                                 |
| CloudTrail status shows `IsLogging: false`     | Trail exists but logging is stopped                  | Run `aws cloudtrail start-logging --name workshop-trail`                                          |
| Lambda manual invoke returns `statusCode: 400` | Test event is missing the access key ID              | Check `test-event.json` and replace `<ACCESS_KEY_ID>`                                             |
| Lambda manual invoke returns `statusCode: 500` | Lambda role likely lacks IAM permissions             | Confirm required policies are attached to `workshop-key-revoker-role`                             |
| `put-targets` returns `FailedEntryCount: 1`    | Lambda ARN is wrong or malformed                     | Recheck the Lambda Function ARN                                                                   |
| `add-permission` says statement already exists | Permission was already added                         | Continue if the permission already exists                                                         |
| Key stays active after 10 minutes              | Pipeline is not fully connected                      | Check CloudTrail logging, EventBridge rule, Lambda target, Lambda permission, and CloudWatch logs |
| No Lambda logs appear                          | Lambda was not triggered or lacks logging permission | Check EventBridge target and confirm `AWSLambdaBasicExecutionRole` is attached                    |
| SSO token expired                              | Admin SSO session expired                            | Run `aws sso login --profile <YOUR_PROFILE_NAME>`                                                 |
| Cannot delete test user                        | User still has access keys attached                  | List and delete all access keys first                                                             |

## Cleanup

### Step 1: Remove EventBridge Target

**What I did:**

* Removed Lambda as the EventBridge rule target.

**Command used:**

```bash
aws events remove-targets --rule workshop-auto-revoke --ids lambda-target --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

* EventBridge target was removed.

---

### Step 2: Delete EventBridge Rule

**What I did:**

* Deleted the EventBridge rule.

**Command used:**

```bash
aws events delete-rule --name workshop-auto-revoke --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

* EventBridge rule was deleted.

---

### Step 3: Delete Lambda Function

**What I did:**

* Deleted the Lambda function.

**Command used:**

```bash
aws lambda delete-function --function-name workshop-key-revoker --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

* Lambda function was deleted.

---

### Step 4: Detach Policies and Delete Lambda IAM Role

**What I did:**

* Detached the managed policies from the Lambda role.
* Deleted the role.

**Commands used:**

```bash
aws iam detach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/IAMFullAccess
```

```bash
aws iam detach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```bash
aws iam delete-role --role-name workshop-key-revoker-role
```

**Expected result:**

```text
No output
```

**Result:**

* Lambda IAM role and attached policies were removed.

---

### Step 5: Delete Test User and Remaining Access Keys

**What I did:**

* Listed remaining access keys for the test user.
* Deleted any remaining keys.
* Deleted the test IAM user.

**Commands used:**

```bash
aws iam list-access-keys --user-name auto-revoke-test
```

```bash
aws iam delete-access-key --user-name auto-revoke-test --access-key-id <ACCESS_KEY_ID>
```

```bash
aws iam delete-user --user-name auto-revoke-test
```

**Expected result:**

```text
No output
```

**Result:**

* Test IAM user and access keys were deleted.

---

### Step 6: Stop and Delete CloudTrail Trail

**What I did:**

* Stopped CloudTrail logging.
* Deleted the CloudTrail trail.

**Commands used:**

```bash
aws cloudtrail stop-logging --name workshop-trail
```

```bash
aws cloudtrail delete-trail --name workshop-trail
```

**Expected result:**

```text
No output
```

**Result:**

* CloudTrail trail was stopped and deleted.

---

### Step 7: Delete CloudTrail S3 Bucket

**What I did:**

* Emptied the CloudTrail S3 bucket.
* Deleted the bucket.

**Commands used:**

```bash
aws s3 rm s3://<STUDENT>-workshop-cloudtrail --recursive
```

```bash
aws s3api delete-bucket --bucket <STUDENT>-workshop-cloudtrail --region us-east-1
```

**Result:**

* CloudTrail log bucket was emptied and deleted.

---

### Step 8: Delete Local Lab Folder

**PowerShell command used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-5c
```

**Result:**

* Local project folder was deleted.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| EventBridge target | Removed | Yes |
| EventBridge rule | Deleted | Yes |
| Lambda function | Deleted | Yes |
| Lambda IAM role policies | Detached | Yes |
| Lambda IAM role | Deleted | Yes |
| Test user access keys | Deleted | Yes |
| Test IAM user | Deleted | Yes |
| CloudTrail trail | Stopped and deleted | Yes |
| CloudTrail S3 bucket objects | Deleted | Yes |
| CloudTrail S3 bucket | Deleted | Yes |
| Local lab folder | Deleted | Yes |

## What I Learned

* Automated remediation allows security teams to respond faster than manual processes.
* CloudTrail Event History alone does not send events to EventBridge.
* An active CloudTrail trail is required for EventBridge to receive AWS API call events.
* EventBridge can match specific API calls such as `CreateAccessKey`.
* Lambda can perform containment actions such as deactivating access keys.
* CloudWatch Logs are useful for confirming that Lambda executed successfully.
* This lab demonstrated a basic SOAR workflow.
* In production, the Lambda role should use least privilege instead of broad IAM permissions.
* Cleanup order matters because the EventBridge rule can keep triggering the Lambda function.

## Screenshots

| Screenshot                                      | Description                                                 |
| ----------------------------------------------- | ----------------------------------------------------------- |
| `screenshots/project-folder-created.png`        | Local lab folder created                                    |
| `screenshots/cloudtrail-trails-checked.png`     | Existing CloudTrail trails checked                          |
| `screenshots/cloudtrail-bucket-created.png`     | CloudTrail S3 bucket created                                |
| `screenshots/bucket-policy-created.png`         | CloudTrail bucket policy file created                       |
| `screenshots/bucket-policy-applied.png`         | CloudTrail bucket policy applied                            |
| `screenshots/cloudtrail-trail-created.png`      | CloudTrail trail created                                    |
| `screenshots/cloudtrail-logging-active.png`     | CloudTrail logging verified                                 |
| `screenshots/lambda-role-created.png`           | Lambda IAM role created                                     |
| `screenshots/lambda-policies-attached.png`      | Lambda permissions attached                                 |
| `screenshots/lambda-code-created.png`           | Lambda Python file created                                  |
| `screenshots/lambda-function-created.png`       | Lambda function deployed                                    |
| `screenshots/test-user-created.png`             | Test IAM user created                                       |
| `screenshots/test-access-key-active.png`        | Test access key initially active                            |
| `screenshots/manual-lambda-invoke.png`          | Lambda manually invoked                                     |
| `screenshots/test-access-key-inactive.png`      | Test access key deactivated                                 |
| `screenshots/eventbridge-rule-created.png`      | EventBridge rule created                                    |
| `screenshots/lambda-target-added.png`           | Lambda added as EventBridge target                          |
| `screenshots/eventbridge-lambda-permission.png` | EventBridge granted permission to invoke Lambda             |
| `screenshots/auto-revoke-triggered.png`         | New access key created to trigger pipeline                  |
| `screenshots/auto-revoke-key-inactive.png`      | New key automatically deactivated                           |
| `screenshots/lambda-logs-security-alert.png`    | Lambda logs confirmed key deactivation                      |
| `screenshots/console-verification-1.png`          | Console verification of Lambda |
| `screenshots/console-verification-2.png`          | Console verification of CloudTrail |
| `screenshots/console-verification-3.png`          | Console verification of EventBridge |
| `screenshots/cleanup-verified-1.png`              | Lab cleanup verified                                        |
| `screenshots/cleanup-verified-2.png`              | Lab cleanup verified                                        |
