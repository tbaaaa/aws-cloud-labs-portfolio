# Lab 25: Drift Detection

## Lab Summary

In this lab, I created an automated drift detection workflow for my Infrastructure as Code pipeline.

In Lab 8B, I built a GitHub Actions pipeline that runs `tofu plan` on pull requests and `tofu apply` when changes are merged to `main`. That pipeline keeps AWS aligned with code when all changes go through Git. This lab focused on what happens when someone bypasses Git and makes a manual change directly in AWS.

I added a scheduled GitHub Actions workflow that runs `tofu plan -detailed-exitcode`. This command returns a special exit code when OpenTofu detects changes between the code and the real AWS environment. I then manually changed a managed S3 bucket tag to simulate drift, ran the workflow, verified that the workflow failed, and then triggered an apply to reconcile AWS back to the code.

This lab demonstrated the full drift lifecycle: clean state, drift created, drift detected, drift corrected, and clean state restored.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 8C — Drift Detection — Catching Manual Changes Automatically
* Session: 8 — CI/CD for Infrastructure
* Track: Solutions Architecture + Infrastructure as Code
* Difficulty: Advanced
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Reopen the `workshop-iac` repository
* Sync local `main` with GitHub
* Create a scheduled GitHub Actions drift detection workflow
* Configure the workflow to authenticate to AWS through GitHub OIDC
* Configure the workflow to install OpenTofu
* Configure `tofu_wrapper: false` so OpenTofu exit codes are preserved
* Run `tofu plan -detailed-exitcode`
* Commit and push the drift workflow
* Confirm the workflow appears in GitHub Actions
* Manually run the workflow to establish a green baseline
* Simulate drift by manually changing an S3 bucket tag outside of Git
* Run the drift workflow again
* Verify the workflow fails when drift is detected
* Review the OpenTofu plan output showing the tag mismatch
* Trigger an apply workflow to correct the drift
* Verify the S3 tag is restored to the value defined in code
* Run drift detection one final time
* Verify the workflow passes after reconciliation
* Document lessons learned and troubleshooting notes

## Services / Tools Used

| Service / Tool | Purpose                                                            |
| -------------- | ------------------------------------------------------------------ |
| GitHub Actions | Runs the scheduled and manual drift detection workflow             |
| GitHub OIDC    | Authenticates GitHub Actions to AWS without long-lived access keys |
| AWS IAM        | Provides the GitHub pipeline role and OpenTofu deploy role         |
| AWS STS        | Issues temporary credentials for the GitHub Actions workflow       |
| OpenTofu       | Compares code, state, and real infrastructure                      |
| Amazon S3      | Demo bucket used to simulate drift                                 |
| DynamoDB       | Provides OpenTofu remote-state locking                             |
| Git            | Commits and pushes the new workflow                                |
| AWS CLI        | Creates the manual drift and verifies the corrected S3 tag         |
| PowerShell     | Runs local Git and AWS CLI commands                                |

## Prerequisites

* Completed Lab 7A: OpenTofu Infrastructure as Code Foundation
* Completed Lab 7B: Lambda with Infrastructure as Code
* Completed Lab 7C: Event-Driven Architecture with Reusable Modules
* Completed Lab 8A: GitHub OIDC
* Completed Lab 8B: CI/CD Plan and Apply
* `workshop-iac` repository exists locally
* `workshop-iac` repository exists on GitHub
* GitHub Actions workflow from Lab 8B exists
* GitHub OIDC provider exists in AWS IAM
* `github-actions-infra` IAM role exists
* `workshop-tofu-deploy-role` IAM role exists
* S3 remote-state bucket exists
* DynamoDB `terraform-locks` table exists
* Demo S3 bucket from Lab 8B exists
* Demo S3 bucket versioning is enabled from Lab 8B
* AWS CLI is installed and authenticated locally

## Cost Notice

Estimated cost: `$0.00`

| Service        | Cost Consideration                                                       |
| -------------- | ------------------------------------------------------------------------ |
| GitHub Actions | Uses workflow minutes; expected to remain within free usage for this lab |
| Amazon S3      | Existing demo bucket from Lab 8B; expected cost is negligible            |
| DynamoDB       | Existing state-lock table; expected to remain within free usage          |
| AWS IAM        | Free                                                                     |
| OpenTofu       | Free and open source                                                     |

