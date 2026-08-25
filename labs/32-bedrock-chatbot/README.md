# Lab 32: Bedrock Chatbot

## Lab Summary

In this lab, I deployed my first generative AI chatbot on AWS using AWS Lambda, Amazon API Gateway, Amazon Bedrock, and Amazon CloudWatch Logs.

The lab replaced the external trivia API pattern from Session 10 with Amazon Bedrock. I first tested Amazon Bedrock directly from the AWS CLI to confirm that my account could call the Nova Micro model. After confirming access, I created a Lambda function that can serve a browser-based chat interface, receive user messages, call Amazon Bedrock through the Converse API, and return generated AI responses.

I then created an API Gateway REST API with a public `/chat` endpoint so the chatbot could be opened from a browser. After deploying the API, I tested the chat interface, asked several questions, and reviewed structured CloudWatch logs that captured request events, Bedrock latency, model ID, and token usage.

This lab demonstrated how a serverless AI application can combine Lambda for compute, API Gateway for public access, Bedrock for AI inference, and CloudWatch Logs for observability.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 11A — Your First AI Chatbot — Call Amazon Bedrock from Lambda
- Session: 11 — AI Engineering
- Track: AI Engineering
- Difficulty: Beginner
- Estimated time: 40–50 minutes
- Target certification: AWS Certified AI Practitioner

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Test Amazon Bedrock directly from the AWS CLI
- Create a Lambda function that calls Amazon Bedrock
- Use Amazon Nova Micro as the foundation model
- Add structured JSON logging to the Lambda function
- Create an IAM trust policy for Lambda
- Create a Lambda execution role
- Attach Lambda logging permissions
- Add least-privilege Bedrock model invocation permissions
- Package and deploy the Lambda function
- Create an API Gateway REST API
- Create a `/chat` API Gateway resource
- Add GET, POST, and OPTIONS methods
- Connect API Gateway methods to the Lambda function
- Grant API Gateway permission to invoke the Lambda function
- Deploy the API Gateway stage
- Open the chatbot in a browser
- Send prompts to the chatbot
- Verify the AI responses and response metadata
- Review structured logs in CloudWatch Logs
- Understand token usage, latency, and model tracking
- Preserve resources for Lab 11B or clean up if stopping here

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| Amazon Bedrock | Provides access to the Amazon Nova Micro foundation model |
| Amazon Nova Micro | AI model used to generate chatbot responses |
| AWS Lambda | Runs the chatbot backend and serves the browser-based UI |
| Amazon API Gateway | Provides a public HTTPS endpoint for the chatbot |
| Amazon CloudWatch Logs | Stores structured Lambda logs for requests, Bedrock calls, latency, and token usage |
| AWS IAM | Provides the Lambda execution role and Bedrock permissions |
| AWS CLI | Creates, deploys, tests, and deletes AWS resources |
| Boto3 | Python SDK used by Lambda to call Bedrock Runtime |
| PowerShell | Runs AWS CLI commands and packages Lambda code |
| VS Code | Creates and edits code and JSON policy files |

## Prerequisites

- Completed Lab 1A: AWS Account & CLI Setup
- AWS CLI installed
- AWS CLI authenticated with a working profile
- Amazon Bedrock access available in the AWS account
- Access to the Amazon Nova Micro model in `us-east-1`
- VS Code or another text editor installed
- PowerShell available on Windows
- Basic understanding of Lambda and API Gateway

> This lab is standalone. It does not require Sessions 7–10 to be completed first.

## Cost Notice

Estimated cost: **under `$0.05`**

| Service | Cost Consideration |
|---|---|
| AWS Lambda | Expected to remain within the Lambda Always Free usage for this lab |
| Amazon API Gateway | Expected to remain within the free-tier level for the small number of test calls |
| Amazon CloudWatch Logs | Small structured log volume; expected to remain low cost |
| Amazon Bedrock | Charged per token; short prompts with Nova Micro should cost only a few cents or less |
| AWS IAM | Free |

> Amazon Bedrock is not completely free. It charges based on token usage. Keep prompts short, avoid unnecessary testing, and complete cleanup if you are not continuing to Lab 11B.

## Key Concepts

