# Lab 21: Lambda with Infrastructure as Code

## Lab Summary

In this lab, I used the OpenTofu foundation created in Lab 7A to deploy a complete AWS Lambda stack through Infrastructure as Code.

I expanded the OpenTofu deploy role so it could manage Lambda functions, IAM roles, and CloudWatch log groups. I created a reusable PowerShell helper script, wrote a Python Lambda function, and defined the Lambda execution role, CloudWatch log group, role policy, and Lambda function in OpenTofu configuration files.

I then deployed the entire stack with OpenTofu, invoked the Lambda function to confirm it worked, changed the Lambda memory allocation from 128 MB to 256 MB, and applied the update in place.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 7B — Deploying a Lambda Function with Infrastructure as Code
* Session: 7 — Infrastructure as Code with OpenTofu
* Track: Solutions Architecture
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Set the active AWS CLI profile
* Verify AWS CLI authentication
* Reopen the `workshop-iac` project from Lab 7A
* Expand OpenTofu deploy role permissions
* Create a PowerShell OpenTofu helper script
* Configure PowerShell execution policy for local scripts
* Create Lambda source code
* Add the OpenTofu Archive provider
* Define a Lambda IAM execution role
* Define a CloudWatch log group with seven-day retention
* Define a Lambda logging policy
* Define a Lambda function in OpenTofu
* Recreate Lambda outputs
* Reinitialize OpenTofu
* Preview infrastructure changes
* Deploy the Lambda stack
* Invoke the Lambda function
* Verify Lambda configuration and logs
* Update Lambda memory from 128 MB to 256 MB
* Verify OpenTofu performs an in-place update
* Update `.gitignore`
* Commit Lab 7B changes to Git
* Preserve resources for Lab 7C

## Services / Tools Used

| Service / Tool         | Purpose                                                                |
| ---------------------- | ---------------------------------------------------------------------- |
| OpenTofu               | Defines and deploys AWS infrastructure declaratively                   |
| AWS Lambda             | Runs the Python greeting function                                      |
| AWS IAM                | Provides the Lambda execution role and OpenTofu deployment permissions |
| Amazon CloudWatch Logs | Stores Lambda logs with seven-day retention                            |
| AWS CLI                | Verifies identity and invokes Lambda                                   |
| Git                    | Tracks Infrastructure as Code changes                                  |
| VS Code                | Creates and edits OpenTofu and Python files                            |
| PowerShell             | Runs OpenTofu, AWS CLI, and helper-script commands                     |
| Archive Provider       | Automatically packages Lambda source code into a ZIP file              |

## Prerequisites

* Completed Lab 7A: OpenTofu Infrastructure as Code Foundation
* `workshop-iac` folder still exists on the Desktop
* S3 remote-state bucket still exists
* DynamoDB `terraform-locks` table still exists
* `workshop-tofu-deploy-role` still exists
* OpenTofu installed and available through `tofu --version`
* Git installed
* AWS CLI authenticated through SSO
* VS Code or another code editor installed

## Cost Notice

Estimated cost: `$0.00`

| Service         | Cost Consideration                             |
| --------------- | ---------------------------------------------- |
| AWS Lambda      | Expected to remain within the Lambda free tier |
| CloudWatch Logs | Small log volume with seven-day retention      |
| AWS IAM         | Free                                           |
| OpenTofu        | Free and open source                           |

## Key Concepts

| Concept                    | Meaning                                                                                         |
| -------------------------- | ----------------------------------------------------------------------------------------------- |
| Resource Block             | OpenTofu configuration that declares infrastructure to create or manage.                        |
| Data Source                | OpenTofu configuration that reads or prepares information instead of creating a cloud resource. |
| Archive Provider           | Provider used to automatically package Lambda source code into a ZIP file.                      |
| Dependency Graph           | OpenTofu automatically determines resource creation order from references in the configuration. |
| Lambda Execution Role      | IAM role assumed by Lambda while the function runs.                                             |
| In-Place Update            | Updating an existing resource without deleting and recreating it.                               |
| Declarative Infrastructure | Describing the desired end state in code.                                                       |
| Imperative Infrastructure  | Creating infrastructure through manual commands or console actions.                             |
| Helper Script              | A script that standardizes how a command is run.                                                |
| Source Code Hash           | A hash that lets OpenTofu detect Lambda source-code changes and redeploy the function.          |

## Security Notes

