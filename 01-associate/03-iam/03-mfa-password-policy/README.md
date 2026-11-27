# 01-associate / 03-iam - Exercise 3: Password Policy + Enforce MFA for NexaPay

**Date:** Nov 27, 2026
**Cost:** $0.00 (IAM is Always Free)
**Status:** Done and Tested

## 1. Business Problem

After Exercise 2, NexaPay has 2 security gaps:

1.  **Passwords are weak:** AWS default allows `12345678`. An attacker can brute-force that in minutes.
2.  **MFA is only on Admin/Root:** The 11 users in Exercise 1 have NO MFA. If someone steals `alice-dev` password via phishing, they can log in as Alice with no second check. That is how most real breaches happen - password stolen, no MFA to stop it.

NexaPay security team says: "Every password must be at least 12 characters, with upper, lower, number, and symbol. Users must change password every 90 days, cannot reuse old passwords. AND, no user can do anything in AWS until they turn on MFA. If you don't have MFA, you can't even see S3."

This is AWS Security Best Practice + SAA exam: Password Policy + MFA Enforcement with `aws:MultiFactorAuthPresent`.

## 2. Solution / Architecture

- Set Account Password Policy in IAM > Account settings: Min 12, Upper + Lower + Number + Symbol, Expiration 90 days, Prevent reuse of 5 previous passwords, Allow users to change own password.
- Created Custom Policy: `NexaPay-Require-MFA` - Denies everything except MFA setup if user has no MFA.
- Attached `NexaPay-Require-MFA` to ALL 4 groups (Developers, Finance-ReadOnly, Auditors-ReadOnly, Support-ReadOnly), so it applies to all 11 users.
- Tested with ONE user (`alice-dev`) to prove it works. In production, every user would do MFA setup themselves on first login. We did not set up MFA for all 11 users with 11 phones - that would take 2 hours.

## 3. The Policy JSON - Explained