| Concept | Meaning |
|---|---|
| Amazon Bedrock | Fully managed AWS service for using foundation models through API calls |
| Foundation Model | A large AI model trained on broad data that can generate text, answer questions, summarize, and reason over prompts |
| Amazon Nova Micro | A fast, low-cost Amazon text model available through Amazon Bedrock |
| Token | Unit used by AI models to measure text input and output; roughly similar to a word or part of a word |
| Token Usage | Count of input, output, and total tokens used during an AI model request |
| Converse API | Bedrock API that provides a standard message-based interface for calling supported models |
| AI Inference | Sending input to an AI model and receiving generated output |
| API Gateway | AWS service used to expose the Lambda chatbot through a public HTTPS URL |
| CORS | Browser security mechanism that requires permission headers for cross-origin requests |
| Lambda Proxy Integration | API Gateway integration where the full HTTP request is passed to Lambda and Lambda returns an HTTP-style response |
| Structured Logging | Logging data as JSON so it is easier to search, filter, and analyze later |
| Latency | Time taken for the Bedrock call or total request to complete |
| Serverless AI Application | An AI application built without managing servers, usually using managed services like Lambda, API Gateway, and Bedrock |

## Security Notes

| Topic | Explanation |
|---|---|
| Least privilege Bedrock access | The Lambda role is allowed to invoke only the Amazon Nova Micro foundation model used in the lab |
| No secrets in code | The Lambda function uses IAM role permissions instead of hardcoded AWS access keys |
| Public endpoint awareness | API Gateway creates a public URL; anyone with the link can send prompts until cleanup is completed |
| Token cost awareness | Each user message consumes input and output tokens, which can create Bedrock charges |
| Structured logs may contain prompts | User messages are logged, so avoid entering secrets, personal data, or sensitive company information |
| Cleanup matters | The public API should be deleted if not continuing to the next lab |
| CORS headers | The Lambda returns CORS headers so the browser can send POST requests to the API endpoint |

## Architecture Overview

```text
Browser Chat UI
      |
      | GET /chat loads HTML
      | POST /chat sends message
      v
Amazon API Gateway
      |
      | Lambda proxy integration
      v
AWS Lambda: workshop-ai-chatbot-lab11
      |
      | bedrock-runtime Converse API
      v
Amazon Bedrock: Nova Micro
      |
      v
AI-generated response
      |
      v
CloudWatch Logs
(structured request, latency, token usage, model ID)
```

## Lab Steps

### Step 1: Set AWS Profile and Create Project Folder

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my active AWS identity.
- Created a local project folder for the lab.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity

mkdir ~\Desktop\workshop-lab-11a
cd ~\Desktop\workshop-lab-11a
pwd
```

**Expected result:**

- AWS CLI returned my account ID and active role.
- PowerShell showed the path to the new `workshop-lab-11a` folder.

**Notes:**

- If the SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

- I recorded my 12-digit AWS account ID because it is needed for Lambda role and API Gateway commands.

---

### Step 2: Test Bedrock Directly

**What I did:**

- Created a test prompt file.
- Called Amazon Bedrock directly from the AWS CLI.
- Verified that the account could access Amazon Nova Micro.

**File created:**

```text
test-prompt.json
```

**File content:**

```json
{
    "modelId": "amazon.nova-micro-v1:0",
    "messages": [
        {
            "role": "user",
            "content": [
                {
                    "text": "What is AWS Lambda in one sentence?"
                }
            ]
        }
    ],
    "inferenceConfig": {
        "maxTokens": 100,
        "temperature": 0.7
    }
}
```

**Command used:**

```powershell
aws bedrock-runtime converse `
  --cli-input-json file://test-prompt.json `
  --region us-east-1
```

**Expected result:**

- The command returned JSON output with an AI-generated response.
- The output included token usage such as `inputTokens`, `outputTokens`, and `totalTokens`.
- The output included latency information.

**Notes:**

- This confirmed that Bedrock access worked before building the Lambda function.
- If the command failed with an operation or access error, Bedrock model access needed to be checked in the AWS Console.

---

### Step 3: Create the AI Chatbot Lambda Code

**What I did:**

- Created the Python Lambda function.
- Added support for GET, POST, and OPTIONS requests.
- Added a browser-based chat UI directly inside the Lambda function.
- Added Bedrock Converse API calls.
- Added structured logging for request tracking, latency, token usage, and model ID.

**File created:**

```text
handler.py
```

**Simplified code summary:**

```python
import json
import logging
import time
import boto3

