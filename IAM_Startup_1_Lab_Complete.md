# 🚀 AWS IAM Lab — Real Startup Project (Complete Edition)
### Hands-On: Build Production-Grade IAM from Scratch for a 10-Person Tech Startup

---

## 🏢 Startup Scenario

**Company:** `TechNova Pvt Ltd` — A SaaS product startup  
**Team Size:** 10 People  
**Product:** A web application hosted on AWS  
**Your Role:** You are the AWS Admin / DevOps Engineer

---

## 👥 Team & Roles (Who Needs What)

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

## 🗺️ Full Plan — Before You Start

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

Real-World Additions (What most guides skip):
├── CloudTrail              → Audit log every single IAM action
├── IAM Access Analyzer     → Detect open/risky permissions
├── Permission Boundaries   → Stop DevOps from escalating own privileges
├── Credential Report       → Full security audit of all users
├── Secrets Manager         → Store DB passwords, API keys (not hardcoded)
├── Tag-Based Access        → Devs only touch resources tagged with their env
├── IP Restriction Policy   → Only allow access from office IP
├── Offboarding Procedure   → What to do when someone leaves
└── Quarterly Access Review → Scheduled security cleanup
```

---

## ⚙️ Prerequisites

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

> ⚠️ This is the most important step. Root = unlimited power with no restrictions.

**Via Console (browser only — cannot be done via CLI):**
1. Login → `console.aws.amazon.com` with root email + password
2. Top-right → your account name → **Security credentials**
3. **Multi-factor authentication** → **Assign MFA device**
4. Choose **Authenticator app** → Next
5. Open **Google Authenticator** on phone → Scan QR code
6. Enter 2 consecutive 6-digit codes → **Add MFA**
7. **Access keys** section → confirm none exist → delete if any found
8. **Activate IAM billing access** → Account → IAM user/role access to Billing → Activate
   *(This lets Ravi view billing without using root)*

```
Root Account Checklist:
  ✅ MFA enabled
  ✅ No access keys exist
  ✅ Billing access activated for IAM users
  ✅ Log out and never use root again
```

---

## LAB 2 — Create the Admin User (Arjun)

> All daily work is done as Arjun from here. Root stays locked away.

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

# Switch CLI to use Arjun's credentials
aws configure
# Enter Arjun's keys now

# Confirm you are now Arjun
aws iam get-user
```

> **Enable MFA for Arjun via Console** — same steps as root above.

---

## LAB 3 — Create All Groups

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

> AWS managed policies are too broad. We write exact permissions needed.

### Policy 1: Backend Developer
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
```

### Policy 2: Frontend Developer
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

### Policy 3: Data Engineer
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

### Policy 4: DevOps Engineer
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

### Policy 5: QA Engineer
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

### Role 1: EC2 → S3 Access
```bash
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

aws iam create-role \
  --role-name role-ec2-s3-access \
  --assume-role-policy-document file://trust-ec2.json

aws iam attach-role-policy \
  --role-name role-ec2-s3-access \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess

echo "✅ EC2-S3 role created"
```

### Role 2: Lambda → DynamoDB
```bash
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
```

### Role 3: EC2 → CloudWatch Logs
```bash
aws iam create-role \
  --role-name role-ec2-cloudwatch \
  --assume-role-policy-document file://trust-ec2.json

aws iam attach-role-policy \
  --role-name role-ec2-cloudwatch \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

echo "✅ EC2-CloudWatch role created"
```

### ✅ Attach Role to EC2 Instance (Instance Profile)
> This is how EC2 actually uses the role — via an Instance Profile.

```bash
# Create instance profile
aws iam create-instance-profile \
  --instance-profile-name profile-ec2-s3-access

# Add role to instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name profile-ec2-s3-access \
  --role-name role-ec2-s3-access

# Attach profile to a running EC2 instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-0abc1234def567890 \
  --iam-instance-profile Name=profile-ec2-s3-access

# Verify — SSH into EC2 and run:
# aws s3 ls   ← should work WITHOUT any aws configure keys
```

---

## LAB 8 — Enforce MFA for All Users

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

echo "✅ Password policy applied"
```

---

## LAB 10 — Test Permissions (Policy Simulator)

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

# ✅ Should PASS — Priya can describe EC2
aws ec2 describe-instances --profile priya

