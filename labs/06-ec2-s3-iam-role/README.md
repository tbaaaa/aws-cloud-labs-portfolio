# Lab 05: EC2 S3 IAM Role

## Lab Summary

In this lab, I launched an EC2 instance that could read from and write to Amazon S3 without storing access keys on the server. I created an S3 bucket, created an IAM role for EC2, attached permissions for Session Manager and S3 access, launched an EC2 instance with the role attached, and tested file uploads/downloads between EC2 and S3.

This lab is important because IAM roles are the secure and production-standard way to give AWS resources access to other AWS services.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 2B — Access S3 from EC2 Using an IAM Role
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Create an S3 bucket
- Retrieve the latest Amazon Linux 2023 AMI ID
- Create an IAM trust policy for EC2
- Create an IAM role for EC2
- Attach Session Manager permissions to the role
- Attach S3 access permissions to the role
- Create an instance profile
- Add the IAM role to the instance profile
- Launch an EC2 instance with the instance profile attached
- Connect to the EC2 instance using Session Manager
- Upload a file from EC2 to S3
- Upload a file from the local machine to S3
- Download a file from S3 to EC2
- Verify files in the AWS Console
- Clean up all lab resources

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| Amazon EC2 | Runs the virtual server used to test S3 access |
| Amazon S3 | Stores files uploaded from EC2 and the local machine |
| IAM Role | Grants AWS permissions to the EC2 instance |
| IAM Instance Profile | Attaches the IAM role to the EC2 instance |
| AWS Systems Manager Session Manager | Connects to the EC2 instance without SSH keys |
| AWS CLI | Creates, configures, verifies, and deletes AWS resources |
| Amazon Linux 2023 | Operating system used on the EC2 instance |
| PowerShell | Local terminal used to run AWS CLI commands |
| Linux Shell | Terminal environment inside the EC2 instance |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- Completed or understood Lab 04: Launch EC2 Instance
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- PowerShell terminal
- Access to the AWS Management Console
- Basic understanding of EC2, S3, IAM roles, and Session Manager

## Cost Notice

Estimated cost: low, but cleanup is required.

Notes:

- The source lab uses a `t3.nano` EC2 instance.
- The source lab estimates the cost at about `$3.77` if the EC2 instance runs for a full month and S3 storage stays under 1 GB.
- Amazon S3 storage costs depend on how much data is stored and how many requests are made.
- IAM roles and instance profiles do not create charges.
- Session Manager is free for this lab use case.
- Terminate the EC2 instance and delete the S3 bucket after the lab to avoid unnecessary charges.

## Key Concepts

| Concept | Meaning |
|---|---|
| IAM Role for EC2 | An identity that an EC2 instance can assume to access AWS services securely. |
| Temporary credentials | Short-lived credentials automatically provided through an IAM role. |
| Instance Profile | A container that allows an IAM role to be attached to an EC2 instance. |
| Amazon S3 | AWS object storage service used to store files in buckets. |
| S3 Bucket | A container for objects stored in S3. |
| Object | A file stored in S3, such as `hello-from-ec2.txt`. |
| Session Manager | Allows secure access to EC2 instances without SSH keys or open inbound ports. |
| Access keys | Long-term credentials that should not be stored on EC2 instances. |
| Least privilege | Security principle of granting only the permissions needed. |

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

- Set my AWS CLI profile for the current PowerShell session.

**Command used:**

```powershell
$env:AWS_PROFILE="<PROFILE_NAME>"
```

**Result:**

- AWS CLI used my selected profile for commands in the current terminal session.

**Notes:**

- This avoids typing `--profile <PROFILE_NAME>` on every command.
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

- Account ID and ARN details should be redacted before uploading screenshots.
- If the SSO session expires, run:

```bash
aws sso login --profile <PROFILE_NAME>
```

---

### Step 3: Create Local Project Folder

**What I did:**

