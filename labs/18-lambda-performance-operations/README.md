# Lab 18: Lambda Performance & Operations

## Lab Summary

In this lab, I applied the Performance Efficiency and Operational Excellence pillars of the AWS Well-Architected Framework to an AWS Lambda function.

I deployed a Lambda function with low memory, measured its performance while completing CPU-intensive work, then increased the memory allocation and compared the results. I also simulated a Lambda failure, created a CloudWatch alarm for Lambda errors, connected the alarm to an SNS topic, and verified that an email notification was sent when the function crashed.

This lab demonstrated how Lambda memory affects CPU allocation and why automated monitoring is important for production workloads.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 6B — Architecting Lambda for Performance & Operational Excellence
* Session: 6 — AWS Well-Architected Framework
* Track: Solutions Architecture
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Set the active AWS CLI profile
* Verify AWS CLI authentication
* Create a local project folder
* Create a Lambda IAM execution role
* Attach Lambda logging permissions
* Create a Python Lambda workload function
* Package and deploy the Lambda function with 128 MB memory
* Measure function performance with a CPU-intensive workload
* Increase Lambda memory to 256 MB
* Measure and compare the new execution time
* Simulate a Lambda crash
* Create an SNS topic and email subscription
* Create a CloudWatch alarm for Lambda `Errors`
* Trigger the alarm with another function crash
* Verify the alarm state and email notification
* Clean up all lab resources

## Services / Tools Used

| Service / Tool                 | Purpose                                                             |
| ------------------------------ | ------------------------------------------------------------------- |
| AWS Lambda                     | Runs the CPU-intensive workload and simulated failure               |
| AWS IAM                        | Provides Lambda with permissions to write logs                      |
| Amazon CloudWatch              | Stores Lambda metrics and monitors function errors                  |
| CloudWatch Alarm               | Detects Lambda errors and changes state when the threshold is met   |
| Amazon SNS                     | Sends the alarm notification to an email subscriber                 |
| AWS CLI                        | Creates, deploys, tests, monitors, and deletes resources            |
| PowerShell                     | Runs commands and packages the Lambda function                      |
| Python                         | Lambda runtime language                                             |
| AWS Well-Architected Framework | Provides Performance Efficiency and Operational Excellence guidance |

## Prerequisites

* Completed Lab 01: AWS Account & CLI Setup
* AWS CLI installed and configured
* Working AWS IAM Identity Center / SSO profile
* Access to an email inbox
* Text editor such as VS Code or Notepad
* Basic understanding of Lambda, IAM roles, CloudWatch, and SNS

## Cost Notice

Estimated cost: `$0.00`

| Service    | Cost Consideration                                                                |
| ---------- | --------------------------------------------------------------------------------- |
| AWS Lambda | Expected to remain inside the Lambda free tier for this lab                       |
| CloudWatch | Expected to remain inside the free tier due to low metric and log usage           |
| Amazon SNS | Expected to remain inside the free tier for a small number of email notifications |

## Key Concepts

| Concept                | Meaning                                                                                                  |
| ---------------------- | -------------------------------------------------------------------------------------------------------- |
| Performance Efficiency | Using computing resources efficiently while meeting workload requirements.                               |
| Operational Excellence | Monitoring workloads, responding to failures, and improving operations over time.                        |
| Lambda Memory          | Memory allocated to a Lambda function. Increasing memory also increases CPU allocation.                  |
| Right-Sizing           | Selecting the memory configuration that provides a good balance between cost and performance.            |
| CPU-Bound Workload     | A task limited mainly by processing power, such as calculating prime numbers.                            |
| CloudWatch Metric      | A measurable value published by AWS services, such as Lambda duration or errors.                         |
| CloudWatch Alarm       | A rule that evaluates a metric and takes an action when a threshold is met.                              |
| SNS Topic              | A notification channel used to send messages to subscribers.                                             |
| Lambda Error           | An invocation that fails because the function throws an exception or encounters another unhandled error. |

## Security Notes

