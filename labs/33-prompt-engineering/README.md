# Lab 11B: Shape Your AI — Prompt Engineering & System Prompts

## Lab Summary

In this lab, I built on the Bedrock-powered chatbot from Lab 11A by learning prompt engineering — controlling AI behavior through carefully crafted instructions instead of code changes.

Lab 11A gave the chatbot the ability to answer questions using Amazon Bedrock, but with no personality, no guardrails, and no constraints. In this lab, I added a system prompt to the `bedrock.converse()` call to give the bot a defined personality and topic boundaries, lowered `maxTokens` and adjusted `temperature` to control cost and consistency, and tested the bot with on-topic questions, off-topic questions, and an attempt to override the brevity constraint.

I then swapped in two additional system prompts — a pirate persona and a Socratic technical interviewer — to see how dramatically behavior changes with the same model, same infrastructure, and same code, just different prompt text. Finally, I implemented a `build_messages()` helper function to support conversation history, tested a multi-turn exchange, and compared token usage between a stateless request and one carrying prior conversation context.

This showed that the system prompt is effectively the product: the same underlying chatbot can become a cautious tutor, a pirate, or an interviewer purely through prompt design, and that both system prompts and conversation history have direct, measurable token-cost implications.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 11B — Shape Your AI: Prompt Engineering & System Prompts
- Session: 11 — AI Engineering
- Track: AI Engineering
- Difficulty: Intermediate
- Target certification: AWS Certified AI Practitioner

## Objectives

- Set the active AWS CLI profile and return to the Lab 11A project folder
- Verify the `workshop-ai-chatbot-lab11` Lambda function still responds correctly
- Add a system prompt to the `bedrock.converse()` call to give the bot a defined personality and constraints
- Reduce `maxTokens` and adjust `temperature` to control response cost and consistency
- Re-deploy the updated Lambda function
- Test the chatbot with an on-topic AWS question and confirm the system prompt shapes the response
- Test the chatbot with an off-topic question and confirm it redirects back to cloud topics
- Test the chatbot's resistance to a prompt attempting to override the brevity constraint
- Compare output token usage across the different test prompts
- Swap in a pirate-themed system prompt and confirm behavior changes with no code logic changes
- Swap in a "strict technical interviewer" system prompt and confirm the bot asks probing questions instead of answering directly
- Restore the tutor system prompt
- Implement a `build_messages()` helper function to support conversation history
- Update the `bedrock.converse()` call to use `build_messages()` instead of a single hardcoded message
- Test a multi-turn conversation by sending prior history alongside a new message
- Compare total token usage between a stateless request and a request with conversation history
- Understand the cost implications of system prompts and conversation history
- Clean up the Lambda function, IAM role, and log group when finished (or keep them if continuing to Lab 11C)

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Lambda | Runs the chatbot code and applies the prompt engineering changes |
| Amazon Bedrock (Nova Micro) | Generates AI responses via `bedrock.converse()`, including system prompt and history handling |
| AWS IAM | Provides Lambda execution permissions and Bedrock invoke access (reused from Lab 11A) |
| AWS CLI | Invokes the Lambda function, updates function code, reviews responses |
| PowerShell | Runs AWS CLI commands and packages Lambda code |
| Python | Implements the `handler.py` logic, including `build_messages()` |
| VS Code | Edits `handler.py` and `payload.json` |

## Prerequisites

- Completed Lab 11A: Bedrock Chatbot
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
| Amazon Bedrock (Nova Micro) | Small per-invocation inference cost across multiple test prompts (~$0.01–0.03 total) |

> Complete cleanup after this lab if not continuing to Lab 11C.

## Key Concepts