- Created a local folder for this lab.
- Changed into that folder before creating files.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-2b
cd ~\Desktop\workshop-lab-2b
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-2b
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-2b`.

**Notes:**

- This folder stores files created for the lab, such as `ec2-trust-policy.json` and `local-file.txt`.

---

### Step 4: Create an S3 Bucket

**What I did:**

- Created a globally unique S3 bucket in `us-east-1`.

**Command used:**

```bash
aws s3 mb s3://<YOUR_UNIQUE_BUCKET_NAME> --region us-east-1
```

**Expected result:**

```text
make_bucket: <YOUR_UNIQUE_BUCKET_NAME>
```

**Result:**

- S3 bucket was created successfully.

**Notes:**

- S3 bucket names must be globally unique.
- Use lowercase letters, numbers, and hyphens.
- If `BucketAlreadyExists` appears, choose another bucket name.

---

### Step 5: Retrieve Latest Amazon Linux 2023 AMI ID

**What I did:**

- Retrieved the latest Amazon Linux 2023 AMI ID using AWS Systems Manager Parameter Store.

**Command used:**

```bash
aws ssm get-parameters-by-path --path "/aws/service/ami-amazon-linux-latest" --region us-east-1 --query "Parameters[?Name=='/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64'].Value" --output text
```

**Expected result:**

```text
ami-xxxxxxxxxxxxxxxxx
```

**Result:**

- Latest Amazon Linux 2023 AMI ID was returned.

**My AMI ID:**

```text
REDACTED_OR_RECORDED_HERE
```

**Notes:**

- AMI IDs can change over time as AWS releases updated images.
- This is why the lab retrieves the latest AMI ID instead of hardcoding one.

---

### Step 6: Create EC2 Trust Policy File

**What I did:**

- Created a file named `ec2-trust-policy.json`.
- Added a trust policy that allows EC2 to assume the IAM role.

**File path:**

```text
ec2-trust-policy.json
```

**File content:**

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Result:**

- The trust policy file was created locally.

**Notes:**

- A trust policy defines which AWS service is allowed to assume the role.
- In this lab, the trusted service is EC2.
- Make sure the file is saved as `.json`, not `.json.txt`.

---

### Step 7: Create IAM Role for EC2

**What I did:**

- Created an IAM role named `workshop-ec2-s3-role`.
- Used the trust policy file created in the previous step.

**Command used:**

```bash
aws iam create-role --role-name workshop-ec2-s3-role --assume-role-policy-document file://ec2-trust-policy.json
```

**Result:**

- IAM role was created successfully.

**Notes:**

- This role allows EC2 instances to assume the role.
- The role does not provide S3 or Session Manager permissions until policies are attached.

---

### Step 8: Attach Session Manager Policy to IAM Role

**What I did:**

- Attached the AWS managed policy `AmazonSSMManagedInstanceCore` to the IAM role.

**Command used:**

```bash
aws iam attach-role-policy --role-name workshop-ec2-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**Result:**

- Session Manager permissions were attached to the role.

**Notes:**

- This policy allows the EC2 instance to communicate with Systems Manager.
- Without this policy, Session Manager connection may fail.

---

### Step 9: Attach S3 Access Policy to IAM Role

**What I did:**

- Attached the AWS managed policy `AmazonS3FullAccess` to the IAM role.

**Command used:**

```bash
aws iam attach-role-policy --role-name workshop-ec2-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

**Result:**

- S3 permissions were attached to the role.

**Notes:**

- This gives the EC2 instance permission to access S3.
- The source lab uses `AmazonS3FullAccess` for simplicity.
- In a real production environment, a more restrictive custom policy should be used to limit access to only the required bucket and actions.

---

### Step 10: Create Instance Profile

**What I did:**

- Created an instance profile named `workshop-ec2-s3-profile`.

**Command used:**

```bash
aws iam create-instance-profile --instance-profile-name workshop-ec2-s3-profile
```

**Result:**

- Instance profile was created successfully.

**Notes:**

- EC2 instances use instance profiles to receive IAM role permissions.

---

### Step 11: Add Role to Instance Profile

**What I did:**

- Added the IAM role to the instance profile.

**Command used:**

```bash
aws iam add-role-to-instance-profile --instance-profile-name workshop-ec2-s3-profile --role-name workshop-ec2-s3-role
```

**Result:**

- IAM role was added to the instance profile.

**Notes:**

- IAM changes may take a few seconds to propagate.
- Wait briefly before launching the EC2 instance if needed.

---

### Step 12: Launch EC2 Instance

**What I did:**

- Launched an EC2 instance using the Amazon Linux 2023 AMI.
- Used the `t3.nano` instance type.
- Attached the `workshop-ec2-s3-profile` instance profile.
- Added the name tag `workshop-s3-access`.

**Command used:**

```powershell
aws ec2 run-instances --image-id <AMI_ID> --instance-type t3.nano --iam-instance-profile Name=workshop-ec2-s3-profile --region us-east-1 --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=workshop-s3-access}]" --query "Instances[0].InstanceId" --output text
```

**Expected result:**

```text
i-xxxxxxxxxxxxxxxxx
```

**Result:**

- EC2 instance was launched successfully.

**My Instance ID:**

```text
REDACTED_OR_RECORDED_HERE
```

**Notes:**

- The instance ID is needed for later commands.
- The instance may start in a `pending` state before changing to `running`.

---

### Step 13: Wait for Instance to Run

**What I did:**

- Waited until the instance entered the running state.

**Command used:**

```bash
aws ec2 wait instance-running --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

