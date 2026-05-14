# Lab 01: AWS Account & CLI Setup

## Lab Summary

In this lab, I created and secured access to my AWS account, enabled IAM Identity Center, created a user and group, assigned administrator permissions, installed the AWS CLI, and configured CLI access using SSO.

This lab is the foundation for future AWS labs because it sets up secure console and command-line access.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 1A — Create Your AWS Account & Set Up Secure CLI Access
- My purpose: Document my own setup process, commands, notes, validation steps, and lessons learned.

## Objectives

- Create or access an AWS account
- Understand the purpose of the AWS root user
- Enable IAM Identity Center
- Create an IAM Identity Center user
- Create a group for administrative access
- Create and assign a permission set
- Install and verify the AWS CLI
- Configure AWS CLI access using SSO
- Confirm CLI access using `aws sts get-caller-identity`
- Set a default AWS CLI profile for the current terminal session

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Account | Personal cloud environment used for labs |
| AWS Root User | Owner account used only for initial account-level setup |
| IAM Identity Center | Secure user access and SSO login |
| AWS Organizations | Required when enabling IAM Identity Center |
| Permission Sets | Defines access level for users/groups |
| AWS CLI | Command-line tool used to interact with AWS services |
| PowerShell | Terminal used to run AWS CLI commands |

## Prerequisites

- Email address for AWS account verification
- Phone number for identity verification
- Credit/debit card for AWS account verification
- Computer running Windows
- PowerShell terminal
- Web browser
- Access to the AWS Management Console

## Cost Notice

Estimated cost: `$0.00`

Notes:

- Creating an AWS account is free.
- IAM Identity Center is free to use.
- AWS Organizations is free to use.
- AWS CLI is free to install and use.
- No billable AWS resources were created in this lab.
- Future labs may create resources, so cleanup steps will be important.

## Key Concepts

| Concept | Meaning |
|---|---|
| Root user | The owner of the AWS account with full access. It should not be used for daily work. |
| IAM Identity Center | AWS service used to manage secure user access to AWS accounts. |
| User | An identity created for daily AWS access. |
| Group | A collection of users that can receive permissions together. |
| Permission set | Defines what level of access a user or group receives. |
| AdministratorAccess | A high-level permission policy used for full administrative access in labs. |
| AWS CLI | Command-line tool used to interact with AWS from a terminal. |
| SSO | Single Sign-On method used to authenticate securely without long-term access keys. |
| AWS CLI profile | A named CLI configuration used to connect to an AWS account and role. |

## Lab Steps

### Step 1: Create / Access AWS Account

**What I did:**

- Created or accessed my AWS account.
- Used the root account only for the initial setup process.

**Result:**

- AWS account was available and ready for IAM Identity Center setup.

**Notes:**

- The root account should be treated like a master key.
- The root account should only be used for account-level tasks such as billing, support, payment methods, or account closure.
- Daily work should be done using a separate user.

---

### Step 2: Complete AWS Account Verification

**What I did:**

- Completed the required AWS account signup/verification steps.
- Verified email address.
- Added contact information.
- Added payment information for account verification.
- Completed phone verification.
- Selected the required support plan.

**Result:**

- AWS account signup was completed successfully.

**Notes:**

- AWS requires payment information for identity verification.
- This does not automatically mean charges will occur.
- Charges depend on which AWS resources are created and whether they remain running.

---

### Step 3: Sign In to the AWS Management Console

**What I did:**

- Signed in to the AWS Management Console.
- Used the root user only for initial configuration tasks.

**Result:**

- AWS Console access was confirmed.

**Notes:**

- After Identity Center is configured, I should use the Identity Center user instead of the root user.

---

### Step 4: Enable IAM Identity Center

**What I did:**

- Opened IAM Identity Center in the AWS Console.
- Enabled IAM Identity Center.
- Allowed AWS Organizations to be enabled/used as required.

**Result:**

- IAM Identity Center was enabled successfully.

**Notes:**

- IAM Identity Center provides a secure way to manage access.
- It is better than using the root account for regular work.
- It also avoids using long-term access keys for CLI authentication.

---

### Step 5: Create IAM Identity Center User

**What I did:**

- Created an IAM Identity Center user for daily AWS access.
- Entered the required user details.
- Completed the invitation or password setup process.

**Result:**

- A daily-use Identity Center user was created.

**Notes:**

- This user is separate from the AWS root user.
- This user should be used for regular AWS lab work.

---

### Step 6: Create Administrative Group

**What I did:**

- Created an administrative group.
- Added my Identity Center user to the group.

**Result:**

- The user was added to the administrative group.

**Notes:**

- Groups make permission management easier.
- Permissions can be assigned to a group instead of assigning permissions directly to each user.

---

### Step 7: Create Permission Set

**What I did:**

- Created or selected an administrator permission set.
- Used administrator-level permissions for the lab environment.

**Result:**

- Permission set was created successfully.

