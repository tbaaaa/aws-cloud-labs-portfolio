# Lab 12C: Capstone — Ship It. Cost Governance & the Full AI Stack

## Lab Summary

This capstone lab closed out the AI Engineering track by adding the one control that had been missing across Labs 11A–12B: an account-level financial ceiling that works independently of anything inside the application itself. Up to this point, cost was controlled entirely from inside the Lambda function — input validation, `maxTokens`, and a token-spike alarm. This lab added AWS Budgets as the outermost layer, so that even if every application-level control somehow failed, there would still be an account-wide alert before spending ran away.

I created a $5 monthly budget with an email notification that fires once actual spend crosses 80% ($4), verified it in both the CLI and the Billing console, and mapped out the resulting three-layer cost defense: input validation rejects expensive prompts before they reach the model, the token-spike alarm catches anomalies per request, and the budget is the account-wide backstop behind both. I then ran six end-to-end tests that exercised every layer built across the whole track at once — RAG grounding and citation, honest refusal outside the knowledge base, the guardrail's denied-topic block, the guardrail's PII block, the Lab 11C input-length rejection, and a normal "happy path" question — and confirmed the token alarm and dashboard were both healthy and reflecting the test traffic.

With the full stack verified, I scored the build against a production-readiness checklist covering whether it works, is controllable, stateful, grounded, honest, safe, cost-controlled, observable, and least-privilege, and reviewed what a real production version would still need (rate limiting, persistent history storage, managed retrieval, invocation logging, CI/CD, and guardrail-intervention alarms). Finally, I tore down every resource created across Sessions 11 and 12 — in the right order, with verification commands to confirm nothing was left running.

The core lesson of this capstone: anyone can wire an AI model to an endpoint, but shipping one means grounding it, making it safe, monitoring it, and putting a hard financial ceiling on it — and then being able to prove all of that actually works.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 12C — Capstone: Ship It. Cost Governance & the Full AI Stack
- Session: 12 — AI Engineering (Capstone)
- Track: AI Engineering
- Difficulty: Advanced
- Target certification: AWS Certified AI Practitioner

## Objectives

- Set the active AWS CLI profile and return to the Lab 11A project folder
- Define a $5 monthly AWS Budget (`budget.json`) and an 80%-threshold email notification (`notifications.json`)
- Create the account-level budget with its notification subscriber
- Verify the budget exists via the CLI and in the AWS Billing console
- Map out the three-layer cost defense: input validation, the token-spike alarm, and the account budget
- Run six end-to-end tests covering RAG grounding with citation, an honest out-of-scope refusal, a guardrail denied-topic block, a guardrail PII block, an input-length rejection, and a normal happy-path question
- Confirm the token-spike alarm is healthy and the AI operations dashboard reflects the test traffic
- Score the finished build against a nine-dimension production-readiness checklist
- Identify what a real production deployment would still need beyond this workshop build
- Reflect on the arc of the AI Engineering track across Sessions 11 and 12
- Tear down every resource created in Sessions 11 and 12 in the correct order: budget, guardrail, S3 knowledge base, CloudWatch dashboard/alarm/metric filter, API Gateway, Lambda function and log group, IAM role and policies, and the local project folder
- Verify the account is clean after teardown

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Budgets | Account-level monthly cost budget with an 80% threshold email alert — the outermost cost control |
| AWS Lambda | Runs the full chatbot stack being verified end-to-end |
| Amazon Bedrock (Nova Micro) + Guardrails | Generates grounded answers and enforces the safety layer exercised in the end-to-end tests |
| Amazon S3 | Hosts the RAG knowledge base exercised in the end-to-end tests |
| Amazon CloudWatch (Logs, Metrics, Dashboard, Alarms) | Confirms the monitoring stack is healthy and reflects the verification traffic |
| Amazon API Gateway | Public endpoint for the chatbot, removed as part of final teardown |
| AWS IAM | Execution role and scoped policies for the Lambda function, removed as part of final teardown |
| AWS CLI | Creates the budget, runs verification invocations, and tears down every resource |
| PowerShell | Runs AWS CLI commands throughout the lab |
| VS Code | Edits `budget.json`, `notifications.json`, and `payload.json` |

