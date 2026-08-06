# Lab 29: Deploy Chatbot with Structured Logging

## Lab Summary

In this lab, I deployed a trivia chatbot using AWS Lambda and configured it to write structured JSON logs to Amazon CloudWatch Logs.

The Lambda function accepts a user message through a CLI invocation payload, calls the Open Trivia Database API to retrieve a random trivia question, and returns a formatted chatbot response containing the trivia category, question, and answer. The function also records structured log events at each major stage of the request: when the request is received, when the external API call succeeds or fails, and when the request completes.

This lab introduced a more realistic serverless application than earlier Lambda labs because the function depends on an external API. The main goal was not just to deploy a working Lambda function, but to make the application observable by recording useful fields such as request ID, API status, API latency, total duration, and event type.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 10A — Deploy a Trivia Chatbot — Your First Structured Logging App
- Session: 10 — Chatbot Observability
- Track: Solutions Architecture
- Difficulty: Beginner
- Target certification: AWS Solutions Architect – Associate

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder for Lab 10A
- Create a Python Lambda function for a trivia chatbot
- Parse user input from a Lambda event payload
- Call the Open Trivia Database external API
- Measure external API latency
- Return a formatted trivia response
- Write structured JSON logs at each stage of execution
- Create a Lambda trust policy
- Create a Lambda execution role
- Attach CloudWatch Logs permissions to the Lambda role
- Package the Lambda code into a ZIP file
- Deploy the Lambda function with AWS CLI
- Create a test payload file
- Invoke the chatbot Lambda function
- Read the Lambda response
- Generate several test invocations
- Retrieve the most recent CloudWatch log stream
- Review structured JSON log entries in CloudWatch Logs
- Verify the log group and log stream in the AWS Console
- Understand why structured logging is important for observability
- Preserve the function and logs for Lab 10B, or clean them up if stopping here

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Lambda | Runs the trivia chatbot function without managing servers |
| AWS IAM | Provides the Lambda execution role and trust relationship |
| Amazon CloudWatch Logs | Stores Lambda logs and structured JSON log entries |
| AWS CLI | Creates, deploys, invokes, and verifies AWS resources |
| PowerShell | Runs AWS CLI commands and packages the Lambda code |
| Python 3.12 | Runtime used by the Lambda function |
| Open Trivia Database API | External API used to fetch random trivia questions |
| VS Code | Creates and edits the Lambda code and JSON files |

## Prerequisites

- Completed Lab 1A: AWS Account & CLI Setup
- AWS CLI installed
- AWS CLI authenticated with a working profile
- PowerShell available on Windows
- VS Code or another text editor installed
- Internet access available from the Lambda function

> This lab is standalone. It does not require Sessions 7–9, but it builds naturally on the Lambda, CloudWatch, and observability concepts from Session 9.

## Cost Notice

Estimated cost: `$0.00`

| Service | Cost Consideration |
|---|---|
| AWS Lambda | Expected to remain within the Lambda Always Free usage for this lab |
| Amazon CloudWatch Logs | Small log volume; expected to remain within free-tier usage |
| AWS IAM | Free |

> If continuing to Lab 10B, skip cleanup for now because Lab 10B uses the same chatbot function and CloudWatch logs. If stopping here, complete the cleanup section to avoid leaving unused resources behind.

## Key Concepts

| Concept | Meaning |
|---|---|
| AWS Lambda | A serverless compute service that runs code without managing servers |
| Chatbot | An application that accepts a user message and returns a conversational response |
| External API | A service outside the application that provides data over the internet |
| Open Trivia Database | Public trivia API used by the Lambda function to retrieve a random question |
| Structured Logging | Writing logs as consistent JSON objects instead of plain text messages |
| CloudWatch Logs | AWS service that stores logs from Lambda and other AWS services |
| Request ID | Unique identifier for a Lambda invocation, useful for tracing one request through multiple log events |
| API Latency | How long the external API call takes to return a response |
| Total Duration | How long the full Lambda request takes from start to finish |
| JSON Logging | A log format that makes fields searchable, filterable, and queryable |
| Observability | The ability to understand what an application is doing by looking at its logs, metrics, and traces |
| Failure Handling | Returning a safe response when a dependency such as an external API fails |

## Security Notes

| Topic | Explanation |
|---|---|
| Least privilege | The Lambda role only needs basic CloudWatch Logs permissions for this lab |
| No secrets in code | The Lambda function does not store access keys, API keys, or sensitive credentials |
| External dependency risk | External APIs can be slow, unavailable, or return unexpected data |
| Timeout protection | The external API call uses a timeout so the function does not hang indefinitely |
| Structured logs | JSON logs help investigate issues without exposing unnecessary sensitive data |
| Cleanup | Unused Lambda functions, log groups, and IAM roles should be removed when no longer needed |

