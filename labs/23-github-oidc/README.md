# Lab 23: GitHub OIDC

## Lab Summary

In this lab, I connected my local `workshop-iac` Infrastructure as Code project to GitHub and configured secure, keyless authentication between GitHub Actions and AWS using OpenID Connect.

Before this lab, I ran OpenTofu commands locally from my laptop. In a real team environment, CI/CD pipelines usually run those commands automatically when code is pushed. The unsafe way to give a pipeline AWS access is to create long-lived AWS access keys and store them as GitHub secrets. This lab used the safer method: GitHub OIDC.

With OIDC, GitHub Actions can present a short-lived identity token to AWS. AWS validates the token and allows the workflow to assume a pipeline IAM role. This avoids storing long-lived AWS credentials in GitHub.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 8A — Connect Your Repo to GitHub with OIDC (No Stored Keys)
* Session: 8 — CI/CD for Infrastructure
* Track: Solutions Architecture + Infrastructure as Code
* Difficulty: Beginner
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Set the active AWS CLI profile
* Verify AWS CLI authentication
* Reopen the `workshop-iac` project from Session 7
* Install GitHub CLI if needed
* Authenticate to GitHub CLI
* Create a private GitHub repository for the `workshop-iac` project
* Connect the local `workshop-iac` Git repository to GitHub
* Push the local IaC project to GitHub
* Confirm state files were not uploaded
* Understand why long-lived AWS access keys should not be stored in GitHub
* Create a GitHub OIDC identity provider in AWS IAM
* Locate the GitHub OIDC subject claim prefix for the repository
* Create a pipeline trust policy
* Create an IAM role for GitHub Actions
* Attach least-privilege pipeline permissions
* Commit and push the OIDC policy files
* Preserve the GitHub repository, OIDC provider, and pipeline role for Lab 8B

## Services / Tools Used

| Service / Tool | Purpose                                                                 |
| -------------- | ----------------------------------------------------------------------- |
| GitHub         | Hosts the `workshop-iac` Infrastructure as Code repository              |
| GitHub Actions | Future CI/CD platform that will run OpenTofu commands                   |
| GitHub OIDC    | Allows GitHub Actions to authenticate to AWS without stored access keys |
| AWS IAM        | Creates the OIDC provider and pipeline role                             |
| AWS STS        | Issues temporary credentials through `AssumeRoleWithWebIdentity`        |
| AWS CLI        | Creates IAM OIDC provider, role, and permissions                        |
| Git            | Pushes local IaC code to GitHub                                         |
| GitHub CLI     | Helps authenticate Git operations through GitHub                        |
| VS Code        | Creates and edits policy files                                          |
| PowerShell     | Runs AWS CLI, Git, and GitHub CLI commands                              |
| OpenTofu       | Existing IaC tool from Labs 7A–7C that future pipelines will run        |

## Prerequisites

* Completed Lab 7A: OpenTofu Infrastructure as Code Foundation
* Completed Lab 7B: Lambda with Infrastructure as Code
* Completed Lab 7C: Event-Driven Architecture with Reusable Modules
* `workshop-iac` folder still exists on the Desktop
* S3 remote-state bucket still exists
* DynamoDB `terraform-locks` table still exists
* `workshop-tofu-deploy-role` still exists
* AWS CLI installed and authenticated
* Git installed
* GitHub account available
* GitHub CLI installed or available for installation
* VS Code or another code editor installed

## Cost Notice

Estimated cost: `$0.00`

| Service           | Cost Consideration                                                        |
| ----------------- | ------------------------------------------------------------------------- |
| AWS IAM           | OIDC provider and IAM roles are free                                      |
| GitHub Repository | Free for public repositories and available on GitHub free/private plans   |
| GitHub Actions    | No workflow is run in this lab; later labs may use GitHub Actions minutes |

## Key Concepts

