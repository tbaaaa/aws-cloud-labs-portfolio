# Lab 11C: Production AI — Token Budgets, Cost Monitoring & Guardrails

## Lab Summary

This lab took the chatbot from "working with a personality" (Lab 11B) to "production-ready" by adding the controls that actually matter once an AI application is live: cost limits, input validation, monitoring, and alerting.

I added input validation to `handler.py` that rejects any prompt over 500 characters before it ever reaches Bedrock, so an oversized or malicious prompt costs nothing. I then extracted token usage from the Lambda's structured logs into a CloudWatch custom metric using a metric filter, built a four-panel AI operations dashboard (invocations, latency, tokens per request, cumulative tokens), and created an alarm that fires if a single request's token usage spikes above 500 — a second line of defense in case something ever got past the input validation. Finally, I tested the full stack end-to-end with a normal request and confirmed the alarm was reporting a healthy state.

The core lesson from this lab: building an AI feature is the easy part. Operating it responsibly — with cost controls, observability, and defense in depth — is what separates a demo from something you could actually run in production.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 11C — Production AI: Token Budgets, Cost Monitoring & Guardrails
- Session: 11 — AI Engineering
- Track: AI Engineering
- Difficulty: Advanced
- Target certification: AWS Certified AI Practitioner

## Objectives

- Set the active AWS CLI profile and return to the Lab 11A project folder
- Add input validation to `handler.py` that rejects prompts longer than 500 characters before calling Bedrock
- Re-deploy the updated Lambda function
- Test the token budget with a normal short message and confirm it passes validation
- Test the token budget with a deliberately long message and confirm it's rejected with a `400` response before reaching Bedrock
- Create a metric transformation file (`token-transform.json`) defining a `TotalTokensUsed` custom metric
- Create a CloudWatch Logs metric filter that extracts token usage from `bedrock_call_success` log events
- Invoke the chatbot multiple times to generate metric data
- Build a CloudWatch dashboard definition (`dashboard.json`) with panels for invocations, latency, tokens per request, and cumulative tokens
- Create the AI operations dashboard from that definition
- Verify all four dashboard panels render correctly in the CloudWatch console
- Create a CloudWatch alarm that fires if any single request's token usage exceeds 500
- Test the full stack end-to-end with a normal request and confirm token count and latency are reasonable
- Confirm the token spike alarm is in an `OK` (or `INSUFFICIENT_DATA`) state
- Understand defense in depth for AI applications: validation (prevent) + monitoring (detect) + alarm (alert)
- Clean up the alarm, dashboard, metric filter, Lambda function, IAM role, and log group when finished

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Lambda | Runs the chatbot code, including the new input validation logic |
| Amazon Bedrock (Nova Micro) | Generates AI responses; only invoked for requests that pass input validation |
| Amazon CloudWatch Logs (Metric Filters) | Extracts `TotalTokensUsed` as a custom metric from structured JSON logs |
| Amazon CloudWatch (Dashboards) | Visualizes invocations, latency, tokens per request, and cumulative tokens in one view |
| Amazon CloudWatch (Alarms) | Alerts on a token usage spike per request as a second layer of defense |
| AWS IAM | Provides Lambda execution permissions and Bedrock invoke access (reused from Lab 11A) |
| AWS CLI | Invokes the Lambda function, creates the metric filter, dashboard, and alarm |
| PowerShell | Runs AWS CLI commands, packages Lambda code, loops through repeated invocations |
| Python | Implements the input validation block in `handler.py` |
| VS Code | Edits `handler.py`, `payload.json`, `token-transform.json`, and `dashboard.json` |

## Prerequisites

- Completed Lab 11A: Bedrock Chatbot (and ideally Lab 11B: Prompt Engineering)
- `workshop-ai-chatbot-lab11` Lambda function still exists
- `workshop-lab11-lambda-role` IAM role still exists
- AWS CLI installed and authenticated
- PowerShell available on Windows
- VS Code or another text editor installed

> If Lab 11A resources were cleaned up, follow Lab 11A Steps 3–5 to recreate the role and function before continuing.

## Cost Notice

Estimated cost: `under $0.03`