| Concept | Meaning |
|---|---|
| System Prompt | Invisible instructions sent with every request that define how the AI behaves, independent of the user's message |
| Prompt Engineering | The practice of designing prompts to control AI output quality, consistency, and cost |
| Temperature | Controls response randomness; low values (0.3–0.5) favor consistent factual answers, high values (0.7–1.0) favor creative variation |
| maxTokens | Hard limit on response length; acts as a cost backstop even when the system prompt's length instruction isn't fully followed |
| Conversation History | Prior messages sent along with a new message so the AI has context across turns |
| Multi-Turn Chat | A conversation where the AI's response accounts for earlier exchanges, not just the current message |
| Token Economics | The relationship between prompt/history length and both response cost and context window usage |

## Security Notes

| Topic | Explanation |
|---|---|
| Least privilege preserved | No new IAM permissions were required; the Lambda role's existing Bedrock invoke permission from Lab 11A was reused as-is |
| Behavior constrained via prompt | The system prompt limits the bot to cloud/AWS topics, reducing the chance it's used for unrelated or inappropriate purposes |
| maxTokens as a cost/abuse backstop | Even if the system prompt's instructions are ignored, `maxTokens` caps how much a single request can generate |
| No secrets in prompts | System prompts and payloads contain only instructional text and sample questions, never credentials or sensitive data |
| History size awareness | Longer conversation history increases token usage; unbounded history could be leveraged for cost-based abuse in production, so real deployments should cap history length |

## Architecture Overview

```text
User / AWS CLI
      |
      | invokes with message (+ optional history)
      v
Lambda Function: workshop-ai-chatbot-lab11
      |
      | build_messages(body, user_message)
      v
Amazon Bedrock — Nova Micro (bedrock.converse)
      |
      | system prompt + messages + inferenceConfig
      v
      +-------------------------------+
      |                               |
      v                               v
On-topic AWS question           Off-topic question
      |                               |
      v                               v
Short, analogy-based answer     Polite redirect to cloud topics
      |                               |
      +---------------+---------------+
                      |
                      v
        Response + token usage metadata returned
```

## Lab Steps

### Step 1: Set AWS Profile and Verify the Chatbot Still Works

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Navigated back to the chatbot project folder from Lab 11A.
- Invoked the existing chatbot to confirm it still responds before making changes.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~\Desktop\workshop-lab-11a
```

```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**Expected result:**

- PowerShell showed the `workshop-lab-11a` folder path.
- The invoke returned a response containing an AI-generated answer.

**Notes:**

- If the function didn't exist, Lab 11A Steps 3–5 needed to be re-run to recreate the role and function first.

---

### Step 2: Add a System Prompt — Give the Bot a Personality

**What I did:**

- Opened `handler.py` and located the `bedrock.converse()` call in the POST handler section.
- Added a `system` block giving the bot a defined persona: a friendly AWS cloud tutor that explains concepts with real-world analogies, stays under 3 sentences, and redirects off-topic questions back to cloud computing.
- Lowered `maxTokens` from `200` to `150` to keep responses shorter and cheaper.
- Lowered `temperature` from `0.7` to `0.5` for more consistent, factual answers.

**Code change:**

```python
response = bedrock.converse(
    modelId=MODEL_ID,
    system=[{"text": "You are a friendly AWS cloud tutor for beginners. Explain concepts in simple language with real-world analogies. Keep responses under 3 sentences. If asked something unrelated to cloud computing or AWS, politely redirect to cloud topics."}],
    messages=[
        {
            "role": "user",
            "content": [{"text": user_message}]
        }
    ],
    inferenceConfig={
        "maxTokens": 150,
        "temperature": 0.5
    }
)
```

**Deploy commands:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

**Expected result:**

- Deploy command returned `InProgress` or `Successful`.

**What this means:**

- The system prompt is invisible instruction sent with every request — it shapes tone, scope, and length without any change to the request-handling logic itself.

**Notes:**

- Verified I was in the `workshop-lab-11a` folder (`pwd`) before packaging, since the deploy command depends on the correct working directory.

---

### Step 3: Test the System Prompt

**What I did:**

- Asked an on-topic AWS question through `payload.json` (`"What is an S3 bucket?"`) and invoked the function.
- Asked an off-topic question (`"What is the best pizza topping?"`) and invoked again to confirm redirection.
- Tested resistance to a prompt attempting to override the brevity constraint (`"Explain EC2 vs Lambda in detail with 10 examples"`).
- Compared output token counts across the responses using the `metadata` field.

