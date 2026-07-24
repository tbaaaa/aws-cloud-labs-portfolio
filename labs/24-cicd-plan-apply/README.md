# Lab 24: CI/CD Plan and Apply

## Lab Summary

In this lab, I built a CI/CD pipeline for Infrastructure as Code using GitHub Actions, OpenTofu, GitHub OIDC, and AWS.

In Lab 8A, I configured GitHub OIDC so GitHub Actions can authenticate to AWS without storing long-lived AWS access keys. In this lab, I used that authentication foundation to create an automated infrastructure workflow.

The workflow runs `tofu plan` automatically on pull requests so changes can be reviewed before merging. When the pull request is merged into `main`, the workflow runs `tofu apply` automatically. I also added a separate guarded destroy workflow that can only be triggered manually and requires a confirmation word before infrastructure can be destroyed.

This lab demonstrates a GitOps-style workflow where Git becomes the source of truth for infrastructure changes.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 8B — A CI/CD Pipeline — Plan on Pull Request, Apply on Merge
* Session: 8 — CI/CD for Infrastructure
* Track: Solutions Architecture + Infrastructure as Code
* Difficulty: Intermediate
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Reopen the `workshop-iac` repository from Lab 8A
* Confirm the local Git repository is clean
* Replace the development OpenTofu environment with a simple CI/CD-managed S3 bucket
* Create a GitHub Actions workflow file
* Configure the workflow to authenticate to AWS through GitHub OIDC
* Configure the workflow to install OpenTofu
* Run `tofu init` in the pipeline
* Run `tofu plan` automatically on pull requests
* Run `tofu apply` automatically on pushes to `main`
* Push the workflow to GitHub
* Watch the first automated deployment run
* Verify the S3 demo bucket was created
* Create a feature branch
* Add S3 bucket versioning through OpenTofu
* Open a pull request
* Verify `tofu plan` runs on the pull request
* Confirm the pull request does not apply changes yet
* Merge the pull request
* Verify `tofu apply` runs after merge
* Confirm versioning is enabled on the S3 bucket
* Add a guarded manual destroy workflow
* Test the destroy workflow failure path with the wrong confirmation word
* Verify infrastructure was not destroyed
* Commit and push all CI/CD workflow files
* Preserve resources for Lab 8C

## Services / Tools Used

| Service / Tool    | Purpose                                                        |
| ----------------- | -------------------------------------------------------------- |
| GitHub Actions    | Runs the CI/CD workflow                                        |
| GitHub OIDC       | Authenticates GitHub Actions to AWS without stored access keys |
| AWS IAM           | Provides the GitHub pipeline role and OpenTofu deploy role     |
| AWS STS           | Issues temporary credentials through OIDC role assumption      |
| OpenTofu          | Plans and applies infrastructure changes                       |
| Amazon S3         | Demo infrastructure managed by the pipeline                    |
| DynamoDB          | Provides OpenTofu remote-state locking                         |
| Git               | Creates branches, commits, pushes, and pull requests           |
| GitHub Repository | Stores the IaC source code and workflow files                  |
| PowerShell        | Runs Git and AWS CLI commands locally                          |
| AWS CLI           | Verifies deployed infrastructure                               |

## Prerequisites

* Completed Lab 7A: OpenTofu Infrastructure as Code Foundation
* Completed Lab 7B: Lambda with Infrastructure as Code
* Completed Lab 7C: Event-Driven Architecture with Reusable Modules
* Completed Lab 8A: GitHub OIDC
* `workshop-iac` repository exists locally
* `workshop-iac` repository exists on GitHub
* GitHub OIDC provider exists in AWS IAM
* `github-actions-infra` pipeline role exists
* `workshop-tofu-deploy-role` exists
* S3 remote-state bucket exists
* DynamoDB `terraform-locks` table exists
* GitHub Actions is enabled for the repository
* AWS CLI is installed and authenticated locally

## Cost Notice

Estimated cost: `$0.00`

