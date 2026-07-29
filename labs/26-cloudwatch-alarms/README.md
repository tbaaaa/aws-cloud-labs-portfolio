# Lab 26: CloudWatch Alarms

## Lab Summary

In this lab, I deployed a Lambda function and configured monitoring around it using Amazon CloudWatch and Amazon SNS.

The lab started with a healthy Lambda function that returned a successful response and wrote structured logs. I invoked the function several times to create a normal baseline, then reviewed the Lambda `Invocations` and `Errors` metrics in CloudWatch.

After confirming the function was healthy, I created an SNS topic, subscribed my email address, and created a CloudWatch alarm that watches the Lambda `Errors` metric. I then deployed a broken version of the Lambda function and invoked it several times to trigger errors. The CloudWatch alarm moved into the `ALARM` state and sent an email notification through SNS.

This lab demonstrated how monitoring turns silent failures into visible alerts.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 9A — Deploy, Monitor, and Alert — Your First CloudWatch Alarm
- Session: 9 — Monitoring & Observability
- Track: Solutions Architecture
- Difficulty: Beginner
- Target certification: AWS Solutions Architect – Associate

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local lab project folder
- Create healthy Lambda function code
- Create an IAM trust policy for Lambda
- Create a Lambda execution role
- Attach the AWS managed Lambda logging policy
- Package the Lambda code into a ZIP file
- Deploy the Lambda function
- Invoke the healthy Lambda function
- Generate baseline invocation metrics
- Review Lambda `Invocations` metrics in CloudWatch
- Review Lambda `Errors` metrics in CloudWatch
- Create an SNS topic for alert notifications
- Subscribe an email address to the SNS topic
- Confirm the SNS email subscription
- Create a CloudWatch alarm on Lambda errors
- Check the alarm state
- Deploy a broken Lambda function version
- Invoke the broken function to generate errors
- Confirm the alarm changes to `ALARM`
- Confirm the SNS email alert is received
- Clean up the alarm, SNS topic, Lambda function, log group, IAM role, and local project folder

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Lambda | Runs the workshop API function |
| AWS IAM | Provides the Lambda execution role |
| Amazon CloudWatch Metrics | Tracks Lambda invocations, errors, duration, and other metrics |
| Amazon CloudWatch Alarms | Watches the Lambda `Errors` metric and triggers an alert |
| Amazon CloudWatch Logs | Stores Lambda log output |
| Amazon SNS | Sends email alert notifications |
| AWS CLI | Creates, invokes, monitors, and deletes AWS resources |
| PowerShell | Runs AWS CLI commands and packages Lambda code |
| VS Code | Creates and edits Lambda and policy files |

## Prerequisites

- Completed Lab 1A: AWS Account & CLI Setup
- AWS CLI installed
- AWS CLI authenticated with a working profile
- An email address available for SNS confirmation
- VS Code or another text editor installed
- PowerShell available on Windows

## Cost Notice

Estimated cost: `$0.00`

| Service | Cost Consideration |
|---|---|
| AWS Lambda | Expected to remain within the Lambda Always Free usage for this lab |
| Amazon CloudWatch | Basic Lambda metrics, small logs, and one alarm are expected to remain within free-tier usage |
| Amazon SNS | Email notification volume is very small for this lab |
| AWS IAM | Free |

## Key Concepts

| Concept | Meaning |
|---|---|
| Monitoring | Collecting metrics, logs, and signals to understand whether a system is working |
| Observability | The ability to understand system behavior using outputs like metrics, logs, and traces |
| CloudWatch Metric | A time-based measurement collected by AWS services |
| Lambda Invocations | The number of times a Lambda function was called |
| Lambda Errors | The number of Lambda invocations that failed |
| CloudWatch Alarm | A rule that watches a metric and changes state when a threshold is crossed |
| `OK` State | Alarm state meaning the metric is within the expected threshold |
| `ALARM` State | Alarm state meaning the metric crossed the threshold |
| `INSUFFICIENT_DATA` State | Alarm state meaning CloudWatch does not have enough data yet |
| SNS Topic | A notification channel that can deliver messages to subscribers |
| SNS Subscription | An endpoint, such as an email address, that receives messages from a topic |
| Structured Logging | Logging data in a consistent format, such as JSON, to make it easier to search and analyze |
| Healthy Baseline | Normal system behavior used as a reference before testing failures |
| Mean Time to Detect | How long it takes to notice that something is wrong |

