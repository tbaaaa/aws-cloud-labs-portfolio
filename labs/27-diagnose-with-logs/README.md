# Lab 27: Diagnose with CloudWatch Logs and Log Insights

## Lab Summary

In this lab, I investigated a broken Lambda function using CloudWatch metrics, CloudWatch Logs, and CloudWatch Logs Insights.

In Lab 9A, the CloudWatch alarm told me that something was wrong because the Lambda function started generating errors. In this lab, I treated that alarm as the starting point of an incident investigation. I checked CloudWatch metrics to understand when the problem started and how severe it was, then reviewed CloudWatch log streams to find the exact error message and stack trace.

After finding the root cause, I used CloudWatch Logs Insights to query the logs at scale instead of manually scrolling through entries. I then fixed the Lambda code, redeployed the function, invoked it again, and verified that the errors stopped and the alarm returned to a healthy state.

This lab demonstrated the investigation workflow engineers use after an alert fires: alert, metrics, logs, query, root cause, fix, and verify.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 9B — Diagnose the Problem — CloudWatch Logs & Log Insights
- Session: 9 — Monitoring & Observability
- Track: Solutions Architecture
- Difficulty: Intermediate
- Target certification: AWS Solutions Architect – Associate

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Confirm the broken Lambda function from Lab 9A exists
- Confirm the Lambda function returns an unhandled error
- Treat the CloudWatch alarm as the start of an investigation
- Check Lambda `Errors` metrics to identify when failures started
- Check Lambda `Invocations` metrics to understand failure severity
- Find the Lambda CloudWatch log group
- List recent CloudWatch log streams
- Read raw Lambda log events
- Identify the exact error type, message, and code location
- Use CloudWatch Logs Insights to search for error logs
- Use CloudWatch Logs Insights to count errors per minute
- Fix the Lambda code by removing the undefined variable reference
- Repackage and redeploy the fixed Lambda function
- Invoke the fixed function successfully
- Generate healthy invocation metrics
- Verify recent Lambda errors return to zero
- Verify the CloudWatch alarm returns to `OK`
- Review the incident visually in the CloudWatch console
- Preserve the Lambda function and role if continuing to Lab 9C

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Lambda | Runs the workshop API function being investigated |
| Amazon CloudWatch Metrics | Shows when errors started and how severe the outage was |
| Amazon CloudWatch Logs | Stores Lambda logs, stack traces, and application messages |
| CloudWatch Logs Insights | Queries Lambda logs at scale |
| Amazon CloudWatch Alarms | Provides the alert that starts the investigation |
| Amazon SNS | Sends the alert email from Lab 9A |
| AWS CLI | Invokes Lambda, checks metrics, reads logs, and runs Logs Insights queries |
| PowerShell | Runs investigation and deployment commands on Windows |
| VS Code | Edits the Lambda handler code |

## Prerequisites

- Completed Lab 9A: CloudWatch Alarms
- AWS CLI installed
- AWS CLI authenticated with a working profile
- Lambda function named `workshop-api-lab9` exists
- Broken Lambda code from Lab 9A is deployed
- CloudWatch alarm named `workshop-lab9-errors` exists if using the full alarm recovery workflow
- Local project folder `workshop-lab-9a` exists, or the function can be recreated from Lab 9A
- PowerShell available on Windows
- VS Code or another text editor installed

> If Lab 9A was cleaned up, recreate the Lambda role and function from Lab 9A, then deploy the broken version of `handler.py` before starting this lab.

## Cost Notice

Estimated cost: `$0.00`

| Service | Cost Consideration |
|---|---|
| AWS Lambda | Expected to remain within the Lambda Always Free usage for this lab |
| CloudWatch Logs | Small log volume; expected to remain within free-tier usage |
| CloudWatch Logs Insights | Small query volume; expected to remain within free-tier usage |
| CloudWatch Metrics | Basic Lambda metrics are collected automatically |
| AWS IAM | Free |

> If continuing to Lab 9C, keep the Lambda function and role. If stopping here, complete the cleanup section.

