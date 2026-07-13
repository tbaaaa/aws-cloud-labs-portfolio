# Lab 22: Event-Driven Architecture with Reusable Modules

## Lab Summary

In this lab, I used OpenTofu to refactor my Lambda infrastructure into a reusable module and then built an event-driven architecture where uploading a file to an S3 bucket automatically triggers a Lambda function.

This lab built directly on Lab 7B. In Lab 7B, I deployed a standalone Lambda function with Infrastructure as Code. In this lab, I moved the Lambda function, IAM role, IAM policy, and CloudWatch log group into a reusable OpenTofu module under the `infra/modules/lambda` folder. I then updated the development environment to call that module and added S3 event notification wiring.

The final architecture included an S3 upload bucket, a Lambda processor function, permission for S3 to invoke the Lambda function, and an S3 bucket notification that triggers Lambda on object upload.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 7C — Event-Driven Architecture with Reusable Modules
* Session: 7 — Infrastructure as Code with OpenTofu
* Track: Solutions Architecture
* Difficulty: Advanced
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Set the active AWS CLI profile
* Verify AWS CLI authentication
* Reopen the `workshop-iac` project from Labs 7A and 7B
* Create a reusable Lambda module under `infra/modules/lambda`
* Define module input variables
* Define module Lambda resources
* Define module outputs
* Update the Lambda handler to process S3 events
* Rewrite the development environment to use the Lambda module
* Create an S3 upload bucket through OpenTofu
* Grant S3 permission to invoke the Lambda function
* Configure S3 bucket notifications for object-created events
* Reinitialize OpenTofu after adding the module
* Preview the refactor and event-driven architecture with `tofu plan`
* Deploy the architecture with `tofu apply`
* Upload a file to S3 to trigger the Lambda function
* Verify the Lambda ran automatically by checking CloudWatch Logs
* Verify the architecture in the AWS Console
* Destroy the event-driven application resources with OpenTofu
* Commit the reusable module and event-driven architecture code to Git

## Services / Tools Used

| Service / Tool         | Purpose                                                                    |
| ---------------------- | -------------------------------------------------------------------------- |
| OpenTofu               | Defines, deploys, updates, and destroys infrastructure through code        |
| OpenTofu Modules       | Reusable Infrastructure as Code building blocks                            |
| Amazon S3              | Stores uploaded files and sends object-created events                      |
| AWS Lambda             | Processes S3 upload events                                                 |
| AWS IAM                | Provides the Lambda execution role and cross-service invocation permission |
| Amazon CloudWatch Logs | Stores Lambda execution logs                                               |
| AWS CLI                | Uploads test files and checks logs                                         |
| Git                    | Tracks IaC code changes                                                    |
| VS Code                | Edits OpenTofu and Python files                                            |
| PowerShell             | Runs OpenTofu, AWS CLI, and Git commands                                   |

## Prerequisites

* Completed Lab 7A: OpenTofu Infrastructure as Code Foundation
* Completed Lab 7B: Lambda with Infrastructure as Code
* `workshop-iac` folder still exists on the Desktop
* S3 remote-state bucket still exists
* DynamoDB `terraform-locks` table still exists
* `workshop-tofu-deploy-role` still exists
* OpenTofu installed and available through `tofu --version`
* AWS CLI authenticated through SSO
* Git installed
* VS Code or another code editor installed

## Cost Notice

Estimated cost: `$0.00`

| Service         | Cost Consideration                                       |
| --------------- | -------------------------------------------------------- |
| AWS Lambda      | Expected to remain within the Lambda free tier           |
| Amazon S3       | Very small test file upload; expected cost is negligible |
| CloudWatch Logs | Small log volume with seven-day retention                |
| AWS IAM         | Free                                                     |
| OpenTofu        | Free and open source                                     |

## Key Concepts

| Concept                      | Meaning                                                                                 |
| ---------------------------- | --------------------------------------------------------------------------------------- |
| OpenTofu Module              | A reusable package of infrastructure resources that can be called from environments     |
| Module Input                 | A variable passed into a module to customize its behavior                               |
| Module Output                | A value returned by a module, such as a Lambda function ARN                             |
| Event-Driven Architecture    | A design where one service reacts automatically to events from another service          |
| S3 Event Notification        | S3 configuration that sends events when actions such as object uploads occur            |
| `aws_lambda_permission`      | OpenTofu resource that gives another AWS service permission to invoke a Lambda function |
| `aws_s3_bucket_notification` | OpenTofu resource that wires an S3 bucket event to a target such as Lambda              |
| `force_destroy`              | S3 bucket setting that allows OpenTofu to delete a bucket even if it contains objects   |
| Refactoring                  | Restructuring code without changing the overall purpose of the architecture             |
| Dependency Management        | OpenTofu determining the correct creation and deletion order from resource references   |

