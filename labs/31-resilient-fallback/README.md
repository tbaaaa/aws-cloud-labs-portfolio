# Lab 31: Resilient Fallback for Chatbot

## Lab Summary

In this lab, I improved the trivia chatbot from Lab 10A and Lab 10B by adding a resilient fallback mode.

Previously, the chatbot depended on the external Open Trivia Database API. In Lab 10B, I monitored that dependency using custom CloudWatch metrics, dashboards, and alarms. In this lab, I went one step further by making the bot tolerate dependency problems instead of only detecting them.

I created an AWS Systems Manager Parameter Store value that controls whether the chatbot runs in `live` mode or `fallback` mode. In live mode, the bot calls the external trivia API and returns a random question. In fallback mode, the bot skips the external API call and returns a cached trivia response immediately.

This showed how graceful degradation can keep an application responsive even when a dependency is slow, degraded, or unavailable.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 10C — Make the Bot Self-Healing — Fallback & Graceful Degradation
- Session: 10 — Chatbot Observability
- Track: Solutions Architecture
- Difficulty: Advanced
- Target certification: AWS Solutions Architect – Associate

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Reuse the chatbot project folder from Lab 10A and Lab 10B
- Create an SSM Parameter Store value to control chatbot mode
- Give the Lambda execution role permission to read SSM parameters
- Update the chatbot Lambda handler to support live and fallback modes
- Deploy the resilient Lambda version
- Test the chatbot in live mode
- Switch the chatbot to fallback mode without redeploying code
- Test the chatbot in fallback mode
- Generate fallback invocation data for CloudWatch
- Switch the chatbot back to live mode
- Verify the chatbot returns live random trivia questions again
- Review the CloudWatch dashboard for live, fallback, and error states
- Query CloudWatch Logs for `fallback_used` events
- Understand graceful degradation and the circuit breaker pattern
- Clean up Session 10 resources when finished

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Lambda | Runs the trivia chatbot code |
| AWS IAM | Provides Lambda execution permissions and SSM read access |
| AWS Systems Manager Parameter Store | Stores the chatbot mode switch as `live` or `fallback` |
| Amazon CloudWatch Logs | Stores structured chatbot logs |
| Amazon CloudWatch Metrics | Stores custom metrics from Lab 10B |
| Amazon CloudWatch Dashboards | Displays live, fallback, latency, and error behavior |
| Amazon CloudWatch Alarms | Alerts when the external API dependency becomes slow |
| AWS CLI | Creates parameters, updates Lambda code, invokes the bot, and queries logs |
| PowerShell | Runs AWS CLI commands and packages Lambda code |
| Python | Implements the chatbot Lambda handler |
| VS Code | Edits handler and policy files |
| Open Trivia Database API | External trivia dependency used by the chatbot |

## Prerequisites

- Completed Lab 10A: Deploy Chatbot with Structured Logging
- Completed Lab 10B: Monitor Chatbot Dependencies
- `workshop-chatbot-lab10` Lambda function still exists
- `workshop-lab10-lambda-role` IAM role still exists
- CloudWatch metric filters from Lab 10B still exist
- CloudWatch dashboard from Lab 10B still exists
- AWS CLI installed and authenticated
- PowerShell available on Windows
- VS Code or another text editor installed

> If Lab 10A or Lab 10B resources were cleaned up, the Lambda function, Lambda role, metric filters, dashboard, and alarm need to be recreated before continuing.

## Cost Notice

Estimated cost: `$0.00`

| Service | Cost Consideration |
|---|---|
| AWS Lambda | Expected to remain within Always Free usage for this lab |
| AWS Systems Manager Parameter Store | Standard parameters are expected to remain within free usage |
| Amazon CloudWatch | Small metric, log, dashboard, and alarm usage for a short lab window |
| AWS IAM | Free |

> Complete cleanup after Session 10 if the resources are no longer needed.

## Key Concepts