## Key Concepts

| Concept | Meaning |
|---|---|
| Incident Investigation | The process of finding what failed, when it failed, why it failed, and how to prove recovery |
| CloudWatch Log Group | A container for logs from one source, such as `/aws/lambda/workshop-api-lab9` |
| CloudWatch Log Stream | A sequence of log events from a specific Lambda execution environment |
| Structured Logging | Writing logs in a consistent format, such as JSON, to make searching and analysis easier |
| Stack Trace | Error output that shows where code failed and the path that led to the failure |
| CloudWatch Logs Insights | AWS query service for searching and analyzing CloudWatch logs |
| `@timestamp` | Built-in Logs Insights field showing when a log event occurred |
| `@message` | Built-in Logs Insights field containing the log message |
| `filter` | Logs Insights command used to return matching log events |
| `stats` | Logs Insights command used to aggregate log data |
| `bin(1m)` | Groups log events into one-minute time windows |
| MTTD | Mean Time to Detect; how long it takes to know something is broken |
| MTTR | Mean Time to Resolve; how long it takes to fix the problem after detection |
| Root Cause | The underlying reason the failure happened |
| Recovery Verification | Proving through metrics and logs that the system is healthy again |

## Security Notes

| Topic | Explanation |
|---|---|
| Logs can contain sensitive data | Production logs should avoid secrets, tokens, passwords, and private user data |
| Structured logs improve investigations | JSON-style logs make it easier to find request IDs, error levels, and failure patterns |
| Least privilege remains important | The Lambda role only needs permissions required to write logs |
| Monitoring is security-relevant | Fast detection helps reduce impact from failures and suspicious behavior |
| Error visibility matters | Silent failures can hide operational and security issues |
| Log retention should be managed | Keeping logs forever can increase cost and risk; retention should match business needs |
| Investigation evidence should be preserved | Metrics, log timestamps, and screenshots help support incident reports and post-incident reviews |

## Architecture Overview

```text
CloudWatch Alarm
      |
      | Alert: Lambda Errors >= threshold
      v
Investigation Workflow
      |
      +--> CloudWatch Metrics
      |       - When did errors start?
      |       - How many invocations failed?
      |
      +--> CloudWatch Logs
      |       - What error occurred?
      |       - Where did the code fail?
      |
      +--> CloudWatch Logs Insights
      |       - Query errors at scale
      |       - Count errors per minute
      |
      v
Fix Lambda Code
      |
      v
Redeploy and Verify Recovery
```

## Lab Steps

### Step 1: Set AWS Profile and Confirm Broken Function

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my AWS identity.
- Invoked the Lambda function to confirm the broken version was deployed.
- Confirmed that the Lambda function returned an unhandled error.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity

cd ~\Desktop\workshop-lab-9a

aws lambda invoke `
  --function-name workshop-api-lab9 `
  --region us-east-1 `
  response.json

Get-Content response.json
```

**Expected result:**

- The Lambda invoke command shows `FunctionError: Unhandled`.
- The response file includes an error similar to:

```text
name 'database_url' is not defined
```

**Notes:**

- If the SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

- If the function does not exist, I need to recreate the Lab 9A Lambda resources first.
- If the function returns a healthy response, I need to redeploy the broken Lab 9A version before continuing.

---

### Step 2: Start From the Alarm

**What I did:**

- Treated the CloudWatch alarm as the page/alert that started the investigation.
- Used the alert as evidence that Lambda errors crossed the configured threshold.

**Example alert scenario:**

```text
Subject: ALARM: "workshop-lab9-errors" in US East (N. Virginia)
Body: Threshold Crossed: 1 datapoint was greater than or equal to the threshold.
```

**Investigation questions:**

| Question | Why It Matters |
|---|---|
| When did the errors start? | Helps correlate the failure with a deployment or change |
| How many requests are failing? | Determines whether this is partial degradation or total outage |
| What is the exact error? | Points toward the root cause |
| Where did the code fail? | Identifies the file and line to fix |
| Did the system recover? | Proves whether the fix worked |

