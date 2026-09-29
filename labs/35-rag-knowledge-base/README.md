# Lab 12A: Give Your Chatbot a Brain — Retrieval-Augmented Generation (RAG)

## Lab Summary

In this lab, I fixed the biggest weakness of the chatbot I built across Labs 11A–11C: it only knows what the underlying model was trained on. Ask it about private or newer information and it either admits it doesn't know or, worse, confidently makes something up (a hallucination).

To fix this, I implemented Retrieval-Augmented Generation (RAG) by hand. I stored three small knowledge documents in an S3 bucket, gave the Lambda function a tightly scoped read-only IAM permission for that bucket, and added retrieval logic to `handler.py` that loads the documents, splits them into paragraph-sized chunks, scores each chunk against the user's question by keyword overlap, and injects the best matches into the system prompt as context. The prompt instructs the model to answer only from that context, cite the source filename, and reply "I don't have that in my knowledge base." when the answer isn't there.

I then proved it works with three tests: a question answerable only from my knowledge base (a made-up project codename the model could never have been trained on), a question answerable from a different document, and an out-of-scope question (the capital of France) that the bot correctly refused to answer. Finally, I checked the CloudWatch logs to confirm each request's retrieval was observable through a `rag_retrieval` event.

This lab showed that RAG is how production assistants answer questions about private or changing data without retraining a model: update a file in S3 instead of the model, and ground every answer in a source.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 12A — Give Your Chatbot a Brain: Retrieval-Augmented Generation (RAG)
- Session: 12 — AI Engineering (Capstone)
- Track: AI Engineering
- Difficulty: Beginner → Intermediate
- Target certification: AWS Certified AI Practitioner

## Objectives

- Set the active AWS CLI profile and return to the Lab 11A project folder
- Create three knowledge documents locally, including one containing a made-up fact that exists only in my files
- Create a globally unique S3 bucket for the knowledge base and upload the three documents
- Verify the uploaded documents in the S3 console
- Create a least-privilege IAM policy allowing the Lambda role to read only the knowledge base bucket
- Attach the policy to the Lambda execution role as an inline policy (`kb-s3-read`)
- Back up the end-of-Lab-11C `handler.py` before making changes
- Add an S3 client, a `KB_BUCKET` environment variable lookup, and a warm-start cache to `handler.py`
- Implement `load_knowledge_base()` to load and chunk documents from S3
- Implement `retrieve_context()` to score chunks by keyword overlap and return the top matches
- Build a grounded system prompt that restricts the model to the retrieved context and requires source citations
- Surface the retrieved sources in the response metadata
- Re-deploy the Lambda function and set the `KB_BUCKET` environment variable
- Ask a question answerable only from the knowledge base and confirm the bot answers with a citation
- Ask a question answerable from a different document and confirm the citation changes
- Ask an out-of-scope question and confirm the bot refuses instead of hallucinating
- Review the `rag_retrieval` events in CloudWatch Logs
- Understand RAG vs fine-tuning trade-offs and how this hand-built version maps to Amazon Bedrock Knowledge Bases

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| Amazon S3 | Stores the knowledge base documents that the chatbot retrieves from |
| AWS Lambda | Runs the chatbot, including the retrieval, chunking, and prompt-building logic |
| Amazon Bedrock (Nova Micro) | Generates answers grounded in the retrieved context |
| AWS IAM | Grants the Lambda role scoped, read-only access to the knowledge base bucket |
| Amazon CloudWatch Logs | Records `rag_retrieval` events showing chunks retrieved and sources used |
| AWS CLI | Creates the bucket, uploads documents, attaches the policy, deploys code, sets configuration, and invokes the function |
| PowerShell | Runs AWS CLI commands and packages Lambda code |
| Python | Implements the retrieval and prompt-grounding logic in `handler.py` |
| VS Code | Edits `handler.py`, `payload.json`, `kb-s3-policy.json`, and the knowledge documents |

## Prerequisites

- Completed Labs 11A, 11B, and 11C
- `workshop-ai-chatbot-lab11` Lambda function and `workshop-lab11-lambda-role` IAM role still exist
- `handler.py` already contains the Lab 11B system prompt and Lab 11C input validation
- The `workshop-lab-11a` project folder exists with `handler.py` and `payload.json`
- AWS CLI installed and authenticated
- PowerShell available on Windows
- VS Code or another text editor installed

