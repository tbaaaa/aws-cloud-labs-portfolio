# Lab 19: Cost Optimization & Sustainability

## Lab Summary

In this lab, I applied the Cost Optimization and Sustainability pillars of the AWS Well-Architected Framework.

I created a Lambda workload function using the default `x86_64` architecture, measured its performance, and used a second Lambda function to calculate projected monthly infrastructure costs. The calculator published custom CloudWatch metrics, which were displayed on a CloudWatch dashboard.

I then changed the workload function to AWS Graviton `arm64`, measured the updated performance, and compared the projected compute cost. Finally, I applied a 7-day CloudWatch Logs retention policy to stop logs from growing indefinitely and verified the lower projected storage cost on the dashboard.

## Source Lab

* Repository: AICloudFusion
* Original lab: Lab 6C — Architecting for Cost Optimization & Sustainability
* Session: 6 — AWS Well-Architected Framework
* Track: Solutions Architecture
* Target certification: AWS Solutions Architect – Associate

## Objectives

* Set the active AWS CLI profile
* Verify AWS CLI authentication
* Create a local project folder
* Create a Lambda IAM role for cost calculation and CloudWatch access
* Deploy a CPU-intensive Lambda workload using `x86_64`
* Measure baseline Lambda performance
* Deploy a Lambda cost calculator
* Publish projected monthly cost metrics to CloudWatch
* Create a CloudWatch cost projection dashboard
* Record the baseline projected cost
* Switch the workload Lambda from `x86_64` to Graviton `arm64`
* Measure the updated performance
* Recalculate projected compute cost
* Review CloudWatch Logs retention settings
* Set a 7-day retention policy
* Recalculate projected log storage cost
* Verify the dashboard shows a downward cost trend
* Clean up all resources

## Services / Tools Used

| Service / Tool                 | Purpose                                                                             |
| ------------------------------ | ----------------------------------------------------------------------------------- |
| AWS Lambda                     | Runs the workload function and cost calculator                                      |
| AWS Graviton                   | Provides the `arm64` architecture for lower-cost, energy-efficient Lambda execution |
| AWS IAM                        | Provides Lambda execution permissions                                               |
| Amazon CloudWatch              | Stores metrics, Lambda logs, and the custom dashboard                               |
| CloudWatch Custom Metrics      | Stores projected compute, logs, and total monthly cost values                       |
| CloudWatch Dashboard           | Visualizes projected costs and Lambda performance                                   |
| CloudWatch Logs                | Stores Lambda logs and applies retention settings                                   |
| AWS CLI                        | Creates, configures, tests, and cleans up AWS resources                             |
| PowerShell                     | Runs commands and creates deployment packages                                       |
| Python                         | Lambda runtime language                                                             |
| AWS Well-Architected Framework | Provides Cost Optimization and Sustainability design guidance                       |

## Prerequisites

* Completed Lab 01: AWS Account & CLI Setup
* AWS CLI installed and configured
* Working AWS IAM Identity Center / SSO profile
* Access to the AWS Management Console
* Text editor such as VS Code or Notepad
* Basic understanding of Lambda, IAM, CloudWatch, and CloudWatch Logs

## Cost Notice

Estimated cost: `$0.00`

| Service         | Cost Consideration                                                                 |
| --------------- | ---------------------------------------------------------------------------------- |
| AWS Lambda      | Expected to remain within the free tier for this small lab workload                |
| CloudWatch      | Dashboard, metrics, and small log usage should remain within free-tier limits      |
| AWS IAM         | Free                                                                               |
| CloudWatch Logs | Small volume, but retention is configured to prevent unnecessary long-term storage |

## Key Concepts

| Concept              | Meaning                                                                                                     |
| -------------------- | ----------------------------------------------------------------------------------------------------------- |
| Cost Optimization    | Delivering business value at the lowest practical cost by removing waste and right-sizing resources.        |
| Sustainability       | Reducing the environmental impact of cloud workloads through efficient compute and controlled data storage. |
| Graviton             | AWS-designed ARM-based processors used through Lambda `arm64` architecture.                                 |
| `x86_64`             | Default Lambda architecture based on Intel or AMD processors.                                               |
| `arm64`              | Lambda architecture used by AWS Graviton processors.                                                        |
| Cost Projection      | An estimated monthly cost based on workload configuration and assumptions.                                  |
| Custom Metric        | A metric published by an application rather than automatically created by AWS.                              |
| CloudWatch Dashboard | A visual display of metrics, graphs, and operational data.                                                  |
| Log Retention        | The number of days that CloudWatch Logs are stored before automatic deletion.                               |
| Unbounded Storage    | Data storage that grows indefinitely because no retention limit exists.                                     |

