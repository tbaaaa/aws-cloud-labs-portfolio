# Lab 28: CI Smoke Test

## Lab Summary

In this lab, I improved the deployment workflow for the Lab 9 Lambda function by adding a CI/CD smoke test.

In Labs 9A and 9B, the Lambda function was monitored with CloudWatch alarms and investigated with CloudWatch Logs and Logs Insights after a failure occurred. This lab focused on prevention. Instead of waiting for an alarm to show that the Lambda function is broken after deployment, I added a GitHub Actions workflow that deploys the Lambda function and immediately invokes it to confirm that it returns a healthy response.

The workflow uses GitHub OIDC to authenticate to AWS without storing long-lived AWS access keys. It packages the Lambda code, deploys the function, waits briefly for the update to complete, then runs a smoke test. The smoke test checks the Lambda response for errors and verifies that the response contains the expected healthy status.

I also tested the pipeline by pushing a good change that passed and a bad change that introduced the same type of bug from Lab 9B. The bad change caused the smoke test to fail, proving that the pipeline can catch broken code before it is merged. Finally, I created a CloudWatch dashboard to provide a single operational view of Lambda invocations, errors, duration, and error rate.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 9C — Prevent It Next Time — Add a Smoke Test to CI
- Session: 9 — Monitoring & Observability
- Track: Solutions Architecture + CI/CD
- Difficulty: Advanced
- Estimated time: 45–55 minutes
- Target certification: AWS Solutions Architect – Associate

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Reopen the `workshop-iac` repository from Session 8
- Verify GitHub CLI authentication
- Create an application folder for Lambda source code
- Verify or recreate the Lab 9 Lambda execution role
- Add the Lambda handler code to the repository
- Create a GitHub Actions workflow for deployment and smoke testing
- Configure the workflow to authenticate to AWS through GitHub OIDC
- Package the Lambda function inside the pipeline
- Deploy the Lambda function from GitHub Actions
- Invoke the Lambda function after deployment
- Verify the response contains the expected healthy status
- Grant the GitHub Actions pipeline role permission to deploy and invoke Lambda
- Commit and push the workflow
- Watch the pipeline pass with healthy code
- Create a branch containing intentionally broken Lambda code
- Open a pull request for the bad change
- Verify that the smoke test catches the failure
- Confirm the pull request is blocked by the failed workflow
- Close the bad pull request and clean up the branch
- Create a CloudWatch dashboard for operational visibility
- Document how CI/CD smoke tests, alarms, logs, and dashboards work together

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| GitHub Actions | Runs the deployment and smoke test workflow |
| GitHub OIDC | Allows GitHub Actions to authenticate to AWS without stored access keys |
| AWS IAM | Provides the GitHub pipeline role and Lambda execution role |
| AWS STS | Issues temporary credentials during the OIDC authentication flow |
| AWS Lambda | Hosts the Lab 9 API function |
| Amazon CloudWatch Metrics | Tracks Lambda invocations, errors, duration, and error rate |
| Amazon CloudWatch Dashboard | Provides a single-pane operational view of Lambda health |
| AWS CLI | Deploys, invokes, and checks AWS resources from the workflow and local terminal |
| Git | Tracks branches, commits, and pushes |
| GitHub CLI | Watches workflow runs and creates or closes pull requests |
| PowerShell | Runs local setup, Git, AWS CLI, and GitHub CLI commands |
| VS Code | Creates and edits workflow, Python, JSON, and documentation files |

## Prerequisites

- Completed Lab 8A: GitHub OIDC
- Completed Lab 8B: CI/CD Plan and Apply
- Completed Lab 8C: Drift Detection
- Completed Lab 9A: CloudWatch Alarms
- Completed Lab 9B: Diagnose with CloudWatch Logs and Log Insights
- `workshop-iac` GitHub repository exists
- `workshop-iac` local folder exists on the Desktop
- GitHub OIDC provider exists in AWS IAM
- `github-actions-infra` pipeline role exists
- `workshop-lab9-lambda-role` Lambda execution role exists or can be recreated
- AWS CLI is authenticated locally
- GitHub CLI is authenticated locally
- Git is installed and configured

