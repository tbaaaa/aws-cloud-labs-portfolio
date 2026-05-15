# Lab 04: Launch EC2 Instance

## Lab Summary

In this lab, I launched a virtual server in AWS using Amazon EC2. I created an IAM role and instance profile so the EC2 instance could connect to AWS Systems Manager Session Manager, launched an Amazon Linux 2023 `t2.micro` instance, connected to it without SSH keys, explored the server, viewed instance details, and cleaned up the resources.

This lab is important because EC2 is one of the core AWS compute services. It introduces cloud servers, AMIs, instance types, IAM roles for EC2, instance profiles, and browser-based server access through Session Manager.

## Source Lab

- Repository: AICloudFusion
- Original lab: Lab 2A — Launch and Connect to a Virtual Server (EC2)
- My purpose: Document my own process, commands, validation steps, issues, cleanup, and lessons learned.

## Objectives

- Set the active AWS CLI profile
- Verify AWS CLI authentication
- Create a local project folder
- Create an IAM trust policy for EC2
- Create an IAM role for the EC2 instance
- Attach the Session Manager policy to the role
- Create an instance profile
- Add the IAM role to the instance profile
- Retrieve the latest Amazon Linux 2023 AMI ID
- Launch a `t2.micro` EC2 instance
- Wait for the instance to enter the running state
- Confirm the instance is online in Systems Manager
- Connect to the instance using Session Manager
- Explore the instance operating system, memory, disk, user, and private IP address
- View instance details from the CLI
- Terminate the instance
- Delete the IAM role and instance profile
- Verify cleanup

## Services / Tools Used

| Service / Tool | Purpose |
|---|---|
| Amazon EC2 | Launches and runs the virtual server |
| Amazon Machine Image AMI | Provides the operating system image for the instance |
| Amazon Linux 2023 | Linux operating system used for the EC2 instance |
| IAM Role | Gives the EC2 instance permission to use AWS services |
| IAM Instance Profile | Attaches the IAM role to the EC2 instance |
| AWS Systems Manager Session Manager | Allows browser-based or CLI-based connection to the instance without SSH |
| AWS CLI | Creates, configures, verifies, and deletes AWS resources |
| PowerShell | Terminal used to run AWS CLI commands |
| AWS Console | Used to visually verify the EC2 instance and connection options |

## Prerequisites

- Completed Lab 01: AWS Account & CLI Setup
- AWS CLI installed and configured
- AWS SSO profile working
- AWS CLI authentication verified
- PowerShell terminal
- Access to the AWS Management Console
- Basic understanding of Linux terminal commands

## Cost Notice

Estimated cost: `$0.00`

Notes:

- The lab uses a `t2.micro` EC2 instance, which is free-tier eligible.
- AWS Systems Manager Session Manager is free for this lab use case.
- IAM roles and instance profiles do not create charges.
- Cleanup is required because a running EC2 instance can use free-tier hours and may create charges if left running.

## Key Concepts

| Concept | Meaning |
|---|---|
| Amazon EC2 | AWS service used to create and run virtual servers in the cloud. |
| EC2 instance | A virtual server running in AWS. |
| Instance type | Defines the size of the server, including CPU and memory. |
| `t2.micro` | A small free-tier eligible EC2 instance type with 1 vCPU and 1 GiB memory. |
| AMI | Amazon Machine Image, the operating system template used to launch an instance. |
| Amazon Linux 2023 | AWS-managed Linux distribution used for the EC2 instance. |
| IAM role | An AWS identity that grants permissions to a service or resource. |
| Trust policy | Defines which AWS service can assume a role. |
| Instance profile | A container that allows an IAM role to be attached to an EC2 instance. |
| Session Manager | Systems Manager feature used to connect to instances without SSH keys or open inbound ports. |
| SSM Agent | Software on the instance that communicates with Systems Manager. |

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

- Created a local folder for the lab files.
- Changed into that folder before creating the IAM trust policy file.

**Command used:**

```powershell
mkdir ~\Desktop\workshop-lab-2a
cd ~\Desktop\workshop-lab-2a
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-2a
```

**Result:**

- PowerShell showed a path ending in `workshop-lab-2a`.

**Notes:**

- This folder stores the local `ec2-trust-policy.json` file for the lab.

---

### Step 4: Create EC2 Trust Policy File

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

- A trust policy controls who or what can assume a role.
- In this lab, the trusted service is EC2.
- Make sure the file is saved as `.json`, not `.json.txt`.

---

### Step 5: Create IAM Role for EC2

