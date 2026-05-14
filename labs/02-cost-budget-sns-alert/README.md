# Lab 02: Cost Budget & SNS Alert

## Lab Summary

In this lab, I created an AWS cost budget and connected it to an Amazon SNS email notification. The goal was to monitor AWS spending and receive an alert if costs approached the configured budget threshold.

This is an important CloudOps task because cost monitoring helps prevent unexpected cloud charges.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 1B — Create a Cost Budget Alert with Email Notification
- My purpose: Document my own process, commands, files created, validation steps, troubleshooting, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI identity
- Create a local lab working folder
- Create an SNS topic
- Subscribe an email address to the SNS topic
- Confirm the email subscription
- Create a budget definition file
- Create a notification configuration file
- Create the AWS Budget using the CLI
- Verify the budget and SNS topic
- Clean up all lab resources
- Verify cleanup

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| AWS Budgets | Tracks AWS spending against a defined budget |
| Amazon SNS | Sends notifications to subscribed endpoints |
| Email | Receives budget alert notifications |
| AWS CLI | Creates and verifies AWS resources from the terminal |
| PowerShell | Terminal used to run commands |
| JSON | Used to define the budget and notification settings |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- AWS CLI installed
- AWS SSO profile configured
- AWS CLI authentication working
- AWS account ID available
- Email address available for SNS confirmation
- PowerShell terminal
- Access to AWS Billing and Cost Management

## Cost Notice

Estimated cost: `$0.00`

Notes:

- AWS Budgets includes a limited number of free budgets.
- SNS email notifications are free within the free monthly allowance.
- The configured budget amount was intentionally low for lab purposes.
- Cleanup is required so temporary lab resources do not remain active.

## Key Concepts

| Concept | Meaning |
|---|---|
| AWS Budgets | Service used to track cost or usage against a defined limit. |
| Budget threshold | The point where AWS should trigger an alert. |
| Amazon SNS | Simple Notification Service used to send messages to subscribers. |
| SNS topic | A communication channel used to publish notifications. |
| SNS subscription | Connects an endpoint, such as an email address, to an SNS topic. |
| `budget.json` | Local JSON file used to define the budget settings. |
| `notifications.json` | Local JSON file used to define when and where budget alerts are sent. |
| Actual cost alert | Alert based on real accumulated spending. |

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.

**Command used:**

```powershell
$env:AWS_PROFILE="<PROFILE_NAME>"
```

**Result:**

- The AWS CLI used my selected profile for commands in the current terminal session.

**Notes:**

- This helps avoid typing `--profile <PROFILE_NAME>` on every command.
- This setting only applies to the current PowerShell session.

---

### Step 2: Verify AWS CLI Identity

**What I did:**

- Verified that my terminal was authenticated to AWS.

**Command used:**

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

- AWS CLI returned my active identity successfully.

**Notes:**

- Account ID and ARN details should be redacted before uploading screenshots or command output.
- If the SSO session has expired, run:

```bash
aws sso login --profile <PROFILE_NAME>
```

---

### Step 3: Create Local Lab Folder

**What I did:**

