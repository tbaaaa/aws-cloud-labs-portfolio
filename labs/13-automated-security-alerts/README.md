# Lab 13: Automated Security Alerts

## Lab Summary

In this lab, I built an automated security alerting pipeline using Amazon GuardDuty, Amazon EventBridge, and Amazon SNS.

The goal was to automatically send an email alert when GuardDuty produces a medium or high severity finding. GuardDuty acts as the detection layer, EventBridge acts as the routing/orchestration layer, and SNS acts as the notification layer.

This lab is important because it introduces the foundation of SOAR: Security Orchestration, Automation, and Response.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 4C — Build Automated Security Alerts with EventBridge
- Session: 4 — Threat Detection & AWS Security Services
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Check whether GuardDuty is already enabled
- Enable GuardDuty if needed
- Create an SNS topic for security alerts
- Subscribe an email address to the SNS topic
- Confirm the email subscription
- Create an SNS topic policy that allows EventBridge to publish messages
- Create an EventBridge event pattern for GuardDuty findings with severity `>= 4`
- Create an EventBridge rule
- Add the SNS topic as the EventBridge target
- Generate a GuardDuty sample finding
- Verify that an email alert is received
- Verify the pipeline in the AWS Console
- Clean up EventBridge, SNS, GuardDuty, and local files

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| Amazon GuardDuty | Detects suspicious or malicious activity |
| Amazon EventBridge | Routes matching GuardDuty finding events |
| Amazon SNS | Sends email notifications |
| SNS Topic Policy | Allows EventBridge to publish messages to the SNS topic |
| AWS CLI | Creates, configures, tests, and deletes lab resources |
| PowerShell | Terminal used to run AWS CLI commands |
| JSON | Used to define the SNS topic policy and EventBridge event pattern |
| Email Inbox | Receives the automated security alert |
| AWS Console | Used to visually verify GuardDuty, EventBridge, and SNS |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- Access to an email inbox
- Text editor such as VS Code or Notepad
- Basic understanding of GuardDuty findings, EventBridge rules, and SNS topics

## Cost Notice

Estimated cost: `$0.00`

Notes:

- GuardDuty has a 30-day free trial, then charges based on analyzed data.
- EventBridge is free for AWS service events in this lab.
- SNS email notifications are free within the monthly free tier.
- Cleanup is required. Delete the GuardDuty detector at the end of the lab to avoid charges after the free trial.

## Key Concepts

| Concept | Meaning |
|---|---|
| GuardDuty | AWS threat detection service that generates security findings. |
| EventBridge | Serverless event bus that routes events from AWS services to targets. |
| Event Pattern | JSON filter that decides which events should match a rule. |
| SNS | Simple Notification Service used to send notifications to subscribers. |
| SNS Topic | A notification channel that subscribers can receive messages from. |
| SNS Subscription | An endpoint, such as an email address, subscribed to a topic. |
| Topic Policy | A policy that controls who can publish messages to an SNS topic. |
| SOAR | Security Orchestration, Automation, and Response. |
| Event-driven alerting | Automatically responding when an AWS event happens. |
| Alert fatigue | Too many low-value alerts causing important alerts to be ignored. |

## Security Notes

| Topic | Explanation |
|---|---|
| Severity filter | The EventBridge rule only matches GuardDuty findings with severity `>= 4`, meaning medium and high findings. |
| Low severity ignored | Low severity findings are not sent to email to reduce alert fatigue. |
| SNS topic policy | EventBridge needs permission to publish to the SNS topic. |
| Email confirmation | SNS email subscriptions must be confirmed before notifications can be delivered. |
| Sample findings | GuardDuty sample findings are safe simulations and do not mean the account was attacked. |
| Deduplication | GuardDuty may treat repeated sample findings as updates instead of new findings, which may prevent another email from being sent. |
| Production improvement | A real SOC pipeline could use Lambda to format alerts, create tickets, notify Slack/Teams, or trigger remediation. |