## Cost Notice

Estimated cost: `$0.00`

| Service | Cost Consideration |
|---|---|
| AWS Lambda | Expected to remain within the Lambda Always Free usage for this lab |
| Amazon CloudWatch | One dashboard and small metric/log volume are expected to remain within free-tier usage |
| GitHub Actions | Expected to remain within included GitHub Actions minutes for this lab |
| AWS IAM | Free |

> The CloudWatch dashboard can be deleted during cleanup to avoid leaving unused monitoring resources behind.

## Key Concepts

| Concept | Meaning |
|---|---|
| Smoke Test | A quick automated check that confirms the application can run without immediately failing |
| Post-Deploy Verification | Testing a service immediately after deployment to confirm the deployment worked |
| CI/CD | Automation that builds, tests, deploys, and verifies changes through a pipeline |
| GitHub Actions Workflow | A YAML file stored in `.github/workflows/` that defines automated jobs |
| GitHub OIDC | Keyless authentication method that allows GitHub Actions to assume an AWS IAM role |
| Pipeline Role | IAM role assumed by GitHub Actions during workflow runs |
| Lambda Execution Role | IAM role used by Lambda while the function runs |
| Shift Left | Catching problems earlier in the delivery process before they affect users |
| Defence in Depth | Using multiple layers of controls instead of relying on one safeguard |
| Pull Request Check | A workflow result that can help block bad code before it is merged |
| CloudWatch Dashboard | A visual page showing multiple metrics in one place |
| Error Rate | A percentage showing how many invocations failed compared to total invocations |

## Security Notes

| Topic | Explanation |
|---|---|
| No stored AWS keys | The workflow uses GitHub OIDC instead of AWS access keys stored in GitHub secrets |
| Least privilege | The pipeline role receives only the Lambda deployment, invocation, and pass-role permissions needed for this lab |
| Pull request protection | A failed smoke test makes risky code visible before it is merged |
| Post-deploy testing | The workflow verifies that deployment succeeded and the function can respond correctly |
| CloudWatch visibility | The dashboard gives an operational view of health after deployment |
| Secrets in code | The Lambda code does not include hardcoded credentials or sensitive configuration |
| IAM pass role | The pipeline needs `iam:PassRole` only so it can create the Lambda function with the correct execution role |
| Dashboard cleanup | Dashboards should be deleted if they are no longer needed |

## Architecture Overview

```text
Developer pushes code or opens pull request
        |
        v
GitHub Actions Workflow
        |
        | OIDC authentication
        v
AWS IAM Role: github-actions-infra
        |
        v
Package Lambda code
        |
        v
Deploy Lambda: workshop-api-lab9
        |
        v
Smoke Test invokes Lambda
        |
        +---------------------------+
        |                           |
        v                           v
Healthy response              Error or unexpected response
Pipeline passes               Pipeline fails
        |
        v
CloudWatch Dashboard shows operational metrics
```

## Lab Steps

### Step 1: Set AWS Profile and Open the Project

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my AWS identity.
- Opened the `workshop-iac` repository from Session 8.
- Created an `app` folder for the Lambda source code.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity

cd ~\Desktop\workshop-iac
code .

New-Item -ItemType Directory -Path app -Force
```

**Expected result:**

- AWS CLI returned the active AWS account identity.
- VS Code opened the `workshop-iac` project.
- The `app` folder existed in the project root.

**Notes:**

- If the AWS SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

- If GitHub CLI needed to be checked, I used:

```powershell
gh auth status
```

---

### Step 2: Verify or Recreate the Lambda Execution Role

**What I did:**

- Checked whether the Lambda execution role from Lab 9A still existed.
- Recreated it only if it had been deleted during cleanup.

**Verification command:**

```powershell
aws iam get-role `
  --role-name workshop-lab9-lambda-role `
  --query "Role.RoleName" `
  --output text
```

**Expected result:**

```text
workshop-lab9-lambda-role
```

**If the role did not exist:**

Create the trust policy:

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

