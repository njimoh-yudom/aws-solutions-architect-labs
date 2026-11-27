# 01-associate / 03-iam - Exercise 6: IAM via CLI - Same Labs, Second Way (Console = Learning, CLI = Speed)

**Date:** Nov 27, 2026
**Cost:** $0.00 (all IAM CLI commands are free)
**Status:** Done and Tested

## 1. Business Problem

Exercises 2-5 were done in the Console (clicking). In a real job, you do not click 100 times to create 100 users. You use the CLI (or IaC) to do it in seconds.

Scenario: Redo Exercises 2-5 in CLI only, same NexaPay lab, second way, to prove Console = Learning, CLI = Speed.

## 2. What Was Built via CLI

- Policy: `NexaPay-Auditors-Reports-Only-CLI` (ARN: arn:aws:iam::961341527302:policy/NexaPay-Auditors-Reports-Only-CLI)
- Password Policy: Same as Exercise 3 (12 chars, symbols, numbers, upper, lower, allow change, 90 days, reuse 5)
- Role: `NexaPay-EC2-S3-Reports-ReadOnly-CLI` (Trust: ec2.amazonaws.com, Permissions: CLI policy above)
- Audit: Credential Report via CLI (generate + get)

## 3. Setup - Laptop CLI Login (The Key Lesson)

- First error: `aws sts get-caller-identity` -> `NoCredentials: Unable to locate credentials`
- Why: EC2 had a Role (inside the house, no key needed). Laptop is outside the house, needs a key.
- Fix: Created 1 Access Key for `admin-jimbo` (IAM -> Users -> admin-jimbo -> Security credentials -> Create access key -> CLI), then `aws configure` (Access Key ID + Secret + us-east-1 + json).
- `aws login` is for SSO (IAM Identity Center), NOT for IAM Users. For IAM Users, use `aws configure`.
- Security Lesson: Secret in a screenshot = Leaked. First 2 keys were deleted after screenshots showed the Secret in plain text. Rule: If a Secret is ever seen in a screenshot, delete it and make a new one. If it is only visible on your own screen and you do NOT screenshot it, it is NOT leaked. After final key, ran `clear` and closed terminal to throw away scroll-back. Clicked Done on browser Retrieve page so Secret is no longer visible there.

## 4. Verification / Testing