**Notes:**

- Administrator access is useful for beginner labs because it reduces permission issues.
- In real organizations, least privilege access should be used instead.

---

### Step 8: Assign Group to AWS Account

**What I did:**

- Assigned the administrative group to my AWS account.
- Attached the administrator permission set to the group.

**Result:**

- The Identity Center user received access to the AWS account.

**Notes:**

- This connects the user/group to the AWS account.
- Without this assignment, the user may exist but not have access to the account.

---

### Step 9: Test IAM Identity Center Login

**What I did:**

- Opened the AWS access portal.
- Signed in using the Identity Center user.
- Opened the AWS Management Console from the assigned account.

**Result:**

- Login worked using the Identity Center user instead of the root user.

**Notes:**

- This confirms that daily AWS Console access is working through IAM Identity Center.

---

### Step 10: Install AWS CLI

**What I did:**

- Installed AWS CLI v2 on Windows.
- Reopened PowerShell after installation.

**Command used:**

```bash
aws --version
```

**Result:**

- AWS CLI returned the installed version.

**Notes:**

- Reopening PowerShell may be required after installing AWS CLI so the terminal recognizes the `aws` command.

---

### Step 11: Configure AWS CLI with SSO

**What I did:**

- Started AWS CLI SSO configuration.
- Entered the required SSO details:
  - SSO start URL
  - SSO region
  - AWS account
  - Permission set/role
  - Default AWS region
  - Output format
  - CLI profile name

**Command used:**

```bash
aws configure sso
```

**Result:**

- AWS CLI profile was created successfully.

**Notes:**

- The profile name is important because it is used when running AWS CLI commands.
- The profile name should be easy to recognize.

---

### Step 12: Log in with AWS SSO

**What I did:**

- Logged into AWS from the CLI using my configured SSO profile.

**Command used:**

```bash
aws sso login --profile <PROFILE_NAME>
```

**Result:**

- Browser-based login opened.
- SSO authentication completed successfully.
- CLI session was authenticated.

**Notes:**

- If the SSO session expires, I need to run this command again.

---

### Step 13: Verify CLI Identity

**What I did:**

- Verified that the AWS CLI was connected to the correct AWS account and role.

**Command used:**

```bash
aws sts get-caller-identity --profile <PROFILE_NAME>
```

**Result:**

```json
{
  "UserId": "REDACTED",
  "Account": "REDACTED",
  "Arn": "arn:aws:sts::REDACTED:assumed-role/AWSReservedSSO_AdministratorAccess_.../REDACTED"
}
```

**Notes:**

- This confirms that AWS CLI authentication is working.
- Account ID and ARN details should be redacted before uploading screenshots or command output.

---

### Step 14: Set Default AWS Profile for Current Session

**What I did:**

- Set the AWS CLI profile as the default for the current PowerShell session.

**Command used:**

```powershell
$env:AWS_PROFILE="<PROFILE_NAME>"
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Result:**

- The identity command worked without typing `--profile`.

**Notes:**

- This only applies to the current terminal session.
- If PowerShell is closed, the profile may need to be set again.
- This is helpful because future AWS CLI commands can be shorter.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS account created or accessed | Passed |
| Root user access confirmed | Passed |
| IAM Identity Center enabled | Passed |
| Identity Center user created | Passed |
| Administrative group created | Passed |
| User added to group | Passed |
| Permission set created or selected | Passed |
| Group assigned to AWS account | Passed |
| Identity Center login tested | Passed |
| AWS CLI installed | Passed |
| `aws --version` worked | Passed |
| AWS CLI SSO configured | Passed |
| SSO login completed | Passed |
| `aws sts get-caller-identity` worked with profile | Passed |
| Default AWS profile worked in current session | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Cleanup

| Resource | Cleanup Action | Verified |
|---|---|---|
| AWS account | Keep for future labs | Yes |
| IAM Identity Center | Keep for future labs | Yes |
| Identity Center user | Keep for future labs | Yes |
| Identity Center group | Keep for future labs | Yes |
| Permission set | Keep for future labs | Yes |
| AWS CLI profile | Keep for future labs | Yes |

## What I Learned

- The AWS root user should not be used for daily work.
- IAM Identity Center provides a safer way to access AWS.
- Users, groups, and permission sets work together to control access.
- AWS CLI is an important tool for CloudOps and cloud administration.
- SSO authentication avoids the need for long-term access keys.
- `aws sts get-caller-identity` is useful for confirming which AWS identity is active.
- Setting `AWS_PROFILE` in PowerShell makes CLI commands easier to run during a session.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/aws-cli-version.png` | AWS CLI installed successfully |
| `screenshots/sso-login-success.png` | SSO login completed successfully |
| `screenshots/sts-caller-identity-redacted.png` | CLI identity verified with sensitive details redacted |
| `screenshots/default-profile-validation.png` | CLI worked after setting the default AWS profile |