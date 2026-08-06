# Lab 30: Monitor Chatbot Dependencies

## Lab Summary

In this lab, I expanded the chatbot observability work from Lab 10A by monitoring the external API dependency that the chatbot relies on.

The chatbot function was already deployed and writing structured JSON logs to CloudWatch Logs. In this lab, I created CloudWatch Logs metric filters that extracted values from those structured logs and converted them into custom CloudWatch metrics. One metric tracked external API latency, and another metric counted external API failures.

After publishing custom metrics, I built a CloudWatch dashboard with four panels: chatbot invocations, Lambda errors, external API latency, and external API errors. I then created a CloudWatch alarm that triggers when the external API latency exceeds 2 seconds.

Finally, I deployed a slow version of the chatbot that intentionally added a 3-second delay before calling the trivia API. The chatbot still worked, but the dashboard showed the dependency slowdown and the latency alarm entered the `ALARM` state. I then restored the healthy version and verified that the chatbot response time recovered.

This lab showed that an application can appear to be working while still providing a poor user experience because of a slow dependency. Monitoring dependency latency helps detect issues that normal Lambda error monitoring would miss.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 10B — Monitor Your Bot's Dependencies — Custom Metrics & Dashboards
- Session: 10 — Chatbot Observability
- Track: Solutions Architecture
- Difficulty: Intermediate
- Target certification: AWS Solutions Architect – Associate

## Objectives

- Set the active AWS CLI profile
- Verify the Lab 10A chatbot function exists and works
- Confirm the chatbot writes structured JSON logs
- Create a CloudWatch Logs metric filter for external API latency
- Create a CloudWatch Logs metric filter for external API errors
- Invoke the chatbot to generate new log entries
- Verify custom CloudWatch metrics are publishing
- Build a CloudWatch dashboard for chatbot health
- View chatbot invocations, Lambda errors, external API latency, and external API errors in one dashboard
- Create a CloudWatch alarm for slow external API latency
- Deploy a degraded chatbot version with simulated dependency latency
- Invoke the slow chatbot several times
- Confirm the dashboard shows a latency spike
- Confirm the CloudWatch alarm enters the `ALARM` state
- Restore the healthy chatbot version
- Verify latency returns to normal
- Keep resources for Lab 10C or clean up if stopping

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Lambda | Runs the trivia chatbot function |
| Amazon CloudWatch Logs | Stores structured JSON logs from the chatbot |
| CloudWatch Logs Metric Filters | Extracts custom metric values from structured logs |
| Amazon CloudWatch Metrics | Stores custom dependency latency and error metrics |
| Amazon CloudWatch Dashboard | Provides a single-pane operational view of chatbot health |
| Amazon CloudWatch Alarms | Alerts when external API latency exceeds a threshold |
| AWS IAM | Provides the Lambda execution role from Lab 10A |
| AWS CLI | Creates filters, metrics, dashboards, alarms, and updates Lambda code |
| PowerShell | Runs AWS CLI commands and packages Lambda code |
| VS Code | Edits JSON files and Lambda handler code |
| Open Trivia Database API | External trivia dependency called by the chatbot |

## Prerequisites

- Completed Lab 10A: Deploy Chatbot with Structured Logging
- `workshop-chatbot-lab10` Lambda function exists
- `workshop-lab10-lambda-role` IAM role exists
- `workshop-lab-10a` local folder exists on the Desktop
- `handler.py`, `function.zip`, and `payload.json` are available from Lab 10A
- AWS CLI installed
- AWS CLI authenticated with a working profile
- CloudWatch log group `/aws/lambda/workshop-chatbot-lab10` exists
- VS Code or another text editor installed
- PowerShell available on Windows

> If Lab 10A resources were cleaned up, recreate the Lambda role and redeploy `workshop-chatbot-lab10` using Lab 10A Steps 2–4 before starting this lab.

## Cost Notice

Estimated cost: `$0.00`