**What I did:**

- Created an IAM role named `workshop-ec2-ssm-role`.
- Used the trust policy file from the previous step.

**Command used:**

```bash
aws iam create-role --role-name workshop-ec2-ssm-role --assume-role-policy-document file://ec2-trust-policy.json
```

**Result:**

- IAM role was created successfully.

**Notes:**

- This role allows EC2 instances to assume the role.
- The role does not have useful permissions until a policy is attached.

---

### Step 6: Attach Session Manager Policy to IAM Role

**What I did:**

- Attached the AWS managed policy `AmazonSSMManagedInstanceCore` to the IAM role.

**Command used:**

```bash
aws iam attach-role-policy --role-name workshop-ec2-ssm-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**Result:**

- Session Manager permissions were attached to the role.

**Notes:**

- This policy allows the EC2 instance to communicate with AWS Systems Manager.
- Without this policy, Session Manager connection may fail.

---

### Step 7: Create Instance Profile

**What I did:**

- Created an instance profile named `workshop-ec2-ssm-profile`.

**Command used:**

```bash
aws iam create-instance-profile --instance-profile-name workshop-ec2-ssm-profile
```

**Result:**

- Instance profile was created successfully.

**Notes:**

- An instance profile is required to attach an IAM role to an EC2 instance.

---

### Step 8: Add Role to Instance Profile

**What I did:**

- Added the IAM role to the instance profile.

**Command used:**

```bash
aws iam add-role-to-instance-profile --instance-profile-name workshop-ec2-ssm-profile --role-name workshop-ec2-ssm-role
```

**Result:**

- IAM role was added to the instance profile.

**Notes:**

- IAM changes may take a short time to propagate.
- If the next steps fail due to role/profile timing, wait briefly and try again.

---

### Step 9: Get the Latest Amazon Linux 2023 AMI ID

**What I did:**

- Retrieved the latest Amazon Linux 2023 AMI ID from AWS Systems Manager Parameter Store.

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
- This is why the lab retrieves the latest ID instead of hardcoding one.

---

### Step 10: Launch EC2 Instance

**What I did:**

- Launched a `t2.micro` EC2 instance using the Amazon Linux 2023 AMI.
- Attached the `workshop-ec2-ssm-profile` instance profile.
- Added the name tag `workshop-session2`.

**Command used:**

```powershell
aws ec2 run-instances --image-id <AMI_ID> --instance-type t2.micro --iam-instance-profile Name=workshop-ec2-ssm-profile --region us-east-1 --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=workshop-session2}]" --query "Instances[0].InstanceId" --output text
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
- Redact the instance ID before uploading screenshots if desired.
- The instance may initially show as `pending` before changing to `running`.

---

### Step 11: Verify Instance in AWS Console

**What I did:**

- Opened the AWS Console.
- Went to EC2.
- Opened the Instances page.
- Confirmed the instance named `workshop-session2` appeared.

**Expected result:**

- Instance state should become `Running`.

**Result:**

- Instance appeared in the EC2 console.

**Notes:**

- It may take a minute for the instance to move from `Pending` to `Running`.

---

### Step 12: Wait for Instance to Run

**What I did:**

- Used the AWS CLI to wait until the instance entered the running state.

**Command used:**

