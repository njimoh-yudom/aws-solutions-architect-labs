# 01-associate / 03-iam - Exercise 1: Build Your Company Structure with Users and Groups

**Date:** Nov 24, 2026
**Cost:** $0.00 (IAM is Always Free)
**Status:** Done and Tested

## 1. Business Problem

NexaPay is a fintech startup with 11 employees who need AWS access:

- **4 Developers** - Build and test the app. Need to manage most services, but NOT IAM, Billing, or Users.
- **2 Finance Staff** - Only need to see bills, costs, budgets. Must NOT create or delete anything.
- **3 Auditors** - External auditors. Need to see everything (read-only) to audit, but NOT change anything.
- **2 Support Staff** - Help customers. Need to see service status (read-only), but NOT change anything or see billing details.

If we give permissions person-by-person, onboarding a new developer takes too long and offboarding is risky. We need Groups.

## 2. Solution / Architecture

I created **4 IAM Groups** and **11 IAM Users**. Permissions are attached to Groups, NOT to individual Users.

Users are added to Groups. When a user leaves, we remove them from the Group.

| Team | How Many? | User Names | Group | Managed Policy Attached |
| :--- | :--- | :--- | :--- | :--- |
| Developers | 4 | alice-dev, bob-dev, chidi-dev, diana-dev | Developers | PowerUserAccess |
| Finance | 2 | emma-finance, frank-finance | Finance-ReadOnly | AWSBillingReadOnlyAccess |
| Auditors | 3 | grace-audit, henry-audit, ivan-audit | Auditors-ReadOnly | ReadOnlyAccess |
| Support | 2 | julia-support, kevin-support | Support-ReadOnly | ReadOnlyAccess |
| **TOTAL** | **11** | | **4 Groups** | |

**Why these policies?**
- `PowerUserAccess` = Can do almost everything EXCEPT IAM and Organizations. Good for Developers.
- `AWSBillingReadOnlyAccess` = Can only view Billing. Good for Finance.
- `ReadOnlyAccess` = Can view everything, change nothing. Good for Auditors and Support. Same policy, two Groups, so we can separate them later if needed.

## 3. What I Built (Steps)

**Part A - Groups:**
Created 4 Groups in IAM > User groups and attached the policies above.

**Part B - Users:**
Created 11 Users in IAM > Users. For this lab, all users used the same lab password for testing speed. In production, each user would have a unique password and be forced to change it on first sign-in.

**Part C - Verification in Console:**
IAM > User groups shows: Developers (4 users), Finance-ReadOnly (2 users), Auditors-ReadOnly (3 users), Support-ReadOnly (2 users).

## 4. Verification / Testing (Proof It Works)

### Test 1: Auditor - grace-audit tries to launch EC2
- Signed in as `grace-audit`
- Tried to launch an EC2 instance
- **Result:** Instance launch failed - `User grace-audit is not authorized to perform: ec2:CreateSecurityGroup because no identity-based policy allows the ec2:CreateSecurityGroup action.`
- **Expected:** Yes. Auditors have ReadOnlyAccess = can view, cannot create.
- **Cost:** $0.00 - AWS blocked before creating the instance.

### Test 2: Finance - emma-finance tries to view EC2
- Signed in as `emma-finance`
- Opened EC2 Dashboard
- **Result:** Almost every resource showed **Access denied** - Instances, Security groups, Volumes, Key pairs, Load balancers, etc.
- **Expected:** Yes. Finance only has Billing access, no EC2 access.

### Test 3: Finance - emma-finance tries to view Billing
- Signed in as `emma-finance`
- Opened Billing > Bills
- **First attempt:** Got error: "You need permissions - You don't have permission to access billing information"
- **Root Cause Found:** AWS has a second switch: **IAM user and role access to Billing information** was Deactivated (default). Having the IAM policy alone is NOT enough.
- **Fix:** Signed in as Root > Account > IAM user and role access to Billing information > Edit > **Activated** - Got message: "Your IAM Access setting was updated successfully."
- **Second attempt:** Bills page loaded. **Estimated grand total: USD 0.00** for Nov 1-30, 2026. No red error.

## 5. Lessons Learned / What I Would Do Differently in Production

1.  **Billing Access is Two Locks:** An IAM policy (`AWSBillingReadOnlyAccess`) + Account setting (`Activate IAM Access to Billing`) must BOTH be on. This is a classic SAA exam question.
2.  **ReadOnlyAccess is Hard to Find:** Searching `ReadOnlyAccess` in Group creation returns 195 policies. The plain `ReadOnlyAccess` is at the end. Easier to create Group empty, then attach via IAM > Policies.
3.  **Shared Passwords are for Lab Only:** Using the same password for 11 users is OK for speed in a lab, but in production each user must have a unique password and be forced to change it on first sign-in. Otherwise you lose accountability (non-repudiation).
4.  **Groups > Individual Policies:** Groups make onboarding/offboarding 1 step. If a 5th Developer joins, I just add them to Developers group.

## 6. Cost & Cleanup

- **Cost:** $0.00
- **Cleanup:** No cleanup needed. IAM Users, Groups, Policies are Always Free. No EC2, RDS, or S3 buckets were left running. Verified in Billing > Bills: USD 0.00.
- **Next:** Exercise 2 - Custom Policy for Auditors
