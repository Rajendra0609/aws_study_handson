# 🔐 AWS IAM — Complete Learning & Lab Guide
### Identity & Access Management: Theory + Hands-On Labs for TechNova Startup
### Includes: Console UI Steps + CLI Commands for Every Action

---

## 📌 What is IAM?

AWS IAM (Identity and Access Management) is the **central security service** of AWS. It controls **who** can access AWS and **what** they can do. Every AWS service — S3, EC2, Lambda, RDS — depends on IAM for access control.

> ⚠️ **Rule #1:** Never use the Root account for day-to-day tasks. Create IAM users immediately after account setup.

---

## 🗺️ IAM Core Concepts (Big Picture)

```
AWS Account (Root)
│
├── Users         → Real people or applications (have credentials)
├── Groups        → Collection of users (attach policies to groups)
├── Roles         → Temporary identity assumed by services or users
├── Policies      → JSON documents that define permissions
└── MFA           → Extra security layer on top of passwords
```

---

## 🧱 IAM Policy Structure

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": ["arn:aws:s3:::my-bucket", "arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

| Field | Description | Values |
|-------|-------------|--------|
| `Version` | Policy language version | Always `"2012-10-17"` |
| `Effect` | Allow or deny the action | `"Allow"` or `"Deny"` |
| `Action` | AWS API actions | e.g., `s3:GetObject`, `ec2:*` |
| `Resource` | Which resources it applies to | ARN or `"*"` for all |
| `Condition` | Optional conditions | IP, time, MFA, tags |

**Types of Policies:**

| Type | Description | Use Case |
|------|-------------|----------|
| **AWS Managed** | Created & maintained by AWS | Common use cases |
| **Customer Managed** | You create & manage | Custom permissions |
| **Inline** | Embedded inside user/role/group | One-off strict control |
| **Resource-based** | Attached to a resource (e.g., S3 bucket policy) | Cross-account access |

---

## 🏢 Startup Scenario

**Company:** `TechNova Pvt Ltd` — A SaaS product startup  
**Team Size:** 10 People  
**Product:** A web application hosted on AWS  
**Your Role:** You are the AWS Admin / DevOps Engineer

---

## 👥 Team & Roles

| # | Person | Job Title | What They Do on AWS |
|---|--------|-----------|---------------------|
| 1 | Arjun | AWS Admin (You) | Full access — manages all AWS |
| 2 | Priya | Backend Developer | EC2, RDS, Lambda, S3 (read) |
| 3 | Rahul | Backend Developer | EC2, RDS, Lambda, S3 (read) |
| 4 | Sneha | Frontend Developer | S3 (deploy frontend), CloudFront |
| 5 | Karthik | Frontend Developer | S3 (deploy frontend), CloudFront |
| 6 | Divya | Data Engineer | S3 (full), RDS (read), Glue, Athena |
| 7 | Anand | DevOps Engineer | EC2, ECS, ECR, CloudWatch, IAM (read) |
| 8 | Meena | QA Engineer | EC2 (read), S3 (read), CloudWatch logs |
| 9 | Ravi | Finance / Manager | Billing dashboard only |
| 10 | Lakshmi | HR (no AWS needed) | No AWS access |

---

## 🗂️ Full Architecture Plan

```
IAM Groups:
├── grp-admins          → AdministratorAccess (Arjun)
├── grp-backend-devs    → EC2 + RDS + Lambda + S3 read (Priya, Rahul)
├── grp-frontend-devs   → S3 (specific bucket) + CloudFront (Sneha, Karthik)
├── grp-data-engineers  → S3 full + RDS read + Glue + Athena (Divya)
├── grp-devops          → EC2 + ECS + ECR + CloudWatch + IAM read (Anand)
└── grp-qa              → EC2 read + S3 read + CloudWatch (Meena)

IAM Roles (for AWS Services):
├── role-ec2-s3-access       → EC2 instances read/write S3
├── role-lambda-dynamodb     → Lambda accesses DynamoDB
├── role-ec2-cloudwatch      → EC2 sends logs to CloudWatch
└── role-github-actions      → GitHub CI/CD deploys to AWS (no keys!)

Real-World Additions:
├── CloudTrail              → Audit log every single IAM action
├── IAM Access Analyzer     → Detect open/risky permissions
├── Permission Boundaries   → Stop privilege escalation
├── Credential Report       → Full security audit of all users
├── Secrets Manager         → Store DB passwords & API keys
├── Tag-Based Access (ABAC) → Devs only touch tagged resources
├── IP Restriction Policy   → Only allow access from office IP
├── Break-Glass Account     → Emergency access when admin unavailable
├── Offboarding Procedure   → When someone leaves the company
└── Quarterly Access Review → Scheduled security cleanup
```

---

## ⚙️ Prerequisites (CLI Setup)

```bash
# 1. AWS Account (Free Tier is fine for this lab)
# 2. AWS CLI installed and configured
# 3. Root account MFA must be enabled before anything else

# Verify CLI installed
aws --version

# Configure CLI
aws configure
# AWS Access Key ID: [enter key]
# AWS Secret Access Key: [enter secret]
# Default region: ap-south-1
# Default output format: json

# Verify connection
aws iam get-user
```

---

## 🔬 LAB START

---

## LAB 1 — Secure the Root Account

> ⚠️ This is the most critical step. Root = unlimited power with no restrictions.
> Cannot be done via CLI — must be done in browser.

### 🖥️ Console UI Steps

1. Open browser → go to `console.aws.amazon.com`
2. Sign in with your **root email address + password**
3. Top-right corner → click your **account name** → select **Security credentials**
4. Under **Multi-factor authentication (MFA)** → click **Assign MFA device**
5. Enter a device name (e.g., `root-mfa`) → choose **Authenticator app** → click **Next**
6. Open **Google Authenticator** (or Authy) on your phone → tap the **+** icon → **Scan a QR code**
7. Scan the QR code shown on screen
8. Enter **two consecutive 6-digit codes** from the app → click **Add MFA**
9. Scroll down to the **Access keys** section → confirm there are **no access keys**
   - If any exist → click **Actions** → **Delete**
10. Go to **Account** (top-right → account name → Account) → scroll to **IAM user and role access to Billing information** → click **Edit** → check **Activate IAM Access** → **Update**
    > This lets Ravi view billing without using root

```
Root Account Checklist:
  ✅ MFA enabled on root
  ✅ No access keys exist on root
  ✅ Billing access activated for IAM users
  ✅ Logged out of root — never use root again
```

---

## LAB 2 — Create the Admin User (Arjun)

> All daily work is done as Arjun from here. Root stays locked away.

### 🖥️ Console UI Steps

1. Sign in with root (last time!) → go to **IAM** → **Users** → **Create user**
2. **User name:** `arjun-admin`
3. Check **Provide user access to the AWS Management Console** → **Next**
4. Select **I want to create an IAM user**
5. Set password: `TechNova@2024!` → check **User must create a new password at next sign-in** → **Next**
6. Select **Attach policies directly** → search and check **AdministratorAccess** → **Next**
7. Click **Create user**
8. **Download or copy** the Console sign-in URL, username, and password shown
9. To enable MFA for Arjun:
   - Go to **IAM → Users → arjun-admin → Security credentials tab**
   - Click **Assign MFA device** → follow same steps as root MFA above

### 💻 CLI Commands

```bash
# Create Arjun
aws iam create-user --user-name arjun-admin

# Set console password (force reset on first login)
aws iam create-login-profile \
  --user-name arjun-admin \
  --password "TechNova@2024!" \
  --password-reset-required

# Give full admin access
aws iam attach-user-policy \
  --user-name arjun-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Create CLI access keys for Arjun
aws iam create-access-key --user-name arjun-admin
# ⚠️ SAVE AccessKeyId + SecretAccessKey — shown ONLY ONCE

# Switch CLI to use Arjun's credentials (not root)
aws configure
# Enter Arjun's keys now

# Confirm you are now Arjun
aws iam get-user
```

---

## LAB 3 — Create All Groups

### 🖥️ Console UI Steps

1. Go to **IAM** → **User groups** → **Create group**
2. **Group name:** `grp-admins` → scroll down → **Create group**
3. Repeat for: `grp-backend-devs`, `grp-frontend-devs`, `grp-data-engineers`, `grp-devops`, `grp-qa`
4. To verify: **IAM → User groups** → you should see all 6 groups listed