| Concept                     | Meaning                                                                                                                      |
| --------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| CI/CD                       | Continuous Integration and Continuous Delivery. Automation that runs tests, plans, deployments, or checks when code changes. |
| GitHub Actions              | GitHub’s automation service that runs workflows from `.github/workflows/`.                                                   |
| OIDC                        | OpenID Connect, a standard for short-lived identity tokens between systems.                                                  |
| GitHub OIDC                 | GitHub Actions proving to AWS that a workflow is running from a specific GitHub repository.                                  |
| OIDC Provider               | AWS IAM configuration that trusts identity tokens from GitHub.                                                               |
| Pipeline Role               | IAM role that GitHub Actions will assume during a workflow.                                                                  |
| Trust Policy                | IAM role policy that controls who is allowed to assume the role.                                                             |
| Permission Policy           | IAM policy that controls what the role can do after it is assumed.                                                           |
| `AssumeRoleWithWebIdentity` | AWS STS action used when a web identity token, such as GitHub OIDC, is exchanged for temporary AWS credentials.              |
| Short-Lived Credentials     | Temporary credentials that expire after a short time instead of remaining valid until manually deleted.                      |
| Long-Lived Access Key       | Static AWS access key and secret key pair. Risky if stored in GitHub or leaked.                                              |
| Subject Claim               | OIDC token field that identifies the GitHub repository and workflow context.                                                 |
| Least Privilege             | Granting only the permissions needed for the pipeline role to do its job.                                                    |

## Security Notes

| Topic                                    | Explanation                                                                                           |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| Do not store AWS access keys in GitHub   | Long-lived AWS keys can leak through secrets, logs, screenshots, dependencies, or accidental commits. |
| OIDC uses temporary credentials          | GitHub Actions receives short-lived AWS credentials only during a workflow run.                       |
| Trust is scoped to one repo              | The OIDC trust policy should only allow workflows from the specific `workshop-iac` repository.        |
| `sub` claim must match exactly           | If the GitHub OIDC subject prefix is wrong, AWS will reject the workflow token.                       |
| Pipeline role stays minimal              | The pipeline role only needs to assume the OpenTofu deploy role and access remote state.              |
| Deploy role does the infrastructure work | The pipeline role does not need direct Lambda, S3 application, or IAM deployment permissions.         |
| State files must not be committed        | `.tfstate`, `.terraform/`, and generated plan files should remain excluded by `.gitignore`.           |
| Keep IaC repository private              | Infrastructure code can reveal account structure, resource names, and architecture details.           |

## Architecture Overview

```text
GitHub Actions Workflow
        |
        | requests OIDC token
        v
GitHub OIDC Token
        |
        | presented to AWS STS
        v
AWS IAM OIDC Provider
        |
        | validates token issuer, audience, and subject
        v
Pipeline Role: github-actions-infra
        |
        | allowed to assume
        v
OpenTofu Deploy Role: workshop-tofu-deploy-role
        |
        v
AWS Infrastructure Managed by OpenTofu
```

## Lab Steps

### Step 1: Set AWS Profile and Open the Project

**What I did:**

* Set my AWS CLI profile for the current PowerShell session.
* Verified my AWS identity.
* Reopened the `workshop-iac` project from Labs 7A–7C.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
aws sts get-caller-identity