## Prerequisites

- Completed Labs 11A–11C and 12A–12B, with all of the following still in place:
  - `workshop-ai-chatbot-lab11` Lambda function
  - `workshop-lab11-lambda-role` IAM role
  - `workshop-ai-kb-<YOUR_ACCOUNT_ID>` S3 knowledge base
  - `workshop-ai-guardrail` guardrail
  - `workshop-ai-chatbot-dashboard` CloudWatch dashboard and the `ai-chatbot-token-spike` alarm
- AWS CLI installed and authenticated
- PowerShell available on Windows
- VS Code or another text editor installed
- The `workshop-lab-11a` project folder

> The cost-governance and retrospective portions of this lab can be done independently, but the full end-to-end verification in Step 4 needs the entire stack from Sessions 11 and 12 intact.

## Cost Notice

Estimated cost: `under $0.02`

| Service | Cost Consideration |
|---|---|
| AWS Budgets | Always Free for the first two budgets on an account |
| AWS Lambda / Amazon Bedrock / CloudWatch | Small cost from the final round of verification invocations (~$0.01 total) |

> This is the final lab in the AI Engineering track — complete the full teardown in Cleanup once finished, so nothing from Sessions 11 or 12 keeps running.

## Key Concepts

| Concept | Meaning |
|---|---|
| AWS Budgets | An account-level service that tracks actual and forecasted spending and sends threshold-based alerts; the outermost cost control, independent of anything inside the application |
| Defense in Depth for Cost | A three-layer approach: input validation rejects expensive prompts before the model is called, `maxTokens` plus a token alarm cap and monitor per-request spend, and an account budget is the hard ceiling on total monthly spending |
| Production-Readiness | A checklist spanning grounding, safety, monitoring, cost control, least-privilege access, and recoverability, used to judge whether a build is ready to operate in production |
| Responsible AI (End to End) | The integration of safety, privacy, transparency, and cost accountability across an entire system, not just one layer of it |

## Security Notes

| Topic | Explanation |
|---|---|
| Account-level backstop | The budget alerts on total account spend regardless of which application or request caused it, so it catches scenarios no single application-level control could |
| No single point of failure | The three cost layers (validation, alarm, budget) are independent; a gap in one is still caught by another |
| Least privilege reviewed end-to-end | The production-readiness review explicitly checks that IAM access for the model, the S3 bucket, and the guardrail all remain scoped, not just that they exist |
| Safe teardown order matters | Resources are removed in dependency order (budget → guardrail → S3 → CloudWatch → API Gateway → Lambda → IAM role) so that deleting one doesn't leave another referencing something that no longer exists |
| Verification after teardown | Confirming deletion with follow-up CLI calls (rather than assuming success) is good practice for any account cleanup, not just this lab |
| Budget email alerts | AWS Budgets requires a real subscriber email address; a confirmation email may need to be accepted before alerts are delivered |

## Architecture Overview

```text
                          +--------------------------------+
   User (browser/CLI)     |   API Gateway (public HTTPS)    |
        |  message  ----->|  GET = chat UI / POST = chat    |
        |                 +----------------+-----------------+
        |                                  |
        |                                  v
        |                 +--------------------------------+
        |                 |        Lambda handler           |
        |                 |  1. Input validation (11C)      |
        |                 |  2. Retrieve KB chunks (12A) <---+---- S3 knowledge base (12A)
        |                 |  3. Build grounded prompt       |
        |                 |  4. Converse + Guardrail (12B) <-+---- Bedrock Guardrail (12B)
        |                 |  5. Structured JSON logs (11C)  |
        |  <---- answer --+     + source citations          |
                          +------+-----------------+--------+
                                 |                  |
                                 v                  v
                     Bedrock Nova Micro      CloudWatch Logs --> Metric filter --> Dashboard + Token alarm
                        (11A)

   Account-level controls:  IAM least-privilege (all sessions)  +  AWS Budgets $5/mo, 80% alert (12C)
```