## Architecture Overview

```text
GuardDuty Finding
      |
      v
EventBridge Rule
  severity >= 4
      |
      v
SNS Topic
      |
      v
Email Alert
```

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.
- Verified my current AWS identity.
- Recorded my AWS account ID.

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

- AWS CLI returned my active identity successfully.

**Notes:**

- The account ID is needed for the SNS topic policy.
- If the SSO token is expired, run:

```bash
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

- Created a local folder for this lab.
- Changed into the folder before creating JSON files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-4c
cd ~\Desktop\workshop-lab-4c
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-4c
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-4c`.

---

### Step 3: Check If GuardDuty Is Already Enabled

**What I did:**

- Checked whether GuardDuty already had a detector in `us-east-1`.

**Command used:**

```bash
aws guardduty list-detectors --region us-east-1
```

**Expected result if GuardDuty is not enabled:**

```json
{
  "DetectorIds": []
}
```

**Expected result if GuardDuty is already enabled:**

```json
{
  "DetectorIds": [
    "REDACTED_DETECTOR_ID"
  ]
}
```

**Result:**

- GuardDuty status was checked.

**My Detector ID:**

```text
REDACTED_OR_RECORDED_HERE
```

**Notes:**

- If the list is empty, continue to Step 4.
- If a detector ID already appears, record it and skip to Step 5.
- GuardDuty detectors are region-specific.

---

### Step 4: Enable GuardDuty

**What I did:**

- Enabled GuardDuty in `us-east-1`.

**Command used:**

```bash
aws guardduty create-detector --enable --region us-east-1
```

**Expected result:**

```json
{
  "DetectorId": "REDACTED_DETECTOR_ID"
}
```

**Result:**

- GuardDuty detector was created successfully.

**Notes:**

- Creating a detector enables GuardDuty in that region.
- Record the detector ID for later commands.

---

### Step 5: Create SNS Topic for Security Alerts

**What I did:**

- Created an SNS topic named `workshop-security-alerts`.

**Command used:**

```bash
aws sns create-topic --name workshop-security-alerts --region us-east-1
```

**Expected result:**

```json
{
  "TopicArn": "arn:aws:sns:us-east-1:REDACTED:workshop-security-alerts"
}
```

**Result:**

- SNS topic was created successfully.

**My Topic ARN:**

```text
REDACTED_OR_RECORDED_HERE
```

**Notes:**

- The topic ARN is needed for the email subscription, SNS policy, EventBridge target, and cleanup.

---

### Step 6: Subscribe Email to SNS Topic

**What I did:**

- Subscribed my email address to the SNS topic.

**Command used:**

```bash
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol email --notification-endpoint <YOUR_EMAIL> --region us-east-1
```

**Expected result:**

```json
{
  "SubscriptionArn": "pending confirmation"
}
```

**Result:**

- Email subscription was created and was initially pending confirmation.

**Notes:**

- Email addresses should be redacted in screenshots.
- The subscription must be confirmed before SNS can send alerts.

---

### Step 7: Confirm Email Subscription

**What I did:**

- Opened the email inbox used in Step 6.
- Found the AWS SNS subscription confirmation email.
- Clicked the confirmation link.
- Verified that the subscription status changed from pending to confirmed.

**Verification command:**

```bash
aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN> --region us-east-1 --query "Subscriptions[].{Endpoint:Endpoint,Protocol:Protocol,Status:SubscriptionArn}" --output table
```

**Expected result:**

- Email appears with a full subscription ARN.
- Status should no longer show `PendingConfirmation`.

**Result:**

- Email subscription was confirmed.

---

### Step 8: Create SNS Topic Policy

**What I did:**

- Created a file named `sns-policy.json`.
- Added a topic policy that allows EventBridge to publish messages to the SNS topic.
- Also allowed my account to manage the topic for cleanup.

**File created:**

