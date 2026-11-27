# 01-associate / 03-iam - Exercise 4: IAM Roles - EC2 to S3 (No Credentials on the Instance)

**Date:** Nov 27, 2026
**Cost:** ~$0.01 (t3.micro in us-east-1, running ~35 minutes, then terminated)
**Status:** Done and Tested, Instance Terminated

## 1. Business Problem

NexaPay has an app running on an EC2 instance. That app needs to read files from `nexapay-reports-jimbo945839/reports/`.

**The WRONG way (what beginners do):**
Create an Access Key for `alice-dev`, copy the Secret Key, paste it into a file on the EC2 instance, and let the app use it.

**Why that is terrible:**
- If the EC2 instance is hacked, attacker steals Alice keys and can do everything Alice can do (PowerUser).
- If Alice leaves the company, you have to change keys on every EC2 instance.
- Keys are stored in plain text on disk - anyone who gets the disk gets the keys.

**The RIGHT way (AWS Best Practice + SAA exam):**
Create an IAM Role for EC2. Attach the Role to the EC2 instance. The instance automatically gets temporary credentials that rotate every hour. No keys stored on disk. No keys to steal. If you delete the Role, the instance loses access instantly.

**Scenario:** EC2 instance `nexapay-app-server` needs to:
- List and Download files from `nexapay-reports-jimbo945839/reports/` - Same as auditors, but for a machine, not a person.
- NOTHING else - No EC2 control, No IAM, No other buckets.

## 2. Solution / Architecture

- Created IAM Role: `NexaPay-EC2-S3-Reports-ReadOnly`
  - Trust Policy: `ec2.amazonaws.com` - Only EC2 instances can assume this Role, not users.
  - Permissions Policy: Reused `NexaPay-Auditors-Reports-Only` from Exercise 2 (List + Get in reports/ only). Reusing policies is good practice.
- Launched EC2 Instance: `nexapay-app-server` (i-04b88fa983d81d81f)
  - AMI: Amazon Linux 2023
  - Instance type: t3.micro (1 vCPU, 1 GiB, $0.0104 per hour in us-east-1)
  - Key pair: Proceed without a key pair (used EC2 Instance Connect, not SSH keys)
  - IAM role: `NexaPay-EC2-S3-Reports-ReadOnly` attached in Advanced details
  - Region: us-east-1
  - Public IP: 34.224.71.180 (for this lab only)
- Tested from INSIDE the instance via EC2 Instance Connect terminal.
- Terminated instance immediately after test.

## 3. What Is a Role vs a User? (Key SAA Concept)

| | IAM User | IAM Role |
| :--- | :--- | :--- |
| Who is it for? | A person (Alice, Bob) or an app that needs long-term access | A machine / service / temporary access (EC2, Lambda, another AWS account) |
| Credentials? | Long-term: Password + Access Keys (never expire unless you rotate them) | Temporary: Temporary credentials, rotate automatically every 1 hour |
| How to use? | Sign in with username + password, or use Access Key in code | Assume the Role, get temporary credentials via STS |
| Example | `alice-dev` signs in to console | EC2 instance assumes `NexaPay-EC2-S3-Reports-ReadOnly` |

**Trust Policy** = WHO can assume the Role (e.g., ec2.amazonaws.com).
**Permissions Policy** = WHAT can the Role do once assumed (e.g., S3 List + Get in reports/).

## 4. Verification / Testing (Proof It Works - All 5 Tests from Inside EC2)

Connected via EC2 Instance Connect to `nexapay-app-server`, ran commands in terminal:

### Test 1: No Credentials on Disk - Should show No File
- Command: `ls -la ~/.aws/`
- **Result:** `ls: cannot access '.aws/': No such file or directory`
- **Expected:** Yes. No credentials file exists. That is the point of Roles.

### Test 2: Instance Uses Role, Not User - Should show assumed-role
- Command: `aws sts get-caller-identity`
- **Result:** `"Arn": "arn:aws:sts::961341527302:assumed-role/NexaPay-EC2-S3-Reports-ReadOnly/i-04b88fa983d81d81f"`
- **Expected:** Yes. `assumed-role` proves it is using the Role, not a user.

### Test 3: List S3 Reports - Should WORK
- Command: `aws s3 ls s3://nexapay-reports-jimbo945839/reports/`
- **Result:** `2026-11-25 23:31:01 55 test-report.txt`
- **Expected:** Yes. Role allows List in reports/.