| Topic                          | Explanation                                                                                                                  |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| Email confirmation is required | SNS will not send notifications until the email subscription is confirmed.                                                   |
| Alarm scope matters            | The CloudWatch alarm is limited to the specific Lambda function using a dimension filter.                                    |
| Lambda execution role          | The function only receives basic CloudWatch Logs permissions for this lab.                                                   |
| Production monitoring          | Production workloads may send alarms to an incident platform, ticketing system, or messaging platform instead of only email. |
| Cost awareness                 | Increasing Lambda memory can improve speed, but should be tested to find the most efficient configuration.                   |

## Architecture Overview

```text
Performance Testing

Lambda Function
  128 MB memory
      |
      v
CPU-intensive workload
      |
      v
Measure duration
      |
      v
Increase to 256 MB
      |
      v
Measure duration again


Failure Monitoring

Lambda crash
      |
      v
CloudWatch Errors metric
      |
      v
CloudWatch Alarm
      |
      v
SNS Topic
      |
      v
Email notification
```

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

* Set the AWS CLI profile for the current PowerShell session.
* Verified the active AWS identity.

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

* If the SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

* Created a local workspace for Lab 6B.

**Commands used:**

```powershell
mkdir ~\Desktop\workshop-lab-6b
cd ~\Desktop\workshop-lab-6b
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-6b
```

---

### Step 3: Create Lambda IAM Role

#### Step 3A: Create Trust Policy

Created `trust-policy.json`:

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

#### Step 3B: Create Role and Attach Logging Policy

```powershell
aws iam create-role `
  --role-name workshop-waf-lambda-role `
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy `
  --role-name workshop-waf-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**Result:**

* Lambda execution role was created.
* Lambda received permission to create and write CloudWatch Logs.

**Notes:**

* IAM role propagation may take 10–30 seconds before Lambda can use the role.

---

### Step 4: Create Lambda Workload Function

**What I did:**

* Created a Lambda function that calculates prime numbers.
* Added a crash option for testing error alerting.

Created `workload_function.py`:

```python
import json
import time
import math

def lambda_handler(event, context):
    action = event.get("action", "")

    if action == "crash":
        raise Exception("ERROR: Invalid input received. Application crashed!")

    start = time.time()

    n = event.get("workload_size", 50000)
    primes = []

    for num in range(2, n):
        is_prime = True

        for i in range(2, int(math.sqrt(num)) + 1):
            if num % i == 0:
                is_prime = False
                break

        if is_prime:
            primes.append(num)

    duration_ms = round((time.time() - start) * 1000, 2)
    memory_mb = context.memory_limit_in_mb

    result = (
        f"PERFORMANCE REPORT\n"
        f"==================\n"
        f"Memory allocated: {memory_mb} MB\n"
        f"Computation time: {duration_ms} ms\n"
        f"Primes found: {len(primes)} (up to {n})\n"
        f"=================="
    )

    return {
        "statusCode": 200,
        "body": result
    }
```

**Notes:**

* The function performs CPU-intensive work so memory-related performance differences are easier to measure.
* Sending `{"action": "crash"}` intentionally causes the function to fail.

---

### Step 5: Package and Deploy Lambda at 128 MB

#### Step 5A: Create Deployment Package

```powershell
Compress-Archive -Path workload_function.py -DestinationPath workload.zip -Force
```

#### Step 5B: Deploy Lambda Function

```powershell
aws lambda create-function `
  --function-name workshop-waf-workload `
  --runtime python3.12 `
  --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-lambda-role" `
  --handler workload_function.lambda_handler `
  --zip-file fileb://workload.zip `
  --memory-size 128 `
  --timeout 30 `
  --region us-east-1
```

**Result:**

* Lambda function was deployed with 128 MB memory.

## Part 1: Performance Efficiency Pillar

### Step 6: Measure Performance at 128 MB

#### Step 6A: Create Workload Payload

Created `workload-payload.json`:

```json
{
  "workload_size": 50000
}
```

#### Step 6B: Invoke Lambda

```powershell
aws lambda invoke `
  --function-name workshop-waf-workload `
  --payload file://workload-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  response-128mb.json