## Security Notes

| Topic                                | Explanation                                                                    |
| ------------------------------------ | ------------------------------------------------------------------------------ |
| Cross-service permission is required | S3 cannot invoke Lambda until `aws_lambda_permission` explicitly allows it     |
| Source ARN restriction               | The Lambda permission is scoped to the specific S3 bucket ARN                  |
| Least privilege                      | The Lambda execution role only needs CloudWatch Logs permissions for this lab  |
| Module consistency                   | Reusable modules help teams apply the same secure baseline across environments |
| Log retention                        | The module keeps CloudWatch Logs retention at seven days                       |
| Build artifacts                      | Module build folders should be ignored by Git and not committed                |
| State protection                     | The remote OpenTofu state from Lab 7A remains the source of truth              |

## Architecture Overview

```text
S3 Upload Bucket
      |
      | ObjectCreated event
      v
S3 Bucket Notification
      |
      v
Lambda Permission
      |
      v
Lambda Processor Function
      |
      v
CloudWatch Logs
```

## Lab Steps

### Step 1: Set AWS Profile and Reopen the Project

**What I did:**

* Set my AWS CLI profile for the current PowerShell session.
* Verified my active AWS identity.
* Reopened the `workshop-iac` project from Labs 7A and 7B.

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

* If OpenTofu cannot find AWS credentials, I made sure `AWS_PROFILE` was set in the same terminal used to run `tofu`.

---

## Part 1: Build a Reusable Lambda Module

### Step 2: Create Lambda Module Folder

**What I did:**

* Created a reusable Lambda module folder under `infra/modules`.

**Folder created:**

```text
infra/modules/lambda
```

**PowerShell command:**

```powershell
mkdir infra\modules\lambda
```

**Result:**

* The module folder was ready for `variables.tf`, `main.tf`, and `outputs.tf`.

---

### Step 3: Create Module Variables

**What I did:**

* Created the module input variables.

**File created:**

```text
infra/modules/lambda/variables.tf
```

**File content:**

```hcl
variable "function_name" {
  description = "Name of the Lambda function"
  type        = string
}

variable "source_file" {
  description = "Path to the Lambda source .py file"
  type        = string
}

variable "handler" {
  description = "Lambda handler in file.function format"
  type        = string
  default     = "handler.lambda_handler"
}

variable "memory_size" {
  description = "Memory in MB"
  type        = number
  default     = 128
}

variable "retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 7
}
```

**Result:**

* The module could now accept function name, source file, handler, memory size, and log retention values.

---

### Step 4: Create Module Resources

**What I did:**

* Created the reusable module resources.
* The module packages Lambda source code, creates the Lambda execution role, creates the CloudWatch log group, attaches logging permissions, and deploys the Lambda function.

**File created:**

```text
infra/modules/lambda/main.tf
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

data "archive_file" "zip" {
  type        = "zip"
  source_file = var.source_file
  output_path = "${path.module}/build/${var.function_name}.zip"
}

data "aws_iam_policy_document" "assume" {
  statement {
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "this" {
  name               = "${var.function_name}-role"
  assume_role_policy = data.aws_iam_policy_document.assume.json
}

resource "aws_cloudwatch_log_group" "this" {
  name              = "/aws/lambda/${var.function_name}"
  retention_in_days = var.retention_days
}

data "aws_iam_policy_document" "logs" {
  statement {
    actions = [
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ]

    resources = ["${aws_cloudwatch_log_group.this.arn}:*"]
  }
}

resource "aws_iam_role_policy" "logs" {
  name   = "cloudwatch-logs"
  role   = aws_iam_role.this.id
  policy = data.aws_iam_policy_document.logs.json
}

resource "aws_lambda_function" "this" {
  function_name    = var.function_name
  role             = aws_iam_role.this.arn
  handler          = var.handler
  runtime          = "python3.12"
  filename         = data.archive_file.zip.output_path
  source_code_hash = data.archive_file.zip.output_base64sha256
  memory_size      = var.memory_size
  timeout          = 10

  depends_on = [
    aws_iam_role_policy.logs,
    aws_cloudwatch_log_group.this
  ]
}
```