**Result:**

- The alarm told me that something was wrong.
- The next step was to use metrics and logs to determine what happened and why.

---

### Step 3: Check Error Metrics

**What I did:**

- Queried the Lambda `Errors` metric for the last 30 minutes.
- Looked for the timestamp where errors first appeared.

**Commands used:**

```powershell
$endTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$startTime = (Get-Date).AddMinutes(-30).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

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

- Earlier datapoints show no errors or `0.0`.
- Later datapoints show errors such as `1.0` or higher.

**What I learned:**

- The error metric showed when the problem started.
- This timestamp can be compared with deployment or code-change timing.

---

### Step 4: Compare Errors With Invocations

**What I did:**

- Queried the Lambda `Invocations` metric for the same time range.
- Compared invocation count with error count.

**Command used:**

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

**Expected result:**

- Invocations appear in the same period as the errors.
- If errors equal invocations, every request is failing.

**What I learned:**

| Observation | Meaning |
|---|---|
| Errors started suddenly | The issue likely came from a change or deploy |
| Errors match invocations | The function may have a 100% failure rate |
| Error spike is not gradual | This looks more like a code/config issue than a scaling issue |

**Result:**

- Metrics helped answer when the problem started and how severe it was.

---

### Step 5: Find the CloudWatch Log Group

**What I did:**

- Located the CloudWatch log group created for the Lambda function.
- Listed recent log streams to find the newest logs.

**List the log group:**

```powershell
aws logs describe-log-groups `
  --log-group-name-prefix "/aws/lambda/workshop-api-lab9" `
  --region us-east-1 `
  --query "logGroups[].logGroupName" `
  --output text
```

**Expected result:**

```text
/aws/lambda/workshop-api-lab9
```

**List recent log streams:**

```powershell
aws logs describe-log-streams `
  --log-group-name "/aws/lambda/workshop-api-lab9" `
  --region us-east-1 `
  --order-by LastEventTime `
  --descending `
  --limit 3 `
  --query "logStreams[].[logStreamName,lastEventTimestamp]" `
  --output table
```

**Expected result:**

- A table showing one or more recent log streams.
- The most recent stream contains the latest Lambda errors.

**Notes:**

- Lambda automatically writes logs to `/aws/lambda/<function-name>` after invocation.
- Every `print()`, `logger.info()`, and unhandled exception appears in CloudWatch Logs.

---

### Step 6: Read Raw Log Events

**What I did:**

- Stored the most recent log stream name in a PowerShell variable.
- Read the latest log events from that stream.
- Found the stack trace and root error.

**Get the most recent stream:**

```powershell
$stream = aws logs describe-log-streams `
  --log-group-name "/aws/lambda/workshop-api-lab9" `
  --region us-east-1 `
  --order-by LastEventTime `
  --descending `
  --limit 1 `
  --query "logStreams[0].logStreamName" `
  --output text

Write-Host "Most recent stream: $stream"
```

**Read log events:**

```powershell
aws logs get-log-events `
  --log-group-name "/aws/lambda/workshop-api-lab9" `
  --log-stream-name "$stream" `
  --region us-east-1 `
  --limit 20 `
  --query "events[].[timestamp,message]" `
  --output text
```

**Expected log evidence:**

```text
[INFO]  ... {"level": "INFO", "message": "Request received", ...}
[ERROR] NameError: name 'database_url' is not defined
Traceback (most recent call last):
  File "/var/task/handler.py", line 21, in lambda_handler
    connection_string = database_url
```

**Root cause evidence:**

| Evidence | Meaning |
|---|---|
| Error type | `NameError` |
| Error message | `name 'database_url' is not defined` |
| File | `handler.py` |
| Failed line | `connection_string = database_url` |
| Root cause | Code references a variable that was never defined |

**PowerShell note:**

Lambda log stream names can contain `$LATEST`. Because `$` has special meaning in PowerShell, I used the `$stream` variable instead of manually typing the stream name.

---

### Step 7: Query Errors With CloudWatch Logs Insights

**What I did:**

- Used CloudWatch Logs Insights to search for error entries across the full log group.
- Queried the last hour of logs.
- Returned the newest error entries first.

**Start the query:**

```powershell
$endTime = [int64](([datetime]::UtcNow) - ([datetime]'1970-01-01')).TotalSeconds
$startTime = $endTime - 3600