| Concept | Meaning |
|---|---|
| Graceful Degradation | Keeping an application functional, even with reduced capability, when a dependency fails or becomes slow |
| Fallback Response | A pre-prepared response returned when the primary data source is unavailable or intentionally bypassed |
| SSM Parameter Store | AWS service used to store configuration values that applications can read at runtime |
| Runtime Configuration | Application behavior controlled by configuration instead of redeploying code |
| Circuit Breaker Pattern | A resilience pattern where calls to an unhealthy dependency are stopped temporarily |
| Live Mode | The chatbot calls the real external trivia API |
| Fallback Mode | The chatbot skips the external API and returns a cached trivia response |
| Dependency Failure | When an external service becomes unavailable, slow, or unreliable |
| Structured Logging | Writing logs in JSON format so they can be searched and filtered more easily |
| Observability | Understanding application behavior through logs, metrics, dashboards, and alarms |
| Resilience | The ability of a system to continue operating during failures or degraded conditions |

## Security Notes

| Topic | Explanation |
|---|---|
| Least privilege | The Lambda role only receives `ssm:GetParameter` for the `/workshop/chatbot/*` parameter path |
| No code redeploy needed | The mode switch is controlled through SSM Parameter Store instead of editing application code each time |
| Runtime control | During an incident, operators can switch to fallback quickly while investigating the external API |
| No secrets in parameter | The parameter stores only a simple mode value, not credentials or sensitive secrets |
| Observable fallback | The Lambda writes a `fallback_used` log event so operators can see when fallback mode was active |
| Safer user experience | Users receive a useful fallback response instead of waiting on a slow dependency or seeing an error |

## Architecture Overview

```text
User / AWS CLI
      |
      | invokes
      v
Lambda Function: workshop-chatbot-lab10
      |
      | reads mode
      v
SSM Parameter: /workshop/chatbot/mode
      |
      +-------------------------------+
      |                               |
      v                               v
mode = live                    mode = fallback
      |                               |
      v                               v
Call Open Trivia DB API        Return cached trivia response
      |                               |
      +---------------+---------------+
                      |
                      v
             Structured CloudWatch Logs
                      |
                      v
          CloudWatch Metrics and Dashboard
```

## Lab Steps

### Step 1: Set AWS Profile and Open Project Folder

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Navigated back to the chatbot project folder from Lab 10A.
- Verified I was in the correct folder.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~\Desktop\workshop-lab-10a
pwd
```

**Recommended identity check:**

```powershell
aws sts get-caller-identity
```

**Expected result:**

- PowerShell showed the `workshop-lab-10a` folder path.
- AWS CLI returned the active AWS account identity.

**Notes:**

- If the SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create the Fallback Mode Parameter in SSM

**What I did:**

- Created an SSM Parameter Store value named `/workshop/chatbot/mode`.
- Set the initial value to `live`.

**Command used:**

```powershell
aws ssm put-parameter `
  --name "/workshop/chatbot/mode" `
  --type String `
  --value "live" `
  --region us-east-1
```

**Expected result:**

```json
{
  "Version": 1,
  "Tier": "Standard"
}
```

**What this means:**

- The chatbot starts in live mode.
- In live mode, the Lambda function calls the external trivia API normally.
- Later, changing this value to `fallback` changes the bot behavior without redeploying code.

---

### Step 3: Give the Lambda Role Permission to Read SSM

**What I did:**

- Created an IAM inline policy allowing the Lambda role to read chatbot parameters from SSM Parameter Store.
- Attached the policy to the existing Lab 10 Lambda execution role.

**File created:**

```text
ssm-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["ssm:GetParameter"],
            "Resource": "arn:aws:ssm:us-east-1:*:parameter/workshop/chatbot/*"
        }
    ]
}
```

**Attach policy command:**

```powershell
aws iam put-role-policy `
  --role-name workshop-lab10-lambda-role `
  --policy-name ssm-read `
  --policy-document file://ssm-policy.json
```

**Expected result:**

```text
No output
```

**Notes:**

