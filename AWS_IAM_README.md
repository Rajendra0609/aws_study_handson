# 🔐 AWS IAM — Identity & Access Management
### Complete Learning Guide: Users, Roles, Policies, MFA & Least-Privilege

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

## 🧱 Step-by-Step Learning Path

---

### STEP 1 — Secure Your Root Account

The root account has **unlimited power** and cannot be restricted. Protect it immediately.

**What to do:**
1. Log in to [console.aws.amazon.com](https://console.aws.amazon.com)
2. Click your account name → **Security credentials**
3. Enable **MFA** on the root account (use an authenticator app like Google Authenticator)
4. **Never** create access keys for root
5. Lock root credentials in a safe place and stop using it

```
Root Account Checklist:
 ✅ MFA enabled
 ✅ No access keys created
 ✅ Strong password set
 ✅ Not used for daily tasks
```

---

### STEP 2 — Create Your First IAM Admin User

Instead of using root, create an IAM user with admin privileges for daily use.

**Console Steps:**
1. Go to **IAM → Users → Add Users**
2. Set a username (e.g., `admin-yourname`)
3. Select **"Provide user access to the AWS Management Console"**
4. Set a custom password
5. Attach policy: **AdministratorAccess** (AWS managed)
6. Enable MFA on this user too

**AWS CLI:**
```bash
# Create a user
aws iam create-user --user-name admin-yourname

# Attach AdministratorAccess policy
aws iam attach-user-policy \
  --user-name admin-yourname \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Create login profile (console password)
aws iam create-login-profile \
  --user-name admin-yourname \
  --password "YourStr0ngP@ssword!" \
  --password-reset-required
```

---

### STEP 3 — Understand IAM Policies (The Most Important Concept)

Policies are **JSON documents** that allow or deny actions on AWS resources.

**Policy Structure:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

**Key Fields Explained:**

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

### STEP 4 — Create IAM Groups

Groups let you **manage permissions at scale**. Attach policies to groups, not individual users.

**Example Group Structure:**
```
Developers Group    → AmazonEC2FullAccess + AmazonS3ReadOnlyAccess
Admins Group        → AdministratorAccess
ReadOnly Group      → ReadOnlyAccess
DataEngineers Group → AmazonS3FullAccess + AmazonRDSReadOnlyAccess
```

**Console Steps:**
1. IAM → User groups → Create group
2. Name the group (e.g., `Developers`)
3. Attach policies to the group
4. Add users to the group

**AWS CLI:**
```bash
# Create a group
aws iam create-group --group-name Developers

# Attach policy to group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess

# Add user to group
aws iam add-user-to-group \
  --group-name Developers \
  --user-name john-dev
```

---

### STEP 5 — Create IAM Roles

Roles are **temporary credentials** assumed by AWS services, applications, or users.
Unlike users, roles have **no permanent passwords or access keys**.

**When to use Roles:**
- EC2 instance needs to access S3 → Create a role for EC2
- Lambda function needs to write to DynamoDB → Create a role for Lambda
- One AWS account needs access to another → Cross-account role
- External user (Google, GitHub login) needs AWS access → Federated role

**How Roles Work:**
```
EC2 Instance
    │
    ├── Assumes IAM Role (e.g., EC2-S3-Reader)
    │       └── Policy: Allow s3:GetObject
    │
    └── Automatically gets temporary credentials (rotated every hour)
```

**Creating a Role for EC2 (Console):**
1. IAM → Roles → Create role
2. Trusted entity: **AWS service → EC2**
3. Attach policy: e.g., `AmazonS3ReadOnlyAccess`
4. Name it: `EC2-S3-Reader-Role`
5. Launch/modify EC2 → Attach this IAM role

**Creating a Role via CLI:**
```bash
# Trust policy — who can assume this role
cat > trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "ec2.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

# Create the role
aws iam create-role \
  --role-name EC2-S3-Reader-Role \
  --assume-role-policy-document file://trust-policy.json

# Attach permissions policy to role
aws iam attach-role-policy \
  --role-name EC2-S3-Reader-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
```

---

### STEP 6 — Apply Least-Privilege Principle

**Least-privilege** = Give only the minimum permissions needed, nothing more.

**How to apply it:**

#### ❌ Bad Practice
```json
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}
```
This gives full access to everything — never do this for regular users or applications.

#### ✅ Good Practice
```json
{
  "Effect": "Allow",
  "Action": [
    "s3:GetObject",
    "s3:PutObject"
  ],
  "Resource": "arn:aws:s3:::my-specific-bucket/*"
}
```
Only allows reading/writing to one specific S3 bucket.

**Least-Privilege Workflow:**
```
1. Start with NO permissions
2. Identify exactly what the user/service needs
3. Grant only those specific actions
4. Use specific resource ARNs (not *)
5. Add conditions where possible (e.g., MFA required, specific IP)
6. Review and remove unused permissions regularly
```

**Using IAM Access Analyzer:**
```bash
# See what permissions are actually being used
# Go to: IAM → Access Analyzer → Findings
# Or check "Last accessed" data on any user/role
aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::123456789012:user/john-dev
```

---

### STEP 7 — Enable and Enforce MFA

MFA adds a second layer of authentication (password + code from device).

**Types of MFA in AWS:**
| Type | Description |
|------|-------------|
| Virtual MFA | Google Authenticator, Authy (most common) |
| Hardware MFA | Physical security key (YubiKey) |
| SMS MFA | Text message (least secure, avoid) |

**Enforce MFA via Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllWithoutMFA",
      "Effect": "Deny",
      "NotAction": [
        "iam:CreateVirtualMFADevice",
        "iam:EnableMFADevice",
        "iam:GetUser",
        "iam:ListMFADevices",
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
```
Attach this policy to any group to **force users to set up MFA before doing anything**.

---

### STEP 8 — Create Access Keys for Programmatic Access

Access keys are used by CLI, SDKs, and applications to call AWS APIs.

**Best Practices:**
- Never put access keys in code (use roles instead)
- Never commit access keys to Git
- Rotate keys every 90 days
- Delete unused keys

```bash
# Create access keys
aws iam create-access-key --user-name john-dev

# List access keys
aws iam list-access-keys --user-name john-dev

# Delete old access key
aws iam delete-access-key \
  --user-name john-dev \
  --access-key-id AKIAIOSFODNN7EXAMPLE

# Configure CLI with keys
aws configure
# AWS Access Key ID: [your key]
# AWS Secret Access Key: [your secret]
# Default region: ap-south-1
# Default output format: json
```

---

### STEP 9 — Write Custom Policies

Learn to write policies for real-world scenarios.

**Example 1: Allow read-only access to a specific S3 bucket**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::company-reports",
      "arn:aws:s3:::company-reports/*"
    ]
  }]
}
```

**Example 2: Allow EC2 start/stop but not terminate**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:StartInstances",
      "ec2:StopInstances",
      "ec2:DescribeInstances"
    ],
    "Resource": "*"
  },
  {
    "Effect": "Deny",
    "Action": "ec2:TerminateInstances",
    "Resource": "*"
  }]
}
```

**Example 3: Require MFA for sensitive actions**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:DeleteObject",
    "Resource": "arn:aws:s3:::critical-bucket/*",
    "Condition": {
      "Bool": { "aws:MultiFactorAuthPresent": "true" }
    }
  }]
}
```