### Test 4: Download S3 File - Should WORK (with a gotcha)
- First attempts: `aws s3 cp s3://nexapay-reports-jimbo945839/reports/test-report.txt` (without destination)
- **Result:** Error: `aws: [ERROR]: the following arguments are required: paths`
- **Root Cause:** `aws s3 cp` needs TWO paths - source + destination. I missed the destination dot `.` at the end. Also first attempt had typo `test-reports.txt` (with s) vs real file `test-report.txt` (no s). The dot `.` is tiny at end of line, easy to miss when copy-pasting.
- **Fix:** Pressed Up Arrow, added `space + .` at end: `aws s3 cp s3://nexapay-reports-jimbo945839/reports/test-report.txt .`
- **Result:** `download: s3://nexapay-reports-jimbo945839/reports/test-report.txt to ./test-report.txt`
- Command: `cat test-report.txt`
- **Result:** `This is a test audit report for NexaPay - Q4 2026`
- **Expected:** Yes. Download + read works.

### Test 5: EC2 DescribeInstances - Should FAIL (Least Privilege)
- Command: `aws ec2 describe-instances`
- **Result:** `aws: [ERROR]: An error occurred (UnauthorizedOperation) when calling the DescribeInstances operation: You are not authorized to perform this operation. User: arn:aws:sts::961341527302:assumed-role/NexaPay-EC2-S3-Reports-ReadOnly/i-04b88fa983d81d81f is not authorized to perform: ec2:DescribeInstances`
- **Expected:** Yes. Role only allows S3, not EC2. Least Privilege works for machines too.

## 5. Lessons Learned / What I Would Do Differently in Production

1.  **Never Put Access Keys on EC2:** Always use Roles for EC2, Lambda, ECS, etc. If you see a tutorial that says "put your Access Key on the instance", that tutorial is wrong and insecure. This is a top SAA exam question.
2.  **Roles Use Temporary Credentials:** The instance gets credentials via Instance Metadata Service (IMDSv2) that rotate every hour. You never see them, you never store them. If the instance is compromised, the attacker only gets temporary credentials that expire in 1 hour, not long-term keys.
3.  **`aws s3 cp` Needs Destination:** `aws s3 cp <source> <destination>` - The destination `.` (dot) means "current folder". It is tiny and easy to miss when copy-pasting. Always check for `space + .` at end. Use Up Arrow to recall last command and add the dot.
4.  **File Names Must Match Exactly:** `test-report.txt` vs `test-reports.txt` (with s) - S3 is case-sensitive and exact. One letter wrong = file not found. Always use `aws s3 ls` to see the exact file name before you `cp`.
5.  **Status Checks: 2/2 = System + Instance:** On Instances list page, it says `2/2 checks passed`. On Instance details page, it shows as 2 lines: System status = Check passed, Instance status = Check passed. Same thing. Don't wait for a literal "2/2" on the details page - look for both lines saying Check passed.
6.  **EC2 Costs Even If You Do Nothing:** t3.micro is $0.0104 per hour. My instance ran ~35 minutes = ~$0.006. If I had left it overnight, ~$0.25. If I had left it a month, ~$7.50. Always Launch -> Test -> Terminate in same session. Check Billing > Bills after termination.
7.  **EC2 Instance Connect vs SSH - Side Note for EC2 Chapter:** For this lab we used EC2 Instance Connect (browser-based, no key pair, uses IAM). In EC2 Chapter (Module 05-EC2), I will do a full deep-dive on Instance Connect vs SSH vs Systems Manager Session Manager - meanings, differences, implications, use cases. Saved as must-cover topic.

## 6. Cost & Cleanup

- **Cost:** ~$0.01 - t3.micro, 1 instance, ~35 minutes in us-east-1. Verified in Billing after termination.
- **Cleanup:** Instance `nexapay-app-server` (i-04b88fa983d81d81f) **TERMINATED** immediately after test. No EBS volume left (default Delete on Termination). S3 bucket kept for future exercises. IAM Role `NexaPay-EC2-S3-Reports-ReadOnly` kept (free, will reuse for future EC2 labs).
- **Next:** Exercise 5 - Audit / CloudTrail or Exercise 6 - Final IAM Audit (to be confirmed)