logger = logging.getLogger()
logger.setLevel(logging.INFO)

bedrock = boto3.client("bedrock-runtime", region_name="us-east-1")
MODEL_ID = "amazon.nova-micro-v1:0"
```

**Main behavior:**

| Request Type | What the Lambda Does |
|---|---|
| GET | Returns the browser-based chatbot HTML page |
| POST | Parses the user message, calls Bedrock, and returns the AI response |
| OPTIONS | Returns CORS headers for browser preflight requests |

**Structured log events included:**

| Event | Purpose |
|---|---|
| `request_received` | Captures the incoming user message and model ID |
| `bedrock_call_success` | Captures Bedrock latency and token usage |
| `bedrock_call_failed` | Captures Bedrock errors if the model call fails |
| `request_completed` | Captures total request duration |

**Notes:**

- The full Lambda code from the source lab was saved as `handler.py`.
- The embedded HTML makes the app self-contained because no separate frontend hosting is needed.
- The function uses `boto3.client("bedrock-runtime")` to call Amazon Bedrock.

---

### Step 4: Create the Lambda Role with Bedrock Permission

**What I did:**

- Created a Lambda trust policy.
- Created a least-privilege Bedrock permission policy.
- Created the Lambda execution role.
- Attached CloudWatch Logs permissions.
- Attached the inline Bedrock invocation policy.

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

**File created:**

```text
bedrock-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel"
            ],
            "Resource": "arn:aws:bedrock:us-east-1::foundation-model/amazon.nova-micro-v1:0"
        }
    ]
}
```

**Commands used:**

```powershell
aws iam create-role `
  --role-name workshop-lab11-lambda-role `
  --assume-role-policy-document file://lambda-trust.json
```

```powershell
aws iam attach-role-policy `
  --role-name workshop-lab11-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```powershell
aws iam put-role-policy `
  --role-name workshop-lab11-lambda-role `
  --policy-name bedrock-invoke `
  --policy-document file://bedrock-policy.json
```

**Expected result:**

- The role creation command returned JSON with the role ARN.
- The policy attachment and inline policy commands returned no output.

**Notes:**

- I waited briefly after creating the role to allow IAM propagation.
- The Bedrock policy is scoped to the Nova Micro foundation model instead of granting broad Bedrock access.

---

### Step 5: Package and Deploy the Lambda Function

**What I did:**

- Packaged the Lambda handler into a ZIP file.
- Created the Lambda function using Python 3.12.
- Configured the function with a 30-second timeout and 256 MB of memory.

**Package command:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
```

**Deploy command:**

```powershell
aws lambda create-function `
  --function-name workshop-ai-chatbot-lab11 `
  --runtime python3.12 `
  --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab11-lambda-role `
  --handler handler.lambda_handler `
  --zip-file fileb://function.zip `
  --timeout 30 `
  --memory-size 256 `
  --region us-east-1
```

**Expected result:**

- AWS returned JSON showing the Lambda function details.
- The function state showed `Pending` or `Active`.

**Notes:**

- The longer timeout gives Bedrock enough time to respond.
- The 256 MB memory size provides more room for the SDK and Bedrock client than a minimal test Lambda.

---

### Step 6: Create a Public URL with API Gateway

**What I did:**

- Created an API Gateway REST API.
- Captured the API ID in a PowerShell variable.
- Captured the root resource ID.
- Created a `/chat` resource.
- Added GET, POST, and OPTIONS methods.
- Connected each method to the Lambda function using Lambda proxy integration.
- Gave API Gateway permission to invoke the Lambda function.
- Deployed the API to the `live` stage.

**Create API:**

```powershell
$API_ID = aws apigateway create-rest-api `
  --name workshop-ai-chatbot `
  --region us-east-1 `
  --query "id" `
  --output text

