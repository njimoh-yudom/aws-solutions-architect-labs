# AWS Solutions Architect Labs - NexaPay Project

Hi, I'm **Njimoh Yudom** — Aspiring AWS Solutions Architect (Associate to Professional) to AWS Cloud Security.

This repo is my hands-on portfolio. I don't just watch tutorials. I build, test, break, and document everything as if I was working for a real company.

**Fictional Company:** NexaPay — A fintech startup in Douala, Cameroon.
**My Role:** Cloud Admin / Solutions Architect

**Course Followed:** AWS Solutions Architect Associate (SAA) - Full course from Introduction to Exam Preparation. This repo follows the course order, module by module.

## How I Work

For every lab:
1.  **Business Problem:** Why does NexaPay need this?
2.  **Solution:** What did I build?
3.  **Verification:** How did I prove it works? (I log in as different users and test)
4.  **Lessons Learned:** What went wrong and what I would do differently in production?
5.  **Cost:** Exact cost + cleanup steps. I track cost because Cost Optimization is a core AWS pillar.

**Cost Strategy:** My AWS account is more than 12 months old, so the 12-month Free Tier has expired. I use **Always Free** services where possible, smallest instance sizes, and I **terminate** resources after each lab. I also have a $5 Billing Budget Alarm.

## Repo Structure - Follows My SAA Course Order

This structure matches the course I am following. I create a folder only when I start that module. Folders marked Planned are not created yet.

```
aws-solutions-architect-labs/
├── 01-associate/
│   ├── 00-introduction/               <- Course overview, how to use the platform
│   ├── 01-supporting-courses/         <- Supplemental foundational material
│   ├── 02-aws-fundamentals/           <- Regions, Availability Zones, service categories
│   ├── 03-iam/                        <- YOU ARE HERE - Users, groups, roles, policies
│   │   ├── 00-secure-account/         <- Exercise 0: Secure Root + Create Admin + MFA (DO FIRST)
│   │   ├── 01-users-groups/           <- Exercise 1: 4 Groups, 11 Users
│   │   ├── 02-custom-policy/          <- Exercise 2: Coming next
│   │   ├── 03-mfa-password-policy/    <- Exercise 3: Coming next
│   │   ├── 04-roles/                  <- Exercise 4: Coming next
│   │   ├── 05-enforce-mfa/            <- Exercise 5: Coming next
│   │   └── 06-audit/                  <- Exercise 6: Coming next
│   ├── 04-s3/                         <- Object storage, security, lifecycle
│   ├── 05-ec2/                        <- Virtual servers, instance types, pricing
│   ├── 06-ebs-efs/                    <- Block and file storage
│   ├── 07-databases/                  <- RDS, DynamoDB, Redshift, Aurora, ElastiCache
│   ├── 08-vpc-networking/             <- Subnets, route tables, security groups, gateways
│   ├── 09-route53/                    <- DNS management and routing policies
│   ├── 10-elb/                        <- Elastic Load Balancing
│   ├── 11-monitoring/                 <- CloudWatch and monitoring
│   ├── 12-high-availability-scaling/  <- Auto Scaling, fault tolerance
│   ├── 13-decoupling-workflows/       <- SQS, SNS, API Gateway
│   ├── 14-big-data/                   <- Kinesis, Athena, data processing
│   ├── 15-serverless/                 <- Lambda and serverless
│   ├── 16-security/                    <- KMS, WAF, Shield, security best practices
│   ├── 17-automation/                 <- CloudFormation and Infrastructure as Code
│   ├── 18-caching/                    <- CloudFront and ElastiCache strategies
│   ├── 19-governance/                 <- Organizations, Trusted Advisor, cost management
│   ├── 20-migration/                  <- Snowball, Storage Gateway, DMS
│   ├── 21-exam-preparation/           <- Practice exams and final review
│   └── 22-conclusion/                 <- Course wrap-up
└── 02-professional/ <- After Associate
    ├── 01-multi-account/
    └── 02-advanced-networking/
```

**Important Order:** In IAM (03-iam), Exercise 0 MUST come before Exercise 1. You secure the account (Root + Admin + MFA) BEFORE you create any other users. This is AWS best practice.

## Progress

| Module | Status |
| :--- | :--- |
| 00-introduction | Done - Course overview |
| 01-supporting-courses | Done |
| 02-aws-fundamentals | Done |
| 03-iam / 00-secure-account | Done - Secure Root + Admin + MFA |
| 03-iam / 01-users-groups | Done - Exercise 1 Done |
| 03-iam / 02-custom-policy | Next |
| 04-s3 | Planned |
| 05-ec2 | Planned |
| 06-ebs-efs | Planned |
| 07-databases | Planned |
| 08-vpc-networking | Planned |
| 09-route53 | Planned |
| 10-elb | Planned |
| 11-monitoring | Planned |
| 12-high-availability-scaling | Planned |
| 13-decoupling-workflows | Planned |
| 14-big-data | Planned |
| 15-serverless | Planned |
| 16-security | Planned |
| 17-automation | Planned |
| 18-caching | Planned |
| 19-governance | Planned |
| 20-migration | Planned |
| 21-exam-preparation | Planned |
| 22-conclusion | Planned |
| 02-professional | After Associate |

## Completed Labs

### 03-iam / 00-secure-account: Secure the Account
- **Status:** Done
- **Cost:** $0.00 (IAM is Always Free)
- **Built:** 1 Admin user (admin-jimbo) with AdministratorAccess, MFA for Root and Admin, Account Alias
- **Rule Learned:** Never use Root for daily work. Root = break glass only. Admin = daily work.
- **Details:** See `01-associate/03-iam/00-secure-account/README.md`

### 03-iam / 01-users-groups: Users and Groups
- **Status:** Done - Nov 24, 2026
- **Cost:** $0.00 (IAM is Always Free)
- **Built:** 4 Groups, 11 Users for NexaPay
- **Tested:** grace-audit blocked from launching EC2, emma-finance blocked from EC2 but allowed on Billing
- **Details:** See `01-associate/03-iam/01-users-groups/README.md`

## Connect

- GitHub: github.com/njimoh-yudom
- LinkedIn: [Add your LinkedIn link here later]