> If Session 11 resources were cleaned up, follow Lab 11A Steps 3–5 to recreate the role and function, then re-apply the Lab 11B system prompt before starting here.

## Cost Notice

Estimated cost: `under $0.05`

| Service | Cost Consideration |
|---|---|
| Amazon S3 | Expected to remain within Always Free usage (5 GB) for three small text files |
| AWS Lambda | Expected to remain within Always Free usage for this lab |
| Amazon Bedrock (Nova Micro) | Small per-invocation inference cost (~$0.01–0.03 total); slightly more input tokens per request because retrieved context is injected into the prompt |

> Labs 12B and 12C build on the resources created here, so cleanup is optional at the end of this lab. See the Cleanup section for both paths.

## Key Concepts

| Concept | Meaning |
|---|---|
| RAG (Retrieval-Augmented Generation) | Retrieving relevant text from a knowledge source at question time and adding it to the prompt so the model generates a grounded answer: Retrieve → Augment → Generate |
| Grounding | Forcing the model to base its answer on provided facts rather than its training data, so it can cite sources and admit when the answer isn't there |
| Hallucination | When a model confidently states something false; RAG reduces this by instructing the model to answer only from supplied context |
| Chunking | Splitting documents into small passages so only the relevant piece is retrieved and injected, instead of sending whole documents on every request |
| Retrieval | Finding the chunks most relevant to the question; this lab uses a simple keyword-overlap score, while production systems typically use vector embeddings that match meaning rather than exact words |
| RAG vs Fine-Tuning | Fine-tuning retrains the model (expensive, slow, hard to update); RAG only changes what is sent in the prompt (cheap, instant, and updated by editing a file) |
| Warm-Start Caching | A module-level cache (`_KB_CACHE`) that persists between invocations while the Lambda stays warm, so the knowledge base is downloaded from S3 once per cold start |
| Environment Variables | Configuration that changes between environments (here, `KB_BUCKET`) is kept outside the code, so the same file works anywhere |

## Security Notes

| Topic | Explanation |
|---|---|
| Least privilege | The new inline policy grants only `s3:GetObject` and `s3:ListBucket`, and only on the knowledge base bucket, so the function can read its knowledge and nothing else in S3 |
| Knowledge base contents are exposed to the chatbot's users | Anything stored in the knowledge base can be repeated back by the bot to anyone who can ask the right question, so real secrets, credentials, or sensitive data should never be stored there |
| Fictional demo secret | `project-northstar.txt` contains a made-up "password" line that exists purely as test data to prove retrieval works; it is not a real credential |
| Control who can write to the bucket | Retrieved text is placed directly into the prompt, so anyone able to modify knowledge documents can influence what the model says; write access to the bucket should be restricted |
| Grounding as a safety control | Instructing the model to answer only from context and to refuse otherwise reduces hallucination and stops the bot from confidently inventing policy or facts |
| Configuration outside code | The bucket name is supplied through the `KB_BUCKET` environment variable rather than hard-coded |
| Existing controls still apply | The Lab 11C input validation (500-character limit) and the `maxTokens` limit still run on every request, alongside the new retrieval step |

## Architecture Overview

```text
Knowledge documents (workshop-faq.txt, aws-glossary.txt, project-northstar.txt)
      |
      | aws s3 cp --recursive
      v
Amazon S3 bucket: workshop-ai-kb-<YOUR_ACCOUNT_ID>
      ^
      | s3:GetObject / s3:ListBucket (inline policy: kb-s3-read)
      |
User / AWS CLI
      |
      | invokes with question
      v
Lambda Function: workshop-ai-chatbot-lab11
      |
      v
Input validation (Lab 11C)
      |
      v
load_knowledge_base()  ---> cached across warm invocations (_KB_CACHE)
      |
      v
retrieve_context(question)  ---> top 3 chunks by keyword overlap
      |
      v
Grounded system prompt = rules + "=== CONTEXT ===" + retrieved chunks
      |
      v
Amazon Bedrock — Nova Micro (bedrock.converse)
      |
      +-------------------------------+--------------------------------+
      |                               |                                |
      v                               v                                v
In knowledge base             Different document               Not in knowledge base
      |                               |                                |
      v                               v                                v
Answer + [project-              Answer + [workshop-              "I don't have that in
northstar.txt] citation         faq.txt] citation                my knowledge base."
      |
      v
CloudWatch Logs: rag_retrieval event (chunks_retrieved, sources)
```

