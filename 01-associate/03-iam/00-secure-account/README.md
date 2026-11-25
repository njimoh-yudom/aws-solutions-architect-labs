# 01-associate / 01-iam - Exercise 0: Secure the Account - Create Admin + MFA for Root and Admin

**Date:** Before Nov 24, 2026 (Prerequisite, done before Exercise 1)
**Cost:** $0.00 (IAM is Always Free)
**Status:** Done and Tested

## 1. Business Problem

NexaPay is a new AWS account. Before we create ANY other users, we must secure the account itself.

The AWS account has 2 very powerful accounts by default:

- **Root user** - Created with the email address. Can do EVERYTHING, including delete the account, change billing, and remove all other users. If Root is hacked, NexaPay loses everything.
- **No Admin user yet** - If we use Root for daily work, every click is a risk. One mistake, one phishing link, and the whole account is compromised.

We need:
1. Create ONE Admin user for daily work, so we never use Root again except for Root-only tasks.
2. Turn on MFA for Root AND for Admin, so a stolen password alone is useless.
3. Lock Root away.

This must be done FIRST, before creating the 11 users in Exercise 1.

## 2. Solution / Architecture

| Account | Who Is It? | What Can It Do? | MFA? | When To Use It? |
| :--- | :--- | :--- | :--- | :--- |
| **Root** | Owner, created with email | Everything, including account deletion, billing changes | YES - Authenticator app | ONLY for Root-only tasks: Activate IAM Access to Billing, change account settings, close account. Otherwise never use it. |
| **admin-jimbo** | Cloud Admin (me) | AdministratorAccess - Can do everything EXCEPT a few Root-only tasks | YES - Authenticator app | Daily work: Create users, groups, policies, roles, etc. |

**Rule:** Root = Break glass in emergency only. Admin = Daily work.

## 3. What I Built (Steps)

**Part A - Create Admin User**
1. Signed in as Root (first time, with email).
2. Went to IAM > Users > Create user.
3. User name: `admin-jimbo`
4. Enabled: Provide user access to the AWS Management Console.
5. Set a strong, unique password for Admin (different from Root password).
6. Permissions: Attached policy `AdministratorAccess` directly (for this lab Admin only - in production, Admin would be in an Admins group).
7. Created the user.

**Part B - Enable MFA for Admin**
1. Signed in as `admin-jimbo`.
2. Went to IAM > Users > admin-jimbo > Security credentials > Assign MFA device.
3. Device name: `Admin-Phone`
4. MFA device: Authenticator app (Google Authenticator / Authy).
5. Scanned QR code, entered 2 codes to verify.
6. Result: Admin now requires password + MFA code to sign in.

**Part C - Enable MFA for Root**
1. Signed out, signed in as Root with email.
2. Went to IAM > Dashboard - Saw warning: Add MFA for root.
3. Assigned MFA device: Authenticator app, scanned QR, entered 2 codes.
4. Signed out of Root immediately.
5. Result: Root now requires password + MFA code.

**Part D - Create Account Alias (for easier sign-in)**
1. In IAM > Dashboard > Account alias, created alias `nexapay-lab` (lab alias).
2. Sign-in URL becomes: `https://nexapay-lab.signin.aws.amazon.com/console`
3. All 11 users in Exercise 1 will use this alias + their username + password, not the 12-digit Account ID.

## 4. Verification / Testing (Proof It Works)

### Test 1: Try to sign in as Root without MFA
- Tried to sign in as Root with password only.
- **Result:** AWS asks for MFA code. Cannot proceed without phone code.
- **Expected:** Yes. MFA is enforced.

### Test 2: Try to sign in as Admin without MFA
- Tried to sign in as `admin-jimbo` with password only.
- **Result:** AWS asks for MFA code. Cannot proceed without phone code.
- **Expected:** Yes.

### Test 3: Daily work as Admin
- Signed in as `admin-jimbo` WITH MFA.
- Created 4 Groups and 11 Users in Exercise 1 successfully.
- **Result:** Admin can do all daily IAM work. No need to use Root.
- **Expected:** Yes. This proves Root is not needed for daily work.

### Test 4: Root-only task still needs Root
- Later in Exercise 1, when trying to let `emma-finance` see Bills, got error: "You need permissions".
- Fix required Root: Signed in as Root > Account > IAM user and role access to Billing information > Activated.
- **Result:** Only Root could change that setting. Admin could not.
- **Expected:** Yes. This proves why we keep Root, but locked away with MFA.

## 5. Lessons Learned / What I Would Do Differently in Production

1.  **Secure First, Build Later:** The correct order is ALWAYS: Secure Root + Create Admin + MFA, THEN create other users. I did Admin creation before Exercise 1, but I documented it as Exercise 0 to make the order clear for future readers.
2.  **Never Use Root for Daily Work:** Root can delete the account, change billing, and remove Admin. One phishing click on Root = total loss. Admin can be recreated if compromised, Root cannot be easily recovered.
3.  **MFA is Not Optional:** A password can be phished, guessed, or leaked. MFA code is on your phone, expires in 30 seconds, and cannot be reused. In production, MFA must be enforced for ALL users, not just Admin/Root (we enforce it for Developers in Exercise 5).
4.  **Each Account Needs Unique Password:** Root password, Admin password, and each of the 11 users' passwords must all be different. If one leaks, the others are still safe.
5.  **Account Alias Helps Users:** Users should not have to remember a 12-digit Account ID. An alias like `nexapay-lab` makes sign-in easier and reduces support tickets.

## 6. Cost & Cleanup

- **Cost:** $0.00
- **Cleanup:** No cleanup. IAM Users and MFA are Always Free. Keep Root MFA enabled, keep Admin MFA enabled. Never delete the Admin user. Never disable Root MFA.
- **Next:** Exercise 1 - Users and Groups for 11 users (in folder `01-users-groups/`)