```bash
aws ec2 wait instance-running --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

**Result:**

- Command completed after the instance reached the `running` state.

**Notes:**

- This command may appear to hang while it waits.
- No output usually means the wait command completed successfully.

---

### Step 13: Verify Session Manager Registration

**What I did:**

- Checked whether the EC2 instance was online in Systems Manager.

**Command used:**

```bash
aws ssm describe-instance-information --region us-east-1 --filters "Key=InstanceIds,Values=<YOUR_INSTANCE_ID>" --query "InstanceInformationList[0].PingStatus" --output text
```

**Expected result:**

```text
Online
```

**Result:**

- Instance appeared as online in Systems Manager.

**Notes:**

- If the result is `None` or blank, wait and try again.
- The SSM Agent needs time to register.
- If it never becomes online, check the IAM role and instance profile.

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

- The console method is easier for beginners.
- The CLI method may require the Session Manager plugin.
- Session Manager avoids SSH keys, passwords, and open inbound SSH ports.

---

### Step 15: Check Operating System

**What I did:**

- Ran a Linux command on the EC2 instance to check the operating system.

**Command used inside EC2 session:**

```bash
cat /etc/os-release
```

**Expected result:**

```text
Amazon Linux 2023
```

**Result:**

- The instance showed Amazon Linux 2023.

---

### Step 16: Check Memory

**What I did:**

- Checked the memory available on the EC2 instance.

**Command used inside EC2 session:**

```bash
free -h
```

**Expected result:**

- Around 1 GiB of memory.

**Result:**

- Memory information displayed successfully.

**Notes:**

- This matches the small size of the `t2.micro` instance type.

---

### Step 17: Check Disk Space

**What I did:**

- Checked the disk space on the EC2 instance.

**Command used inside EC2 session:**

```bash
df -h
```

**Expected result:**

- Root filesystem information displayed.

**Result:**

- Disk usage information displayed successfully.

---

### Step 18: Check Current User

**What I did:**

- Checked which Linux user I was connected as.

**Command used inside EC2 session:**

```bash
whoami
```

**Expected result:**

```text
ssm-user
```

**Result:**

- Session Manager connected as `ssm-user`.

**Notes:**

- `ssm-user` is the default user created for Session Manager sessions.

---

### Step 19: Check Private IP Address

**What I did:**

- Checked the instance private IP address.

**Command used inside EC2 session:**

```bash
hostname -I
```

**Expected result:**

```text
<PRIVATE_IP_ADDRESS>
```

**Result:**

- Private IP address was displayed.

**Notes:**

- Do not upload sensitive IP address information if you do not want it public.

---

### Step 15: Explore Linux File System Permissions

**What I did:**

- Used Session Manager to explore the EC2 instance file system.
- Checked my current directory.
- Navigated from `/usr/bin` to `/usr`.
- Listed the contents of `/usr`.
- Tried to create a test directory inside `/usr`.

**Commands used inside EC2 session:**

```bash
pwd
cd ..
pwd
ls -l
mkdir testDir
```

**Result**
mkdir: cannot create directory 'testDir': Permission denied

**Screenshot**
screenshots/permission-denied-ec2-instance-cli.png
---

### Step 21: Exit the Session

**What I did:**

- Disconnected from the EC2 instance.

**Command used inside EC2 session:**

```bash
exit
```

**Result:**

- Session Manager connection closed.

---

### Step 22: View Instance Details from CLI

**What I did:**

- Viewed details about the EC2 instance from my local terminal.

**Command used:**

```bash
aws ec2 describe-instances --instance-ids <YOUR_INSTANCE_ID> --region us-east-1 --query "Reservations[0].Instances[0].{State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone,PublicIP:PublicIpAddress,LaunchTime:LaunchTime}" --output table
```

**Expected result:**

- State: `running`
- Type: `t2.micro`
- Availability Zone: example `us-east-1a`
- Public IP: may show a public IP depending on subnet settings
- Launch time: timestamp of instance creation

**Result:**

- Instance details displayed successfully.

## Validation / Checkpoints

| Checkpoint | Result |
|---|---|
| AWS CLI profile set | Passed |
| `aws sts get-caller-identity` worked | Passed |
| Local project folder created | Passed |
| `ec2-trust-policy.json` created | Passed |
| IAM role created | Passed |
| `AmazonSSMManagedInstanceCore` policy attached | Passed |
| Instance profile created | Passed |
| Role added to instance profile | Passed |
| Latest Amazon Linux 2023 AMI ID retrieved | Passed |
| EC2 instance launched | Passed |
| Instance visible in AWS Console | Passed |
| Instance reached running state | Passed |
| SSM PingStatus showed `Online` | Passed |
| Session Manager connection worked | Passed |
| OS checked successfully | Passed |
| Memory checked successfully | Passed |
| Disk checked successfully | Passed |
| Current user checked successfully | Passed |
| Private IP checked successfully | Passed |
| Instance details viewed from CLI | Passed |

## Issues Encountered

| Issue | Cause | Fix |
|---|---|---|
| `mkdir testDir` returned `Permission denied` inside `/usr` | `/usr` is a protected system directory owned by `root`, and the current Session Manager user did not have write permission there | Use the home directory for normal files, such as `cd ~ && mkdir testDir`, or use `sudo` only when elevated permissions are required |

## Troubleshooting Notes

| Issue | Possible Cause | Fix |
|---|---|---|
| `InvalidParameterValue` when launching instance | AMI ID may be incorrect or expired | Rerun the AMI lookup command and use the latest AMI ID |
| `UnauthorizedOperation` | CLI profile does not have required EC2/IAM permissions | Confirm the correct AWS profile is active |
| Instance remains `Pending` | Normal startup delay | Wait and refresh the EC2 console |
| SSM status is blank or `None` | SSM Agent has not registered yet | Wait and rerun the SSM status command |
| Session Manager connection unavailable | IAM role, instance profile, or SSM registration issue | Confirm the role has `AmazonSSMManagedInstanceCore` and is attached through the instance profile |
| CLI Session Manager connection fails | Session Manager plugin may not be installed | Use the AWS Console Session Manager tab instead |

## Cleanup

### Step 1: Terminate EC2 Instance

**What I did:**

- Terminated the EC2 instance created for the lab.

**Command used:**

```bash
aws ec2 terminate-instances --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