- No output usually means the policy attached successfully.
- Without this permission, the Lambda function would receive an `AccessDeniedException` when reading the SSM parameter.

---

### Step 4: Create the Resilient Lambda Handler

**What I did:**

- Updated the chatbot handler to read `/workshop/chatbot/mode` from SSM Parameter Store.
- Added a fallback response path.
- Added structured logs for live requests, fallback requests, API success, API failure, and request completion.

**File updated:**

```text
handler.py
```

**File content:**

```python
import json
import logging
import time
import urllib.request
import urllib.error
import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

TRIVIA_API_URL = "https://opentdb.com/api.php?amount=1&type=multiple"
SSM_PARAMETER = "/workshop/chatbot/mode"

ssm_client = boto3.client("ssm", region_name="us-east-1")

FALLBACK_RESPONSE = (
    "Category: General Knowledge\n"
    "Question: What is the largest planet in our solar system?\n"
    "Answer: Jupiter"
)


def get_mode():
    """Check SSM Parameter Store for current mode: live or fallback."""
    try:
        result = ssm_client.get_parameter(Name=SSM_PARAMETER)
        return result["Parameter"]["Value"]
    except Exception:
        return "live"


def lambda_handler(event, context):
    """Resilient chatbot with fallback mode."""
    start_time = time.time()
    request_id = context.aws_request_id

    body = {}
    if event.get("body"):
        try:
            body = json.loads(event["body"])
        except (json.JSONDecodeError, TypeError):
            body = {}

    user_message = body.get("message", "Give me a trivia question")
    mode = get_mode()

    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_received",
        "request_id": request_id,
        "user_message": user_message,
        "mode": mode
    }))

    if mode == "fallback":
        api_latency_ms = 0

        logger.info(json.dumps({
            "level": "INFO",
            "event": "fallback_used",
            "request_id": request_id,
            "reason": "Mode set to fallback via SSM parameter"
        }))

        bot_response = FALLBACK_RESPONSE

    else:
        api_start = time.time()

        try:
            req = urllib.request.Request(TRIVIA_API_URL)
            with urllib.request.urlopen(req, timeout=5) as response:
                api_data = json.loads(response.read().decode())
                api_status = response.status

            api_latency_ms = round((time.time() - api_start) * 1000, 2)

            logger.info(json.dumps({
                "level": "INFO",
                "event": "api_call_success",
                "request_id": request_id,
                "api_url": TRIVIA_API_URL,
                "api_status": api_status,
                "api_latency_ms": api_latency_ms
            }))

            question_data = api_data["results"][0]
            question = question_data["question"]
            correct = question_data["correct_answer"]
            category = question_data["category"]

            bot_response = (
                f"Category: {category}\n"
                f"Question: {question}\n"
                f"Answer: {correct}"
            )

        except (urllib.error.URLError, urllib.error.HTTPError, Exception) as e:
            api_latency_ms = round((time.time() - api_start) * 1000, 2)

            logger.error(json.dumps({
                "level": "ERROR",
                "event": "api_call_failed",
                "request_id": request_id,
                "api_url": TRIVIA_API_URL,
                "api_latency_ms": api_latency_ms,
                "error": str(e)
            }))

            bot_response = "Sorry, I couldn't fetch a trivia question right now. Try again later!"

    total_duration_ms = round((time.time() - start_time) * 1000, 2)

    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_completed",
        "request_id": request_id,
        "total_duration_ms": total_duration_ms,
        "api_latency_ms": api_latency_ms,
        "mode": mode
    }))

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "response": bot_response,
            "metadata": {
                "api_latency_ms": api_latency_ms,
                "total_duration_ms": total_duration_ms,
                "mode": mode
            }
        })
    }
```

**What changed:**

| Change | Purpose |
|---|---|
| `boto3` import | Allows Lambda to read SSM Parameter Store |
| `SSM_PARAMETER` constant | Defines the mode switch path |
| `get_mode()` function | Reads the current mode from Parameter Store |
| `FALLBACK_RESPONSE` | Provides a cached response when the API is bypassed |
| `fallback_used` log event | Makes fallback behavior observable in CloudWatch Logs |
| `mode` metadata | Shows whether the request used live or fallback mode |