## Security Notes

| Topic | Explanation |
|---|---|
| Least privilege | The Lambda role only needs permission to write logs to CloudWatch Logs |
| Email confirmation | SNS requires email subscription confirmation before it sends alerts |
| No secrets in code | The Lambda code does not contain access keys, credentials, or sensitive configuration |
| Monitoring is security-relevant | Alerts help detect failures, suspicious behavior, and operational issues quickly |
| Cleanup matters | Unused alarms, log groups, and functions should be removed after the lab |
| Structured logs help investigation | JSON logs make later troubleshooting and log queries easier |

## Architecture Overview

```text
User / AWS CLI
      |
      | invokes
      v
Lambda Function: workshop-api-lab9
      |
      +--------------------+
      |                    |
      v                    v
CloudWatch Logs      CloudWatch Metrics
                           |
                           | Errors >= 1 in 60 seconds
                           v
                   CloudWatch Alarm
                           |
                           v
                       SNS Topic
                           |
                           v
                      Email Alert
```

## Lab Steps

### Step 1: Set AWS Profile and Create Project Folder

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my AWS identity.
- Created a project folder for the lab.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity

mkdir ~\Desktop\workshop-lab-9a
cd ~\Desktop\workshop-lab-9a
pwd
```

**Expected result:**

- AWS CLI returned my account ID and active role.
- PowerShell showed the path to the new `workshop-lab-9a` folder.

**Notes:**

- If the SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Healthy Lambda Function Code

**What I did:**

- Created a Python Lambda function that acts like a basic API endpoint.
- Added structured JSON logging.
- Returned a healthy response.

**File created:**

```text
handler.py
```

**File content:**

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Workshop API - returns a health check response."""
    start_time = time.time()

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request received",
        "request_id": context.aws_request_id,
        "path": event.get("path", "/"),
        "method": event.get("httpMethod", "GET")
    }))

    result = {
        "status": "healthy",
        "message": "Hello from the workshop API!"
    }

    duration = (time.time() - start_time) * 1000

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request completed successfully",
        "request_id": context.aws_request_id,
        "duration_ms": round(duration, 2)
    }))

    return {
        "statusCode": 200,
        "body": json.dumps(result)
    }
```

**Notes:**

- This function returns a `200` status code.
- The structured logs will become useful in Lab 9B.

---

### Step 3: Create Lambda Execution Role

**What I did:**

- Created a trust policy that allows the Lambda service to assume the role.
- Created the Lambda execution role.
- Attached the AWS managed policy that allows Lambda to write logs.

**File created:**

```text
lambda-trust.json
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

**Create the role:**

```powershell
aws iam create-role `
  --role-name workshop-lab9-lambda-role `
  --assume-role-policy-document file://lambda-trust.json
```

**Attach logging permissions:**

```powershell
aws iam attach-role-policy `
  --role-name workshop-lab9-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**Expected result:**

- The role creation command returned JSON containing the role ARN.
- The policy attachment command returned no output.

**Notes:**

- Wait about 10 seconds after attaching the policy so IAM can propagate.

---

### Step 4: Package and Deploy the Lambda Function

**What I did:**

- Packaged `handler.py` into a ZIP file.
- Created the Lambda function using Python 3.12.

**Package command:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
```

**Deploy command:**

```powershell
aws lambda create-function `
  --function-name workshop-api-lab9 `
  --runtime python3.12 `
  --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab9-lambda-role `
  --handler handler.lambda_handler `
  --zip-file fileb://function.zip `
  --timeout 10 `
  --memory-size 128 `
  --region us-east-1