## Security Notes

| Topic                 | Explanation                                                                                                              |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Least privilege       | The lab uses managed policies for simplicity, but production deployments should use custom least-privilege IAM policies. |
| No sensitive data     | The dashboard contains configuration and projected cost data, not secrets or personal information.                       |
| Log retention balance | Retention periods should be chosen based on operational and compliance requirements.                                     |
| Production approvals  | Architecture changes such as moving to `arm64` should be tested in a non-production environment before rollout.          |
| Native dependencies   | Some compiled libraries may require rebuilding or testing before switching from `x86_64` to `arm64`.                     |

## Architecture Overview

```text
Workload Lambda
  x86_64 architecture
       |
       v
Performance measurement
       |
       v
Cost Calculator Lambda
       |
       v
Custom CloudWatch metrics
       |
       v
CloudWatch Cost Dashboard

Optimization changes:
- x86_64 -> arm64 Graviton
- Logs never expire -> 7-day retention

Result:
- Lower projected compute cost
- Lower projected log storage cost
- Reduced storage growth
```

## Lab Steps

### Step 1: Set AWS CLI Profile

**What I did:**

* Set the AWS CLI profile for the current PowerShell session.
* Verified my active AWS identity.
* Recorded my account ID for Lambda role ARNs.

**Commands used:**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
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

**Notes:**

* If the SSO session has expired:

```powershell
aws sso login --profile <YOUR_PROFILE_NAME>
```

---

### Step 2: Create Local Project Folder

**What I did:**

* Created a local working folder for Lab 6C.

**Commands used:**

```powershell
mkdir ~\Desktop\workshop-lab-6c
cd ~\Desktop\workshop-lab-6c
pwd
```

**Expected result:**

```text
C:\Users\<YOUR_USERNAME>\Desktop\workshop-lab-6c
```

---

### Step 3: Create Lambda IAM Role

#### Step 3A: Create Trust Policy

Created `trust-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Step 3B: Create Role

```powershell
aws iam create-role `
  --role-name workshop-waf-cost-role `
  --assume-role-policy-document file://trust-policy.json
```

#### Step 3C: Attach Required Policies

```powershell
aws iam attach-role-policy `
  --role-name workshop-waf-cost-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy `
  --role-name workshop-waf-cost-role `
  --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess

aws iam attach-role-policy `
  --role-name workshop-waf-cost-role `
  --policy-arn arn:aws:iam::aws:policy/AWSLambda_ReadOnlyAccess
```

**Result:**

* Lambda role was created.
* The role could write logs, publish custom CloudWatch metrics, read CloudWatch Logs configuration, and read Lambda workload configuration.

**Notes:**

* Wait about 10–30 seconds before creating Lambda functions because IAM role propagation can take time.

---

### Step 4: Create Workload Lambda Function

**What I did:**

* Created a CPU-intensive prime-number workload.
* The function reports its memory allocation, execution time, and number of primes found.

Created `workload_function.py`:

```python
import json
import time
import math

def lambda_handler(event, context):
    start = time.time()

    n = event.get("workload_size", 50000)
    primes = []

    for num in range(2, n):
        is_prime = True

        for i in range(2, int(math.sqrt(num)) + 1):
            if num % i == 0:
                is_prime = False
                break

        if is_prime:
            primes.append(num)

    duration_ms = round((time.time() - start) * 1000, 2)
    memory_mb = context.memory_limit_in_mb

    return {
        "statusCode": 200,
        "body": json.dumps({
            "memory_mb": int(memory_mb),
            "duration_ms": duration_ms,
            "primes_found": len(primes)
        })
    }
```

### Step 5: Package and Deploy Workload Lambda

