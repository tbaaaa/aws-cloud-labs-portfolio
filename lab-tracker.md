# Lab Tracker

| Date | Lab | Source Lab | Status | Notes |
|---|---|---|---|---|
| 2026-05-12 | AWS Account & CLI Setup | Lab 1A | Completed | Set up IAM Identity Center and AWS CLI access |
| 2026-05-13 | Cost Budget & SNS Alert | Lab 1B | Completed | Created and cleaned up AWS Budget and SNS topic |
| 2026-05-14 | S3 Static Website | Lab 1C | Completed | Hosting a static website using Amazon S3 |
| 2026-05-15 | Launch EC2 Instance | Lab 2A | Completed | Launching and connecting to an EC2 instance using Session Manager |
| 2026-05-18 | EC2 S3 IAM Role | Lab 2B | Completed | Using an IAM role to allow EC2 to upload and download files from S3 |
| 2026-05-18 | Serverless File Pipeline | Lab 2C | Completed | Building a serverless web app using S3, Lambda, presigned URLs, CORS, and S3 event triggers |
| 2026-05-20 | IAM Least Privilege | Lab 3A | Completed | Creating a read-only IAM user and testing least privilege access to one S3 bucket |
| 2026-05-20 | IAM Groups, Custom Policies & MFA | Lab 3B | Completed | Creating an IAM group, applying a custom policy with explicit Deny, testing inherited permissions, and enabling root MFA |
| 2026-05-26 | Roles, CloudTrail & Audit Logging | Lab 3C | Completed | Creating CloudTrail audit logging, developer/auditor roles, and testing separation of duties |
| 2026-06-01 | CloudFront HTTPS Static Website | Side Quest Lab 1D | Completed | Adding CloudFront HTTPS and OAC to secure an S3 static website |
| 2026-06-02 | GuardDuty Findings | Lab 4A | Completed | Enabling GuardDuty, generating sample findings, reviewing severity levels, and cleaning up the detector |
| 2026-06-03 | CloudTrail Investigation | Lab 4B | Completed | Creating CloudTrail audit logs, simulating suspicious IAM/S3 activity, and investigating deleted activity with lookup-events |
| 2026-06-04 | Automated Security Alerts | Lab 4C | Completed | Building an automated GuardDuty alert pipeline using EventBridge and SNS email notifications |
| 2026-06-08 | Credential Revocation | Lab 5A | Completed | Simulating a compromised AWS access key, deactivating it for containment, testing failed authentication, and deleting it permanently |
| 2026-06-11 | Investigate and Report | Lab 5B | Completed | Simulating S3 data exfiltration, investigating with CloudTrail and CloudWatch Logs Insights, revoking credentials, and writing an incident report |
| 2026-06-18 | Automated Remediation | Lab 5C | Completed | Building an automated incident response pipeline using CloudTrail, EventBridge, and Lambda to deactivate newly created IAM access keys |
| 2026-06-22 | S3 Security & Reliability | Lab 6A | Completed | Applying least-privilege S3 bucket access with IAM roles and a bucket policy, then using S3 versioning to recover deleted data |
| 2026-06-23 | Lambda Performance & Operations | Lab 6B | In Progress | Right-sizing Lambda memory for improved performance and creating CloudWatch/SNS alerting for function errors |