Save as:

```text
lambda-trust.json
```

Then run:

```powershell
aws iam create-role `
  --role-name workshop-lab9-lambda-role `
  --assume-role-policy-document file://lambda-trust.json

aws iam attach-role-policy `
  --role-name workshop-lab9-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**Notes:**

- The role is required because the GitHub Actions workflow deploys the Lambda function using this execution role.
- Wait about 10 seconds after creating or updating IAM roles so IAM propagation can complete.

---

### Step 3: Create Lambda Function Code in the Repository

**What I did:**

- Created a Lambda handler inside the `app` folder.
- Used healthy Lambda code that returns a successful response.
- Added structured logging for operational visibility.

**File created:**

```text
app/handler.py
```

**File content:**

```python
import json
import logging
import time

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """Workshop API endpoint."""
    start_time = time.time()

    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request received",
        "request_id": context.aws_request_id,
        "path": event.get("path", "/"),
        "method": event.get("httpMethod", "GET")
    }))

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

- The application code was now stored in the GitHub repository instead of only existing in a temporary local lab folder.

---

## Part 1: Create the CI/CD Workflow with Smoke Test

### Step 4: Create the Workflow Folder

**What I did:**

- Created the GitHub Actions workflow folder.

**PowerShell command:**

```powershell
New-Item -ItemType Directory -Path ".github\workflows" -Force
```

**Expected result:**

```text
.github/workflows/
```

**Notes:**

- GitHub only detects workflow files placed inside `.github/workflows/`.

---

### Step 5: Create Deployment and Smoke Test Workflow

**What I did:**

- Created a GitHub Actions workflow that deploys the Lambda function and then runs a smoke test.
- Configured the workflow to run on pushes and pull requests to `main`.
- Configured OIDC authentication using the `github-actions-infra` role.

**File created:**

```text
.github/workflows/deploy-and-test.yml
```

**File content:**

```yaml
name: Deploy and Smoke Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/github-actions-infra
          aws-region: us-east-1

      - name: Package Lambda
        run: |
          cd app
          zip -r ../function.zip handler.py

      - name: Deploy Lambda
        run: |
          aws lambda update-function-code \
            --function-name workshop-api-lab9 \
            --zip-file fileb://function.zip \
            --region us-east-1 \
            --query "LastUpdateStatus" \
            --output text || \
          aws lambda create-function \
            --function-name workshop-api-lab9 \
            --runtime python3.12 \
            --role arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab9-lambda-role \
            --handler handler.lambda_handler \
            --zip-file fileb://function.zip \
            --timeout 10 \
            --memory-size 128 \
            --region us-east-1

      - name: Wait for function to be active
        run: |
          echo "Waiting for function to be ready..."
          sleep 5
          echo "Function is active."

      - name: Smoke Test - Invoke and verify response
        run: |
          echo "Running smoke test..."

          aws lambda invoke \
            --function-name workshop-api-lab9 \
            --region us-east-1 \
            response.json

          echo "Response:"
          cat response.json
          echo ""

          if grep -q "errorMessage" response.json; then
            echo ""
            echo "SMOKE TEST FAILED!"
            echo "The function returned an error:"
            cat response.json | python3 -m json.tool
            echo ""
            echo "This deploy would break the application."
            echo "Fix the code before merging."
            exit 1
          fi

          if grep -q '"status": "healthy"' response.json; then
            echo ""
            echo "SMOKE TEST PASSED!"
            echo "Function responds correctly."
          else
            echo ""
            echo "SMOKE TEST FAILED!"
            echo "Unexpected response format:"
            cat response.json
            exit 1
          fi