---

### STEP 10 — IAM Security Best Practices Checklist

```
Account Security:
 ✅ Root account MFA enabled
 ✅ Root access keys deleted
 ✅ IAM admin user created for daily use
 ✅ Admin user MFA enabled

User Management:
 ✅ Individual accounts for each person (no shared users)
 ✅ Users organized into Groups
 ✅ Permissions attached to Groups, not individual users
 ✅ Regular access review every 90 days

Permissions:
 ✅ Least-privilege applied everywhere
 ✅ No wildcard (*) actions unless necessary
 ✅ Resource-level permissions used (specific ARNs)
 ✅ AWS IAM Access Analyzer enabled

Credentials:
 ✅ Access keys rotated every 90 days
 ✅ Unused access keys deleted
 ✅ No access keys in code or Git
 ✅ Use IAM Roles for EC2/Lambda (not keys)

Monitoring:
 ✅ AWS CloudTrail enabled (logs all IAM actions)
 ✅ AWS Config rules for IAM compliance
 ✅ IAM Access Analyzer findings reviewed
```

---

## 🧪 Hands-On Labs (Do These!)

| Lab | What You'll Learn |
|-----|-------------------|
| 1. Create IAM user + group + policy | Core IAM workflow |
| 2. Launch EC2 with IAM Role to access S3 | Role-based access |
| 3. Create custom S3 read-only policy | Policy writing |
| 4. Enable MFA on user + enforce via policy | MFA enforcement |
| 5. Use IAM Access Analyzer | Permission analysis |
| 6. Create cross-account role | Advanced roles |
| 7. Simulate policy with IAM Policy Simulator | Debugging permissions |

**IAM Policy Simulator:** [policysim.aws.amazon.com](https://policysim.aws.amazon.com)
Use this to test what your policies allow/deny before deploying.

---

## 📚 Key ARN Format

Every AWS resource has a unique ARN (Amazon Resource Name):

```
arn:aws:iam::123456789012:user/john-dev
arn:aws:iam::123456789012:group/Developers
arn:aws:iam::123456789012:role/EC2-S3-Reader-Role
arn:aws:iam::123456789012:policy/MyCustomPolicy
arn:aws:s3:::my-bucket
arn:aws:ec2:ap-south-1:123456789012:instance/i-0abc123
```

Format: `arn:partition:service:region:account-id:resource`

---

## 🎯 Learning Resources

| Resource | Link | Cost |
|----------|------|------|
| AWS IAM Documentation | docs.aws.amazon.com/IAM | Free |
| AWS Skill Builder — IAM Course | skillbuilder.aws | Free |
| AWS Free Tier (Practice) | aws.amazon.com/free | Free |
| Stephane Maarek's AWS Course | Udemy | Paid |
| AWS Well-Architected — Security Pillar | aws.amazon.com/architecture | Free |

---

## ⚡ Quick Reference Cheat Sheet

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

# IAM Role commands
aws iam create-role --role-name ROLENAME --assume-role-policy-document file://trust.json
aws iam attach-role-policy --role-name ROLENAME --policy-arn POLICY_ARN
aws iam list-roles

# IAM Policy commands
aws iam create-policy --policy-name POLICYNAME --policy-document file://policy.json
aws iam list-policies --scope Local   # Your custom policies
aws iam get-policy-version --policy-arn POLICY_ARN --version-id v1

# Check what you have access to
aws iam get-user
aws iam list-groups-for-user --user-name USERNAME
aws iam simulate-principal-policy \
  --policy-source-arn USER_ARN \
  --action-names s3:GetObject \
  --resource-arns "arn:aws:s3:::my-bucket/*"
```

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
  2. Always enable MFA
  3. Grant least-privilege (minimum needed permissions)
  4. Use Roles instead of access keys for EC2/Lambda
  5. Review permissions regularly
```

---

*Happy learning! IAM mastery is the foundation of everything in AWS. Once you understand this, every other AWS service becomes easier.* 🚀