## Lab Steps

### Step 1: Set AWS Profile and Return to Project Folder

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Navigated back to the chatbot project folder used in Labs 11A–11C.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
cd ~\Desktop\workshop-lab-11a
```

**Expected result:**

- PowerShell showed the `workshop-lab-11a` folder path, ready for the new files.

**Notes:**

- All remaining commands in this lab are run from this folder.

---

### Step 2: Create a Knowledge Base in S3

**What I did:**

- Created a local `knowledge-base` folder.
- Created three text documents inside it: a workshop FAQ, an AWS mini-glossary, and a project codes file that contains a made-up fact the model could not know from training.
- Created a globally unique S3 bucket using my account ID in the name.
- Uploaded all three documents to the bucket.
- Confirmed the three files appear in the S3 console.

**Commands used:**

```powershell
mkdir knowledge-base
```

```powershell
aws s3 mb s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --region us-east-1
```

```powershell
aws s3 cp knowledge-base s3://workshop-ai-kb-<YOUR_ACCOUNT_ID>/ --recursive --region us-east-1
```

**Files created (in `knowledge-base/`):**

`workshop-faq.txt`:

```text
AI Cloud Fusion Workshop — Frequently Asked Questions

The AI Cloud Fusion workshop meets every Wednesday afternoon at 2:00 PM Atlantic Standard Time.
Each session includes a lecture, three hands-on labs (Beginner, Intermediate, Advanced),
and a career preparation segment.

Recorded sessions focus on lecture and Beginner lab and are then posted in the Microsoft Teams channel within 24 hours.

The recommended certification path is: AWS Solutions Architect - Associate first,
then AWS Certified AI Practitioner, then a specialty certification of your choice.
```

`aws-glossary.txt`:

```text
AWS Mini-Glossary for Beginners

S3 (Simple Storage Service): object storage for files. Think of it as an infinite hard drive in the cloud.
Lambda: serverless compute. Your code runs on demand without you managing any servers.
Bedrock: a managed service for calling foundation models (AI models) through one API.
CloudWatch: AWS's monitoring service for logs, metrics, dashboards, and alarms.
IAM: Identity and Access Management. Controls who and what can do what in your account.
API Gateway: creates public HTTPS endpoints that trigger your Lambda functions.
```

`project-northstar.txt` (all values are fictional test data):

```text
AI Cloud Fusion — Internal Project Codes

The internal codename for the workshop's capstone chatbot project is "Project Northstar".
The support escalation password for facilitators is: FUSION-2024-CLOUD.
The workshop's cloud budget alert email is sent whenever spend exceeds 80 percent of the monthly limit.
```

**Expected result:**

- `mb` returned `make_bucket: workshop-ai-kb-<YOUR_ACCOUNT_ID>`.
- `cp --recursive` printed three `upload:` lines, one per file.
- The S3 console showed `workshop-faq.txt`, `aws-glossary.txt`, and `project-northstar.txt` in the bucket.

**What this means:**

- The knowledge lives in S3 as plain files, so it can be updated at any time without touching the model. The made-up "Project Northstar" fact exists only in my file, so if the bot can later state it, the answer must have come from retrieval rather than the model's memory.

**Notes:**

- S3 bucket names are globally unique across all of AWS, which is why the account ID is part of the name.
- On Windows, Notepad needs "Save as type" set to "All Files" to avoid saving `workshop-faq.txt.txt`.
- The `password` line in `project-northstar.txt` is fictional demo data supplied by the lab, not a real credential.

---

### Step 3: Let Lambda Read the Knowledge Base (IAM)

**What I did:**

- Created `kb-s3-policy.json` with a policy allowing `s3:GetObject` and `s3:ListBucket` on only the knowledge base bucket and its objects.
- Attached the policy to the Lambda execution role as an inline policy named `kb-s3-read`.

**File created:** `kb-s3-policy.json`

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": ["s3:GetObject", "s3:ListBucket"],
            "Resource": [
                "arn:aws:s3:::workshop-ai-kb-<YOUR_ACCOUNT_ID>",
                "arn:aws:s3:::workshop-ai-kb-<YOUR_ACCOUNT_ID>/*"
            ]
        }
    ]
}
```