```

**Replacements made:**

| Placeholder | Replacement |
|---|---|
| `<YOUR_ACCOUNT_ID>` | My 12-digit AWS account ID |

**Important workflow lines:**

| Line / Section | Purpose |
|---|---|
| `id-token: write` | Allows GitHub Actions to request an OIDC token |
| `role-to-assume` | Tells the workflow which AWS role to assume |
| `Package Lambda` | Creates `function.zip` from `app/handler.py` |
| `Deploy Lambda` | Updates the existing function or creates it if missing |
| `Smoke Test` | Invokes the function and verifies the response |

**Result:**

- A CI/CD workflow now existed that could deploy and test the Lambda function automatically.

---

## Part 2: Give the Pipeline Lambda Permissions

### Step 6: Create Pipeline Lambda Permissions

**What I did:**

- Created a permissions policy allowing the GitHub Actions pipeline role to update, create, inspect, and invoke the Lab 9 Lambda function.
- Added `iam:PassRole` so the workflow can create the Lambda function using the Lambda execution role.

**File created:**

```text
lambda-pipeline-permissions.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "LambdaDeploy",
            "Effect": "Allow",
            "Action": [
                "lambda:UpdateFunctionCode",
                "lambda:CreateFunction",
                "lambda:GetFunction",
                "lambda:InvokeFunction",
                "lambda:GetFunctionConfiguration"
            ],
            "Resource": "arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-api-lab9"
        },
        {
            "Sid": "PassRoleToLambda",
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lab9-lambda-role"
        }
    ]
}
```

**Replacements made:**

| Placeholder | Replacement |
|---|---|
| `<YOUR_ACCOUNT_ID>` | My 12-digit AWS account ID |

**Attach policy command:**

```powershell
aws iam put-role-policy `
  --role-name github-actions-infra `
  --policy-name lambda-deploy-permissions `
  --policy-document file://lambda-pipeline-permissions.json
```

**Expected result:**

```text
No output
```

**Notes:**

- No output from the AWS CLI usually means the inline policy was attached successfully.
- This policy is added to the pipeline role from Lab 8A.

---

### Step 7: Commit and Push the Healthy Workflow

**What I did:**

- Added the new Lambda code, workflow file, and permissions file to Git.
- Committed the changes.
- Pushed the changes to GitHub.

**Commands used:**

```powershell
git add .
git commit -m "Add Lambda deploy pipeline with smoke test"
git push
```

**Expected result:**

- GitHub Actions started a new workflow run.
- The workflow deployed the Lambda function.
- The smoke test passed.

**Watch the workflow:**

```powershell
gh run watch
```

**Expected smoke test output:**

```text
SMOKE TEST PASSED!
Function responds correctly.
```

**Alternative check:**

- Opened the GitHub repository in the browser.
- Went to the **Actions** tab.
- Opened the **Deploy and Smoke Test** workflow run.
- Confirmed all steps passed.

---

## Part 3: Push Bad Code and Prove the Pipeline Catches It

### Step 8: Create a Bad Change Branch

**What I did:**

- Created a new branch named `bad-deploy`.
- Edited the Lambda handler to introduce a broken code change.

**Create branch:**

```powershell
git checkout -b bad-deploy
```

**Broken code added inside `app/handler.py`:**

```python
    # Someone added a database connection but forgot to define the variable
    connection_string = database_url
```

**Example placement:**

```python
    logger.info(json.dumps({
        "level": "INFO",
        "message": "Request received",
        "request_id": context.aws_request_id,
        "path": event.get("path", "/"),
        "method": event.get("httpMethod", "GET")
    }))

    # Someone added a database connection but forgot to define the variable
    connection_string = database_url

    result = {
        "status": "healthy",
        "message": "Hello from the workshop API!"
    }
```

**Commit and push bad branch:**

```powershell
git add .
git commit -m "Add database connection"
git push -u origin bad-deploy
```

**Result:**

- The bad branch was pushed to GitHub.

---

### Step 9: Open a Pull Request and Watch the Smoke Test Fail

**What I did:**

- Opened a pull request from `bad-deploy` into `main`.
- Watched the GitHub Actions workflow run against the pull request.
- Verified that the smoke test failed.

**Create pull request:**

```powershell
gh pr create --title "Add database connection" --body "Adding database connectivity to the API"
```

**Watch workflow:**

```powershell
gh run watch
```

**Expected failure output:**