---

### Step 5: Deploy the Resilient Version

**What I did:**

- Repackaged the updated handler.
- Updated the existing chatbot Lambda function code.

**Commands used:**

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

- The updated Lambda function now supports both live and fallback modes.
- If the function still behaves like the previous version, delete `function.zip`, repackage, and redeploy.

---

### Step 6: Test in Live Mode

**What I did:**

- Invoked the chatbot while the SSM parameter was set to `live`.
- Confirmed the bot returned a random trivia question.
- Confirmed the response metadata showed `mode: live`.

**Command used:**

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

- The chatbot returned a random trivia question.
- The response metadata included:

```json
"mode": "live"
```

- API latency was usually around a few hundred milliseconds.

---

### Step 7: Switch to Fallback Mode

**What I did:**

- Changed the SSM parameter value from `live` to `fallback`.
- Did not redeploy the Lambda function.

**Command used:**

```powershell
aws ssm put-parameter `
  --name "/workshop/chatbot/mode" `
  --type String `
  --value "fallback" `
  --overwrite `
  --region us-east-1
```

**Expected result:**

```json
{
  "Version": 2,
  "Tier": "Standard"
}
```

**What this proved:**

- Application behavior can be changed through configuration.
- No code change or deployment was needed to switch modes.

---

### Step 8: Test in Fallback Mode

**What I did:**

- Invoked the chatbot again after switching the mode to `fallback`.
- Confirmed that the chatbot returned the cached Jupiter trivia response.
- Confirmed no external API call was made.

**Command used:**

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

- The response contained the fallback question:

```text
Question: What is the largest planet in our solar system?
Answer: Jupiter
```

- The response metadata included:

```json
"mode": "fallback"
```

- The response metadata also showed:

```json
"api_latency_ms": 0
```

**What this proved:**

- The bot skipped the external API.
- The fallback response was returned quickly.
- Users would still receive a useful answer even during dependency problems.

---

### Step 8A: Generate More Fallback Data

**What I did:**

- Invoked the chatbot multiple times while it was in fallback mode.
- Generated enough data for CloudWatch metrics and dashboard visibility.

**Command used:**

```powershell
1..5 | ForEach-Object {
  aws lambda invoke `
    --function-name workshop-chatbot-lab10 `
    --region us-east-1 `
    --cli-binary-format raw-in-base64-out `
    --payload file://payload.json `
    response.json | Out-Null

  Write-Host "Fallback invocation $_ done"
}
```

**Expected result:**

```text
Fallback invocation 1 done
Fallback invocation 2 done
Fallback invocation 3 done
Fallback invocation 4 done
Fallback invocation 5 done
```

---

### Step 9: Switch Back to Live Mode

**What I did:**

- Changed the SSM parameter value back to `live`.
- Invoked the chatbot again.
- Confirmed the bot returned live random trivia questions again.

**Switch back command:**

```powershell
aws ssm put-parameter `
  --name "/workshop/chatbot/mode" `
  --type String `
  --value "live" `
  --overwrite `
  --region us-east-1
```

**Expected result:**

```json
{
  "Version": 3,
  "Tier": "Standard"
}
```

**Invoke again:**

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

- The response contained a random trivia question.
- The response metadata included:

```json
"mode": "live"
```

**What this proved:**

- The mode switch is reversible.
- Runtime configuration can return the application to normal behavior without redeployment.

---

### Step 10: Review the CloudWatch Dashboard and Logs

**What I did:**

- Opened the CloudWatch dashboard from Lab 10B.
- Reviewed how live mode and fallback mode appeared in the graphs.
- Queried CloudWatch Logs for `fallback_used` events.

**Console checkpoint:**

```text
CloudWatch Console
Dashboards
workshop-chatbot-dashboard
```