| Service | Cost Consideration |
|---|---|
| AWS Lambda | Expected to remain within the Lambda Always Free usage for this lab |
| CloudWatch Custom Metrics | Expected to remain within the free custom-metric allowance for this lab |
| CloudWatch Dashboard | Expected to remain within the free dashboard allowance |
| CloudWatch Alarm | Expected to remain within the free alarm allowance |
| CloudWatch Logs | Small amount of log data; expected cost is negligible |
| AWS IAM | Free |

> If continuing to Lab 10C, keep the chatbot function, metric filters, dashboard, and alarm. Lab 10C builds on them.

## Key Concepts

| Concept | Meaning |
|---|---|
| Dependency Monitoring | Monitoring the health and performance of services that an application depends on |
| External Dependency | A service outside the application, such as the Open Trivia Database API |
| Structured Logs | Logs written in a consistent JSON format so fields can be searched and extracted |
| Metric Filter | A CloudWatch Logs rule that matches log events and publishes extracted values as metrics |
| Custom Metric | A CloudWatch metric created by the user instead of automatically collected by AWS |
| Metric Namespace | A container for related CloudWatch metrics, such as `WorkshopChatbot` |
| API Latency | The time it takes for the external API call to complete |
| API Error Count | A metric counting failed external API calls |
| Dashboard | A visual CloudWatch page that combines multiple metrics in one view |
| Alarm Threshold | The value that causes an alarm to change state |
| `INSUFFICIENT_DATA` | Alarm state meaning CloudWatch does not have enough data to evaluate yet |
| `OK` | Alarm state meaning the metric is within the expected threshold |
| `ALARM` | Alarm state meaning the metric crossed the configured threshold |
| Latency Degradation | A condition where a dependency still responds but is too slow |
| Operational Excellence | AWS Well-Architected pillar focused on monitoring, understanding, and improving systems |

## Security Notes

| Topic | Explanation |
|---|---|
| No secrets in logs | Structured logs should not include credentials, tokens, API keys, or sensitive user data |
| Dependency visibility | Monitoring external services helps separate application issues from third-party service issues |
| Least privilege | The Lambda role only needs permissions required for logging and execution |
| Dashboard visibility | Dashboards may reveal architecture and service names, so access should be limited appropriately |
| Alarm hygiene | Alarms should use thresholds that are meaningful enough to avoid alert fatigue |
| Cleanup awareness | Metric filters, dashboards, alarms, and log groups should be cleaned up when no longer needed |

## Architecture Overview

```text
User / AWS CLI
      |
      | invokes
      v
Lambda Chatbot: workshop-chatbot-lab10
      |
      | calls external dependency
      v
Open Trivia Database API
      |
      v
Structured JSON Logs in CloudWatch Logs
      |
      +------------------------------+
      |                              |
      v                              v
Metric Filter: Latency        Metric Filter: Errors
      |                              |
      v                              v
Custom Metric:                Custom Metric:
ExternalAPILatency            ExternalAPIErrors
      |                              |
      +--------------+---------------+
                     |
                     v
          CloudWatch Dashboard
                     |
                     v
          CloudWatch Alarm: chatbot-api-slow
```

## Lab Steps

### Step 1: Set AWS Profile and Verify the Chatbot

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified that my Lab 10A chatbot function still existed.
- Invoked the chatbot using the existing payload file.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity

cd ~\Desktop\workshop-lab-10a
aws lambda invoke `
  --function-name workshop-chatbot-lab10 `
  --region us-east-1 `
  --cli-binary-format raw-in-base64-out `
  --payload file://payload.json `
  response.json