# ❌ Should FAIL — Sneha cannot touch backend bucket
aws configure --profile sneha  # use sneha-frontend keys
aws s3 ls s3://technova-backend-data --profile sneha

# ✅ Should PASS — Sneha's own bucket
aws s3 ls s3://technova-frontend --profile sneha
```

**Test via Console — IAM Policy Simulator:**
```
URL: https://policysim.aws.amazon.com
Steps:
  1. Select user: priya-backend
  2. Service: EC2
  3. Action: TerminateInstances
  4. Run Simulation → Expected: DENIED ✅
```

---

## LAB 11 — Enable CloudTrail (Audit Every Action)

> 🔥 Critical for production. Every API call, every login, every IAM change is logged.
> Without this, you have zero visibility into what happened if something goes wrong.

```bash
# Create S3 bucket to store logs
aws s3 mb s3://technova-cloudtrail-logs-${ACCOUNT_ID} \
  --region ap-south-1

# Set bucket policy to allow CloudTrail to write
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
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket technova-cloudtrail-logs-${ACCOUNT_ID} \
  --policy file://cloudtrail-bucket-policy.json

# Create the trail
aws cloudtrail create-trail \
  --name technova-audit-trail \
  --s3-bucket-name technova-cloudtrail-logs-${ACCOUNT_ID} \
  --include-global-service-events \
  --is-multi-region-trail

# Start logging
aws cloudtrail start-logging \
  --name technova-audit-trail

# Verify it is active
aws cloudtrail get-trail-status --name technova-audit-trail

echo "✅ CloudTrail logging every API call to S3"
```

**What CloudTrail captures:**
```
- Who logged in and when
- Who created/deleted/modified IAM users
- Who changed a policy
- Who accessed which S3 bucket
- Who launched or stopped EC2
- Failed login attempts
- Every single AWS API call
```

---

## LAB 12 — Enable IAM Access Analyzer

> Finds policies that give access to the outside world accidentally.
> For example: an S3 bucket policy accidentally allowing public access.

```bash
# Enable Access Analyzer for the account
aws accessanalyzer create-analyzer \
  --analyzer-name technova-access-analyzer \
  --type ACCOUNT

# List findings (things that are open to external access)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-south-1:${ACCOUNT_ID}:analyzer/technova-access-analyzer

echo "✅ Access Analyzer watching for external access issues"
```

**Via Console:**
```
IAM → Access Analyzer → Create analyzer → type: Account
Then review "Findings" — any finding means something is exposed externally
Fix it or archive it with a reason
```

---

## LAB 13 — Permission Boundaries (Stop Privilege Escalation)

> Real problem: Anand (DevOps) has IAM read access. But what if he tries to create
> a new IAM role with admin access for himself? Permission Boundaries block this.
>
> Rule: DevOps can create roles BUT those roles can never exceed this boundary.

```bash
cat > boundary-devops.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServices",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "ecs:*",
        "ecr:*",
        "cloudwatch:*",
        "logs:*",
        "s3:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyIAMWrite",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:DeleteUser",
        "iam:AttachUserPolicy",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:CreatePolicy",
        "iam:DeletePolicy",
        "iam:PutUserPermissionsBoundary",
        "iam:PutRolePermissionsBoundary"
      ],
      "Resource": "*"
    }
  ]
}
EOF

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-policy \
  --policy-name boundary-devops \
  --policy-document file://boundary-devops.json

# Apply boundary to Anand
aws iam put-user-permissions-boundary \
  --user-name anand-devops \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/boundary-devops

echo "✅ Anand cannot escalate his own privileges now"
```

---

## LAB 14 — GitHub Actions OIDC Role (CI/CD Without Access Keys)

> Real-world CI/CD never uses hardcoded AWS access keys in GitHub secrets.
> Instead, GitHub gets a temporary role via OIDC trust. Safer, no key rotation needed.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Step 1: Add GitHub as OIDC identity provider
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

# Step 2: Create trust policy for GitHub repo
# Replace YOUR-ORG and YOUR-REPO with your actual GitHub org and repo name
cat > trust-github.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:YOUR-ORG/YOUR-REPO:*"
      }
    }
  }]
}
EOF

# Replace ACCOUNT_ID placeholder
sed -i "s/ACCOUNT_ID/${ACCOUNT_ID}/g" trust-github.json

# Step 3: Create the role
aws iam create-role \
  --role-name role-github-actions \
  --assume-role-policy-document file://trust-github.json

# Step 4: Attach deploy permissions (only what CI/CD needs)
aws iam attach-role-policy \
  --role-name role-github-actions \
  --policy-arn arn:aws:iam::aws:policy/AmazonECR-FullAccess

aws iam attach-role-policy \
  --role-name role-github-actions \
  --policy-arn arn:aws:iam::aws:policy/AmazonECS_FullAccess

echo "✅ GitHub Actions role ready — no hardcoded keys needed"
```