**Result:**

- Command completed after the instance reached the `running` state.

**Notes:**

- This command may appear to pause while AWS waits for the instance state to change.
- No output usually means the wait command completed successfully.

---

### Step 14: Connect Using Session Manager

**What I did:**

- Connected to the EC2 instance using AWS Systems Manager Session Manager.

**Console method:**

1. Open EC2 in the AWS Console.
2. Select the instance.
3. Click **Connect**.
4. Select the **Session Manager** tab.
5. Click **Connect**.

**Optional CLI method:**

```bash
aws ssm start-session --target <YOUR_INSTANCE_ID> --region us-east-1
```

**Result:**

- A terminal session opened on the EC2 instance.

**Notes:**

- Session Manager avoids SSH keys, passwords, and open inbound SSH ports.
- The CLI method may require the Session Manager plugin.

---

### Step 15: Create a Test File on EC2

**What I did:**

- Created a text file on the EC2 instance in the `/tmp` directory.

**Command used inside EC2 session:**

```bash
echo "Hello from EC2! This file was uploaded from a virtual server in the cloud." > /tmp/hello-from-ec2.txt
```

**Result:**

- File was created on the EC2 instance.

**Notes:**

- `/tmp` is a common temporary directory that normal users can write to.
- This avoids permission issues that may happen in system directories such as `/usr`.

---

### Step 16: Upload File from EC2 to S3

**What I did:**

- Uploaded the EC2-created file to the S3 bucket.

**Command used inside EC2 session:**

```bash
aws s3 cp /tmp/hello-from-ec2.txt s3://<YOUR_UNIQUE_BUCKET_NAME>/hello-from-ec2.txt
```

**Expected result:**

```text
upload: /tmp/hello-from-ec2.txt to s3://<YOUR_UNIQUE_BUCKET_NAME>/hello-from-ec2.txt
```

**Result:**

- File uploaded from EC2 to S3 successfully.

**Notes:**

- I did not configure access keys on the EC2 instance.
- The upload worked because the attached IAM role provided temporary credentials automatically.
- This is the secure way to give AWS resources access to other AWS services.

---

### Step 17: List S3 Bucket Contents from EC2

**What I did:**

- Listed the contents of the S3 bucket from inside the EC2 instance.

**Command used inside EC2 session:**

```bash
aws s3 ls s3://<YOUR_UNIQUE_BUCKET_NAME>/
```

**Expected result:**

```text
hello-from-ec2.txt
```

**Result:**

- The uploaded file appeared in the S3 bucket listing.

---

### Step 18: Disconnect from Session Manager

**What I did:**

- Exited the Session Manager terminal so I could upload a second file from my local machine.

**Command used inside EC2 session:**

```bash
exit
```

**Result:**

- Session Manager connection closed.

---

### Step 19: Create Local File

**What I did:**

- Created a file on my local computer.

**PowerShell command used locally:**

```powershell
"This file was uploaded from my local computer" | Out-File -Encoding utf8 local-file.txt
```

**Alternative Bash command:**

```bash
echo "This file was uploaded from my local computer" > local-file.txt
```

**Result:**

- `local-file.txt` was created in my local lab folder.

---

### Step 20: Upload Local File to S3

**What I did:**

- Uploaded the local file to the S3 bucket.

**Command used locally:**

```bash
aws s3 cp local-file.txt s3://<YOUR_UNIQUE_BUCKET_NAME>/local-file.txt
```

**Expected result:**

```text
upload: ./local-file.txt to s3://<YOUR_UNIQUE_BUCKET_NAME>/local-file.txt
```

**Result:**

- Local file uploaded to S3 successfully.

---

### Step 21: Reconnect to EC2

**What I did:**

- Reconnected to the EC2 instance using Session Manager.

**Console method:**

1. Open EC2 in the AWS Console.
2. Select the instance.
3. Click **Connect**.
4. Select the **Session Manager** tab.
5. Click **Connect**.

**Result:**

- Session Manager connection opened again.

---

### Step 22: Download File from S3 to EC2

**What I did:**

- Downloaded `local-file.txt` from S3 to the EC2 instance.

**Command used inside EC2 session:**

```bash
aws s3 cp s3://<YOUR_UNIQUE_BUCKET_NAME>/local-file.txt /tmp/downloaded-file.txt
```

**Expected result:**

```text
download: s3://<YOUR_UNIQUE_BUCKET_NAME>/local-file.txt to ../../tmp/downloaded-file.txt
```