## Key Concepts

| Concept              | Meaning                                                                                                      |
| -------------------- | ------------------------------------------------------------------------------------------------------------ |
| Drift                | When real AWS infrastructure no longer matches the Infrastructure as Code configuration.                     |
| Drift Detection      | The process of checking whether deployed infrastructure still matches the code.                              |
| Manual Change        | A change made directly in the AWS Console, AWS CLI, or another tool instead of through Git and OpenTofu.     |
| Source of Truth      | The trusted place that defines infrastructure; in this lab, the OpenTofu code in Git is the source of truth. |
| `tofu plan`          | Compares the desired configuration, state, and real infrastructure to preview changes.                       |
| `-detailed-exitcode` | Special OpenTofu plan mode that returns exit code `0`, `1`, or `2`.                                          |
| Exit Code `0`        | No changes were detected. Code and reality match.                                                            |
| Exit Code `1`        | An actual error occurred.                                                                                    |
| Exit Code `2`        | Changes were detected. In this lab, that means drift exists.                                                 |
| Scheduled Workflow   | A GitHub Actions workflow that runs automatically on a cron schedule.                                        |
| `workflow_dispatch`  | A GitHub Actions trigger that allows a workflow to be manually started from the Actions tab.                 |
| Reconciliation       | Correcting infrastructure so the real environment matches the code again.                                    |
| GitOps               | A workflow where Git is the source of truth for infrastructure changes.                                      |

## Security Notes

| Topic                               | Explanation                                                                                                  |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| Manual changes create risk          | If someone changes AWS directly, the change may not be reviewed, approved, or documented in Git.             |
| Drift can hide security issues      | A manual change could weaken a configuration without appearing in the repository.                            |
| Scheduled checks improve visibility | Drift detection runs regularly even if no one remembers to check manually.                                   |
| Failure is intentional              | In this lab, a red workflow is good when drift exists because it alerts the team.                            |
| `tofu_wrapper: false` is critical   | Without this, the OpenTofu setup action may not pass the real `-detailed-exitcode` result to GitHub Actions. |
| Code remains the source of truth    | When drift is found, either revert the manual change or update the code intentionally.                       |
| OIDC remains keyless                | The drift workflow uses GitHub OIDC instead of stored AWS access keys.                                       |
| Remote state locking still matters  | DynamoDB locking prevents overlapping OpenTofu runs from corrupting state.                                   |

## Architecture Overview

```text
Scheduled or Manual GitHub Actions Run
        |
        v
GitHub OIDC Authentication
        |
        v
AWS Pipeline Role
        |
        v
OpenTofu Init
        |
        v
tofu plan -detailed-exitcode
        |
        +---------------------------+
        |                           |
        v                           v
Exit 0: No Drift             Exit 2: Drift Detected
Workflow Green               Workflow Red
```

## Lab Steps

### Step 1: Open Project and Sync Local Main

**What I did:**

* Opened the local `workshop-iac` project.
* Switched to the `main` branch.
* Pulled the latest changes from GitHub.
* Opened the project in VS Code.

**PowerShell commands used:**

```powershell
cd ~\Desktop\workshop-iac
git checkout main
git pull origin main
code .
```

**Expected result:**

* Local `main` was up to date with GitHub.
* The Lab 8B versioning change was present.
* The `.github/workflows/infra.yml` workflow existed.
* The `.github/workflows/destroy.yml` workflow existed.

**Result:**

* Local repository was synced and ready for the drift detection workflow.

---

## Part 1: Create Drift Detection Workflow

### Step 2: Create Drift Workflow File

**What I did:**

* Created a new GitHub Actions workflow file for drift detection.

**File created:**

```text
.github/workflows/drift.yml
```

**File content:**

```yaml
name: Drift Detection

on:
  schedule:
    - cron: "0 8 * * 1"
  workflow_dispatch: {}

permissions:
  id-token: write
  contents: read

jobs:
  drift:
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
        with:
          tofu_wrapper: false

      - name: Tofu Init
        run: tofu init

      - name: Check for drift
        run: tofu plan -detailed-exitcode
```

