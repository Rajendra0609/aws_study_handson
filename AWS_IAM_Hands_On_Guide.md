# 🔐 AWS IAM — Identity & Access Management Hands-On Guide

> A structured, practical guide to mastering AWS IAM — organized step-by-step from **Beginner → Advanced**, with every lesson mapped to console practice.

---

## 📋 Table of Contents

1. [How to Use This Guide](#how-to-use-this-guide)
2. [Phase 1 — IAM Foundations (Beginner)](#phase-1--iam-foundations-beginner)
   - 1.1 IAM Introduction — Users, Groups & Policies
   - 1.2 IAM Users & Groups — Hands On
   - 1.3 AWS Console Simultaneous Sign-in
3. [Phase 2 — Policies Deep Dive (Beginner–Intermediate)](#phase-2--policies-deep-dive-beginnerintermediate)
   - 2.1 IAM Policies — Concepts
   - 2.2 IAM Policies — Hands On
4. [Phase 3 — Security & Access (Intermediate)](#phase-3--security--access-intermediate)
   - 3.1 IAM MFA — Overview & Hands On
   - 3.2 AWS Access Keys, CLI & SDK
   - 3.3 AWS CLI Setup (Windows / Mac / Linux)
   - 3.4 AWS CLI — Hands On
   - 3.5 AWS CloudShell
5. [Phase 4 — IAM Roles (Intermediate–Advanced)](#phase-4--iam-roles-intermediateadvanced)
   - 4.1 IAM Roles for AWS Services
   - 4.2 IAM Roles — Hands On
6. [Phase 5 — Security Tools & Best Practices (Advanced)](#phase-5--security-tools--best-practices-advanced)
   - 5.1 IAM Security Tools — Hands On
   - 5.2 IAM Best Practices
   - 5.3 IAM Summary & Cheat Sheet
7. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
8. [Common Gotchas & Pro Tips](#common-gotchas--pro-tips)

---

## How to Use This Guide

- Follow the **phases in order** — each phase builds on the previous.
- Every topic has a **Concept Summary**, **Console Steps**, and **Hands-On Exercise**.
- Topics marked 🖥️ have an official AWS Console hands-on component.
- Complete the **Checkpoint** at the end of each phase before moving on.

> **Prerequisites:** An AWS root account. You'll create IAM users to avoid using the root account for daily work — that's literally the first thing we do.

---

## Phase 1 — IAM Foundations (Beginner)

### 1.1 IAM Introduction — Users, Groups & Policies

**Why it matters:** IAM is the security backbone of every AWS account. Every API call, every service interaction, every resource access is controlled by IAM. Getting this wrong means either locking yourself out or leaving the door wide open.

**Concept Summary:**

| Term | What it is | Analogy |
|------|-----------|---------|
| **Root Account** | God-mode account created with your email | CEO with a master key |
| **IAM User** | A person or app with long-term credentials | An employee badge |
| **IAM Group** | A collection of users sharing the same permissions | A department (e.g., DevOps) |
| **IAM Policy** | A JSON document that defines what is allowed or denied | A permission slip |
| **IAM Role** | Temporary identity assumed by a service or user | A visitor pass |

**Key Design Principle:**
- Users inherit permissions from the **groups** they belong to
- Permissions can also be attached **directly to a user** (but this is bad practice — use groups)
- One user can be a member of **multiple groups**

**Architecture Example:**

```
AWS Account
├── Group: Admins       → Policy: AdministratorAccess
│   └── User: alice
├── Group: Developers   → Policy: PowerUserAccess
│   ├── User: bob
│   └── User: carol
└── Group: Auditors     → Policy: ReadOnlyAccess
    └── User: dave
```

> 💡 **Golden Rule:** Never use the root account for daily tasks. Create an IAM admin user on day one and lock away the root account.

---

### 1.2 🖥️ IAM Users & Groups — Hands On

**Goal:** Create your first IAM admin user so you can stop using root.

**Console Steps:**

#### Step A — Create a Group

1. Go to **AWS Console → IAM → User Groups → Create Group**
2. **Group Name:** `Admins`
3. Under **Attach permissions policies**, search for and check `AdministratorAccess`
4. Click **Create User Group**

#### Step B — Create a User

1. Go to **IAM → Users → Create User**
2. **User name:** `admin-yourname` (e.g., `admin-alice`)
3. Check **Provide user access to the AWS Management Console**
4. Choose **I want to create an IAM user**
5. Set a **Custom password** (or auto-generate and force change on login)
6. Click **Next**
7. Under **Add user to group**, select the `Admins` group you just created
8. Click **Next → Create User**
9. **Download the `.csv` file** — this contains the login URL, username, and password

> ⚠️ **Important:** The login URL for IAM users is different from the root login. It looks like:
> `https://<account-id>.signin.aws.amazon.com/console`

#### Step C — Verify Permissions

1. Open an **incognito/private browser window**
2. Navigate to the IAM user login URL from the `.csv`
3. Sign in with your new IAM user credentials
4. Confirm you can access services (e.g., EC2, S3)

**Exercise:** Create two more groups — `Developers` (attach `PowerUserAccess`) and `ReadOnly` (attach `ReadOnlyAccess`). Create a user in each group and confirm the different permission levels.

---

### 1.3 🖥️ AWS Console Simultaneous Sign-in

**Goal:** Log in to multiple AWS accounts or users simultaneously using different browsers.

**Why it's useful:** Test your IAM user's permissions while still being logged in as admin in the main browser.

**Steps:**

1. **Chrome:** Sign in as IAM user `admin-alice`
2. **Firefox:** Sign in as IAM user `developer-bob`
3. **Incognito window:** Sign in as root (for emergencies only)

Each browser maintains a separate session — this lets you compare permission behavior side by side.

> 💡 **Tip:** Use browser profile colors (Chrome Profiles) to visually distinguish accounts. Color-code: 🔴 Root, 🟡 Admin, 🟢 Dev.

---

### Phase 1 Checkpoint ✅

Before moving on, confirm you can:
- [ ] Explain the difference between a User, Group, Role, and Policy
- [ ] Create an IAM group with a managed policy attached
- [ ] Create an IAM user and add them to a group
- [ ] Log in as an IAM user (not root) in a separate browser window
- [ ] Find your account's IAM sign-in URL

---

## Phase 2 — Policies Deep Dive (Beginner–Intermediate)

### 2.1 IAM Policies — Concepts

**Concept Summary:**

A policy is a **JSON document** with one or more statements. Each statement defines:

| Field | Purpose | Example |
|-------|---------|---------|
| `Effect` | Allow or Deny | `"Allow"` |
| `Action` | What API calls are permitted | `"s3:GetObject"` |
| `Resource` | Which resources the action applies to | `"arn:aws:s3:::my-bucket/*"` |
| `Condition` | Optional — extra restrictions | `"aws:MultiFactorAuthPresent": "true"` |

**Policy Types:**

| Type | Description | Managed by |
|------|-------------|------------|
| **AWS Managed** | Pre-built by AWS, maintained by AWS | AWS |
| **Customer Managed** | Custom policies you create | You |
| **Inline** | Embedded directly in a user/group/role | You (avoid for most cases) |

**Example — Custom Read-Only S3 Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-company-bucket",
        "arn:aws:s3:::my-company-bucket/*"
      ]
    }
  ]
}
```

**Policy Evaluation Logic:**

```
Request comes in
    ↓
Is there an explicit DENY?  → YES → ❌ DENY
    ↓ NO
Is there an explicit ALLOW? → YES → ✅ ALLOW
    ↓ NO
                                    ❌ DENY (implicit default)
```

> 💡 **Key Rule:** An explicit `Deny` **always wins** over any `Allow`, even if the allow is on a different policy.

---

### 2.2 🖥️ IAM Policies — Hands On

**Goal:** Create a custom policy, attach it to a user, and verify it works.

#### Step A — Explore AWS Managed Policies

1. Go to **IAM → Policies**
2. Filter by **Type: AWS managed**
3. Click on `AmazonS3ReadOnlyAccess`
4. Click the **JSON** tab — study the structure
5. Note the `Effect`, `Action`, and `Resource` fields

#### Step B — Create a Custom Policy

1. Go to **IAM → Policies → Create Policy**
2. Click the **JSON** tab and paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:List*"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": "ec2:TerminateInstances",
      "Resource": "*"
    }
  ]
}
```

3. Click **Next**
4. **Policy name:** `EC2-ReadOnly-NoTerminate`
5. **Description:** `Allows viewing EC2 resources but blocks termination`
6. Click **Create Policy**

#### Step C — Attach the Policy & Test

1. Go to **IAM → Users → developer-bob → Add permissions**
2. Choose **Attach policies directly**
3. Search for and attach `EC2-ReadOnly-NoTerminate`
4. Log in as `developer-bob` in another browser
5. Go to **EC2** and confirm you can *view* instances
6. Try to **terminate an instance** — you should get an "Access Denied" error ✅

#### Step D — Use the Policy Simulator

1. Go to **IAM → Policy Simulator** (search for it in the top bar)
2. Select **IAM User: developer-bob**
3. Select **Service: EC2**, **Action: TerminateInstances**
4. Click **Run Simulation** — confirms `DENY` before you even try it in real life

**Exercise:** Modify the policy to also deny `ec2:StopInstances`. Test again with the simulator.

---

### Phase 2 Checkpoint ✅

Before moving on, confirm you can:
- [ ] Read a JSON IAM policy and explain what it allows/denies
- [ ] Create a custom IAM policy from scratch
- [ ] Attach a policy to a user and test its effect
- [ ] Use the IAM Policy Simulator to test permissions without making real API calls
- [ ] Explain what happens when Allow and Deny conflict

---

## Phase 3 — Security & Access (Intermediate)

### 3.1 🖥️ IAM MFA — Overview & Hands On

**Concept Summary:**

MFA (Multi-Factor Authentication) adds a second layer of security. Even if your password is stolen, an attacker can't log in without the physical MFA device.

**MFA Device Types:**

| Type | Device | How it works |
|------|--------|-------------|
| **Virtual MFA** | Google Authenticator, Authy | 6-digit rotating code on your phone |
| **Hardware MFA** | YubiKey, Gemalto | Physical USB/NFC key |
| **U2F Security Key** | YubiKey | Tap to authenticate |

**Console Steps — Enable MFA on your IAM user:**

1. Sign in as your IAM admin user
2. Click your username in the top right → **Security credentials**
3. Under **Multi-factor authentication (MFA)** → click **Assign MFA device**
4. **Device name:** `my-phone-mfa`
5. Choose **Authenticator app** → click **Next**
6. Open **Google Authenticator** or **Authy** on your phone
7. Scan the QR code shown in the console
8. Enter **two consecutive MFA codes** from the app (wait for the second one to refresh)
9. Click **Add MFA**

**Enforce MFA via Policy (Best Practice):**

Add this condition to policies that should require MFA:

```json
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "BoolIfExists": {
      "aws:MultiFactorAuthPresent": "false"
    }
  }
}
```

> ⚠️ **Important:** Always enable MFA on the **root account** first. This is the most critical security action you can take.

**Exercise:** Enable MFA on the root account and on your `admin-alice` user. Verify login now prompts for the MFA code.

---

### 3.2 AWS Access Keys, CLI & SDK

**Concept Summary:**

| Access Method | Use Case | Credentials Used |
|--------------|---------|-----------------|
| **Console** | Human, browser-based access | Username + Password (+ MFA) |
| **CLI** | Command-line scripting & automation | Access Key ID + Secret Access Key |
| **SDK** | Application code (Python, Java, Node.js) | Access Key ID + Secret Access Key |

**Access Key Rules:**
- Each IAM user can have **max 2 access keys** (allows rotation without downtime)
- Access keys are shown **only once** at creation — store them securely
- Never commit access keys to Git or embed them in code
- **Prefer IAM Roles over access keys** whenever possible (especially on EC2, Lambda, etc.)

> 🔴 **Security Warning:** Leaked access keys are the #1 cause of AWS account breaches and surprise bills. Tools like `git-secrets` and AWS's own secret scanner can detect leaked keys automatically.

---

### 3.3 🖥️ AWS CLI Setup

**Goal:** Install and configure the AWS CLI so you can interact with AWS from your terminal.

#### Windows

1. Download the AWS CLI installer:
   [https://awscli.amazonaws.com/AWSCLIV2.msi](https://awscli.amazonaws.com/AWSCLIV2.msi)
2. Run the `.msi` installer and follow the prompts
3. Open **Command Prompt** and verify:

```cmd
aws --version
```

#### Mac OS X

```bash
# Option 1: Homebrew (recommended)
brew install awscli

# Option 2: Official installer
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# Verify
aws --version
```

#### Linux (Ubuntu / Amazon Linux)

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify
aws --version
```

---

### 3.4 🖥️ AWS CLI — Hands On

**Goal:** Create an access key for your IAM user and configure the CLI.

#### Step A — Create an Access Key

1. Go to **IAM → Users → admin-alice → Security credentials**
2. Under **Access keys** → click **Create access key**
3. Use case: **Command Line Interface (CLI)**
4. Check the acknowledgment → **Next**
5. (Optional) Add a description tag: `dev-laptop`
6. Click **Create access key**
7. **Download the `.csv`** or copy the Secret Access Key NOW — you won't see it again

#### Step B — Configure the CLI

```bash
aws configure
```

You'll be prompted for:

```
AWS Access Key ID [None]:     AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]:   us-east-1
Default output format [None]: json
```

#### Step C — Test the CLI

```bash
# List your IAM users
aws iam list-users

# Get your identity (who am I?)
aws sts get-caller-identity

# List S3 buckets
aws s3 ls

# List EC2 instances in your region
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name]' --output table
```

**CLI Credentials are stored at:**

```
~/.aws/credentials      # Access key ID and secret
~/.aws/config           # Region and output format
```

**Multiple Profiles (for multiple accounts):**

```bash
# Configure a second profile
aws configure --profile production

# Use a specific profile
aws s3 ls --profile production

# Or set an environment variable
export AWS_PROFILE=production
```

**Exercise:** Use the CLI to create an S3 bucket:
```bash
aws s3 mb s3://my-test-bucket-$(date +%s)
aws s3 ls
aws s3 rb s3://my-test-bucket-<timestamp>
```

---

### 3.5 🖥️ AWS CloudShell

**Concept Summary:**

CloudShell is a browser-based terminal inside the AWS Console. It comes pre-installed with the AWS CLI, Python, Node.js, and other tools. No setup required — it automatically uses your console session credentials.

**How to Open:**

1. In the AWS Console top bar, click the **CloudShell icon** (looks like a terminal `>_`)
2. Wait for the shell to initialize (first launch takes ~30 seconds)
3. Run any AWS CLI command directly — you're already authenticated

```bash
# You're already authenticated as your console user
aws sts get-caller-identity

# Try it out
aws iam list-users
aws ec2 describe-regions --output table
```

**CloudShell Features:**
- 1 GB persistent storage per region (files survive between sessions)
- Pre-installed: `aws`, `python3`, `node`, `git`, `jq`, `vim`
- Available in most (not all) AWS regions — check the region availability note

> 💡 **Tip:** CloudShell is perfect for quick CLI tasks when you're not at your own machine. No credential setup needed.

> ⚠️ **Region Availability:** CloudShell is NOT available in every region. Check the [CloudShell region list](https://docs.aws.amazon.com/cloudshell/latest/userguide/supported-aws-regions.html) before relying on it.

---

### Phase 3 Checkpoint ✅

Before moving on, confirm you can:
- [ ] Enable MFA on an IAM user and the root account
- [ ] Create an access key and configure the CLI with `aws configure`
- [ ] Run `aws sts get-caller-identity` and understand the output
- [ ] Use multiple CLI profiles for different accounts
- [ ] Open CloudShell and run CLI commands from the browser

---

## Phase 4 — IAM Roles (Intermediate–Advanced)

### 4.1 IAM Roles for AWS Services

**Concept Summary:**

An IAM Role is like a permission set that can be **assumed** by:
- AWS services (EC2, Lambda, ECS, etc.)
- Other IAM users (cross-account access)
- External identity providers (SAML, OIDC — for SSO)

**Why Roles Instead of Access Keys on Services?**

| Method | Problem |
|--------|---------|
| Hardcoded access keys in code | Keys can be leaked via Git, logs, or environment variables |
| Access keys on EC2 | If the instance is compromised, keys are stolen |
| **IAM Role on EC2** | ✅ AWS auto-rotates temporary credentials — never exposed |

**How It Works (Assume Role):**

```
EC2 Instance → assumes Role "EC2-S3-ReadOnly"
             → gets temporary credentials from STS (expiry ~1 hour)
             → credentials auto-refresh via Instance Metadata Service
             → app calls S3 without any hardcoded keys
```

**Trust Policy vs Permission Policy:**

| Policy | Purpose | Example |
|--------|---------|---------|
| **Trust Policy** | Who can assume the role | `ec2.amazonaws.com` can assume this role |
| **Permission Policy** | What the role can do | `s3:GetObject` on `my-bucket` |

---

### 4.2 🖥️ IAM Roles — Hands On

**Goal:** Attach an IAM Role to an EC2 instance so it can access S3 without any credentials.

#### Step A — Create the Role

1. Go to **IAM → Roles → Create Role**
2. **Trusted entity type:** AWS service
3. **Service:** EC2 → click **Next**
4. Search for and attach: `AmazonS3ReadOnlyAccess`
5. Click **Next**
6. **Role name:** `EC2-S3-ReadOnly-Role`
7. **Description:** `Allows EC2 instances to read from S3`
8. Click **Create Role**

#### Step B — Attach Role to EC2 Instance

1. Go to **EC2 → Instances**
2. Select your running instance
3. **Actions → Security → Modify IAM Role**
4. Select `EC2-S3-ReadOnly-Role` from the dropdown
5. Click **Update IAM Role**

#### Step C — Test on the Instance

Connect via EC2 Instance Connect or SSH, then run:

```bash
# This works even with no ~/.aws/credentials file!
aws s3 ls

# See the temporary credentials the role provides
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/EC2-S3-ReadOnly-Role

# Verify the caller identity
aws sts get-caller-identity
```

You should see the role ARN in the output — confirming the instance is using the role, not personal credentials.

#### Step D — Test AssumeRole (Advanced)

You can also manually assume a role from the CLI:

```bash
# Assume the role and get temporary credentials
aws sts assume-role \
  --role-arn "arn:aws:iam::<ACCOUNT_ID>:role/EC2-S3-ReadOnly-Role" \
  --role-session-name "test-session"

# This returns AccessKeyId, SecretAccessKey, and SessionToken (temporary)
```

**Exercise:** Create a new role called `Lambda-DynamoDB-Role` for the Lambda service with `AmazonDynamoDBReadOnlyAccess`. Don't attach it anywhere — just review the trust policy JSON that AWS generates automatically.

---

### Phase 4 Checkpoint ✅

Before moving on, confirm you can:
- [ ] Explain why roles are better than access keys for services
- [ ] Create an IAM Role for an AWS service (EC2)
- [ ] Attach a role to a running EC2 instance
- [ ] Verify credentials are coming from the role using the metadata endpoint
- [ ] Explain the difference between Trust Policy and Permission Policy

---

## Phase 5 — Security Tools & Best Practices (Advanced)

### 5.1 🖥️ IAM Security Tools — Hands On

**Two essential security tools built into IAM:**

#### Tool 1 — IAM Credentials Report (Account-Level)

Generates a CSV report of all IAM users and the status of their credentials: password age, access key usage, MFA status, etc.

**Console Steps:**

1. Go to **IAM → Credential report** (left sidebar)
2. Click **Download credential report**
3. Open the CSV and review:
   - `password_last_used` — who hasn't logged in recently?
   - `access_key_1_last_used_date` — any unused access keys?
   - `mfa_active` — who is missing MFA?

```bash
# Via CLI
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d
```

**What to look for:**
- [ ] Users with `mfa_active = false` → enforce MFA
- [ ] Access keys older than 90 days → rotate them
- [ ] Users who never logged in → delete or deactivate

#### Tool 2 — IAM Access Advisor (User-Level)

Shows which services a user has **permission** to access and **when they last accessed** each service. Used for tightening over-permissive policies.

**Console Steps:**

1. Go to **IAM → Users → Select a user**
2. Click the **Access Advisor** tab
3. Review the list — look for services marked "Never accessed" or last accessed years ago

**Principle of Least Privilege in Action:**

```
User has: AdministratorAccess
Access Advisor shows: Only ever used S3 and EC2

Action: Replace AdministratorAccess with:
→ AmazonS3FullAccess + AmazonEC2FullAccess

Result: Same functionality, massively reduced blast radius if compromised
```

**Exercise:** Run the Credentials Report and identify any account security gaps. Use Access Advisor on your developer user to see which services they've never used — then propose a tighter policy.

---

### 5.2 IAM Best Practices

These are AWS's official recommendations — and exam topics:

| # | Best Practice | Details |
|---|--------------|---------|
| 1 | **Lock away root** | Enable MFA on root. Never use root for daily tasks. Don't create access keys for root. |
| 2 | **One user per person** | Never share IAM users. Each human = one unique IAM user. |
| 3 | **Use groups for permissions** | Assign policies to groups, not individual users. |
| 4 | **Principle of Least Privilege** | Grant only the permissions needed to do the job — no more. |
| 5 | **Use IAM Roles for services** | EC2, Lambda, ECS should all use roles — never hardcoded access keys. |
| 6 | **Enable MFA for all users** | Especially for users with console access. |
| 7 | **Rotate access keys regularly** | Rotate every 90 days. Use the 2-key rotation pattern to avoid downtime. |
| 8 | **Use IAM Policies, not inline policies** | Customer-managed policies are reusable and auditable. |
| 9 | **Monitor with Credentials Report** | Run monthly to catch stale or unused credentials. |
| 10 | **Use Access Advisor** | Periodically tighten policies based on actual usage. |
| 11 | **Never put credentials in code** | Use environment variables, AWS Secrets Manager, or Parameter Store. |
| 12 | **Tag IAM resources** | Tag users, roles, and policies with team/project for auditability. |

**Access Key Rotation Pattern (Zero-Downtime):**

```
Step 1: Create Key 2 (now have Key 1 + Key 2)
Step 2: Update all apps/scripts to use Key 2
Step 3: Verify Key 2 is working in all places
Step 4: Deactivate Key 1 (don't delete yet — easy to re-enable if needed)
Step 5: Wait 24–48 hours; confirm nothing broke
Step 6: Delete Key 1
```

---

### 5.3 IAM Summary

**Everything in IAM in one reference:**

| Concept | Key Points |
|---------|-----------|
| **Users** | Long-term credentials (password + access keys). One per person. |
| **Groups** | Collection of users. Policies are attached to groups, not users directly. |
| **Policies** | JSON documents defining Allow/Deny for Actions on Resources. |
| **Roles** | Temporary credentials assumed by services or users. Preferred over access keys for services. |
| **MFA** | Second factor for console login. Always enable, especially for admin users and root. |
| **Access Keys** | For CLI/SDK access. Max 2 per user. Rotate every 90 days. Never commit to code. |
| **Credentials Report** | Account-wide audit of all user credential statuses. |
| **Access Advisor** | Per-user audit of which services were used — helps tighten policies. |
| **Policy Simulator** | Test IAM policies before applying them in production. |
| **CloudShell** | Browser-based CLI, pre-authenticated using your console session. |

---

### Phase 5 Checkpoint ✅

Before moving on, confirm you can:
- [ ] Generate and interpret the IAM Credentials Report
- [ ] Use Access Advisor to identify over-permissioned users
- [ ] Recall and apply the top IAM best practices
- [ ] Explain the access key rotation process
- [ ] Identify what each IAM concept is used for (Users, Groups, Roles, Policies)

---

## Quick Reference Cheat Sheet

```
# WHO AM I?
aws sts get-caller-identity

# LIST USERS
aws iam list-users

# LIST GROUPS
aws iam list-groups

# LIST POLICIES ATTACHED TO USER
aws iam list-attached-user-policies --user-name alice

# LIST USERS IN A GROUP
aws iam get-group --group-name Developers

# CREATE A USER
aws iam create-user --user-name bob

# ADD USER TO GROUP
aws iam add-user-to-group --user-name bob --group-name Developers

# CREATE ACCESS KEY FOR USER
aws iam create-access-key --user-name bob

# GENERATE CREDENTIALS REPORT
aws iam generate-credential-report
aws iam get-credential-report --output text --query Content | base64 -d

# ASSUME ROLE
aws sts assume-role --role-arn arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME> --role-session-name my-session

# INSTANCE METADATA (from inside EC2)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

---

## Common Gotchas & Pro Tips

| # | Gotcha / Tip |
|---|-------------|
| 1 | 🔴 **Explicit Deny always wins.** Even if 10 policies allow something, one explicit Deny overrides all of them. |
| 2 | 🔴 **Secret Access Key is shown only once.** If you lose it, you must deactivate the key and create a new one. |
| 3 | 🔴 **Root account has no restriction.** Any policy you write applies to IAM users/roles, NOT root. Root bypasses all IAM policies. |
| 4 | 🔴 **Never put access keys in code or Git.** GitHub scans for leaked AWS keys and AWS will alert you — but the damage may already be done. |
| 5 | 🟡 **IAM is global, not regional.** Users, groups, roles, and policies exist across all regions. |
| 6 | 🟡 **New IAM users have zero permissions** by default — they can't do anything until you attach a policy or add them to a group. |
| 7 | 🟡 **`*` in Action means all actions.** Be careful with `"Action": "*"` — it grants full access to that service or all services if Resource is also `*`. |
| 8 | 🟡 **Access Advisor shows last access up to the last 400 services.** Data can be up to 4 hours delayed. |
| 9 | 🟢 **Use IAM Roles for everything that runs code** — EC2, Lambda, ECS tasks, CodeBuild. Never hardcode credentials. |
| 10 | 🟢 **Policy Simulator is your best friend** — test policies before you apply them and accidentally lock yourself out. |
| 11 | 🟢 **Use Condition keys for extra control** — restrict by IP, require MFA, restrict by time of day, restrict by source VPC. |
| 12 | 🟢 **Permissions boundaries** let you delegate user creation safely — set a maximum limit on what someone can grant, even if they're an admin. |

---

## Recommended Practice Order

```
Week 1, Day 1–2: Phase 1 — Create users & groups, stop using root
Week 1, Day 3–4: Phase 2 — Write custom policies, use Policy Simulator
Week 1, Day 5:   Phase 3 — Enable MFA, set up CLI, explore CloudShell
Week 2, Day 1–2: Phase 4 — Create roles, attach to EC2, test AssumeRole
Week 2, Day 3:   Phase 5 — Audit with Credentials Report & Access Advisor
Week 2, Day 4–5: Review best practices + practice scenario questions
```

---

*Happy Cloud Learning! ☁️ — IAM is the foundation. Get this right and everything else gets easier.*