**Expected dashboard observations:**

| Dashboard Area | Expected Observation |
|---|---|
| External API Latency | Latency appears during live mode and drops to zero during fallback mode |
| Chatbot Invocations | Invocations continue in both live and fallback mode |
| Lambda Errors | Errors remain at zero if fallback works correctly |
| Custom Metrics | Metrics show the bot remains available even when bypassing the API |

**CloudWatch Logs Insights query command:**

```powershell
$endTime = [int64](([datetime]::UtcNow) - ([datetime]'1970-01-01')).TotalSeconds
$startTime = $endTime - 1800

$queryId = aws logs start-query `
  --log-group-name "/aws/lambda/workshop-chatbot-lab10" `
  --start-time $startTime `
  --end-time $endTime `
  --query-string "fields @timestamp, @message | filter @message like /fallback_used/ | sort @timestamp desc | limit 5" `
  --region us-east-1 `
  --query "queryId" `
  --output text

Start-Sleep 5

aws logs get-query-results `
  --query-id $queryId `
  --region us-east-1 `
  --query "results[*][*].value" `
  --output text
```

**Expected result:**

- Logs showed entries containing:

```json
"event": "fallback_used"
```

**What this proved:**

- Fallback mode was visible in logs.
- The dashboard told the story of live requests, fallback requests, and continued availability.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| `workshop-lab-10a` folder reused | Passed |
| SSM parameter `/workshop/chatbot/mode` created | Passed |
| Initial parameter value set to `live` | Passed |
| SSM read policy file created | Passed |
| SSM read policy attached to Lambda role | Passed |
| Resilient Lambda handler created | Passed |
| Lambda code packaged | Passed |
| Lambda function updated | Passed |
| Live mode invocation tested | Passed |
| Live mode metadata verified | Passed |
| SSM parameter changed to `fallback` | Passed |
| Fallback mode invocation tested | Passed |
| Cached Jupiter response verified | Passed |
| `api_latency_ms` showed `0` in fallback mode | Passed |
| Multiple fallback invocations generated | Passed |
| SSM parameter changed back to `live` | Passed |
| Live mode restored | Passed |
| CloudWatch dashboard reviewed | Passed |
| `fallback_used` log events queried | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `AccessDeniedException` reading SSM parameter | Lambda role does not have permission to read `/workshop/chatbot/mode` | Reattach the inline `ssm-read` policy to `workshop-lab10-lambda-role` |
| Bot still returns random questions in fallback mode | Old Lambda code may still be deployed or the parameter is still set to `live` | Verify `handler.py` includes `get_mode()`, repackage and redeploy, then check the parameter value |
| `ParameterNotFound` | Parameter name or region is wrong | Run `aws ssm get-parameter --name "/workshop/chatbot/mode" --region us-east-1` |
| `api_latency_ms` is not `0` in fallback mode | The bot is still taking the live API path | Confirm the SSM parameter value is `fallback` |
| `No module named 'boto3'` | Lambda runtime issue or wrong runtime selected | Python 3.12 Lambda normally includes `boto3`; verify the runtime and redeploy |
| `put-role-policy` fails | JSON policy file is malformed or the terminal is in the wrong folder | Confirm `ssm-policy.json` exists in the current folder and validate the JSON |
| Lambda returns old response structure | Old ZIP package was deployed | Delete `function.zip`, recreate it from the updated `handler.py`, and run `update-function-code` again |
| Dashboard does not update immediately | CloudWatch metrics and filters can take time to appear | Wait a few minutes and invoke the Lambda several more times |
| Logs query returns no fallback events | No recent fallback requests were logged or the time window is too narrow | Generate fallback invocations and increase the query time window |
| AWS CLI authentication error | SSO session expired or profile is not set | Run `aws sso login --profile <YOUR_PROFILE_NAME>` and verify with `aws sts get-caller-identity` |

## Cleanup

This cleanup removes the resources created across Session 10.

> Only complete this section if you are finished with Labs 10A, 10B, and 10C.

### Step 1: Delete the SSM Parameter

```powershell
aws ssm delete-parameter `
  --name "/workshop/chatbot/mode" `
  --region us-east-1