**Replacements made:**

| Placeholder         | Replacement                |
| ------------------- | -------------------------- |
| `<YOUR_ACCOUNT_ID>` | My 12-digit AWS account ID |

**Important configuration:**

| Workflow Line                  | Why It Matters                                               |
| ------------------------------ | ------------------------------------------------------------ |
| `schedule`                     | Runs the drift check automatically every Monday at 08:00 UTC |
| `workflow_dispatch`            | Allows manual testing from the GitHub Actions tab            |
| `id-token: write`              | Allows the workflow to request a GitHub OIDC token           |
| `role-to-assume`               | Uses the GitHub Actions IAM role from Lab 8A                 |
| `working-directory`            | Runs OpenTofu from the dev environment folder                |
| `tofu_wrapper: false`          | Preserves the real OpenTofu exit code                        |
| `tofu plan -detailed-exitcode` | Makes drift detection return a pass/fail signal              |

**Result:**

* Drift detection workflow file was created.

---

### Step 3: Commit and Push the Drift Workflow

**What I did:**

* Committed the new workflow file.
* Pushed it to GitHub.

**Commands used from the project root:**

```powershell
git add .
git commit -m "Add scheduled drift detection workflow"
git push
```

**Expected result:**

* The new `drift.yml` workflow was pushed to GitHub.
* Because Lab 8B’s `infra.yml` runs on pushes to `main`, the Infrastructure CI/CD workflow also runs.
* Since this commit only adds a workflow file and does not change infrastructure, the apply workflow should report no infrastructure changes.

**Result:**

* Drift detection workflow was available in GitHub Actions.

---

### Step 4: Confirm Drift Workflow Appears in GitHub

**What I did:**

* Opened the GitHub repository.
* Clicked the **Actions** tab.
* Confirmed that **Drift Detection** appeared in the workflow sidebar.

**Expected result:**

```text
Drift Detection
```

**Result:**

* The workflow appeared in GitHub Actions.

---

## Part 2: Establish a Clean Baseline

### Step 5: Run Drift Detection Before Creating Drift

**What I did:**

* Manually ran the drift detection workflow before making any manual AWS changes.

**GitHub steps:**

```text
GitHub repository
Actions
Drift Detection
Run workflow
Run workflow
```

**Expected result:**

| Step                               | Expected Status |
| ---------------------------------- | --------------- |
| Checkout code                      | Success         |
| Configure AWS credentials via OIDC | Success         |
| Setup OpenTofu                     | Success         |
| Tofu Init                          | Success         |
| Check for drift                    | Success         |

**Expected plan result:**

```text
No changes. Your infrastructure matches the configuration.
```

**Result:**

* Drift workflow passed.
* This confirmed a clean baseline.

**What this proved:**

* GitHub Actions could authenticate to AWS through OIDC.
* OpenTofu could initialize remote state.
* The real AWS environment matched the code.

---

## Part 3: Create Drift Manually

### Step 6: Manually Change the S3 Bucket Tag

**What I did:**

* Simulated an engineer bypassing Git by changing a managed S3 bucket tag directly through the AWS CLI.
* Changed the `ManagedBy` tag from `opentofu` to `SOMEONE-CHANGED-THIS-BY-HAND`.

**PowerShell command used:**

```powershell
aws s3api put-bucket-tagging `
  --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> `
  --tagging "TagSet=[{Key=Project,Value=workshop-iac},{Key=Environment,Value=dev},{Key=ManagedBy,Value=SOMEONE-CHANGED-THIS-BY-HAND}]"
```

**Expected result:**

```text
No output
```

**Result:**

* The S3 bucket tag was changed outside of Git and OpenTofu.

**What this means:**

* The OpenTofu code still says:

```text
ManagedBy = opentofu
```

* The real AWS bucket now says:

```text
ManagedBy = SOMEONE-CHANGED-THIS-BY-HAND
```

* Code and reality no longer match.
* This is infrastructure drift.

---

### Step 7: Confirm the Manual Tag Change

**What I did:**

