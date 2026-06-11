# Incident Report

## Incident Summary

| Field | Value |

|-------|-------|

| **Incident ID** | IR-2026-001 |

| **Date Detected** | 2026-06-11 |

| **Severity** | High |

| **Status** | Resolved |

| **Reported By** | Ayden |

| **Compromised Identity** | attacker-simulation / AKIA56R3F7MWGDKVXG67 |

## Detection

**How was the incident discovered?**

Automated GuardDuty alert for unusual API activity from IAM user attacker-simulation.

## Timeline of Events

| Time (UTC) | Event | Details | Evidence Source |

|------------|-------|---------|-----------------|

| 16:53:02 | Attacker credentials created | Access key created for attacker-simulation | CloudTrail — management event (CLI lookup-events) |

| 16:53:22 | Attacker authenticated | GetCallerIdentity called | CloudTrail — management event (CLI lookup-events + Logs Insights) |

| 16:58:13 | Reconnaissance | ListBuckets — attacker surveyed all S3 buckets | CloudTrail — management event (CLI lookup-events + Logs Insights) |

| 17:02:22 | Data discovery | ListObjects on target-incident-evidence — attacker enumerated bucket contents | CloudTrail — data event (CloudWatch Logs Insights only) |

| 17:05:33 | Data exfiltration | GetObject on sensitive-data.txt — file downloaded from target-incident-evidence | CloudTrail — data event (CloudWatch Logs Insights only) |

| 17:06:55 | Containment | Access key AKIA56R3F7MWGDKVXG67 deactivated | Responder action |

| 17:07:10 | Eradication | attacker-simulation user and all credentials deleted | Responder action |

## Affected Resources

| Resource | Type | Impact |

|----------|------|--------|

| target-incident-evidence | S3 Bucket | Unauthorized read access |

| sensitive-data.txt | S3 Object | Confirmed download via GetObject (CloudWatch Logs Insights) |

| attacker-simulation | IAM User | Unauthorized user with S3 read access; now deleted |

## Key Forensic Evidence

**GetObject event (data exfiltration — from CloudWatch Logs Insights):**

| Field | Value |

|-------|-------|

| Event Time | 17:05:33 |

| Event Name | GetObject |

| Username | attacker-simulation |

| Access Key ID | AKIA56R3F7MWGDKVXG67 |

| Source IP | 65.xxx.xxx.xxx |

| Bucket | target-incident-evidence |

| Object Key | sensitive-data.txt |

## Containment Actions

1. Deactivated the compromised access key immediately upon detection

2. Verified the key could no longer authenticate

## Eradication Actions

1. Deleted the compromised access key permanently

2. Removed the attacker's IAM inline policy

3. Deleted the attacker IAM user

## Root Cause

Access keys were committed to a public repository due to lack of .gitignore rules and pre-commit secret scanning.

## Lessons Learned & Recommendations

1. [e.g., "Implement automated secret scanning in CI/CD pipeline (e.g., git-secrets, truffleHog)"]

2. [e.g., "Enable GuardDuty for continuous threat detection and automated alerting"]

3. [e.g., "Use IAM roles and SSO instead of long-lived access keys wherever possible"]

4. [e.g., "Apply least-privilege S3 bucket policies — restrict access to known principals and source VPCs"]

5. [e.g., "Ensure CloudTrail trails with S3 data event logging and CloudWatch Logs integration are configured in all accounts before an incident occurs — evidence that does not exist cannot be recovered"]