**In your GitHub Actions workflow `.github/workflows/deploy.yml`:**
```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::ACCOUNT_ID:role/role-github-actions
          aws-region: ap-south-1
```

---

## LAB 15 — Tag-Based Access Control

> Real startups have dev, staging, and prod environments.
> Tag-based policies ensure backend devs can only touch dev resources.

```bash
# Tag your EC2 instances when launching them:
# Key: Environment  Value: dev
# Key: Team         Value: backend

# Policy: Backend devs can only start/stop EC2 tagged Environment=dev
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
      "Sid": "StartStopOnlyDevInstances",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Environment": "dev",
          "ec2:ResourceTag/Team": "backend"
        }
      }
    }
  ]
}
EOF

echo "✅ Backend devs are now restricted to dev-tagged EC2 instances only"
echo "   They cannot touch staging or prod instances even if they try"
```

---

## LAB 16 — Restrict Access to Office IP Only

> In real companies, AWS console and API calls should only come from the office network.
> This condition blocks access if someone tries from home or a coffee shop.

```bash
# Replace 203.0.113.50 with your actual office public IP
# Find your IP: curl ifconfig.me

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

# Attach to all regular user groups (not admins)
for GROUP in grp-backend-devs grp-frontend-devs grp-data-engineers grp-qa; do
  aws iam attach-group-policy \
    --group-name $GROUP \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-ip-restriction
done

echo "✅ Non-admin users can only access AWS from the office IP"
```

---

## LAB 17 — Secrets Manager (Never Hardcode Passwords)

> Real-world apps never put DB passwords or API keys in code or environment variables.
> AWS Secrets Manager stores and rotates them automatically.

```bash
# Store the RDS database password
aws secretsmanager create-secret \
  --name "technova/prod/db-password" \
  --description "TechNova production RDS password" \
  --secret-string '{"username":"admin","password":"SuperSecret@RDS123!"}'

# Store a third-party API key
aws secretsmanager create-secret \
  --name "technova/prod/stripe-api-key" \
  --description "Stripe payment API key" \
  --secret-string '{"api_key":"sk_live_abc123xyz"}'

# List all secrets
aws secretsmanager list-secrets \
  --query 'SecretList[].Name' --output table

echo "✅ Secrets stored in Secrets Manager"
echo "   In your application code, fetch like this:"
echo "   aws secretsmanager get-secret-value --secret-id technova/prod/db-password"
```

**In Python application (backend dev uses this):**
```python
import boto3
import json

client = boto3.client('secretsmanager', region_name='ap-south-1')

# Fetch secret — no hardcoded password in code!
response = client.get_secret_value(SecretId='technova/prod/db-password')
secret = json.loads(response['SecretString'])

db_user = secret['username']
db_pass = secret['password']
```

---

## LAB 18 — Generate IAM Credential Report (Security Audit)

> Run this monthly. Shows every user, their last login, key age, MFA status.
> If any key is older than 90 days or MFA is missing, fix it immediately.

```bash
# Generate report
aws iam generate-credential-report

# Wait a moment, then download and read it
aws iam get-credential-report \
  --query 'Content' --output text | base64 -d > credential-report.csv

# View it
cat credential-report.csv

# Look for problems — check these columns:
# - password_last_used     → if never, user never logged in (remove them)
# - mfa_active             → must be true for all users
# - access_key_1_last_used → if older than 90 days, rotate or delete
# - access_key_1_active    → if true but unused, deactivate it

echo "✅ Credential report saved to credential-report.csv"
echo "   Review it and fix any user with mfa_active=false or old keys"
```

---

## LAB 19 — Employee Offboarding (When Someone Leaves)

> This is critical. When a developer leaves the company, their access must be
> removed immediately — the same day. Forgotten accounts are a top security risk.