**Commands used:**

```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

**Expected result:**

- The S3 question returned a short, beginner-friendly answer with an analogy.
- The pizza question returned a polite redirect back to cloud topics.
- The "detail with 10 examples" question still returned a short response, staying near the 3-sentence guidance because `maxTokens` enforced the limit even though the system prompt alone might not have.

**What this means:**

- The system prompt constrains the bot in three ways at once: topic (AWS only), length (3 sentences), and style (analogies, beginner-friendly).
- `maxTokens` acts as the hard backstop for cases where the prompt's instructions aren't fully honored by the model.

**Notes:**

- No code logic changed between these three tests — only the input message did. The differences in behavior came entirely from the system prompt.

---

### Step 4: Experiment with Different Personalities

**What I did:**

- Changed only the system prompt text (leaving code logic untouched) to a pirate persona that responds in pirate speak, uses nautical metaphors, stays under 50 words, and ends every response with "Arrr!"
- Re-deployed and asked `"What is a Lambda function?"`.
- Changed the system prompt again to a strict technical interviewer persona that responds with a probing follow-up question instead of a direct answer.
- Re-deployed and asked `"What is EC2?"`.

**Deploy commands:** Same as Step 2 (`Compress-Archive` + `update-function-code`).

**Expected result:**

- The pirate persona returned a pirate-themed explanation of Lambda under 50 words, ending in "Arrr!"
- The interviewer persona returned a follow-up question instead of an answer.

**What this means:**

- Same model, same code, same infrastructure — only the prompt text changed, and the bot's entire behavior changed with it. This is the core idea behind prompt engineering as a dedicated skill.

**Notes:**

- (add anything that surprised me about how the model handled either persona, or any tweaks made to the wording to get the expected tone)

---

### Step 5: Implement Conversation History (Multi-Turn Chat)

**What I did:**

- Restored the system prompt back to the tutor version.
- Added a `build_messages()` helper function above `lambda_handler` that combines any prior `history` entries from the request body with the current message.
- Updated the `bedrock.converse()` call to use `messages=build_messages(body, user_message)` instead of a single hardcoded user message.
- Edited `payload.json` to include a two-turn history (a prior question about AWS compute services and the assistant's earlier answer) alongside a new follow-up question about storage.
- Re-deployed and invoked.

**Code added:**

```python
def build_messages(body, current_message):
    """Build message list with optional conversation history."""
    messages = []

    # Add history if provided
    history = body.get("history", [])
    for msg in history:
        messages.append({
            "role": msg["role"],
            "content": [{"text": msg["content"]}]
        })

    # Add current message
    messages.append({
        "role": "user",
        "content": [{"text": current_message}]
    })

    return messages