| Service        | Cost Consideration                                                             |
| -------------- | ------------------------------------------------------------------------------ |
| GitHub Actions | Uses GitHub Actions minutes; expected to remain within free usage for this lab |
| Amazon S3      | One small demo bucket; expected cost is negligible                             |
| DynamoDB       | Existing state-lock table; expected to remain within free usage                |
| AWS IAM        | Free                                                                           |
| OpenTofu       | Free and open source                                                           |

## Key Concepts

| Concept                  | Meaning                                                                                                        |
| ------------------------ | -------------------------------------------------------------------------------------------------------------- |
| CI/CD                    | Continuous Integration and Continuous Delivery. Automation that validates, plans, and deploys changes.         |
| GitOps                   | A workflow where Git is the source of truth and infrastructure changes flow through commits and pull requests. |
| GitHub Actions Workflow  | A YAML file stored in `.github/workflows/` that defines automated jobs.                                        |
| Pull Request             | A proposed code change that can be reviewed before merging into `main`.                                        |
| `tofu plan`              | Previews infrastructure changes without applying them.                                                         |
| `tofu apply`             | Applies the approved infrastructure changes.                                                                   |
| OIDC Authentication      | Keyless authentication where GitHub exchanges a short-lived identity token for AWS credentials.                |
| `id-token: write`        | GitHub Actions permission required for the workflow to request an OIDC token.                                  |
| Conditional Step         | A workflow step that runs only when a condition is true.                                                       |
| Manual Workflow Dispatch | A workflow trigger that requires a human to start it from the GitHub UI.                                       |
| Guardrail                | A safety control that reduces the chance of dangerous or accidental actions.                                   |
| Blast Radius             | The amount of damage or change that can happen if an action goes wrong.                                        |

## Security Notes

| Topic                          | Explanation                                                                                          |
| ------------------------------ | ---------------------------------------------------------------------------------------------------- |
| No stored AWS keys             | The pipeline uses GitHub OIDC instead of long-lived AWS access keys stored as GitHub secrets.        |
| Plan before apply              | Infrastructure changes are previewed on pull requests before they can be merged.                     |
| Apply only on merge            | The workflow applies changes only when code reaches `main`.                                          |
| `id-token: write` is required  | Without this permission, GitHub Actions cannot request the OIDC token needed for AWS authentication. |
| Destroy is manual only         | Destructive actions should not run automatically on normal pushes or pull requests.                  |
| Confirmation word reduces risk | The destroy workflow requires the user to type `destroy` before running destructive actions.         |
| Remote state stays protected   | The pipeline uses the same S3 remote state and DynamoDB locking from the IaC foundation.             |
| Pipeline role remains scoped   | The GitHub pipeline role assumes the deploy role rather than using broad static credentials.         |

## Architecture Overview

```text
Pull Request to main
        |
        v
GitHub Actions Workflow
        |
        v
OIDC Authentication to AWS
        |
        v
OpenTofu Init
        |
        v
OpenTofu Plan only
        |
        v
Reviewer sees proposed infrastructure change


Merge to main
        |
        v
GitHub Actions Workflow
        |
        v
OIDC Authentication to AWS
        |
        v
OpenTofu Init
        |
        v
OpenTofu Apply
        |
        v
AWS infrastructure updated


Manual destroy workflow
        |
        v
User types confirmation word
        |
        v
OpenTofu Destroy
```

## Lab Steps

### Step 1: Open the `workshop-iac` Project

**What I did:**

* Opened the local `workshop-iac` project.
* Verified that my local Git working tree was clean.

**Commands used:**

```powershell
cd ~\Desktop\workshop-iac
code .
git status
```

**Expected result:**

```text
nothing to commit, working tree clean
```

**If there were uncommitted changes:**

```powershell
git add .
git commit -m "sync"
git push
```

**Result:**

* Local repository was ready for the CI/CD workflow work.

---

### Step 2: Set a Simple Resource for the Pipeline to Manage

**What I did:**

* Replaced the existing development environment with a simple S3 demo bucket.
* This gives the pipeline a predictable resource to plan and apply.

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