**Result:**

- File downloaded from S3 to EC2 successfully.

---

### Step 23: Read Downloaded File on EC2

**What I did:**

- Displayed the contents of the downloaded file.

**Command used inside EC2 session:**

```bash
cat /tmp/downloaded-file.txt
```

**Expected result:**

```text
This file was uploaded from my local computer
```

**Result:**

- EC2 successfully read the file that was uploaded to S3 from my local machine.

**Notes:**

- This demonstrates S3 as shared storage between a local machine and a cloud server.

---

### Step 24: Verify Files in AWS Console

**What I did:**

- Opened the S3 console.
- Opened my lab bucket.
- Confirmed both files existed:
  - `hello-from-ec2.txt`
  - `local-file.txt`

**Result:**

- Both files were visible in the S3 bucket.

**Notes:**

- This confirmed that the EC2 instance and local machine both successfully interacted with S3.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| `aws sts get-caller-identity` worked | Passed |
| Local project folder created | Passed |
| S3 bucket created | Passed |
| Latest Amazon Linux 2023 AMI ID retrieved | Passed |
| `ec2-trust-policy.json` created | Passed |
| IAM role created | Passed |
| `AmazonSSMManagedInstanceCore` policy attached | Passed |
| `AmazonS3FullAccess` policy attached | Passed |
| Instance profile created | Passed |
| Role added to instance profile | Passed |
| EC2 instance launched | Passed |
| Instance reached running state | Passed |
| Session Manager connection worked | Passed |
| Test file created on EC2 | Passed |
| EC2 uploaded file to S3 | Passed |
| EC2 listed S3 bucket contents | Passed |
| Local file created | Passed |
| Local file uploaded to S3 | Passed |
| EC2 downloaded file from S3 | Passed |
| EC2 read downloaded file successfully | Passed |
| Files verified in S3 console | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| None currently documented | N/A | N/A |

## Troubleshooting Notes

| Issue | Possible Cause | Fix |
|---|---|---|
| `Unable to locate credentials` inside EC2 | IAM role is not attached, has not propagated, or instance profile is missing | Wait 1–2 minutes, then verify the instance profile is attached to the EC2 instance |
| `AccessDenied` when accessing S3 | The EC2 role does not have S3 permissions | Confirm `AmazonS3FullAccess` was attached to the IAM role |
| Session Manager does not connect | SSM Agent has not registered or role lacks Session Manager permissions | Wait 1–2 minutes and confirm `AmazonSSMManagedInstanceCore` is attached |
| `BucketAlreadyExists` | S3 bucket name is already taken globally | Choose a new unique bucket name |
| `NoSuchBucket` | Bucket name was typed incorrectly or bucket was deleted | Confirm the exact bucket name with `aws s3 ls` |
| `Permission denied` when creating a file on EC2 | Trying to write in a protected Linux directory | Use `/tmp` or the user’s home directory instead |

## Cleanup

### Step 1: Disconnect from Session Manager

**What I did:**

- Disconnected from the EC2 instance.

**Command used inside EC2 session:**

```bash
exit
```

**Result:**

- Session Manager connection closed.

---

### Step 2: Terminate EC2 Instance

**What I did:**

- Terminated the EC2 instance created for this lab.

**Command used locally:**

```bash
aws ec2 terminate-instances --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

**Expected result:**

- Instance state changes to `shutting-down`.

**Result:**

- Instance termination started successfully.

---

### Step 3: Wait for Instance Termination

**What I did:**

- Waited until the EC2 instance was fully terminated.

**Command used locally:**

```bash
aws ec2 wait instance-terminated --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

**Result:**

- Wait command completed after the instance reached the terminated state.

---

### Step 4: Empty the S3 Bucket

**What I did:**

- Removed all objects from the S3 bucket.

**Command used locally:**

```bash
aws s3 rm s3://<YOUR_UNIQUE_BUCKET_NAME> --recursive
```

**Expected result:**

```text
delete: s3://<YOUR_UNIQUE_BUCKET_NAME>/hello-from-ec2.txt
delete: s3://<YOUR_UNIQUE_BUCKET_NAME>/local-file.txt
```

**Result:**

- Bucket objects were removed.

---

### Step 5: Delete the S3 Bucket

**What I did:**

- Deleted the empty S3 bucket.

**Command used locally:**

```bash
aws s3 rb s3://<YOUR_UNIQUE_BUCKET_NAME>
```

**Expected result:**

```text
remove_bucket: <YOUR_UNIQUE_BUCKET_NAME>
```

**Result:**

- S3 bucket was deleted.

