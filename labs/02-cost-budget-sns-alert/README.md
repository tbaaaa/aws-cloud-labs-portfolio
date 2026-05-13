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
- Set a monthly budget amount
- Created an alert threshold
- Configured an email notification
- Confirmed the email subscription if required
- Verified the budget was created successfully

## Why This Matters

Cost monitoring is an important part of Cloud Operations. Even small cloud labs can create unexpected charges if resources are left running. Budget alerts help detect spending early and support better cloud cost management.

Useful screenshots to include:

- Budget created successfully
- Budget alert threshold
- Email subscription confirmation, with email address hidden
- Budget dashboard summary

## Issues Encountered

Example:

```text
Issue: SNS email notification stayed in PendingConfirmation.
Fix: Opened my email inbox and confirmed the SNS subscription.