Get-Content response.json
```

**Expected result:**

- The AWS CLI identity command returned my active AWS account identity.
- The Lambda invoke command returned `StatusCode: 200`.
- The chatbot response contained a trivia question.

**Notes:**

- If the SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

- If the function was missing, I needed to recreate the Lab 10A Lambda role and function before continuing.

---

### Step 2: Create a Metric Filter for External API Latency

**What I did:**

- Created a metric transformation file.
- Configured CloudWatch Logs to extract `api_latency_ms` values from successful external API calls.
- Published those values as a custom metric named `ExternalAPILatency` in the `WorkshopChatbot` namespace.

**File created:**

```text
latency-transform.json
```

**File content:**

```json
[
    {
        "metricName": "ExternalAPILatency",
        "metricNamespace": "WorkshopChatbot",
        "metricValue": "$.api_latency_ms",
        "defaultValue": 0
    }
]
```

**Command used:**

```powershell
aws logs put-metric-filter `
  --log-group-name "/aws/lambda/workshop-chatbot-lab10" `
  --filter-name api-latency `
  --filter-pattern "{ $.event = ""api_call_success"" }" `
  --metric-transformations file://latency-transform.json `
  --region us-east-1
```

**Expected result:**

```text
No output
```

**Notes:**

- No output means the metric filter was created successfully.
- The PowerShell command uses double-double-quotes around `api_call_success` because of quote handling in PowerShell.
- The filter only matches new log events created after the filter exists.

---

### Step 3: Create a Metric Filter for External API Errors

**What I did:**

- Created a second metric transformation file.
- Configured CloudWatch Logs to count each external API failure as `1`.
- Published the count as a custom metric named `ExternalAPIErrors` in the `WorkshopChatbot` namespace.

**File created:**

```text
error-transform.json
```

**File content:**

```json
[
    {
        "metricName": "ExternalAPIErrors",
        "metricNamespace": "WorkshopChatbot",
        "metricValue": "1",
        "defaultValue": 0
    }
]
```

**Command used:**

```powershell
aws logs put-metric-filter `
  --log-group-name "/aws/lambda/workshop-chatbot-lab10" `
  --filter-name api-errors `
  --filter-pattern "{ $.event = ""api_call_failed"" }" `
  --metric-transformations file://error-transform.json `
  --region us-east-1
```

**Expected result:**

```text
No output
```

**Notes:**

- The latency filter extracts a numeric value from the log.
- The error filter publishes a fixed value of `1` each time the failure pattern appears.

---

### Step 4: Generate Metric Data

**What I did:**

- Invoked the chatbot five times.
- Generated new structured log entries after the metric filters were created.
- Allowed the metric filters to publish new custom metric datapoints.

**Command used:**

```powershell
1..5 | ForEach-Object {
  aws lambda invoke `
    --function-name workshop-chatbot-lab10 `
    --region us-east-1 `
    --cli-binary-format raw-in-base64-out `
    --payload file://payload.json `
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

**Notes:**

- Metric filters do not backfill old log entries.
- I waited 1–2 minutes for CloudWatch to publish the custom metrics.

---

### Step 5: Verify Custom Metrics Are Publishing

**What I did:**

- Queried the custom latency metric.
- Checked that CloudWatch returned datapoints for `ExternalAPILatency`.
- Used these values as the healthy baseline.

**Set metric time range:**

```powershell
$endTime = (Get-Date).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
$startTime = (Get-Date).AddMinutes(-10).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
```

**Check latency metric:**

```powershell
aws cloudwatch get-metric-statistics `
  --namespace "WorkshopChatbot" `
  --metric-name ExternalAPILatency `
  --start-time $startTime `
  --end-time $endTime `
  --period 60 `
  --statistics Average Maximum `
  --region us-east-1
```

**Expected result:**

- The output showed datapoints with `Average` and `Maximum` values.
- Healthy latency was expected to be far below the 2000 ms alarm threshold.

**Notes:**

- If datapoints were empty, I waited a few more minutes and invoked the chatbot again.
- To verify the metric filters existed, I used:

```powershell
aws logs describe-metric-filters `
  --log-group-name "/aws/lambda/workshop-chatbot-lab10" `
  --region us-east-1
```

---

### Step 6: Build a CloudWatch Dashboard

**What I did:**

- Created a CloudWatch dashboard definition.
- Added four widgets:
  - Chatbot invocations
  - Lambda errors
  - External API latency
  - External API errors
- Deployed the dashboard using the AWS CLI.

**File created:**

```text
dashboard.json
```

**File content:**

```json
{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Chatbot Invocations",
                "metrics": [
                    ["AWS/Lambda", "Invocations", "FunctionName", "workshop-chatbot-lab10"]
                ],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1"
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Lambda Errors",
                "metrics": [
                    ["AWS/Lambda", "Errors", "FunctionName", "workshop-chatbot-lab10"]
                ],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1"
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "External API Latency (ms)",
                "metrics": [
                    ["WorkshopChatbot", "ExternalAPILatency"]
                ],
                "period": 60,
                "stat": "Average",
                "region": "us-east-1",
                "annotations": {
                    "horizontal": [
                        {
                            "label": "Alarm threshold (2000ms)",
                            "value": 2000,
                            "color": "#d13212"
                        }
                    ]
                }
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "External API Errors",
                "metrics": [
                    ["WorkshopChatbot", "ExternalAPIErrors"]
                ],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1"
            }
        }
    ]
}
```

**Deploy command:**

```powershell
aws cloudwatch put-dashboard `
  --dashboard-name workshop-chatbot-dashboard `
  --dashboard-body file://dashboard.json `
  --region us-east-1
