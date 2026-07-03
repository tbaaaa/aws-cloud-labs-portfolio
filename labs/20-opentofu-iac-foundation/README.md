# Lab 20: OpenTofu Infrastructure as Code Foundation

## Lab Summary

In this lab, I created the foundation for managing AWS infrastructure with Infrastructure as Code (IaC) using OpenTofu.

I installed OpenTofu, created an S3 bucket for remote state storage, enabled versioning on the state bucket, created a DynamoDB table for state locking, and created a dedicated IAM role that OpenTofu can assume instead of using my administrator credentials directly.

I also created a reusable Infrastructure as Code project structure with separate environment folders for development, staging, and production. Finally, I used OpenTofu to create and destroy a test S3 bucket, proving that the full `init → plan → apply → destroy` workflow worked.

The S3 state bucket, DynamoDB lock table, IAM deploy role, and local `workshop-iac` project folder remain after this lab because Labs 7B and 7C build on the same foundation.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 7A — Setting Up Your Infrastructure as Code Foundation
* Session: 7 — Infrastructure as Code with OpenTofu
* Track: Solutions Architecture
* Difficulty: Beginner
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Set the active AWS CLI profile
* Verify AWS CLI authentication
* Install and verify OpenTofu
* Create a local Infrastructure as Code project folder
* Create an S3 bucket for remote OpenTofu state
* Enable S3 versioning on the state bucket
* Create a DynamoDB table for state locking
* Create an IAM role for OpenTofu deployments
* Attach least-privilege permissions to the deploy role
* Create a professional IaC folder structure
* Configure a development environment
* Configure an S3 backend
* Configure OpenTofu to assume the deploy role
* Create a test S3 bucket with OpenTofu
* Run `tofu init`
* Run `tofu plan`
* Run `tofu apply`
* Run `tofu destroy`
* Remove the test resource from the code
* Initialize a Git repository for the IaC project
* Create the first Git commit
* Preserve the IaC foundation for Labs 7B and 7C

## Services / Tools Used

| Service / Tool  | Purpose                                                                |
| --------------- | ---------------------------------------------------------------------- |
| OpenTofu        | Infrastructure as Code tool used to declaratively manage AWS resources |
| Amazon S3       | Stores the remote OpenTofu state file                                  |
| Amazon DynamoDB | Stores state locks to prevent concurrent infrastructure changes        |
| AWS IAM         | Provides a dedicated OpenTofu deployment role                          |
| AWS CLI         | Creates bootstrap AWS resources and verifies identity                  |
| Git             | Tracks IaC code changes through version control                        |
| VS Code         | Creates and edits OpenTofu configuration files                         |
| PowerShell      | Runs AWS CLI, OpenTofu, and Git commands                               |

## Prerequisites

* Completed Lab 01: AWS Account & CLI Setup
* AWS CLI installed and authenticated
* Working AWS IAM Identity Center / SSO profile
* Git installed
* VS Code installed or another code editor available
* Basic understanding of S3, IAM roles, and the AWS CLI

## Cost Notice

Estimated cost: `$0.00`

| Service         | Purpose              | Cost Consideration                                 |
| --------------- | -------------------- | -------------------------------------------------- |
| Amazon S3       | Remote state storage | Very small state file; expected cost is negligible |
| Amazon DynamoDB | State locking        | Expected to remain within Always Free usage        |
| AWS IAM         | OpenTofu deploy role | Free                                               |
| OpenTofu        | IaC tool             | Free and open source                               |

> The remote-state bucket, DynamoDB table, IAM role, and local IaC project are intentionally kept for Labs 7B and 7C.

## Key Concepts

