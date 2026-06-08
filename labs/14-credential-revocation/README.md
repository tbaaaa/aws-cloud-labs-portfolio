# Lab 14: Credential Revocation

## Lab Summary

In this lab, I practiced responding to a compromised AWS access key. I created a test IAM user, created access keys for that user, simulated an alert that the key was exposed in a public GitHub repository, deactivated the key to contain the incident, verified that the inactive key could no longer authenticate, restored my admin credentials, and permanently deleted the compromised key.

This lab is important because exposed AWS access keys are a common cloud security incident. The first response should usually be fast containment by deactivating the key before deeper investigation.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 5A — Respond to a Compromised Access Key
- Session: 5 — Incident Response on AWS
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Create an IAM user to simulate a compromised identity
- Create access keys for the user
- Verify the access key is active
- Simulate detection of leaked credentials
- Deactivate the access key for containment
- Verify the key status changed to inactive
- Test that the inactive key can no longer authenticate
- Switch back to admin credentials
- Delete the compromised access key permanently
- Verify in the AWS Console that the user has no access keys
- Delete the IAM user
- Delete the local lab folder

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS IAM | Creates the test user and manages the access key |
| IAM User | Simulates a user whose access key was exposed |
| IAM Access Key | Simulates compromised programmatic credentials |
| AWS STS | Used with `get-caller-identity` to test whether credentials can authenticate |
| AWS CLI | Creates, deactivates, tests, and deletes IAM credentials |
| PowerShell | Terminal used to run AWS CLI commands |
| AWS Console | Used to verify the access key was deleted |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- Basic understanding of IAM users and access keys
- Access to the AWS Management Console

## Cost Notice

Estimated cost: `$0.00`

Notes:

- IAM is always free.
- This lab does not create billable compute, storage, or networking resources.
- Cleanup is still required because unused IAM users and access keys are a security risk.

## Key Concepts

| Concept | Meaning |
|---|---|
| Incident Response | The process of detecting, containing, eradicating, and recovering from a security event. |
| Detection | Discovering that something suspicious or harmful happened. |
| Containment | Stopping the threat from causing more damage. |
| Eradication | Removing the threat completely. |
| Recovery | Returning to normal operations safely. |
| Access Key | Long-term credential used by applications or CLI tools to access AWS programmatically. |
| Deactivate Key | Sets an access key to `Inactive`, preventing authentication while keeping the key record. |
| Delete Key | Permanently removes the access key. This cannot be reversed. |
| Credential Revocation | Disabling or removing credentials so they can no longer be used. |
| Environment Variables | Temporary shell variables that tell the AWS CLI which credentials to use. |

## Security Notes

| Topic | Explanation |
|---|---|
| Deactivate first | In a real incident, deactivate the exposed key immediately to stop unauthorized use. |
| Delete after confirmation | Delete the key only after confirming the compromise and completing required investigation steps. |
| Access keys are sensitive | Never commit access keys or secret keys to GitHub. |
| Secret key shown once | AWS only shows the secret access key at creation time. |
| Environment variable risk | Credentials stored in terminal environment variables should be cleared after testing. |
| Admin profile restoration | Always switch back to admin credentials before cleanup. |
| Real-world follow-up | In production, teams would also review CloudTrail logs, rotate credentials, and assess damage. |

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my current AWS identity.

**Command used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Validation command:**

```bash
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

**Result:**

- AWS CLI returned my active admin identity successfully.

**Notes:**

- If the SSO token is expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

- Created a local folder for this lab.
- Changed into the folder before running the remaining commands.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-5a
cd ~\Desktop\workshop-lab-5a
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-5a
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-5a`.

---

### Step 3: Create the Compromised User

**What I did:**

- Created an IAM user named `compromised-user`.

**Command used:**

```bash
aws iam create-user --user-name compromised-user
```

**Expected result:**

- JSON output showing the IAM user details.

**Result:**

- IAM user was created successfully.

**Notes:**

- This user simulates a developer or service account whose credentials were exposed.
- New IAM users have no permissions by default.

---

### Step 4: Create Access Keys for the User

**What I did:**

- Created access keys for `compromised-user`.

**Command used:**

```bash
aws iam create-access-key --user-name compromised-user
```

**Expected result:**