```

**Expected result:**

- The AI answered the storage follow-up question with awareness that it followed a prior question about compute services — demonstrating multi-turn context.

**What this means:**

- Without history, every request is independent and the model has no memory of prior turns.
- With history, each prior turn is re-sent as part of the input, which is what gives the appearance of "memory," but it comes at the cost of additional input tokens on every request.

**Notes:**

- Used `body.get("history", [])` specifically to avoid a `KeyError` when a request doesn't include history at all.

---

### Step 6: Compare Token Usage Across Strategies

**What I did:**

- Reset `payload.json` to a simple, no-history message (`"What is S3?"`) and invoked, noting `total_tokens` from the response.
- Re-invoked with the conversation-history payload from Step 5 and noted `total_tokens` again.
- Compared the two to see the direct cost impact of sending history.

**Expected result:**

- The history-included request used noticeably more input tokens than the stateless request, since the system prompt, all prior messages, and the new message were all sent together.

**What this means:**

- System prompts and conversation history both add to the token count on every single request, not just the first one.
- In production, this is why chatbots typically cap history length (e.g., last 10 messages only) rather than sending an entire conversation indefinitely.

**Notes:**

- (fill in actual token counts observed for the no-history vs. with-history requests once run)

---

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| Bot ignores the system prompt | System prompt wasn't actually included in the `bedrock.converse()` call | Verify `system=[{"text": "..."}]` is present in the call, save, and re-deploy |
| Bot still gives long responses despite "3 sentences" instruction | AI models don't always follow length instructions perfectly | Lower `maxTokens` as a hard limit (e.g., 100); treat the system prompt as a guideline and `maxTokens` as the enforced backstop |
| `KeyError: 'history'` | Code tries to access history but the payload doesn't include it | Use `body.get("history", [])` with a default empty list, as in `build_messages()` |
| Response says "context length exceeded" | History + system prompt + message together exceed the token limit | Reduce history length or shorten the system prompt |
| Higher cost than expected | Long history being re-sent with every request | Limit history to the last 5–10 messages inside `build_messages()` |
| Deploy succeeds but behavior doesn't change | Old `function.zip` was deployed instead of the updated one | Delete `function.zip`, recreate it from the current `handler.py`, and re-run `update-function-code` |
| AWS CLI authentication error | SSO session expired or profile not set | Run `aws sso login --profile <YOUR_PROFILE_NAME>` and verify with `aws sts get-caller-identity` |

## Cleanup

> If continuing to Lab 11C, keep the function and role — skip this section.

### Step 1: Delete the Lambda Function, Log Group, and Role

```powershell
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

### Verify Log Group Is Deleted

```powershell
aws logs describe-log-groups `
  --log-group-name-prefix /aws/lambda/workshop-ai-chatbot-lab11 `
  --region us-east-1 `
  --query "logGroups[].logGroupName"
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
| `workshop-ai-chatbot-lab11` Lambda function | Deleted |
| `/aws/lambda/workshop-ai-chatbot-lab11` log group | Deleted |
| `bedrock-invoke` inline role policy | Deleted |
| `workshop-lab11-lambda-role` IAM role | Deleted |
| Local `workshop-lab-11a` folder | Deleted |

## What I Learned

- The system prompt is effectively the product — changing only its text produced three completely different bots from the same model, code, and infrastructure.
- Temperature controls consistency: lower values suit factual Q&A, higher values suit creative or varied output.
- `maxTokens` is a necessary cost backstop, since a system prompt's length instruction alone isn't always fully honored by the model.
- Conversation history gives the appearance of memory, but it works by re-sending prior turns as input on every request — context comes with a direct token cost.
- Off-topic redirection through a system prompt is a practical, low-maintenance way to keep a customer-facing bot on-task without fine-tuning.
- Token usage scales predictably: no system prompt < system prompt alone < system prompt with multi-turn history.
- Prompt engineering is a distinct skill from coding — small wording changes can produce large behavioral differences.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/system-prompt-added-handler.png` | `handler.py` updated with the tutor system prompt, lowered maxTokens, and lowered temperature |
| `screenshots/system-prompt-deployed.png` | Updated Lambda code deployed successfully |
| `screenshots/ontopic-question-response.png` | Short, analogy-based answer returned for an on-topic AWS question |
| `screenshots/offtopic-question-redirect.png` | Bot politely redirected an off-topic (pizza) question back to cloud topics |
| `screenshots/override-attempt-still-brief.png` | Bot stayed brief despite a prompt attempting to request "10 examples in detail" |
| `screenshots/pirate-prompt-handler.png` | `handler.py` updated with the pirate persona system prompt |
| `screenshots/pirate-response.png` | Pirate-themed explanation of Lambda, ending in "Arrr!" |
| `screenshots/interviewer-prompt-response.png` | Technical interviewer persona responded with a probing follow-up question instead of an answer |
| `screenshots/build-messages-function-added.png` | `build_messages()` helper function added above `lambda_handler` |
| `screenshots/token-usage-no-history.png` | `total_tokens` for a stateless, no-history request |
| `screenshots/token-usage-with-history.png` | `total_tokens` for the same style of request with conversation history included, showing the increase |