```

**Expected result:**

```json
{
    "DashboardValidationMessages": []
}
```

**Notes:**

- An empty `DashboardValidationMessages` array means the dashboard definition was accepted.

---

### Step 7: View the CloudWatch Dashboard

**What I did:**

- Opened the CloudWatch dashboard in the AWS Console.
- Verified all four panels were present.
- Confirmed the chatbot appeared healthy before simulating the slow dependency.

**Console path:**

```text
AWS Console
CloudWatch
Dashboards
workshop-chatbot-dashboard
```

**Expected dashboard panels:**

| Panel | Expected Healthy State |
|---|---|
| Chatbot Invocations | Shows recent chatbot invocations |
| Lambda Errors | Shows 0 or no error spike |
| External API Latency (ms) | Shows latency below the 2000 ms threshold line |
| External API Errors | Shows 0 or no error spike |

**Notes:**

- This dashboard represents the healthy baseline before the simulated degradation.

---

### Step 8: Create a Latency Alarm

**What I did:**

- Created a CloudWatch alarm named `chatbot-api-slow`.
- Configured it to watch the custom `ExternalAPILatency` metric.
- Set the alarm to trigger when the maximum latency exceeds 2000 ms in a 60-second period.

**Command used:**

```powershell
aws cloudwatch put-metric-alarm `
  --alarm-name chatbot-api-slow `
  --metric-name ExternalAPILatency `
  --namespace "WorkshopChatbot" `
  --statistic Maximum `
  --period 60 `
  --threshold 2000 `
  --comparison-operator GreaterThanThreshold `
  --evaluation-periods 1 `
  --alarm-description "External API latency exceeds 2 seconds" `
  --region us-east-1
```

**Verify alarm:**

```powershell
aws cloudwatch describe-alarms `
  --alarm-names chatbot-api-slow `
  --region us-east-1 `
  --query "MetricAlarms[0].[AlarmName,StateValue]" `
  --output text