### Part 1 - Custom Policy via CLI (Exercise 2 in CLI)
- Command: `cat > /tmp/NexaPay-Auditors-Reports-Only-CLI.json` (JSON with ListBucket in reports/ + GetObject in reports/*)
- Command: `aws iam create-policy --policy-name NexaPay-Auditors-Reports-Only-CLI --policy-document file:///tmp/NexaPay-Auditors-Reports-Only-CLI.json`
- Result: `PolicyName: NexaPay-Auditors-Reports-Only-CLI`, `Arn: arn:aws:iam::961341527302:policy/NexaPay-Auditors-Reports-Only-CLI`, `CreateDate: 2026-11-27T18:02:22`
- Second run: `EntityAlreadyExists: A policy called NexaPay-Auditors-Reports-Only-CLI already exists` - Expected, because I ran it twice. Not a failure.
- Verify: `aws iam get-policy --policy-arn arn:aws:iam::961341527302:policy/NexaPay-Auditors-Reports-Only-CLI` -> Shows PolicyName + ARN.

### Part 2 - Password Policy via CLI (Exercise 3 in CLI)
- Command: `aws iam update-account-password-policy --minimum-password-length 12 --require-symbols --require-numbers --require-uppercase-characters --require-lowercase-characters --allow-users-to-change-password --max-password-age 90 --password-reuse-prevention 5`
- Result: No output, fresh prompt = Success (CLI is quiet when it works).
- Verify: `aws iam get-account-password-policy` -> `MinimumPasswordLength: 12, RequireSymbols: true, RequireNumbers: true, RequireUppercase: true, RequireLowercase: true, AllowUsersToChangePassword: true, ExpirePasswords: true, MaxPasswordAge: 90, PasswordReusePrevention: 5`

### Part 3 - Role via CLI (Exercise 4 in CLI)
- Command: `cat > /tmp/trust-policy.json` (Trust: ec2.amazonaws.com, Action: sts:AssumeRole)
- Command: `aws iam create-role --role-name NexaPay-EC2-S3-Reports-ReadOnly-CLI --assume-role-policy-document file:///tmp/trust-policy.json`
- Result: `RoleName: NexaPay-EC2-S3-Reports-ReadOnly-CLI`, `Arn: arn:aws:iam::961341527302:role/NexaPay-EC2-S3-Reports-ReadOnly-CLI`, `CreateDate: 2026-11-27T18:27:10`
- Command: `aws iam attach-role-policy --role-name NexaPay-EC2-S3-Reports-ReadOnly-CLI --policy-arn arn:aws:iam::961341527302:policy/NexaPay-Auditors-Reports-Only-CLI`
- Result: No output = Success.

### Part 4 - Audit via CLI (Exercise 5 in CLI)
- Command: `aws iam generate-credential-report` -> First: `State: STARTED, Description: No report exists. Starting a new report generation task` (Wait 10 sec, run again)
- Second run: `State: COMPLETE`
- Command: `aws iam get-credential-report --query Content --output text | base64 -d | head -5`
- Result: CSV header + Root (mfa_active=true), admin-jimbo (true), alice-dev (true), bob-dev (false). Same as console Credential Report in Exercise 5.

## 5. Lessons Learned

1.  **Hidden Entity Tags - Copy-Paste Trap:** When you copy from chat to terminal, chat adds hidden tags like `[entity-aws|canonical_name=AWS]`. Terminal shows them as `^[[200~[entity-aws...]`. Fix: Copy to Notepad first, delete the whole `[entity-...]` tag, type just `aws` (3 letters). Or type the command by hand. For GitHub READMEs, always copy to Notepad first. For terminal, typing by hand is safest.
2.  **Access Key ID vs Secret:** Access Key ID (`AKIA...`) is like username/house number - allowed to be visible, terminal WILL show it. Secret (long, no AKIA) is like password - must be hidden. When `aws configure` asks for Secret, it tries to hide it (Mode 2 - Stay quiet). When you paste a long string, Ubuntu sometimes bypasses hiding and shows it. If it shows on YOUR screen only and you do NOT screenshot it, it is NOT leaked. If you screenshot it and send it, it IS leaked and must be deleted.
3.  **Terminal Focus:** If cursor is in LibreOffice Writer behind terminal, typing goes to Writer, not terminal. Writer shows `Selected: 1 word`. Fix: Click Terminal icon in sidebar, click inside black terminal, press Ctrl+C to cancel, `clear` to clean, start again. Always check which window is on top.
4.  **CLI is Quiet on Success:** Console shows green banners. CLI shows nothing on success for `update-account-password-policy`, `attach-role-policy`. No output = Success. Error = Red `aws: [ERROR]`.
5.  **STARTED vs COMPLETE:** `generate-credential-report` first says STARTED (cooking), second says COMPLETE (ready). Wait 10 seconds and run again.
6.  **Key Pair vs Instance Connect vs Session Manager:** Key Pair = Metal key file (.pem), lose it = locked out. Instance Connect = Hotel reception, temporary key, needs Public IP + Port 22. Session Manager = Secret tunnel, no Public IP, no Port 22, needs SSM Role, logs commands. For exam: Private Subnet, no public IP, no bastion -> Session Manager. Quick browser connect, public IP -> Instance Connect.

## 6. Cost & Cleanup

- **Cost:** $0.00 - All IAM CLI commands free. No EC2 launched in Exercise 6. No new charges.
- **Cleanup:** CLI Policy and CLI Role kept (free, for reference). They have `-CLI` suffix so they do not clash with console versions. No EC2 to terminate. Access Key for admin-jimbo kept (current key, after 2 rotations for leaked screenshots). Never upload `*_credentials.csv` to GitHub/S3/email.
- **Next:** Module 04-S3 (deep dive) or Module 05-EC2 (deep dive: Instance Connect vs SSH vs Session Manager, with bonus lab: EC2 with NO public IP + Session Manager)
