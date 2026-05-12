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
    - The modern, sercure way to manage who an access your account. Companies never allow you to login as root. Instead you create and log in as individual users with specific permissions.
- Created a user for daily AWS access
    - A user account "ayden" was created and provisioned with a password and MFA set up.
    - A new group was created "administrators" and "ayden" was added to it.
    - A permission set has been created to assign the level of access a group or user gets. 
    - The persmission set is then assigned to the group "administrators"
    - Logged in as "ayden" who is in the "Administrators" group with "AdminAccess" permission set
    - It is said to always use this login method instead of the signing in on the root user.
    - The root user is usually used for account-level tasks like managing the payment method or closing the account
- Installed AWS CLI
- Configured AWS CLI using SSO
    - Started Configuration process
    - > aws configure sso
- Tested CLI access
    - Verified Identity using the command below
    - > aws sts get-caller-identity
    - {
    "UserId": "AROAEXAMPLEID:your-username",
    "Account": "123456789012",
    "Arn": "arn:aws:sts::123456789012:assumed-role/AWSReservedSSO_AdministratorAccess_.../your-username"
    }

## Commands Used

```bash
aws --version
aws configure sso
aws sts get-caller-identity --profile PROFILE-NAME
```

## Troubleshooting 
N/A