* Verified that the S3 bucket tag had been changed manually.

**Command used:**

```powershell
aws s3api get-bucket-tagging `
  --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> `
  --query "TagSet[?Key=='ManagedBy'].Value" `
  --output text
```

**Expected result:**

```text
SOMEONE-CHANGED-THIS-BY-HAND
```

**Result:**

* The manual tag change was confirmed.

---

## Part 4: Detect the Drift

### Step 8: Run Drift Detection Again

**What I did:**

* Manually triggered the drift detection workflow again.

**GitHub steps:**

```text
GitHub repository
Actions
Drift Detection
Run workflow
Run workflow
```

**Expected result:**

* The workflow should fail.
* The **Check for drift** step should show the planned correction.
* The red failure is expected because drift was detected.

**Expected workflow status:**

```text
Failed
```

**Result:**

* Drift detection failed as expected.

---

### Step 9: Review the Drift Plan Output

**What I did:**

* Opened the failed GitHub Actions run.
* Opened the **Check for drift** step.
* Reviewed the OpenTofu plan output.

**Expected plan output example:**

```text
# aws_s3_bucket.demo will be updated in-place
~ tags = {
    ~ "ManagedBy" = "SOMEONE-CHANGED-THIS-BY-HAND" -> "opentofu"
  }

Plan: 0 to add, 1 to change, 0 to destroy.
```

**Result:**

* OpenTofu detected that the real S3 bucket tag no longer matched the code.
* The workflow failed because `tofu plan -detailed-exitcode` returned exit code `2`.

**What this proved:**

* Manual AWS changes can be detected automatically.
* The workflow correctly turned drift into a visible failure.
* The red pipeline is a useful alert, not a broken pipeline.

---

## Part 5: Resolve the Drift

### Step 10: Trigger Apply to Correct Drift

**What I did:**

* Created an empty commit on `main`.
* Pushed it to GitHub to trigger the Lab 8B apply pipeline.
* The apply pipeline reconciled the S3 bucket tag back to the code-defined value.

**Commands used from project root:**

```powershell
git commit --allow-empty -m "Trigger apply to correct drift"
git push
```

**Expected result:**

* The **Infrastructure CI/CD** workflow runs.
* The **Tofu Apply** step runs.
* OpenTofu changes the `ManagedBy` tag back to `opentofu`.

**Result:**

* Apply workflow completed successfully.
* Drift was corrected.

**Alternative method:**

```powershell
cd ~\Desktop\workshop-iac\infra\environments\dev
tofu apply
```

**Notes:**

* In this lab, I used the pipeline method to keep the correction flowing through GitHub Actions.
* The code remained the source of truth.

---

### Step 11: Verify Tag Was Corrected

**What I did:**

* Checked the S3 bucket tag again from the terminal.

**Command used:**

```powershell
aws s3api get-bucket-tagging `
  --bucket workshop-dev-cicd-demo-<YOUR_ACCOUNT_ID> `
  --query "TagSet[?Key=='ManagedBy'].Value" `
  --output text
```

**Expected result:**

```text
opentofu
```

**Result:**

* The S3 bucket tag was restored to the value defined in code.

---

### Step 12: Run Final Drift Detection

**What I did:**

* Ran the drift detection workflow one final time.

**GitHub steps:**

```text
GitHub repository
Actions
Drift Detection
Run workflow
Run workflow
```

**Expected result:**

| Step                               | Expected Status |
| ---------------------------------- | --------------- |
| Checkout code                      | Success         |
| Configure AWS credentials via OIDC | Success         |
| Setup OpenTofu                     | Success         |
| Tofu Init                          | Success         |
| Check for drift                    | Success         |

**Expected plan result:**

```text
No changes. Your infrastructure matches the configuration.
```

**Result:**

* Drift detection passed.
* The environment returned to a clean state.

---

## Validation / Checkpoints