### Full Policy (Final Fixed Version)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowViewAccountInfo",
      "Effect": "Allow",
      "Action": ["iam:ListUsers", "iam:ListVirtualMFADevices", "iam:ListMFADevices"],
      "Resource": "*"
    },
    {
      "Sid": "AllowManageOwnMFA",
      "Effect": "Allow",
      "Action": ["iam:CreateVirtualMFADevice", "iam:EnableMFADevice", "iam:ResyncMFADevice", "iam:DeleteVirtualMFADevice", "iam:ListMFADevices"],
      "Resource": ["arn:aws:iam::*:mfa/*", "arn:aws:iam::*:user/${aws:username}"]
    },
    {
      "Sid": "AllowManageOwnPassword",
      "Effect": "Allow",
      "Action": ["iam:ChangePassword", "iam:GetUser"],
      "Resource": "arn:aws:iam::*:user/${aws:username}"
    },
    {
      "Sid": "DenyAllExceptMFAIfNoMFA",
      "Effect": "Deny",
      "NotAction": ["iam:CreateVirtualMFADevice", "iam:EnableMFADevice", "iam:ResyncMFADevice", "iam:DeleteVirtualMFADevice", "iam:ListMFADevices", "iam:ListUsers", "iam:ListVirtualMFADevices", "iam:ChangePassword", "iam:GetUser", "sts:GetSessionToken"],
      "Resource": "*",
      "Condition": {"BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}}
    }
  ]
}
```

### What Each Statement Means

**Statement 1 - AllowViewAccountInfo:** Can list users and MFA devices. Needed so console can show MFA page.

**Statement 2 - AllowManageOwnMFA:** Can create/enable/resync/delete your OWN MFA device, and list your MFA devices. Resource is `mfa/*` (any device name) + `user/${aws:username}` (your own user). AWS requires BOTH resources for EnableMFADevice to work.

**Statement 3 - AllowManageOwnPassword:** Can change your own password and get your own user info.

**Statement 4 - DenyAllExceptMFAIfNoMFA:** The key statement. Effect = Deny. NotAction = Everything EXCEPT the MFA setup actions above. Condition = `aws:MultiFactorAuthPresent = false` (user has NO MFA). In plain English: "If you do NOT have MFA, DENY everything except setting up MFA or changing your password."

If user HAS MFA, this Deny does NOT apply, and they get their normal group permissions (e.g., PowerUserAccess for Alice).

## 4. Verification / Testing (Proof It Works)

### Test 1: alice-dev - EC2 WITHOUT MFA - Should FAIL
- Signed in as `alice-dev` with password only, no MFA yet.
- Opened EC2 Dashboard.
- **Result:** Access denied on everything.
- **Expected:** Yes. Deny policy blocks her because she has no MFA.

### Test 2: alice-dev - Set Up MFA - Should be ALLOWED
- Still as `alice-dev`, went to IAM > Security credentials > Assign MFA device.
- Device name: `alice-phone`, Device type: Authenticator app.
- **First attempt FAILED:** Error: `User: alice-dev is not authorized to perform: iam:CreateVirtualMFADevice on resource: arn:aws:iam::961341527302:mfa/alice-phone`
- **Root Cause:** First version of policy had Resource `arn:aws:iam::*:mfa/${aws:username}` which only allows a device named exactly `alice-dev`. Device name `alice-phone` != `alice-dev`, so DENIED.
- **Fix:** Changed Resource to `["arn:aws:iam::*:mfa/*", "arn:aws:iam::*:user/${aws:username}"]` - Allow any MFA device name + user ARN. Saved policy as admin-jimbo.
- **Second attempt:** Set up device page loaded, showed QR code. Scanned QR with phone, added entry `alice-dev` (separate from `admin-jimbo` entry), entered 2 codes.
- **Result:** "MFA device assigned" green banner.
- **Expected:** Yes. Policy allows MFA setup even without MFA.

### Test 3: alice-dev - Sign In AGAIN WITH MFA - Should WORK
- Signed out, signed in again as `alice-dev`. This time AWS asked for MFA code after password. Entered code from phone for `alice-dev`.
- Opened EC2 Dashboard.
- **Result:** EC2 loaded normally - Instances (running): 0, Auto Scaling Groups: 0, Security groups: 1, Volumes: 0, EC2 cost: $0.00, Service health: "This service is operating normally." No Access denied errors.
- **Expected:** Yes. New session has `aws:MultiFactorAuthPresent = true`, so Deny no longer applies. She gets PowerUserAccess back.
- **Important:** Old session (before MFA) still had `MultiFactorAuthPresent = false`, so it still showed Deny errors (e.g., `iam:GetLoginProfile` denied) even after MFA was assigned. Must sign out and sign in again to get a new session with MFA. This is a common gotcha.

## 5. Lessons Learned / What I Would Do Differently in Production

1.  **MFA Device Name Must Match Policy:** If policy uses `${aws:username}` for MFA resource, device name MUST equal username. If you want flexible names like `alice-phone`, use `mfa/*`. Always test with a real device name, not just theory.
2.  **EnableMFADevice Needs TWO Resources:** Needs permission on BOTH `mfa/*` AND `user/${aws:username}`. If you only allow MFA ARN, Enable will fail. AWS docs for self-service MFA show this.
3.  **Old Session Does Not Get MFA Automatically:** After assigning MFA, you MUST sign out and sign in again with MFA code to get a session with `MultiFactorAuthPresent = true`. The old password-only session still has MFA = false and will still be denied. This confused me at first (saw `GetLoginProfile` denied after assigning MFA).
4.  **Deny + NotAction Is Powerful But Dangerous:** `Deny` with `NotAction` is a whitelist: "Deny everything except these". If you forget an action in NotAction (e.g., `sts:GetSessionToken`), users cannot even get a session token. Always include `sts:GetSessionToken` and `iam:GetUser` in the allowed list.
5.  **Test With ONE User, Not All 11:** Setting up MFA for all 11 users with 11 phones would take 2 hours. In production, each user sets up their own MFA on first login. For a lab, proving it with ONE user (`alice-dev`) is enough to prove the policy works. Do not waste time doing all 11.
6.  **Password Policy Is Account-Wide:** Password policy applies to ALL users in the account, including Admin. You cannot set different password policies per group in IAM (you would need AWS Organizations + SCPs or a custom solution for that). For NexaPay, one strong policy for everyone is correct.
7.  **Copy-Paste Tags Break JSON Again:** Same lesson as Exercise 2 - copying JSON from chat adds `[entity-...]` tags. Always paste via Notepad or use downloadable files.

## 6. Cost & Cleanup

- **Cost:** $0.00 - IAM policies, password policy, MFA devices are Always Free. No EC2 instances created. EC2 cost shown in test: $0.00.
- **Cleanup:** Keep policy attached. Keep `alice-dev` MFA enabled. In production, every user would now be forced to set up MFA on next login. For this lab, we leave other 10 users without MFA (they are still blocked by Deny policy, which is correct - they cannot do anything until they set up MFA).
- **Next:** Exercise 4 - Roles (EC2 to S3)