## Architecture Overview

```text
User / AWS CLI
      |
      | invokes Lambda with payload.json
      v
Lambda Function: workshop-chatbot-lab10
      |
      | calls external API
      v
Open Trivia Database API
      |
      | returns trivia question
      v
Lambda formats chatbot response
      |
      +----------------------+
      |                      |
      v                      v
response.json          CloudWatch Logs
                       structured JSON events
```

## Lab Steps

### Step 1: Set AWS Profile and Create Project Folder

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my AWS identity.
- Created a local folder for the lab.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity

mkdir ~\Desktop\workshop-lab-10a
cd ~\Desktop\workshop-lab-10a
pwd
```

**Expected result:**

- AWS CLI returned my account ID and active role.
- PowerShell showed the path to the new `workshop-lab-10a` folder.

**Notes:**

- If the SSO token expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

- I recorded my 12-digit AWS account ID for the Lambda deployment command.

---

### Step 2: Create the Chatbot Lambda Function Code

**What I did:**

- Created a Python Lambda function named `handler.py`.
- Added code to parse a user message from the event body.
- Added code to call the Open Trivia Database API.
- Added structured JSON logging for request receipt, API call result, and request completion.
- Added fallback handling in case the external API call fails.

**File created:**

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

logger = logging.getLogger()
logger.setLevel(logging.INFO)

TRIVIA_API_URL = "https://opentdb.com/api.php?amount=1&type=multiple"


def lambda_handler(event, context):
    """Chatbot that returns a trivia question from an external API."""
    start_time = time.time()
    request_id = context.aws_request_id

    body = {}
    if event.get("body"):
        try:
            body = json.loads(event["body"])
        except (json.JSONDecodeError, TypeError):
            body = {}

    user_message = body.get("message", "Give me a trivia question")

    logger.info(json.dumps({
        "level": "INFO",
        "event": "request_received",
        "request_id": request_id,
        "user_message": user_message
    }))

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
        "api_latency_ms": api_latency_ms
    }))

    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "response": bot_response,
            "metadata": {
                "api_latency_ms": api_latency_ms,
                "total_duration_ms": total_duration_ms
            }
        })
    }
```

**Notes:**

- The function writes three main successful log events:
  - `request_received`
  - `api_call_success`
  - `request_completed`
- If the external API fails, the function logs `api_call_failed` and returns a user-friendly fallback message.

---

### Step 3: Create the Lambda Execution Role

**What I did:**

- Created a trust policy that allows Lambda to assume the role.
- Created an IAM role for the chatbot Lambda function.
- Attached the AWS managed Lambda logging policy.

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
  --role-name workshop-lab10-lambda-role `
  --assume-role-policy-document file://lambda-trust.json
```

**Attach logging permissions:**

```powershell
aws iam attach-role-policy `
  --role-name workshop-lab10-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**Expected result:**

- The role creation command returned JSON output with the role ARN.
- The policy attachment command returned no output.

**Notes:**

- I waited about 10 seconds after attaching the policy so IAM role propagation could complete.

---

### Step 4: Package and Deploy the Lambda Function

**What I did:**

- Compressed `handler.py` into `function.zip`.
- Created the Lambda function named `workshop-chatbot-lab10`.
- Used Python 3.12 as the runtime.
- Set a 10-second timeout because the function calls an external API.

**Package command:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
```

**Deploy command:**

```powershell
aws lambda create-function `
  --function-name workshop-chatbot-lab10 `
  --runtime python3.12 `
  --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab10-lambda-role `
  --handler handler.lambda_handler `
  --zip-file fileb://function.zip `
  --timeout 10 `
  --memory-size 128 `
  --region us-east-1
```

**Expected result:**

- AWS returned JSON output showing the function configuration.
- The Lambda function state showed `Pending` or `Active`.

**Notes:**

- Replace `<YOUR_ACCOUNT_ID>` with the 12-digit AWS account ID.
- The ZIP file must contain `handler.py` at the root level.

---

### Step 5: Invoke the Chatbot and Review the Response

**What I did:**

- Created a test payload file.
- Invoked the Lambda function with the payload.
- Reviewed the chatbot response.

**File created:**

```text
payload.json
```

**File content:**

```json
{"body": "{\"message\": \"Give me a trivia question\"}"}
```

**Invoke command:**

```powershell
aws lambda invoke `
  --function-name workshop-chatbot-lab10 `
  --region us-east-1 `
  --cli-binary-format raw-in-base64-out `
  --payload file://payload.json `
  response.json