```

### Step 2: Delete the CloudWatch Alarm and Dashboard

```powershell
aws cloudwatch delete-alarms `
  --alarm-names chatbot-api-slow `
  --region us-east-1

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

### Step 4: Delete Lambda Function and Log Group

```powershell
aws lambda delete-function `
  --function-name workshop-chatbot-lab10 `
  --region us-east-1

aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-chatbot-lab10 `
  --region us-east-1
```

### Step 5: Remove the Lambda Role

```powershell
aws iam delete-role-policy `
  --role-name workshop-lab10-lambda-role `
  --policy-name ssm-read

aws iam detach-role-policy `
  --role-name workshop-lab10-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role `
  --role-name workshop-lab10-lambda-role
```

### Step 6: Delete the Local Project Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-10a
```

## Cleanup Verification

### Verify SSM Parameter Is Deleted

```powershell
aws ssm get-parameter `
  --name "/workshop/chatbot/mode" `
  --region us-east-1
```

**Expected result:**

```text
ParameterNotFound
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

### Verify Lambda Role Is Deleted

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
| `/workshop/chatbot/mode` SSM parameter | Deleted |
| `chatbot-api-slow` CloudWatch alarm | Deleted |
| `workshop-chatbot-dashboard` CloudWatch dashboard | Deleted |
| `api-latency` metric filter | Deleted |
| `api-errors` metric filter | Deleted |
| `workshop-chatbot-lab10` Lambda function | Deleted |
| `/aws/lambda/workshop-chatbot-lab10` log group | Deleted |
| `ssm-read` inline role policy | Deleted |
| `workshop-lab10-lambda-role` IAM role | Deleted |
| Local `workshop-lab-10a` folder | Deleted |

## What I Learned

- Detecting problems is useful, but resilient architecture also needs to tolerate problems.
- Graceful degradation keeps an application useful even when one dependency is unhealthy.
- A fallback response is better than an error or a long timeout.
- SSM Parameter Store can be used as a runtime configuration switch.
- The chatbot can switch between live and fallback behavior without redeploying Lambda code.
- The circuit breaker pattern helps prevent repeated calls to unhealthy dependencies.
- Structured logs make fallback behavior visible in CloudWatch Logs.
- CloudWatch dashboards can show different operating states such as live mode, fallback mode, and errors.
- A well-designed system should remain responsive even when external services are unreliable.
- Resilience is a major part of the AWS Well-Architected Framework and Solutions Architect design thinking.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/project-folder-opened.png` | Existing Lab 10 project folder opened |
| `screenshots/ssm-parameter-created-live.png` | `/workshop/chatbot/mode` parameter created with value `live` |
| `screenshots/ssm-policy-created.png` | SSM read policy file created |
| `screenshots/ssm-policy-attached.png` | SSM read policy attached to Lambda role |
| `screenshots/resilient-handler-created.png` | Updated resilient Lambda handler created |
| `screenshots/lambda-resilient-code-deployed.png` | Updated Lambda code deployed |
| `screenshots/live-mode-response.png` | Chatbot returned live random trivia response |
| `screenshots/ssm-parameter-fallback.png` | SSM parameter changed to `fallback` |
| `screenshots/fallback-mode-response.png` | Chatbot returned cached Jupiter fallback response |
| `screenshots/fallback-invocations-generated.png` | Multiple fallback invocations generated |
| `screenshots/ssm-parameter-live-restored.png` | SSM parameter changed back to `live` |
| `screenshots/live-mode-restored.png` | Chatbot returned live random trivia again |
| `screenshots/dashboard-three-states.png` | CloudWatch dashboard showed live/fallback/error behavior |
| `screenshots/fallback-used-logs.png` | CloudWatch Logs query showed `fallback_used` events |