# 🚀 AWS IAM Lab — Real Startup Project
### Hands-On: Build IAM from Scratch for a 10-Person Tech Startup

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

## 🗂️ Groups & Policies Plan (Before You Start)

```
IAM Groups to Create:
│
├── grp-admins          → AdministratorAccess (Arjun)
├── grp-backend-devs    → EC2 + RDS + Lambda + S3 read (Priya, Rahul)
├── grp-frontend-devs   → S3 full (specific bucket) + CloudFront (Sneha, Karthik)
├── grp-data-engineers  → S3 full + RDS read + Glue + Athena (Divya)
├── grp-devops          → EC2 + ECS + ECR + CloudWatch + IAM read (Anand)
└── grp-qa              → EC2 read + S3 read + CloudWatch (Meena)

Billing Access:
└── Ravi → Billing policy attached directly (special case)

IAM Roles to Create:
├── role-ec2-s3-access       → For EC2 instances to read/write S3
├── role-lambda-dynamodb     → For Lambda to access DynamoDB
└── role-ec2-cloudwatch      → For EC2 to send logs to CloudWatch
```

---

## ⚙️ Prerequisites

```bash
# 1. AWS Account (Free Tier is fine for this lab)
# 2. AWS CLI installed
# 3. Root account MFA enabled (do this first!)

# Verify CLI is installed
aws --version

# Configure CLI with your root/admin credentials
aws configure
# Enter: Access Key, Secret Key, Region (ap-south-1), Output (json)

# Verify it works
aws iam get-user
```

---

## 🔬 LAB START

---

## LAB 1 — Secure the Root Account

> ⚠️ Do this FIRST before anything else. Root account = god mode.

**Via Console (must be done in browser):**
1. Login to `console.aws.amazon.com` with root email + password
2. Top right → click your account name → **Security credentials**
3. Under **Multi-factor authentication (MFA)** → click **Assign MFA device**
4. Choose **Authenticator app** → Next
5. Open Google Authenticator on your phone → Scan the QR code
6. Enter 2 consecutive 6-digit codes → **Add MFA**
7. Under **Access keys** → confirm there are none (if there are, delete them)

**Verify Root is Secure:**
```
✅ MFA enabled on root
✅ No access keys on root
✅ Log out of root
✅ Never use root again (except billing emergencies)
```

---

## LAB 2 — Create the Admin User (Arjun)

> From now on, all work is done as Arjun (the IAM admin user), not root.

```bash
# Step 1: Create Arjun's user
aws iam create-user --user-name arjun-admin

# Step 2: Create a console login password
aws iam create-login-profile \
  --user-name arjun-admin \
  --password "TechNova@2024!" \
  --password-reset-required

# Step 3: Give full admin access
aws iam attach-user-policy \
  --user-name arjun-admin \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Step 4: Create access keys for CLI
aws iam create-access-key --user-name arjun-admin
# SAVE the AccessKeyId and SecretAccessKey output — shown only once!

# Step 5: Reconfigure CLI to use Arjun's keys (not root)
aws configure
# Enter Arjun's new access keys

# Verify you are now Arjun
aws iam get-user
```

> Now enable MFA for Arjun via AWS Console just like root.

---

## LAB 3 — Create All Groups

```bash
# Create all 5 groups for the startup
aws iam create-group --group-name grp-admins
aws iam create-group --group-name grp-backend-devs
aws iam create-group --group-name grp-frontend-devs
aws iam create-group --group-name grp-data-engineers
aws iam create-group --group-name grp-devops
aws iam create-group --group-name grp-qa

# Verify groups created
aws iam list-groups
```

---

## LAB 4 — Create Custom Policies