### 💻 CLI Commands

```bash
aws iam create-group --group-name grp-admins
aws iam create-group --group-name grp-backend-devs
aws iam create-group --group-name grp-frontend-devs
aws iam create-group --group-name grp-data-engineers
aws iam create-group --group-name grp-devops
aws iam create-group --group-name grp-qa

# Verify
aws iam list-groups --query 'Groups[].GroupName' --output table
```

---

## LAB 4 — Create Custom Policies (Least-Privilege)

> AWS managed policies are too broad. We write exact permissions needed per role.

### 🖥️ Console UI Steps (same for all policies)

1. Go to **IAM** → **Policies** → **Create policy**
2. Select **JSON** tab → paste the policy JSON below
3. Click **Next** → Enter policy name → **Create policy**

### Policy 1: Backend Developer Policy

```bash
cat > policy-backend-dev.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2Access",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeVpcs"
      ],
      "Resource": "*"
    },
    {
      "Sid": "LambdaAccess",
      "Effect": "Allow",
      "Action": [
        "lambda:CreateFunction",
        "lambda:UpdateFunctionCode",
        "lambda:InvokeFunction",
        "lambda:GetFunction",
        "lambda:ListFunctions",
        "lambda:DeleteFunction"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3ReadOnly",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::technova-*",
        "arn:aws:s3:::technova-*/*"
      ]
    },
    {
      "Sid": "RDSAccess",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBClusters",
        "rds:Connect"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogsRead",
      "Effect": "Allow",
      "Action": [
        "logs:GetLogEvents",
        "logs:FilterLogEvents",
        "logs:DescribeLogGroups"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-backend-dev \
  --policy-document file://policy-backend-dev.json
# SAVE the Policy ARN from output
```

### Policy 2: Frontend Developer Policy

```bash
cat > policy-frontend-dev.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3FrontendOnly",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::technova-frontend",
        "arn:aws:s3:::technova-frontend/*"
      ]
    },
    {
      "Sid": "CloudFrontInvalidate",
      "Effect": "Allow",
      "Action": [
        "cloudfront:CreateInvalidation",
        "cloudfront:GetDistribution",
        "cloudfront:ListDistributions"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-frontend-dev \
  --policy-document file://policy-frontend-dev.json
```

### Policy 3: Data Engineer Policy

```bash
cat > policy-data-engineer.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3DataBuckets",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::technova-data-*",
        "arn:aws:s3:::technova-data-*/*"
      ]
    },
    {
      "Sid": "GlueAll",
      "Effect": "Allow",
      "Action": "glue:*",
      "Resource": "*"
    },
    {
      "Sid": "AthenaQuery",
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryResults",
        "athena:ListWorkGroups",
        "athena:GetQueryExecution"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RDSRead",
      "Effect": "Allow",
      "Action": ["rds:DescribeDBInstances", "rds:DescribeDBSnapshots"],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-data-engineer \
  --policy-document file://policy-data-engineer.json
```

### Policy 4: DevOps Engineer Policy

```bash
cat > policy-devops.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EC2Full",
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    },
    {
      "Sid": "ContainersAll",
      "Effect": "Allow",
      "Action": ["ecs:*", "ecr:*"],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchAll",
      "Effect": "Allow",
      "Action": ["cloudwatch:*", "logs:*"],
      "Resource": "*"
    },
    {
      "Sid": "IAMReadOnly",
      "Effect": "Allow",
      "Action": ["iam:Get*", "iam:List*"],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-devops \
  --policy-document file://policy-devops.json
```

### Policy 5: QA Engineer Policy

```bash
cat > policy-qa.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadAll",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "s3:GetObject",
        "s3:ListBucket",
        "cloudwatch:GetMetricData",
        "cloudwatch:ListMetrics",
        "logs:GetLogEvents",
        "logs:DescribeLogGroups",
        "logs:FilterLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-qa \
  --policy-document file://policy-qa.json
```

---

## LAB 5 — Attach Policies to Groups

### 🖥️ Console UI Steps

1. Go to **IAM** → **User groups** → click `grp-admins`
2. Click **Permissions** tab → **Add permissions** → **Attach policies**
3. Search for **AdministratorAccess** → check it → **Attach policies**
4. Repeat for each group below with their corresponding policy:
   - `grp-backend-devs` → `policy-backend-dev`
   - `grp-frontend-devs` → `policy-frontend-dev`
   - `grp-data-engineers` → `policy-data-engineer`
   - `grp-devops` → `policy-devops`
   - `grp-qa` → `policy-qa`

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $ACCOUNT_ID"

# Admins
aws iam attach-group-policy \
  --group-name grp-admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Backend Devs
aws iam attach-group-policy \
  --group-name grp-backend-devs \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-backend-dev

# Frontend Devs
aws iam attach-group-policy \
  --group-name grp-frontend-devs \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-frontend-dev

# Data Engineers
aws iam attach-group-policy \
  --group-name grp-data-engineers \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-data-engineer

# DevOps
aws iam attach-group-policy \
  --group-name grp-devops \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-devops

# QA
aws iam attach-group-policy \
  --group-name grp-qa \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-qa

echo "✅ All policies attached"
```

---

## LAB 6 — Create All Users and Add to Groups

### 🖥️ Console UI Steps (repeat for each user)

1. Go to **IAM** → **Users** → **Create user**
2. Enter **User name** (e.g., `priya-backend`)
3. Check **Provide user access to the AWS Management Console**
4. Select **I want to create an IAM user** → set a password → check **User must create a new password at next sign-in** → **Next**
5. Select **Add user to group** → check the appropriate group → **Next** → **Create user**

**User → Group mapping:**
| User | Group |
|------|-------|
| `priya-backend` | `grp-backend-devs` |
| `rahul-backend` | `grp-backend-devs` |
| `sneha-frontend` | `grp-frontend-devs` |
| `karthik-frontend` | `grp-frontend-devs` |
| `divya-data` | `grp-data-engineers` |
| `anand-devops` | `grp-devops` |
| `meena-qa` | `grp-qa` |
| `ravi-finance` | *(no group — billing policy attached directly)* |

**For Ravi (Billing only):**
- Create user → do NOT add to any group → **Next** → **Create user**
- Then: **IAM → Users → ravi-finance → Permissions → Add permissions → Attach policies directly** → search **Billing** → select `Billing` → **Add permissions**

> ⚠️ Billing must also be enabled from root: **Account → IAM user and role access to Billing → Activate** (done in LAB 1)

### 💻 CLI Commands

```bash
# ---- Backend Devs ----
for USER in priya-backend rahul-backend; do
  aws iam create-user --user-name $USER
  aws iam create-login-profile --user-name $USER \
    --password "TechNova@${USER}1!" --password-reset-required
  aws iam add-user-to-group --group-name grp-backend-devs --user-name $USER
  echo "✅ Created $USER"
done

# ---- Frontend Devs ----
for USER in sneha-frontend karthik-frontend; do
  aws iam create-user --user-name $USER
  aws iam create-login-profile --user-name $USER \
    --password "TechNova@${USER}1!" --password-reset-required
  aws iam add-user-to-group --group-name grp-frontend-devs --user-name $USER
  echo "✅ Created $USER"
done

# ---- Data Engineer ----
aws iam create-user --user-name divya-data
aws iam create-login-profile --user-name divya-data \
  --password "TechNova@Divya1!" --password-reset-required
aws iam add-user-to-group --group-name grp-data-engineers --user-name divya-data

# ---- DevOps ----
aws iam create-user --user-name anand-devops
aws iam create-login-profile --user-name anand-devops \
  --password "TechNova@Anand1!" --password-reset-required
aws iam add-user-to-group --group-name grp-devops --user-name anand-devops

# ---- QA ----
aws iam create-user --user-name meena-qa
aws iam create-login-profile --user-name meena-qa \
  --password "TechNova@Meena1!" --password-reset-required
aws iam add-user-to-group --group-name grp-qa --user-name meena-qa

# ---- Finance (Billing only — no group, direct policy) ----
aws iam create-user --user-name ravi-finance
aws iam create-login-profile --user-name ravi-finance \
  --password "TechNova@Ravi1!" --password-reset-required
