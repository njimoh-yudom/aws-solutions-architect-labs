# 01-associate / 03-iam - Exercise 2: Custom Policy for Auditors - Least Privilege

**Date:** Nov 26, 2026
**Cost:** $0.00 (IAM + S3 - Small test file, Always Free tier for S3 storage)
**Status:** Done and Tested

## 1. Business Problem

In Exercise 1, Auditors had `ReadOnlyAccess` - they could view EVERYTHING in AWS: EC2, S3, IAM, VPC, RDS, everything.

NexaPay CEO says: "External auditors should ONLY see the audit reports in S3. They should NOT see our EC2 servers, databases, IAM users, or VPC. That is too much access for external auditors."

This is the #1 principle in AWS Security: **Least Privilege** - Give a user ONLY what they need to do their job, nothing more.

If an auditor laptop is stolen, the thief should only see the reports bucket, not the whole company infrastructure.

**Scenario:** NexaPay has an S3 bucket `nexapay-reports-jimbo945839` with a folder `reports/` inside. The 3 auditors (grace-audit, henry-audit, ivan-audit) need to:
- List what files are in `reports/` - `s3:ListBucket`
- Download files from `reports/` - `s3:GetObject`
- NOTHING else - No EC2, No IAM, No other S3 buckets, No delete, No upload

AWS has NO managed policy that does exactly this. We must create a Custom Policy.

## 2. Solution / Architecture

- Created S3 bucket: `nexapay-reports-jimbo945839` in `us-east-1`
- Created folder `reports/` inside the bucket
- Uploaded test file `test-report.txt` (55 bytes, content: "This is a test audit report for NexaPay - Q4 2026")
- Created Custom Policy: `NexaPay-Auditors-Reports-Only`
- Removed `ReadOnlyAccess` from group `Auditors-ReadOnly`
- Attached custom policy to `Auditors-ReadOnly` group
- Now Auditors have ONLY this policy. No other permissions.

## 3. The Policy JSON - Explained

### Full Policy (Final Version)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ListReportsFolderOnly",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::nexapay-reports-jimbo945839",
      "Condition": {
        "StringLike": {
          "s3:prefix": ["", "reports/", "reports/*"],
          "s3:delimiter": ["/"]
        }
      }
    },
    {
      "Sid": "GetReportsOnly",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::nexapay-reports-jimbo945839/reports/*"
    }
  ]
}
```

### What Each Part Means

**Version:** `2012-10-17` - Version of IAM policy language. Always use this.

**Statement 1 - ListReportsFolderOnly:**
- **Effect:** Allow
- **Action:** `s3:ListBucket` - See file names, not download them
- **Resource:** The bucket itself (not files) - You list a bucket, not a file
- **Condition:** You can list ONLY if prefix is `""` (root, to see the reports folder name), `"reports/"` (to enter folder), or `"reports/*"` (to see files inside). And delimiter is `/` so console can group folders.
- In plain English: "Auditors can see the list of file names inside reports/ only."

**Statement 2 - GetReportsOnly:**
- **Effect:** Allow
- **Action:** `s3:GetObject` - Download / read a file
- **Resource:** `nexapay-reports-jimbo945839/reports/*` - Only files inside reports/
- In plain English: "Auditors can download any file inside reports/. They cannot download from any other folder."

**What is NOT in the policy (and why that is the point):**
- No `s3:PutObject` - Cannot upload
- No `s3:DeleteObject` - Cannot delete reports
- No `ec2:*` - No EC2 access at all
- No `iam:*` - No IAM access
- No other bucket - Cannot even list any other bucket

AWS denies by default (implicit deny). We only Allow what is needed.

## 4. Verification / Testing (Proof It Works)

### Test 1: grace-audit - S3 Reports - Should WORK
- Signed in as `grace-audit`
- Went directly to bucket URL (console homepage shows "Create a bucket" because policy does NOT allow `s3:ListAllMyBuckets` - expected with Least Privilege)
- URL: `https://us-east-1.console.aws.amazon.com/s3/buckets/nexapay-reports-jimbo945839?region=us-east-1&tab=objects`
- **Result:** Saw `reports/` folder, opened it, saw `test-report.txt`, downloaded it successfully.
- **Expected:** Yes. List + Get in reports/ allowed.

### Test 2: grace-audit - EC2 - Should FAIL
- Signed in as `grace-audit`, opened EC2 Dashboard
- **Result:** EVERYTHING showed Access denied - Instances, Auto Scaling Groups, Capacity Reservations, Dedicated Hosts, Elastic IPs, Key pairs, Load balancers, Placement groups, Security groups, Snapshots, Volumes, Service health, Account attributes.
- **Expected:** Yes. In Exercise 1, grace-audit could SEE EC2 with ReadOnlyAccess. Now she has ZERO EC2 access. Least Privilege works.

### Test 3: grace-audit - IAM - Should FAIL
- Opened IAM > Users
- **Result:** Access denied.
- **Expected:** Yes. Auditors have no IAM access.

## 5. Lessons Learned / What I Would Do Differently in Production

1.  **Console Needs Extra List Permissions:** First version of policy had Condition `s3:prefix: ["reports/*"]` only. It failed in console with "Insufficient permissions to list objects" because console first asks to list with prefix `""` (empty, root). Fix: Allow `["", "reports/", "reports/*"]` and `s3:delimiter: ["/"]`. A policy that works for CLI/API can fail in console because console makes extra List calls. Always test in console, not just in theory.
2.  **S3 Bucket Names Are Globally Unique:** `nexapay-reports` was already taken. Had to use `nexapay-reports-jimbo945839`. In production, use a naming convention with company + environment + random suffix, e.g., `nexapay-reports-prod-123456`.
3.  **Never Upload Credentials to S3:** During this lab, I accidentally uploaded `alice-dev_credentials.csv` to `reports/`. That file contains Access Keys. I deleted it immediately from S3 with "permanently delete" and deleted it from Downloads. In production, if a credentials file is exposed, you must deactivate those access keys immediately in IAM > Users > Security credentials. Never upload `*_credentials.csv` to S3, GitHub, or email.
4.  **Description Field Restrictions:** AWS Policy Description does NOT allow `/` and `*`. First attempt with description `... in nexapay-reports/reports/*` failed with "Invalid characters. Use alphanumeric and '+=,.@-_'". Fix: Changed to `... in nexapay-reports reports folder`. Description is not the policy - it is just a label.
5.  **Copy-Paste Tags Break JSON:** Copying JSON directly from chat adds hidden `[entity-...]` tags (e.g., `arn: [entity-aws;...] :s3:::`) which break AWS validation with red error on line 8. Fix: Paste into Notepad first, then copy from Notepad to AWS. Same lesson as GitHub READMEs. For all future labs, use downloadable files, not chat copy-paste.
6.  **ListAllMyBuckets vs ListBucket:** S3 console homepage tries `s3:ListAllMyBuckets` (show all buckets). Our policy only allows `s3:ListBucket` on ONE bucket. So console homepage shows "Create a bucket" page, not the bucket list. That is expected. Auditors must be given a direct link to the bucket, not the console homepage. In production, document the direct bucket URL for auditors.

## 6. Cost & Cleanup

- **Cost:** $0.00 - IAM policy is free. S3 bucket with 1 small file (55 bytes) is within Always Free / negligible. No EC2 instances created.
- **Cleanup:** Keep bucket for next exercises (Exercises 3-6 may reuse it). If you want to delete it later: Empty the bucket first (delete test-report.txt), then delete bucket. Do NOT leave credentials files in it.
- **Next:** Exercise 3 - MFA and Password Policy