```text
SMOKE TEST FAILED!
The function returned an error:
{
    "errorMessage": "name 'database_url' is not defined",
    "errorType": "NameError"
}
This deploy would break the application.
Fix the code before merging.
```

**Console checkpoint:**

- Opened the GitHub repository.
- Opened the **Pull requests** tab.
- Clicked the **Add database connection** pull request.
- Confirmed the workflow check showed a red failure.
- Opened **Details** to view the smoke test output.

**What this proved:**

- The same type of bug from Lab 9B was caught before being merged.
- The pull request was blocked by the failing smoke test.
- The pipeline shifted detection earlier in the development process.

---

### Step 10: Clean Up the Bad Branch

**What I did:**

- Closed the bad pull request.
- Deleted the bad branch.
- Returned to `main`.

**Commands used:**

```powershell
gh pr close bad-deploy --delete-branch
git checkout main
```

**If GitHub CLI did not close the PR:**

- Closed the pull request manually in GitHub.
- Then ran:

```powershell
git checkout main
git branch -D bad-deploy
```

**Result:**

- The bad code was not merged.
- The repository returned to the healthy `main` branch.

---

## Part 4: Create a CloudWatch Dashboard

### Step 11: Create Dashboard Definition

**What I did:**

- Created a CloudWatch dashboard definition file.
- Added widgets for invocations, errors, duration, and error rate.

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
                "title": "Invocations",
                "metrics": [
                    ["AWS/Lambda", "Invocations", "FunctionName", "workshop-api-lab9"]
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
                "title": "Errors",
                "metrics": [
                    ["AWS/Lambda", "Errors", "FunctionName", "workshop-api-lab9"]
                ],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1",
                "annotations": {
                    "horizontal": [
                        {
                            "label": "Alarm threshold",
                            "value": 1,
                            "color": "#d13212"
                        }
                    ]
                }
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Duration (ms)",
                "metrics": [
                    ["AWS/Lambda", "Duration", "FunctionName", "workshop-api-lab9"]
                ],
                "period": 60,
                "stat": "Average",
                "region": "us-east-1"
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "title": "Error Rate (%)",
                "metrics": [
                    [{"expression": "(errors / invocations) * 100", "label": "Error Rate", "id": "rate"}],
                    ["AWS/Lambda", "Errors", "FunctionName", "workshop-api-lab9", {"id": "errors", "visible": false}],
                    ["AWS/Lambda", "Invocations", "FunctionName", "workshop-api-lab9", {"id": "invocations", "visible": false}]
                ],
                "period": 60,
                "stat": "Sum",
                "region": "us-east-1"
            }
        }
    ]
}
```

**Notes:**

- The dashboard provides a quick operational view of Lambda health.
- The Errors widget includes a threshold line at `1` to match the alarm threshold from Lab 9A.

---

### Step 12: Create the CloudWatch Dashboard

**What I did:**

- Created the CloudWatch dashboard using the dashboard definition file.

**Command used:**

```powershell
aws cloudwatch put-dashboard `
  --dashboard-name workshop-lab9-overview `
  --dashboard-body file://dashboard.json `
  --region us-east-1