> AWS managed policies are broad. We create specific ones for least-privilege.

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
        "lambda:ListFunctions"
      ],
      "Resource": "*"
    },
    {
      "Sid": "S3ReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
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
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-backend-dev \
  --policy-document file://policy-backend-dev.json
# SAVE the Policy ARN from output (looks like arn:aws:iam::123456789012:policy/policy-backend-dev)
```

### Policy 2: Frontend Developer Policy
```bash
cat > policy-frontend-dev.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3FrontendBucketAccess",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::technova-frontend",
        "arn:aws:s3:::technova-frontend/*"
      ]
    },
    {
      "Sid": "CloudFrontAccess",
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
      "Sid": "S3FullAccess",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::technova-data-*",
        "arn:aws:s3:::technova-data-*/*"
      ]
    },
    {
      "Sid": "GlueAccess",
      "Effect": "Allow",
      "Action": [
        "glue:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AthenaAccess",
      "Effect": "Allow",
      "Action": [
        "athena:StartQueryExecution",
        "athena:GetQueryResults",
        "athena:ListWorkGroups"
      ],
      "Resource": "*"
    },
    {
      "Sid": "RDSReadOnly",
      "Effect": "Allow",
      "Action": [
        "rds:DescribeDBInstances",
        "rds:DescribeDBSnapshots"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-data-engineer \
  --policy-document file://policy-data-engineer.json
```

### Policy 4: DevOps Policy
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
      "Sid": "ContainerAccess",
      "Effect": "Allow",
      "Action": [
        "ecs:*",
        "ecr:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchFull",
      "Effect": "Allow",
      "Action": "cloudwatch:*",
      "Resource": "*"
    },
    {
      "Sid": "IAMReadOnly",
      "Effect": "Allow",
      "Action": [
        "iam:Get*",
        "iam:List*"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-devops \
  --policy-document file://policy-devops.json
```

### Policy 5: QA Policy
```bash
cat > policy-qa.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOnlyAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "s3:GetObject",
        "s3:ListBucket",
        "cloudwatch:GetMetricData",
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
# ⚠️ Replace 123456789012 with YOUR actual AWS Account ID in all ARNs below
# Find your account ID: aws sts get-caller-identity --query Account --output text

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Your Account ID: $ACCOUNT_ID"

# grp-admins → AdministratorAccess (AWS managed)
aws iam attach-group-policy \
  --group-name grp-admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# grp-backend-devs → custom backend policy
aws iam attach-group-policy \
  --group-name grp-backend-devs \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-backend-dev

# grp-frontend-devs → custom frontend policy
aws iam attach-group-policy \
  --group-name grp-frontend-devs \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-frontend-dev

# grp-data-engineers → custom data policy
aws iam attach-group-policy \
  --group-name grp-data-engineers \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-data-engineer

# grp-devops → custom devops policy
aws iam attach-group-policy \
  --group-name grp-devops \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-devops

# grp-qa → custom qa policy
aws iam attach-group-policy \
  --group-name grp-qa \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-qa

echo "✅ All policies attached to groups"
```

---

## LAB 6 — Create All Users & Add to Groups

```bash
# ---------- Backend Devs ----------
aws iam create-user --user-name priya-backend
aws iam create-user --user-name rahul-backend

aws iam create-login-profile --user-name priya-backend \
  --password "TechNova@Priya1!" --password-reset-required
aws iam create-login-profile --user-name rahul-backend \
  --password "TechNova@Rahul1!" --password-reset-required

aws iam add-user-to-group --group-name grp-backend-devs --user-name priya-backend
aws iam add-user-to-group --group-name grp-backend-devs --user-name rahul-backend

# ---------- Frontend Devs ----------
aws iam create-user --user-name sneha-frontend
aws iam create-user --user-name karthik-frontend

aws iam create-login-profile --user-name sneha-frontend \
  --password "TechNova@Sneha1!" --password-reset-required
aws iam create-login-profile --user-name karthik-frontend \
  --password "TechNova@Karthik1!" --password-reset-required

aws iam add-user-to-group --group-name grp-frontend-devs --user-name sneha-frontend
aws iam add-user-to-group --group-name grp-frontend-devs --user-name karthik-frontend

# ---------- Data Engineer ----------
aws iam create-user --user-name divya-data
aws iam create-login-profile --user-name divya-data \
  --password "TechNova@Divya1!" --password-reset-required
aws iam add-user-to-group --group-name grp-data-engineers --user-name divya-data

# ---------- DevOps ----------
aws iam create-user --user-name anand-devops
aws iam create-login-profile --user-name anand-devops \
  --password "TechNova@Anand1!" --password-reset-required
aws iam add-user-to-group --group-name grp-devops --user-name anand-devops

# ---------- QA ----------
aws iam create-user --user-name meena-qa
aws iam create-login-profile --user-name meena-qa \
  --password "TechNova@Meena1!" --password-reset-required
aws iam add-user-to-group --group-name grp-qa --user-name meena-qa

# ---------- Finance (Ravi — Billing only, no group) ----------
aws iam create-user --user-name ravi-finance
aws iam create-login-profile --user-name ravi-finance \
  --password "TechNova@Ravi1!" --password-reset-required

# Billing access must be enabled from root account console:
# Root login → Account → IAM user and role access to Billing → Activate
aws iam attach-user-policy \
  --user-name ravi-finance \
  --policy-arn arn:aws:iam::aws:policy/job-function/Billing

echo "✅ All 9 users created and grouped"
```

---

## LAB 7 — Create IAM Roles for AWS Services

### Role 1: EC2 can read/write S3

```bash
# Trust policy — who can assume this role (EC2 service)
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

echo "✅ EC2 S3 Role created"
```

### Role 2: Lambda can access DynamoDB

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

echo "✅ Lambda DynamoDB Role created"
```

### Role 3: EC2 sends logs to CloudWatch

```bash
aws iam create-role \
  --role-name role-ec2-cloudwatch \
  --assume-role-policy-document file://trust-ec2.json

aws iam attach-role-policy \
  --role-name role-ec2-cloudwatch \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

echo "✅ EC2 CloudWatch Role created"
```

---

## LAB 8 — Test & Verify Permissions

### Test 1: Verify Priya (backend dev) CANNOT delete EC2

```bash
# Get Priya's access keys first
aws iam create-access-key --user-name priya-backend
# Configure a second CLI profile for Priya
aws configure --profile priya
# Enter Priya's keys, region: ap-south-1

# Test: Priya tries to terminate EC2 — should FAIL
aws ec2 terminate-instances \
  --instance-ids i-1234567890abcdef0 \
  --profile priya
# Expected: AccessDenied error ✅ (permission denied = policy working correctly)

# Test: Priya can describe EC2 — should PASS  
aws ec2 describe-instances --profile priya
# Expected: Returns instance list ✅
```

### Test 2: Verify Sneha (frontend dev) CANNOT access backend S3 bucket

```bash
aws iam create-access-key --user-name sneha-frontend
aws configure --profile sneha

# Should FAIL — Sneha can only access technova-frontend bucket
aws s3 ls s3://technova-backend-data --profile sneha
# Expected: AccessDenied ✅

# Should PASS — Sneha's bucket
aws s3 ls s3://technova-frontend --profile sneha
# Expected: Lists files ✅
```

### Test 3: Use IAM Policy Simulator (Console)

```
1. Go to: https://policysim.aws.amazon.com
2. Select: priya-backend
3. Service: EC2
4. Action: TerminateInstances
5. Click: Run Simulation
6. Expected result: DENIED ✅
```

---

## LAB 9 — Enforce MFA for All Users

```bash
# Create MFA enforcement policy
cat > policy-require-mfa.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowViewAccountInfo",
      "Effect": "Allow",
      "Action": ["iam:GetAccountPasswordPolicy", "iam:ListVirtualMFADevices"],
      "Resource": "*"
    },
    {
      "Sid": "AllowManageOwnMFA",
      "Effect": "Allow",
      "Action": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
        "iam:ResyncMFADevice"
      ],
      "Resource": [
        "arn:aws:iam::*:mfa/${aws:username}",
        "arn:aws:iam::*:user/${aws:username}"
      ]
    },
    {
      "Sid": "DenyAllWithoutMFA",
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

# Attach MFA enforcement to ALL groups
for GROUP in grp-backend-devs grp-frontend-devs grp-data-engineers grp-devops grp-qa; do
  aws iam attach-group-policy \
    --group-name $GROUP \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-require-mfa
  echo "MFA policy attached to $GROUP"
done

echo "✅ MFA enforcement applied to all groups"
```

---

## LAB 10 — Set Password Policy for the Account

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

echo "✅ Password policy set:
  - Minimum 12 characters
  - Must have symbols, numbers, upper & lowercase
  - Expires every 90 days
  - Cannot reuse last 5 passwords"
```

---

## ✅ Final Verification Checklist

```bash
# Check all users exist
aws iam list-users --query 'Users[].UserName' --output table

# Check all groups exist
aws iam list-groups --query 'Groups[].GroupName' --output table

# Check who is in each group
for GROUP in grp-admins grp-backend-devs grp-frontend-devs grp-data-engineers grp-devops grp-qa; do
  echo "--- $GROUP ---"
  aws iam get-group --group-name $GROUP \
    --query 'Users[].UserName' --output table
done

# Check all roles exist
aws iam list-roles \
  --query 'Roles[?starts_with(RoleName, `role-`)].RoleName' \
  --output table

# Check custom policies
aws iam list-policies --scope Local \
  --query 'Policies[].PolicyName' --output table
```

---

## 🗺️ Final Architecture — What You Built

```
TechNova AWS Account
│
├── 👤 Users (9 total)
│   ├── arjun-admin      → grp-admins
│   ├── priya-backend    → grp-backend-devs
│   ├── rahul-backend    → grp-backend-devs
│   ├── sneha-frontend   → grp-frontend-devs
│   ├── karthik-frontend → grp-frontend-devs
│   ├── divya-data       → grp-data-engineers
│   ├── anand-devops     → grp-devops
│   ├── meena-qa         → grp-qa
│   └── ravi-finance     → Billing policy (direct)
│
├── 👥 Groups (6 total)
│   ├── grp-admins          → AdministratorAccess
│   ├── grp-backend-devs    → policy-backend-dev + policy-require-mfa
│   ├── grp-frontend-devs   → policy-frontend-dev + policy-require-mfa
│   ├── grp-data-engineers  → policy-data-engineer + policy-require-mfa
│   ├── grp-devops          → policy-devops + policy-require-mfa
│   └── grp-qa              → policy-qa + policy-require-mfa
│
├── 📋 Custom Policies (6 total)
│   ├── policy-backend-dev
│   ├── policy-frontend-dev
│   ├── policy-data-engineer
│   ├── policy-devops
│   ├── policy-qa
│   └── policy-require-mfa
│
└── 🎭 Roles (3 total)
    ├── role-ec2-s3-access      → Used by EC2 instances
    ├── role-lambda-dynamodb    → Used by Lambda functions
    └── role-ec2-cloudwatch     → Used by EC2 for logging
```

---

## 🧹 Cleanup (Delete Everything After Lab)

```bash
# Remove users from groups first, then delete
USERS=(priya-backend rahul-backend sneha-frontend karthik-frontend divya-data anand-devops meena-qa ravi-finance)
GROUPS=(grp-admins grp-backend-devs grp-frontend-devs grp-data-engineers grp-devops grp-qa)
ROLES=(role-ec2-s3-access role-lambda-dynamodb role-ec2-cloudwatch)
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Detach policies from groups
for GROUP in "${GROUPS[@]}"; do
  POLICIES=$(aws iam list-attached-group-policies --group-name $GROUP \
    --query 'AttachedPolicies[].PolicyArn' --output text)
  for POLICY in $POLICIES; do
    aws iam detach-group-policy --group-name $GROUP --policy-arn $POLICY
  done
done

# Remove users from groups and delete users
for USER in "${USERS[@]}"; do
  # Get groups for user
  USER_GROUPS=$(aws iam list-groups-for-user --user-name $USER \
    --query 'Groups[].GroupName' --output text)
  for GROUP in $USER_GROUPS; do
    aws iam remove-user-from-group --group-name $GROUP --user-name $USER
  done
  # Detach direct policies
  aws iam list-attached-user-policies --user-name $USER \
    --query 'AttachedPolicies[].PolicyArn' --output text | \
    xargs -I {} aws iam detach-user-policy --user-name $USER --policy-arn {}
  # Delete login profile
  aws iam delete-login-profile --user-name $USER 2>/dev/null
  # Delete access keys
  aws iam list-access-keys --user-name $USER \
    --query 'AccessKeyMetadata[].AccessKeyId' --output text | \
    xargs -I {} aws iam delete-access-key --user-name $USER --access-key-id {}
  aws iam delete-user --user-name $USER
  echo "Deleted user: $USER"
done

# Delete groups
for GROUP in "${GROUPS[@]}"; do
  aws iam delete-group --group-name $GROUP
  echo "Deleted group: $GROUP"
done

# Delete roles (detach policies first)
for ROLE in "${ROLES[@]}"; do
  POLICIES=$(aws iam list-attached-role-policies --role-name $ROLE \
    --query 'AttachedPolicies[].PolicyArn' --output text)
  for POLICY in $POLICIES; do
    aws iam detach-role-policy --role-name $ROLE --policy-arn $POLICY
  done
  aws iam delete-role --role-name $ROLE
  echo "Deleted role: $ROLE"
done

# Delete custom policies
CUSTOM_POLICIES=(policy-backend-dev policy-frontend-dev policy-data-engineer policy-devops policy-qa policy-require-mfa)
for POLICY in "${CUSTOM_POLICIES[@]}"; do
  aws iam delete-policy \
    --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/$POLICY
  echo "Deleted policy: $POLICY"
done

echo "✅ Full cleanup complete"
```

---

*Lab complete! You've built a production-style IAM setup for a 10-person startup from scratch.* 🎉