Write-Host "API ID: $API_ID"
```

**Get root resource ID:**

```powershell
$ROOT_ID = aws apigateway get-resources `
  --rest-api-id $API_ID `
  --region us-east-1 `
  --query "items[0].id" `
  --output text

Write-Host "Root ID: $ROOT_ID"
```

**Create `/chat` resource:**

```powershell
$CHAT_ID = aws apigateway create-resource `
  --rest-api-id $API_ID `
  --parent-id $ROOT_ID `
  --path-part chat `
  --region us-east-1 `
  --query "id" `
  --output text

Write-Host "Chat ID: $CHAT_ID"
```

**Add methods:**

```powershell
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method GET --authorization-type NONE --region us-east-1 | Out-Null
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method POST --authorization-type NONE --region us-east-1 | Out-Null
aws apigateway put-method --rest-api-id $API_ID --resource-id $CHAT_ID --http-method OPTIONS --authorization-type NONE --region us-east-1 | Out-Null
Write-Host "Methods created: GET, POST, OPTIONS"
```

**Create Lambda integration URI:**

```powershell
$LAMBDA_URI = "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-ai-chatbot-lab11/invocations"
```

**Connect integrations:**

```powershell
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method GET --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 | Out-Null
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method POST --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 | Out-Null
aws apigateway put-integration --rest-api-id $API_ID --resource-id $CHAT_ID --http-method OPTIONS --type AWS_PROXY --integration-http-method POST --uri $LAMBDA_URI --region us-east-1 | Out-Null
Write-Host "Integrations connected"
```

**Add Lambda permission:**

```powershell
aws lambda add-permission `
  --function-name workshop-ai-chatbot-lab11 `
  --statement-id apigateway-invoke `
  --action lambda:InvokeFunction `
  --principal apigateway.amazonaws.com `
  --source-arn "arn:aws:execute-api:us-east-1:<YOUR_ACCOUNT_ID>:$API_ID/*" `
  --region us-east-1 | Out-Null

Write-Host "Lambda permission added"
```

**Deploy API:**

```powershell
aws apigateway create-deployment `
  --rest-api-id $API_ID `
  --stage-name live `
  --region us-east-1 | Out-Null

Write-Host "Deployed! Your chatbot URL is:"
Write-Host "https://$API_ID.execute-api.us-east-1.amazonaws.com/live/chat"
```

**Expected result:**

- API Gateway returned a public URL ending in `/live/chat`.
- The URL could be opened in a browser.

**Notes:**

- The `$API_ID`, `$ROOT_ID`, and `$CHAT_ID` variables only exist in the current PowerShell session.
- If the terminal closes, the API ID can be retrieved later with `aws apigateway get-rest-apis`.

---

### Step 7: Open and Test the Chatbot

**What I did:**

- Opened the API Gateway URL in a browser.
- Confirmed that the AI Cloud Tutor chat interface loaded.
- Sent several questions to the chatbot.
- Verified that the bot returned AI-generated responses.
- Reviewed the metadata displayed under responses.

**Chatbot URL format:**

```text
https://<API_ID>.execute-api.us-east-1.amazonaws.com/live/chat
```

**Example prompts tested:**

```text
What is S3?
```

```text
Explain IAM roles like I'm 5.
```

```text
Write a haiku about cloud computing.
```

**Expected result:**

- The message appeared on the right side of the chat UI.
- A temporary `Thinking...` message appeared.
- The AI response appeared on the left side.
- Metadata showed latency and token usage.

**Notes:**

- Every message sent to the chatbot uses Bedrock tokens.
- The URL remains public until API Gateway and Lambda resources are cleaned up.

---

### Step 8: View Logs in CloudWatch

**What I did:**

- Opened CloudWatch Logs.
- Found the Lambda log group.
- Opened the most recent log stream.
- Reviewed structured log entries for chatbot requests and Bedrock calls.

**Log group:**

```text
/aws/lambda/workshop-ai-chatbot-lab11
```

**Expected structured log events:**

| Log Event | What It Shows |
|---|---|
| `request_received` | User message, request ID, and model ID |
| `bedrock_call_success` | Bedrock latency, input tokens, output tokens, total tokens, and model ID |
| `bedrock_call_failed` | Error details if Bedrock invocation fails |
| `request_completed` | Total request duration and Bedrock latency |