```

**Expected result:**

```json
{
    "DashboardValidationMessages": []
}
```

**Console checkpoint:**

- Opened the CloudWatch console.
- Clicked **Dashboards**.
- Opened `workshop-lab9-overview`.
- Verified four panels:
  - Invocations
  - Errors
  - Duration
  - Error Rate

**Result:**

- A CloudWatch dashboard was created for the Lab 9 Lambda function.

---

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| `workshop-iac` repository opened | Passed |
| `app` folder created | Passed |
| GitHub CLI authentication verified | Passed |
| Lambda execution role verified or recreated | Passed |
| Healthy Lambda handler created in repo | Passed |
| GitHub Actions workflow folder created | Passed |
| Deploy and smoke test workflow created | Passed |
| OIDC role ARN added to workflow | Passed |
| Lambda pipeline permissions file created | Passed |
| Lambda permissions attached to `github-actions-infra` role | Passed |
| Healthy workflow committed and pushed | Passed |
| Pipeline deployed Lambda successfully | Passed |
| Smoke test passed with healthy code | Passed |
| `bad-deploy` branch created | Passed |
| Broken code added intentionally | Passed |
| Bad branch pushed | Passed |
| Pull request opened | Passed |
| Smoke test failed on broken code | Passed |
| Pull request blocked by failed check | Passed |
| Bad pull request closed | Passed |
| Bad branch deleted or abandoned | Passed |
| CloudWatch dashboard definition created | Passed |
| CloudWatch dashboard deployed | Passed |
| Dashboard panels verified | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `gh auth status` fails | GitHub CLI is not authenticated | Run `gh auth login` and authenticate through the browser |
| `gh` is not recognized | GitHub CLI is not in PATH or terminal was not reopened after installation | Reopen PowerShell or run GitHub CLI directly from its install path |
| Pipeline fails at `Configure AWS credentials` | GitHub OIDC trust policy does not match the repo or `id-token: write` is missing | Check the OIDC provider, role trust policy, and workflow permissions |
| `Not authorized to perform lambda:UpdateFunctionCode` | Pipeline role does not have Lambda deployment permissions | Attach `lambda-deploy-permissions` to `github-actions-infra` |
| `Not authorized to perform iam:PassRole` | Pipeline role cannot pass the Lambda execution role | Confirm `PassRoleToLambda` statement uses the correct role ARN |
| `No such file or directory: function.zip` | Package step failed or `app/handler.py` does not exist | Confirm `app/handler.py` exists and is committed |
| `Function not found` during update | Lambda function does not exist yet | The workflow should fall back to `create-function`; confirm the execution role exists |
| `NoSuchEntity` for `workshop-lab9-lambda-role` | Lambda role was deleted after Lab 9A | Recreate the role using Step 2 |
| Smoke test passes even though code was changed | The Lambda update may not have completed before invocation | Increase the wait time in the workflow and rerun |
| Smoke test fails unexpectedly | Lambda response does not include the expected healthy status | Check `response.json` and confirm `handler.py` returns `status: healthy` |
| Pull request workflow does not start | Workflow file may not exist on `main` yet | Push the workflow to `main` first, then create the pull request |
| `gh pr create` fails | Branch was not pushed or repo remote is not configured | Run `git push -u origin bad-deploy` and check `git remote -v` |
| Dashboard shows no data | Lambda has not been invoked recently | Run the workflow or invoke the function manually to generate metrics |
| Dashboard validation returns errors | JSON syntax or widget configuration is invalid | Validate `dashboard.json` and rerun `put-dashboard` |

## Cleanup

Complete cleanup only if I am done with the Lab 9 application and do not want to keep the CI smoke test project running.

### Step 1: Delete the CloudWatch Dashboard

```powershell
aws cloudwatch delete-dashboards `
  --dashboard-names workshop-lab9-overview `
  --region us-east-1
```

### Step 2: Remove Lambda Deploy Permissions from the Pipeline Role

```powershell
aws iam delete-role-policy `
  --role-name github-actions-infra `
  --policy-name lambda-deploy-permissions
```

### Step 3: Delete the Lambda Function

```powershell
aws lambda delete-function `
  --function-name workshop-api-lab9 `
  --region us-east-1
```

### Step 4: Delete the CloudWatch Log Group

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-api-lab9 `
  --region us-east-1
```

### Step 5: Delete the Lambda Execution Role

```powershell
aws iam detach-role-policy `
  --role-name workshop-lab9-lambda-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam delete-role `
  --role-name workshop-lab9-lambda-role
```

### Step 6: Delete Alarm and SNS Topic If They Still Exist

```powershell
aws cloudwatch delete-alarms `
  --alarm-names workshop-lab9-errors `
  --region us-east-1

aws sns delete-topic `
  --topic-arn arn:aws:sns:us-east-1:<YOUR_ACCOUNT_ID>:workshop-lab9-alerts `
  --region us-east-1
```

## Cleanup Verification

### Verify Dashboard Is Deleted

```powershell
aws cloudwatch list-dashboards `
  --region us-east-1 `
  --query "DashboardEntries[?DashboardName=='workshop-lab9-overview'].DashboardName"