aws iam attach-user-policy \
  --user-name ravi-finance \
  --policy-arn arn:aws:iam::aws:policy/job-function/Billing

echo "✅ All 9 users created and assigned"
```

---

## LAB 7 — Create IAM Roles for AWS Services

> ⚠️ Real-world rule: EC2 and Lambda NEVER use access keys. They use Roles.

### 🖥️ Console UI Steps (for Role 1: EC2 → S3)

1. Go to **IAM** → **Roles** → **Create role**
2. **Trusted entity type:** AWS service → **Use case:** EC2 → **Next**
3. Search and select **AmazonS3FullAccess** → **Next**
4. **Role name:** `role-ec2-s3-access` → **Create role**

**For Role 2 (Lambda → DynamoDB):** Same steps but:
- Use case: **Lambda**
- Attach: `AmazonDynamoDBFullAccess` + `AWSLambdaBasicExecutionRole`
- Role name: `role-lambda-dynamodb`

**For Role 3 (EC2 → CloudWatch):** Same steps as Role 1 but:
- Attach: `CloudWatchAgentServerPolicy`
- Role name: `role-ec2-cloudwatch`

**Attaching Role to EC2 (Instance Profile) via Console:**
1. Go to **EC2** → **Instances** → select your instance
2. **Actions** → **Security** → **Modify IAM role**
3. Select `role-ec2-s3-access` from the dropdown → **Update IAM role**

### 💻 CLI Commands

```bash
# ---- Trust policy for EC2 ----
cat > trust-ec2.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Role 1: EC2 → S3
aws iam create-role \
  --role-name role-ec2-s3-access \
  --assume-role-policy-document file://trust-ec2.json

aws iam attach-role-policy \
  --role-name role-ec2-s3-access \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

echo "✅ EC2-S3 role created"

# Role 2: Lambda → DynamoDB
cat > trust-lambda.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name role-lambda-dynamodb \
  --assume-role-policy-document file://trust-lambda.json

aws iam attach-role-policy \
  --role-name role-lambda-dynamodb \
  --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess

aws iam attach-role-policy \
  --role-name role-lambda-dynamodb \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

echo "✅ Lambda-DynamoDB role created"

# Role 3: EC2 → CloudWatch
aws iam create-role \
  --role-name role-ec2-cloudwatch \
  --assume-role-policy-document file://trust-ec2.json

aws iam attach-role-policy \
  --role-name role-ec2-cloudwatch \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

echo "✅ EC2-CloudWatch role created"

# ---- Attach Role to EC2 via Instance Profile (CLI) ----
# Without this step, EC2 cannot actually use the role

aws iam create-instance-profile \
  --instance-profile-name profile-ec2-s3-access

aws iam add-role-to-instance-profile \
  --instance-profile-name profile-ec2-s3-access \
  --role-name role-ec2-s3-access

# Attach to a running EC2 instance (replace instance ID)
aws ec2 associate-iam-instance-profile \
  --instance-id i-0abc1234def567890 \
  --iam-instance-profile Name=profile-ec2-s3-access

# Verify the EC2 has the role
aws ec2 describe-instances \
  --instance-ids i-0abc1234def567890 \
  --query 'Reservations[0].Instances[0].IamInstanceProfile'