```bash
# Example: Rahul resigned today. Here is the exact procedure.
DEPARTED_USER="rahul-backend"

echo "🚨 Starting offboarding for: $DEPARTED_USER"

# Step 1: Disable their console login IMMEDIATELY (first thing you do)
aws iam update-login-profile \
  --user-name $DEPARTED_USER \
  --password "DISABLED-$(date +%s)" \
  --password-reset-required
echo "Step 1 ✅ Console login disabled"

# Step 2: Deactivate all access keys (CLI/SDK access blocked)
KEYS=$(aws iam list-access-keys --user-name $DEPARTED_USER \
  --query 'AccessKeyMetadata[].AccessKeyId' --output text)
for KEY in $KEYS; do
  aws iam update-access-key \
    --user-name $DEPARTED_USER \
    --access-key-id $KEY \
    --status Inactive
  echo "Step 2 ✅ Key deactivated: $KEY"
done

# Step 3: Remove from all groups (revoke all permissions)
GROUPS=$(aws iam list-groups-for-user --user-name $DEPARTED_USER \
  --query 'Groups[].GroupName' --output text)
for GROUP in $GROUPS; do
  aws iam remove-user-from-group \
    --group-name $GROUP \
    --user-name $DEPARTED_USER
  echo "Step 3 ✅ Removed from group: $GROUP"
done

# Step 4: Revoke any active sessions (invalidate existing tokens)
aws iam attach-user-policy \
  --user-name $DEPARTED_USER \
  --policy-arn arn:aws:iam::aws:policy/AWSDenyAll
echo "Step 4 ✅ DenyAll policy attached — all sessions revoked"

# Step 5: Delete access keys permanently
for KEY in $KEYS; do
  aws iam delete-access-key \
    --user-name $DEPARTED_USER \
    --access-key-id $KEY
done
echo "Step 5 ✅ Access keys deleted"

# Step 6: Delete the user entirely after 7 days review period
# aws iam delete-login-profile --user-name $DEPARTED_USER
# aws iam delete-user --user-name $DEPARTED_USER
echo "Step 6 ⏳ Schedule user deletion after 7-day review: $DEPARTED_USER"

echo ""
echo "✅ Offboarding complete for $DEPARTED_USER"
echo "   Notify: IT, HR, Project Manager"
echo "   Rotate: Any shared secrets or passwords Rahul may have known"
```

---

## LAB 20 — Quarterly Access Review

> Real companies do this every 90 days. Review who has access to what,
> remove people who don't need it anymore, and rotate old keys.

```bash
echo "=========================================="
echo "   TECHNOVA QUARTERLY IAM ACCESS REVIEW"
echo "   Date: $(date)"
echo "=========================================="

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo ""
echo "--- ALL USERS ---"
aws iam list-users \
  --query 'Users[*].{User:UserName,Created:CreateDate}' \
  --output table

echo ""
echo "--- USERS WITH NO MFA (Fix These!) ---"
aws iam generate-credential-report > /dev/null 2>&1
sleep 3
aws iam get-credential-report --query 'Content' --output text | \
  base64 -d | awk -F',' 'NR>1 && $8=="false" {print "⚠️  NO MFA: "$1}'

echo ""
echo "--- ACCESS KEYS OLDER THAN 90 DAYS (Rotate These!) ---"
for USER in $(aws iam list-users --query 'Users[].UserName' --output text); do
  aws iam list-access-keys --user-name $USER \
    --query "AccessKeyMetadata[?Status=='Active'].{User:'$USER',KeyId:AccessKeyId,Created:CreateDate}" \
    --output text
done

echo ""
echo "--- USERS WHO NEVER LOGGED IN (Consider Removing) ---"
aws iam get-credential-report --query 'Content' --output text | \
  base64 -d | awk -F',' 'NR>1 && $5=="N/A" {print "❓ NEVER LOGGED IN: "$1}'

echo ""
echo "--- ALL CUSTOM POLICIES ---"
aws iam list-policies --scope Local \
  --query 'Policies[*].{Policy:PolicyName,Updated:UpdateDate}' \
  --output table

echo ""
echo "--- ALL ROLES ---"
aws iam list-roles \
  --query 'Roles[*].{Role:RoleName,Created:CreateDate}' \
  --output table

echo ""
echo "✅ Review complete. Take action on all ⚠️ and ❓ items above."
```

---