$queryId = aws logs start-query `
  --log-group-name "/aws/lambda/workshop-api-lab9" `
  --start-time $startTime `
  --end-time $endTime `
  --query-string "fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 10" `
  --region us-east-1 `
  --query "queryId" `
  --output text

Write-Host "Query started. ID: $queryId"
```

**Get the query results:**

```powershell
Start-Sleep 5
aws logs get-query-results `
  --query-id $queryId `
  --region us-east-1
```

**Expected result:**

- Query status is `Complete`.
- Results include error log entries.
- Results show `@timestamp` and `@message` fields.
- The message includes the `NameError` and stack trace.

**Query explanation:**

| Query Part | Purpose |
|---|---|
| `fields @timestamp, @message` | Shows the time and message |
| `filter @message like /ERROR/` | Returns only logs containing `ERROR` |
| `sort @timestamp desc` | Shows newest errors first |
| `limit 10` | Limits output to the 10 most recent errors |

**Result:**

- Logs Insights found the error quickly without manually scrolling through log streams.

---

### Step 8: Count Errors Per Minute

**What I did:**

- Ran a second Logs Insights query to group errors by minute.
- Used the query to visualize the error spike pattern.

**Command used:**

```powershell
$queryId2 = aws logs start-query `
  --log-group-name "/aws/lambda/workshop-api-lab9" `
  --start-time $startTime `
  --end-time $endTime `
  --query-string "filter @message like /ERROR/ | stats count(*) as error_count by bin(1m)" `
  --region us-east-1 `
  --query "queryId" `
  --output text

Start-Sleep 5

aws logs get-query-results `
  --query-id $queryId2 `
  --region us-east-1
```

**Expected result:**

- Results are grouped by one-minute intervals.
- The broken period shows error counts.
- Healthy periods show no error spike.

**What I learned:**

| Investigation Finding | Result |
|---|---|
| When | Errors started at the timestamp shown in CloudWatch metrics |
| Severity | Errors matched invocations, indicating a full failure during the broken period |
| What | `NameError` |
| Where | `handler.py` line with `database_url` |
| Why | Code referenced an undefined variable |

---

### Step 9: Fix the Lambda Code

**What I did:**

- Replaced the broken handler with a fixed version.
- Removed the undefined `database_url` reference.
- Kept the structured logging from the original healthy function.

**Updated `handler.py`:**

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Workshop API - FIXED version."""
    start_time = time.time()

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request received",
        "request_id": context.aws_request_id,
        "path": event.get("path", "/"),
        "method": event.get("httpMethod", "GET")
    }))

    # FIXED: removed the undefined database_url reference
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

**Result:**

- The code no longer references the undefined variable.
- The function should return a healthy response after redeployment.

---

### Step 10: Repackage and Redeploy the Fix

**What I did:**

- Repackaged the fixed `handler.py` file.
- Updated the Lambda function code.

**Commands used:**

```powershell
cd ~\Desktop\workshop-lab-9a

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

- I verified that I was inside the folder containing `handler.py` before packaging.
- If Lambda still returns the broken error, the ZIP may contain stale code.

---

### Step 11: Verify the Fixed Lambda Function

**What I did:**

- Invoked the Lambda function after redeploying the fixed code.
- Confirmed the function returned a healthy response.
- Generated several healthy invocations.

**Invoke once:**

```powershell
aws lambda invoke `
  --function-name workshop-api-lab9 `
  --region us-east-1 `
  response.json