- Created a temporary local folder for the lab files.
- Changed into that folder before creating JSON files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-1b
cd ~\Desktop\workshop-lab-1b
pwd
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-1b`.

**Notes:**

- This folder was used to store local lab files such as `budget.json` and `notifications.json`.

---

### Step 4: Create SNS Topic

**What I did:**

- Created an SNS topic named `workshop-budget-alert`.

**Command used:**

```bash
aws sns create-topic --name workshop-budget-alert --region us-east-1
```

**Expected result:**

```json
{
  "TopicArn": "arn:aws:sns:us-east-1:REDACTED:workshop-budget-alert"
}
```

**Result:**

- SNS topic was created successfully.

**Notes:**

- The `TopicArn` was needed for later steps.
- The account ID inside the Topic ARN should be redacted in documentation and screenshots.

---

### Step 5: Subscribe Email Address to SNS Topic

**What I did:**

- Subscribed my email address to the SNS topic.

**Command used:**

```bash
aws sns subscribe --topic-arn <YOUR_TOPIC_ARN> --protocol email --notification-endpoint <YOUR_EMAIL_ADDRESS> --region us-east-1
```

**Expected result:**

```json
{
  "SubscriptionArn": "pending confirmation"
}
```

**Result:**

- Email subscription was created with a pending confirmation status.

**Notes:**

- `pending confirmation` is expected at this stage.
- The subscription does not become active until the email confirmation link is clicked.
- Email addresses should be redacted before uploading screenshots.

---

### Step 6: Confirm Email Subscription

**What I did:**

- Opened my email inbox.
- Found the AWS SNS subscription confirmation email.
- Clicked the confirmation link.
- Confirmed that the subscription was active.

**Result:**

- SNS email subscription was confirmed successfully.

**Notes:**

- If the email does not appear, check spam/junk.
- SNS cannot send email notifications to the address until the subscription is confirmed.

---

### Step 7: Create Budget Definition File

**What I did:**

- Created a file named `budget.json`.
- Defined a monthly cost budget.

**File created:**

```json
{
  "BudgetName": "workshop-monthly-budget",
  "BudgetLimit": {
    "Amount": "1",
    "Unit": "USD"
  },
  "BudgetType": "COST",
  "TimeUnit": "MONTHLY"
}
```

**Result:**

- `budget.json` was created and saved in the local lab folder.

**Notes:**

- The budget amount was set to `$1`.
- The budget type was `COST`.
- The time unit was `MONTHLY`.

---

### Step 8: Create Notification Configuration File

**What I did:**

- Created a file named `notifications.json`.
- Configured the budget alert to trigger at 80% of actual spending.
- Connected the alert to the SNS topic.

**File created:**

```json
[
  {
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80,
      "ThresholdType": "PERCENTAGE"
    },
    "Subscribers": [
      {
        "SubscriptionType": "SNS",
        "Address": "<YOUR_TOPIC_ARN>"
      }
    ]
  }
]
```

**Result:**

- `notifications.json` was created and saved in the local lab folder.

**Notes:**

- The alert triggers when actual spending is greater than 80% of the budget.
- Since the budget was `$1`, the alert threshold would be `$0.80`.
- The real SNS Topic ARN should not be committed publicly if it includes sensitive account information.

---

### Step 9: Create AWS Budget

**What I did:**

- Created the AWS Budget using the local JSON files.

**Command used:**

```bash
aws budgets create-budget --account-id <YOUR_ACCOUNT_ID> --budget file://budget.json --notifications-with-subscribers file://notifications.json
```

**Result:**

- The budget was created successfully.

**Notes:**

- This command may return no output when successful.
- No output does not always mean failure.
- If a duplicate budget error appears, delete the existing budget first or use a different budget name.

---

### Step 10: Verify Budget from CLI

**What I did:**

- Verified that the budget existed using the AWS CLI.

**Command used:**

```bash
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --output table
```

**Expected result:**

- Budget name: `workshop-monthly-budget`
- Budget amount: `$1.00`
- Budget type: `COST`
- Time unit: `MONTHLY`

**Result:**

- Budget appeared successfully in the CLI output.

---

### Step 11: Verify Budget from AWS Console

**What I did:**

- Opened AWS Billing and Cost Management.
- Viewed the Budgets section.
- Confirmed that the budget appeared in the AWS Console.

**Result:**

- Budget was visible in the AWS Console.

**Notes:**

- Console verification is useful because it confirms the resource exists outside of the CLI.

---

### Step 12: Verify SNS Topic

**What I did:**

- Listed SNS topics to confirm the topic existed.

**Command used:**

```bash
aws sns list-topics --region us-east-1
```

**Expected result:**

- `workshop-budget-alert` appeared in the list of SNS topics.

**Result:**

- SNS topic was confirmed.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| `aws sts get-caller-identity` worked | Passed |
| Local lab folder created | Passed |
| SNS topic created | Passed |
| Email subscription created | Passed |
| Email subscription confirmed | Passed |
| `budget.json` created | Passed |
| `notifications.json` created | Passed |
| AWS Budget created | Passed |
| Budget verified from CLI | Passed |
| Budget verified from AWS Console | Passed |
| SNS topic verified | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| PowerShell could not delete the `workshop-lab-1b` folder | The folder or one of its files was still open in PowerShell, File Explorer, VS Code, or another program | Closed anything using the folder, moved PowerShell out of the folder, then reran the delete command |

## Cleanup

### Step 1: Delete Budget

**What I did:**

- Deleted the temporary budget created for the lab.

**Command used:**

```bash
aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name "workshop-monthly-budget"
```

**Result:**

- Budget was deleted successfully.

---

### Step 2: Delete SNS Topic

**What I did:**

- Deleted the SNS topic created for the lab.

**Command used:**

```bash
aws sns delete-topic --topic-arn <YOUR_TOPIC_ARN> --region us-east-1
```

**Result:**

- SNS topic was deleted successfully.

---

### Step 3: Verify Budget Cleanup

**What I did:**

- Checked whether the budget still existed.

**Command used:**

```bash
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --output json
```

**Result:**

- The deleted budget no longer appeared.

---

### Step 4: Verify SNS Cleanup

**What I did:**

- Checked whether the SNS topic still existed.

**Command used:**

```bash
aws sns list-topics --region us-east-1
```

**Result:**

- The deleted SNS topic no longer appeared.

---

### Step 5: Delete Local Lab Folder

**What I did:**

- Deleted the temporary local lab folder.

**Command used:**

```powershell
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-1b
```

**Result:**

- Local lab folder was removed.

**Notes:**

- If PowerShell gives an error, make sure the terminal is not currently inside the folder.
- Also close any files from that folder that may still be open.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| AWS Budget | Deleted using AWS CLI | Yes |
| SNS Topic | Deleted using AWS CLI | Yes |
| SNS Email Subscription | Removed when SNS topic was deleted | Yes |
| Local lab folder | Deleted from desktop | Yes |

## What I Learned

- AWS Budgets helps monitor cloud spending.
- SNS can send notifications to email subscribers.
- SNS email subscriptions must be confirmed before they can receive messages.
- JSON files can define AWS resources when using the CLI.
- Some successful AWS CLI commands may return no output.
- Budget alerts are useful for preventing unexpected cloud charges.
- Cleanup should always be completed and verified.
- Redacting account IDs, ARNs, and email addresses is important before uploading documentation.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/email-subscription-confirmed.png` | Email subscription confirmed |
| `screenshots/budget-json-created.png` | `budget.json` file created |
| `screenshots/notifications-json-created.png` | `notifications.json` file created |
| `screenshots/budget-created-cli.png` | Budget verified from CLI |
| `screenshots/budget-created-console.png` | Budget visible in AWS Console |
| `screenshots/terminal-problem-solved` | Terminal exiting folder |