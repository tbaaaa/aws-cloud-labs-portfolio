# Lab 02: Cost Budget & SNS Alert

## Objective

Create a basic AWS cost budget and configure an alert so I can monitor cloud spending and avoid unexpected charges.

## AWS Services / Tools Used

- AWS Budgets
- Amazon SNS
- Email notification
- AWS Billing and Cost Management

## What I Did

- Opened AWS Billing and Cost Management
- Created a cost budget
    - Created the budget which uses both the budget and notifications json files
- Set a monthly budget amount
    - Created a budget definition file "budget.json" and set the monthly budget amount to $1USD
- Created an alert threshold
    - Created a notification configuration file "notifications.json" and set threshold to 80%
- Configured an email notification
- Confirmed the email subscription if required
- Verified the budget was created successfully
    - Verified via CLI and AWS Budgets
- Cleanup
    - Deleted the Budget
    - Deleted the SNS Topic
    - Verified both has been deleted
    - Delete Local Files

## Why This Matters

Cost monitoring is an important part of Cloud Operations. Even small cloud labs can create unexpected charges if resources are left running. Budget alerts help detect spending early and support better cloud cost management.

## Commands Used
```text
aws sns create-topic --name workshop-budget-alert --region us-east-1
aws sns subscribe --topic-arn <YOUR_TOPIC_ARN> --protocol email --notification-endpoint <YOUR_EMAIL_ADDRESS> --region us-east-1
aws budgets create-budget --account-id <YOUR_ACCOUNT_ID> --budget file://budget.json --notifications-with-subscribers file://notifications.json
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --output table

CLEANUP:
aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name "workshop-monthly-budget"
aws sns delete-topic --topic-arn <YOUR_TOPIC_ARN> --region us-east-1
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --output json
aws sns list-topics --region us-east-1
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-1b
```
## Issues Encountered

```text
Issue: Received an error in Windows Powershell when attempting to delete the folder "workshop-lab-1b".
Fix: Ensure that any instances of the folder opened are closed or exited. This included Notepad, File Explorer and even the Windows Powershell terminal. Check `screenshots/terminal-problem-solved`