```

**Expected result:**

```text
chatbot-api-slow    INSUFFICIENT_DATA
```

or:

```text
chatbot-api-slow    OK
```

**Notes:**

- `INSUFFICIENT_DATA` is normal if CloudWatch has not collected enough recent datapoints yet.
- The lab uses `Maximum` so even a single slow request can trigger the alarm.

---

### Step 9: Deploy a Slow Dependency Version

**What I did:**

- Replaced the healthy chatbot code with a slow version.
- Added a 3-second delay before the external API call.
- Repackaged and deployed the updated Lambda function.

**Updated file:**

```text
handler.py
```

**Key change:**

```python
# SIMULATED DEGRADATION: 3-second delay
time.sleep(3)
```

**Repackage and deploy:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force

aws lambda update-function-code `
  --function-name workshop-chatbot-lab10 `
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

- The chatbot should still return trivia questions.
- The problem being simulated is not a crash; it is slow dependency performance.

---

### Step 10: Invoke the Slow Version and Watch Latency Spike

**What I did:**

- Invoked the slow chatbot three times.
- Confirmed each call took noticeably longer.
- Checked the response metadata for higher `api_latency_ms` values.

**Command used:**

```powershell
1..3 | ForEach-Object {
  aws lambda invoke `
    --function-name workshop-chatbot-lab10 `
    --region us-east-1 `
    --cli-binary-format raw-in-base64-out `
    --payload file://payload.json `
    response.json

  Write-Host "Slow invocation $_ complete"
  Get-Content response.json | Write-Host
}
```

**Expected result:**

- Each invocation took around 3–4 seconds.
- The response metadata showed `api_latency_ms` values above 3000 ms.
- The chatbot still returned a response successfully.

**Notes:**

- This shows why dependency monitoring matters: the application did not crash, but the user experience became slower.

---

### Step 11: Watch the Alarm Fire

**What I did:**

- Waited 1–2 minutes for CloudWatch to process the new metrics.
- Checked the latency alarm state.
- Verified the dashboard showed a latency spike above the threshold.

**Command used:**

```powershell
aws cloudwatch describe-alarms `
  --alarm-names chatbot-api-slow `
  --region us-east-1 `
  --query "MetricAlarms[0].[AlarmName,StateValue,StateReason]" `
  --output text
```

**Expected result:**

```text
chatbot-api-slow    ALARM    Threshold Crossed
```

**Dashboard checkpoint:**

| Panel | Expected Degraded State |
|---|---|
| External API Latency | Spike above the 2000 ms threshold |
| Chatbot Invocations | Invocations still appear successful |
| Lambda Errors | Still 0 or no crash-related spike |
| External API Errors | Still 0 if the API responded slowly but successfully |

**What this proved:**

- The chatbot code was still working.
- Lambda was not failing.
- The dependency became slow.
- The custom latency metric and alarm detected the degraded dependency.

---

### Step 12: Restore the Healthy Version

**What I did:**

- Removed the `time.sleep(3)` delay from `handler.py`.
- Repackaged and redeployed the healthy chatbot code.
- Invoked the chatbot again to verify performance recovered.

**Repackage and deploy:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force

aws lambda update-function-code `
  --function-name workshop-chatbot-lab10 `
  --zip-file fileb://function.zip `
  --region us-east-1 `
  --query "LastUpdateStatus" `
  --output text
```

**Verify healthy response:**

```powershell
aws lambda invoke `
  --function-name workshop-chatbot-lab10 `
  --region us-east-1 `
  --cli-binary-format raw-in-base64-out `
  --payload file://payload.json `
  response.json