cd ~\Desktop\workshop-iac
code .
```

**Expected result:**

* AWS CLI returned my active account identity.
* VS Code opened the `workshop-iac` project.

**Notes:**

* If the SSO session expired, I used:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Install and Authenticate GitHub CLI

**What I did:**

* Installed GitHub CLI if it was not already installed.
* Logged in to GitHub through the browser authentication flow.
* Verified GitHub CLI authentication.

**Install command for Windows:**

```powershell
winget install --id GitHub.cli
```

**Login command:**

```powershell
gh auth login
```

**Selected options:**

```text
GitHub.com
HTTPS
Login with a web browser
```

**Verification command:**

```powershell
gh auth status
```

**Expected result:**

* GitHub CLI showed that I was logged in to GitHub.

**Notes:**

* GitHub CLI helps avoid issues with Git asking for a password during `git push`.

---

## Part 1: Push the IaC Project to GitHub

### Step 3: Create a Private GitHub Repository

**What I did:**

* Created a new GitHub repository for the `workshop-iac` project.

**Repository settings:**

| Setting         | Value          |
| --------------- | -------------- |
| Repository name | `workshop-iac` |
| Visibility      | Private        |
| README          | Not selected   |
| `.gitignore`    | Not selected   |
| License         | Not selected   |

**Notes:**

* I did not add a README, `.gitignore`, or license from GitHub because the local repository already had files.
* Adding files from GitHub first could create a Git history conflict.
* I kept the repository private because infrastructure code can reveal resource names, configuration, and account structure.

---

### Step 4: Confirm Local Git Status

**What I did:**

* Checked that the local `workshop-iac` repository had no uncommitted changes.

**Command used:**

```powershell
cd ~\Desktop\workshop-iac
git status
```

**Expected result:**

```text
nothing to commit, working tree clean
```

**If changes existed:**

```powershell
git add .
git commit -m "Session 8 starting point"
```

**Result:**

* Local IaC files were committed before pushing to GitHub.

---

### Step 5: Connect Local Repo to GitHub

**What I did:**

* Added the GitHub repository as the remote origin.
* Set the branch name to `main`.
* Pushed the project to GitHub.

**Command used:**

```powershell
git remote add origin https://github.com/<YOUR_GITHUB_USERNAME>/workshop-iac.git
```

**If origin already existed:**

```powershell
git remote set-url origin https://github.com/<YOUR_GITHUB_USERNAME>/workshop-iac.git
```

**Set branch and push:**

```powershell
git branch -M main
git push -u origin main
```

**Expected result:**

```text
branch 'main' set up to track 'origin/main'
```

**Result:**

* The `workshop-iac` project was pushed to GitHub.

---

### Step 6: Verify Repository Contents

**What I did:**

* Refreshed the GitHub repository page in the browser.
* Confirmed that the `infra/` folder and policy files were visible.
* Confirmed that state files were not uploaded.

**Expected files/folders on GitHub:**

```text
infra/
.gitignore
tofu-trust-policy.json
tofu-permissions.json
```

**Files/folders that should NOT appear:**

```text
.terraform/
*.tfstate
*.tfstate.backup
*.tfplan
```

**Result:**

* Infrastructure code was pushed.
* Local state and provider files were not committed.

---

## Part 2: Understand the Problem

### Step 7: Review Why Long-Lived Keys Are Dangerous

**Scenario:**

The old way to give a pipeline AWS access is to create an IAM access key and store the access key ID and secret access key as GitHub repository secrets.

**Why this is risky:**

* Access keys are long-lived.
* They work until manually deleted or deactivated.
* If they leak once, an attacker may have lasting access.
* Keys can leak through logs, screenshots, dependencies, misconfigured workflows, or accidental commits.
* A pipeline should not depend on permanent credentials.

**Safer approach:**

* Use GitHub OIDC.
* GitHub Actions proves it is running from my specific repository.
* AWS returns short-lived temporary credentials.
* No AWS access key is stored in GitHub.

---

## Part 3: Set Up GitHub OIDC Trust

### Step 8: Create the GitHub OIDC Identity Provider

**What I did:**

* Created an IAM OIDC identity provider for GitHub Actions.

**Command used:**

```powershell
aws iam create-open-id-connect-provider `
  --url https://token.actions.githubusercontent.com `
  --client-id-list sts.amazonaws.com `
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

**Expected result:**

```json
{
  "OpenIDConnectProviderArn": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
}
```

**If provider already exists:**

```text
EntityAlreadyExists
```

**What I did if it already existed:**

* Treated this as acceptable.
* Continued to the next step and reused the existing provider.

**What this provider means:**

| Field      | Meaning                                             |
| ---------- | --------------------------------------------------- |
| URL        | GitHub’s OIDC token issuer                          |
| Client ID  | AWS STS audience value                              |
| Thumbprint | Certificate fingerprint required by the CLI command |

---

### Step 9: Find the GitHub OIDC Subject Prefix

**What I did:**

* Opened the `workshop-iac` repository in GitHub.
* Went to repository settings.
* Opened the Actions OIDC settings.
* Copied the default subject claim prefix.

**GitHub path:**

```text
workshop-iac repository
Settings
Actions
OIDC
Default subject claim prefix
```

**Possible subject prefix formats:**

```text
repo:<YOUR_GITHUB_USERNAME>/workshop-iac
```

Or:

```text
repo:<YOUR_GITHUB_USERNAME>@<ID>/workshop-iac@<ID>
```

**Result:**

```text
<GITHUB_OIDC_PREFIX>
```

**Notes:**

* The value must be copied exactly.
* If the value includes `@` and numeric IDs, those must be included.
* The trust policy will append `:*` to allow workflow runs from that repository.

---

### Step 10: Create Pipeline Trust Policy

**What I did:**

* Created a trust policy allowing only GitHub Actions from my `workshop-iac` repository to assume the pipeline role.

**File created:**

```text
pipeline-trust.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
                },
                "StringLike": {
                    "token.actions.githubusercontent.com:sub": "<GITHUB_OIDC_PREFIX>:*"
                }
            }
        }
    ]
}
```

**Replacements made:**

| Placeholder            | Replacement                                    |
| ---------------------- | ---------------------------------------------- |
| `<YOUR_ACCOUNT_ID>`    | My 12-digit AWS account ID                     |
| `<GITHUB_OIDC_PREFIX>` | Exact subject prefix from GitHub OIDC settings |

**Example:**

```json
"token.actions.githubusercontent.com:sub": "repo:janedoe/workshop-iac:*"
```

**Result:**

* Pipeline trust policy file was created.

**Notes:**

* The `aud` condition ensures the token is intended for AWS STS.
* The `sub` condition scopes access to my specific repository.
* If the `sub` value is wrong, GitHub Actions will not be able to assume the role.

---

### Step 11: Create Pipeline Role

**What I did:**

* Created an IAM role named `github-actions-infra`.
* Used the GitHub OIDC trust policy as the role trust policy.

**Command used:**

```powershell
aws iam create-role `
  --role-name github-actions-infra `
  --assume-role-policy-document file://pipeline-trust.json