| Concept                   | Meaning                                                                                                                         |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| Infrastructure as Code    | Managing infrastructure through version-controlled configuration files instead of manual console clicks or one-off CLI commands |
| OpenTofu                  | Open-source IaC tool compatible with Terraform-style HCL configuration                                                          |
| Declarative Configuration | Describing the desired end state instead of giving every creation step manually                                                 |
| Imperative Configuration  | A sequence of commands that tells a system exactly what to do                                                                   |
| State File                | A record of resources OpenTofu manages, allowing it to compare declared infrastructure with real infrastructure                 |
| Remote State              | Storing the state file in a shared remote location such as S3 instead of on a local laptop                                      |
| State Locking             | Preventing simultaneous infrastructure changes from multiple terminals or users                                                 |
| Backend                   | Configuration that tells OpenTofu where state is stored                                                                         |
| IAM Assume Role           | Using a dedicated IAM role for deployments instead of using administrator credentials directly                                  |
| `tofu init`               | Initializes the backend and downloads required provider plugins                                                                 |
| `tofu plan`               | Previews proposed infrastructure changes                                                                                        |
| `tofu apply`              | Creates or updates infrastructure                                                                                               |
| `tofu destroy`            | Removes infrastructure tracked in the OpenTofu state                                                                            |
| HCL                       | HashiCorp Configuration Language used in `.tf` configuration files                                                              |

## Security Notes

| Topic                            | Explanation                                                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| State files are sensitive        | State files can contain infrastructure metadata and sometimes sensitive values, so they should not be committed to GitHub |
| Remote state is versioned        | S3 versioning protects previous state versions if a state file is changed or overwritten incorrectly                      |
| State locking prevents conflicts | DynamoDB locking helps prevent two deployments from modifying the same environment at once                                |
| Dedicated deploy role            | OpenTofu assumes a separate IAM role instead of using administrator credentials directly                                  |
| Least privilege                  | The OpenTofu role is limited to S3 resources beginning with `workshop-` and the DynamoDB lock table                       |
| `.gitignore` matters             | Local state files, provider files, plans, and temporary files should not be committed                                     |
| Environment separation           | Dev, staging, and production should use separate state paths and variable values                                          |

## Architecture Overview

```text
Local OpenTofu Project
        |
        | tofu init / plan / apply / destroy
        v
AWS Provider
        |
        | assumes
        v
IAM Role: workshop-tofu-deploy-role
        |
        +--------------------+
        |                    |
        v                    v
S3 Remote State       DynamoDB Lock Table
(versioning enabled)  (prevents concurrent changes)
        |
        v
AWS Infrastructure
```

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

* Set my AWS CLI profile for the current PowerShell session.
* Verified my active AWS identity.
* Recorded my AWS account ID for later role and backend configuration.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity
```

**Expected result:**

```json
{
  "UserId": "REDACTED",
  "Account": "REDACTED",
  "Arn": "REDACTED"
}
```

**Notes:**

* If the SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Install and Verify OpenTofu

**What I did:**

* Downloaded and extracted OpenTofu.
* Added the OpenTofu folder to the current PowerShell PATH.
* Verified that the `tofu` command worked.

**Commands used:**

```powershell
Invoke-WebRequest `
  -Uri "https://github.com/opentofu/opentofu/releases/download/v<OPENTOFU_VERSION>/tofu_<OPENTOFU_VERSION>_windows_amd64.zip" `
  -OutFile "tofu.zip"

Expand-Archive -Path "tofu.zip" -DestinationPath "C:\OpenTofu" -Force

$env:Path += ";C:\OpenTofu"

tofu --version
```

**Expected result:**

```text
OpenTofu v<VERSION>
on windows_amd64
```

**Notes:**

* Use the current Windows AMD64 release asset from the official OpenTofu releases page rather than assuming an older version number.
* If `tofu` is not recognized, close PowerShell, open a new terminal, add `C:\OpenTofu` to PATH again if needed, and rerun `tofu --version`.

---

### Step 3: Create Local IaC Project Folder

**What I did:**

* Created the local project folder that will remain in place for Labs 7B and 7C.
* Opened the folder in VS Code.

**Commands used:**

```powershell
mkdir ~\Desktop\workshop-iac
cd ~\Desktop\workshop-iac
pwd
code .
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-iac
```

**Notes:**

* `workshop-iac` is the main local project folder for the remainder of the IaC track.
* Keep this folder after completing Lab 7A.

---

## Part 1: Bootstrap the Remote State Backend

### Step 4: Create State Bucket

**What I did:**

* Created an S3 bucket for OpenTofu state storage.
* Enabled S3 versioning to protect earlier state versions.

**Command used:**

```powershell
aws s3 mb s3://workshop-tofu-state-<INITIALS> --region us-east-1
```

**Enable versioning:**

```powershell
aws s3api put-bucket-versioning `
  --bucket workshop-tofu-state-<INITIALS> `
  --versioning-configuration Status=Enabled