| Checkpoint                                       | Result |
| ------------------------------------------------ | ------ |
| Local `workshop-iac` repo opened                 | Passed |
| Local `main` synced with GitHub                  | Passed |
| `drift.yml` workflow created                     | Passed |
| OIDC authentication configured in drift workflow | Passed |
| `tofu_wrapper: false` configured                 | Passed |
| `tofu plan -detailed-exitcode` configured        | Passed |
| Drift workflow committed and pushed              | Passed |
| Drift workflow appeared in GitHub Actions        | Passed |
| Baseline drift run passed                        | Passed |
| Manual S3 tag change made                        | Passed |
| Manual tag change verified                       | Passed |
| Drift workflow failed after manual change        | Passed |
| Drift output reviewed                            | Passed |
| Drift corrected through apply pipeline           | Passed |
| `ManagedBy` tag restored to `opentofu`           | Passed |
| Final drift workflow passed                      | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                              | What It Means                                                    | How to Fix                                                                                      |
| -------------------------------------------------- | ---------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Drift run shows green even after manual tag change | `tofu_wrapper: false` may be missing or placed incorrectly       | Confirm it is under the `Setup OpenTofu` step’s `with:` block                                   |
| `Run workflow` button does not appear              | `workflow_dispatch` is missing or workflow file is not on `main` | Confirm `workflow_dispatch: {}` exists and push `drift.yml` to `main`                           |
| Drift workflow fails at OIDC step                  | GitHub Actions cannot assume the pipeline role                   | Recheck `id-token: write`, the pipeline role ARN, and the OIDC trust policy from Lab 8A         |
| `tofu init` fails with S3 access denied            | Pipeline role cannot access remote state                         | Check `pipeline-permissions.json` and the S3 state bucket name                                  |
| `tofu plan` fails with state lock error            | Another OpenTofu run may still hold the DynamoDB lock            | Wait for other GitHub Actions runs to finish; use `tofu force-unlock <LOCK_ID>` only if safe    |
| Manual tag change does not cause drift             | The changed tag may not be managed by the code                   | Use the exact lab command that changes the managed `ManagedBy` tag                              |
| Apply pipeline does not run after empty commit     | Push did not go to `main` or workflow trigger is wrong           | Confirm branch is `main` and `.github/workflows/infra.yml` triggers on push to `main`           |
| Tag is not restored after apply                    | Apply failed or the code does not define the expected tag        | Check the Infrastructure CI/CD run logs and confirm default tags in `main.tf`                   |
| Drift still appears after correction               | AWS may not have updated yet or another manual change exists     | Wait briefly, rerun drift detection, and review the plan output                                 |
| AWS CLI command fails locally                      | Local SSO session expired                                        | Run `aws sso login --profile <YOUR_PROFILE_NAME>` and verify with `aws sts get-caller-identity` |

## Cleanup

This lab is the end of the Session 8 CI/CD track.

The following resources can remain if I want to keep the project as a portfolio piece:

| Resource                         | Why It Can Stay                                           |
| -------------------------------- | --------------------------------------------------------- |
| GitHub `workshop-iac` repository | Demonstrates a professional IaC and CI/CD workflow        |
| GitHub OIDC provider             | Supports keyless GitHub Actions authentication            |
| `github-actions-infra` IAM role  | Allows the GitHub Actions pipeline to authenticate to AWS |
| `workshop-tofu-deploy-role`      | Used by OpenTofu for deployments                          |
| S3 remote-state bucket           | Stores OpenTofu state                                     |
| DynamoDB `terraform-locks` table | Provides state locking                                    |
| Drift detection workflow         | Demonstrates automated infrastructure consistency checks  |
| Guarded destroy workflow         | Provides a safer manual cleanup path                      |

### Optional Cleanup If Ending the CI/CD Track

#### Step 1: Destroy Application Resources

Use the guarded destroy workflow from Lab 8B:

```text
GitHub repository
Actions
Destroy Infrastructure
Run workflow
confirm = destroy
Run workflow
```

**Expected result:**

* OpenTofu destroys the demo S3 bucket and related resources.

**Alternative local cleanup:**

```powershell
cd ~\Desktop\workshop-iac\infra\environments\dev
tofu destroy
```

#### Step 2: Remove Pipeline Role

```powershell
aws iam delete-role-policy `
  --role-name github-actions-infra `
  --policy-name pipeline-permissions

aws iam delete-role `
  --role-name github-actions-infra