**Commands used:**

```powershell
aws iam put-role-policy --role-name workshop-lab11-lambda-role --policy-name kb-s3-read --policy-document file://kb-s3-policy.json --region us-east-1
```

**Expected result:**

- No output, indicating the policy was attached successfully.

**What this means:**

- The function can now read files from its knowledge base bucket and nothing else, which is the least-privilege principle applied to an AI workload.

**Notes:**

- The policy is passed as a `file://` reference instead of inline JSON because PowerShell mangles quotes in inline `--policy-document` values (this produces an `Unknown options: Version...` error). This is the same approach used to create IAM policies in Lab 11A.
- Both `<YOUR_ACCOUNT_ID>` placeholders in the policy needed replacing with the real account ID.

---

### Step 4: Add Retrieval to the Lambda

**What I did:**

- Backed up my end-of-Lab-11C `handler.py` as `handler-11c-backup.py` so I could restore it later.
- Added the `os` and `re` imports, an S3 client, the `KB_BUCKET` environment variable lookup, and a module-level `_KB_CACHE` near the top of `handler.py`.
- Added two helper functions above `lambda_handler`: `load_knowledge_base()` and `retrieve_context()`.
- Added a RAG block inside `lambda_handler`, right before the `# Call Bedrock` comment, that retrieves context, builds the source list, logs a `rag_retrieval` event, and assembles a grounded system prompt.
- Replaced the Lab 11B `system=[...]` line in the `bedrock.converse()` call with `system=[{"text": system_prompt}]`.
- Added a `sources` field to the response metadata.

**Commands used:**

```powershell
Copy-Item handler.py handler-11c-backup.py
```

**Code added — top of file:**

```python
import os
import re

s3 = boto3.client("s3", region_name="us-east-1")
KB_BUCKET = os.environ.get("KB_BUCKET", "")

# Cache the knowledge base across warm invocations (loaded once per cold start)
_KB_CACHE = None
```

**Code added — helper functions (above `lambda_handler`):**

```python
def load_knowledge_base():
    """Load and cache all KB chunks from S3 (runs once per cold start)."""
    global _KB_CACHE
    if _KB_CACHE is not None:
        return _KB_CACHE

    chunks = []
    if KB_BUCKET:
        listing = s3.list_objects_v2(Bucket=KB_BUCKET)
        for obj in listing.get("Contents", []):
            key = obj["Key"]
            text = s3.get_object(Bucket=KB_BUCKET, Key=key)["Body"].read().decode("utf-8")
            # Split each document into paragraph-sized chunks (blank line = new chunk)
            for para in re.split(r"\n\s*\n", text):
                para = para.strip()
                if para:
                    chunks.append({"source": key, "text": para})

    _KB_CACHE = chunks
    return chunks

def retrieve_context(query, top_k=3):
    """Return the top_k KB chunks most relevant to the query (keyword overlap)."""
    chunks = load_knowledge_base()
    query_words = set(re.findall(r"[a-z0-9]+", query.lower()))

    scored = []
    for chunk in chunks:
        chunk_words = set(re.findall(r"[a-z0-9]+", chunk["text"].lower()))
        overlap = len(query_words & chunk_words)
        if overlap > 0:
            scored.append((overlap, chunk))

    scored.sort(key=lambda item: item[0], reverse=True)
    return [chunk for _score, chunk in scored[:top_k]]
```

**Code added — inside `lambda_handler`, before `# Call Bedrock`:**