#### Step 5A: Create Deployment Package

```powershell
Compress-Archive -Path workload_function.py -DestinationPath workload.zip -Force
```

#### Step 5B: Deploy With `x86_64`

```powershell
aws lambda create-function `
  --function-name workshop-waf-cost-workload `
  --runtime python3.12 `
  --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-cost-role" `
  --handler workload_function.lambda_handler `
  --zip-file fileb://workload.zip `
  --memory-size 256 `
  --timeout 30 `
  --architectures x86_64 `
  --region us-east-1
```

**Expected result:**

* Lambda function created successfully.
* Function configuration shows:

```text
x86_64
```

---

### Step 6: Measure Baseline Workload Performance

#### Step 6A: Create Workload Payload

Created `workload-payload.json`:

```json
{
  "workload_size": 50000
}
```

#### Step 6B: Invoke Workload Lambda

```powershell
aws lambda invoke `
  --function-name workshop-waf-cost-workload `
  --payload file://workload-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  response.json

Write-Output "=== WORKLOAD RESULT ==="
(Get-Content response.json | ConvertFrom-Json).body
```

**Expected result:**

```json
{
  "memory_mb": 256,
  "duration_ms": 363.4,
  "primes_found": 5133
}
```

**Result:**

* I invoked the function two or three times.
* I recorded a typical `duration_ms` result instead of using the highest or lowest value.

**My typical x86_64 duration:**

```text
<RECORD_TYPICAL_DURATION_MS>
```

---

### Step 7: Create Cost Calculator Lambda

**What I did:**

* Created a Lambda function that reads the workload Lambda configuration and CloudWatch Logs retention settings.
* Calculated projected monthly compute and log storage costs.
* Published the calculated values as custom CloudWatch metrics.

Created `cost_calculator.py`:

```python
import boto3

def lambda_handler(event, context):
    region = "us-east-1"

    lambda_client = boto3.client("lambda", region_name=region)
    logs_client = boto3.client("logs", region_name=region)
    cloudwatch_client = boto3.client("cloudwatch", region_name=region)

    workload_fn_name = "workshop-waf-cost-workload"
    log_group_name = "/aws/lambda/workshop-waf-cost-workload"

    assumed_monthly_invocations = 1000000
    avg_duration_ms = event.get("avg_duration_ms", 400)

    function_config = lambda_client.get_function_configuration(
        FunctionName=workload_fn_name
    )

    memory_mb = function_config["MemorySize"]
    architecture = function_config.get("Architectures", ["x86_64"])[0]

    if architecture == "arm64":
        price_per_gb_ms = 0.0000000133
    else:
        price_per_gb_ms = 0.0000000167

    price_per_request = 0.0000002

    memory_gb = memory_mb / 1024
    cost_per_invocation = (
        memory_gb * avg_duration_ms * price_per_gb_ms
    ) + price_per_request

    monthly_compute_cost = cost_per_invocation * assumed_monthly_invocations

    try:
        log_groups = logs_client.describe_log_groups(
            logGroupNamePrefix=log_group_name
        )

        log_group = log_groups["logGroups"][0]
        retention_days = log_group.get("retentionInDays", 0)
        stored_bytes = log_group.get("storedBytes", 0)

    except Exception:
        retention_days = 0
        stored_bytes = 0

    logs_price_per_gb = 0.03

    if retention_days == 0:
        projected_logs_gb = (stored_bytes / (1024 ** 3)) + (0.5 * 12)
        monthly_logs_cost = projected_logs_gb * logs_price_per_gb
        logs_status = "NEVER EXPIRE - grows forever"
    else:
        daily_ingest_gb = 0.5 / 30
        projected_logs_gb = daily_ingest_gb * retention_days
        monthly_logs_cost = projected_logs_gb * logs_price_per_gb
        logs_status = f"{retention_days}-day retention - bounded"

    total_monthly_cost = monthly_compute_cost + monthly_logs_cost

    cloudwatch_client.put_metric_data(
        Namespace="Workshop/CostProjection",
        MetricData=[
            {
                "MetricName": "ProjectedComputeCostUSD",
                "Value": round(monthly_compute_cost, 4),
                "Unit": "None"
            },
            {
                "MetricName": "ProjectedLogsCostUSD",
                "Value": round(monthly_logs_cost, 4),
                "Unit": "None"
            },
            {
                "MetricName": "ProjectedTotalCostUSD",
                "Value": round(total_monthly_cost, 4),
                "Unit": "None"
            }
        ]
    )

    report = (
        f"COST PROJECTION REPORT\n"
        f"======================\n"
        f"COMPUTE:\n"
        f"  Function: {workload_fn_name}\n"
        f"  Architecture: {architecture}\n"
        f"  Memory: {memory_mb} MB\n"
        f"  Avg duration: {avg_duration_ms} ms\n"
        f"  Monthly invocations: {assumed_monthly_invocations:,}\n"
        f"  Projected compute cost: ${monthly_compute_cost:.4f}/month\n"
        f"\n"
        f"LOGS:\n"
        f"  Log group: {log_group_name}\n"
        f"  Retention: {logs_status}\n"
        f"  Projected logs cost: ${monthly_logs_cost:.4f}/month\n"
        f"\n"
        f"TOTAL PROJECTED COST: ${total_monthly_cost:.4f}/month\n"
        f"======================\n"
        f"Metrics published to CloudWatch dashboard."
    )

    return {
        "statusCode": 200,
        "body": report
    }
```