---

### Step 6: Detach Session Manager Policy from Role

**What I did:**

- Detached the Session Manager managed policy from the IAM role.

**Command used locally:**

```bash
aws iam detach-role-policy --role-name workshop-ec2-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**Result:**

- Policy was detached from the IAM role.

---

### Step 7: Detach S3 Policy from Role

**What I did:**

- Detached the S3 managed policy from the IAM role.

**Command used locally:**

```bash
aws iam detach-role-policy --role-name workshop-ec2-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

**Result:**

- Policy was detached from the IAM role.

---

### Step 8: Remove Role from Instance Profile

**What I did:**

- Removed the IAM role from the instance profile.

**Command used locally:**

```bash
aws iam remove-role-from-instance-profile --instance-profile-name workshop-ec2-s3-profile --role-name workshop-ec2-s3-role
```

**Result:**

- Role was removed from the instance profile.

---

### Step 9: Delete Instance Profile

**What I did:**

- Deleted the instance profile.

**Command used locally:**

```bash
aws iam delete-instance-profile --instance-profile-name workshop-ec2-s3-profile
```

**Result:**

- Instance profile was deleted.

---

### Step 10: Delete IAM Role

**What I did:**

- Deleted the IAM role.

**Command used locally:**

```bash
aws iam delete-role --role-name workshop-ec2-s3-role
```

**Result:**

- IAM role was deleted.

---

### Step 11: Delete Local Lab Folder

**What I did:**

- Deleted the local project folder from my desktop.

**Command used locally:**

```powershell
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-2b
```

**Result:**

- Local lab folder was removed.

---

### Step 12: Verify Cleanup

**What I did:**

- Verified cleanup in AWS Console:
  - EC2 instance showed as terminated
  - S3 bucket was gone
  - IAM role was gone

**Result:**

- Lab resources were cleaned up successfully.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| EC2 instance | Terminated | Yes |
| S3 objects | Deleted | Yes |
| S3 bucket | Deleted | Yes |
| `AmazonSSMManagedInstanceCore` policy attachment | Detached | Yes |
| `AmazonS3FullAccess` policy attachment | Detached | Yes |
| Instance profile | Deleted | Yes |
| IAM role | Deleted | Yes |
| Local lab folder | Deleted | Yes |

## Production Improvement: Least Privilege S3 Policy

The lab uses the AWS managed policy `AmazonS3FullAccess` to keep the exercise simple. This is regarded as acceptable for a beginner lab, but it gives broader S3 permissions than needed.

It is said that a more secure production approach would be to create a custom IAM policy that only allows access to the specific lab bucket.


## What I Learned

- IAM roles are the secure way to give EC2 instances access to AWS services.
- Access keys should not be stored on EC2 instances.
- Instance profiles allow IAM roles to be attached to EC2 instances.
- The EC2 instance can use temporary credentials from the attached IAM role.
- S3 can act as shared storage between a local computer and a cloud server.
- Session Manager allows EC2 access without SSH keys or open inbound ports.
- `AmazonS3FullAccess` is useful for a beginner lab, but production environments should use least privilege.
- Cleanup is important because EC2 and S3 resources can create charges if left running.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/s3-bucket-created.png` | S3 bucket created successfully |
| `screenshots/ami-id-retrieved.png` | Latest Amazon Linux 2023 AMI ID retrieved |
| `screenshots/trust-policy-file-created.png` | `ec2-trust-policy.json` created locally |
| `screenshots/iam-role-created.png` | IAM role created for EC2 S3 access |
| `screenshots/iam-policies-attached.png` | Session Manager and S3 policies attached |
| `screenshots/instance-profile-created.png` | Instance profile created and role attached |
| `screenshots/ec2-instance-launched.png` | EC2 instance launched successfully |
| `screenshots/ec2-instance-running-console.png` | Instance shown as running in AWS Console |
| `screenshots/session-manager-connected.png` | Session Manager connection established |
| `screenshots/ec2-file-created.png` | Test file created on EC2 |
| `screenshots/ec2-upload-to-s3.png` | File uploaded from EC2 to S3 |
| `screenshots/s3-list-from-ec2.png` | S3 bucket listed from EC2 |
| `screenshots/local-file-uploaded.png` | Local file uploaded to S3 |
| `screenshots/ec2-download-from-s3.png` | File downloaded from S3 to EC2 |
| `screenshots/downloaded-file-read.png` | Downloaded file content displayed on EC2 |
| `screenshots/s3-console-files-verified.png` | Both files verified in S3 console |
| `screenshots/cleanup-verified.png` | EC2, S3, and IAM cleanup verified |