```

**Expected result:**

* JSON output containing the IAM role ARN.

**Pipeline role ARN:**

```text
arn:aws:iam::<YOUR_ACCOUNT_ID>:role/github-actions-infra
```

**Result:**

* Pipeline role was created successfully.

**Notes:**

* This role will be used in Lab 8B by GitHub Actions.
* Record the role ARN because the workflow file will need it later.

---

### Step 12: Create Pipeline Permissions Policy

**What I did:**

* Created a permissions policy for the pipeline role.
* Allowed the pipeline role to assume the OpenTofu deploy role.
* Allowed access to the remote state S3 bucket.
* Allowed access to the DynamoDB lock table.

**File created:**

```text
pipeline-permissions.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AssumeDeployRole",
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-tofu-deploy-role"
        },
        {
            "Sid": "StateBucketAccess",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::workshop-tofu-state-<INITIALS>",
                "arn:aws:s3:::workshop-tofu-state-<INITIALS>/*"
            ]
        },
        {
            "Sid": "StateLockAccess",
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:DeleteItem"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:<YOUR_ACCOUNT_ID>:table/terraform-locks"
        }
    ]
}
```

**Replacements made:**

| Placeholder         | Replacement                                      |
| ------------------- | ------------------------------------------------ |
| `<YOUR_ACCOUNT_ID>` | My 12-digit AWS account ID                       |
| `<INITIALS>`        | The suffix used in my OpenTofu state bucket name |

**Result:**

* Pipeline permissions file was created.

**Notes:**

* The pipeline role does not directly manage Lambda, S3 application resources, or IAM application roles.
* It only assumes the deploy role and accesses remote state.
* The deploy role performs the actual infrastructure work.

---

### Step 13: Attach Pipeline Permissions

**What I did:**

* Attached the inline permissions policy to the `github-actions-infra` role.

**Command used:**

```powershell
aws iam put-role-policy `
  --role-name github-actions-infra `
  --policy-name pipeline-permissions `
  --policy-document file://pipeline-permissions.json
```

**Expected result:**

```text
No output
```

**Result:**

* Pipeline role permissions were attached successfully.

---

### Step 14: Commit and Push OIDC Files

**What I did:**

* Committed the new OIDC-related files to the `workshop-iac` repository.
* Pushed the changes to GitHub.

**Commands used:**

```powershell
cd ~\Desktop\workshop-iac

git add .
git commit -m "Add GitHub OIDC pipeline role and trust policy"
git push
```

**Expected result:**

* Commit was created successfully.
* Push completed successfully.

**Files committed:**

```text
pipeline-trust.json
pipeline-permissions.json
```

**Notes:**

* These files do not contain secrets.
* They define trust and permissions for the future GitHub Actions pipeline.

---

### Step 15: Console Checkpoint

**What I verified in AWS Console:**

| Console Area           | What I Checked                                                        |
| ---------------------- | --------------------------------------------------------------------- |
| IAM Identity Providers | GitHub OIDC provider exists for `token.actions.githubusercontent.com` |
| IAM Roles              | `github-actions-infra` role exists                                    |
| Role Trust Policy      | Trust policy uses `sts:AssumeRoleWithWebIdentity`                     |
| Role Trust Conditions  | Trust policy includes the correct `aud` and `sub` conditions          |
| Role Permissions       | Pipeline role has the `pipeline-permissions` inline policy            |