### Step 8: Package and Deploy Cost Calculator

```powershell
Compress-Archive -Path cost_calculator.py -DestinationPath calculator.zip -Force
```

```powershell
aws lambda create-function `
  --function-name workshop-waf-cost-calculator `
  --runtime python3.12 `
  --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-waf-cost-role" `
  --handler cost_calculator.lambda_handler `
  --zip-file fileb://calculator.zip `
  --memory-size 256 `
  --timeout 30 `
  --region us-east-1
```

**Result:**

* Cost calculator Lambda function deployed successfully.

---

### Step 9: Create CloudWatch Cost Dashboard

Created `dashboard.json`:

```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Projected Monthly Cost (USD)",
        "metrics": [
          ["Workshop/CostProjection", "ProjectedTotalCostUSD", {"label": "Total Cost"}],
          ["Workshop/CostProjection", "ProjectedComputeCostUSD", {"label": "Compute Cost"}],
          ["Workshop/CostProjection", "ProjectedLogsCostUSD", {"label": "Logs Cost"}]
        ],
        "region": "us-east-1",
        "period": 60,
        "stat": "Average",
        "view": "timeSeries"
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Lambda Performance (Duration ms)",
        "metrics": [
          ["AWS/Lambda", "Duration", "FunctionName", "workshop-waf-cost-workload", {"label": "Avg Duration", "stat": "Average"}]
        ],
        "region": "us-east-1",
        "period": 60,
        "view": "timeSeries"
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 6,
      "width": 12,
      "height": 6,
      "properties": {
        "title": "Lambda Invocations",
        "metrics": [
          ["AWS/Lambda", "Invocations", "FunctionName", "workshop-waf-cost-workload", {"label": "Invocations", "stat": "Sum"}]
        ],
        "region": "us-east-1",
        "period": 60,
        "view": "timeSeries"
      }
    }
  ]
}
```

Create the dashboard:

```powershell
aws cloudwatch put-dashboard `
  --dashboard-name Workshop-WAF-CostOptimization `
  --dashboard-body file://dashboard.json `
  --region us-east-1
```

**Expected result:**

```json
{
  "DashboardValidationMessages": []
}
```

**Console checkpoint:**

* Open CloudWatch.
* Select **Dashboards**.
* Open `Workshop-WAF-CostOptimization`.
* The dashboard may initially show no data because custom metrics have not yet been published.

## Part 1: Cost Optimization and Sustainability — Graviton

### Step 10: Capture Baseline Cost Projection

Created `calculator-payload.json`:

```json
{
  "avg_duration_ms": <YOUR_TYPICAL_X86_DURATION>
}
```

Invoke the calculator:

```powershell
aws lambda invoke `
  --function-name workshop-waf-cost-calculator `
  --payload file://calculator-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  calc-response.json

Write-Output "=== COST PROJECTION (BEFORE) ==="
(Get-Content calc-response.json | ConvertFrom-Json).body
```

**Expected result:**

```text
COST PROJECTION REPORT