Write-Output "=== RESULT (128 MB Memory) ==="
(Get-Content response-128mb.json | ConvertFrom-Json).body
```

**Expected result:**

```text
=== RESULT (128 MB Memory) ===
PERFORMANCE REPORT
==================
Memory allocated: 128 MB
Computation time: <RESULT> ms
Primes found: 5133 (up to 50000)
==================
```

**Result:**

* I invoked the function multiple times and recorded a typical computation time.

**My typical 128 MB result:**

```text
<RECORD_TYPICAL_TIME_HERE> ms
```

---

### Step 7: Increase Memory to 256 MB

**What I did:**

* Increased Lambda memory from 128 MB to 256 MB.
* This also increased the CPU allocated to the function.

**Command used:**

```powershell
aws lambda update-function-configuration `
  --function-name workshop-waf-workload `
  --memory-size 256 `
  --region us-east-1 `
  --query "MemorySize" `
  --output text
```

**Expected result:**

```text
256
```

**Notes:**

* Waited a few seconds for the Lambda configuration update to finish.

---

### Step 8: Measure Performance at 256 MB

**Command used:**

```powershell
aws lambda invoke `
  --function-name workshop-waf-workload `
  --payload file://workload-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  response-256mb.json

Write-Output "=== RESULT (256 MB Memory) ==="
(Get-Content response-256mb.json | ConvertFrom-Json).body
```

**Expected result:**

```text
=== RESULT (256 MB Memory) ===
PERFORMANCE REPORT
==================
Memory allocated: 256 MB
Computation time: <RESULT> ms
Primes found: 5133 (up to 50000)
==================
```

**Result:**

* I invoked the function multiple times and recorded a typical computation time.

**My typical 256 MB result:**

```text
<RECORD_TYPICAL_TIME_HERE> ms
```

---

### Step 9: Compare Performance Results

| Configuration | Typical Computation Time | Improvement                |
| ------------- | -----------------------: | -------------------------- |
| 128 MB memory |              `<TIME> ms` | Baseline                   |
| 256 MB memory |              `<TIME> ms` | `<CALCULATED_IMPROVEMENT>` |

**Finding:**

* Increasing Lambda memory increased CPU allocation.
* The same workload completed faster at 256 MB.
* The goal is not always to choose the largest memory value, but to find the best balance between speed and cost.

## Part 2: Operational Excellence Pillar

### Step 10: Simulate a Silent Failure

#### Step 10A: Create Crash Payload

Created `crash-payload.json`:

```json
{
  "action": "crash"
}
```

#### Step 10B: Invoke Lambda With Crash Payload

```powershell
aws lambda invoke `
  --function-name workshop-waf-workload `
  --payload file://crash-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  crash-response.json

Write-Output "=== CRASH RESULT ==="
Get-Content crash-response.json
```

**Expected result:**

```text
FunctionError: Unhandled
```

**Expected response body:**

```json
{
  "errorMessage": "ERROR: Invalid input received. Application crashed!"
}
```

**Finding:**

* The Lambda function crashed.
* Without an alarm, the failure would only be visible to someone manually checking the invocation result or logs.

---

### Step 11: Create SNS Alert Topic

#### Step 11A: Create SNS Topic

```powershell
aws sns create-topic `
  --name workshop-waf-alerts `
  --region us-east-1
```

**Expected result:**

```json
{
  "TopicArn": "arn:aws:sns:us-east-1:REDACTED:workshop-waf-alerts"
}
```

**My Topic ARN:**

```text
<PASTE_TOPIC_ARN_HERE>
```

#### Step 11B: Subscribe Email Address

```powershell
aws sns subscribe `
  --topic-arn <TOPIC_ARN> `
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

#### Step 11C: Confirm Subscription

**What I did:**

* Opened the SNS confirmation email.
* Clicked the confirmation link.
* Confirmed the subscription was active.

**Verification command:**

```powershell
aws sns list-subscriptions-by-topic `
  --topic-arn <TOPIC_ARN> `
  --region us-east-1 `
  --query "Subscriptions[].{Protocol:Protocol,Status:SubscriptionArn}" `
  --output table