**What I verified in GitHub:**

| GitHub Area           | What I Checked                             |
| --------------------- | ------------------------------------------ |
| Repository            | `workshop-iac` exists                      |
| Repository visibility | Private                                    |
| Code tab              | `infra/` folder is present                 |
| Code tab              | OIDC policy files are present              |
| Code tab              | `.tfstate` files are not present           |
| OIDC settings         | Subject claim prefix was copied accurately |

## Validation / Checkpoints

| Checkpoint                                         | Result |
| -------------------------------------------------- | ------ |
| AWS CLI profile set                                | Passed |
| AWS identity verified                              | Passed |
| `workshop-iac` project reopened                    | Passed |
| GitHub CLI installed                               | Passed |
| GitHub CLI authenticated                           | Passed |
| Private `workshop-iac` GitHub repository created   | Passed |
| Local repo status checked                          | Passed |
| Local repo connected to GitHub remote              | Passed |
| Local repo pushed to GitHub                        | Passed |
| GitHub repository contents verified                | Passed |
| State files confirmed absent from GitHub           | Passed |
| Long-lived key risk reviewed                       | Passed |
| GitHub OIDC provider created or confirmed existing | Passed |
| GitHub OIDC subject prefix found                   | Passed |
| Pipeline trust policy created                      | Passed |
| Pipeline IAM role created                          | Passed |
| Pipeline permissions policy created                | Passed |
| Pipeline permissions attached                      | Passed |
| OIDC files committed and pushed                    | Passed |
| AWS IAM console verification completed             | Passed |
| GitHub repository verification completed           | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| GitHub CLI installed but `gh` was not recognized | The terminal session did not have GitHub CLI in its PATH yet, or PowerShell needed to be reopened after installation | Closed and reopened PowerShell, verified installation with `winget list --id GitHub.cli`, checked for `gh.exe`, and reran `gh --version` |

## Troubleshooting Notes

| Issue                                                                                     | What It Means                                                         | How to Fix                                                                                 |
| ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `git push` asks for a password                                                            | GitHub no longer accepts account passwords for HTTPS Git operations   | Use GitHub CLI browser login or a Personal Access Token                                    |
| Browser opened but push still hangs                                                       | Git credential authorization was not completed                        | Complete the browser authorization and rerun `git push -u origin main`                     |
| `remote origin already exists`                                                            | The local repo already has an origin remote configured                | Run `git remote set-url origin https://github.com/<YOUR_GITHUB_USERNAME>/workshop-iac.git` |
| Push is rejected                                                                          | GitHub repo may not be empty or histories may differ                  | Recreate the repo without README/gitignore/license or pull/rebase carefully                |
| `.tfstate` files appear on GitHub                                                         | `.gitignore` did not exclude state files or they were already tracked | Stop, remove them from Git tracking with `git rm --cached`, fix `.gitignore`, and recommit |
| `EntityAlreadyExists` when creating OIDC provider                                         | GitHub OIDC provider already exists in the AWS account                | Continue and reuse the existing provider                                                   |
| `MalformedPolicyDocument`                                                                 | JSON file contains invalid syntax or unreplaced placeholders          | Validate JSON and replace account ID, prefix, and bucket placeholders                      |
| `NoSuchEntity` for `workshop-tofu-deploy-role`                                            | The deploy role from Lab 7A is missing                                | Recreate the Lab 7A deploy role before continuing                                          |
| GitHub Actions later fails with `Not authorized to perform sts:AssumeRoleWithWebIdentity` | OIDC `sub` condition does not match the token GitHub sends            | Recheck the GitHub OIDC subject prefix and update `pipeline-trust.json`                    |
| SSO token expired                                                                         | AWS IAM Identity Center session expired                               | Run `aws sso login --profile <YOUR_PROFILE_NAME>`                                          |
| `AccessDenied` attaching pipeline policy                                                  | Current AWS identity lacks IAM permission                             | Confirm admin profile is active with `aws sts get-caller-identity`                         |

## Cleanup

> Do not clean up this lab if continuing to Lab 8B.

The following resources should remain:

| Resource                         | Why It Stays                                            |
| -------------------------------- | ------------------------------------------------------- |
| GitHub `workshop-iac` repository | Lab 8B will use it to run a GitHub Actions pipeline     |
| GitHub OIDC identity provider    | Allows GitHub Actions to authenticate to AWS            |
| `github-actions-infra` IAM role  | Pipeline role used by GitHub Actions                    |
| `pipeline-trust.json`            | Documents the role trust relationship                   |
| `pipeline-permissions.json`      | Documents the role permissions                          |
| `workshop-tofu-deploy-role`      | Existing deploy role that the pipeline role will assume |
| S3 remote-state bucket           | Stores OpenTofu state                                   |
| DynamoDB `terraform-locks` table | Provides state locking                                  |

### Full Cleanup Only If Stopping the CI/CD Track

Delete the pipeline role policy:

```powershell
aws iam delete-role-policy `
  --role-name github-actions-infra `
  --policy-name pipeline-permissions
```

Delete the pipeline role:

```powershell
aws iam delete-role `
  --role-name github-actions-infra
```

Delete the OIDC provider only if nothing else uses it:

```powershell
aws iam delete-open-id-connect-provider `
  --open-id-connect-provider-arn arn:aws:iam::<YOUR_ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

Delete the GitHub repository from:

```text
GitHub repository
Settings
General
Danger Zone
Delete this repository
```

## Cleanup Verification

| Resource                         | Expected State     |
| -------------------------------- | ------------------ |
| GitHub `workshop-iac` repository | Kept for Lab 8B    |
| GitHub OIDC provider             | Kept for Lab 8B    |
| `github-actions-infra` role      | Kept for Lab 8B    |
| Pipeline trust policy            | Kept in repository |
| Pipeline permissions policy      | Kept in repository |
| S3 remote-state bucket           | Kept               |
| DynamoDB lock table              | Kept               |
| OpenTofu deploy role             | Kept               |

## What I Learned

* CI/CD pipelines need a secure way to authenticate to AWS.
* Storing long-lived AWS access keys in GitHub is risky.
* GitHub OIDC allows GitHub Actions to authenticate to AWS without stored AWS keys.
* OIDC uses short-lived tokens and temporary credentials.
* AWS IAM can trust GitHub’s OIDC token issuer through an identity provider.
* The pipeline role trust policy controls which GitHub repository can assume the role.
* The OIDC `sub` claim is one of the most important security controls in this setup.
* `AssumeRoleWithWebIdentity` is used for OIDC-based role assumption.
* The pipeline role should have minimal permissions.
* The pipeline role can assume the existing OpenTofu deploy role instead of directly managing infrastructure.
* Remote state access is required because the pipeline needs to run OpenTofu against the same backend.
* State files must never be committed to GitHub.
* This lab prepares the foundation for Lab 8B, where GitHub Actions will run the actual pipeline.

## Screenshots

| Screenshot                                      | Description                               |
| ----------------------------------------------- | ----------------------------------------- |
| `screenshots/github-cli-authenticated.png`      | GitHub CLI authenticated successfully     |
| `screenshots/workshop-iac-reopened.png`         | Existing IaC project reopened             |
| `screenshots/github-repo-created.png`           | Private `workshop-iac` repository created |
| `screenshots/git-status-clean.png`              | Local repository clean before push        |
| `screenshots/git-remote-added.png`              | GitHub remote configured                  |
| `screenshots/git-push-success.png`              | Local IaC project pushed to GitHub        |
| `screenshots/github-repo-files.png`             | `infra/` folder visible on GitHub         |
| `screenshots/no-state-files-on-github.png`      | Confirmed no state files were uploaded    |
| `screenshots/oidc-provider-created.png`         | GitHub OIDC provider created or confirmed |
| `screenshots/github-oidc-subject-prefix.png`    | GitHub OIDC subject prefix found          |
| `screenshots/pipeline-trust-created.png`        | Pipeline trust policy file created        |
| `screenshots/pipeline-role-created.png`         | `github-actions-infra` IAM role created   |
| `screenshots/pipeline-permissions-created.png`  | Pipeline permissions file created         |
| `screenshots/pipeline-permissions-attached.png` | Inline policy attached to pipeline role   |
| `screenshots/iam-role-trust-policy.png`         | IAM role trust policy verified in console |
| `screenshots/iam-role-permissions.png`          | IAM role permissions verified in console  |
| `screenshots/oidc-files-committed.png`          | OIDC policy files committed and pushed    |