Get-Content response.json
```

**Expected result:**

- The invoke output does not show `FunctionError`.
- The response file shows:

```json
{
  "statusCode": 200,
  "body": "{\"status\": \"healthy\", \"message\": \"Hello from the workshop API!\"}"
}
```

**Generate healthy invocations:**

```powershell
1..5 | ForEach-Object {
  aws lambda invoke `
    --function-name workshop-api-lab9 `
    --region us-east-1 `
    response.json | Out-Null

  Write-Host "Healthy invocation $_ done"
}
```

**Expected result:**

```text
Healthy invocation 1 done
Healthy invocation 2 done
Healthy invocation 3 done
Healthy invocation 4 done
Healthy invocation 5 done
```

---

### Step 12: Verify Recovery in Metrics

**What I did:**

- Waited 1–2 minutes for CloudWatch metrics to update.
- Checked the Lambda `Errors` metric again.
- Confirmed recent errors returned to zero.

**Commands used:**

```powershell
$endTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$startTime = (Get-Date).AddMinutes(-10).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")

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

- Recent datapoints show `0.0`, or no new error spike appears after the fix.

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
workshop-lab9-errors    OK
```

**Result:**

- The function recovered.
- The alarm returned to a healthy state after CloudWatch evaluated the new error-free period.

---

### Step 13: Console Checkpoint

**What I did:**

- Opened the CloudWatch console.
- Used Logs Insights to view the error pattern.
- Opened Lambda metrics to view the spike and recovery.

**CloudWatch Logs Insights check:**

```text
CloudWatch Console
Logs Insights
Select /aws/lambda/workshop-api-lab9
Run query: filter @message like /ERROR/ | stats count(*) by bin(5m)
```

**Expected result:**

- The chart shows errors concentrated during the broken period.
- The error count stops after the fix.

**CloudWatch Metrics check:**

```text
CloudWatch Console
Metrics
All metrics
Lambda
By Function Name
workshop-api-lab9
Errors
```

**Expected result:**

- The graph shows a spike when the broken code was deployed.
- The graph drops back down after the fixed code was deployed.

**What this proved:**

- The alarm detected the issue.
- Metrics showed when and how severe the issue was.
- Logs showed the exact root cause.
- The fix was verified through metrics and alarm recovery.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| Broken Lambda function confirmed | Passed |
| Alarm scenario reviewed | Passed |
| Lambda `Errors` metric checked | Passed |
| Lambda `Invocations` metric checked | Passed |
| Failure severity identified | Passed |
| CloudWatch log group found | Passed |
| Recent log streams listed | Passed |
| Raw log events reviewed | Passed |
| Stack trace identified | Passed |
| Root cause identified | Passed |
| Logs Insights error query ran | Passed |
| Logs Insights error-per-minute query ran | Passed |
| Lambda code fixed | Passed |
| Fixed code repackaged | Passed |
| Lambda code redeployed | Passed |
| Fixed Lambda invocation succeeded | Passed |
| Healthy invocations generated | Passed |
| Recent error metric returned to zero | Passed |
| Alarm returned to `OK` | Passed |
| CloudWatch console verification completed | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `ResourceNotFoundException` on log group | The function has not been invoked yet or the log group was deleted | Invoke the function first and verify the region is `us-east-1` |
| Log Insights query returns no results | Wrong time range, wrong log group, or logs have not arrived yet | Increase the query range to one hour and confirm the function was invoked recently |
| `$stream` variable is empty | The log stream lookup failed | Confirm the log group exists with `describe-log-groups` and that the function has recent invocations |
| `get-log-events` says stream not found | The stream name may contain `$LATEST`, which PowerShell can misread if typed manually | Store the stream name in `$stream` and pass `"$stream"` to the command |
| `MalformedQueryException` on `start-query` | Query syntax or timestamp values are invalid | Confirm `$startTime` and `$endTime` are Unix timestamps for Logs Insights queries |
| Query status remains `Running` | The query has not finished yet | Wait another 5 seconds and rerun `get-query-results` |
| Error spike is not visible in console | CloudWatch graph time range is too narrow | Set the graph to the last hour and use a one-minute period |
| Function still fails after fix | The ZIP file may contain stale or incorrect code | Delete `function.zip`, confirm `handler.py` is fixed, repackage, and redeploy |
| Alarm does not return to `OK` immediately | CloudWatch needs time and new healthy datapoints to evaluate | Invoke the fixed function several times and wait 1–2 minutes |
| AWS CLI authentication fails | SSO session expired or profile is not set | Run `aws sso login --profile <YOUR_PROFILE_NAME>` and reset `$env:AWS_PROFILE` |

## Cleanup

> If continuing to Lab 9C, keep the Lambda function and role. Lab 9C uses them.

If stopping here, clean up everything created for Labs 9A and 9B.

### Step 1: Delete the Lambda Function

```powershell
aws lambda delete-function `
  --function-name workshop-api-lab9 `
  --region us-east-1
```

### Step 2: Delete the CloudWatch Log Group

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-api-lab9 `
  --region us-east-1
```

### Step 3: Delete the CloudWatch Alarm

```powershell
aws cloudwatch delete-alarms `
  --alarm-names workshop-lab9-errors `
  --region us-east-1
```

### Step 4: Delete the SNS Topic

```powershell
aws sns delete-topic `
  --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts `
  --region us-east-1
```

### Step 5: Delete the Lambda IAM Role

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
| `workshop-api-lab9` Lambda function | Deleted |
| `/aws/lambda/workshop-api-lab9` log group | Deleted |
| `workshop-lab9-errors` CloudWatch alarm | Deleted |
| `workshop-lab9-alerts` SNS topic | Deleted |
| `workshop-lab9-lambda-role` IAM role | Deleted |
| Local `workshop-lab-9a` folder | Deleted |

## What I Learned

- A CloudWatch alarm tells me that something is wrong, but the investigation starts after the alarm fires.
- Metrics help answer when the problem started and how severe it is.
- Comparing `Errors` and `Invocations` can show whether failures are partial or total.
- CloudWatch Logs store the application messages and stack traces needed to identify root cause.
- Raw logs can show the exact error type, error message, file name, and failing line.
- CloudWatch Logs Insights is better than manual scrolling when there are many log entries.
- Logs Insights queries can filter for errors, sort by time, and group errors by minute.
- The root cause in this lab was a `NameError` caused by an undefined `database_url` variable.
- Fixing the code is not enough; I also need to verify recovery through metrics and alarm state.
- The alarm returning to `OK` proves the system has recovered from the error condition.
- Good incident response follows a repeatable workflow: detect, investigate, fix, and verify.
- Logs and metrics together reduce mean time to resolve.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/broken-function-confirmed.png` | Lambda invocation confirmed `FunctionError` |
| `screenshots/alarm-alert-context.png` | CloudWatch alarm or SNS alert reviewed |
| `screenshots/errors-metric-checked.png` | Lambda `Errors` metric checked |
| `screenshots/invocations-metric-checked.png` | Lambda `Invocations` metric checked |
| `screenshots/log-group-found.png` | Lambda CloudWatch log group found |
| `screenshots/recent-log-streams.png` | Recent Lambda log streams listed |
| `screenshots/raw-log-events.png` | Raw log events reviewed |
| `screenshots/nameerror-stack-trace.png` | Stack trace showed `database_url` NameError |
| `screenshots/log-insights-error-query.png` | Logs Insights query found error entries |
| `screenshots/log-insights-error-count.png` | Logs Insights grouped errors per minute |
| `screenshots/fixed-handler-created.png` | Fixed Lambda handler code created |
| `screenshots/fixed-code-deployed.png` | Fixed Lambda code deployed |
| `screenshots/fixed-invocation-success.png` | Fixed function returned successful response |
| `screenshots/healthy-invocations-generated.png` | Healthy invocations generated after fix |
| `screenshots/errors-returned-to-zero.png` | Recent error metric returned to zero |
| `screenshots/alarm-returned-ok.png` | CloudWatch alarm returned to `OK` |
| `screenshots/cloudwatch-error-spike-and-recovery.png` | CloudWatch graph showed error spike and recovery |
| `screenshots/cleanup-verified.png` | Cleanup verified if resources were removed |