**Result:**

* The Lambda infrastructure was now defined once as a reusable building block.

---

### Step 5: Create Module Outputs

**What I did:**

* Created outputs so environments can reference the Lambda function name and ARN.

**File created:**

```text
infra/modules/lambda/outputs.tf
```

**File content:**

```hcl
output "function_name" {
  value = aws_lambda_function.this.function_name
}

output "function_arn" {
  value = aws_lambda_function.this.arn
}
```

**Result:**

* The calling environment can now use `module.processor.function_name` and `module.processor.function_arn`.

---

## Part 2: Compose the Event-Driven Architecture

### Step 6: Update Lambda Code to Handle S3 Events

**What I did:**

* Updated the Lambda handler from Lab 7B.
* Added logic to detect S3 event records and log the uploaded bucket and object key.

**File updated:**

```text
infra/environments/dev/src/handler.py
```

**File content:**

```python
import json

def lambda_handler(event, context):
    print("Lambda invoked. Event received:", json.dumps(event))

    records = event.get("Records", [])

    if records:
        for record in records:
            bucket = record["s3"]["bucket"]["name"]
            key = record["s3"]["object"]["key"]
            print(f"New file uploaded to S3: bucket={bucket}, key={key}")

        return {
            "statusCode": 200,
            "body": f"Processed {len(records)} S3 event(s)."
        }

    name = event.get("name", "world")
    message = f"Hello, {name}! This Lambda was deployed with OpenTofu."

    print(message)

    return {
        "statusCode": 200,
        "body": message
    }
```

**Result:**

* The function can still handle direct invocations.
* It can now also process S3 upload events.

---

### Step 7: Rewrite Development `main.tf`

**What I did:**

* Rewrote the development environment to call the reusable Lambda module.
* Created an S3 upload bucket.
* Granted S3 permission to invoke the Lambda.
* Configured the S3 bucket notification.

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

data "aws_caller_identity" "current" {}

module "processor" {
  source = "../../modules/lambda"

  function_name  = "workshop-${var.environment}-processor"
  source_file    = "${path.root}/src/handler.py"
  memory_size    = 128
  retention_days = 7
}

resource "aws_s3_bucket" "uploads" {
  bucket        = "workshop-${var.environment}-uploads-<YOUR_INITIALS>"
  force_destroy = true
}

resource "aws_lambda_permission" "allow_s3" {
  statement_id  = "AllowS3Invoke"
  action        = "lambda:InvokeFunction"
  function_name = module.processor.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = aws_s3_bucket.uploads.arn
}

resource "aws_s3_bucket_notification" "uploads" {
  bucket = aws_s3_bucket.uploads.id

  lambda_function {
    lambda_function_arn = module.processor.function_arn
    events              = ["s3:ObjectCreated:*"]
  }

  depends_on = [
    aws_lambda_permission.allow_s3
  ]
}
```

**Important note:**

* Replace `<YOUR_INITIALS>` with unique lowercase initials or another unique suffix.
* Example:

```hcl
bucket = "workshop-dev-uploads-ar"
```

**Result:**

* The dev environment now uses a module instead of standalone Lambda resources.
* The S3 upload bucket is wired to trigger the Lambda automatically.

---

### Step 8: Update Outputs

**What I did:**

* Updated `outputs.tf` to show the Lambda function name and upload bucket name.

**File updated:**

```text
infra/environments/dev/outputs.tf
```

**File content:**

```hcl
output "lambda_function_name" {
  description = "Name of the deployed Lambda function"
  value       = module.processor.function_name
}