COMPUTE:
  Architecture: x86_64
  Projected compute cost: $<VALUE>/month

LOGS:
  Retention: NEVER EXPIRE - grows forever
  Projected logs cost: $<VALUE>/month

TOTAL PROJECTED COST: $<VALUE>/month
```

**My baseline projected total cost:**

```text
$<RECORD_BASELINE_TOTAL>/month
```

---

### Step 11: Switch Workload Lambda to Graviton `arm64`

**What I did:**

* Redeployed the same workload package using the `arm64` architecture.
* This changed the Lambda processor architecture from x86_64 to AWS Graviton.

```powershell
aws lambda update-function-code `
  --function-name workshop-waf-cost-workload `
  --zip-file fileb://workload.zip `
  --architectures arm64 `
  --region us-east-1 `
  --query "Architectures" `
  --output text
```

**Expected result:**

```text
arm64
```

**Notes:**

* Python code does not need recompilation for this lab.
* Compiled native dependencies may need separate ARM-compatible builds in real workloads.

---

### Step 12: Measure Graviton Performance

```powershell
aws lambda invoke `
  --function-name workshop-waf-cost-workload `
  --payload file://workload-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  response-arm64.json

Write-Output "=== WORKLOAD RESULT (ARM64 GRAVITON) ==="
(Get-Content response-arm64.json | ConvertFrom-Json).body
```

**Expected result:**

```json
{
  "memory_mb": 256,
  "duration_ms": <RESULT>,
  "primes_found": 5133
}
```

**My typical arm64 duration:**

```text
<RECORD_TYPICAL_ARM64_DURATION_MS>
```

---

### Step 13: Recalculate Cost After Graviton

Update `calculator-payload.json` with the typical arm64 duration:

```json
{
  "avg_duration_ms": <YOUR_TYPICAL_ARM64_DURATION>
}
```

Run the calculator again:

```powershell
aws lambda invoke `
  --function-name workshop-waf-cost-calculator `
  --payload file://calculator-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  calc-response2.json

Write-Output "=== COST PROJECTION (AFTER GRAVITON) ==="
(Get-Content calc-response2.json | ConvertFrom-Json).body
```

**Expected result:**

* Architecture displays as `arm64`.
* Projected compute cost is lower than the x86_64 baseline.
* The dashboard receives a second custom-metric data point.

**My projected total after Graviton:**

```text
$<RECORD_GRAVITON_TOTAL>/month
```

## Part 2: Cost Optimization and Sustainability — Log Retention

### Step 14: Review Default Log Retention

```powershell
aws logs describe-log-groups `
  --log-group-name-prefix "/aws/lambda/workshop-waf-cost-workload" `
  --query "logGroups[0].{LogGroup:logGroupName,Retention:retentionInDays}" `
  --output table `
  --region us-east-1
```

**Expected result before setting retention:**

```text
Retention: None
```

**Meaning:**

* `None` means logs never expire.
* Without a retention policy, logs can grow indefinitely.

---

### Step 15: Apply 7-Day Log Retention

```powershell
aws logs put-retention-policy `
  --log-group-name /aws/lambda/workshop-waf-cost-workload `
  --retention-in-days 7 `
  --region us-east-1
```

Verify:

```powershell
aws logs describe-log-groups `
  --log-group-name-prefix "/aws/lambda/workshop-waf-cost-workload" `
  --query "logGroups[0].retentionInDays" `
  --output text `
  --region us-east-1
```

**Expected result:**

```text
7
```

**Result:**

* CloudWatch Logs will automatically delete events older than seven days.

---

### Step 16: Recalculate Final Cost Projection

```powershell
aws lambda invoke `
  --function-name workshop-waf-cost-calculator `
  --payload file://calculator-payload.json `
  --cli-binary-format raw-in-base64-out `
  --region us-east-1 `
  calc-response3.json

Write-Output "=== COST PROJECTION (AFTER RETENTION FIX) ==="
(Get-Content calc-response3.json | ConvertFrom-Json).body
```

**Expected result:**

```text
LOGS:
  Retention: 7-day retention - bounded
  Projected logs cost: $<LOWER_VALUE>/month