```

**Read response:**

```powershell
Get-Content response.json
```

**Expected result:**

```json
{
  "statusCode": 200,
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"response\": \"Category: ...\\nQuestion: ...\\nAnswer: ...\", \"metadata\": {\"api_latency_ms\": 342.15, \"total_duration_ms\": 345.67}}"
}
```

**Notes:**

- The exact trivia question changes because the external API returns a random question.
- The response includes latency metadata from the Lambda function.

---

### Step 6: Invoke Several Times and Check CloudWatch Logs

**What I did:**

- Invoked the chatbot several times to generate log data.
- Waited for logs to arrive in CloudWatch.
- Retrieved the most recent log stream.
- Reviewed the latest log events from the CLI.

**Invoke several times:**

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

**Get latest log stream:**

```powershell
$logStream = (aws logs describe-log-streams `
  --log-group-name /aws/lambda/workshop-chatbot-lab10 `
  --order-by LastEventTime `
  --descending `
  --limit 1 `
  --region us-east-1 | ConvertFrom-Json).logStreams[0].logStreamName
```

**Read log events:**

```powershell
aws logs get-log-events `
  --log-group-name /aws/lambda/workshop-chatbot-lab10 `
  --log-stream-name $logStream `
  --limit 20 `
  --region us-east-1
```

**Expected log events:**

```json
{"level": "INFO", "event": "request_received", "request_id": "...", "user_message": "Give me a trivia question"}
{"level": "INFO", "event": "api_call_success", "request_id": "...", "api_url": "https://opentdb.com/api.php?amount=1&type=multiple", "api_status": 200, "api_latency_ms": 342.15}
{"level": "INFO", "event": "request_completed", "request_id": "...", "total_duration_ms": 345.67, "api_latency_ms": 342.15}
```

**Notes:**

- Each invocation should produce structured JSON logs.
- Each log entry shares the same `request_id` for a given invocation.
- This makes it possible to trace a single request across multiple stages.

---

### Step 7: Console Checkpoint — View Logs in CloudWatch

**What I did:**

- Opened the AWS Console.
- Navigated to CloudWatch Logs.
- Opened the chatbot Lambda log group.
- Reviewed the most recent log stream.

**Console path:**

```text
AWS Console
CloudWatch
Logs
Log groups
/aws/lambda/workshop-chatbot-lab10
Most recent log stream
```

**What I verified:**

| Log Entry Type | Meaning |
|---|---|
| `START` | Lambda invocation started |
| Structured `[INFO]` entries | Custom JSON logs from the chatbot code |
| `END` | Lambda invocation ended |
| `REPORT` | Lambda execution summary including duration and memory |

**Expected structured log fields:**

```text
event
request_id
api_status
api_latency_ms
total_duration_ms
user_message
```

**Result:**

- CloudWatch Logs showed the chatbot execution history.
- The structured JSON logs proved that the function was instrumented for observability.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| Local `workshop-lab-10a` folder created | Passed |
| Chatbot `handler.py` created | Passed |
| Lambda trust policy created | Passed |
| Lambda execution role created | Passed |
| Lambda logging policy attached | Passed |
| Lambda ZIP package created | Passed |
| Chatbot Lambda deployed | Passed |
| Payload file created | Passed |
| Chatbot invoked successfully | Passed |
| Trivia response returned | Passed |
| Multiple invocations generated | Passed |
| CloudWatch log stream retrieved | Passed |
| Structured JSON logs reviewed | Passed |
| CloudWatch Console verification completed | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `No such file or directory: lambda-trust.json` | Terminal is not inside the Lab 10A project folder | Run `cd ~\Desktop\workshop-lab-10a` and verify with `pwd` |
| `The role cannot be assumed` when creating the function | IAM role propagation has not completed | Wait 10–30 seconds and retry the Lambda create command |
| `Function already exists` | A previous attempt already created the Lambda function | Skip creation or delete it first with `aws lambda delete-function --function-name workshop-chatbot-lab10 --region us-east-1` |
| `Unable to import module 'handler'` | The ZIP file contains the wrong structure or `handler.py` is nested inside a folder | Delete `function.zip`, confirm `handler.py` is in the current folder, then rerun `Compress-Archive -Path handler.py -DestinationPath function.zip -Force` |
| `Invalid base64` on invoke | Missing `--cli-binary-format raw-in-base64-out` | Rerun the invoke command with the `--cli-binary-format raw-in-base64-out` flag |
| `No such file or directory: payload.json` | Terminal is not in the folder where `payload.json` was saved | Run `pwd`, move into `workshop-lab-10a`, and retry |
| Response says `Sorry, I couldn't fetch a trivia question` | The external trivia API may be unavailable, slow, or unreachable | Wait briefly and invoke the function again; check CloudWatch logs for `api_call_failed` |
| CloudWatch log group not found | Logs may not have been created yet | Invoke the Lambda once, wait 30 seconds, then check again |
| Log stream query returns empty | Logs may not have arrived yet or the log group name is wrong | Wait briefly and confirm the log group name is exactly `/aws/lambda/workshop-chatbot-lab10` |
| `ExpiredTokenException` | AWS SSO session expired | Run `aws sso login --profile <YOUR_PROFILE_NAME>`, then reset `$env:AWS_PROFILE` |
| `AccessDenied` | Current AWS identity lacks permission | Confirm the correct admin/SSO profile is active with `aws sts get-caller-identity` |

## Cleanup

> If continuing to Lab 10B, skip cleanup for now. Lab 10B uses the same chatbot function and CloudWatch logs.

If stopping here, remove all resources created in Lab 10A.

### Step 1: Delete the Lambda Function

```powershell
aws lambda delete-function `
  --function-name workshop-chatbot-lab10 `
  --region us-east-1
```

### Step 2: Delete the CloudWatch Log Group

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-chatbot-lab10 `
  --region us-east-1
```

### Step 3: Remove the Lambda IAM Role

```powershell
aws iam detach-role-policy `
  --role-name workshop-lab10-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role `
  --role-name workshop-lab10-lambda-role
```

### Step 4: Delete the Local Lab Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-10a
```

## Cleanup Verification

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

### Verify Log Group Is Deleted

```powershell
aws logs describe-log-groups `
  --log-group-name-prefix /aws/lambda/workshop-chatbot-lab10 `
  --region us-east-1 `
  --query "logGroups[].logGroupName"
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
| `workshop-chatbot-lab10` Lambda function | Deleted if stopping here |
| `/aws/lambda/workshop-chatbot-lab10` log group | Deleted if stopping here |
| `workshop-lab10-lambda-role` IAM role | Deleted if stopping here |
| Local `workshop-lab-10a` folder | Deleted if stopping here |

## What I Learned

- AWS Lambda can run a chatbot-style application without managing servers.
- Lambda functions can call external APIs over the internet.
- External APIs introduce latency and failure risk.
- Structured JSON logging makes logs easier to search, filter, and query.
- Plain text logs are readable by humans, but structured logs are better for production troubleshooting.
- Request IDs help trace a single request across multiple log events.
- Measuring API latency inside the function provides useful operational data.
- CloudWatch Logs automatically captures Lambda logging output.
- The `START`, `END`, and `REPORT` lines are Lambda-managed logs, while the JSON entries are custom application logs.
- Good observability starts in the application code, not only in the AWS Console.
- Lab 10B will use these structured logs to query application behavior with CloudWatch Logs Insights.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/lab-folder-created.png` | Local `workshop-lab-10a` folder created |
| `screenshots/chatbot-handler-created.png` | Chatbot Lambda handler code created |
| `screenshots/lambda-trust-policy-created.png` | Lambda trust policy created |
| `screenshots/lambda-role-created.png` | Lambda execution role created |
| `screenshots/lambda-policy-attached.png` | Lambda logging policy attached |
| `screenshots/function-zip-created.png` | Lambda ZIP package created |
| `screenshots/chatbot-lambda-created.png` | Chatbot Lambda function deployed |
| `screenshots/payload-json-created.png` | Test payload file created |
| `screenshots/chatbot-invocation-success.png` | Lambda invocation returned status code 200 |
| `screenshots/chatbot-response-json.png` | Trivia chatbot response viewed in `response.json` |
| `screenshots/multiple-invocations-generated.png` | Multiple invocations generated log data |
| `screenshots/latest-log-stream-command.png` | Latest CloudWatch log stream retrieved |
| `screenshots/structured-json-logs-cli.png` | Structured JSON logs viewed from CLI |
| `screenshots/cloudwatch-log-group.png` | CloudWatch log group viewed in console |
| `screenshots/cloudwatch-log-stream.png` | Most recent log stream opened |
| `screenshots/request-received-log.png` | `request_received` structured log verified |
| `screenshots/api-call-success-log.png` | `api_call_success` structured log verified |
| `screenshots/request-completed-log.png` | `request_completed` structured log verified |
| `screenshots/cleanup-verified.png` | Cleanup verified if resources were removed |