```text
sns-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Id": "AllowEventBridgePublish",
    "Statement": [
        {
            "Sid": "AllowAccountToManage",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<YOUR_ACCOUNT_ID>:root"
            },
            "Action": [
                "SNS:Publish",
                "SNS:Subscribe",
                "SNS:Receive",
                "SNS:GetTopicAttributes",
                "SNS:SetTopicAttributes",
                "SNS:AddPermission",
                "SNS:RemovePermission",
                "SNS:ListSubscriptionsByTopic"
            ],
            "Resource": "<TOPIC_ARN>"
        },
        {
            "Sid": "AllowEventBridgeToPublish",
            "Effect": "Allow",
            "Principal": {
                "Service": "events.amazonaws.com"
            },
            "Action": "sns:Publish",
            "Resource": "<TOPIC_ARN>"
        }
    ]
}
```

**Result:**

- SNS topic policy file was created locally.

**Notes:**

- Replace `<YOUR_ACCOUNT_ID>` once.
- Replace `<TOPIC_ARN>` in both places.
- EventBridge cannot publish to the topic unless this permission exists.

---

### Step 9: Apply SNS Topic Policy

**What I did:**

- Applied the SNS topic policy to the SNS topic.

**Command used:**

```bash
aws sns set-topic-attributes --topic-arn <TOPIC_ARN> --attribute-name Policy --attribute-value file://sns-policy.json --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

- SNS topic policy was applied successfully.

---

### Step 10: Create EventBridge Event Pattern

**What I did:**

- Created a file named `eventbridge-rule.json`.
- Added an event pattern that matches GuardDuty findings with severity greater than or equal to `4`.

**File created:**

```text
eventbridge-rule.json
```

**File content:**

```json
{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
        "severity": [{"numeric": [">=", 4]}]
    }
}
```

**Result:**

- EventBridge event pattern file was created locally.

**Notes:**

- Low severity findings below `4` are ignored.
- Medium and high severity findings match the rule.
- This helps reduce alert fatigue.

---

### Step 11: Create EventBridge Rule

**What I did:**

- Created an EventBridge rule named `workshop-guardduty-alert`.

**Command used:**

```bash
aws events put-rule --name workshop-guardduty-alert --event-pattern file://eventbridge-rule.json --state ENABLED --region us-east-1
```

**Expected result:**

```json
{
  "RuleArn": "arn:aws:events:us-east-1:REDACTED:rule/workshop-guardduty-alert"
}
```

**Result:**

- EventBridge rule was created and enabled.

**Notes:**

- The rule listens for GuardDuty finding events that match the event pattern.

---

### Step 12: Add SNS Topic as EventBridge Target

**What I did:**

- Connected the EventBridge rule to the SNS topic.

**Command used:**

```bash
aws events put-targets --rule workshop-guardduty-alert --targets '[{\"Id\":\"sns-target\",\"Arn\":\"<TOPIC_ARN>\"}]' --region us-east-1
```

**Expected result:**

```json
{
  "FailedEntryCount": 0,
  "FailedEntries": []
}
```

**Result:**

- SNS topic was added as the EventBridge target successfully.

**Notes:**

- `FailedEntryCount: 0` means the target was added correctly.
- If this fails, check the SNS topic policy.

---

### Step 13: Generate GuardDuty Sample Finding

**What I did:**

- Generated a high-severity GuardDuty sample finding.

**Command used:**

```bash
aws guardduty create-sample-findings --detector-id <DETECTOR_ID> --finding-types "CryptoCurrency:EC2/BitcoinTool.B!DNS" --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

- GuardDuty sample finding was generated.

**Notes:**

- This sample finding has high severity.
- It should match the EventBridge event pattern.
- The alert may take 2–5 minutes to reach email.

---

### Step 14: Verify Email Alert

**What I did:**

- Checked my email inbox.
- Checked spam/junk folder if needed.
- Looked for an AWS Notifications email.
- Reviewed the JSON details in the alert.

**Expected result:**