| Service | Cost Consideration |
|---|---|
| AWS Lambda | Expected to remain within Always Free usage for this lab |
| Amazon Bedrock (Nova Micro) | Small per-invocation inference cost across test prompts (~$0.01–0.02 total) |
| CloudWatch Custom Metrics | Always Free (up to 10 metrics; this lab uses 1) |
| CloudWatch Dashboard | Always Free (up to 3 dashboards; this lab uses 1) |

> Complete cleanup after this lab, since this is the last lab in Session 11.

## Key Concepts

| Concept | Meaning |
|---|---|
| Token Budget | A code-enforced limit that rejects a request before calling Bedrock if it would use too many tokens |
| Input Validation | Checking user input BEFORE calling the AI model; an oversized, empty, or forbidden prompt is rejected immediately with no Bedrock call and no cost |
| Cost Estimation | Calculating approximate cost from token counts (Nova Micro: roughly $0.035/million input tokens, $0.14/million output tokens) |
| AI Operations Dashboard | A CloudWatch dashboard purpose-built for AI workloads — standard Lambda metrics plus tokens per request and cumulative token usage |
| Defense in Depth (for AI) | Layering validation (prevents bad requests), monitoring (detects anomalies), and alarms (alerts on them) rather than relying on any single control |

## Security Notes

| Topic | Explanation |
|---|---|
| Input validation as a security control | Rejecting oversized prompts before they reach Bedrock is a first line of defense against prompt injection and cost-based abuse, not just a cost optimization |
| Defense in depth | Validation (Step 2) prevents most abuse; the token spike alarm (Step 6) is an independent second layer in case something bypasses validation |
| Least privilege preserved | No new IAM permissions were required; the Lambda role's existing Bedrock invoke permission from Lab 11A was reused as-is |
| No secrets in configuration files | `token-transform.json` and `dashboard.json` contain only metric/dashboard definitions, never credentials or sensitive data |
| Observability supports incident response | The token usage metric and alarm give an early signal of abuse or misconfiguration, similar in spirit to the CloudWatch alerting built in Lab 10B/10C |

## Architecture Overview

```text
User / AWS CLI
      |
      | invokes with message
      v
Lambda Function: workshop-ai-chatbot-lab11
      |
      v
Input Validation (MAX_INPUT_CHARS = 500)
      |
      +----------------------+----------------------+
      |                                              |
   Too long                                    Within limit
      |                                              |
      v                                              v
400 response                              Amazon Bedrock (Nova Micro)
(no Bedrock call, no cost)                            |
                                                       v
                                          Structured JSON log: bedrock_call_success
                                                       |
                                                       v
                                   CloudWatch Logs Metric Filter (token-usage)
                                                       |
                                                       v
                                     Custom Metric: WorkshopAIChatbot/TotalTokensUsed
                                                       |
                              +------------------------+------------------------+
                              |                                                 |
                              v                                                 v
                 AI Operations Dashboard                          Token Spike Alarm
        (invocations, latency, tokens/req,                    (fires if any request
              cumulative tokens)                                  exceeds 500 tokens)
```

## Lab Steps

### Step 1: Set AWS Profile and Return to Project Folder

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Navigated back to the chatbot project folder used in Labs 11A/11B.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~\Desktop\workshop-lab-11a
```

**Expected result:**

- PowerShell showed the `workshop-lab-11a` folder path, ready for edits.

**Notes:**

- No invocation needed yet — this step is just positioning for the changes in Step 2.

---

### Step 2: Add Input Validation and a Token Budget

**What I did:**

- Opened `handler.py` and added a validation block immediately after the `user_message = ...` line and before the Bedrock call.
- The block checks `len(user_message)` against a `MAX_INPUT_CHARS = 500` limit (roughly 125 tokens).
- If the message is too long, it logs a structured `WARNING` event (`input_rejected`) and immediately returns a `400` response — without ever calling Bedrock.

**Code added:**

```python
# Input validation — reject prompts that are too long
MAX_INPUT_CHARS = 500  # ~125 tokens max input
if len(user_message) > MAX_INPUT_CHARS:
    logger.warning(json.dumps({
        "level": "WARNING",
        "event": "input_rejected",
        "request_id": request_id,
        "reason": "prompt_too_long",
        "input_length": len(user_message),
        "max_allowed": MAX_INPUT_CHARS
    }))
    return {
        "statusCode": 400,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps({
            "error": "Your message is too long. Please keep it under 500 characters.",
            "input_length": len(user_message),
            "max_allowed": MAX_INPUT_CHARS
        })
    }