```python
# --- RAG: retrieve relevant knowledge and build a grounded prompt ---
retrieved = retrieve_context(user_message)
if retrieved:
    context_block = "\n\n".join(
        f"[Source: {c['source']}]\n{c['text']}" for c in retrieved
    )
    sources = sorted({c["source"] for c in retrieved})
else:
    context_block = "(no relevant documents found)"
    sources = []

logger.info(json.dumps({
    "level": "INFO",
    "event": "rag_retrieval",
    "request_id": request_id,
    "chunks_retrieved": len(retrieved),
    "sources": sources
}))

system_prompt = (
    "You are the AI Cloud Fusion workshop assistant. "
    "Answer the user's question using ONLY the context provided below. "
    "If the answer is not in the context, reply exactly: "
    "\"I don't have that in my knowledge base.\" "
    "Never invent information. When you use the context, cite the source "
    "filename in square brackets, e.g. [workshop-faq.txt].\n\n"
    "=== CONTEXT ===\n" + context_block + "\n=== END CONTEXT ==="
)
```

**Code changed — `bedrock.converse()` call and response metadata:**

```python
system=[{"text": system_prompt}],
```

```python
"sources": sources,
```

**Expected result:**

- `handler.py` saved with the retrieval helpers, the RAG block, the grounded system prompt, and the `sources` metadata field.

**What this means:**

- The retrieval step does the "R" in RAG: it scores every chunk by how many words it shares with the question and returns the best matches. Keyword overlap is simple, but the retrieve-then-inject flow is identical to production systems that use semantic embeddings.
- The grounded prompt is what turns retrieval into a safeguard: the model is told to use only the supplied context, cite sources, and refuse when the answer isn't there.
- `_KB_CACHE` lives outside the handler, so the knowledge base is downloaded from S3 once per cold start and reused while the function stays warm.

**Notes:**

- Python indentation matters here: the RAG block sits directly in the `lambda_handler` body (4 spaces), lined up with the `logger.info(` and `# Call Bedrock` lines around it, not inside the `try:` block.
- Placing retrieval before the `try:` block also means `api_latency_ms` measures only the Bedrock call, not the S3 lookup.
- (add anything that needed fixing during editing, for example an indentation error or a misplaced block)

---

### Step 5: Deploy and Point the Function at the Bucket

**What I did:**

- Re-packaged and deployed the updated `handler.py` after confirming I was in the `workshop-lab-11a` folder.
- Set the `KB_BUCKET` environment variable on the function so the code knows which bucket holds the knowledge base.

**Commands used:**