output "uploads_bucket" {
  description = "Name of the S3 bucket that triggers the Lambda"
  value       = aws_s3_bucket.uploads.bucket
}
```

**Result:**

* After `tofu apply`, OpenTofu will display the Lambda function name and S3 upload bucket.

---

## Part 3: Deploy and Test the Event-Driven Architecture

### Step 9: Reinitialize OpenTofu

**What I did:**

* Reinitialized OpenTofu because a module was added.

**Command used from project root:**

```powershell
cd ~\Desktop\workshop-iac
.\infra\scripts\tofu.ps1 dev init
```

**Expected result:**

```text
OpenTofu has been successfully initialized!
```

**Notes:**

* If `tofu` is not recognized, I added OpenTofu to the current terminal path:

```powershell
$env:Path += ";C:\OpenTofu"
```

---

### Step 10: Preview the Plan

**What I did:**

* Ran `tofu plan` to preview the refactor and new event-driven resources.

**Command used:**

```powershell
.\infra\scripts\tofu.ps1 dev plan
```

**Expected result:**

* The plan should include resources from `module.processor`.
* The plan should include an S3 upload bucket.
* The plan should include Lambda permission for S3.
* The plan should include an S3 bucket notification.

**Expected note if Lab 7B resources still exist:**

* OpenTofu may destroy or replace the old standalone `workshop-dev-hello` Lambda resources.
* This is expected because the architecture is being refactored from standalone resources into a reusable module.

**Result:**

* The plan completed successfully.

---

### Step 11: Apply the Architecture

**What I did:**

* Applied the OpenTofu configuration.
* Confirmed the deployment by typing `yes`.

**Command used:**

```powershell
.\infra\scripts\tofu.ps1 dev apply
```

**Expected result:**

```text
Apply complete!
```

**Expected outputs:**

```text
lambda_function_name = "workshop-dev-processor"
uploads_bucket       = "workshop-dev-uploads-<YOUR_INITIALS>"
```

**Result:**

* Lambda processor function was deployed.
* S3 upload bucket was created.
* S3-to-Lambda event notification was configured.

---

### Step 12: Trigger Lambda by Uploading a File

#### Step 12A: Create and Upload Test File

**What I did:**

* Created a local test file.
* Uploaded it to the S3 upload bucket.
* This upload should automatically trigger the Lambda function.

**PowerShell command:**

```powershell
"This file should trigger my Lambda." | Set-Content -Path "$env:TEMP\test-upload.txt"

aws s3 cp "$env:TEMP\test-upload.txt" s3://<UPLOADS_BUCKET>/test-upload.txt
```

**Example:**

```powershell
aws s3 cp "$env:TEMP\test-upload.txt" s3://workshop-dev-uploads-ar/test-upload.txt
```

**Expected result:**

```text
upload: ... to s3://<UPLOADS_BUCKET>/test-upload.txt
```

**Result:**

* File uploaded successfully to S3.

---

### Step 13: Check Lambda Logs

**What I did:**

* Waited about 10–30 seconds.
* Retrieved the latest Lambda log stream.
* Checked the log events to prove the Lambda function ran automatically.

**Commands used:**

```powershell
$stream = aws logs describe-log-streams `
  --log-group-name "/aws/lambda/workshop-dev-processor" `
  --order-by LastEventTime `
  --descending `
  --max-items 1 `
  --query "logStreams[0].logStreamName" `
  --output text `
  --region us-east-1
```

```powershell
aws logs get-log-events `
  --log-group-name "/aws/lambda/workshop-dev-processor" `
  --log-stream-name $stream `
  --query "events[].message" `
  --output text `
  --region us-east-1
```

**Expected log message:**

```text
New file uploaded to S3: bucket=<UPLOADS_BUCKET>, key=test-upload.txt
```

**Result:**

* CloudWatch Logs showed that the Lambda function ran automatically after the S3 upload.

**What this proved:**

* I did not manually invoke the Lambda.
* The S3 object upload triggered the function.
* The event-driven architecture worked.

---

### Step 14: Console Checkpoint

**What I did:**

* Opened the AWS Console.
* Verified the Lambda function.
* Verified the S3 bucket and uploaded file.
* Verified the IAM role created by the module.
* Verified CloudWatch Logs.

**Checks performed:**

| Console Area       | What I Verified                                                              |
| ------------------ | ---------------------------------------------------------------------------- |
| Lambda             | `workshop-dev-processor` exists                                              |
| Lambda Monitor tab | Invocation activity appeared after the file upload                           |
| S3                 | Upload bucket contains `test-upload.txt`                                     |
| IAM Roles          | Lambda role named `workshop-dev-processor-role` exists                       |
| CloudWatch Logs    | Log group `/aws/lambda/workshop-dev-processor` contains the upload event log |

---

## Part 4: Destroy the Event-Driven Application

### Step 15: Destroy the Architecture

**What I did:**

* Used OpenTofu to destroy the event-driven application resources.
* Confirmed the destroy by typing `yes`.

**Command used from project root:**

```powershell
cd ~\Desktop\workshop-iac
.\infra\scripts\tofu.ps1 dev destroy
```

**Expected result:**

```text
Destroy complete!
```

**Resources removed:**

| Resource                  | Purpose                                |
| ------------------------- | -------------------------------------- |
| S3 bucket notification    | Removed the S3-to-Lambda wiring        |
| Lambda permission         | Removed S3 permission to invoke Lambda |
| S3 upload bucket          | Removed upload bucket and test file    |
| Lambda processor function | Removed event processor                |
| Lambda execution role     | Removed function role                  |
| CloudWatch log group      | Removed function logs                  |

**Notes:**

* `force_destroy = true` allows OpenTofu to delete the S3 bucket even if the test file remains inside it.
* The Lab 7A foundation should remain.

---

## Part 5: Commit Changes to Git

### Step 16: Update `.gitignore`

**What I did:**

* Confirmed module build artifacts are ignored.

**Lines added or confirmed in `.gitignore`:**

```gitignore
infra/environments/*/build/
infra/modules/*/build/
```

**Result:**

* Generated ZIP files are not committed to GitHub.

---

### Step 17: Commit Lab 7C Changes

**What I did:**

* Added the reusable module and event-driven architecture changes to Git.
* Created a commit.

**Commands used:**

```powershell
cd ~\Desktop\workshop-iac

