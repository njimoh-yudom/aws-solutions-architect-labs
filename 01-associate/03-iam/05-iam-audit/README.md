# 01-associate / 03-iam - Exercise 5: IAM Audit - Credential Report, Access Analyzer, CloudTrail

**Date:** Nov 27, 2026
**Cost:** $0.00 (Credential Report, Access Analyzer External Access, CloudTrail Event History 90 days are all free)
**Status:** Done and Tested

## 1. Business Problem

NexaPay has been running for a few weeks. We have:
- 1 Admin (admin-jimbo)
- 11 users in 4 groups
- 3 custom policies
- 1 Role (NexaPay-EC2-S3-Reports-ReadOnly)
- 1 S3 bucket (nexapay-reports-jimbo945839)

The CEO asks: "Are we actually secure? Who has not set up MFA? Is anything exposed to the public? Did anyone try to do something they should not have?"

In a real company, you do not just build IAM and forget it. You AUDIT it every month. AWS gives you 3 free tools for this, all in IAM / CloudTrail.

## 2. Solution / Architecture

- **Credential Report** - CSV that shows for every user: password_enabled, mfa_active, password_last_used, access_key_1_active, access_key_1_last_used_date.
- **Access Analyzer** - Analyzer `nexapay-analyzer`, Type: Resource analysis - External access (free). Checks if any resource (S3, IAM Role, etc.) allows access from outside the account.
  - Did NOT enable Internal access or Unused access analyzers - those are paid (See Pricing). Only External access is free.
- **CloudTrail Event History** - Free for 90 days, already enabled by default. Logs every API call: who, what, when, from which IP, success or AccessDenied. No Trail created (Trails with S3 storage would cost money).

## 3. Verification / Testing (Proof It Works)

### Part A - Credential Report
- Path: IAM -> Credential report -> Download report (.csv)
- **Result:**
  - Root account: `mfa_active = TRUE`
  - `admin-jimbo`: `mfa_active = TRUE`
  - `alice-dev`: `mfa_active = TRUE` (set up in Exercise 3)
  - Other 10 users (bob-dev, carol-dev, david-dev, emma-finance, frank-finance, henry-audit, ivan-audit, judy-support, karen-support): `mfa_active = FALSE`
- **Audit Finding:** 10 out of 12 users (83%) have not enrolled in MFA. They are blocked by the `NexaPay-Require-MFA` Deny policy until they set up MFA, so they are safe but also stuck - they cannot work. In production: Send MFA setup guide to all 10 users, deadline 7 days. This is exactly what a real auditor would write.

### Part B - Access Analyzer
- Path: IAM -> Access Analyzer -> Analyzer settings -> Create analyzer
- Analyzer: `nexapay-analyzer`, Zone of trust: Current account (961341527302), Type: External access
- **Result:** Green banner "Successfully created analyzer nexapay-analyzer." Findings (0) - "No findings to show"
- **Expected:** Yes. S3 bucket has Block Public Access ON, policies are Least Privilege, no resource is shared outside account. In a real audit: "Access Analyzer: 0 external access findings - No resources exposed to the public."

### Part C - CloudTrail Event History
- Path: CloudTrail -> Event history (left menu). Did NOT click Create a trail. Did NOT click CloudTrail Lake (paid). Event history is free for 90 days.
- **Result - admin-jimbo events (proves logging works):**
  - `CreateAnalyzer` - Nov 27, 14:30:15 - admin-jimbo - nexapay-analyzer
  - `TerminateInstances` - Nov 27, 11:06:56 - admin-jimbo - i-04b88fa983d81d81f (Exercise 4 termination)
  - `SendSSHPublicKey` - Nov 27, 10:47:44 - admin-jimbo - EC2 Instance Connect
- **Result - grace-audit events (filtered by User name = grace-audit):**
  - `ListManagedNotifications` - Nov 26, 02:41:17 - grace-audit
  - `GetTotal` - Nov 26, 02:41:16 - grace-audit - billingconsole.amazonaws.com (Billing page view)
  - `GetAccountPlanState` - Nov 26, 02:41:16 - grace-audit - freetier.amazonaws.com
  - `GetCommercialInvoice...`, `GetAccountInformation`, `GetBillingNotifications` - Nov 26, 02:41 - grace-audit - Billing page views
  - These are from Exercise 1 Part D testing where grace-audit could view Billing (ReadOnlyAccess allows Billing View) but could NOT launch EC2.
- **Expected:** Yes. CloudTrail logs every API call with username, time, source IP, success or AccessDenied. If an attacker tries something they should not, we will see it in CloudTrail. The RunInstances denial for grace-audit is on Page 2/3 of her history (earlier, Nov 25), because Page 1 shows newest Billing events first.

## 4. Lessons Learned / What I Would Do Differently in Production

1.  **Audit Every Month:** Build != Secure forever. Users join, leave, change roles. Run Credential Report + Access Analyzer every month. Set a calendar reminder.
2.  **MFA Enrollment Is a People Problem, Not a Tech Problem:** Tech (Deny policy) blocks users without MFA, but they still need a human to chase them to set it up in Week 1. In production, send a setup guide + deadline + follow-up.
3.  **Access Analyzer - Only External Is Free:** On Create Analyzer page, there are 3 options: External access (no additional cost), Internal access (See Pricing = paid), Unused access (See Pricing = paid). For SAA labs, always pick External only. Do not click the paid ones.
4.  **CloudTrail Event History vs Trails:** Event History = Free, 90 days, already on, view in console. Trails = Paid (S3 storage), long-term, for compliance (e.g., keep logs for 1 year). For SAA labs, use Event History. Only create a Trail if the task explicitly asks for long-term log retention.
5.  **CloudTrail Lake Banner - Ignore:** Blue banner "CloudTrail Lake will no longer be open to new customers starting May 31, 2026" is about a paid feature. Ignore it. Event History still works.
6.  **Filter by User Name to Audit One Person:** In Event History, change Lookup attributes from Read-only to User name, type the username. That is how you audit one user in an incident: "What did grace-audit do in the last 7 days?"

## 5. Cost & Cleanup

- **Cost:** $0.00 - Credential Report (free), Access Analyzer External (free), CloudTrail Event History (free for 90 days). No Trail created, no S3 bucket for logs.
- **Cleanup:** Nothing to terminate. Analyzer `nexapay-analyzer` kept (free, will keep monitoring). Credential Report CSV is local download, not stored in AWS. S3 bucket kept for future exercises.
- **Next:** Exercise 6 - Final IAM Audit / Cleanup + CLI version of Exercises 2-6