```powershell
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

```powershell
aws lambda update-function-configuration --function-name workshop-ai-chatbot-lab11 --environment "Variables={KB_BUCKET=workshop-ai-kb-<YOUR_ACCOUNT_ID>}" --region us-east-1 --query "LastUpdateStatus" --output text
```

**Expected result:**

- Both commands returned `InProgress` or `Successful`.

**What this means:**

- Configuration that differs between environments belongs outside the code. The same `handler.py` can run in dev, test, or prod by changing only the `KB_BUCKET` value.

**Notes:**

- If the second command runs while the first update is still in progress, wait a few seconds and retry.
- (note the status each command returned)

---

### Step 6: Prove RAG Works

**What I did:**

- Asked a question that only my knowledge base can answer (the capstone project codename) and confirmed the bot answered from the retrieved document with a citation.
- Asked a question covered by a different document (workshop schedule and certification path) and confirmed the citation changed.
- Asked an out-of-scope question (`"What is the capital of France?"`) and confirmed the bot refused instead of answering from its training data.
- Queried CloudWatch Logs for the `rag_retrieval` events to confirm retrieval is observable.

**Test payloads (`payload.json`):**

```json
{"body": "{\"message\": \"What is the codename for the capstone chatbot project?\"}"}
```

```json
{"body": "{\"message\": \"When does the workshop meet and what certification should I get first?\"}"}
```

```json
{"body": "{\"message\": \"What is the capital of France?\"}"}
```

**Commands used:**

```powershell
aws lambda invoke --function-name workshop-ai-chatbot-lab11 --region us-east-1 --cli-binary-format raw-in-base64-out --payload file://payload.json response.json; Get-Content response.json
```

```powershell
aws logs filter-log-events --log-group-name "/aws/lambda/workshop-ai-chatbot-lab11" --filter-pattern "rag_retrieval" --region us-east-1 --query "events[-3:].message" --output text
```

**Expected result:**

- Codename question: the bot answered "Project Northstar" and cited `[project-northstar.txt]`.
- Schedule and certification question: the bot answered "Wednesday afternoon at 2:00 PM Atlantic Standard Time" and "AWS Solutions Architect – Associate first", citing `[workshop-faq.txt]`.
- Capital of France question: the bot replied "I don't have that in my knowledge base." instead of "Paris".
- The log query returned `rag_retrieval` JSON lines showing `chunks_retrieved` and the `sources` list for recent questions.

**What this means:**

- In-knowledge-base questions get a grounded answer with a traceable citation, while out-of-scope questions get an honest refusal instead of a hallucination. The model and infrastructure are unchanged from Lab 11C; the only new ingredient is retrieved context.
- Retrieval is observable in the same way as everything monitored in Session 11, since each request writes a structured `rag_retrieval` event.

**Notes:**

- (record the actual responses returned, including whether the citation format matched `[filename]`)
- (note the `chunks_retrieved` value and `sources` shown in the log lines)
- If a knowledge file is edited later, the function has to be re-deployed to force a cold start, because a warm Lambda keeps serving the cached copy.

---

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| Bot says "I don't have that" for everything | `KB_BUCKET` is not set, or the bucket is empty | Re-run the `update-function-configuration` command in Step 5 and confirm the files uploaded in Step 2 |
| `AccessDenied` in the logs | The IAM S3 policy is missing or contains the wrong account ID | Check `kb-s3-policy.json` has the real account ID in both places, then re-run the `put-role-policy` command in Step 3 |
| Answers ignore the knowledge base | Old code is still deployed | Confirm the Step 4 edits are saved, re-zip `handler.py`, and re-deploy |
| `NoSuchBucket` | Bucket name typo or wrong region | The bucket must be named `workshop-ai-kb-<YOUR_ACCOUNT_ID>` and be in `us-east-1` |
| Citation filename missing from the answer | The model didn't follow the prompt format | Lower `temperature` (for example to 0.3) and confirm the `[Source: ...]` format is present in `context_block` |
| Changes to a knowledge file aren't reflected | The old knowledge base is still cached in a warm Lambda | Re-upload the file to S3, then re-deploy the function to force a cold start |
| `Unknown options: Version...` when attaching the policy | PowerShell mangled the quotes in inline JSON | Pass the policy as `--policy-document file://kb-s3-policy.json` |
| Knowledge file saved as `.txt.txt` | Notepad added a second extension | Set "Save as type" to "All Files" when saving |
| Syntax or indentation error after editing `handler.py` | Python treats indentation as syntax; the RAG block was pasted at the wrong level | Keep the RAG block at the `lambda_handler` body level (4 spaces), or restore the whole file and redo the edit |

## Cleanup

Cleanup depends on what comes next.

### Path A — Continuing to Lab 12B

Keep everything: the function, role, S3 bucket, knowledge base, and updated `handler.py`. Labs 12B and 12C build directly on all of it, so no cleanup is needed at this point.

### Path B — Full reset back to the end of Lab 11C

Use this to remove everything Lab 12A created and return the chatbot to its end-of-Lab-11C state. Run from the `workshop-lab-11a` folder.

### Step 1: Delete the S3 Knowledge Base (Objects, Then the Bucket)

```powershell
aws s3 rm s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --recursive --region us-east-1
aws s3 rb s3://workshop-ai-kb-<YOUR_ACCOUNT_ID> --region us-east-1
```

### Step 2: Remove the S3 Read Permission from the Role

```powershell
aws iam delete-role-policy --role-name workshop-lab11-lambda-role --policy-name kb-s3-read
```

### Step 3: Clear the `KB_BUCKET` Environment Variable

```powershell
aws lambda update-function-configuration --function-name workshop-ai-chatbot-lab11 --environment "Variables={}" --region us-east-1 --query "LastUpdateStatus" --output text
```

### Step 4: Restore the Lambda Code to the Lab 11C Version

```powershell
Copy-Item handler-11c-backup.py handler.py -Force
Compress-Archive -Path handler.py -DestinationPath function.zip -Force
aws lambda update-function-code --function-name workshop-ai-chatbot-lab11 --zip-file fileb://function.zip --region us-east-1 --query "LastUpdateStatus" --output text
```

### Step 5: Remove the Local Lab 12A Working Files

```powershell
Remove-Item -Recurse -Force knowledge-base; Remove-Item -Force kb-s3-policy.json
```