```json
{
  "AccessKey": {
    "UserName": "compromised-user",
    "AccessKeyId": "REDACTED",
    "SecretAccessKey": "REDACTED",
    "Status": "Active"
  }
}
```

**Result:**

- Access key was created successfully.

**Notes:**

- The `SecretAccessKey` is shown only once.
- Do not upload access keys or screenshots containing access keys to GitHub.
- Record the Access Key ID temporarily because it is needed for deactivation and deletion.

---

### Step 5: Verify the Key Is Active

**What I did:**

- Listed access keys for `compromised-user`.
- Confirmed the key status was `Active`.

**Command used:**

```bash
aws iam list-access-keys --user-name compromised-user --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**Expected result:**

```text
Id        Status
--------  ------
AKIA...   Active
```

**Result:**

- Access key was active.

---

### Step 6: Simulate Detection

**Scenario:**

A security alert reports that the access key for `compromised-user` was found in a public GitHub repository.

**What this means:**

- The access key should be treated as compromised.
- Automated scanners may find exposed keys very quickly.
- The first response should be containment.

**Result:**

- Incident response process began.

**Notes:**

- This is a simulation.
- No real credentials should be exposed.

---

### Step 7: Containment — Deactivate the Key

**What I did:**

- Deactivated the compromised access key by changing its status to `Inactive`.

**Command used:**

```bash
aws iam update-access-key --user-name compromised-user --access-key-id <ACCESS_KEY_ID> --status Inactive
```

**Expected result:**

```text
No output
```

**Result:**

- Access key was deactivated successfully.

**Why this matters:**

- Deactivation is immediate containment.
- The key still exists, but it can no longer authenticate.
- This step is reversible if the alert turns out to be a false positive.

---

### Step 8: Verify the Key Is Inactive

**What I did:**

- Checked the access key status again.

**Command used:**

```bash
aws iam list-access-keys --user-name compromised-user --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**Expected result:**

```text
Id        Status
--------  --------
AKIA...   Inactive
```

**Result:**

- Access key status changed to `Inactive`.

---

### Step 9: Test That the Key No Longer Works

#### Step 9a: Set Compromised Credentials

**What I did:**

- Temporarily set the compromised access key credentials as environment variables.
- Removed `AWS_PROFILE` so the AWS CLI would use the compromised access keys instead of my admin SSO profile.

**PowerShell commands used:**

```powershell
$env:AWS_ACCESS_KEY_ID="<ACCESS_KEY_ID>"
$env:AWS_SECRET_ACCESS_KEY="<SECRET_ACCESS_KEY>"
$env:AWS_DEFAULT_REGION="us-east-1"
Remove-Item Env:\AWS_PROFILE
```

**Result:**

- AWS CLI was temporarily configured to use the compromised credentials.

**Notes:**

- This is only for testing.
- Do not keep compromised credentials in the terminal longer than needed.

#### Step 9b: Try to Authenticate

**What I did:**

- Tried to authenticate using the deactivated access key.

**Command used:**

```bash
aws sts get-caller-identity 2>&1
```

**Expected result:**

```text
An error occurred (InvalidClientTokenId) when calling the GetCallerIdentity operation: The security token included in the request is invalid.
```

**Result:**

- Authentication failed.

**Why this is good:**

- The deactivated key can no longer be used.
- This proves containment worked.
- Even if an attacker has the key, they are locked out.

---

### Step 10: Switch Back to Admin Credentials

**What I did:**

- Cleared the compromised credential environment variables.
- Restored my admin AWS CLI profile.
- Verified that the CLI was using the admin role again.

**PowerShell commands used:**