```

---

### Step 12: Create CloudWatch Error Alarm

**What I did:**

* Created an alarm that monitors the Lambda `Errors` metric.
* Configured the alarm to send an SNS notification when one or more errors occur within a 60-second period.

**Command used:**

```powershell
aws cloudwatch put-metric-alarm `
  --alarm-name workshop-lambda-errors `
  --metric-name Errors `
  --namespace AWS/Lambda `
  --dimensions "Name=FunctionName,Value=workshop-waf-workload" `
  --statistic Sum `
  --period 60 `
  --evaluation-periods 1 `
  --threshold 1 `
  --comparison-operator GreaterThanOrEqualToThreshold `
  --alarm-actions <TOPIC_ARN> `
  --region us-east-1
```

**Result:**

* CloudWatch alarm was created successfully.

**Alarm logic:**

```text
If Lambda Errors >= 1 during one 60-second period:
Change alarm state to ALARM
Send an SNS email notification
```

---

### Step 13: Trigger and Verify the Alarm

#### Step 13A: Trigger Another Crash

```powershell
aws lambda invoke `
  --function-name workshop-waf-workload `
  --payload file://crash-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  crash-response2.json
```

#### Step 13B: Check Alarm State

```powershell
aws cloudwatch describe-alarms `
  --alarm-names workshop-lambda-errors `
  --query "MetricAlarms[0].{Name:AlarmName,State:StateValue}" `
  --output table `
  --region us-east-1
```

**Possible states:**

| State               | Meaning                                                       |
| ------------------- | ------------------------------------------------------------- |
| `INSUFFICIENT_DATA` | Alarm has not received enough metric data yet.                |
| `ALARM`             | Lambda error was detected and the notification action ran.    |
| `OK`                | No errors were detected during the current evaluation period. |

**Expected result:**

```text
ALARM
```

#### Step 13C: Verify Email Notification

**What I did:**

* Checked my email inbox and spam folder.
* Confirmed receipt of an AWS CloudWatch alarm notification.

**Expected email subject:**

```text
ALARM: workshop-lambda-errors
```

#### Step 13D: Console Checkpoint

**What I did:**

* Opened CloudWatch in the AWS Console.
* Opened **Alarms**.
* Selected `workshop-lambda-errors`.
* Confirmed the alarm state and error metric graph.

**Result:**

* Lambda failures were now automatically detected and reported.

## Validation / Checkpoints

| Checkpoint                           | Result |
| ------------------------------------ | ------ |
| AWS CLI profile set                  | Passed |
| AWS identity verified                | Passed |
| Local project folder created         | Passed |
| Lambda trust policy created          | Passed |
| Lambda IAM role created              | Passed |
| Lambda logging policy attached       | Passed |
| Workload function created            | Passed |
| Lambda deployment package created    | Passed |
| Lambda deployed with 128 MB memory   | Passed |
| 128 MB performance measured          | Passed |
| Lambda memory updated to 256 MB      | Passed |
| 256 MB performance measured          | Passed |
| Performance comparison completed     | Passed |
| Crash payload created                | Passed |
| Lambda crash simulated               | Passed |
| SNS topic created                    | Passed |
| Email subscription created           | Passed |
| Email subscription confirmed         | Passed |
| CloudWatch error alarm created       | Passed |
| Alarm triggered with Lambda error    | Passed |
| Alarm state verified                 | Passed |
| Email alert received                 | Passed |
| CloudWatch alarm verified in console | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                                    | What It Means                                                 | How to Fix                                                                                       |
| -------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| Lambda creation fails because the role cannot be assumed | IAM role propagation is still in progress                     | Wait 10–30 seconds and retry the create-function command                                         |
| Lambda payload error occurs                              | PowerShell or AWS CLI did not read the JSON payload correctly | Use `--payload file://filename.json` rather than inline JSON                                     |
| 128 MB and 256 MB results are too similar                | Workload may be too small or results may vary between runs    | Test two or three times and use a typical result; increase `workload_size` to `100000` if needed |
| Alarm remains in `INSUFFICIENT_DATA`                     | The Lambda error metric has not been evaluated yet            | Trigger another crash, wait 1–3 minutes, and check again                                         |
| Alarm does not send an email                             | SNS subscription may still be pending or email may be in spam | Confirm the subscription and check junk/spam folders                                             |
| Response file cannot be read                             | File was not created or PowerShell is in the wrong directory  | Run `pwd`, confirm the folder, and rerun the Lambda invoke command                               |
| SSO token expired                                        | AWS IAM Identity Center session expired                       | Run `aws sso login --profile <YOUR_PROFILE_NAME>`                                                |