- Email alert received.
- Alert includes GuardDuty finding details such as:
  - Finding type
  - Severity
  - Description
  - Affected resource

**Result:**

- Email alert was verified.

**Notes:**

- The email content is raw JSON.
- In production, a Lambda function could format the alert before sending it to SNS.
- GuardDuty may deduplicate repeated sample findings. If the same finding was generated recently, it may be treated as an update instead of a new event, and EventBridge may not send a new email.

---

### Step 15: Verify Pipeline in AWS Console

**What I did:**

- Verified all parts of the pipeline in the AWS Console.

**EventBridge checks:**

- Rule `workshop-guardduty-alert` exists
- Rule is enabled
- Event pattern matches GuardDuty findings
- SNS topic is listed as the target

**SNS checks:**

- Topic `workshop-security-alerts` exists
- Email subscription is confirmed
- Topic policy allows EventBridge to publish

**GuardDuty checks:**

- Detector exists
- Sample finding appears in Findings
- Finding severity is medium/high

**Result:**

- Automated alerting pipeline was verified.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| AWS identity verified | Passed |
| Local project folder created | Passed |
| GuardDuty detector checked | Passed |
| GuardDuty enabled or existing detector recorded | Passed |
| SNS topic created | Passed |
| Email subscribed to SNS topic | Passed |
| Email subscription confirmed | Passed |
| SNS topic policy created | Passed |
| SNS topic policy applied | Passed |
| EventBridge event pattern created | Passed |
| EventBridge rule created and enabled | Passed |
| SNS topic added as EventBridge target | Passed |
| GuardDuty sample finding generated | Passed |
| Email alert checked | Passed |
| EventBridge rule verified in console | Passed |
| SNS topic/subscription verified in console | Passed |
| GuardDuty finding verified in console | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | What It Means | How to Fix |
|---|---|---|
| Email subscription stays pending | The confirmation email has not been clicked | Check inbox/spam and click the SNS confirmation link |
| Email confirmation not received | Email address may be wrong or message may be in spam | Re-run the subscribe command with the correct email |
| `put-targets` returns `FailedEntryCount: 1` | EventBridge may not have permission to publish to SNS | Verify and reapply `sns-policy.json` |
| No email alert received | Subscription not confirmed, rule not enabled, SNS policy wrong, or GuardDuty sample finding deduplicated | Confirm subscription, verify rule/target, reapply SNS policy, and wait several minutes |
| Sample finding does not trigger another email | GuardDuty may treat repeated sample findings as updates within a deduplication window | Try a different sample finding type or wait longer |
| `InvalidParameterValue` when creating EventBridge rule | Event pattern JSON has a syntax error | Validate `eventbridge-rule.json` |
| `A detector already exists for the current account` | GuardDuty is already enabled | Use `list-detectors` and continue with the existing detector ID |
| `ResourceNotFoundException` for EventBridge rule | Rule name or region is wrong | Confirm rule name is `workshop-guardduty-alert` and region is `us-east-1` |
| SSO token expired | AWS CLI SSO session expired | Run `aws sso login --profile <YOUR_PROFILE_NAME>` |

## Cleanup

### Step 1: Remove EventBridge Target

**What I did:**

- Removed the SNS target from the EventBridge rule.

**Command used:**

```bash
aws events remove-targets --rule workshop-guardduty-alert --ids sns-target --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

- EventBridge target was removed.

---

### Step 2: Delete EventBridge Rule

**What I did:**

- Deleted the EventBridge rule.

**Command used:**

```bash
aws events delete-rule --name workshop-guardduty-alert --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

- EventBridge rule was deleted.

---

### Step 3: Delete SNS Subscription

**What I did:**

- Retrieved the SNS subscription ARN.
- Unsubscribed the email endpoint.

**Get subscription ARN:**

```bash
aws sns list-subscriptions-by-topic --topic-arn <TOPIC_ARN> --region us-east-1 --query "Subscriptions[].SubscriptionArn" --output text
```