TOTAL PROJECTED COST: $<LOWER_VALUE>/month
```

**My final projected total:**

```text
$<RECORD_FINAL_TOTAL>/month
```

### Cost Comparison

| Stage                 | Architecture | Log Retention | Projected Compute Cost | Projected Logs Cost | Projected Total Cost |
| --------------------- | ------------ | ------------- | ---------------------: | ------------------: | -------------------: |
| Before optimization   | x86_64       | Never expire  |             `$<VALUE>` |          `$<VALUE>` |           `$<VALUE>` |
| After Graviton        | arm64        | Never expire  |             `$<VALUE>` |          `$<VALUE>` |           `$<VALUE>` |
| Final optimized state | arm64        | 7 days        |             `$<VALUE>` |          `$<VALUE>` |           `$<VALUE>` |

### Step 17: Console Verification

**What I did:**

* Opened the `Workshop-WAF-CostOptimization` CloudWatch dashboard.
* Set the time range to **Last 1 hour**.
* Verified the projected cost graph displayed three data points.
* Confirmed the total cost showed a downward trend after:

  1. Switching to Graviton.
  2. Setting a 7-day log retention policy.

## Validation / Checkpoints

| Checkpoint                                          | Result |
| --------------------------------------------------- | ------ |
| AWS CLI profile set                                 | Passed |
| AWS identity verified                               | Passed |
| Local project folder created                        | Passed |
| Lambda trust policy created                         | Passed |
| Cost role created                                   | Passed |
| Lambda, CloudWatch, and read-only policies attached | Passed |
| Workload Lambda code created                        | Passed |
| Workload Lambda deployed with x86_64                | Passed |
| Baseline x86_64 performance recorded                | Passed |
| Cost calculator code created                        | Passed |
| Cost calculator Lambda deployed                     | Passed |
| CloudWatch dashboard created                        | Passed |
| Baseline cost metrics published                     | Passed |
| Baseline cost displayed on dashboard                | Passed |
| Workload Lambda switched to arm64                   | Passed |
| Arm64 performance recorded                          | Passed |
| Updated cost metrics published                      | Passed |
| Lower compute cost verified                         | Passed |
| Default logs retention reviewed                     | Passed |
| Seven-day logs retention applied                    | Passed |
| Final projected cost calculated                     | Passed |
| Final dashboard cost trend verified                 | Passed |

## Issues Encountered

| Issue                     | Cause | Fix |
| ------------------------- | ----- | --- |
| None currently documented | N/A   | N/A |

## Troubleshooting Notes

| Issue                                                | What It Means                                                           | How to Fix                                                          |
| ---------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------- |
| Lambda creation fails because role cannot be assumed | IAM role propagation is still in progress                               | Wait 10–30 seconds and retry the command                            |
| Lambda payload error                                 | The AWS CLI did not read JSON payload correctly                         | Use `--payload file://filename.json` instead of inline JSON         |
| Cost calculator fails at `GetFunctionConfiguration`  | The Lambda role lacks read permission for Lambda configuration          | Reattach `AWSLambda_ReadOnlyAccess`                                 |
| Dashboard shows no data                              | Custom metrics may not have propagated yet                              | Wait one or two minutes and refresh the dashboard                   |
| Dashboard cost trend is unclear                      | Cost data points were created too close together                        | Set dashboard range to Last 15 minutes or Last 1 hour and zoom in   |
| `arm64` update fails                                 | AWS CLI may be outdated                                                 | Check `aws --version` and ensure AWS CLI v2 is installed            |
| Calculator response file cannot be read              | PowerShell is in the wrong folder or invocation did not create the file | Run `pwd`, confirm the current directory, then rerun the invocation |
| Retention still displays `None`                      | Wrong log group was queried or policy was not applied                   | Confirm the exact log group name and rerun `put-retention-policy`   |
| SSO token expired                                    | IAM Identity Center session expired                                     | Run `aws sso login --profile <YOUR_PROFILE_NAME>`                   |

## Cleanup

### Step 1: Delete CloudWatch Dashboard

```powershell
aws cloudwatch delete-dashboards `
  --dashboard-names Workshop-WAF-CostOptimization `
  --region us-east-1
```