```

**Deploy commands:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**Expected result:**

- Deploy command returned `InProgress` or `Successful`.

**What this means:**

- Rejecting an oversized prompt before calling Bedrock means zero tokens consumed and zero cost for that request — the cheapest AI call is the one you never make.

**Notes:**

- Placed the validation block strictly before the Bedrock call so it can't be bypassed by any code path that reaches `bedrock.converse()`.

---

### Step 3: Test the Token Budget

**What I did:**

- Sent a normal, short message through `payload.json` (`"What is S3?"`) to confirm it still passes validation and works normally.
- Sent a deliberately long, multi-sentence prompt (well over 500 characters) to confirm it gets rejected.

**Commands used:**

```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**Expected result:**

- The short message returned a normal AI response.
- The long message returned a `400` status with a body like:

```json
{"error": "Your message is too long. Please keep it under 500 characters.", "input_length": 5XX, "max_allowed": 500}
```

**What this means:**

- The validation rejected the oversized request before it reached Bedrock — zero tokens consumed, zero cost incurred. In production, this is what stops someone from pasting an entire document into the chatbot and burning through the token budget.

**Notes:**

- (record the actual `input_length` returned for the long prompt once tested)

---

### Step 4: Create a Token Usage Metric Filter

**What I did:**

- Created `token-transform.json`, defining a `TotalTokensUsed` custom metric in the `WorkshopAIChatbot` namespace, sourced from the `$.total_tokens` field in the structured logs.
- Created a CloudWatch Logs metric filter (`token-usage`) on `/aws/lambda/workshop-ai-chatbot-lab11` that matches log events where `$.event = "bedrock_call_success"` and applies the metric transformation.
- Invoked the chatbot five times in a loop to generate metric data, then waited for the filter to process the new logs.

**Files created:**

`token-transform.json`:

```json
[
    {
        "metricName": "TotalTokensUsed",
        "metricNamespace": "WorkshopAIChatbot",
        "metricValue": "$.total_tokens",
        "defaultValue": 0
    }
]
```

**Commands used:**

```powershell
aws logs put-metric-filter --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-name token-usage --filter-pattern "{ $.event = ""bedrock_call_success"" }" --metric-transformations file://token-transform.json --region us-east-1
```

```powershell
1..5 | ForEach-Object { aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json | Out-Null; Write-Host "Invoke $_ done" }
```

**Expected result:**

- `put-metric-filter` returned no output, indicating success.
- All 5 invokes printed `Invoke N done`.

**What this means:**

- Every successful Bedrock call now automatically contributes to a queryable CloudWatch metric, without any extra code in the Lambda function itself — the metric is derived entirely from the existing structured logs.

**Notes:**

- Waited roughly 2 minutes after invoking before expecting the metric data to show up, since metric filters only process new log events going forward.

---

### Step 5: Create an AI Operations Dashboard

**What I did:**

- Created `dashboard.json` defining four widgets: AI chatbot invocations (Lambda `Invocations`, Sum), Bedrock latency (Lambda `Duration`, Average), total tokens per request (`TotalTokensUsed`, Average), and cumulative total tokens (`TotalTokensUsed`, Sum over a 5-minute period).
- Created the CloudWatch dashboard from that definition.
- Verified in the CloudWatch console that all four panels rendered with data.

**File created:** `dashboard.json` (four-widget layout — invocations, latency, tokens/request, cumulative tokens).

**Commands used:**

```powershell
aws cloudwatch put-dashboard --dashboard-name workshop-ai-chatbot-dashboard --dashboard-body file://dashboard.json --region us-east-1
```

**Expected result:**

- Response included `"DashboardValidationMessages": []`, confirming a valid dashboard definition.

**Console checkpoint:**

- Opened CloudWatch → Dashboards → `workshop-ai-chatbot-dashboard` and confirmed all four panels were present: invocations, latency, tokens per request, and cumulative tokens.