**What this proved:**

- The AI application was observable.
- Each request generated traceable structured logs.
- Token usage and latency could be reviewed after each chat request.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| Local lab folder created | Passed |
| Bedrock test prompt created | Passed |
| Bedrock direct CLI call succeeded | Passed |
| Lambda handler created | Passed |
| Lambda trust policy created | Passed |
| Bedrock permission policy created | Passed |
| Lambda IAM role created | Passed |
| Lambda logging policy attached | Passed |
| Bedrock inline policy attached | Passed |
| Lambda package created | Passed |
| Lambda function deployed | Passed |
| API Gateway REST API created | Passed |
| `/chat` resource created | Passed |
| GET, POST, and OPTIONS methods created | Passed |
| API Gateway integrations connected to Lambda | Passed |
| Lambda permission added for API Gateway | Passed |
| API deployed to `live` stage | Passed |
| Chatbot URL opened in browser | Passed |
| AI chatbot response returned successfully | Passed |
| Response metadata displayed | Passed |
| CloudWatch structured logs reviewed | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| Bedrock test failed with `ThrottlingException: Too many tokens per day` | The account, model, or region appeared to have reached or been assigned a low daily token quota for Bedrock model invocation | Reduced the prompt size and `maxTokens`, checked Amazon Bedrock model access and Service Quotas for the selected model in `us-east-1`, and planned to request a quota increase if the applied quota remained too low |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `Operation not allowed` when calling Bedrock | Bedrock model access may not be enabled or allowed in the account | Try the Bedrock console and model playground for Nova Micro; request access or contact AWS Support if needed |
| `AccessDeniedException` when invoking Bedrock | Lambda role does not have permission to invoke the Bedrock model | Re-run the `aws iam put-role-policy` command for `bedrock-invoke` |
| `The role cannot be assumed` | IAM role propagation delay | Wait 10–30 seconds and retry the Lambda create command |
| `Unable to import module 'handler'` | ZIP file has the wrong structure or missing `handler.py` at the root | Recreate `function.zip` from inside the project folder using `Compress-Archive -Path handler.py` |
| `Missing Authentication Token` in browser | The URL path is wrong | Make sure the URL ends with `/live/chat` |
| Chat UI loads but messages fail | API Gateway integration or Lambda permission may be incorrect | Recheck the integration URI and `aws lambda add-permission` source ARN |
| First response is slow | Lambda cold start and Bedrock model warmup | Wait and try again; later responses should be faster |
| API Gateway URL does not work after changes | API was not redeployed | Run `aws apigateway create-deployment --rest-api-id $API_ID --stage-name live` again |
| PowerShell variables are blank | Terminal session was closed or variables were lost | Retrieve API IDs using AWS CLI or recreate the variables in the same terminal |
| Unexpected Bedrock charges | Too many prompts were sent or endpoint was left public | Stop testing, complete cleanup, and monitor AWS Billing |
| AWS CLI authentication error | SSO session expired or profile not set | Run `aws sso login --profile <YOUR_PROFILE_NAME>` and reset `$env:AWS_PROFILE` |

## Cleanup

> If continuing to Lab 11B, keep the Lambda function, IAM role, API Gateway, and log group because Lab 11B updates the same chatbot.

### Step 1: Delete the API Gateway REST API

If the same terminal session is still open:

```powershell
aws apigateway delete-rest-api `
  --rest-api-id $API_ID `
  --region us-east-1
```

If `$API_ID` is blank, find it first:

```powershell
aws apigateway get-rest-apis `
  --region us-east-1 `
  --query "items[?name=='workshop-ai-chatbot'].id" `
  --output text
```

Then delete it:

```powershell
aws apigateway delete-rest-api `
  --rest-api-id <API_ID> `
  --region us-east-1
```

### Step 2: Delete the Lambda Function

```powershell
aws lambda delete-function `
  --function-name workshop-ai-chatbot-lab11 `
  --region us-east-1
```

### Step 3: Delete the CloudWatch Log Group

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-ai-chatbot-lab11 `
  --region us-east-1