## Lab Steps

### Step 1: Set AWS Profile and Return to Project Folder

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Navigated back to the chatbot project folder used throughout Sessions 11 and 12.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~\Desktop\workshop-lab-11a
```

**Expected result:**

- PowerShell showed the `workshop-lab-11a` folder path, ready for the budget definition files.

**Notes:**

- All remaining commands in this lab are run from this folder.

---

### Step 2: Define the Budget

**What I did:**

- Created `budget.json` defining a $5 monthly cost budget.
- Created `notifications.json` defining an email alert that fires when actual spend exceeds 80% (that is, $4) of the budget.

**Files created:**

`budget.json`:

```json
{
    "BudgetName": "workshop-ai-monthly-budget",
    "BudgetLimit": {"Amount": "5", "Unit": "USD"},
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST"
}
```

`notifications.json`:

```json
[
    {
        "Notification": {
            "NotificationType": "ACTUAL",
            "ComparisonOperator": "GREATER_THAN",
            "Threshold": 80,
            "ThresholdType": "PERCENTAGE"
        },
        "Subscribers": [
            {"SubscriptionType": "EMAIL", "Address": "<YOUR_EMAIL>"}
        ]
    }
]
```

**Expected result:**

- Both files saved in the `workshop-lab-11a` folder, ready to be referenced in Step 3.

**What this means:**

- This configuration defines a hard $5/month ceiling for the account with a warning email once spend passes $4, well above what this entire track has cost in practice but low enough to catch any real misconfiguration quickly.

**Notes:**

- Replaced `<YOUR_EMAIL>` with my real email address before creating the budget.

---

### Step 3: Create the Budget

**What I did:**

- Created the budget and its notification subscriber in one command.
- Verified the budget exists via the CLI.
- Confirmed the budget and its 80% email alert in the AWS Billing console.
- Reviewed the resulting three-layer cost defense across the whole track.

**Commands used:**

```powershell
aws budgets create-budget --account-id <YOUR_ACCOUNT_ID> --budget file://budget.json --notifications-with-subscribers file://notifications.json
```

```powershell
aws budgets describe-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-ai-monthly-budget --query "Budget.[BudgetName,BudgetLimit.Amount,BudgetLimit.Unit]" --output text
```

**Expected result:**

- `create-budget` returned no output, indicating success.
- `describe-budget` returned: `workshop-ai-monthly-budget    5    USD`

**Console checkpoint:**

- Opened the AWS Billing console → Budgets and confirmed `workshop-ai-monthly-budget` appears with the 80% email alert configured.
- A confirmation email may need to be accepted for the alert subscription to become active.

**Three-layer cost defense summary:**

| Layer | Location | Function |
|---|---|---|
| Input validation (500 chars) | Lambda (Lab 11C) | Blocks expensive prompts before the model is ever called |
| Token spike alarm (>500 tokens) | CloudWatch (Lab 11C) | Detects per-request anomalies |
| $5 monthly budget (80% alert) | AWS Budgets (Lab 12C) | Account-level spending ceiling, independent of the application |

**What this means:**

- AWS Budgets is a global service, so no `--region` parameter is needed — this control applies to the account as a whole, not to one specific region or resource.

**Notes:**

- (confirm whether the subscription confirmation email arrived and was accepted)

---

### Step 4: Full-Stack End-to-End Verification

**What I did:**

- Ran six tests back to back, each editing `payload.json` and invoking the function, to exercise every layer built across the whole track in one pass.
- Checked that the token-spike alarm was healthy and that the AI operations dashboard reflected the new test traffic.

**Test matrix:**

| Test | Payload | Exercises | Expected Result |
|---|---|---|---|
| 1 | `{"body": "{\"message\": \"What is the codename for the capstone chatbot project?\"}"}` | RAG (Lab 12A) | Answers "Project Northstar" with citation `[project-northstar.txt]` |
| 2 | `{"body": "{\"message\": \"What is the capital of France?\"}"}` | Grounding (Lab 12A) | Returns "I don't have that in my knowledge base." |
| 3 | `{"body": "{\"message\": \"Should I put my savings into Bitcoin?\"}"}` | Guardrail — denied topic (Lab 12B) | Message blocked |
| 4 | `{"body": "{\"message\": \"My card is 4111 1111 1111 1111, tell me about EC2.\"}"}` | Guardrail — PII block (Lab 12B) | Blocked due to credit card detection |
| 5 | (the full >500-character prompt block from Lab 11C Step 3b) | Input validation (Lab 11C) | `400` response: "message is too long" |
| 6 | `{"body": "{\"message\": \"Explain what an S3 bucket is for a beginner.\"}"}` | Happy path (all layers together) | Short, grounded, on-topic answer |

**Commands used:**

```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