```

**Expected result:**

- AWS returned JSON showing the Lambda function details.
- The function state showed `Pending` or `Active`.

**Notes:**

- Replace `<YOUR_ACCOUNT_ID>` with the 12-digit AWS account ID.
- The ZIP file must contain `handler.py` at the root level.
- If the ZIP contains a folder instead of the file directly, Lambda may fail with `Unable to import module 'handler'`.

---

### Step 5: Invoke the Healthy Function

**What I did:**

- Invoked the Lambda function once.
- Checked the response file.
- Invoked the function five more times to generate CloudWatch metric data.

**Invoke once:**

```powershell
aws lambda invoke `
  --function-name workshop-api-lab9 `
  --region us-east-1 `
  response.json

Get-Content response.json
```

**Expected response:**

```json
{
  "statusCode": 200,
  "body": "{\"status\": \"healthy\", \"message\": \"Hello from the workshop API!\"}"
}
```

**Invoke five more times:**

```powershell
1..5 | ForEach-Object {
  aws lambda invoke `
    --function-name workshop-api-lab9 `
    --region us-east-1 `
    response.json | Out-Null

  Write-Host "Invocation $_ complete"
}
```

**Expected result:**

```text
Invocation 1 complete
Invocation 2 complete
Invocation 3 complete
Invocation 4 complete
Invocation 5 complete
```

---

### Step 6: Review Lambda Metrics in CloudWatch

**What I did:**

- Queried the Lambda `Invocations` metric.
- Queried the Lambda `Errors` metric.
- Confirmed that the function had successful invocations and zero errors.

**Set metric time range:**

```powershell
$endTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$startTime = (Get-Date).AddMinutes(-10).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
```

**Check invocations:**

```powershell
aws cloudwatch get-metric-statistics `
  --namespace "AWS/Lambda" `
  --metric-name Invocations `
  --dimensions "Name=FunctionName,Value=workshop-api-lab9" `
  --start-time $startTime `
  --end-time $endTime `
  --period 60 `
  --statistics Sum `
  --region us-east-1
```

**Check errors:**

```powershell
aws cloudwatch get-metric-statistics `
  --namespace "AWS/Lambda" `
  --metric-name Errors `
  --dimensions "Name=FunctionName,Value=workshop-api-lab9" `
  --start-time $startTime `
  --end-time $endTime `
  --period 60 `
  --statistics Sum `
  --region us-east-1
```

**Expected result:**

- `Invocations` showed datapoints from the Lambda calls.
- `Errors` showed `0.0` or no error spike.

**Console checkpoint:**

- Opened CloudWatch.
- Went to **All metrics**.
- Selected **Lambda**.
- Selected **By Function Name**.
- Found `workshop-api-lab9`.
- Viewed the `Invocations` graph.

**Notes:**

- CloudWatch metrics can take 1–2 minutes to appear.

---

### Step 7: Create SNS Topic and Email Subscription

**What I did:**

- Created an SNS topic for alert emails.
- Subscribed my email address to the topic.
- Confirmed the subscription from my email inbox.

**Create SNS topic:**

```powershell
aws sns create-topic `
  --name workshop-lab9-alerts `
  --region us-east-1
```

**Expected result:**

```json
{
  "TopicArn": "arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts"
}
```

**Subscribe email:**

```powershell
aws sns subscribe `
  --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts `
  --protocol email `
  --notification-endpoint <YOUR_EMAIL> `
  --region us-east-1
```

**Expected result:**

```json
{
  "SubscriptionArn": "pending confirmation"
}
```

**Email confirmation:**

- Checked inbox and spam folder.
- Opened the AWS subscription confirmation email.
- Clicked **Confirm subscription**.

**Notes:**

- The alarm cannot send email alerts until the SNS subscription is confirmed.

---

### Step 8: Create CloudWatch Alarm

**What I did:**

- Created a CloudWatch alarm named `workshop-lab9-errors`.
- Configured the alarm to watch the Lambda `Errors` metric.
- Set the alarm to trigger if errors are greater than or equal to `1` in a 60-second period.
- Configured the alarm to send alerts to the SNS topic.

**Command used:**

```powershell
aws cloudwatch put-metric-alarm `
  --alarm-name workshop-lab9-errors `
  --metric-name Errors `
  --namespace "AWS/Lambda" `
  --statistic Sum `
  --period 60 `
  --threshold 1 `
  --comparison-operator GreaterThanOrEqualToThreshold `
  --evaluation-periods 1 `
  --dimensions "Name=FunctionName,Value=workshop-api-lab9" `
  --alarm-actions arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts `
  --region us-east-1
```