# Test from INSIDE EC2 (SSH in and run):
# aws s3 ls    ← should work without any aws configure keys!
```

---

## LAB 8 — Enforce MFA for All Users

### 🖥️ Console UI Steps

1. Go to **IAM** → **Policies** → **Create policy** → **JSON tab**
2. Paste the `policy-require-mfa` JSON below → **Next** → name it `policy-require-mfa` → **Create policy**
3. Go to **IAM** → **User groups** → click `grp-backend-devs`
4. **Permissions** tab → **Add permissions** → **Attach policies** → search `policy-require-mfa` → **Attach policies**
5. Repeat step 3-4 for all groups: `grp-frontend-devs`, `grp-data-engineers`, `grp-devops`, `grp-qa`

### 💻 CLI Commands

```bash
cat > policy-require-mfa.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowMFASetup",
      "Effect": "Allow",
      "Action": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ResyncMFADevice",
        "iam:GetAccountPasswordPolicy",
        "iam:ListVirtualMFADevices"
      ],
      "Resource": [
        "arn:aws:iam::*:mfa/${aws:username}",
        "arn:aws:iam::*:user/${aws:username}"
      ]
    },
    {
      "Sid": "DenyEverythingWithoutMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ResyncMFADevice",
        "sts:GetSessionToken"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-policy \
  --policy-name policy-require-mfa \
  --policy-document file://policy-require-mfa.json

for GROUP in grp-backend-devs grp-frontend-devs grp-data-engineers grp-devops grp-qa; do
  aws iam attach-group-policy \
    --group-name $GROUP \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-require-mfa
  echo "MFA enforced on: $GROUP"
done
```

---

## LAB 9 — Set Account Password Policy

### 🖥️ Console UI Steps

1. Go to **IAM** → **Account settings**
2. Under **Password policy** → click **Edit**
3. Configure:
   - Minimum password length: **12**
   - Check: **Require at least one uppercase letter**
   - Check: **Require at least one lowercase letter**
   - Check: **Require at least one number**
   - Check: **Require at least one non-alphanumeric character**
   - Check: **Allow users to change their own password**
   - Enable password expiration: **90 days**
   - Prevent password reuse: **5 passwords**
4. Click **Save changes**

### 💻 CLI Commands

```bash
aws iam update-account-password-policy \
  --minimum-password-length 12 \
  --require-symbols \
  --require-numbers \
  --require-uppercase-characters \
  --require-lowercase-characters \
  --allow-users-to-change-password \
  --max-password-age 90 \
  --password-reuse-prevention 5

echo "✅ Password policy applied:
  - Minimum 12 characters
  - Must have symbols, numbers, upper & lowercase
  - Expires every 90 days
  - Cannot reuse last 5 passwords"
```

---

## LAB 10 — Test Permissions

### 🖥️ Console UI Steps — IAM Policy Simulator

1. Open: [https://policysim.aws.amazon.com](https://policysim.aws.amazon.com)
2. Under **Users, Groups, and Roles** → select `priya-backend`
3. Under **Policy Simulator** → select service: **EC2**
4. Check action: **TerminateInstances**
5. Click **Run Simulation**
6. Expected result: **DENIED** ✅ (Priya cannot terminate EC2)

7. Repeat with action **DescribeInstances** → Expected: **ALLOWED** ✅

8. Select `sneha-frontend` → service: **S3** → action: **GetObject**
   - Resource: `arn:aws:s3:::technova-backend-data/*`
   - Expected: **DENIED** ✅ (Sneha can't access backend bucket)
   - Resource: `arn:aws:s3:::technova-frontend/*`
   - Expected: **ALLOWED** ✅

### 💻 CLI Commands

```bash
# Create test access keys for Priya
aws iam create-access-key --user-name priya-backend

# Add Priya as a CLI profile
aws configure --profile priya
# Enter Priya's keys

# ❌ Should FAIL — Priya cannot terminate EC2
aws ec2 terminate-instances \
  --instance-ids i-1234567890abcdef0 \
  --profile priya
# Expected: AccessDenied ✅

# ✅ Should PASS — Priya can describe EC2
aws ec2 describe-instances --profile priya

# Test Sneha
aws configure --profile sneha  # use sneha-frontend keys

# ❌ Should FAIL — Sneha cannot touch backend bucket
aws s3 ls s3://technova-backend-data --profile sneha
# Expected: AccessDenied ✅

# ✅ Should PASS — Sneha's own bucket
aws s3 ls s3://technova-frontend --profile sneha
```

---

## LAB 11 — S3 Bucket Policies (Resource-Based Policies)

> IAM user policies alone are NOT enough. S3 buckets need their own policy too.
> **Both** must allow for access to work. Either one denying = access denied.

### 🖥️ Console UI Steps

1. Go to **S3** → **Create bucket**
   - Bucket name: `technova-frontend` → Region: `ap-south-1`
   - Keep all defaults → **Create bucket**
2. Click on the bucket → **Permissions** tab → **Bucket policy** → **Edit**
3. Paste the bucket policy JSON below (replace `ACCOUNT_ID`) → **Save changes**
4. Still on **Permissions** tab → **Block public access** → **Edit** → check all 4 boxes → **Save changes**

### 💻 CLI Commands

```bash
# Create the bucket
aws s3api create-bucket \
  --bucket technova-frontend \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Bucket policy — only frontend devs can access
cat > s3-frontend-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFrontendDevsOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::${ACCOUNT_ID}:user/sneha-frontend",
          "arn:aws:iam::${ACCOUNT_ID}:user/karthik-frontend"
        ]
      },
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::technova-frontend",
        "arn:aws:s3:::technova-frontend/*"
      ]
    },
    {
      "Sid": "DenyEveryoneElse",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::technova-frontend",
        "arn:aws:s3:::technova-frontend/*"
      ],
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::${ACCOUNT_ID}:user/sneha-frontend",
            "arn:aws:iam::${ACCOUNT_ID}:user/karthik-frontend",
            "arn:aws:iam::${ACCOUNT_ID}:user/arjun-admin"
          ]
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket technova-frontend \
  --policy file://s3-frontend-bucket-policy.json

# Block all public access
aws s3api put-public-access-block \
  --bucket technova-frontend \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "✅ S3 bucket created with access control and public access blocked"
```

---

## LAB 12 — Enable CloudTrail (Audit Every Action)

> 🔥 Critical for production. Every API call, every login, every IAM change is logged.
> Without this, you have zero visibility into what happened if something goes wrong.

### 🖥️ Console UI Steps

1. Go to **CloudTrail** (search in the top bar) → **Create trail**
2. **Trail name:** `technova-audit-trail`
3. **Storage location:** Create new S3 bucket → name it `technova-cloudtrail-logs-YOURACCOUNT`
4. **Log file SSE-KMS encryption:** optional, can leave off for lab
5. Under **Events:** keep **Management events** checked → **Read** and **Write**
6. Click **Next** → **Next** → **Create trail**
7. Verify: trail status shows **Logging: On** (green)

**Reading CloudTrail logs via Console:**
- Go to **CloudTrail** → **Event history**
- Filter by **User name**, **Event name**, or **Resource name**
- Example: filter Event name = `DeletePolicy` to find who deleted what

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create S3 bucket for logs
aws s3api create-bucket \
  --bucket technova-cloudtrail-logs-${ACCOUNT_ID} \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Bucket policy to allow CloudTrail to write
cat > cloudtrail-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::technova-cloudtrail-logs-${ACCOUNT_ID}"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::technova-cloudtrail-logs-${ACCOUNT_ID}/AWSLogs/${ACCOUNT_ID}/*",
      "Condition": {
        "StringEquals": { "s3:x-amz-acl": "bucket-owner-full-control" }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket technova-cloudtrail-logs-${ACCOUNT_ID} \
  --policy file://cloudtrail-bucket-policy.json

# Create and start the trail
aws cloudtrail create-trail \
  --name technova-audit-trail \
  --s3-bucket-name technova-cloudtrail-logs-${ACCOUNT_ID} \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name technova-audit-trail

# Verify it's active
aws cloudtrail get-trail-status --name technova-audit-trail \
  --query '{Logging: IsLogging, LatestDelivery: LatestDeliveryTime}'

echo "✅ CloudTrail logging every API call to S3"

# Search logs — all IAM actions by Priya in last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=priya-backend \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[].{Time:EventTime, Action:EventName, Source:EventSource}' \
  --output table

# Find all DeletePolicy events across all users
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeletePolicy \
  --query 'Events[].{Time:EventTime, User:Username, Action:EventName}' \
  --output table
```

---

## LAB 13 — IAM Access Analyzer

> Finds resources accidentally accessible from outside your account.
> Example: an S3 bucket policy accidentally allowing public access.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Access Analyzer** (left sidebar) → **Create analyzer**
2. **Analyzer name:** `technova-access-analyzer` → **Zone of trust:** Current account → **Create analyzer**
3. Wait a few minutes → click on the analyzer → go to **Findings** tab
4. Each finding means something in your account is accessible from outside
5. Review each finding → click **Archive** if intentional, or fix the policy if not

**Validate a policy before applying:**
1. **IAM → Access Analyzer → Policy validation tab**
2. Select **Identity-based policy** → paste your JSON → click **Validate**

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Enable Access Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name technova-access-analyzer \
  --type ACCOUNT

# List findings (things that might be over-exposed)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-south-1:${ACCOUNT_ID}:analyzer/technova-access-analyzer \
  --query 'findings[].{Resource:resource, ResourceType:resourceType, Status:status}' \
  --output table

# Validate a custom policy BEFORE applying it
aws accessanalyzer validate-policy \
  --policy-document file://policy-backend-dev.json \
  --policy-type IDENTITY_POLICY \
  --query 'findings[].{Type:findingType, Message:findingDetails, Issue:issueCode}' \
  --output table

echo "✅ Access Analyzer active — scans for unintended external access"
```

---

## LAB 14 — Permission Boundaries (Stop Privilege Escalation)

> Real problem: Anand (DevOps) has IAM read access. What if he creates a new IAM role
> with admin access for himself? Permission Boundaries block this.
> The boundary = the MAXIMUM permissions a user can ever have, regardless of policies.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Policies** → **Create policy** → **JSON tab**
2. Paste the `boundary-devops` JSON below → **Next** → name `boundary-devops` → **Create policy**
3. Go to **IAM** → **Users** → click `anand-devops`
4. **Permissions** tab → scroll to **Permissions boundary** → click **Set boundary**
5. Search `boundary-devops` → select it → **Set boundary**
6. Repeat step 3-5 for `priya-backend` and `rahul-backend` using the `permission-boundary-dev` policy

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > boundary-devops.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServices",
      "Effect": "Allow",
      "Action": ["ec2:*", "ecs:*", "ecr:*", "cloudwatch:*", "logs:*", "s3:*"],
      "Resource": "*"
    },
    {
      "Sid": "DenyIAMWrite",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser", "iam:DeleteUser", "iam:AttachUserPolicy",
        "iam:CreateRole", "iam:DeleteRole", "iam:AttachRolePolicy",
        "iam:CreatePolicy", "iam:DeletePolicy",
        "iam:PutUserPermissionsBoundary", "iam:PutRolePermissionsBoundary"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name boundary-devops \
  --policy-document file://boundary-devops.json

# Apply boundary to Anand — he can never touch IAM or billing now
aws iam put-user-permissions-boundary \
  --user-name anand-devops \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/boundary-devops

# Apply to backend devs too
aws iam put-user-permissions-boundary \
  --user-name priya-backend \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/boundary-devops

aws iam put-user-permissions-boundary \
  --user-name rahul-backend \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/boundary-devops

echo "✅ Permission boundaries applied — devs cannot escalate their own privileges"
```

---

## LAB 15 — GitHub Actions OIDC Role (CI/CD Without Access Keys)

> Real CI/CD deployments never use hardcoded AWS access keys in GitHub secrets.
> GitHub gets a temporary role via OIDC trust — safer, no key rotation needed.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Identity providers** → **Add provider**
2. Provider type: **OpenID Connect**
3. Provider URL: `https://token.actions.githubusercontent.com` → click **Get thumbprint**
4. Audience: `sts.amazonaws.com` → **Add provider**
5. Go to **IAM → Roles → Create role**
6. Trusted entity: **Web identity** → select the GitHub provider → audience: `sts.amazonaws.com`
7. Add condition: `token.actions.githubusercontent.com:sub` = `repo:YOUR-ORG/YOUR-REPO:*`
8. Click **Next** → attach `AmazonECR-FullAccess` + `AmazonECS_FullAccess`
9. Role name: `role-github-actions` → **Create role**

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
GITHUB_ORG="your-org-name"       # Replace with your GitHub org/username
GITHUB_REPO="your-repo-name"     # Replace with your repo name

# Step 1: Register GitHub as OIDC identity provider
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Step 2: Create trust policy
cat > trust-github.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:${GITHUB_ORG}/${GITHUB_REPO}:*"
      }
    }
  }]
}
EOF

# Step 3: Create the role
aws iam create-role \
  --role-name role-github-actions \
  --assume-role-policy-document file://trust-github.json

# Step 4: Attach deploy permissions
aws iam attach-role-policy \
  --role-name role-github-actions \
  --policy-arn arn:aws:iam::aws:policy/AmazonECR-FullAccess

aws iam attach-role-policy \
  --role-name role-github-actions \
  --policy-arn arn:aws:iam::aws:policy/AmazonECS_FullAccess

echo "✅ GitHub Actions role ready — no hardcoded keys needed"
```

**GitHub Actions Workflow `.github/workflows/deploy.yml`:**
```yaml
name: Deploy to AWS

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/role-github-actions
          aws-region: ap-south-1

      - name: Deploy to S3
        run: aws s3 sync ./dist s3://technova-frontend --delete
```

---

## LAB 16 — Tag-Based Access Control (ABAC)

> Instead of creating new policies for every team, use resource TAGS to control access.
> This scales much better as the startup grows.

### 🖥️ Console UI Steps

1. Tag your EC2 instances:
   - Go to **EC2 → Instances** → select your instance
   - **Actions** → **Instance settings** → **Manage tags** → **Add tag**
   - Add: Key=`Environment`, Value=`dev` and Key=`Team`, Value=`backend`
2. Create the tag-based policy via **IAM → Policies → Create policy → JSON tab**
3. Paste the `policy-backend-tag-restricted` JSON → **Create policy**
4. Attach to `grp-backend-devs` group

### 💻 CLI Commands

```bash
# Tag EC2 instances by environment
aws ec2 create-tags \
  --resources i-0abc123 \
  --tags Key=Environment,Value=dev Key=Team,Value=backend

aws ec2 create-tags \
  --resources i-0def456 \
  --tags Key=Environment,Value=prod Key=Team,Value=backend

# Policy: Backend devs can only start/stop EC2 tagged Team=backend
# And cannot stop prod instances without MFA
cat > policy-backend-tag-restricted.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDescribeAlways",
      "Effect": "Allow",
      "Action": "ec2:Describe*",
      "Resource": "*"
    },
    {
      "Sid": "StartStopOnlyTeamResources",
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances", "ec2:RebootInstances"],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Team": "backend"
        }
      }
    },
    {
      "Sid": "DenyProdWithoutMFA",
      "Effect": "Deny",
      "Action": ["ec2:StopInstances", "ec2:TerminateInstances"],
      "Resource": "*",
      "Condition": {
        "StringEquals": { "ec2:ResourceTag/Environment": "prod" },
        "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" }
      }
    }
  ]
}
EOF

echo "✅ ABAC policy ready — tag your resources, not your users"
# Priya (backend dev): CAN stop dev backend EC2 instances
# Priya: CANNOT stop prod instances without MFA
# Priya: CANNOT touch frontend or data instances at all
```

---

## LAB 17 — Restrict Access to Office IP Only

> AWS console and API calls should only come from the office network.

### 🖥️ Console UI Steps

1. Find your office public IP: go to [whatismyip.com](https://whatismyip.com)
2. Go to **IAM → Policies → Create policy → JSON tab**
3. Paste `policy-ip-restriction` JSON below (replace IP) → name `policy-ip-restriction` → **Create policy**
4. Attach to groups: `grp-backend-devs`, `grp-frontend-devs`, `grp-data-engineers`, `grp-qa`
   - Do NOT attach to `grp-admins` (or you'll lock yourself out if working remotely)

### 💻 CLI Commands

```bash
# Find your office IP first: curl ifconfig.me

cat > policy-ip-restriction.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyIfNotFromOfficeIP",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": [
            "203.0.113.50/32",
            "203.0.113.51/32"
          ]
        },
        "Bool": {
          "aws:ViaAWSService": "false"
        }
      }
    }
  ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-policy \
  --policy-name policy-ip-restriction \
  --policy-document file://policy-ip-restriction.json

# Attach to all regular user groups (NOT admins)
for GROUP in grp-backend-devs grp-frontend-devs grp-data-engineers grp-qa; do
  aws iam attach-group-policy \
    --group-name $GROUP \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-ip-restriction
done

echo "✅ Non-admin users can only access AWS from the office IP"
```

---

## LAB 18 — Secrets Manager (Never Hardcode Passwords)

> Developers must NEVER hardcode DB passwords or API keys in code or .env files.
> AWS Secrets Manager stores and auto-rotates them.

### 🖥️ Console UI Steps

1. Go to **Secrets Manager** (search in top bar) → **Store a new secret**
2. **Secret type:** Other type of secret
3. Add key/value pairs:
   - Key: `username` Value: `admin`
   - Key: `password` Value: `SuperSecret@RDS123!`
   - Key: `host` Value: `prod-db.cluster-xyz.ap-south-1.rds.amazonaws.com`
4. Click **Next** → **Secret name:** `technova/prod/db-password` → **Next** → **Next** → **Store**
5. To retrieve in Console: click on the secret → **Retrieve secret value**

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Store the RDS database password
aws secretsmanager create-secret \
  --name "technova/prod/db-password" \
  --description "TechNova production RDS password" \
  --secret-string '{"username":"admin","password":"Sup3rS3cureP@ss!","host":"prod-db.cluster-xyz.ap-south-1.rds.amazonaws.com","port":"5432","dbname":"technova_prod"}'

# Store a third-party API key
aws secretsmanager create-secret \
  --name "technova/prod/stripe-api-key" \
  --description "Stripe payment API key" \
  --secret-string '{"api_key":"sk_live_abc123xyz"}'

# List all secrets
aws secretsmanager list-secrets \
  --query 'SecretList[].Name' --output table

# Create policy so backend EC2 can READ the secret
cat > policy-read-db-secret.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "secretsmanager:GetSecretValue",
      "secretsmanager:DescribeSecret"
    ],
    "Resource": "arn:aws:secretsmanager:ap-south-1:*:secret:technova/prod/*"
  }]
}
EOF

aws iam create-policy \
  --policy-name policy-read-db-secret \
  --policy-document file://policy-read-db-secret.json

# Attach to the EC2 role
aws iam attach-role-policy \
  --role-name role-ec2-s3-access \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-read-db-secret

echo "✅ DB password stored in Secrets Manager — no more .env files with passwords"
```

**How backend code retrieves the secret (Python):**
```python
import boto3
import json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='ap-south-1')
    response = client.get_secret_value(SecretId='technova/prod/db-password')
    secret = json.loads(response['SecretString'])
    return secret['host'], secret['username'], secret['password']

host, user, password = get_db_credentials()
# Connect to DB using these — never stored in code!
```

---

## LAB 19 — Generate IAM Credential Report (Security Audit)

> Run this monthly. Shows every user, their last login, key age, MFA status.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Credential report** (left sidebar)
2. Click **Download credential report**
3. Open the CSV file and check these columns:
   - `mfa_active` → must be `true` for all users
   - `password_last_used` → if `N/A` user never logged in → consider removing
   - `access_key_1_last_used_date` → if older than 90 days → rotate or delete
   - `access_key_1_active` → if `true` but unused → deactivate

### 💻 CLI Commands

```bash
# Generate report
aws iam generate-credential-report

# Wait a moment, then download
sleep 5
aws iam get-credential-report \
  --query 'Content' --output text | base64 --decode > credential-report.csv

# View it
cat credential-report.csv | column -t -s','

echo "
Check these columns:
  password_last_used        → Anyone not logged in 90+ days? Deactivate.
  mfa_active                → Anyone with 'false'? Enforce MFA immediately.
  access_key_1_last_used    → Keys unused in 90 days? Rotate or delete.
  access_key_1_last_rotated → Keys older than 90 days? Rotate now.
"

# Quick scan — find users with NO MFA
aws iam generate-credential-report > /dev/null 2>&1
sleep 3
aws iam get-credential-report --query 'Content' --output text | \
  base64 -d | awk -F',' 'NR>1 && $8=="false" {print "⚠️  NO MFA: "$1}'
```

---

## LAB 20 — Access Advisor (Find & Remove Unused Permissions)

> Access Advisor shows the last time each service was actually used.
> If a dev hasn't used EC2 in 6 months, remove EC2 from their policy.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Users** → click `priya-backend`
2. Click the **Access Advisor** tab
3. Review the list — services with **Not accessed in last X days** can be removed from her policy
4. Repeat for each user and role

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Check what services Priya actually used recently
JOB_ID=$(aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::${ACCOUNT_ID}:user/priya-backend \
  --query 'JobId' --output text)

echo "Waiting for report..."
sleep 8

aws iam get-service-last-accessed-details \
  --job-id $JOB_ID \
  --query 'ServicesLastAccessed[].{Service:ServiceName, LastAccessed:LastAuthenticated}' \
  --output table

# Services showing "never accessed" = safe to remove from her policy
```

---

## LAB 21 — Employee Offboarding (When Someone Leaves)

> When a developer leaves, their access must be removed **the same day**.
> Forgotten accounts are a top security risk.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Users** → click the departing user (e.g., `rahul-backend`)
2. **Security credentials** tab:
   - Under **Console sign-in** → click **Disable console access**
   - Under **Access keys** → click **Deactivate** on each key
3. **Groups** tab → click **Remove** next to each group
4. **Permissions** tab → click **X** next to each attached policy
5. After 7-day review period → **IAM → Users → rahul-backend → Delete**

### 💻 CLI Commands

```bash
DEPARTED_USER="rahul-backend"

echo "🚨 Starting offboarding for: $DEPARTED_USER"

# Step 1: Disable console login IMMEDIATELY
aws iam update-login-profile \
  --user-name $DEPARTED_USER \
  --password "DISABLED-$(date +%s)" \
  --password-reset-required
echo "Step 1 ✅ Console login disabled"

# Step 2: Deactivate all access keys
KEYS=$(aws iam list-access-keys --user-name $DEPARTED_USER \
  --query 'AccessKeyMetadata[].AccessKeyId' --output text)
for KEY in $KEYS; do
  aws iam update-access-key \
    --user-name $DEPARTED_USER \
    --access-key-id $KEY \
    --status Inactive
  echo "Step 2 ✅ Key deactivated: $KEY"
done

# Step 3: Remove from all groups
GROUPS=$(aws iam list-groups-for-user --user-name $DEPARTED_USER \
  --query 'Groups[].GroupName' --output text)
for GROUP in $GROUPS; do
  aws iam remove-user-from-group \
    --group-name $GROUP \
    --user-name $DEPARTED_USER
  echo "Step 3 ✅ Removed from group: $GROUP"
done

# Step 4: Attach DenyAll to immediately revoke all active sessions
aws iam attach-user-policy \
  --user-name $DEPARTED_USER \
  --policy-arn arn:aws:iam::aws:policy/AWSDenyAll
echo "Step 4 ✅ DenyAll attached — all sessions revoked"

# Step 5: Delete access keys permanently
for KEY in $KEYS; do
  aws iam delete-access-key \
    --user-name $DEPARTED_USER \
    --access-key-id $KEY
done
echo "Step 5 ✅ Access keys deleted"

# Step 6: Delete user after 7-day review (uncomment when ready)
# aws iam delete-login-profile --user-name $DEPARTED_USER
# aws iam delete-user --user-name $DEPARTED_USER
echo "Step 6 ⏳ Schedule user deletion after 7-day review"

echo "✅ Offboarding complete. Notify: IT, HR, Project Manager"
echo "   Rotate any shared secrets Rahul may have known"
```

---

## LAB 22 — Break-Glass Emergency Access

> What if Arjun is unavailable? You need emergency access that is:
> locked away normally but usable in a real emergency.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Users** → **Create user** → name: `breakglass-emergency` → **Next**
2. **Attach policies directly** → select **AdministratorAccess** → **Next** → **Create user**
3. Click on the user → **Security credentials** → **Create access key**
4. **Use case:** Other → **Create access key**
5. **⚠️ Print these keys. Store in a physical safe or sealed envelope. NEVER store digitally.**
6. Immediately go back to **Access keys** → **Actions** → **Deactivate** the key
   > Enable only in a real emergency, and disable immediately after

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create break-glass user
aws iam create-user --user-name breakglass-emergency

aws iam attach-user-policy \
  --user-name breakglass-emergency \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Create access keys (store OFFLINE in a password manager vault or physical safe)
aws iam create-access-key --user-name breakglass-emergency
# ⚠️ Print these keys. Store in a physical safe.
# NEVER store digitally unless in a secure vault like 1Password or Bitwarden.

# Disable the key immediately (only enable in real emergency)
aws iam update-access-key \
  --user-name breakglass-emergency \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive

# Set up CloudWatch alarm to alert when this account is used
aws cloudwatch put-metric-alarm \
  --alarm-name "EMERGENCY-BreakGlass-Used" \
  --alarm-description "Break-glass account was accessed — INVESTIGATE IMMEDIATELY" \
  --metric-name "ErrorCount" \
  --namespace "CloudTrailMetrics" \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

echo "✅ Break-glass account created, locked, and monitored"
```

---

## LAB 23 — CloudWatch Alarms for Suspicious IAM Activity

> Get instant email alerts when something dangerous happens.

### 🖥️ Console UI Steps (SNS Alert Setup)

1. Go to **SNS** → **Topics** → **Create topic** → Standard → name: `security-alerts` → **Create topic**
2. Click on the topic → **Create subscription** → Protocol: **Email** → Endpoint: `arjun@technova.com` → **Create subscription**
3. Confirm the subscription from your email inbox

**Create alarms via Console:**
1. Go to **CloudWatch** → **Alarms** → **Create alarm**
2. **Select metric** → **CloudTrailMetrics** → choose your metric → configure threshold
3. Under **Notification** → select `security-alerts` SNS topic
4. Name the alarm (e.g., `ALERT-Root-Account-Used`) → **Create alarm**

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create SNS topic for security alerts
aws sns create-topic --name security-alerts

aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts \
  --protocol email \
  --notification-endpoint arjun@technova.com
# ⚠️ Arjun must confirm the email subscription

LOG_GROUP_NAME="aws-cloudtrail-logs-technova"
aws logs create-log-group --log-group-name $LOG_GROUP_NAME

# Alarm 1: CloudTrail stopped (very suspicious)
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name "CloudTrailStopped" \
  --filter-pattern '{ ($.eventName = StopLogging) }' \
  --metric-transformations metricName=CloudTrailStopped,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "ALERT-CloudTrail-Stopped" \
  --metric-name CloudTrailStopped \
  --namespace SecurityAlerts \
  --statistic Sum --period 300 \
  --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

# Alarm 2: Root account used
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name "RootAccountUsed" \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations metricName=RootAccountUsed,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "ALERT-Root-Account-Used" \
  --metric-name RootAccountUsed \
  --namespace SecurityAlerts \
  --statistic Sum --period 60 \
  --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

# Alarm 3: IAM Policy deleted
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name "IAMPolicyDeleted" \
  --filter-pattern '{ ($.eventName = DeleteGroupPolicy) || ($.eventName = DeleteRolePolicy) || ($.eventName = DeleteUserPolicy) || ($.eventName = DetachGroupPolicy) || ($.eventName = DetachRolePolicy) || ($.eventName = DetachUserPolicy) }' \
  --metric-transformations metricName=IAMPolicyDeleted,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "ALERT-IAM-Policy-Deleted" \
  --metric-name IAMPolicyDeleted \
  --namespace SecurityAlerts \
  --statistic Sum --period 300 \
  --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

# Alarm 4: MFA disabled on any user
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name "MFADisabled" \
  --filter-pattern '{ ($.eventName = DeleteVirtualMFADevice) || ($.eventName = DeactivateMFADevice) }' \
  --metric-transformations metricName=MFADisabled,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "ALERT-MFA-Disabled" \
  --metric-name MFADisabled \
  --namespace SecurityAlerts \
  --statistic Sum --period 300 \
  --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

echo "✅ 4 security alarms active — instant email alerts on critical events"
```

---

## LAB 24 — Access Key Rotation (90-Day Rule)

> Old keys = biggest security risk. Real teams rotate every 90 days.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Users** → click `priya-backend` → **Security credentials** tab
2. Under **Access keys** → click **Create access key**
3. Copy the new key and secret → give to Priya to update her CLI (`aws configure`)
4. After Priya confirms new key works (24 hours later):
   - Go back to her security credentials → find the OLD key → **Actions** → **Deactivate**
   - Test for another day → then **Actions** → **Delete**

### 💻 CLI Commands

```bash
# Check age of all access keys
aws iam generate-credential-report
sleep 5
aws iam get-credential-report --query 'Content' --output text | \
  base64 --decode | \
  awk -F',' 'NR==1 || $10 != "N/A" {print $1, $10}' | \
  column -t
# Shows: username and access_key_1_last_rotated date

# Create new key for Priya
aws iam create-access-key --user-name priya-backend
# → Give Priya the new keys → she runs: aws configure

# Find old key ID
aws iam list-access-keys --user-name priya-backend

# Deactivate old key first (test for 24hrs before deleting)
aws iam update-access-key \
  --user-name priya-backend \
  --access-key-id AKIAOLDKEYID12345 \
  --status Inactive

# After confirming nothing broke — delete the old key
aws iam delete-access-key \
  --user-name priya-backend \
  --access-key-id AKIAOLDKEYID12345

echo "✅ Key rotated — Priya now has a fresh access key"
```

---

## LAB 25 — Quarterly Access Review

> Run every 90 days. Review who has access to what, remove stale permissions.

### 🖥️ Console UI Steps

1. Go to **IAM** → **Credential report** → **Download credential report**
2. Open in Excel/Google Sheets → filter `mfa_active = false` → fix those accounts
3. For each user → check **Access Advisor** tab → identify unused services
4. For each group → **Permissions** tab → review attached policies
5. Remove users from groups if they no longer need access
6. Check **IAM Access Analyzer → Findings** → resolve any external access risks

### 💻 CLI Commands

```bash
echo "=========================================="
echo "   TECHNOVA QUARTERLY IAM ACCESS REVIEW"
echo "   Date: $(date)"
echo "=========================================="

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "--- ALL USERS ---"
aws iam list-users \
  --query 'Users[*].{User:UserName,Created:CreateDate}' \
  --output table

echo "--- USERS WITH NO MFA (Fix These!) ---"
aws iam generate-credential-report > /dev/null 2>&1
sleep 3
aws iam get-credential-report --query 'Content' --output text | \
  base64 -d | awk -F',' 'NR>1 && $8=="false" {print "⚠️  NO MFA: "$1}'

echo "--- USERS WHO NEVER LOGGED IN (Consider Removing) ---"
aws iam get-credential-report --query 'Content' --output text | \
  base64 -d | awk -F',' 'NR>1 && $5=="N/A" {print "❓ NEVER LOGGED IN: "$1}'

echo "--- ALL CUSTOM POLICIES ---"
aws iam list-policies --scope Local \
  --query 'Policies[*].{Policy:PolicyName,Updated:UpdateDate}' \
  --output table

echo "--- GROUP MEMBERSHIPS ---"
for GROUP in grp-admins grp-backend-devs grp-frontend-devs grp-data-engineers grp-devops grp-qa; do
  echo "--- $GROUP ---"
  aws iam get-group --group-name $GROUP \
    --query 'Users[].UserName' --output table
done

echo "✅ Review complete. Take action on all ⚠️ and ❓ items above."
```

---

## ✅ Final Verification — Everything Working

### 🖥️ Console UI Verification

1. **IAM → Users** → verify 9 users exist
2. **IAM → User groups** → verify 6 groups with correct members
3. **IAM → Roles** → filter by `role-` prefix → verify 3-4 roles exist
4. **IAM → Policies → Customer managed** → verify 9 custom policies
5. **CloudTrail → Trails** → verify `technova-audit-trail` shows Logging: On
6. **IAM → Access Analyzer** → verify analyzer is Active with no unresolved findings
7. **Secrets Manager** → verify `technova/prod/db-password` exists

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "=== USERS ==="
aws iam list-users --query 'Users[].UserName' --output table

echo "=== GROUPS & MEMBERS ==="
for GROUP in grp-admins grp-backend-devs grp-frontend-devs \
             grp-data-engineers grp-devops grp-qa; do
  echo "--- $GROUP ---"
  aws iam get-group --group-name $GROUP \
    --query 'Users[].UserName' --output table
done

echo "=== ROLES ==="
aws iam list-roles \
  --query 'Roles[?starts_with(RoleName,`role-`)].RoleName' \
  --output table

echo "=== CUSTOM POLICIES ==="
aws iam list-policies --scope Local \
  --query 'Policies[].PolicyName' --output table

echo "=== CLOUDTRAIL STATUS ==="
aws cloudtrail get-trail-status --name technova-audit-trail \
  --query '{Logging:IsLogging,LastDelivery:LatestDeliveryTime}'

echo "=== ACCESS ANALYZER ==="
aws accessanalyzer list-analyzers \
  --query 'analyzers[].{Name:name,Status:status}' --output table
```

---

## 🗺️ Complete Architecture — What You Built

```
TechNova AWS Account (Production-Ready)
│
├── 🔒 Security Foundation
│   ├── Root MFA enabled + locked away, no root access keys
│   ├── Password policy: 12 chars, symbols, 90-day expiry
│   ├── CloudTrail: Every API call logged to S3 (multi-region)
│   ├── IAM Access Analyzer: Watching for external exposure
│   └── CloudWatch alarms: 4 critical security events monitored
│
├── 👤 Users (9 total)
│   ├── arjun-admin      → grp-admins
│   ├── priya-backend    → grp-backend-devs (+ permission boundary)
│   ├── rahul-backend    → grp-backend-devs (+ permission boundary)
│   ├── sneha-frontend   → grp-frontend-devs
│   ├── karthik-frontend → grp-frontend-devs
│   ├── divya-data       → grp-data-engineers
│   ├── anand-devops     → grp-devops (+ permission boundary)
│   ├── meena-qa         → grp-qa
│   └── ravi-finance     → Billing policy only (no group)
│
├── 👥 Groups (6 total — all have MFA enforcement policy)
│   ├── grp-admins          → AdministratorAccess
│   ├── grp-backend-devs    → policy-backend-dev + policy-require-mfa + policy-ip-restriction
│   ├── grp-frontend-devs   → policy-frontend-dev + policy-require-mfa + policy-ip-restriction
│   ├── grp-data-engineers  → policy-data-engineer + policy-require-mfa + policy-ip-restriction
│   ├── grp-devops          → policy-devops + policy-require-mfa
│   └── grp-qa              → policy-qa + policy-require-mfa + policy-ip-restriction
│
├── 📋 Custom Policies (9 total)
│   ├── policy-backend-dev
│   ├── policy-frontend-dev
│   ├── policy-data-engineer
│   ├── policy-devops
│   ├── policy-qa
│   ├── policy-require-mfa
│   ├── policy-ip-restriction
│   ├── policy-backend-tag-restricted
│   └── boundary-devops (permission boundary)
│
├── 🎭 Roles (4 total)
│   ├── role-ec2-s3-access      → EC2 instances (via instance profile)
│   ├── role-lambda-dynamodb    → Lambda functions
│   ├── role-ec2-cloudwatch     → EC2 log shipping
│   └── role-github-actions     → GitHub CI/CD via OIDC (no keys!)
│
├── 🗄️ Secrets
│   ├── technova/prod/db-password → in Secrets Manager
│   └── technova/prod/stripe-api-key → in Secrets Manager
│
└── 🏗️ Operations
    ├── Credential Report: Monthly audit of all users
    ├── Access Advisor: Quarterly unused permission cleanup
    ├── Offboarding SOP: Step-by-step when someone leaves
    ├── Key Rotation: Every 90 days with verification
    └── Break-glass: Emergency account locked + alarmed
```

---

## 🧹 Cleanup (Delete Everything After Lab)

### 🖥️ Console UI Steps

1. **IAM → Users** → select all lab users → **Delete** (requires typing "delete" to confirm)
2. **IAM → User groups** → select all groups → **Delete**
3. **IAM → Roles** → select all `role-` prefixed roles → **Delete**
4. **IAM → Policies → Customer managed** → select all custom policies → **Actions → Delete**
5. **CloudTrail → Trails** → select `technova-audit-trail` → **Delete**
6. **IAM → Access Analyzer** → delete `technova-access-analyzer`
7. **S3** → delete the CloudTrail logs bucket (first empty it, then delete)

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
USERS=(priya-backend rahul-backend sneha-frontend karthik-frontend \
       divya-data anand-devops meena-qa ravi-finance)
GROUPS=(grp-admins grp-backend-devs grp-frontend-devs \
        grp-data-engineers grp-devops grp-qa)
ROLES=(role-ec2-s3-access role-lambda-dynamodb role-ec2-cloudwatch role-github-actions)
CUSTOM_POLICIES=(policy-backend-dev policy-frontend-dev policy-data-engineer
                 policy-devops policy-qa policy-require-mfa policy-ip-restriction
                 policy-backend-tag-restricted boundary-devops policy-read-db-secret)

# Detach all group policies
for GROUP in "${GROUPS[@]}"; do
  POLICIES=$(aws iam list-attached-group-policies --group-name $GROUP \
    --query 'AttachedPolicies[].PolicyArn' --output text)
  for POLICY in $POLICIES; do
    aws iam detach-group-policy --group-name $GROUP --policy-arn $POLICY
  done
done

# Clean and delete all users
for USER in "${USERS[@]}"; do
  aws iam delete-user-permissions-boundary --user-name $USER 2>/dev/null
  for GROUP in $(aws iam list-groups-for-user --user-name $USER \
    --query 'Groups[].GroupName' --output text); do
    aws iam remove-user-from-group --group-name $GROUP --user-name $USER
  done
  for POLICY in $(aws iam list-attached-user-policies --user-name $USER \
    --query 'AttachedPolicies[].PolicyArn' --output text); do
    aws iam detach-user-policy --user-name $USER --policy-arn $POLICY
  done
  for KEY in $(aws iam list-access-keys --user-name $USER \
    --query 'AccessKeyMetadata[].AccessKeyId' --output text); do
    aws iam delete-access-key --user-name $USER --access-key-id $KEY
  done
  aws iam delete-login-profile --user-name $USER 2>/dev/null
  aws iam delete-user --user-name $USER
  echo "Deleted user: $USER"
done

# Delete groups
for GROUP in "${GROUPS[@]}"; do
  aws iam delete-group --group-name $GROUP
  echo "Deleted group: $GROUP"
done

# Delete roles
for ROLE in "${ROLES[@]}"; do
  for POLICY in $(aws iam list-attached-role-policies --role-name $ROLE \
    --query 'AttachedPolicies[].PolicyArn' --output text); do
    aws iam detach-role-policy --role-name $ROLE --policy-arn $POLICY
  done
  aws iam remove-role-from-instance-profile \
    --instance-profile-name profile-ec2-s3-access \
    --role-name $ROLE 2>/dev/null
  aws iam delete-role --role-name $ROLE 2>/dev/null
  echo "Deleted role: $ROLE"
done

# Delete instance profile
aws iam delete-instance-profile \
  --instance-profile-name profile-ec2-s3-access 2>/dev/null

# Delete custom policies
for POLICY in "${CUSTOM_POLICIES[@]}"; do
  aws iam delete-policy \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/$POLICY 2>/dev/null
  echo "Deleted policy: $POLICY"
done

# Stop CloudTrail and Access Analyzer
aws cloudtrail stop-logging --name technova-audit-trail 2>/dev/null
aws cloudtrail delete-trail --name technova-audit-trail 2>/dev/null
aws accessanalyzer delete-analyzer \
  --analyzer-name technova-access-analyzer 2>/dev/null

echo "✅ Full cleanup complete. All IAM resources removed."
```

---

## 📋 Production IAM Security Checklist

```
Foundation:
 ✅ Root MFA enabled, no root access keys, root never used daily
 ✅ CloudTrail enabled in ALL regions (multi-region trail)
 ✅ CloudTrail logs protected in S3
 ✅ IAM Access Analyzer enabled and findings reviewed
 ✅ CloudWatch alarms for 4 critical IAM events

Users & Groups:
 ✅ No shared user accounts — one user per person
 ✅ All users in groups — no direct policy attachments (except billing)
 ✅ MFA enforced via policy on all groups
 ✅ Credential report reviewed monthly
 ✅ Unused credentials (90+ days) deactivated

Permissions:
 ✅ Least-privilege applied — no Action:* or Resource:* without reason
 ✅ Permission boundaries on all non-admin users
 ✅ Access Advisor reviewed quarterly — unused services removed
 ✅ Resource-based policies (S3 bucket policies) in place
 ✅ IP restriction policy on non-admin groups

Keys & Secrets:
 ✅ Access keys rotated every 90 days
 ✅ No access keys for services — use IAM Roles
 ✅ No keys in GitHub repos, .env files, or source code
 ✅ DB passwords and API keys in Secrets Manager

Roles:
 ✅ EC2 uses instance profiles (not access keys)
 ✅ Lambda uses execution roles (not access keys)
 ✅ GitHub Actions uses OIDC (not access keys)

Monitoring & Response:
 ✅ CloudWatch alarm: Root account used
 ✅ CloudWatch alarm: CloudTrail stopped
 ✅ CloudWatch alarm: IAM policy deleted
 ✅ CloudWatch alarm: MFA disabled
 ✅ Break-glass account exists, locked, and monitored
 ✅ Security team email subscribed to SNS alerts
 ✅ Offboarding checklist followed same day employee leaves
 ✅ Quarterly access review scheduled
```

---

## ⚡ Quick Reference — CLI Cheat Sheet

```bash
# IAM User commands
aws iam create-user --user-name USERNAME
aws iam list-users
aws iam delete-user --user-name USERNAME
aws iam attach-user-policy --user-name USERNAME --policy-arn POLICY_ARN
aws iam list-attached-user-policies --user-name USERNAME

# IAM Group commands
aws iam create-group --group-name GROUPNAME
aws iam add-user-to-group --group-name GROUPNAME --user-name USERNAME
aws iam attach-group-policy --group-name GROUPNAME --policy-arn POLICY_ARN
aws iam list-groups-for-user --user-name USERNAME

# IAM Role commands
aws iam create-role --role-name ROLENAME --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name ROLENAME --policy-arn POLICY_ARN
aws iam list-roles

# IAM Policy commands
aws iam create-policy --policy-name POLICYNAME --policy-document file://policy.json
aws iam list-policies --scope Local   # Your custom policies

# Simulate what a user can do
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::ACCOUNT_ID:user/USERNAME \
  --action-names s3:GetObject \
  --resource-arns "arn:aws:s3:::my-bucket/*"

# Check who you are
aws iam get-user
aws sts get-caller-identity
```

---

## 📚 Key ARN Format

```
arn:aws:iam::123456789012:user/john-dev
arn:aws:iam::123456789012:group/Developers
arn:aws:iam::123456789012:role/EC2-S3-Reader-Role
arn:aws:iam::123456789012:policy/MyCustomPolicy
arn:aws:s3:::my-bucket
arn:aws:ec2:ap-south-1:123456789012:instance/i-0abc123

Format: arn:partition:service:region:account-id:resource
```

---

## 🎯 Learning Resources

| Resource | Link | Cost |
|----------|------|------|
| AWS IAM Documentation | docs.aws.amazon.com/IAM | Free |
| AWS Skill Builder — IAM Course | skillbuilder.aws | Free |
| AWS Free Tier (Practice) | aws.amazon.com/free | Free |
| IAM Policy Simulator | policysim.aws.amazon.com | Free |
| AWS Well-Architected — Security Pillar | aws.amazon.com/architecture | Free |
| Stephane Maarek's AWS Course | Udemy | Paid |

---

## 🔑 Summary

```
IAM = Who are you? + What can you do?
│
├── Users     → People with long-term credentials
├── Groups    → Organize users, attach policies here
├── Roles     → Temporary access for services & apps
├── Policies  → JSON rules (Allow/Deny actions on resources)
└── MFA       → Second factor for extra security

Golden Rules:
  1. Never use Root for daily work
  2. Always enable MFA on every user
  3. Grant least-privilege (minimum needed permissions)
  4. Use Roles instead of access keys for EC2/Lambda
  5. Review permissions every 90 days
  6. Store secrets in Secrets Manager, never in code
  7. Audit with CloudTrail + Access Analyzer
  8. Rotate access keys every 90 days
```

---

*Lab complete! You've built a production-style IAM setup for a 10-person startup from scratch.
This is exactly how a real startup's AWS IAM is built and maintained. 🚀*