| Topic                 | Explanation                                                                                        |
| --------------------- | -------------------------------------------------------------------------------------------------- |
| Least privilege       | The OpenTofu deploy role is expanded only when Lambda, IAM, and CloudWatch permissions are needed. |
| Lambda execution role | The function receives only the permissions needed to write logs.                                   |
| Log retention         | CloudWatch Logs retention is set to seven days to avoid indefinite log storage growth.             |
| IAM pass role         | OpenTofu needs `iam:PassRole` to provide the Lambda execution role to AWS Lambda.                  |
| Build artifacts       | Generated ZIP files should not be committed to GitHub.                                             |
| State protection      | Remote OpenTofu state remains in the versioned S3 bucket created in Lab 7A.                        |

## Architecture Overview

```text
OpenTofu Configuration
        |
        v
OpenTofu Deploy Role
        |
        +--------------------------+
        |                          |
        v                          v
Lambda Execution Role        CloudWatch Log Group
        |                          |
        +-------------+------------+
                      |
                      v
             Lambda Function
             workshop-dev-hello
```

## Lab Steps

### Step 1: Set AWS Profile and Reopen the Project

**What I did:**

* Set my AWS CLI profile.
* Verified my AWS identity.
* Reopened the `workshop-iac` project from Lab 7A.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity

cd ~\Desktop\workshop-iac
code .
```

**Expected result:**

* AWS CLI returned the active account identity.
* VS Code opened the `workshop-iac` project.

**Notes:**

* If the SSO token expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Expand OpenTofu Deploy Role Permissions

**What I did:**

* Updated `tofu-permissions.json` from Lab 7A.
* Added permissions for Lambda functions, Lambda IAM roles, IAM pass-role actions, and CloudWatch Logs.

**File updated:**

```text
tofu-permissions.json
```

**File content:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3ManageWorkshopBuckets",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::workshop-*",
        "arn:aws:s3:::workshop-*/*"
      ]
    },
    {
      "Sid": "DynamoDBStateLock",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:<YOUR_ACCOUNT_ID>:table/terraform-locks"
    },
    {
      "Sid": "LambdaManage",
      "Effect": "Allow",
      "Action": "lambda:*",
      "Resource": "arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-*"
    },
    {
      "Sid": "IAMManageLambdaRoles",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:GetRole",
        "iam:PassRole",
        "iam:TagRole",
        "iam:ListRolePolicies",
        "iam:ListAttachedRolePolicies",
        "iam:ListInstanceProfilesForRole",
        "iam:ListRoleTags",
        "iam:PutRolePolicy",
        "iam:GetRolePolicy",
        "iam:DeleteRolePolicy"
      ],
      "Resource": "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-*"
    },
    {
      "Sid": "LogsManage",
      "Effect": "Allow",
      "Action": "logs:*",
      "Resource": "arn:aws:logs:us-east-1:<YOUR_ACCOUNT_ID>:*"
    }
  ]
}
```

**Apply the updated policy:**

```powershell
aws iam put-role-policy `
  --role-name workshop-tofu-deploy-role `
  --policy-name tofu-deploy-permissions `
  --policy-document file://tofu-permissions.json
```

**Expected result:**

```text
No output
```

**Notes:**

* Replace `<YOUR_ACCOUNT_ID>` in all four locations.
* Wait about 10 seconds for IAM permissions to propagate.

---

## Part 1: Create a Repeatable Workflow

### Step 3: Create OpenTofu Helper Script

**What I did:**

* Created a PowerShell helper script that runs OpenTofu from the correct environment folder.
* This allows commands to run from the `workshop-iac` project root.

**File created:**

```text
infra/scripts/tofu.ps1
```

**File content:**

```powershell
param(
    [Parameter(Mandatory = $true)][string]$Environment,
    [Parameter(Mandatory = $true)][string]$Action
)

$envPath = Join-Path $PSScriptRoot "..\environments\$Environment"

if (-not (Test-Path $envPath)) {
    Write-Host "Environment '$Environment' not found at $envPath"
    exit 1
}

Push-Location $envPath
tofu $Action
$exitCode = $LASTEXITCODE
Pop-Location
exit $exitCode
```

**Allow locally created scripts to run:**

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force
```

**Example helper-script command:**

```powershell
.\infra\scripts\tofu.ps1 dev plan
```

**Notes:**

* The helper script moves into the selected environment folder.
* It runs the requested OpenTofu command.
* It returns the terminal to the original project-root location afterward.

---

## Part 2: Define the Lambda Function in Code