```powershell
Remove-Item Env:\AWS_ACCESS_KEY_ID
Remove-Item Env:\AWS_SECRET_ACCESS_KEY
Remove-Item Env:\AWS_DEFAULT_REGION
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Validation command:**

```bash
aws sts get-caller-identity
```

**Expected result:**

- Output should show the admin role, not `compromised-user`.

**Result:**

- AWS CLI returned to admin credentials.

**Notes:**

- If the SSO token expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 11: Eradication — Delete the Key Entirely

**What I did:**

- Permanently deleted the compromised access key.

**Command used:**

```bash
aws iam delete-access-key --user-name compromised-user --access-key-id <ACCESS_KEY_ID>
```

**Expected result:**

```text
No output
```

**Result:**

- Access key was deleted successfully.

**Notes:**

- Deleting a key is permanent.
- In real incident response, delete after containment and investigation confirm the compromise.

---

### Step 12: Console Checkpoint

**What I did:**

- Opened the AWS Console.
- Went to IAM.
- Opened **Users**.
- Selected `compromised-user`.
- Opened the **Security credentials** tab.
- Checked the **Access keys** section.

**Expected result:**

- No access keys listed for `compromised-user`.

**Result:**

- The compromised user had no access keys.

**Notes:**

- This confirms the threat was eradicated.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| Admin identity verified | Passed |
| Local project folder created | Passed |
| IAM user created | Passed |
| Access key created | Passed |
| Access key initially active | Passed |
| Detection scenario reviewed | Passed |
| Access key deactivated | Passed |
| Access key status changed to inactive | Passed |
| Compromised credentials tested | Passed |
| Deactivated credentials failed authentication | Passed |
| Admin profile restored | Passed |
| Access key deleted | Passed |
| Console confirmed no access keys remain | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| `EntityAlreadyExists` when creating `compromised-user` | User already exists from a previous attempt | Delete the old user first or use a different test username |
| `NoSuchEntity` when deactivating the key | Wrong access key ID or wrong username | Confirm the Access Key ID from Step 4 and username `compromised-user` |
| `get-caller-identity` still shows admin after setting compromised keys | `AWS_PROFILE` is still overriding the access key environment variables | Run `Remove-Item Env:\AWS_PROFILE` and try again |
| `get-caller-identity` still shows compromised credentials after Step 10 | Compromised credential environment variables were not cleared | Remove `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and `AWS_DEFAULT_REGION`, then restore `AWS_PROFILE` |
| SSO token expired after switching back to admin | Admin profile uses IAM Identity Center and the cached token expired | Run `aws sso login --profile <YOUR_PROFILE_NAME>` |
| Cannot delete IAM user during cleanup | User still has access keys attached | List access keys, delete remaining keys, then delete the user |

## Cleanup

### Step 1: Delete IAM User

**What I did:**

- Deleted the `compromised-user` IAM user.

**Command used:**

```bash
aws iam delete-user --user-name compromised-user
```

**Expected result:**

```text
No output
```

**Result:**

- IAM user was deleted successfully.

**Notes:**

- If deletion fails because the user still has access keys, list and delete the keys first:

```bash
aws iam list-access-keys --user-name compromised-user
aws iam delete-access-key --user-name compromised-user --access-key-id <ACCESS_KEY_ID>
aws iam delete-user --user-name compromised-user
```

---

### Step 2: Delete Local Project Folder

**What I did:**

- Deleted the local project folder from my desktop.

**PowerShell command used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-5a
```

**Result:**

- Local project folder was removed.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| Compromised access key | Deleted | Yes |
| IAM user `compromised-user` | Deleted | Yes |
| Local lab folder | Deleted | Yes |

## What I Learned

- Exposed AWS access keys should be treated as compromised immediately.
- In incident response, speed matters.
- The first action should be containment, not a long investigation.
- Deactivating an access key stops it from authenticating but keeps the key record.
- Deleting an access key permanently removes it.
- Deactivate first, investigate second, delete after confirmation.
- Environment variables can temporarily force the AWS CLI to use different credentials.
- `AWS_PROFILE` can override access key environment variables, so it may need to be removed during testing.
- `aws sts get-caller-identity` is useful for confirming which credentials the CLI is using.
- Credential revocation is a core cloud incident response skill.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/project-folder-created.png` | Local project folder created |
| `screenshots/compromised-user-created.png` | IAM user `compromised-user` created |
| `screenshots/access-key-created-redacted.png` | Access key created with sensitive values redacted |
| `screenshots/access-key-active.png` | Access key status shown as active |
| `screenshots/access-key-inactive.png` | Access key status changed to inactive |
| `screenshots/inactive-key-authentication-failed.png` | Deactivated key failed to authenticate |
| `screenshots/admin-profile-restored.png` | AWS CLI switched back to admin profile |
| `screenshots/access-key-deleted-console.png` | IAM console confirmed no access keys remain |
| `screenshots/cleanup-verified.png` | IAM user and local resources cleaned up |