## ✅ Final Verification — Everything Working

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
TechNova AWS Account
│
├── 🔐 Security Foundation
│   ├── Root MFA enabled, no access keys, billing activated
│   ├── Password policy: 12 chars, symbols, 90-day expiry
│   ├── CloudTrail: Every API call logged to S3
│   └── IAM Access Analyzer: Watching for external exposure
│
├── 👤 Users (9 total)
│   ├── arjun-admin      → grp-admins
│   ├── priya-backend    → grp-backend-devs
│   ├── rahul-backend    → grp-backend-devs
│   ├── sneha-frontend   → grp-frontend-devs
│   ├── karthik-frontend → grp-frontend-devs
│   ├── divya-data       → grp-data-engineers
│   ├── anand-devops     → grp-devops (+ permission boundary)
│   ├── meena-qa         → grp-qa
│   └── ravi-finance     → Billing policy only
│
├── 👥 Groups (6 total — all have MFA + IP restriction policy)
│   ├── grp-admins          → AdministratorAccess
│   ├── grp-backend-devs    → policy-backend-dev
│   ├── grp-frontend-devs   → policy-frontend-dev
│   ├── grp-data-engineers  → policy-data-engineer
│   ├── grp-devops          → policy-devops
│   └── grp-qa              → policy-qa
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
└── 🏗️ Operations
    ├── Secrets Manager: DB passwords + API keys stored safely
    ├── Credential Report: Monthly audit of all users
    ├── Offboarding SOP: Step-by-step when someone leaves
    └── Quarterly Review: Script to find stale access & old keys
```

---

## 🧹 Cleanup After Lab

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
USERS=(priya-backend rahul-backend sneha-frontend karthik-frontend \
       divya-data anand-devops meena-qa ravi-finance)
GROUPS=(grp-admins grp-backend-devs grp-frontend-devs \
        grp-data-engineers grp-devops grp-qa)
ROLES=(role-ec2-s3-access role-lambda-dynamodb role-ec2-cloudwatch role-github-actions)
CUSTOM_POLICIES=(policy-backend-dev policy-frontend-dev policy-data-engineer
                 policy-devops policy-qa policy-require-mfa policy-ip-restriction
                 policy-backend-tag-restricted boundary-devops)

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
  # Remove permission boundaries
  aws iam delete-user-permissions-boundary --user-name $USER 2>/dev/null
  # Remove from groups
  for GROUP in $(aws iam list-groups-for-user --user-name $USER \
    --query 'Groups[].GroupName' --output text); do
    aws iam remove-user-from-group --group-name $GROUP --user-name $USER
  done
  # Detach direct policies
  for POLICY in $(aws iam list-attached-user-policies --user-name $USER \
    --query 'AttachedPolicies[].PolicyArn' --output text); do
    aws iam detach-user-policy --user-name $USER --policy-arn $POLICY
  done
  # Delete keys and profile
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

# Stop CloudTrail
aws cloudtrail stop-logging --name technova-audit-trail 2>/dev/null
aws cloudtrail delete-trail --name technova-audit-trail 2>/dev/null

# Delete Access Analyzer
aws accessanalyzer delete-analyzer \
  --analyzer-name technova-access-analyzer 2>/dev/null

echo ""
echo "✅ Full cleanup complete. All IAM resources removed."
```

---

## 📋 What Was Added vs Original Lab

| Lab | Topic | Why It Matters in Real World |
|-----|-------|------------------------------|
| 7 | Instance Profile for EC2 | Without this, EC2 role doesn't actually work |
| 11 | CloudTrail | Zero visibility without it — can't investigate incidents |
| 12 | IAM Access Analyzer | Catches accidental public exposure automatically |
| 13 | Permission Boundaries | Stops privilege escalation by DevOps or developers |
| 14 | GitHub Actions OIDC | CI/CD pipelines must never use hardcoded access keys |
| 15 | Tag-Based Access | Separate dev/staging/prod access for same team |
| 16 | IP Restriction | Blocks access from outside the office network |
| 17 | Secrets Manager | No DB passwords or API keys ever in code |
| 18 | Credential Report | Monthly audit — find weak accounts before hackers do |
| 19 | Offboarding SOP | Departed employee access must be revoked same day |
| 20 | Quarterly Review | Stale permissions are the #1 cause of breaches |

---

*This is exactly how a real startup's AWS IAM is built and maintained. Follow the labs in order and you'll have a production-ready, auditable, least-privilege IAM setup.* 🚀