### Step 4: Create Lambda Source Code

**What I did:**

* Created a `src` folder inside the development environment.
* Created a Python Lambda handler.

**Folder created:**

```text
infra/environments/dev/src
```

**File created:**

```text
infra/environments/dev/src/handler.py
```

**File content:**

```python
import json

def lambda_handler(event, context):
    print("Lambda invoked. Event received:", json.dumps(event))

    name = event.get("name", "world")
    message = f"Hello, {name}! This Lambda was deployed with OpenTofu."

    print(message)

    return {
        "statusCode": 200,
        "body": message
    }
```

---

### Step 5: Update `main.tf`

**What I did:**

* Replaced the development environment `main.tf`.
* Added the Archive provider.
* Defined the Lambda execution role, CloudWatch log group, log policy, and Lambda function.

**File updated:**

```text
infra/environments/dev/main.tf
```

**File content:**

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }

    archive = {
      source  = "hashicorp/archive"
      version = "~> 2.4"
    }
  }
}

provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn = var.tofu_role_arn
  }

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project
      ManagedBy   = "opentofu"
    }
  }
}

data "archive_file" "lambda_zip" {
  type        = "zip"
  source_file = "${path.module}/src/handler.py"
  output_path = "${path.module}/build/handler.zip"
}

data "aws_iam_policy_document" "lambda_assume" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "lambda" {
  name               = "workshop-${var.environment}-hello-lambda"
  assume_role_policy = data.aws_iam_policy_document.lambda_assume.json
}

resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/workshop-${var.environment}-hello"
  retention_in_days = 7
}

data "aws_iam_policy_document" "lambda_logs" {
  statement {
    actions = [
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ]

    resources = ["${aws_cloudwatch_log_group.lambda.arn}:*"]
  }
}

resource "aws_iam_role_policy" "lambda_logs" {
  name   = "cloudwatch-logs"
  role   = aws_iam_role.lambda.id
  policy = data.aws_iam_policy_document.lambda_logs.json
}

resource "aws_lambda_function" "hello" {
  function_name    = "workshop-${var.environment}-hello"
  role             = aws_iam_role.lambda.arn
  handler          = "handler.lambda_handler"
  runtime          = "python3.12"
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
  memory_size      = 128
  timeout          = 10

  depends_on = [
    aws_iam_role_policy.lambda_logs,
    aws_cloudwatch_log_group.lambda
  ]
}
```

**Notes:**

* `archive_file` packages the Python file automatically.
* `source_code_hash` allows OpenTofu to detect source-code changes.
* OpenTofu creates resources in the correct order based on resource references.
* The log group uses seven-day retention for cost control.

---

### Step 6: Create Lambda Outputs

**What I did:**

* Recreated `outputs.tf` to display the Lambda function name and ARN after deployment.

**File created:**

```text
infra/environments/dev/outputs.tf
```

**File content:**

```hcl
output "lambda_function_name" {
  description = "Name of the deployed Lambda function"
  value       = aws_lambda_function.hello.function_name
}

output "lambda_function_arn" {
  description = "ARN of the deployed Lambda function"
  value       = aws_lambda_function.hello.arn
}
```

---

## Part 3: Deploy and Test

### Step 7: Reinitialize OpenTofu

**What I did:**

* Reinitialized OpenTofu because the Archive provider was added.

**Command used:**

```powershell
cd ~\Desktop\workshop-iac
.\infra\scripts\tofu.ps1 dev init
```

**Expected result:**

```text
OpenTofu has been successfully initialized!
```

---

### Step 8: Preview Infrastructure Changes

**What I did:**

* Ran an OpenTofu plan before deployment.

**Command used:**

```powershell
.\infra\scripts\tofu.ps1 dev plan
```

**Expected result:**

```text
Plan: 4 to add, 0 to change, 0 to destroy.
```

**Expected resources:**

| Resource             | Purpose                       |
| -------------------- | ----------------------------- |
| IAM role             | Lambda execution identity     |
| CloudWatch log group | Stores Lambda logs            |
| IAM role policy      | Allows Lambda to write logs   |
| Lambda function      | Runs the Python greeting code |

---

### Step 9: Apply and Invoke the Lambda Function

#### Step 9A: Deploy Infrastructure

```powershell
.\infra\scripts\tofu.ps1 dev apply
```

**Expected result:**

```text
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

#### Step 9B: Create Invocation Payload

**File created:**

```text
infra/environments/dev/invoke.json
```