Get-Content response.json
```

**Expected result:**

- The response returned successfully.
- `api_latency_ms` returned closer to the healthy baseline.
- After 1–2 minutes, the alarm returned to `OK`.
- The dashboard latency graph dropped below the 2000 ms threshold.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| Lab 10A chatbot verified | Passed |
| Latency metric transformation file created | Passed |
| API latency metric filter created | Passed |
| Error metric transformation file created | Passed |
| API error metric filter created | Passed |
| New chatbot invocations generated | Passed |
| Custom latency metric published | Passed |
| Dashboard JSON created | Passed |
| CloudWatch dashboard deployed | Passed |
| Dashboard panels verified | Passed |
| Latency alarm created | Passed |
| Latency alarm verified | Passed |
| Slow chatbot version deployed | Passed |
| Slow invocations generated | Passed |
| Latency spike observed | Passed |
| Alarm entered `ALARM` state | Passed |
| Healthy version restored | Passed |
| Latency returned to normal | Passed |
| Alarm returned to `OK` | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| Custom metrics show empty datapoints | Metric filters only process new logs, not old log entries | Invoke the function several times after creating the metric filters and wait 1–2 minutes |
| Filter pattern error in PowerShell | Quotes in the JSON filter pattern were not escaped correctly | Use double-double-quotes around values inside the PowerShell filter pattern |
| `ResourceNotFoundException` for the log group | The function has not been invoked yet or the log group was deleted | Invoke the Lambda function once and confirm `/aws/lambda/workshop-chatbot-lab10` exists |
| Dashboard shows no data | Metrics have not published yet or the time range is too narrow | Wait 1–2 minutes and set dashboard time range to Last 1 hour |
| Alarm stays in `INSUFFICIENT_DATA` | Not enough recent datapoints exist for evaluation | Invoke the chatbot several times and wait for CloudWatch to evaluate |
| Alarm does not fire after slow version | Slow logs have not yet been converted into metric datapoints | Wait 1–2 minutes and confirm `ExternalAPILatency` has values above 2000 ms |
| Slow version still responds quickly | The Lambda ZIP may still contain the healthy code | Delete `function.zip`, confirm `handler.py` contains `time.sleep(3)`, repackage, and redeploy |
| Lambda update appears successful but behavior does not change | Lambda code update may still be in progress or old ZIP was uploaded | Check `LastUpdateStatus`, wait briefly, recreate the ZIP, and update function code again |
| AWS CLI authentication error | SSO token expired or profile was not set | Run `aws sso login --profile <YOUR_PROFILE_NAME>` and reset `$env:AWS_PROFILE` |

## Cleanup

> If continuing to Lab 10C, keep everything. Lab 10C uses the same function, dashboard, metric filters, and alarm.

If stopping here, remove the resources created in Labs 10A and 10B.

### Step 1: Delete the CloudWatch Alarm

```powershell
aws cloudwatch delete-alarms `
  --alarm-names chatbot-api-slow `
  --region us-east-1
```

### Step 2: Delete the CloudWatch Dashboard

```powershell
aws cloudwatch delete-dashboards `
  --dashboard-names workshop-chatbot-dashboard `
  --region us-east-1
```

### Step 3: Delete Metric Filters

```powershell
aws logs delete-metric-filter `
  --log-group-name "/aws/lambda/workshop-chatbot-lab10" `
  --filter-name api-latency `
  --region us-east-1

aws logs delete-metric-filter `
  --log-group-name "/aws/lambda/workshop-chatbot-lab10" `
  --filter-name api-errors `
  --region us-east-1
```

### Step 4: Delete the Lambda Function

```powershell
aws lambda delete-function `
  --function-name workshop-chatbot-lab10 `
  --region us-east-1
```

### Step 5: Delete the CloudWatch Log Group

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-chatbot-lab10 `
  --region us-east-1
```

### Step 6: Remove the Lambda IAM Role

```powershell
aws iam detach-role-policy `
  --role-name workshop-lab10-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role `
  --role-name workshop-lab10-lambda-role
```

### Step 7: Delete the Local Lab Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-10a
```

## Cleanup Verification

### Verify Alarm Is Deleted

```powershell
aws cloudwatch describe-alarms `
  --alarm-names chatbot-api-slow `
  --region us-east-1 `
  --query "MetricAlarms[].AlarmName"
```

**Expected result:**

```json
[]
```

### Verify Dashboard Is Deleted

```powershell
aws cloudwatch list-dashboards `
  --region us-east-1 `
  --query "DashboardEntries[?DashboardName=='workshop-chatbot-dashboard'].DashboardName"
```

**Expected result:**

```json
[]
```

### Verify Lambda Function Is Deleted

```powershell
aws lambda list-functions `
  --region us-east-1 `
  --query "Functions[?FunctionName=='workshop-chatbot-lab10'].FunctionName"