**Unsubscribe command:**

```bash
aws sns unsubscribe --subscription-arn <SUBSCRIPTION_ARN> --region us-east-1
```

**Result:**

- SNS email subscription was removed.

---

### Step 4: Delete SNS Topic

**What I did:**

- Deleted the SNS topic.

**Command used:**

```bash
aws sns delete-topic --topic-arn <TOPIC_ARN> --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

- SNS topic was deleted.

---

### Step 5: Delete GuardDuty Detector

**What I did:**

- Deleted the GuardDuty detector.

**Command used:**

```bash
aws guardduty delete-detector --detector-id <DETECTOR_ID> --region us-east-1
```

**Expected result:**

```text
No output
```

**Result:**

- GuardDuty detector was deleted.

**Notes:**

- This disables GuardDuty in `us-east-1`.
- Deleting the detector helps prevent charges after the free trial.

---

### Step 6: Verify GuardDuty Is Disabled

**What I did:**

- Confirmed that no GuardDuty detector exists in `us-east-1`.

**Command used:**

```bash
aws guardduty list-detectors --region us-east-1
```

**Expected result:**

```json
{
  "DetectorIds": []
}
```

**Result:**

- GuardDuty was verified as disabled.

---

### Step 7: Delete Local Lab Folder

**What I did:**

- Deleted the local project folder.

**PowerShell command used:**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-4c
```

**Result:**

- Local lab folder was deleted.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| EventBridge target | Removed from rule | Yes |
| EventBridge rule | Deleted | Yes |
| SNS email subscription | Unsubscribed | Yes |
| SNS topic | Deleted | Yes |
| GuardDuty detector | Deleted | Yes |
| Local lab folder | Deleted | Yes |

## What I Learned

- GuardDuty findings can be routed automatically to EventBridge.
- EventBridge event patterns can filter findings by severity.
- SNS can be used to send security alerts by email.
- EventBridge needs permission to publish to an SNS topic.
- The SNS topic policy is required for the pipeline to work.
- Filtering for severity `>= 4` helps reduce alert fatigue by ignoring low-priority findings.
- This pipeline is a basic example of SOAR.
- In production, a Lambda function could format the alert, enrich it, create a ticket, or trigger remediation.
- GuardDuty sample findings are safe but may be deduplicated if repeated too soon.
- Cleanup is important because GuardDuty continues running until the detector is deleted.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/project-folder-created.png` | Local project folder created |
| `screenshots/guardduty-detector-check.png` | Checked whether GuardDuty already had a detector |
| `screenshots/guardduty-detector-created.png` | GuardDuty enabled or existing detector ID recorded |
| `screenshots/sns-topic-created.png` | SNS security alert topic created |
| `screenshots/email-subscription-pending.png` | SNS email subscription pending confirmation |
| `screenshots/email-subscription-confirmed-1.png` | SNS email subscription confirmed |
| `screenshots/email-subscription-confirmed-2.png` | SNS email subscription confirmed |
| `screenshots/sns-policy-created.png` | SNS topic policy file created |
| `screenshots/sns-policy-applied.png` | SNS topic policy applied |
| `screenshots/eventbridge-rule-file-created.png` | EventBridge event pattern file created |
| `screenshots/eventbridge-rule-created.png` | EventBridge rule created and enabled |
| `screenshots/sns-target-added.png` | SNS topic added as EventBridge target |
| `screenshots/sample-finding-generated.png` | GuardDuty sample finding generated |
| `screenshots/security-alert-email.png` | Automated GuardDuty alert received by email |
| `screenshots/eventbridge-rule-console.png` | EventBridge rule verified in AWS Console |
| `screenshots/sns-topic-console.png` | SNS topic and subscription verified in AWS Console |
| `screenshots/guardduty-finding-console.png` | GuardDuty sample finding visible in console |
| `screenshots/cleanup-verified.png` | Lab resources cleaned up |