**File content:**

```json
{
  "name": "Solutions Architect"
}
```

#### Step 9C: Invoke Lambda

```powershell
cd ~\Desktop\workshop-iac\infra\environments\dev

aws lambda invoke `
  --function-name workshop-dev-hello `
  --payload file://invoke.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  response.json

Get-Content response.json
```

**Expected result:**

```json
{
  "statusCode": 200,
  "body": "Hello, Solutions Architect! This Lambda was deployed with OpenTofu."
}
```

#### Step 9D: Console Checkpoint

**What I did:**

* Opened the Lambda function in the AWS Console.
* Verified the tags applied automatically.
* Checked the CloudWatch log group.
* Confirmed the log retention period was seven days.

**Expected tags:**

```text
Environment = dev
Project = workshop-iac
ManagedBy = opentofu
```

---

## Part 4: Update Infrastructure Through Code

### Step 10: Update Lambda Memory

**What I did:**

* Changed the Lambda memory configuration in `main.tf`.

**Original line:**

```hcl
memory_size = 128
```

**Updated line:**

```hcl
memory_size = 256
```

**Preview the update:**

```powershell
cd ~\Desktop\workshop-iac
.\infra\scripts\tofu.ps1 dev plan
```

**Expected result:**

```text
Plan: 0 to add, 1 to change, 0 to destroy.
```

**Apply the update:**

```powershell
.\infra\scripts\tofu.ps1 dev apply
```

**Expected result:**