```

**Expected result:**

```json
[]
```

### Verify IAM Role Is Deleted

```powershell
aws iam get-role `
  --role-name workshop-lab10-lambda-role
```

**Expected result:**

```text
NoSuchEntity
```

**Expected cleanup result:**

| Resource | Expected State |
|---|---|
| `chatbot-api-slow` CloudWatch alarm | Deleted if stopping here |
| `workshop-chatbot-dashboard` CloudWatch dashboard | Deleted if stopping here |
| `api-latency` metric filter | Deleted if stopping here |
| `api-errors` metric filter | Deleted if stopping here |
| `workshop-chatbot-lab10` Lambda function | Deleted if stopping here |
| `/aws/lambda/workshop-chatbot-lab10` log group | Deleted if stopping here |
| `workshop-lab10-lambda-role` IAM role | Deleted if stopping here |
| Local `workshop-lab-10a` folder | Deleted if stopping here |

## What I Learned

- Structured JSON logs can be converted into CloudWatch custom metrics using metric filters.
- Metric filters only process new logs created after the filter exists.
- Dependency monitoring helps detect issues that normal Lambda error monitoring may miss.
- A Lambda function can return successful responses while still providing a poor user experience because of high latency.
- Custom metrics should be organized into meaningful namespaces.
- CloudWatch dashboards help combine traffic, error, latency, and dependency-health signals into one operational view.
- A latency alarm can detect a degraded external dependency before the application fully fails.
- Using `Maximum` for latency alarms is useful because a single slow request can indicate a user-impacting problem.
- `INSUFFICIENT_DATA` is normal when a new alarm does not yet have enough datapoints.
- Restoring the healthy version should cause the dashboard and alarm to recover after CloudWatch receives new normal datapoints.
- Monitoring should include both internal application health and external dependency health.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/chatbot-verified-working.png` | Lab 10A chatbot invoked successfully |
| `screenshots/latency-transform-created.png` | Latency metric transformation file created |
| `screenshots/api-latency-filter-created.png` | CloudWatch metric filter for API latency created |
| `screenshots/error-transform-created.png` | Error metric transformation file created |
| `screenshots/api-errors-filter-created.png` | CloudWatch metric filter for API errors created |
| `screenshots/metric-data-generated.png` | Chatbot invoked to generate new metric data |
| `screenshots/custom-latency-metric.png` | Custom `ExternalAPILatency` metric verified |
| `screenshots/dashboard-json-created.png` | Dashboard definition file created |
| `screenshots/dashboard-deployed.png` | CloudWatch dashboard deployed successfully |
| `screenshots/dashboard-healthy-view.png` | Dashboard showed healthy baseline |
| `screenshots/latency-alarm-created.png` | `chatbot-api-slow` alarm created |
| `screenshots/alarm-initial-state.png` | Alarm showed `INSUFFICIENT_DATA` or `OK` before degradation |
| `screenshots/slow-handler-created.png` | Slow chatbot version created with `time.sleep(3)` |
| `screenshots/slow-version-deployed.png` | Slow Lambda version deployed |
| `screenshots/slow-invocations-generated.png` | Slow chatbot invocations completed |
| `screenshots/latency-spike-response.png` | Response metadata showed high API latency |
| `screenshots/dashboard-latency-spike.png` | Dashboard showed latency spike above threshold |
| `screenshots/alarm-state-alarm.png` | `chatbot-api-slow` entered `ALARM` state |
| `screenshots/healthy-version-restored.png` | Healthy chatbot code redeployed |
| `screenshots/latency-recovered.png` | Latency returned closer to baseline |
| `screenshots/alarm-returned-ok-console.png` | Alarm returned to `OK` after recovery |
| `screenshots/alarm-returned-ok-cli.png` | Alarm returned to `OK` after recovery |