### Step 2: Delete Lambda Functions

```powershell
aws lambda delete-function `
  --function-name workshop-waf-cost-calculator `
  --region us-east-1

aws lambda delete-function `
  --function-name workshop-waf-cost-workload `
  --region us-east-1
```

### Step 3: Delete Lambda Log Groups

```powershell
aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-waf-cost-workload `
  --region us-east-1

aws logs delete-log-group `
  --log-group-name /aws/lambda/workshop-waf-cost-calculator `
  --region us-east-1
```

### Step 4: Detach Policies and Delete IAM Role

```powershell
aws iam detach-role-policy `
  --role-name workshop-waf-cost-role `
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam detach-role-policy `
  --role-name workshop-waf-cost-role `
  --policy-arn arn:aws:iam::aws:policy/CloudWatchFullAccess

aws iam detach-role-policy `
  --role-name workshop-waf-cost-role `
  --policy-arn arn:aws:iam::aws:policy/AWSLambda_ReadOnlyAccess

aws iam delete-role `
  --role-name workshop-waf-cost-role
```

### Step 5: Delete Local Lab Folder

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-6c
```

## Cleanup Verification

| Resource                  | Cleanup Action | Verified |
| ------------------------- | -------------- | -------- |
| CloudWatch dashboard      | Deleted        | Yes      |
| Cost calculator Lambda    | Deleted        | Yes      |
| Workload Lambda           | Deleted        | Yes      |
| Workload Lambda log group | Deleted        | Yes      |
| Cost calculator log group | Deleted        | Yes      |
| Lambda managed policies   | Detached       | Yes      |
| Lambda IAM role           | Deleted        | Yes      |
| Local project folder      | Deleted        | Yes      |

## What I Learned

* Cost optimization starts with visibility into current configuration and projected spend.
* A CloudWatch dashboard can make optimization changes visible and measurable.
* Lambda `arm64` uses AWS Graviton processors and can reduce cost compared with `x86_64`.
* Graviton can also improve sustainability by using less energy per computation.
* Production architecture changes should be tested because native x86 dependencies may not work on arm64 without updates.
* CloudWatch Logs should not remain at unlimited retention by default.
* A log retention policy prevents indefinite storage growth and makes storage cost more predictable.
* Cost Optimization and Sustainability often align because reducing unnecessary compute and storage reduces both spending and energy usage.
* Custom CloudWatch metrics can be used to track projected infrastructure cost over time.
* AWS Well-Architected decisions should be measured rather than assumed.

## Screenshots

| Screenshot                                    | Description                                   |
| --------------------------------------------- | --------------------------------------------- |
| `screenshots/project-folder-created.png`      | Local Lab 6C folder created                   |
| `screenshots/cost-role-created.png`           | IAM role for cost functions created           |
| `screenshots/cost-role-policies-attached.png` | Required managed policies attached            |
| `screenshots/workload-lambda-x86-created.png` | Workload Lambda deployed using x86_64         |
| `screenshots/workload-x86-performance.png`    | Typical x86_64 performance result             |
| `screenshots/cost-calculator-created.png`     | Cost calculator Lambda deployed               |
| `screenshots/dashboard-created.png`           | CloudWatch dashboard created                  |
| `screenshots/baseline-cost-projection.png`    | Baseline projected cost output                |
| `screenshots/baseline-dashboard-cost.png`     | First dashboard cost data point               |
| `screenshots/workload-arm64-updated.png`      | Workload Lambda changed to arm64              |
| `screenshots/workload-arm64-performance.png`  | Typical arm64 performance result              |
| `screenshots/cost-after-graviton.png`         | Lower compute projection after Graviton       |
| `screenshots/dashboard-after-graviton.png`    | Second dashboard cost data point              |
| `screenshots/log-retention-none.png`          | Log group initially showed no retention limit |
| `screenshots/log-retention-seven-days.png`    | Seven-day retention policy applied            |
| `screenshots/final-cost-projection.png`       | Final projected cost after log retention      |
| `screenshots/final-dashboard-cost-trend.png`  | Dashboard showing downward cost trend         |
| `screenshots/cleanup-verified.png`            | Lab cleanup completed                         |