```powershell
aws cloudwatch describe-alarms --alarm-names ai-chatbot-token-spike --region us-east-1 --query "MetricAlarms[0].[AlarmName,StateValue]" --output text
```

**Expected result:**

- All six tests returned the behavior listed in the table above.
- The alarm check returned `ai-chatbot-token-spike    OK`.
- CloudWatch → Dashboards → `workshop-ai-chatbot-dashboard` showed invocations, latency, and token metrics from the test traffic.

**What this means:**

- Passing all six tests together is proof of a production-shaped AI application: grounded, safe, monitored, cost-capped, least-privilege, and fully observable — not just individually functional pieces.

**Notes:**

- (record the actual response for each of the six tests, and the citation/refusal/block text returned)
- Test 5 reuses the same long prompt block used to test input validation back in Lab 11C.

---

### Step 5: Production-Readiness Review

**What I did:**

- Scored the finished build against a nine-dimension production-readiness checklist, mapping each dimension to the specific control and the session it was built in.
- Reviewed what a real production deployment would still need beyond this workshop build.

**Production-readiness checklist:**

| Dimension | Control Implemented | Session |
|---|---|---|
| Works | Serverless chatbot: Lambda + API Gateway + Bedrock | 11A |
| Controllable | System prompt, temperature, maxTokens | 11B |
| Stateful | Conversation history pattern | 11B |
| Grounded | RAG over S3 knowledge base with citations | 12A |
| Honest | Refuses out-of-scope questions | 12A |
| Safe | Bedrock Guardrails: content, topics, PII, prompts | 12B |
| Cost-controlled | Input validation + maxTokens + alarm + budget | 11C / 12C |
| Observable | Structured logs, metric filters, dashboard, alarms | 11C |
| Least-privilege | Scoped IAM for model, bucket, guardrail | 11A / 12A / 12B |

**What a real production system would still need:**

- Rate limiting per user (API Gateway usage plans and keys)
- Persistent storage (DynamoDB) replacing the hand-passed conversation history
- Managed retrieval (Amazon Bedrock Knowledge Bases with a vector store) instead of the hand-built keyword-overlap retrieval
- Bedrock model-invocation logging for audit trails
- CI/CD infrastructure (CloudFormation or CDK) instead of manual CLI deploys
- Guardrail-intervention alarms built on log-based metrics, similar to the token-spike alarm

**What this means:**

- Every dimension in the checklist maps to something concretely built and tested earlier in the track, not an abstract goal — the review is really a recap of evidence already produced.

**Notes:**

- (note any dimension that felt weaker in practice than on paper, to prioritize if extending this build later)

---

### Step 6: AI Engineering Track Retrospective

**What I did:**

- Reflected on the arc of the track from Session 11A through 12C and what it set up for next steps.

**Arc of the track:**

- **Session 11A:** Built a live, serverless AI chatbot from nothing.
- **Session 11B:** Learned prompt engineering as product design, with personality and memory.
- **Session 11C:** Added operational layers — token budgets, validation, filtering, dashboards, alarms.
- **Session 12A:** Applied RAG to enable grounding and eliminate hallucinations with citations.
- **Session 12B:** Integrated Bedrock Guardrails for non-bypassable safety.
- **Session 12C:** Established account-level cost governance and verified the full system end-to-end.