git add .
git commit -m "Add Lambda module and event-driven S3 trigger architecture"
git log --oneline
```

**Expected result:**

* Git created a new commit for the reusable module and S3-to-Lambda architecture.

---

## Validation / Checkpoints

| Checkpoint                             | Result |
| -------------------------------------- | ------ |
| AWS CLI profile set                    | Passed |
| AWS identity verified                  | Passed |
| `workshop-iac` project reopened        | Passed |
| Lambda module folder created           | Passed |
| Module variables created               | Passed |
| Module resources created               | Passed |
| Module outputs created                 | Passed |
| Lambda handler updated for S3 events   | Passed |
| Dev `main.tf` rewritten to call module | Passed |
| S3 upload bucket defined               | Passed |
| Lambda permission for S3 defined       | Passed |
| S3 bucket notification defined         | Passed |
| Dev outputs updated                    | Passed |
| OpenTofu reinitialized                 | Passed |
| OpenTofu plan completed                | Passed |
| OpenTofu apply completed               | Passed |
| Lambda processor deployed              | Passed |
| Upload bucket created                  | Passed |
| Test file uploaded to S3               | Passed |
| Lambda triggered automatically         | Passed |
| CloudWatch Logs confirmed S3 event     | Passed |
| Console verification completed         | Passed |
| Event-driven application destroyed     | Passed |
| `.gitignore` updated                   | Passed |
| Git commit created                     | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                     | What It Means                                                                        | How to Fix                                                                                                                       |
| ----------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| `Module not installed` during plan        | Module was added but OpenTofu was not reinitialized                                  | Run `.\infra\scripts\tofu.ps1 dev init`                                                                                          |
| `No valid credential sources found`       | OpenTofu cannot find the base AWS credentials needed before assuming the deploy role | Set `$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"`, refresh SSO, verify `aws sts get-caller-identity`, then rerun the OpenTofu command |
| `BucketAlreadyExists`                     | The upload bucket name is already taken globally                                     | Change the bucket suffix in `main.tf` to something more unique                                                                   |
| Upload succeeds but no Lambda logs appear | S3 event notification may not be wired yet or logs may be delayed                    | Wait 10–30 seconds, rerun log commands, and confirm `tofu apply` completed                                                       |
| S3 cannot invoke Lambda                   | Lambda permission is missing or was not created before the notification              | Confirm `aws_lambda_permission.allow_s3` exists and `depends_on` is present                                                      |
| `Error putting S3 notification`           | S3 notification target or permission is invalid                                      | Check `module.processor.function_arn`, `aws_lambda_permission.allow_s3`, and rerun `tofu apply`                                  |
| `BucketNotEmpty` during destroy           | Bucket contains files and `force_destroy` is missing                                 | Confirm `force_destroy = true` is set on the S3 bucket                                                                           |
| State lock error                          | A previous OpenTofu command was interrupted                                          | Confirm no command is running, then use `tofu force-unlock <LOCK_ID>` only if safe                                               |
| `tofu` is not recognized                  | OpenTofu is not in PATH                                                              | Add `C:\OpenTofu` to PATH or reopen PowerShell and verify `tofu --version`                                                       |
| Git commits build ZIP files               | Build folders are not ignored                                                        | Add `infra/modules/*/build/` and `infra/environments/*/build/` to `.gitignore`                                                   |

## Cleanup

This lab destroys the event-driven application resources with OpenTofu.