```

### Step 4: Remove the Lambda IAM Role

```powershell
aws iam delete-role-policy `
  --role-name workshop-lab11-lambda-role `
  --policy-name bedrock-invoke
```

```powershell
aws iam detach-role-policy `
  --role-name workshop-lab11-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```powershell
aws iam delete-role `
  --role-name workshop-lab11-lambda-role
```

### Step 5: Delete the Local Lab Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-11a
```

## Cleanup Verification

### Verify API Gateway Is Deleted

```powershell
aws apigateway get-rest-apis `
  --region us-east-1 `
  --query "items[?name=='workshop-ai-chatbot'].name"
```

**Expected result:**

```json
[]
```

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

### Verify IAM Role Is Deleted

```powershell
aws iam get-role `
  --role-name workshop-lab11-lambda-role
```

**Expected result:**

```text
NoSuchEntity
```

**Expected cleanup result:**

| Resource | Expected State |
|---|---|
| `workshop-ai-chatbot` API Gateway API | Deleted unless continuing to Lab 11B |
| `workshop-ai-chatbot-lab11` Lambda function | Deleted unless continuing to Lab 11B |
| `/aws/lambda/workshop-ai-chatbot-lab11` log group | Deleted unless continuing to Lab 11B |
| `workshop-lab11-lambda-role` IAM role | Deleted unless continuing to Lab 11B |
| Local `workshop-lab-11a` folder | Deleted only if not needed |

## What I Learned

- Amazon Bedrock allows applications to call foundation models without managing machine learning infrastructure.
- Amazon Nova Micro is a low-cost model suitable for simple text chatbot use cases.
- Bedrock model calls are API requests that return generated text and token usage.
- Lambda can act as a serverless AI application backend.
- API Gateway can expose a Lambda function through a public HTTPS endpoint.
- A Lambda function can serve both the frontend chat page and backend API responses.
- The Converse API uses a message-based format for model interaction.
- Token usage matters because AI inference cost is based on input and output tokens.
- Structured logs make AI applications easier to monitor and troubleshoot.
- Logging model ID, latency, and token usage helps connect AI behavior to operational and cost data.
- IAM roles allow Lambda to call Bedrock without hardcoded credentials.
- Least privilege can be applied by limiting the role to one specific foundation model.
- Public AI endpoints should be cleaned up or controlled to avoid unnecessary cost.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/lab-folder-created.png` | Local `workshop-lab-11a` folder created |
| `screenshots/bedrock-test-prompt-created.png` | Bedrock test prompt file created |
| `screenshots/bedrock-cli-response.png` | Bedrock direct CLI call succeeded |
| `screenshots/lambda-handler-created.png` | Chatbot Lambda handler created |
| `screenshots/lambda-trust-policy-created.png` | Lambda trust policy created |
| `screenshots/bedrock-policy-created.png` | Bedrock invoke policy created |
| `screenshots/lambda-role-created.png` | Lambda IAM role created |
| `screenshots/lambda-policies-attached.png` | Lambda logging and Bedrock permissions attached |
| `screenshots/function-zip-created.png` | Lambda deployment ZIP created |
| `screenshots/lambda-function-created.png` | Lambda chatbot function deployed |
| `screenshots/api-gateway-created.png` | API Gateway REST API created |
| `screenshots/chat-resource-created.png` | `/chat` API Gateway resource created |
| `screenshots/api-methods-created.png` | GET, POST, and OPTIONS methods created |
| `screenshots/api-integrations-connected.png` | API methods connected to Lambda |
| `screenshots/lambda-permission-added.png` | API Gateway invoke permission added to Lambda |
| `screenshots/api-deployed-url.png` | API deployed and chatbot URL generated |
| `screenshots/chatbot-ui-loaded.png` | Browser chat UI loaded successfully |
| `screenshots/chatbot-response-success.png` | AI chatbot returned a response |
| `screenshots/chatbot-response-metadata.png` | Response metadata displayed latency and token usage |
| `screenshots/cloudwatch-log-group.png` | Lambda CloudWatch log group opened |
| `screenshots/structured-log-bedrock-success.png` | Structured log showed Bedrock success event |
| `screenshots/cleanup-verified.png` | Lab resources removed or confirmed kept for Lab 11B |