## Cleanup

### Step 1: Delete CloudWatch Alarm

```powershell
aws cloudwatch delete-alarms `
  --alarm-names workshop-lambda-errors `
  --region us-east-1
```

### Step 2: Find and Remove SNS Subscription

```powershell
aws sns list-subscriptions-by-topic `
  --topic-arn <TOPIC_ARN> `
  --region us-east-1
```

```powershell
aws sns unsubscribe `
  --subscription-arn <SUBSCRIPTION_ARN> `
  --region us-east-1
```

### Step 3: Delete SNS Topic

```powershell
aws sns delete-topic `
  --topic-arn <TOPIC_ARN> `
  --region us-east-1
```

### Step 4: Delete Lambda Function

```powershell
aws lambda delete-function `
  --function-name workshop-waf-workload `
  --region us-east-1
```

### Step 5: Delete Lambda Log Group

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-waf-workload `
  --region us-east-1
```

### Step 6: Detach Policy and Delete IAM Role

```powershell
aws iam detach-role-policy `
  --role-name workshop-waf-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role `
  --role-name workshop-waf-lambda-role
```

### Step 7: Delete Local Working Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-6b
```

## Cleanup Verification

| Resource               | Cleanup Action | Verified |
| ---------------------- | -------------- | -------- |
| CloudWatch alarm       | Deleted        | Yes      |
| SNS email subscription | Removed        | Yes      |
| SNS topic              | Deleted        | Yes      |
| Lambda function        | Deleted        | Yes      |
| Lambda log group       | Deleted        | Yes      |
| Lambda IAM role policy | Detached       | Yes      |
| Lambda IAM role        | Deleted        | Yes      |
| Local lab folder       | Deleted        | Yes      |

## What I Learned

* Lambda memory allocation affects CPU allocation.
* Increasing Lambda memory can make CPU-bound workloads complete faster.
* Right-sizing means balancing performance and cost rather than always choosing the highest memory setting.
* Lambda performance should be measured multiple times because execution results can vary.
* Lambda `Errors` is a CloudWatch metric that can be monitored automatically.
* CloudWatch alarms can detect failures without requiring someone to watch a dashboard.
* SNS can send an email alert when a CloudWatch alarm enters the `ALARM` state.
* The Performance Efficiency pillar focuses on efficient resource usage.
* The Operational Excellence pillar focuses on monitoring, visibility, and continuous operational improvement.
* Automated alerting reduces the time between a failure and human awareness.

## Screenshots

| Screenshot                                   | Description                                        |
| -------------------------------------------- | -------------------------------------------------- |
| `screenshots/project-folder-created.png`     | Local Lab 6B folder created                        |
| `screenshots/lambda-role-created.png`        | Lambda IAM role created                            |
| `screenshots/lambda-policy-attached.png`     | Lambda logging policy attached                     |
| `screenshots/workload-code-created.png`      | Python workload function created                   |
| `screenshots/lambda-128mb-created.png`       | Lambda deployed with 128 MB memory                 |
| `screenshots/performance-128mb.png`          | Performance result at 128 MB                       |
| `screenshots/lambda-memory-256mb.png`        | Lambda memory increased to 256 MB                  |
| `screenshots/performance-256mb.png`          | Performance result at 256 MB                       |
| `screenshots/performance-comparison.png`     | Before-and-after performance comparison            |
| `screenshots/crash-before-alarm.png`         | Lambda crash before monitoring was configured      |
| `screenshots/sns-topic-created.png`          | SNS alert topic created                            |
| `screenshots/sns-subscription-confirmed.png` | SNS email subscription confirmed                   |
| `screenshots/cloudwatch-alarm-created.png`   | Lambda errors alarm created                        |
| `screenshots/alarm-state-alarm.png`          | CloudWatch alarm entered ALARM state               |
| `screenshots/email-alert-received.png`       | Email notification received                        |
| `screenshots/cloudwatch-console-alarm.png`   | Alarm and error metric graph in CloudWatch console |
| `screenshots/cleanup-verified.png`           | Cleanup completed                                  |