**What this means:**

- This gives a single view of chatbot health that combines standard Lambda operational metrics with AI-specific cost/usage metrics — exactly what's needed to answer "is this healthy?" and "what is this costing me?" at a glance.

**Notes:**

- (note whether panels showed data immediately or needed the full 2-minute wait before populating)

---

### Step 6: Create a Token Spike Alarm

**What I did:**

- Created a CloudWatch alarm (`ai-chatbot-token-spike`) on the `TotalTokensUsed` metric that fires if the Maximum over a 60-second period exceeds 500 tokens in a single evaluation period.

**Commands used:**

```powershell
aws cloudwatch put-metric-alarm --alarm-name ai-chatbot-token-spike --metric-name TotalTokensUsed --namespace "WorkshopAIChatbot" --statistic Maximum --period 60 --threshold 500 --comparison-operator GreaterThanThreshold --evaluation-periods 1 --alarm-description "AI chatbot token usage spike - possible long prompt or injection" --region us-east-1
```

**Expected result:**

- No output, indicating the alarm was created successfully.

**What this means:**

- Input validation (Step 2) should prevent any single request from exceeding this threshold. If a request ever bypassed that check, this alarm is the independent second layer that catches it — defense in depth rather than relying on one control alone.

**Notes:**

- (note the alarm's initial state after creation — likely `INSUFFICIENT_DATA` until enough data points accumulate)

---

### Step 7: Test the Full Stack End-to-End

**What I did:**

- Sent a normal request (`"Explain VPC in simple terms"`) through the complete pipeline and confirmed the AI responded correctly, with a reasonable token count (under 200) and latency visible in the response metadata.
- Checked the token spike alarm's state to confirm the monitoring stack was healthy.

**Commands used:**

```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

```powershell
aws cloudwatch describe-alarms --alarm-names ai-chatbot-token-spike --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue]" --output text
```

**Expected result:**

- The VPC question returned a correct, reasonably short answer with acceptable latency.
- The alarm state command returned `ai-chatbot-token-spike    OK` (or `INSUFFICIENT_DATA` if not enough data points had accumulated yet).

**What this means:**

- This confirms the full production stack — validation, Bedrock call, metric filter, dashboard, and alarm — works together end-to-end, not just in isolation.

**Notes:**

- (record actual token count and latency observed, and the alarm's actual state at the time of testing)

---

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| Metric filter not producing data | Filters only process new logs going forward, not historical ones | Invoke the function several more times and wait about 2 minutes |
| Filter pattern error on Windows | PowerShell handles quotes differently than macOS/Linux | Use doubled double-quotes: `"{ $.event = ""bedrock_call_success"" }"` |
| Dashboard shows "No data" | Metrics haven't published yet | Wait 2 minutes after invocations; set the dashboard's time range to "Last 1 hour" |
| Input validation not working | Old code still deployed | Verify `handler.py` has the `MAX_INPUT_CHARS` block, re-zip with `Compress-Archive`, and re-deploy |
| Alarm stuck in `INSUFFICIENT_DATA` | Not enough data points yet | Invoke several more times and wait for the metric filter to process the new logs |
| Deploy succeeds but validation doesn't trigger | Stale `function.zip` was deployed instead of the current `handler.py` | Delete `function.zip`, recreate it from the current file, and re-run `update-function-code` |

## Cleanup

> This is the last lab in Session 11 — clean up everything unless continuing directly into Session 12.

### Step 1: Delete the Alarm, Dashboard, Metric Filter, Lambda Function, Log Group, and Role

```powershell
aws cloudwatch delete-alarms --alarm-names ai-chatbot-token-spike --region us-east-1
aws cloudwatch delete-dashboards --dashboard-names workshop-ai-chatbot-dashboard --region us-east-1
aws logs delete-metric-filter --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-name token-usage --region us-east-1
aws lambda delete-function --function-name workshop-ai-chatbot-lab11 --region us-east-1
aws logs delete-log-group --log-group-name /aws/lambda/workshop-ai-chatbot-lab11 --region us-east-1
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name bedrock-invoke
aws iam detach-role-policy --role-name workshop-lab11-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab11-lambda-role
```

### Step 2: Delete the Local Project Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-11a
```