**Check alarm state:**

```powershell
aws cloudwatch describe-alarms `
  --alarm-names workshop-lab9-errors `
  --region us-east-1 `
  --query "MetricAlarms[0].[AlarmName,StateValue]" `
  --output text
```

**Expected result:**

```text
workshop-lab9-errors    INSUFFICIENT_DATA
```

**Notes:**

- `INSUFFICIENT_DATA` is normal immediately after alarm creation.
- The alarm should move to `OK` after CloudWatch has enough data to evaluate.
- The alarm watches for any Lambda error in a one-minute period.

---

### Step 9: Deploy Broken Lambda Code

**What I did:**

- Replaced the healthy Lambda function with a broken version.
- The broken version references a variable named `database_url` that does not exist.
- Repackaged and deployed the broken function.

**Updated `handler.py`:**

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Workshop API - THIS VERSION HAS A BUG."""
    start_time = time.time()

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request received",
        "request_id": context.aws_request_id,
        "path": event.get("path", "/"),
        "method": event.get("httpMethod", "GET")
    }))

    # BUG: someone removed the database_url variable but left this reference
    connection_string = database_url

    result = {
        "status": "healthy",
        "message": "Hello from the workshop API!"
    }

    duration = (time.time() - start_time) * 1000

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request completed successfully",
        "request_id": context.aws_request_id,
        "duration_ms": round(duration, 2)
    }))

    return {
        "statusCode": 200,
        "body": json.dumps(result)
    }
```

**Repackage and deploy:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force

aws lambda update-function-code `
  --function-name workshop-api-lab9 `
  --zip-file fileb://function.zip `
  --region us-east-1 `
  --query "LastUpdateStatus" `
  --output text
```

**Expected result:**

```text
InProgress
```

or:

```text
Successful
```

**Notes:**

- The bug is intentional.
- Python will not fail during packaging. It fails when the function is invoked.

---

### Step 10: Invoke Broken Function and Trigger Alarm

**What I did:**

- Invoked the broken Lambda function.
- Confirmed it returned an unhandled function error.
- Invoked it five more times to create an error spike.

**Invoke once:**

```powershell
aws lambda invoke `
  --function-name workshop-api-lab9 `
  --region us-east-1 `
  response.json

Get-Content response.json
```

**Expected result:**

- The invoke output showed:

```text
"FunctionError": "Unhandled"
```

- The `response.json` file showed:

```text
"name 'database_url' is not defined"
```

**Invoke five more times:**

```powershell
1..5 | ForEach-Object {
  aws lambda invoke `
    --function-name workshop-api-lab9 `
    --region us-east-1 `
    response.json | Out-Null

  Write-Host "Error invocation $_ done"
}
```

**Expected result:**

```text
Error invocation 1 done
Error invocation 2 done
Error invocation 3 done
Error invocation 4 done
Error invocation 5 done
```

---

### Step 11: Watch the Alarm Trigger

**What I did:**

- Waited 1–2 minutes for CloudWatch to evaluate the new error data.
- Checked the alarm state.
- Confirmed the alarm entered the `ALARM` state.
- Confirmed an email alert arrived from AWS.

**Command used:**

```powershell
aws cloudwatch describe-alarms `
  --alarm-names workshop-lab9-errors `
  --region us-east-1 `
  --query "MetricAlarms[0].[AlarmName,StateValue,StateReason]" `
  --output text
```

**Expected result:**

```text
workshop-lab9-errors    ALARM    Threshold Crossed
```

**Email checkpoint:**

- Checked email inbox.
- Confirmed an AWS alert email with a subject similar to:

```text
ALARM: workshop-lab9-errors
```

**Console checkpoint:**

- Opened CloudWatch.
- Went to **Alarms** → **All alarms**.
- Confirmed `workshop-lab9-errors` showed **In alarm**.
- Opened the alarm graph and confirmed the error spike.

**What this proved:**

- Without the alarm, the Lambda failures would have been silent.
- With the alarm, CloudWatch detected the errors and SNS sent an email notification.

---

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| Local lab folder created | Passed |
| Healthy `handler.py` created | Passed |
| Lambda trust policy created | Passed |
| Lambda execution role created | Passed |
| Lambda logging policy attached | Passed |
| Lambda ZIP package created | Passed |
| Lambda function deployed | Passed |
| Healthy Lambda invocation succeeded | Passed |
| Baseline invocations generated | Passed |
| `Invocations` metric reviewed | Passed |
| `Errors` metric reviewed | Passed |
| Healthy baseline confirmed | Passed |
| SNS topic created | Passed |
| Email subscription created | Passed |
| SNS email subscription confirmed | Passed |
| CloudWatch alarm created | Passed |
| Alarm state checked | Passed |
| Broken Lambda code deployed | Passed |
| Broken Lambda invocation failed as expected | Passed |
| Error spike generated | Passed |
| Alarm changed to `ALARM` | Passed |
| SNS alert email received | Passed |
| CloudWatch alarm graph reviewed | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `No such file or directory: lambda-trust.json` | Terminal is not inside the lab folder | Run `cd ~\Desktop\workshop-lab-9a` and verify with `pwd` |
| `The role cannot be assumed` | IAM role propagation has not completed | Wait 10–30 seconds and retry the Lambda create command |
| `Function already exists` | A previous attempt already created the Lambda function | Skip creation or delete it with `aws lambda delete-function --function-name workshop-api-lab9 --region us-east-1` |
| `Unable to import module 'handler'` | ZIP file contains the wrong structure | Delete `function.zip`, make sure you are inside `workshop-lab-9a`, then run `Compress-Archive -Path handler.py -DestinationPath function.zip -Force` |
| Lambda still returns healthy response after update | ZIP file contains stale or incorrect code | Delete `function.zip`, confirm `handler.py` contains the `database_url` bug, re-zip, and redeploy |
| CloudWatch metrics show empty datapoints | Metrics have not appeared yet | Wait 1–2 minutes and rerun the metric command |
| Alarm stays in `INSUFFICIENT_DATA` | CloudWatch does not have enough recent data to evaluate | Invoke the function again and wait 1–2 minutes |
| Alarm stays in `OK` after broken invocations | CloudWatch has not evaluated the error period yet or errors were not generated | Confirm the Lambda response includes `FunctionError`, wait, and retry the alarm-state command |
| No email received | SNS subscription was not confirmed | Check inbox/spam and click the AWS confirmation link |
| `InvalidParameterValue` on alarm creation | SNS topic ARN is wrong | Confirm account ID, region, and topic name in the `--alarm-actions` ARN |
| AWS CLI authentication error | SSO session expired or profile not set | Run `aws sso login --profile <YOUR_PROFILE_NAME>`, then set `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"` |

## Cleanup

Complete cleanup after the lab to remove all resources created in Lab 9A.

### Step 1: Delete the CloudWatch Alarm

```powershell
aws cloudwatch delete-alarms `
  --alarm-names workshop-lab9-errors `
  --region us-east-1
```

### Step 2: Delete the SNS Topic

```powershell
aws sns delete-topic `
  --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts `
  --region us-east-1
```

### Step 3: Delete the Lambda Function

```powershell
aws lambda delete-function `
  --function-name workshop-api-lab9 `
  --region us-east-1
```

### Step 4: Delete the CloudWatch Log Group

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-api-lab9 `
  --region us-east-1
```

### Step 5: Remove the Lambda IAM Role

```powershell
aws iam detach-role-policy `
  --role-name workshop-lab9-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role `
  --role-name workshop-lab9-lambda-role
```

### Step 6: Delete the Local Lab Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-9a
```

## Cleanup Verification

### Verify Lambda Function Is Deleted

```powershell
aws lambda list-functions `
  --region us-east-1 `
  --query "Functions[?FunctionName=='workshop-api-lab9'].FunctionName"
```

**Expected result:**

```json
[]
```

### Verify CloudWatch Alarm Is Deleted

```powershell
aws cloudwatch describe-alarms `
  --alarm-names workshop-lab9-errors `
  --region us-east-1 `
  --query "MetricAlarms[].AlarmName"
```

**Expected result:**

```json
[]
```

### Verify SNS Topic Is Deleted

```powershell
aws sns list-topics `
  --region us-east-1 `
  --query "Topics[?contains(TopicArn, 'workshop-lab9-alerts')].TopicArn"
```

**Expected result:**

```json
[]
```

### Verify IAM Role Is Deleted

```powershell
aws iam get-role `
  --role-name workshop-lab9-lambda-role
```

**Expected result:**

```text
NoSuchEntity
```

**Expected cleanup result:**

| Resource | Expected State |
|---|---|
| `workshop-lab9-errors` CloudWatch alarm | Deleted |
| `workshop-lab9-alerts` SNS topic | Deleted |
| `workshop-api-lab9` Lambda function | Deleted |
| `/aws/lambda/workshop-api-lab9` log group | Deleted |
| `workshop-lab9-lambda-role` IAM role | Deleted |
| Local `workshop-lab-9a` folder | Deleted |

## What I Learned

- CloudWatch automatically collects Lambda metrics such as `Invocations`, `Errors`, `Duration`, and `Throttles`.
- Metrics show system behavior over time.
- A healthy baseline helps identify what normal application behavior looks like.
- Lambda errors can happen even if code deploys successfully.
- CloudWatch alarms turn passive metrics into active alerts.
- Alarm states include `OK`, `ALARM`, and `INSUFFICIENT_DATA`.
- SNS can deliver CloudWatch alarm notifications to email subscribers.
- Email subscriptions must be confirmed before SNS sends alerts.
- A one-minute alarm period can detect failures quickly.
- Structured logs make troubleshooting easier and prepare the environment for deeper log analysis in Lab 9B.
- Monitoring reduces silent failures and improves mean time to detect.
- Operational Excellence depends on proactive monitoring and alerting.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/aws-identity-verified.png` | AWS CLI identity verified |
| `screenshots/lab-folder-created.png` | Local `workshop-lab-9a` folder created |
| `screenshots/healthy-handler-created.png` | Healthy Lambda handler code created |
| `screenshots/lambda-trust-policy-created.png` | Lambda trust policy created |
| `screenshots/lambda-role-created.png` | Lambda execution role created |
| `screenshots/lambda-policy-attached.png` | Lambda logging policy attached |
| `screenshots/lambda-zip-created.png` | Lambda ZIP package created |
| `screenshots/lambda-function-created.png` | Lambda function deployed |
| `screenshots/healthy-invocation-success.png` | Healthy Lambda invocation returned successful response |
| `screenshots/baseline-invocations-generated.png` | Multiple healthy invocations generated |
| `screenshots/cloudwatch-invocations-metric.png` | CloudWatch `Invocations` metric reviewed |
| `screenshots/cloudwatch-errors-zero.png` | CloudWatch `Errors` metric showed healthy baseline |
| `screenshots/sns-topic-created.png` | SNS alert topic created |
| `screenshots/sns-subscription-pending.png` | SNS email subscription created |
| `screenshots/sns-subscription-confirmed.png` | SNS subscription confirmed |
| `screenshots/cloudwatch-alarm-created.png` | CloudWatch alarm created |
| `screenshots/alarm-insufficient-data.png` | Alarm initially showed `INSUFFICIENT_DATA` |
| `screenshots/broken-handler-created.png` | Broken Lambda code created |
| `screenshots/broken-code-deployed.png` | Broken Lambda function code deployed |
| `screenshots/lambda-function-error.png` | Lambda invocation returned `FunctionError` |
| `screenshots/error-invocations-generated.png` | Multiple failed invocations generated |
| `screenshots/alarm-state-alarm.png` | CloudWatch alarm entered `ALARM` state |
| `screenshots/sns-alert-email.png` | SNS alert email received |
| `screenshots/alarm-error-spike-graph.png` | CloudWatch graph showed error spike |
| `screenshots/cleanup-verified.png` | Lab resources removed successfully |