```

**Validation command:**

```powershell
aws s3api get-bucket-versioning `
  --bucket workshop-tofu-state-<INITIALS> `
  --query "Status" `
  --output text
```

**Expected result:**

```text
Enabled
```

**Notes:**

* S3 bucket names must be globally unique.
* Keep the bucket name consistent in `backend.tf`.
* The bucket remains after this lab because it stores the state for later labs.

---

### Step 5: Create DynamoDB State Lock Table

**What I did:**

* Created a DynamoDB table named `terraform-locks`.
* Used the `LockID` attribute as the partition key.
* Configured on-demand billing.

**Command used:**

```powershell
aws dynamodb create-table `
  --table-name terraform-locks `
  --attribute-definitions AttributeName=LockID,AttributeType=S `
  --key-schema AttributeName=LockID,KeyType=HASH `
  --billing-mode PAY_PER_REQUEST `
  --region us-east-1
```

**Validation command:**

```powershell
aws dynamodb describe-table `
  --table-name terraform-locks `
  --query "Table.TableStatus" `
  --output text `
  --region us-east-1
```

**Expected result:**

```text
ACTIVE
```

**Notes:**

* The lock table prevents simultaneous changes to the same OpenTofu state.
* The table remains for Labs 7B and 7C.

---

## Part 2: Create OpenTofu Deploy Role

### Step 6: Create Trust Policy

**What I did:**

* Created a trust policy that allows identities in my AWS account to assume the OpenTofu deployment role.

**File created:**

```text
tofu-trust-policy.json
```

**File content:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::<ACCOUNT_ID>:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Notes:**

* Replace `<ACCOUNT_ID>` with the 12-digit AWS account ID from Step 1.
* In a production environment, the trust policy should be restricted to specific roles, users, or CI/CD identities.

---

### Step 7: Create the OpenTofu Deploy Role

**What I did:**

* Created an IAM role named `workshop-tofu-deploy-role`.

**Command used:**

```powershell
aws iam create-role `
  --role-name workshop-tofu-deploy-role `
  --assume-role-policy-document file://tofu-trust-policy.json
```

**Expected result:**

* JSON output containing the role ARN.

---

### Step 8: Attach OpenTofu Deployment Permissions

**What I did:**

* Created an inline IAM policy allowing OpenTofu to manage S3 buckets beginning with `workshop-`.
* Added permission to use the `terraform-locks` DynamoDB table.

**File created:**

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
      "Resource": "arn:aws:dynamodb:us-east-1:<ACCOUNT_ID>:table/terraform-locks"
    }
  ]
}
```

**Attach policy command:**

```powershell
aws iam put-role-policy `
  --role-name workshop-tofu-deploy-role `
  --policy-name tofu-deploy-permissions `
  --policy-document file://tofu-permissions.json
```

**Notes:**

* Replace `<ACCOUNT_ID>` with the AWS account ID.
* Wait about 10 seconds after attaching the policy to allow IAM propagation.

---

## Part 3: Scaffold the IaC Project Structure

### Step 9: Create Folder Structure

**What I did:**

* Created a reusable folder structure for development, staging, production, modules, scripts, and future policies.

**Commands used:**

```powershell
mkdir infra\environments\dev
mkdir infra\environments\staging
mkdir infra\environments\prod
mkdir infra\modules
mkdir infra\scripts
mkdir infra\policies
```

**Expected structure:**

```text
workshop-iac/
├── tofu-trust-policy.json
├── tofu-permissions.json
└── infra/
    ├── environments/
    │   ├── dev/
    │   ├── staging/
    │   └── prod/
    ├── modules/
    ├── scripts/
    └── policies/