## Cleanup Verification

### Verify the Alarm Is Deleted

```powershell
aws cloudwatch describe-alarms `
  --alarm-names ai-chatbot-token-spike `
  --region us-east-1 `
  --query "MetricAlarms[].AlarmName"
```

**Expected result:**

```json
[]
```

### Verify the Dashboard Is Deleted

```powershell
aws cloudwatch list-dashboards `
  --region us-east-1 `
  --query "DashboardEntries[?DashboardName=='workshop-ai-chatbot-dashboard'].DashboardName"
```

**Expected result:**

```json
[]
```

### Verify the Metric Filter Is Deleted

```powershell
aws logs describe-metric-filters `
  --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" `
  --region us-east-1 `
  --query "metricFilters[].filterName"
```

**Expected result:**

```json
[]
```

*(This will naturally also return empty once the log group itself is deleted.)*

### Verify Lambda Function Is Deleted

```powershell
aws lambda list-functions `
  --region us-east-1 `
  --query "Functions[?FunctionName=='workshop-ai-chatbot-lab11'].FunctionName"
```

**Expected result:**

```json
[]
```

### Verify Lambda Role Is Deleted

```powershell
aws iam get-role --role-name workshop-lab11-lambda-role
```

**Expected result:**

```text
NoSuchEntity
```

**Expected cleanup result:**

| Resource | Expected State |
|---|---|
| `ai-chatbot-token-spike` alarm | Deleted |
| `workshop-ai-chatbot-dashboard` dashboard | Deleted |
| `token-usage` metric filter | Deleted |
| `workshop-ai-chatbot-lab11` Lambda function | Deleted |
| `/aws/lambda/workshop-ai-chatbot-lab11` log group | Deleted |
| `bedrock-invoke` inline role policy | Deleted |
| `workshop-lab11-lambda-role` IAM role | Deleted |
| Local `workshop-lab-11a` folder | Deleted |

## What I Learned

- Validating input before calling the AI model is the cheapest possible cost control — a rejected request costs nothing, since Bedrock is never invoked.
- Token usage can be tracked automatically from structured logs using a CloudWatch metric filter, with no extra instrumentation code needed in the Lambda function itself.
- An AI operations dashboard should combine standard compute metrics (invocations, latency) with AI-specific ones (tokens per request, cumulative tokens) to give a complete health-and-cost picture in one place.
- Defense in depth applies to AI just like any other system: input validation prevents most abuse, monitoring detects what gets through, and alarms make sure detection turns into action.
- Hard limits (`maxTokens`) and soft limits (input validation) serve different purposes and are both necessary — one bounds the model's output, the other bounds what's allowed to reach the model at all.
- In a real production system, this stack would still need rate limiting per user, daily budget caps, and content filtering (e.g., Bedrock Guardrails) to be considered complete.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/input-validation-added-handler.png` | `handler.py` updated with the `MAX_INPUT_CHARS` validation block |
| `screenshots/short-message-passes-validation.png` | Normal short message processed successfully after the validation change |
| `screenshots/long-message-rejected-400.png` | Oversized prompt rejected with a `400` response before reaching Bedrock |
| `screenshots/token-transform-json-created.png` | `token-transform.json` metric transformation file |
| `screenshots/metric-filter-created.png` | Successful `put-metric-filter` command output (or confirmation in the console) |
| `screenshots/repeated-invokes-for-metric-data.png` | Output of the 5x invoke loop used to generate metric data |
| `screenshots/dashboard-json-created.png` | `dashboard.json` four-widget definition file |
| `screenshots/dashboard-created-console.png` | CloudWatch console showing the `workshop-ai-chatbot-dashboard` with all four panels populated |
| `screenshots/token-spike-alarm-created.png` | Successful `put-metric-alarm` command for `ai-chatbot-token-spike` |
| `screenshots/full-stack-test-response.png` | End-to-end test response for the VPC question, showing token count and latency |
| `screenshots/alarm-state-ok.png` | `describe-alarms` output showing the token spike alarm in `OK` (or `INSUFFICIENT_DATA`) state |