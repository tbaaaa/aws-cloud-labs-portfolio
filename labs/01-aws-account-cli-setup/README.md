# Lab 01: AWS Account & CLI Setup

## Objective

Set up secure access to AWS and configure the AWS CLI for hands-on cloud practice.

## AWS Services / Tools Used

- AWS Account
- IAM Identity Center
- AWS CLI

## What I Did

- Created or accessed my AWS account
- Enabled IAM Identity Center
- Created a user for daily AWS access
- Installed AWS CLI
- Configured AWS CLI using SSO
- Tested CLI access

## Commands Used

```bash
aws --version
aws configure sso
aws sts get-caller-identity --profile PROFILE-NAME
```

## Issues/Fixes

Issue:
Fix: 