**Expected result:**

- Instance state changes to `shutting-down`.

**Result:**

- Instance termination started successfully.

---

### Step 2: Wait for Instance Termination

**What I did:**

- Waited until the EC2 instance was fully terminated.

**Command used:**

```bash
aws ec2 wait instance-terminated --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

**Result:**

- Wait command completed after the instance reached the terminated state.

---

### Step 3: Detach IAM Policy from Role

**What I did:**

- Detached the Session Manager managed policy from the IAM role.

**Command used:**

```bash
aws iam detach-role-policy --role-name workshop-ec2-ssm-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

**Result:**

- Policy was detached from the IAM role.

---

### Step 4: Remove Role from Instance Profile

**What I did:**

- Removed the IAM role from the instance profile.

**Command used:**

```bash
aws iam remove-role-from-instance-profile --instance-profile-name workshop-ec2-ssm-profile --role-name workshop-ec2-ssm-role
```

**Result:**

- Role was removed from the instance profile.

---

### Step 5: Delete Instance Profile

**What I did:**

- Deleted the instance profile.

**Command used:**

```bash
aws iam delete-instance-profile --instance-profile-name workshop-ec2-ssm-profile
```

**Result:**

- Instance profile was deleted.

---

### Step 6: Delete IAM Role

**What I did:**

- Deleted the IAM role created for the lab.

**Command used:**

```bash
aws iam delete-role --role-name workshop-ec2-ssm-role
```

**Result:**

- IAM role was deleted.

---

### Step 7: Delete Local Lab Folder

**What I did:**

- Deleted the local project folder from my desktop.

**Command used:**

```powershell
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-2a
```

**Result:**

- Local lab folder was removed.

---

### Step 8: Verify Cleanup in AWS Console

**What I did:**

- Opened the EC2 console.
- Confirmed the instance was terminated.
- Opened IAM roles.
- Confirmed `workshop-ec2-ssm-role` was gone.

**Result:**

- Cleanup was verified.

## Cleanup Verification

| Resource | Cleanup Action | Verified |
|---|---|---|
| EC2 instance | Terminated | Yes |
| IAM managed policy attachment | Detached | Yes |
| IAM role | Deleted | Yes |
| Instance profile | Deleted | Yes |
| Local lab folder | Deleted | Yes |

## What I Learned

- EC2 allows me to launch virtual servers in AWS.
- An AMI provides the operating system image for an instance.
- Instance types define the compute size of an EC2 server.
- IAM roles are safer than storing access keys on servers.
- Instance profiles are used to attach IAM roles to EC2 instances.
- Session Manager allows remote access without SSH keys or open inbound ports.
- AWS CLI can be used to launch, inspect, and terminate EC2 instances.
- Cleanup is important because running EC2 instances can create charges.

## Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/trust-policy-file-created.png` | `ec2-trust-policy.json` created locally |
| `screenshots/iam-role-created.png` | IAM role created for EC2 Session Manager access |
| `screenshots/instance-profile-created.png` | Instance profile created and role attached |
| `screenshots/ami-id-retrieved.png` | Latest Amazon Linux 2023 AMI ID retrieved |
| `screenshots/ec2-instance-launched.png` | EC2 instance launched successfully |
| `screenshots/ec2-instance-running-console.png` | Instance shown as running in AWS Console |
| `screenshots/ssm-online.png` | Instance registered as online in Systems Manager |
| `screenshots/session-manager-connected.png` | Session Manager connection established |
| `screenshots/os-release-output.png` | Amazon Linux 2023 verified |
| `screenshots/memory-disk-user-checks.png` | Memory, disk, user, and private IP checked |
| `screenshots/instance-details-cli.png` | Instance details displayed from CLI |
| `screenshots/permission-denied-ec2-instance-cli.png` | Permission denied when creating a new directory |
| `screenshots/instance-terminated.png` | EC2 instance terminated |
| `screenshots/cleanup-verified.png` | IAM role/profile and EC2 cleanup verified |