resource "aws_s3_bucket" "demo" {
  bucket = "workshop-${var.environment}-cicd-demo-${data.aws_caller_identity.current.account_id}"
}
```

**Notes:**

* The provider still assumes `var.tofu_role_arn`.
* The code stays the same whether OpenTofu runs locally or in GitHub Actions.
* The difference is the identity running OpenTofu:

  * Locally: my SSO-authenticated AWS profile
  * Pipeline: GitHub OIDC role

---

### Step 3: Update Outputs

**What I did:**

* Replaced old outputs from Lab 7C with a new output for the demo bucket.

**File updated:**

```text
infra/environments/dev/outputs.tf
```

**File content:**

```hcl
output "demo_bucket" {
  value = aws_s3_bucket.demo.bucket
}
```

**Result:**

* OpenTofu will display the demo bucket name after apply.

---

## Part 1: Create the CI/CD Workflow

### Step 4: Create Workflow Folder

**What I did:**

* Created the GitHub Actions workflow folder structure.

**Folder path:**

```text
.github/workflows/
```

**PowerShell command from the `workshop-iac` root:**

```powershell
mkdir .github
mkdir .github\workflows
```

**Notes:**

* The folder must be named exactly `.github`.
* The workflow folder must be named exactly `workflows`.
* GitHub only runs workflow files from `.github/workflows/`.

---

### Step 5: Create Infrastructure Workflow

**What I did:**

* Created a GitHub Actions workflow that:

  * Runs on pull requests to `main`
  * Runs on pushes to `main`
  * Authenticates to AWS using OIDC
  * Installs OpenTofu
  * Runs `tofu init`
  * Runs `tofu plan` on pull requests
  * Runs `tofu apply` on pushes to `main`

**File created:**

```text
.github/workflows/infra.yml
```

**File content:**

```yaml
name: Infrastructure CI/CD

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: infra/environments/dev

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/github-actions-infra
          aws-region: us-east-1

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: Tofu Init
        run: tofu init

      - name: Tofu Plan
        if: github.event_name == 'pull_request'
        run: tofu plan

      - name: Tofu Apply
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        run: tofu apply -auto-approve
```

**Replacements made:**

| Placeholder         | Replacement                |
| ------------------- | -------------------------- |
| `<YOUR_ACCOUNT_ID>` | My 12-digit AWS account ID |

**Important lines:**

| Workflow Line                                                        | Why It Matters                                 |
| -------------------------------------------------------------------- | ---------------------------------------------- |
| `permissions: id-token: write`                                       | Allows GitHub Actions to request an OIDC token |
| `role-to-assume`                                                     | Tells GitHub Actions which AWS role to assume  |
| `working-directory`                                                  | Runs OpenTofu from the dev environment folder  |
| `if: github.event_name == 'pull_request'`                            | Runs plan only for pull requests               |
| `if: github.event_name == 'push' && github.ref == 'refs/heads/main'` | Runs apply only after merge/push to main       |

---

### Step 6: Commit and Push the Workflow

**What I did:**

* Committed the workflow to `main`.
* Pushed it to GitHub.
* This first push triggered the apply workflow automatically.

**Commands used:**

```powershell
git add .
git commit -m "Add CI/CD pipeline: plan on PR, apply on merge"
git push
```

**Expected result:**

* GitHub Actions workflow starts automatically.
* Because this is a push to `main`, the `Tofu Apply` step runs.
* The `Tofu Plan` step is skipped because this is not a pull request.

---

### Step 7: Watch the First Workflow Run

**What I did:**

* Opened the GitHub repository in a browser.
* Went to the **Actions** tab.
* Opened the workflow run triggered by the commit.
* Opened the `terraform` job.
* Watched the workflow steps run.

**Expected result:**

| Step                               | Expected Status |
| ---------------------------------- | --------------- |
| Checkout code                      | Success         |
| Configure AWS credentials via OIDC | Success         |
| Setup OpenTofu                     | Success         |
| Tofu Init                          | Success         |
| Tofu Plan                          | Skipped         |
| Tofu Apply                         | Success         |

**Result:**

* The pipeline authenticated to AWS without stored access keys.
* The pipeline applied the OpenTofu configuration.
* The demo S3 bucket was created.

---

### Step 8: Verify the Demo Bucket Was Created

**What I did:**

* Verified from my terminal that the CI/CD pipeline created the S3 bucket.

**PowerShell command:**

```powershell
aws s3 ls | Select-String "cicd-demo"
```

**Expected result:**

```text
workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID>
```

**Result:**

* The demo bucket existed in AWS.

**What this proved:**

* GitHub Actions authenticated through OIDC.
* GitHub Actions assumed the pipeline role.
* The pipeline role was able to assume the deploy role.
* OpenTofu applied the infrastructure successfully.

---

## Part 2: Use a Pull Request to Run Plan

### Step 9: Create a Feature Branch

**What I did:**

* Created a new branch for a proposed infrastructure change.

**Command used:**

```powershell
git checkout -b add-versioning
```

**Result:**

* I was working on the `add-versioning` branch.

---

### Step 10: Add S3 Bucket Versioning

**What I did:**

* Added bucket versioning to the demo S3 bucket in OpenTofu code.

**File updated:**

```text
infra/environments/dev/main.tf
```

**Block added after the `aws_s3_bucket "demo"` resource:**

```hcl
resource "aws_s3_bucket_versioning" "demo" {
  bucket = aws_s3_bucket.demo.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

**Result:**

* The branch now proposed enabling versioning on the CI/CD demo bucket.

---

### Step 11: Commit and Push the Feature Branch

**What I did:**

* Committed the bucket versioning change.
* Pushed the branch to GitHub.

**Commands used:**

```powershell
git add .
git commit -m "Enable versioning on the demo bucket"
git push -u origin add-versioning
```

**Expected result:**

* GitHub shows the branch.
* GitHub offers a **Compare & pull request** button.

---

### Step 12: Open a Pull Request

**What I did:**

* Opened the GitHub repository.
* Created a pull request from `add-versioning` into `main`.

**GitHub steps:**

```text
Repository
Compare & pull request
Base: main
Compare: add-versioning
Create pull request
```

**Result:**

* Pull request was created successfully.

---

### Step 13: Watch the Pull Request Plan

**What I did:**

* Opened the pull request.
* Scrolled to the checks section.
* Opened the workflow run details.
* Confirmed that `tofu plan` ran.

**Expected result:**

| Step                               | Expected Status |
| ---------------------------------- | --------------- |
| Checkout code                      | Success         |
| Configure AWS credentials via OIDC | Success         |
| Setup OpenTofu                     | Success         |
| Tofu Init                          | Success         |
| Tofu Plan                          | Success         |
| Tofu Apply                         | Skipped         |

**Expected plan output:**

```text
aws_s3_bucket_versioning.demo will be created
Plan: 1 to add, 0 to change, 0 to destroy.
```

**Result:**

* The workflow previewed the infrastructure change.
* The workflow did not apply the change yet.

---

### Step 14: Verify Versioning Was Not Applied Yet

**What I did:**

* Checked the demo bucket versioning status before merging the pull request.

**Command used:**

```powershell
aws s3api get-bucket-versioning `
  --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> `
  --query "Status" `
  --output text
```

**Expected result:**

```text
No output
```

**Result:**

* Versioning was not enabled yet.

**What this proved:**

* Pull requests only run `tofu plan`.
* Pull requests do not deploy infrastructure changes.
* The plan acts as a review gate.

---

## Part 3: Merge the Pull Request to Apply

### Step 15: Merge the Pull Request

**What I did:**

* Merged the pull request in GitHub.

**GitHub steps:**

```text
Pull request page
Merge pull request
Confirm merge
```

**Result:**

* The feature branch was merged into `main`.

---

### Step 16: Watch the Apply Workflow

**What I did:**

* Opened the GitHub **Actions** tab.
* Opened the workflow run triggered by the merge.
* Confirmed that `tofu apply` ran.

**Expected result:**

| Step                               | Expected Status |
| ---------------------------------- | --------------- |
| Checkout code                      | Success         |
| Configure AWS credentials via OIDC | Success         |
| Setup OpenTofu                     | Success         |
| Tofu Init                          | Success         |
| Tofu Plan                          | Skipped         |
| Tofu Apply                         | Success         |

**Expected apply output:**

```text
aws_s3_bucket_versioning.demo: Creating...
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

**Result:**

* The approved infrastructure change was deployed automatically.

---

### Step 17: Verify Versioning Is Enabled

**What I did:**

* Checked the versioning status again after the merge.

**Command used:**

```powershell
aws s3api get-bucket-versioning `
  --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> `
  --query "Status" `
  --output text
```

**Expected result:**

```text
Enabled
```

**Result:**

* S3 versioning was enabled on the demo bucket.

**What this proved:**

* GitHub Actions applied the change only after merge.
* The GitHub `main` branch became the source of truth.
* The pipeline kept AWS aligned with the approved code.

---

### Step 18: Sync Local `main`

**What I did:**

* Switched my local repository back to `main`.
* Pulled the merge commit from GitHub.

**Commands used:**

```powershell
git checkout main
git pull origin main
```

**Expected result:**

* Local `main` included the merged bucket versioning change.

---

## Part 4: Add a Guarded Destroy Workflow

### Step 19: Create Destroy Workflow

**What I did:**

* Created a separate manual destroy workflow.
* The workflow can only run from GitHub’s **Run workflow** button.
* It requires the user to type the confirmation word `destroy`.

**File created:**

```text
.github/workflows/destroy.yml
```

**File content:**

```yaml
name: Destroy Infrastructure

on:
  workflow_dispatch:
    inputs:
      confirm:
        description: 'Type "destroy" to confirm tearing down the dev environment'
        required: true
        type: string

permissions:
  id-token: write
  contents: read

jobs:
  destroy:
    runs-on: ubuntu-latest

    defaults:
      run:
        working-directory: infra/environments/dev

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Verify confirmation
        run: |
          if [ "${{ github.event.inputs.confirm }}" != "destroy" ]; then
            echo "Confirmation text was not exactly 'destroy'. Aborting - nothing was changed."
            exit 1
          fi

          echo "Confirmation received. Proceeding to destroy the dev environment."

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<YOUR_ACCOUNT_ID>:role/github-actions-infra
          aws-region: us-east-1

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: Tofu Init
        run: tofu init

      - name: Tofu Destroy
        run: tofu destroy -auto-approve
```

**Replacements made:**

| Placeholder         | Replacement                |
| ------------------- | -------------------------- |
| `<YOUR_ACCOUNT_ID>` | My 12-digit AWS account ID |

**What makes this safer:**

| Guardrail                    | Why It Matters                                                             |
| ---------------------------- | -------------------------------------------------------------------------- |
| `workflow_dispatch` only     | Destroy cannot run automatically on push or pull request                   |
| Confirmation input           | A human must type a confirmation word                                      |
| Verification step runs first | The workflow aborts before AWS authentication if the confirmation is wrong |
| Separate workflow            | Destructive actions are separated from normal deploy workflow              |

---

### Step 20: Commit and Push the Destroy Workflow

**What I did:**

* Committed and pushed the destroy workflow.

**Commands used:**

```powershell
git add .
git commit -m "Add guarded manual destroy workflow"
git push
```

**Expected result:**

* This push triggers the normal infrastructure workflow.
* Since no infrastructure code changed, OpenTofu should report no changes.

---

### Step 21: Test the Destroy Guard Failure Path

**What I did:**

* Opened GitHub Actions.
* Selected **Destroy Infrastructure**.
* Clicked **Run workflow**.
* Typed an incorrect confirmation word such as `no`.
* Started the workflow.

**Expected result:**

* Workflow fails at the **Verify confirmation** step.
* Workflow does not authenticate to AWS.
* Workflow does not run `tofu destroy`.

**Expected message:**

```text
Confirmation text was not exactly 'destroy'. Aborting - nothing was changed.
```

**Result:**

* The guard stopped the destroy workflow before any AWS resources were touched.

---

### Step 22: Verify Infrastructure Was Not Destroyed

**What I did:**

* Confirmed the demo bucket still existed.

**PowerShell command:**

```powershell
aws s3 ls | Select-String "cicd-demo"
```

**Expected result:**

```text
workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID>
```

**Result:**

* Demo bucket still existed.
* The destroy guard worked.

---

## Validation / Checkpoints

| Checkpoint                                   | Result |
| -------------------------------------------- | ------ |
| `workshop-iac` opened                        | Passed |
| Git working tree checked                     | Passed |
| Dev `main.tf` replaced with demo S3 bucket   | Passed |
| Dev `outputs.tf` updated                     | Passed |
| `.github/workflows` folder created           | Passed |
| `infra.yml` workflow created                 | Passed |
| OIDC workflow permissions added              | Passed |
| GitHub Actions role ARN added                | Passed |
| Workflow committed and pushed to `main`      | Passed |
| First apply workflow ran                     | Passed |
| Demo S3 bucket created                       | Passed |
| Feature branch created                       | Passed |
| Bucket versioning resource added             | Passed |
| Feature branch pushed                        | Passed |
| Pull request opened                          | Passed |
| Pull request plan ran                        | Passed |
| Pull request did not apply changes           | Passed |
| PR merged                                    | Passed |
| Apply workflow ran after merge               | Passed |
| S3 versioning enabled                        | Passed |
| Local `main` synced                          | Passed |
| Manual destroy workflow created              | Passed |
| Destroy workflow committed and pushed        | Passed |
| Destroy guard tested with wrong confirmation | Passed |
| Infrastructure remained intact               | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                                     | What It Means                                               | How to Fix                                                                        |
| --------------------------------------------------------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Workflow does not run                                     | Workflow file is in the wrong location                      | Confirm the path is `.github/workflows/infra.yml`                                 |
| OIDC authentication fails                                 | Missing `id-token: write` or incorrect trust policy subject | Confirm workflow permissions and check the Lab 8A OIDC trust policy               |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | GitHub cannot assume the pipeline role                      | Recheck the GitHub OIDC provider, role trust policy, and repository subject claim |
| `Not authorized to perform sts:AssumeRole`                | Pipeline role cannot assume the OpenTofu deploy role        | Check `pipeline-permissions.json` from Lab 8A                                     |
| `tofu init` fails with S3 access denied                   | Pipeline role cannot access remote state bucket             | Check state bucket permissions in the pipeline role policy                        |
| State lock error                                          | A previous OpenTofu run may have been interrupted           | Confirm no run is active, then use `tofu force-unlock <LOCK_ID>` only if safe     |
| `tofu plan` runs on merge or `tofu apply` runs on PR      | Workflow `if:` conditions are wrong                         | Compare the `if:` lines in `infra.yml`                                            |
| Pull request shows no workflow checks                     | Workflow may not be committed to `main` yet                 | Confirm `infra.yml` exists on the `main` branch                                   |
| Versioning is already enabled before merge                | The change was applied manually or by a previous run        | Check GitHub Actions history and local AWS changes                                |
| Destroy workflow has no confirm box                       | `workflow_dispatch.inputs.confirm` is missing or malformed  | Check the `destroy.yml` syntax and confirm it is on `main`                        |
| Destroy workflow runs even with wrong input               | Confirmation check is incorrect                             | Recheck the `Verify confirmation` shell script                                    |
| Git push fails                                            | Authentication or branch tracking issue                     | Confirm GitHub CLI authentication and run `git push -u origin <branch>`           |

## Cleanup

> Do not clean up this lab if continuing to Lab 8C.

The following resources should remain:

| Resource                      | Why It Stays                                          |
| ----------------------------- | ----------------------------------------------------- |
| GitHub Actions CI/CD workflow | Lab 8C builds on the pipeline                         |
| Guarded destroy workflow      | Used for safe cleanup later                           |
| Demo S3 bucket                | Used by the CI/CD workflow and Lab 8C drift detection |
| S3 bucket versioning          | Demonstrates the PR-to-merge apply workflow           |
| GitHub OIDC provider          | Required for GitHub Actions authentication            |
| `github-actions-infra` role   | Required by GitHub Actions                            |
| `workshop-tofu-deploy-role`   | Required by OpenTofu                                  |
| S3 remote-state bucket        | Stores OpenTofu state                                 |
| DynamoDB lock table           | Provides state locking                                |

### Full Cleanup Only If Stopping the CI/CD Track

Use the guarded destroy workflow with the correct confirmation word:

```text
GitHub repository
Actions
Destroy Infrastructure
Run workflow
confirm = destroy
Run workflow
```

Expected result:

```text
Tofu Destroy succeeds
Demo S3 bucket is deleted
Bucket versioning resource is removed from state
```

After the destroy workflow completes, optional cleanup from Lab 8A can remove:

* `github-actions-infra` IAM role
* GitHub OIDC provider, only if unused elsewhere
* GitHub `workshop-iac` repository
* Session 7 remote-state resources, only if fully ending the IaC track

## Cleanup Verification

| Resource                | Expected State  |
| ----------------------- | --------------- |
| Demo S3 bucket          | Kept for Lab 8C |
| S3 bucket versioning    | Kept for Lab 8C |
| CI/CD workflow          | Kept            |
| Manual destroy workflow | Kept            |
| GitHub OIDC provider    | Kept            |
| Pipeline role           | Kept            |
| OpenTofu deploy role    | Kept            |
| Remote state bucket     | Kept            |
| DynamoDB lock table     | Kept            |

## What I Learned

* GitHub Actions can run OpenTofu automatically from a Git repository.
* GitHub OIDC allows CI/CD authentication without storing AWS access keys.
* `tofu plan` on pull requests provides a safe review step before infrastructure changes are deployed.
* `tofu apply` on merge makes approved changes deploy automatically.
* Git `main` becomes the source of truth for deployed infrastructure.
* The same OpenTofu code can run locally and in CI/CD; only the authentication method changes.
* `permissions: id-token: write` is required for GitHub Actions OIDC authentication.
* Pull requests are useful for reviewing infrastructure changes before they affect AWS.
* A guarded destroy workflow is safer than automatic destroy behavior.
* Destructive automation should require deliberate human confirmation.
* CI/CD workflows improve consistency because deployments no longer depend on commands from one person’s laptop.
* This lab demonstrated a full GitOps cycle: branch, pull request, plan, merge, apply, and guarded destroy.

## Screenshots

| Screenshot                                            | Description                                            |
| ----------------------------------------------------- | ------------------------------------------------------ |
| `screenshots/workshop-iac-status-clean.png`           | Local `workshop-iac` repository was clean              |
| `screenshots/dev-main-tf-demo-bucket.png`             | Dev `main.tf` replaced with CI/CD demo bucket          |
| `screenshots/dev-outputs-updated.png`                 | Output updated to display demo bucket name             |
| `screenshots/github-workflows-folder.png`             | `.github/workflows` folder created                     |
| `screenshots/infra-workflow-created.png`              | `infra.yml` workflow created                           |
| `screenshots/workflow-committed-pushed.png`           | CI/CD workflow committed and pushed                    |
| `screenshots/actions-first-run.png`                   | First GitHub Actions run started                       |
| `screenshots/actions-apply-success.png`               | First apply workflow completed successfully            |
| `screenshots/demo-bucket-created.png`                 | Demo S3 bucket created by pipeline                     |
| `screenshots/add-versioning-branch.png`               | `add-versioning` branch created                        |
| `screenshots/versioning-code-added.png`               | S3 bucket versioning resource added                    |
| `screenshots/pull-request-created.png`                | Pull request opened                                    |
| `screenshots/pr-plan-success.png`                     | Pull request plan completed successfully               |
| `screenshots/pr-apply-skipped.png`                    | Apply step skipped on pull request                     |
| `screenshots/versioning-not-enabled-before-merge.png` | Versioning not enabled before merge                    |
| `screenshots/pr-merged.png`                           | Pull request merged                                    |
| `screenshots/merge-apply-success.png`                 | Apply workflow completed after merge                   |
| `screenshots/versioning-enabled.png`                  | S3 versioning enabled after merge                      |
| `screenshots/local-main-synced.png`                   | Local `main` synced after merge                        |
| `screenshots/destroy-workflow-created.png`            | Guarded destroy workflow created                       |
| `screenshots/destroy-workflow-wrong-confirmation.png` | Destroy workflow failed with wrong confirmation        |
| `screenshots/infrastructure-still-exists.png`         | Demo bucket still existed after failed destroy attempt |