```

---

### Step 10: Create Development Environment Configuration

Create the following files inside:

```text
infra/environments/dev/
```

#### Step 10A: Create `backend.tf`

```hcl
terraform {
  backend "s3" {
    bucket         = "workshop-tofu-state-<INITIALS>"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

#### Step 10B: Create `variables.tf`

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

variable "project" {
  description = "Project name used for resource naming"
  type        = string
}

variable "tofu_role_arn" {
  description = "ARN of the IAM role OpenTofu assumes"
  type        = string
}
```

#### Step 10C: Create `terraform.tfvars`

```hcl
environment   = "dev"
aws_region    = "us-east-1"
project       = "workshop-iac"
tofu_role_arn = "arn:aws:iam::<ACCOUNT_ID>:role/workshop-tofu-deploy-role"
```

#### Step 10D: Create Initial `main.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
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

resource "aws_s3_bucket" "test" {
  bucket = "${var.project}-${var.environment}-test-<INITIALS>"
}
```

#### Step 10E: Create `outputs.tf`

```hcl
output "test_bucket_name" {
  description = "Name of the test S3 bucket created by OpenTofu"
  value       = aws_s3_bucket.test.bucket
}

output "test_bucket_arn" {
  description = "ARN of the test S3 bucket"
  value       = aws_s3_bucket.test.arn
}
```

**Expected development-folder structure:**

```text
infra/environments/dev/
├── backend.tf
├── main.tf
├── outputs.tf
├── terraform.tfvars
└── variables.tf
```

---

### Step 11: Create `.gitignore`

**What I did:**

* Created a `.gitignore` file in the root `workshop-iac` folder.
* Excluded local state files, plans, provider data, and sensitive override files.

**File created:**

```text
.gitignore
```

**File content:**

```gitignore
# OpenTofu local files
.terraform/
*.tfstate
*.tfstate.backup
*.tfplan
.terraform.lock.hcl

# Sensitive variable overrides
*.auto.tfvars
override.tf
override.tf.json

# OS files
.DS_Store
Thumbs.db
```

**Notes:**

* `.gitignore` must be created in the `workshop-iac` root folder.
* Do not commit state files to GitHub.

---

## Part 4: Initialize and Test OpenTofu

### Step 12: Initialize OpenTofu

**What I did:**

* Changed into the development environment folder.
* Initialized the S3 backend and AWS provider.

**Commands used:**

```powershell
cd ~\Desktop\workshop-iac\infra\environments\dev
tofu init
```

**Expected result:**

```text
Initializing the backend...
Initializing provider plugins...
OpenTofu has been successfully initialized!
```

---

### Step 13: Preview Infrastructure Changes

**What I did:**

* Used `tofu plan` to preview the test S3 bucket before creating it.

**Command used:**

```powershell
tofu plan
```

**Expected result:**

```text
Plan: 1 to add, 0 to change, 0 to destroy.
```

**Notes:**

* `tofu plan` does not create resources.
* It previews what OpenTofu will change.

---

### Step 14: Apply Infrastructure Changes

**What I did:**

* Used OpenTofu to create the test S3 bucket.
* Confirmed the deployment by typing `yes`.

**Command used:**

```powershell
tofu apply
```

**Expected result:**

```text
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Expected outputs:**

```text
test_bucket_name = "workshop-iac-dev-test-<INITIALS>"
test_bucket_arn = "arn:aws:s3:::workshop-iac-dev-test-<INITIALS>"
```

**Console checkpoint:**

* Opened Amazon S3 in the AWS Console.
* Located the test bucket.
* Confirmed these tags existed:

```text
Environment = dev
Project = workshop-iac
ManagedBy = opentofu
```

---

### Step 15: Destroy the Test Resource

**What I did:**

* Removed the temporary test bucket using OpenTofu.
* Confirmed the deletion by typing `yes`.

**Command used:**

```powershell
tofu destroy
```

**Expected result:**

```text
Destroy complete! Resources: 1 destroyed.
```

**Notes:**

* The temporary test bucket is the only resource destroyed in this lab.
* Keep the remote state bucket, DynamoDB lock table, deploy role, and local IaC project.

---

### Step 16: Remove Test Resource From Code

**What I did:**

* Removed the temporary test bucket resource from `main.tf`.
* Removed `outputs.tf` because the temporary bucket outputs were no longer needed.

**Final `main.tf`:**

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
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
```

**Action taken:**

* Deleted `outputs.tf`.

**Notes:**

* IaC configuration should always match the infrastructure that actually exists.

---

## Part 5: Put the IaC Project Under Version Control

### Step 17: Initialize Git

**What I did:**

* Returned to the root `workshop-iac` folder.
* Initialized Git.
* Configured my Git author identity if it had not been configured already.

**Commands used:**

```powershell
cd ~\Desktop\workshop-iac
git init
git config user.name "Your Name"
git config user.email "you@example.com"
```

---

### Step 18: Create First Git Commit

**What I did:**

* Checked the files Git would track.
* Added the project files.
* Created the first commit.
* Verified the commit history.

**Commands used:**

```powershell
git status
git add .
git commit -m "Initial IaC foundation: backend, deploy role, dev environment"
git log --oneline
```

**Expected result:**

* Git shows one commit containing the OpenTofu backend configuration, IAM policies, environment structure, and `.gitignore`.

## Validation / Checkpoints

| Checkpoint                               | Result |
| ---------------------------------------- | ------ |
| AWS CLI profile set                      | Passed |
| AWS identity verified                    | Passed |
| OpenTofu installed                       | Passed |
| OpenTofu version verified                | Passed |
| Local `workshop-iac` folder created      | Passed |
| State bucket created                     | Passed |
| State bucket versioning enabled          | Passed |
| DynamoDB lock table created              | Passed |
| DynamoDB table became active             | Passed |
| OpenTofu trust policy created            | Passed |
| OpenTofu deploy role created             | Passed |
| OpenTofu permissions policy attached     | Passed |
| IaC environment folder structure created | Passed |
| Dev backend configuration created        | Passed |
| Variables file created                   | Passed |
| Variable values file created             | Passed |
| Initial OpenTofu configuration created   | Passed |
| `.gitignore` created                     | Passed |
| `tofu init` completed                    | Passed |
| `tofu plan` completed                    | Passed |
| Test S3 bucket created with `tofu apply` | Passed |
| Test bucket tags verified                | Passed |
| Test bucket removed with `tofu destroy`  | Passed |
| Test resource removed from code          | Passed |
| Git repository initialized               | Passed |
| Initial Git commit created               | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                       | What It Means                                                             | How to Fix                                                                                                                 |
| ------------------------------------------- | ------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `tofu` is not recognized                    | OpenTofu is not in the PowerShell PATH                                    | Confirm `tofu.exe` exists in `C:\OpenTofu`, reopen PowerShell, then run `tofu --version`                                   |
| `No configuration files`                    | OpenTofu was run from the wrong folder or files use the wrong extension   | Confirm the terminal path ends in `infra\environments\dev` and files end in `.tf`, not `.tf.txt`                           |
| `Error configuring S3 Backend`              | Backend bucket name does not match the real state bucket                  | Check the bucket value in `backend.tf`                                                                                     |
| `Cannot assume IAM role`                    | AWS session expired, trust policy is incorrect, or IAM has not propagated | Run `aws sts get-caller-identity`, refresh SSO if needed, verify the account ID in the trust policy, and wait briefly      |
| `Error acquiring the state lock`            | A previous OpenTofu command left a stale DynamoDB lock                    | Use the lock ID from the error with `tofu force-unlock <LOCK_ID>` only after confirming no deployment is currently running |
| `AccessDenied` creating a bucket            | The deploy role does not allow the bucket name                            | Confirm the bucket name begins with `workshop-` and check `tofu-permissions.json`                                          |
| `BucketAlreadyExists` creating state bucket | The selected S3 bucket name is already owned globally                     | Add extra unique characters and update `backend.tf` to use the new name                                                    |
| `Author identity unknown`                   | Git name and email are not configured                                     | Run the two `git config` commands from Step 17                                                                             |
| `git init` says repository already exists   | Git was initialized earlier                                               | Continue; the repository is already initialized                                                                            |
| `main.tf.txt` appears instead of `main.tf`  | Notepad silently added `.txt`                                             | Recreate the file in VS Code or save as “All Files” with the exact filename                                                |

## Cleanup

> **Do not clean up this lab if you are continuing to Lab 7B or Lab 7C.**

The following resources should remain in place:

| Resource                             | Why It Stays                                      |
| ------------------------------------ | ------------------------------------------------- |
| `workshop-iac` local folder          | Contains the IaC project and Git history          |
| S3 state bucket                      | Stores remote OpenTofu state                      |
| DynamoDB `terraform-locks` table     | Prevents concurrent infrastructure changes        |
| `workshop-tofu-deploy-role` IAM role | OpenTofu assumes this role for future deployments |

The temporary test S3 bucket was removed in Step 15.

### Full Cleanup Only If Ending the IaC Track

```powershell
aws s3 rm s3://workshop-tofu-state-<INITIALS> --recursive

$versions = aws s3api list-object-versions `
  --bucket workshop-tofu-state-<INITIALS> `
  --output json | ConvertFrom-Json

foreach ($v in $versions.Versions) {
  aws s3api delete-object `
    --bucket workshop-tofu-state-<INITIALS> `
    --key $v.Key `
    --version-id $v.VersionId | Out-Null
}

foreach ($m in $versions.DeleteMarkers) {
  aws s3api delete-object `
    --bucket workshop-tofu-state-<INITIALS> `
    --key $m.Key `
    --version-id $m.VersionId | Out-Null
}

aws s3 rb s3://workshop-tofu-state-<INITIALS>

aws dynamodb delete-table `
  --table-name terraform-locks `
  --region us-east-1

aws iam delete-role-policy `
  --role-name workshop-tofu-deploy-role `
  --policy-name tofu-deploy-permissions

aws iam delete-role `
  --role-name workshop-tofu-deploy-role

cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-iac
```

## Cleanup Verification

| Resource                          | Expected State            |
| --------------------------------- | ------------------------- |
| Temporary proof-of-life S3 bucket | Deleted by `tofu destroy` |
| State bucket                      | Kept for Labs 7B and 7C   |
| DynamoDB lock table               | Kept for Labs 7B and 7C   |
| OpenTofu deploy role              | Kept for Labs 7B and 7C   |
| Local `workshop-iac` folder       | Kept for Labs 7B and 7C   |
| Git repository                    | Kept for Labs 7B and 7C   |

## What I Learned

* Infrastructure as Code lets me define AWS infrastructure in version-controlled files instead of creating resources manually.
* OpenTofu uses declarative configuration: I describe the desired end state and the tool determines the actions required.
* The OpenTofu state file is the source of truth for tracked infrastructure.
* Remote state in S3 protects the state file from local computer loss and supports collaboration.
* S3 versioning helps recover earlier state versions.
* DynamoDB state locking prevents simultaneous changes to the same infrastructure.
* A dedicated IAM role is safer and more auditable than using administrator credentials directly.
* `tofu plan` previews changes before resources are modified.
* `tofu apply` creates or updates resources.
* `tofu destroy` removes tracked resources cleanly.
* `.gitignore` prevents state files and other sensitive local artifacts from being committed to GitHub.
* Development, staging, and production environments should use separate state paths and values while sharing reusable IaC modules.
* The local IaC project, state bucket, lock table, and deploy role are foundations that later labs will reuse.

## Screenshots

| Screenshot                                        | Description                                                        |
| ------------------------------------------------- | ------------------------------------------------------------------ |
| `screenshots/opentofu-version.png`                | OpenTofu installation verified                                     |
| `screenshots/workshop-iac-folder.png`             | Local IaC project folder created                                   |
| `screenshots/state-bucket-created.png`            | S3 remote-state bucket created                                     |
| `screenshots/state-bucket-versioning-enabled.png` | S3 versioning enabled for state bucket                             |
| `screenshots/dynamodb-lock-table-active.png`      | DynamoDB lock table active                                         |
| `screenshots/tofu-deploy-role-created.png`        | IAM deploy role created                                            |
| `screenshots/tofu-permissions-attached.png`       | OpenTofu deployment policy attached                                |
| `screenshots/iac-folder-structure.png`            | Dev, staging, prod, modules, scripts, and policies folders created |
| `screenshots/dev-config-files.png`                | Development environment `.tf` files created                        |
| `screenshots/gitignore-created.png`               | `.gitignore` configured                                            |
| `screenshots/tofu-init-success.png`               | OpenTofu backend initialized successfully                          |
| `screenshots/tofu-plan-success.png`               | OpenTofu plan showed test bucket creation                          |
| `screenshots/tofu-apply-success.png`              | Test S3 bucket created with OpenTofu                               |
| `screenshots/s3-tags-verified.png`                | Test bucket tags verified in AWS Console                           |
| `screenshots/tofu-destroy-success.png`            | Test S3 bucket destroyed with OpenTofu                             |
| `screenshots/main-tf-cleaned.png`                 | Test resource removed from final configuration                     |
| `screenshots/git-initialized.png`                 | Git repository initialized                                         |
| `screenshots/initial-git-commit.png`              | First IaC Git commit created                                       |