> The complete teardown of the whole AI stack (Lambda, role, API Gateway, and all Session 11 resources) is handled in Lab 12C.

## Cleanup Verification

Only needed if Path B was followed.

### Verify the Knowledge Base Bucket Is Deleted

```powershell
aws s3 ls | Select-String "workshop-ai-kb"
```

**Expected result:**

```text
(no output)
```

### Verify the S3 Read Policy Is Removed from the Role

```powershell
aws iam list-role-policies --role-name workshop-lab11-lambda-role --query "PolicyNames"
```

**Expected result:** only `bedrock-invoke` is listed; `kb-s3-read` is gone.

### Verify the Environment Variable Is Cleared

```powershell
aws lambda get-function-configuration --function-name workshop-ai-chatbot-lab11 --region us-east-1 --query "Environment" --output text
```

**Expected result:**

```text
None
```

**Expected cleanup result (Path B):**

| Resource | Expected State |
|---|---|
| `workshop-ai-kb-<YOUR_ACCOUNT_ID>` S3 bucket and its objects | Deleted |
| `kb-s3-read` inline role policy | Deleted |
| `KB_BUCKET` Lambda environment variable | Cleared |
| Lambda code | Restored to the Lab 11C version |
| Local `knowledge-base` folder and `kb-s3-policy.json` | Deleted |

Invoking the chatbot with a normal question such as "What is S3?" should answer directly again, exactly as at the end of Lab 11C.

## What I Learned

- RAG beats relying on model memory for private or changing facts: to update what the bot knows, I edit a file in S3 instead of retraining a model.
- Grounding is a powerful hallucination control. Telling the model to answer only from the supplied context, and to refuse otherwise, made it decline a question it obviously could have answered from training.
- Citations make answers traceable, which builds trust and supports auditability.
- Retrieval is essentially search plus injection. The keyword-overlap version I built shows the same flow that embedding-based vector search uses in production.
- RAG vs fine-tuning is a classic trade-off: RAG is cheaper, faster to update, and better suited to facts that change.
- What I built by hand is what Amazon Bedrock Knowledge Bases offers as a managed service, automating chunking, embeddings, vector storage, and retrieval.
- Anything placed in a knowledge base can be repeated by the chatbot, so access to the documents (and to the bucket) needs to be treated as part of the security design.
- Caching data outside the handler function is a practical serverless optimisation, but it also means edits to source files need a cold start before they show up.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/knowledge-base-documents-created.png` | The local `knowledge-base` folder containing the three knowledge documents |
| `screenshots/s3-bucket-created-and-uploaded.png` | Successful `mb` and `cp --recursive` output showing the bucket created and three files uploaded (account ID redacted) |
| `screenshots/s3-console-knowledge-base.png` | S3 console showing the three documents in the knowledge base bucket (account ID redacted) |
| `screenshots/kb-s3-policy-attached.png` | `kb-s3-policy.json` and the successful `put-role-policy` command scoped to the knowledge base bucket (account ID redacted) |
| `screenshots/handler-backup-created.png` | The `handler-11c-backup.py` backup created before editing |
| `screenshots/handler-retrieval-functions-1.png` | `handler.py` showing the new imports, cache |
| `screenshots/handler-retrieval-functions-2.png` | `load_knowledge_base()`, and `retrieve_context()` |
| `screenshots/handler-rag-block-grounded-prompt.png` | `handler.py` showing the RAG block, the grounded system prompt, and the updated `bedrock.converse()` call |
| `screenshots/lambda-deployed.png` | Successful `update-function-code` output for the updated function |
| `screenshots/kb-bucket-env-var-set.png` | `update-function-configuration` output or the Lambda console Environment variables tab showing `KB_BUCKET` (account ID redacted) |
| `screenshots/rag-codename-answer.png` | Response answering "Project Northstar" with a `[project-northstar.txt]` citation |
| `screenshots/rag-faq-answer.png` | Response for the schedule and certification question citing `[workshop-faq.txt]` |
| `screenshots/rag-out-of-scope-refusal.png` | Response for "What is the capital of France?" showing the "I don't have that in my knowledge base." refusal |
| `screenshots/rag-retrieval-cloudwatch-logs.png` | `rag_retrieval` log lines showing `chunks_retrieved` and `sources` for recent requests |