**What this prepares for:**

- **Certification:** The AWS Certified AI Practitioner exam covers foundation models, prompting, RAG, responsible AI, guardrails, and cost operations — all exercised hands-on across this track.
- **Portfolio:** A live chatbot with dashboards and a documented architecture is genuine interview material, not just a tutorial completion.
- **Deeper AWS AI:** A natural next step toward Bedrock Knowledge Bases, Agents, and Amazon Q.

---

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `create-budget` fails with a validation error | `budget.json` or `notifications.json` is malformed, or the email address wasn't substituted | Re-check both files, confirm `<YOUR_EMAIL>` was replaced, and re-run the command |
| Budget not visible in the Billing console | AWS Budgets can take a few minutes to reflect a newly created budget in the console | Wait a few minutes and refresh, or re-run `describe-budget` to confirm it exists via the CLI |
| No alert email received at 80% threshold | The subscription confirmation email wasn't accepted, or spend hasn't actually crossed the threshold yet | Check for and accept the AWS Budgets subscription confirmation email; note that in this lab spend is expected to stay far under $4 |
| One of the six end-to-end tests doesn't behave as expected | A control from an earlier lab (11C, 12A, or 12B) isn't actually deployed, or environment variables were reset without re-including all of them | Re-check the specific lab's deployment steps, especially that `KB_BUCKET`, `GUARDRAIL_ID`, and `GUARDRAIL_VERSION` are all still set together |
| Alarm shows `INSUFFICIENT_DATA` instead of `OK` | Not enough recent data points for the alarm to evaluate | Run a few more invocations and wait for the metric filter to process the new logs |
| Teardown command fails with a "not found" style error | The resource was already deleted or never created in this environment | Treat this as already clean for that resource and continue with the remaining teardown steps |

## Cleanup — Complete Teardown

This removes every resource created across Sessions 11 and 12. Run from the `workshop-lab-11a` folder, replacing placeholders throughout.

### Step A: Delete the Budget

```powershell
aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-ai-monthly-budget
```

### Step B: Delete the Guardrail (Lab 12B)

```powershell
aws bedrock delete-guardrail --guardrail-identifier <YOUR_GUARDRAIL_ID> --region us-east-1
```

### Step C: Delete the S3 Knowledge Base (Lab 12A)

```powershell
aws s3 rm s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --recursive --region us-east-1
aws s3 rb s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --region us-east-1
```

### Step D: Delete the CloudWatch Dashboard, Alarm, and Metric Filter (Lab 11C)

```powershell
aws cloudwatch delete-dashboards --dashboard-names workshop-ai-chatbot-dashboard --region us-east-1
aws cloudwatch delete-alarms --alarm-names ai-chatbot-token-spike --region us-east-1
aws logs delete-metric-filter --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-name token-usage --region us-east-1
```

### Step E: Delete the API Gateway (Lab 11A)

```powershell
aws apigateway get-rest-apis --region us-east-1 --query "items[?name=='workshop-ai-chatbot'].id" --output text
```

```powershell
aws apigateway delete-rest-api --rest-api-id <YOUR_API_ID> --region us-east-1
```

### Step F: Delete the Lambda Function and Log Group (Lab 11A)

```powershell
aws lambda delete-function --function-name workshop-ai-chatbot-lab11 --region us-east-1
aws logs delete-log-group --log-group-name /aws/lambda/workshop-ai-chatbot-lab11 --region us-east-1
```

### Step G: Delete the IAM Role and Policies (Labs 11A / 12A / 12B)

```powershell
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name bedrock-invoke
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name kb-s3-read
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name guardrail-apply
aws iam detach-role-policy --role-name workshop-lab11-lambda-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-lab11-lambda-role
```