```text
Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

**What this proved:**

* OpenTofu updated the existing Lambda function in place.
* The Lambda function was not deleted and recreated.
* The function ARN and configuration remained associated with the same resource.

---

## Part 5: Commit Changes to Git

### Step 11: Update `.gitignore`

**What I did:**

* Added the generated Lambda build folder to `.gitignore`.

**Lines added:**

```gitignore
# OpenTofu build artifacts
infra/environments/*/build/
```

### Step 12: Commit Lab 7B Changes

**Commands used:**

```powershell
cd ~\Desktop\workshop-iac

git add .
git commit -m "Add Lambda function (role, log group, function) via IaC"
git log --oneline
```

**Expected result:**

* Git created a new commit containing the Lambda source code, OpenTofu configuration, helper script, outputs, and `.gitignore` update.
* Generated ZIP files, local state files, and provider folders were excluded.

## Validation / Checkpoints

| Checkpoint                             | Result |
| -------------------------------------- | ------ |
| AWS CLI profile set                    | Passed |
| AWS identity verified                  | Passed |
| `workshop-iac` project reopened        | Passed |
| Deploy role permissions expanded       | Passed |
| Updated permissions applied            | Passed |
| PowerShell helper script created       | Passed |
| PowerShell execution policy configured | Passed |
| Lambda Python handler created          | Passed |
| Archive provider added                 | Passed |
| Lambda IAM role defined                | Passed |
| CloudWatch log group defined           | Passed |
| Lambda log policy defined              | Passed |
| Lambda function defined                | Passed |
| Lambda outputs created                 | Passed |
| OpenTofu reinitialized                 | Passed |
| OpenTofu plan completed                | Passed |
| Lambda stack deployed                  | Passed |
| Lambda invocation succeeded            | Passed |
| Lambda tags verified                   | Passed |
| CloudWatch log group verified          | Passed |
| Seven-day retention verified           | Passed |
| Lambda memory updated to 256 MB        | Passed |
| In-place update verified               | Passed |
| Build folder added to `.gitignore`     | Passed |
| Git commit created                     | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                             | What It Means                                                         | How to Fix                                                                                     |
| ------------------------------------------------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| `running scripts is disabled on this system`      | PowerShell blocks local scripts by default                            | Run `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser -Force`              |
| `AccessDenied` for Lambda or IAM actions          | OpenTofu deploy-role policy was not updated or has not propagated yet | Reapply `tofu-permissions.json`, wait 10 seconds, then retry                                   |
| Lambda role cannot be assumed                     | IAM role propagation is still in progress                             | Wait 10–30 seconds, then rerun `tofu apply`                                                    |
| `tofu` is not recognized inside the script        | OpenTofu is not available through PATH                                | Reopen PowerShell and verify `tofu --version`                                                  |
| State-lock error                                  | A previous OpenTofu command may have been interrupted                 | Confirm no deployment is running, then use `tofu force-unlock <LOCK_ID>` only when appropriate |
| Lambda invocation returns `AccessDeniedException` | AWS SSO session expired                                               | Run `aws sso login --profile <YOUR_PROFILE_NAME>`                                              |
| OpenTofu plans a replacement instead of an update | A property that requires replacement was changed                      | Memory and timeout changes should be in-place; check the changed property                      |
| `response.json` cannot be found                   | Command was run outside the development environment folder            | Run `pwd`, change into `infra\environments\dev`, and invoke again                              |

## Cleanup

> Do not clean up this lab if continuing to Lab 7C.

The following resources should remain in place:

| Resource                             | Why It Stays                                      |
| ------------------------------------ | ------------------------------------------------- |
| `workshop-dev-hello` Lambda function | Lab 7C builds event-driven architecture around it |
| Lambda execution role                | Required by the Lambda function                   |
| CloudWatch log group                 | Stores Lambda logs                                |
| OpenTofu project folder              | Contains the IaC code and Git history             |
| S3 state bucket                      | Stores remote OpenTofu state                      |
| DynamoDB lock table                  | Prevents concurrent OpenTofu changes              |
| OpenTofu deploy role                 | Used by future IaC deployments                    |

### Full Cleanup Only If Stopping the IaC Track

```powershell
cd ~\Desktop\workshop-iac
.\infra\scripts\tofu.ps1 dev destroy
```

**Expected result:**

* The Lambda function, Lambda execution role, log group, and role policy are removed.
* The Lab 7A state bucket, DynamoDB table, deploy role, and local project folder remain.

## Cleanup Verification

| Resource                         | Expected State   |
| -------------------------------- | ---------------- |
| Lambda function                  | Kept for Lab 7C  |
| Lambda execution role            | Kept for Lab 7C  |
| CloudWatch log group             | Kept for Lab 7C  |
| OpenTofu development environment | Kept for Lab 7C  |
| S3 remote state bucket           | Kept from Lab 7A |
| DynamoDB lock table              | Kept from Lab 7A |
| OpenTofu deploy role             | Kept from Lab 7A |

## What I Learned

* OpenTofu can deploy a complete Lambda stack through code instead of a long sequence of CLI commands.
* Resource references allow OpenTofu to determine creation order automatically.
* The Archive provider can package Lambda source code without manual ZIP commands.
* A Lambda execution role should have only the permissions required by the function.
* CloudWatch Logs retention can be defined as Infrastructure as Code.
* `tofu plan` previews changes before cloud resources are modified.
* `tofu apply` creates or updates resources based on configuration.
* Changing Lambda memory in code results in an in-place update.
* The code is the source of truth for infrastructure configuration.
* Helper scripts make IaC workflows more repeatable and reduce navigation mistakes.
* Git preserves infrastructure history and makes changes auditable.
* Build artifacts, local state files, and provider folders should not be committed to GitHub.

## Screenshots

| Screenshot                                    | Description                                  |
| --------------------------------------------- | -------------------------------------------- |
| `screenshots/workshop-iac-reopened.png`       | Lab 7A OpenTofu project reopened             |
| `screenshots/tofu-permissions-updated.png`    | OpenTofu deploy-role permissions expanded    |
| `screenshots/tofu-helper-script-created.png`  | PowerShell helper script created             |
| `screenshots/execution-policy-configured.png` | PowerShell execution policy configured       |
| `screenshots/lambda-handler-created.png`      | Python Lambda handler created                |
| `screenshots/main-tf-updated.png`             | Lambda OpenTofu resources added to `main.tf` |
| `screenshots/outputs-tf-created.png`          | Lambda outputs created                       |
| `screenshots/tofu-init-archive-provider.png`  | OpenTofu initialized with Archive provider   |
| `screenshots/tofu-plan-four-resources.png`    | Plan showed four resources to add            |
| `screenshots/tofu-apply-success.png`          | Lambda stack deployed successfully           |
| `screenshots/lambda-invocation-success.png`   | Lambda returned greeting response            |
| `screenshots/lambda-tags-verified.png`        | Lambda tags verified in AWS Console          |
| `screenshots/log-retention-verified.png`      | CloudWatch log group retention verified      |
| `screenshots/memory-change-plan.png`          | Plan showed one in-place memory update       |
| `screenshots/memory-update-applied.png`       | Lambda memory updated to 256 MB              |
| `screenshots/gitignore-build-folder.png`      | Build artifacts added to `.gitignore`        |
| `screenshots/git-commit-created.png`          | Lab 7B Git commit created                    |