```

**Expected result:**

```json
[]
```

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

### Verify Lambda Log Group Is Deleted

```powershell
aws logs describe-log-groups `
  --log-group-name-prefix /aws/lambda/workshop-api-lab9 `
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
  --role-name workshop-lab9-lambda-role
```

**Expected result:**

```text
NoSuchEntity
```

**Expected cleanup result:**

| Resource | Expected State |
|---|---|
| `workshop-lab9-overview` dashboard | Deleted if fully cleaning up |
| `lambda-deploy-permissions` inline policy | Deleted if fully cleaning up |
| `workshop-api-lab9` Lambda function | Deleted if fully cleaning up |
| `/aws/lambda/workshop-api-lab9` log group | Deleted if fully cleaning up |
| `workshop-lab9-lambda-role` IAM role | Deleted if fully cleaning up |
| `workshop-lab9-errors` alarm | Deleted if still present |
| `workshop-lab9-alerts` SNS topic | Deleted if still present |
| `workshop-iac` GitHub repository | Kept as portfolio evidence unless intentionally removed |

## What I Learned

- Monitoring tells me when an application breaks, but smoke tests can help prevent broken code from reaching production.
- A smoke test does not need to test every feature. It only needs to confirm the most important basic behavior works.
- GitHub Actions can package, deploy, and verify a Lambda function automatically.
- GitHub OIDC allows CI/CD workflows to authenticate to AWS without stored AWS access keys.
- The pipeline role needs specific Lambda and `iam:PassRole` permissions to deploy this function.
- A post-deploy smoke test shifts failure detection earlier in the delivery process.
- The same `database_url` bug from Lab 9B can be caught automatically by invoking the function in CI.
- Failed pull request checks are useful because they stop bad changes before merge.
- CloudWatch dashboards provide a single-pane view of application health.
- Defence in depth means using multiple layers: smoke tests for prevention, alarms for detection, logs for diagnosis, and dashboards for visibility.
- Session 9 shows a full operational loop: alert, diagnose, fix, prevent, and observe.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/workshop-iac-opened.png` | `workshop-iac` repository opened |
| `screenshots/app-folder-created.png` | `app` folder created |
| `screenshots/github-cli-authenticated.png` | GitHub CLI authentication verified |
| `screenshots/lambda-role-verified.png` | Lambda execution role verified or recreated |
| `screenshots/handler-created.png` | Healthy Lambda handler created in the repo |
| `screenshots/workflow-folder-created.png` | `.github/workflows` folder created |
| `screenshots/deploy-and-test-workflow-created.png` | Deploy and smoke test workflow created |
| `screenshots/lambda-pipeline-permissions-created.png` | Lambda pipeline permissions file created |
| `screenshots/lambda-permissions-attached.png` | Permissions attached to `github-actions-infra` |
| `screenshots/healthy-workflow-committed.png` | Healthy workflow committed and pushed |
| `screenshots/actions-healthy-run-started.png` | GitHub Actions workflow started |
| `screenshots/smoke-test-passed.png` | Smoke test passed with healthy code |
| `screenshots/bad-deploy-branch-created.png` | `bad-deploy` branch created |
| `screenshots/broken-code-added.png` | Broken `database_url` code added |
| `screenshots/bad-branch-pushed.png` | Bad branch pushed to GitHub |
| `screenshots/pull-request-created.png` | Pull request opened for bad code |
| `screenshots/smoke-test-failed.png` | Smoke test failed on broken code |
| `screenshots/pr-blocked-by-check.png` | Pull request blocked by failed workflow check |
| `screenshots/bad-pr-closed.png` | Bad pull request closed |
| `screenshots/dashboard-json-created.png` | CloudWatch dashboard definition created |
| `screenshots/dashboard-created.png` | CloudWatch dashboard created successfully |
| `screenshots/dashboard-panels-verified.png` | Dashboard panels verified in CloudWatch |
| `screenshots/cleanup-verified.png` | Cleanup verified if resources were removed |