### Step H: Delete the Local Project Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force workshop-lab-11a
```

## Cleanup Verification

### Verify the Lambda Function Is Deleted

```powershell
aws lambda get-function --function-name workshop-ai-chatbot-lab11 --region us-east-1
```

**Expected result:** an error stating the function/resource was not found.

### Verify the S3 Knowledge Base Bucket Is Deleted

```powershell
aws s3 ls | Select-String "workshop-ai-kb"
```

**Expected result:**

```text
(no output)
```

### Verify the Guardrail Is Deleted

```powershell
aws bedrock list-guardrails --region us-east-1
```

**Expected result:** `workshop-ai-guardrail` no longer appears in the list.

### Verify the Budget Is Deleted

```powershell
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --query "Budgets[?BudgetName=='workshop-ai-monthly-budget']"
```

**Expected result:**

```json
[]
```

**Expected cleanup result:**

| Resource | Expected State |
|---|---|
| `workshop-ai-monthly-budget` budget | Deleted |
| `workshop-ai-guardrail` guardrail | Deleted |
| `workshop-ai-kb-<YOUR_ACCOUNT_ID>` S3 bucket and its objects | Deleted |
| `workshop-ai-chatbot-dashboard` dashboard | Deleted |
| `ai-chatbot-token-spike` alarm | Deleted |
| `token-usage` metric filter | Deleted |
| `workshop-ai-chatbot` API Gateway REST API | Deleted |
| `workshop-ai-chatbot-lab11` Lambda function | Deleted |
| `/aws/lambda/workshop-ai-chatbot-lab11` log group | Deleted |
| `bedrock-invoke`, `kb-s3-read`, `guardrail-apply` inline role policies | Deleted |
| `workshop-lab11-lambda-role` IAM role | Deleted |
| Local `workshop-lab-11a` folder | Deleted |

> Leaving an AWS account as clean as it was found is standard professional practice. The budget would have caught any missed resource through an unexpected spend alert, but deleting everything explicitly is preferable to relying on that as a safety net.

## What I Learned

- Defense in depth applies to cost exactly as it applies to security: input validation, a per-request alarm, and an account-wide budget each cover a gap the others don't.
- An account-level budget is the one control that works even if every application-level safeguard somehow fails, because it doesn't depend on the application at all.
- "Shipping" an AI application is a much higher bar than "it answers questions" — it means the system is grounded, safe, monitored, cost-capped, and least-privilege, and that all of that is provable with specific tests, not just claimed.
- Running all the individual controls from earlier labs together in one end-to-end pass is a meaningfully different (and more convincing) test than verifying each one in isolation.
- A production-readiness checklist is most useful when every row points to something concretely built and tested, not an aspirational statement.
- Tearing down a multi-service stack cleanly requires a deliberate order — some resources depend on others still existing, so deletion has to be the reverse of how the dependencies were built up.
- This track's layered build — chatbot, then prompt engineering, then operations, then RAG, then guardrails, then cost governance — mirrors how a real production AI system is actually built up over time, not assembled all at once.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/budget-notification-files-created.png` | `budget.json` and `notifications.json` in the project folder |
| `screenshots/budget-created-and-verified.png` | `describe-budget` output confirming the `$5` monthly budget (account ID redacted) |
| `screenshots/budget-console-checkpoint.png` | AWS Billing console showing `workshop-ai-monthly-budget` with the 80% email alert |
| `screenshots/test1-rag-citation-answer.png` | Response for the capstone codename question, showing the "Project Northstar" answer and citation |
| `screenshots/test2-out-of-scope-refusal.png` | Response for "What is the capital of France?" showing the knowledge-base refusal |
| `screenshots/test3-guardrail-denied-topic.png` | Response for the Bitcoin investment question showing the guardrail block |
| `screenshots/test4-guardrail-pii-block.png` | Response for the credit card message showing the PII block, with the card number not shown |
| `screenshots/test5-input-validation-400.png` | `400` response for the oversized prompt |
| `screenshots/test6-happy-path-answer.png` | Response for the S3 explainer question showing a normal grounded answer |
| `screenshots/alarm-ok-after-tests.png` | `describe-alarms` output showing `ai-chatbot-token-spike` in an `OK` state after the six tests |
| `screenshots/teardown-verification-commands.png` | Final verification commands confirming the Lambda function, S3 bucket, guardrail, and budget are all gone |