```

#### Step 3: Remove GitHub OIDC Provider

Only delete the provider if no other repositories or workflows use it:

```powershell
aws iam delete-open-id-connect-provider `
  --open-id-connect-provider-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

#### Step 4: Remove Session 7 Foundation

Follow the full cleanup from Lab 7A to delete:

* S3 remote-state bucket and all object versions
* DynamoDB `terraform-locks` table
* `workshop-tofu-deploy-role`
* Local `workshop-iac` folder, only if no longer needed

#### Step 5: Delete GitHub Repository

Only delete the GitHub repository if I do not want to keep it as a portfolio artifact:

```text
GitHub repository
Settings
General
Danger Zone
Delete this repository
```

## Cleanup Verification

| Resource                      | Expected State                                                     |
| ----------------------------- | ------------------------------------------------------------------ |
| Demo S3 bucket                | Optional: kept for portfolio or destroyed through guarded workflow |
| Drift workflow                | Kept if preserving the project                                     |
| Infrastructure CI/CD workflow | Kept if preserving the project                                     |
| Guarded destroy workflow      | Kept if preserving the project                                     |
| GitHub OIDC provider          | Kept unless no longer used                                         |
| `github-actions-infra` role   | Kept unless no longer used                                         |
| OpenTofu deploy role          | Kept unless fully ending the IaC track                             |
| Remote state bucket           | Kept unless fully ending the IaC track                             |
| DynamoDB lock table           | Kept unless fully ending the IaC track                             |
| GitHub `workshop-iac` repo    | Kept as portfolio evidence                                         |

## What I Learned

* Infrastructure drift happens when real AWS resources no longer match the Infrastructure as Code configuration.
* Manual changes outside of Git can break the trustworthiness of an IaC workflow.
* A scheduled GitHub Actions workflow can check for drift even when no code changes are being made.
* `tofu plan -detailed-exitcode` turns drift detection into a pipeline signal.
* Exit code `0` means no drift, exit code `2` means changes were detected, and exit code `1` means an error occurred.
* A failed drift detection workflow can be a successful security and operations signal.
* `tofu_wrapper: false` is important because the workflow needs the real OpenTofu exit code.
* Drift should be resolved by either reverting the manual change or intentionally updating the code.
* The code should remain the source of truth for infrastructure.
* GitHub OIDC allows drift checks to run without storing AWS access keys.
* Automated drift detection improves operational visibility and helps teams catch out-of-band changes.
* Session 8 created a professional CI/CD flow: keyless authentication, plan on pull request, apply on merge, guarded destroy, and scheduled drift detection.

## Screenshots

| Screenshot                                       | Description                                         |
| ------------------------------------------------ | --------------------------------------------------- |
| `screenshots/workshop-iac-main-synced.png`       | Local `workshop-iac` main branch synced             |
| `screenshots/drift-workflow-created.png`         | `drift.yml` workflow created                        |
| `screenshots/tofu-wrapper-false.png`             | `tofu_wrapper: false` configured                    |
| `screenshots/drift-workflow-pushed.png`          | Drift workflow committed and pushed                 |
| `screenshots/actions-drift-workflow-visible.png` | Drift Detection workflow visible in GitHub Actions  |
| `screenshots/baseline-drift-run-green.png`       | Initial drift detection run passed                  |
| `screenshots/manual-tag-change-command.png`      | Manual S3 tag change command executed               |
| `screenshots/manual-tag-change-verified.png`     | `ManagedBy` tag changed to manual value             |
| `screenshots/drift-run-failed.png`               | Drift detection workflow failed after manual change |
| `screenshots/drift-plan-output.png`              | OpenTofu plan showed tag drift                      |
| `screenshots/empty-commit-trigger-apply.png`     | Empty commit created to trigger apply               |
| `screenshots/apply-corrected-drift.png`          | Infrastructure CI/CD apply corrected the drift      |
| `screenshots/tag-restored-opentofu.png`          | `ManagedBy` tag restored to `opentofu`              |
| `screenshots/final-drift-run-green.png`          | Final drift detection run passed                    |
| `screenshots/session-8-complete.png`             | Session 8 CI/CD track completed                     |