The following should be destroyed in Step 15:

| Resource                  | Expected State |
| ------------------------- | -------------- |
| S3 upload bucket          | Destroyed      |
| S3 bucket notification    | Destroyed      |
| Lambda permission         | Destroyed      |
| Lambda processor function | Destroyed      |
| Lambda execution role     | Destroyed      |
| CloudWatch log group      | Destroyed      |

The following should remain because they are part of the IaC foundation:

| Resource                         | Why It Stays                             |
| -------------------------------- | ---------------------------------------- |
| `workshop-iac` local folder      | Contains the IaC project and Git history |
| S3 remote-state bucket           | Stores OpenTofu state                    |
| DynamoDB `terraform-locks` table | Provides state locking                   |
| `workshop-tofu-deploy-role`      | Used for future OpenTofu deployments     |

### Full Cleanup Only If Ending the IaC Track

If I am completely done with the IaC track, I would follow the full cleanup instructions from Lab 7A to remove:

* S3 remote-state bucket
* All remote-state object versions
* DynamoDB `terraform-locks` table
* `workshop-tofu-deploy-role`
* Local `workshop-iac` folder

## Cleanup Verification

| Resource                      | Expected State |
| ----------------------------- | -------------- |
| Event-driven S3 upload bucket | Deleted        |
| S3 event notification         | Deleted        |
| Lambda permission             | Deleted        |
| Lambda processor function     | Deleted        |
| Lambda execution role         | Deleted        |
| Lambda log group              | Deleted        |
| Remote state bucket           | Kept           |
| DynamoDB lock table           | Kept           |
| OpenTofu deploy role          | Kept           |
| Local `workshop-iac` project  | Kept           |

## What I Learned

* OpenTofu modules allow infrastructure to be reused across environments.
* A module works like a programming function: it accepts inputs and returns outputs.
* Refactoring standalone resources into a module improves consistency and maintainability.
* Event-driven architecture allows services to react automatically to events.
* S3 can trigger Lambda when objects are uploaded.
* S3 requires a bucket notification configuration to send events to Lambda.
* Lambda requires a resource-based permission before S3 can invoke it.
* OpenTofu makes cross-service wiring explicit in code.
* OpenTofu manages dependency order during both creation and destruction.
* `force_destroy` is useful in labs because it allows S3 buckets with test files to be destroyed cleanly.
* CloudWatch Logs can prove that an event-driven workflow actually fired.
* Git makes IaC changes auditable, including module creation and architecture refactoring.

## Screenshots

| Screenshot                                      | Description                                         |
| ----------------------------------------------- | --------------------------------------------------- |
| `screenshots/workshop-iac-reopened.png`         | Existing IaC project reopened                       |
| `screenshots/module-folder-created.png`         | Lambda module folder created                        |
| `screenshots/module-variables-created.png`      | Module variables file created                       |
| `screenshots/module-main-created.png`           | Module resources file created                       |
| `screenshots/module-outputs-created.png`        | Module outputs file created                         |
| `screenshots/handler-updated-for-s3-events.png` | Lambda handler updated to process S3 events         |
| `screenshots/dev-main-module-s3-trigger.png`    | Dev `main.tf` updated with module and S3 trigger    |
| `screenshots/dev-outputs-updated.png`           | Outputs updated for Lambda and upload bucket        |
| `screenshots/tofu-init-module-success.png`      | OpenTofu reinitialized after module creation        |
| `screenshots/tofu-plan-event-driven.png`        | Plan showed event-driven architecture changes       |
| `screenshots/tofu-apply-success.png`            | Event-driven architecture deployed                  |
| `screenshots/apply-outputs.png`                 | Outputs showed Lambda function and upload bucket    |
| `screenshots/test-file-uploaded.png`            | Test file uploaded to S3                            |
| `screenshots/lambda-log-s3-trigger.png`         | CloudWatch Logs confirmed Lambda ran from S3 upload |
| `screenshots/lambda-monitor-invocation.png`     | Lambda Monitor tab showed invocation activity       |
| `screenshots/s3-upload-bucket-console.png`      | S3 upload bucket contained the test file            |
| `screenshots/module-created-role-console.png`   | IAM role created by module verified                 |
| `screenshots/tofu-destroy-success.png`          | Event-driven application destroyed                  |
| `screenshots/gitignore-updated.png`             | Module build artifacts ignored                      |
| `screenshots/git-commit-created.png`            | Lab 7C Git commit created                           |