# ☁️ AWS COMPUTE MASTERY GUIDE
## EC2 · Auto Scaling · ALB · EBS · AMIs · Launch Templates
> **From Basics to Expert — 40 Progressive Labs | 100% AWS Console UI | No CLI Required**

---

| Category | Details |
|---|---|
| Skill Level | Beginner → Intermediate → Advanced → Expert |
| Total Labs | 40 Hands-On Labs |
| Interface | AWS Management Console (UI only — no CLI) |
| Services Covered | EC2, Auto Scaling, ALB/ELB/NLB, EBS, AMIs, Launch Templates, FIS, Backup, Image Builder, VPC |
| AWS Region | us-east-1 (recommended) |
| Estimated Time | 60–80 hours total |
| Prerequisites | AWS account, basic Linux knowledge |

---

## 📑 Table of Contents

1. [Overview & Architecture](#1-overview--architecture)
2. [Key Concepts Deep Dive](#2-key-concepts-deep-dive)
3. [Prerequisites & Environment Setup](#3-prerequisites--environment-setup)
4. [Startup Team Guide](#4-startup-team-guide)
5. [🟢 Beginner Labs (1–7)](#5--beginner-labs-17)
6. [🟡 Intermediate Labs (8–15)](#6--intermediate-labs-815)
7. [🔴 Advanced Labs (16–25)](#7--advanced-labs-1625)
8. [⚫ Expert Labs (26–40)](#8--expert-labs-2640)
9. [Real-World Scenario: Startup to Scale](#9-real-world-scenario-startup-to-scale)
10. [Troubleshooting Guide](#10-troubleshooting-guide)
11. [Quick Reference Cheatsheet](#11-quick-reference-cheatsheet)

---

## 1. Overview & Architecture

### Core Services

| AWS Service | Purpose | Key Concepts |
|---|---|---|
| EC2 (Elastic Compute Cloud) | Virtual servers in the cloud | Instance types, AMIs, key pairs, security groups, user data |
| Auto Scaling Groups (ASG) | Automatically scale EC2 instances | Launch Templates, scaling policies, health checks, lifecycle hooks |
| ALB / ELB | Distribute traffic across instances | Target groups, listeners, rules, sticky sessions, SSL/TLS |
| EBS (Elastic Block Store) | Persistent block storage for EC2 | Volume types (gp3, io2), snapshots, encryption, multi-attach |
| AMIs (Amazon Machine Images) | Pre-configured OS images | Public AMIs, custom AMIs, AMI lifecycle, cross-region copy |
| Launch Templates | Reusable EC2 configuration | Versioning, parameter inheritance, ASG integration |

### Target Architecture

```
Internet
    │
    ▼
Route 53 (DNS)
    │
    ▼
CloudFront (CDN) ──── S3 (Static Assets)
    │
    ▼
Application Load Balancer (ALB)
├── Public Subnet A (us-east-1a)
└── Public Subnet B (us-east-1b)
    │
    ▼  [Listener Rules: HTTP→HTTPS redirect, path-based routing]
    │
    ▼
Auto Scaling Group (ASG)
├── EC2 Instance (Private Subnet A) ── EBS Volume (gp3, encrypted)
└── EC2 Instance (Private Subnet B) ── EBS Volume (gp3, encrypted)
    │
    ▼
NAT Gateway (for outbound internet from private subnets)
    │
    ▼
RDS / Aurora (Database Tier)
```

---

## 2. Key Concepts Deep Dive

### EC2 Instance Type Families

| Family | Code | Best For | Example Types |
|---|---|---|---|
| General Purpose | t, m | Web servers, dev/test, small DBs | t3.micro, t3.medium, m6i.large |
| Compute Optimized | c | CPU-intensive apps, batch, HPC | c5.large, c6g.xlarge, c7g.2xlarge |
| Memory Optimized | r, x, z | In-memory DBs, big data, SAP HANA | r6i.large, x2idn.16xlarge |
| Storage Optimized | i, d, h | High IOPS databases, data warehouses | i4i.large, d3.2xlarge |
| Accelerated | p, g, trn | ML training, GPU inference | g4dn.xlarge, p3.2xlarge |

> 💡 **Tip:** For labs always use `t3.micro` (Free Tier). For production use `m6i` or `c6i` for most web workloads.

### EBS Volume Types

| Volume Type | API Name | Max IOPS | Max Throughput | Use Case |
|---|---|---|---|---|
| General Purpose SSD v3 | gp3 | 16,000 | 1,000 MB/s | Default for most workloads, boot volumes |
| Provisioned IOPS SSD | io2 | 256,000 | 4,000 MB/s | Databases requiring consistent IOPS |
| Throughput Optimized HDD | st1 | 500 | 500 MB/s | Big data, log processing |
| Cold HDD | sc1 | 250 | 250 MB/s | Infrequently accessed, lowest cost |

> 💡 **Tip:** Always use `gp3` as default — cheaper than `gp2` with better baseline performance.

### Launch Templates vs Launch Configurations

| Feature | Launch Template | Launch Configuration |
|---|---|---|
| Versioning | ✅ Multiple versions | ❌ No versioning |
| T2/T3 Unlimited | ✅ Supported | ❌ Not supported |
| Spot + On-Demand mix | ✅ Supported | ❌ Not supported |
| AWS Recommendation | ✅ **Use this** | ⚠️ Deprecated — avoid |
| Metadata Options (IMDSv2) | ✅ Configurable | ❌ Limited |

### Auto Scaling Policy Types

| Policy Type | How It Works | Best For |
|---|---|---|
| Target Tracking | Maintains a metric at a target value (e.g., CPU at 50%) | Most workloads — simplest and most effective |
| Step Scaling | Adds/removes instances in steps based on alarm brackets | Variable load spikes |
| Scheduled Scaling | Scale at specific times (cron expressions) | Known patterns — business hours, batch jobs |
| Predictive Scaling | ML-based, scales ahead of time using history | Recurring load patterns |

### ALB Components

```
ALB (DNS: my-alb-1234.us-east-1.elb.amazonaws.com)
│
├── Listener: HTTPS:443
│   ├── Rule 1 (priority 10): path /api/* → API Target Group
│   ├── Rule 2 (priority 20): path /static/* → Static Target Group
│   └── Default: → App Target Group
│
└── Listener: HTTP:80
    └── Default: Redirect → HTTPS:443

Target Groups:
├── App TG:    EC2 instances, health check GET /health every 30s
├── API TG:    EC2 instances, health check GET /api/health every 30s
└── Static TG: EC2 instances or S3 origin
```

---

## 3. Prerequisites & Environment Setup

### 3.1 AWS Account Setup (Console)

1. Go to **https://aws.amazon.com** → click **Create an AWS Account**
2. **Enable MFA on the root account:**
   - Top-right menu → **Security credentials** → **Multi-factor authentication** → **Assign MFA device**
3. **Create an IAM user for labs (do NOT use root):**
   - Search bar → **IAM** → **Users** → **Create user**
   - Username: `lab-admin` | Check **Provide user access to the AWS Management Console**
   - Permissions: **Attach policies directly** → select `AdministratorAccess`
   - Click **Create user** → download the CSV with credentials
4. **Set a billing alert:**
   - Search → **AWS Budgets** → **Create budget** → **Use a template** → **Monthly cost budget** → Amount: `$10` → email: your address → **Create budget**

### 3.2 Create a Key Pair (Console)

1. Search bar → **EC2** → left sidebar → **Key Pairs** (under Network & Security)
2. Click **Create key pair**
3. Name: `MyLabKeyPair` | Type: **RSA** | Format: **.pem** (macOS/Linux) or **.ppk** (Windows PuTTY)
4. Click **Create key pair** — the `.pem` file downloads automatically
5. Store it safely; you cannot download it again

> ⚠️ **Warning:** Never upload `.pem` files to GitHub or share them publicly.

### 3.3 Check Your Region

- Always confirm the top-right region dropdown shows **US East (N. Virginia) us-east-1** before starting any lab
- Changing region mid-lab is the #1 cause of "resource not found" errors

---

## 4. Startup Team Guide

### 4.1 Team Structure

| Role | # People | Responsibilities | Tools / Skills |
|---|---|---|---|
| Cloud / DevOps Lead | 1 | Architecture, VPC, security, cost governance | Console, Well-Architected |
| DevOps Engineer | 1–2 | EC2, Launch Templates, ASG, ALB, health checks | AWS Console, CloudFormation |
| Backend Developer | 1–2 | App server setup, user-data scripts, AMI baking | Node.js/Python, Nginx, systemd |
| SRE | 1 | CloudWatch, scaling policies, log aggregation, runbooks | CloudWatch, Grafana, PagerDuty |
| Security Engineer | 1 | IAM, security groups, EBS encryption, SSM, compliance | IAM, Security Hub, GuardDuty, Inspector |
| QA / Testing Engineer | 1 | Load testing, failover testing, chaos engineering | k6, Locust, AWS FIS |

### 4.2 Sprint Execution Plan (3 × 2-Week Sprints)

#### Sprint 1 — Foundation (Week 1–2)
- **Cloud Lead:** Design multi-AZ VPC — public/private subnets, route tables, NAT gateway, IGW (Lab 04)
- **Security Engineer:** Create IAM roles, key pairs, baseline security groups (Lab 05)
- **DevOps Engineer:** Create base AMI, user-data bootstrap script, Launch Template v1 (Labs 03, 08)
- **Backend Dev:** Deploy app to single EC2 for testing; implement `/health` endpoint (Lab 01)
- **Deliverable:** Single working EC2 in private subnet, accessible via SSM (Lab 06)

#### Sprint 2 — Scale & Traffic (Week 3–4)
- **DevOps Engineer:** Create ASG (min 2, max 10), configure scaling policies (Lab 10)
- **DevOps Engineer:** Set up ALB, target group, listener rules (Lab 09)
- **SRE:** Configure CloudWatch dashboards, CPU/memory alarms (Lab 11)
- **Security Engineer:** Enable EBS encryption, SSL on ALB (Lab 15)
- **QA Engineer:** Run load tests, verify scale-out triggers
- **Deliverable:** Full Auto Scaling + ALB stack

#### Sprint 3 — Hardening & Production (Week 5–6)
- **DevOps Engineer:** Golden AMI pipeline via EC2 Image Builder (Lab 27)
- **SRE:** Lifecycle hooks, warm pools, instance refresh strategy (Labs 14, 20)
- **Security Engineer:** Inspector, automated patching via SSM (Lab 23)
- **QA Engineer:** Chaos testing via AWS FIS (Lab 24)
- **Cloud Lead:** Cost review, Compute Optimizer recommendations (Lab 21)
- **Deliverable:** Production-ready, self-healing infrastructure

### 4.3 RACI Matrix

| Task | Cloud Lead | DevOps Eng | Backend Dev | SRE | Security Eng | QA Eng |
|---|---|---|---|---|---|---|
| VPC Architecture | A/R | C | I | I | C | I |
| EC2 Launch Template | A | R | C | I | C | I |
| Auto Scaling Group | A | R | I | C | I | C |
| ALB Configuration | A | R | C | C | I | I |
| CloudWatch Alarms | I | C | I | A/R | I | C |
| IAM & Security | A | C | I | I | R | I |
| AMI Baking | A | R | C | I | C | I |
| Load Testing | I | C | C | C | I | A/R |

> `R` = Responsible · `A` = Accountable · `C` = Consulted · `I` = Informed

### 4.4 AWS Cost Estimate (Startup Scale)

| Resource | Spec | Qty | Est. Monthly Cost |
|---|---|---|---|
| EC2 (On-Demand baseline) | t3.medium (2 vCPU, 4GB) | 2 | $60–80 |
| EC2 (Spot for ASG burst) | t3.large / c5.large | 0–8 | $20–50 |
| Application Load Balancer | ALB + 1 rule | 1 | $20–30 |
| EBS Volumes | gp3, 30GB each | 2–10 | $5–25 |
| NAT Gateway | Per AZ | 2 | $65–90 |
| CloudWatch | Metrics + Logs + Alarms | 1 account | $10–20 |
| **Total Estimate** | | | **$190–345/month** |

---

## 5. 🟢 Beginner Labs (1–7)

> 💡 All beginner labs are Free Tier eligible using `t2.micro` or `t3.micro`. Expected cost: < $1 per lab.

---

### LAB 01 — Launch Your First EC2 Instance

**Objective:** Understand the EC2 launch wizard, AMI selection, instance types, key pairs, and security groups.

#### What You Will Learn
- Navigate the EC2 Console launch wizard
- Select a free-tier Amazon Linux 2023 AMI
- Create and assign a key pair
- Create a security group with inbound HTTP rule
- Launch an instance with a user data bootstrap script

#### Step-by-Step (Console UI)

**Step 1 — Open the Launch Wizard**
1. Sign in to **https://console.aws.amazon.com**
2. Search bar → type **EC2** → click **EC2**
3. In the left sidebar, click **Instances**
4. Click the orange **Launch instances** button (top-right)

**Step 2 — Configure the Instance**
1. **Name and tags** → Name field: `lab01-first-instance`
2. **Application and OS Images (AMI)**
   - Under "Quick Start", click **Amazon Linux**
   - Verify the AMI shown is **Amazon Linux 2023 AMI** (64-bit x86)
   - Confirm the badge **Free tier eligible** appears
3. **Instance type** → click the dropdown → type `t3.micro` → select it
4. **Key pair (login)**
   - Click **Create new key pair** (or select existing `MyLabKeyPair`)
   - Name: `MyLabKeyPair` | Type: RSA | Format: .pem → **Create key pair** (auto-downloads)

**Step 3 — Network Settings**
1. Click **Edit** next to Network settings
2. **VPC**: leave as default VPC
3. **Subnet**: No preference (any AZ)
4. **Auto-assign public IP**: Enable
5. **Firewall (Security Groups)** → **Create security group**
   - Security group name: `lab01-sg`
   - Description: `Lab 01 security group`
   - Inbound rules:
     - Rule 1: Type = **SSH**, Source = **My IP**
     - Click **Add security group rule** → Type = **HTTP**, Source = **Anywhere (0.0.0.0/0)**

**Step 4 — Storage**
1. Keep default: **8 GiB gp3** root volume (Free Tier eligible)
2. Encryption: leave as default (can encrypt in production)

**Step 5 — User Data**
1. Expand **Advanced details** (bottom of the page)
2. Scroll down to **User data** text area
3. Paste the following script:

```
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

echo "<h1>Hello from EC2! Instance ID: $INSTANCE_ID</h1>" > /var/www/html/index.html
```

**Step 6 — Launch**
1. Review the **Summary** panel on the right
2. Click **Launch instance**
3. Click **View all instances**
4. Wait ~60–90 seconds until the **Status check** column shows **2/2 checks passed** ✅

**Step 7 — Test**
1. Click your instance name to open its detail pane
2. Copy the **Public IPv4 address**
3. Open a new browser tab → paste the IP → you should see: `Hello from EC2! Instance ID: i-xxxx`

#### Validation Checklist
- [ ] Instance shows 2/2 status checks passed
- [ ] Browser shows the Apache Hello page with your Instance ID
- [ ] Security group has inbound HTTP (80) and SSH (22) rules

> ⚠️ **Clean up:** Select instance → **Instance state** → **Stop instance** after the lab to avoid charges.

---

### LAB 02 — EBS Volumes: Attach, Format, Mount, Snapshot

**Objective:** Understand EBS volume lifecycle — create, attach, snapshot, and restore via the Console.

#### Step-by-Step (Console UI)

**Step 1 — Create a New EBS Volume**
1. EC2 Console → left sidebar → **Elastic Block Store** → **Volumes**
2. Click **Create volume**
3. Settings:
   - Volume type: **gp3**
   - Size: **10 GiB**
   - Availability Zone: **us-east-1a** ← must match your Lab 01 instance's AZ
   - Encryption: check **Encrypt this volume** → KMS key: default (`aws/ebs`)
4. Click **Create volume** — note the Volume ID (e.g., `vol-0abc123`)

**Step 2 — Attach the Volume**
1. In the Volumes list, select your new volume → **Actions** → **Attach volume**
2. Instance: start typing your instance name → select `lab01-first-instance`
3. Device name: `/dev/sdf` (Linux may rename this to `/dev/nvme1n1`)
4. Click **Attach volume**
5. Wait for the volume **State** to change from `attaching` → `in-use`

**Step 3 — Connect to Your Instance**
1. Go back to **Instances** → select `lab01-first-instance`
2. Click **Connect** (top-right)
3. Choose **Session Manager** tab → click **Connect** (opens a browser terminal)

**Step 4 — Format and Mount (paste commands in browser terminal)**

```
lsblk
sudo mkfs.ext4 /dev/nvme1n1
sudo mkdir -p /data
sudo mount /dev/nvme1n1 /data
df -h /data
echo "Hello EBS" | sudo tee /data/test.txt
```

**Step 5 — Make the Mount Persistent**

```
sudo blkid /dev/nvme1n1
```
Copy the UUID value shown, then:
```
echo "UUID=<YOUR-UUID> /data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
sudo umount /data && sudo mount -a
cat /data/test.txt
```

**Step 6 — Create a Snapshot (Console)**
1. EC2 Console → **Volumes** → select your 10 GiB volume
2. **Actions** → **Create snapshot**
3. Description: `lab02-ebs-snapshot`
4. Click **Create snapshot**
5. Left sidebar → **Snapshots** → wait for State to show **completed** ✅

**Step 7 — Restore from Snapshot**
1. In **Snapshots**, select your snapshot → **Actions** → **Create volume from snapshot**
2. Volume type: `gp3` | Size: `10 GiB` | AZ: `us-east-1a` | Click **Create volume**
3. Attach the restored volume to your instance → mount it → verify `test.txt` still exists

#### Validation Checklist
- [ ] New 10 GiB volume visible with `lsblk`
- [ ] Snapshot shows **completed** state in console
- [ ] Data from snapshot (`test.txt`) visible on the restored volume mount
- [ ] `fstab` entry persists mount after instance reboot (test via **Instance state → Reboot**)

---

### LAB 03 — Create a Custom AMI

**Objective:** Bake a custom AMI from a configured instance, then launch a new instance from it.

#### Step-by-Step (Console UI)

**Step 1 — Prepare the Instance**
1. Connect to `lab01-first-instance` via Session Manager
2. Run the following to install additional software:

```
sudo yum install -y git htop wget curl jq
sudo pip3 install boto3
echo "Custom AMI built on: $(date)" | sudo tee /etc/build-info
```

**Step 2 — Stop the Instance**
1. EC2 Console → **Instances** → select `lab01-first-instance`
2. **Instance state** → **Stop instance** → **Stop**
3. Wait for state to change to **Stopped** ✅ (do NOT terminate)

**Step 3 — Create the AMI**
1. With the stopped instance still selected → **Actions** → **Image and templates** → **Create image**
2. Settings:
   - Image name: `lab03-custom-ami-v1.0`
   - Image description: `Amazon Linux 2023 with httpd, git, python3, boto3`
   - No reboot: **leave unchecked** ← ensures filesystem consistency
   - Root volume: keep 8 GiB gp3 default
3. Click **Create image**
4. Left sidebar → **AMIs** (under Images) → wait for Status to show **available** (~3–5 min)

**Step 4 — Launch from Your Custom AMI**
1. In **AMIs**, select your `lab03-custom-ami-v1.0`
2. Click **Launch instance from AMI**
3. Name: `lab03-from-ami`
4. Instance type: `t3.micro`
5. Key pair: `MyLabKeyPair`
6. Security group: select `lab01-sg`
7. Click **Launch instance**

**Step 5 — Verify**
1. Connect to the new instance via Session Manager
2. Run: `cat /etc/build-info` — should show the timestamp from when you built the AMI ✅

**Step 6 — Copy AMI to Another Region (Console)**
1. EC2 Console → **AMIs** → select your AMI
2. **Actions** → **Copy AMI**
3. Destination region: **US West (Oregon) us-west-2**
4. Name: `lab03-custom-ami-v1.0-copy`
5. Click **Copy AMI** — switch to us-west-2 in the top-right to see it appear

#### Validation Checklist
- [ ] AMI status shows `available` in us-east-1
- [ ] Instance launched from AMI passes 2/2 status checks
- [ ] `/etc/build-info` shows the correct build timestamp
- [ ] Copied AMI appears in us-west-2 console

---

### LAB 04 — VPC Design for EC2

**Objective:** Create a custom VPC with public/private subnets, route tables, IGW, and NAT Gateway entirely via Console.

#### Architecture
```
VPC: 10.0.0.0/16
├── Public Subnet A:  10.0.1.0/24  (us-east-1a) — ALB, Bastion
├── Public Subnet B:  10.0.2.0/24  (us-east-1b) — ALB
├── Private Subnet A: 10.0.11.0/24 (us-east-1a) — EC2 App Servers
└── Private Subnet B: 10.0.12.0/24 (us-east-1b) — EC2 App Servers
```

#### Step-by-Step (Console UI)

**Step 1 — Create the VPC**
1. Search bar → **VPC** → **Your VPCs** → **Create VPC**
2. Select **VPC only** (not VPC and more — we'll create pieces manually)
3. Name tag: `lab-vpc`
4. IPv4 CIDR: `10.0.0.0/16`
5. Tenancy: Default
6. Click **Create VPC** → note the VPC ID

**Step 2 — Enable DNS Hostnames**
1. Select `lab-vpc` → **Actions** → **Edit VPC settings**
2. Check **Enable DNS hostnames** → **Save**

**Step 3 — Create 4 Subnets**
Go to **Subnets** → **Create subnet** (repeat 4 times):

| Subnet Name | AZ | CIDR |
|---|---|---|
| `lab-pub-a` | us-east-1a | 10.0.1.0/24 |
| `lab-pub-b` | us-east-1b | 10.0.2.0/24 |
| `lab-pri-a` | us-east-1a | 10.0.11.0/24 |
| `lab-pri-b` | us-east-1b | 10.0.12.0/24 |

For each:
- VPC: select `lab-vpc`
- Fill in Subnet name, AZ, IPv4 CIDR
- Click **Create subnet**

**Enable auto-assign public IP for public subnets:**
- Select `lab-pub-a` → **Actions** → **Edit subnet settings** → check **Enable auto-assign public IPv4 address** → **Save**
- Repeat for `lab-pub-b`

**Step 4 — Create Internet Gateway**
1. Left sidebar → **Internet gateways** → **Create internet gateway**
2. Name: `lab-igw` → **Create internet gateway**
3. Select `lab-igw` → **Actions** → **Attach to VPC** → select `lab-vpc` → **Attach internet gateway**

**Step 5 — Create Public Route Table**
1. Left sidebar → **Route tables** → **Create route table**
2. Name: `lab-pub-rt` | VPC: `lab-vpc` → **Create**
3. Select `lab-pub-rt` → **Routes** tab → **Edit routes** → **Add route**
   - Destination: `0.0.0.0/0` | Target: **Internet Gateway** → `lab-igw`
   - Click **Save changes**
4. **Subnet associations** tab → **Edit subnet associations**
   - Select `lab-pub-a` and `lab-pub-b` → **Save associations**

**Step 6 — Create NAT Gateway**
1. Left sidebar → **NAT gateways** → **Create NAT gateway**
2. Name: `lab-nat`
3. Subnet: `lab-pub-a` (NAT must be in a public subnet)
4. Connectivity type: **Public**
5. **Allocate Elastic IP** → click the button → an EIP is auto-selected
6. Click **Create NAT gateway** → wait ~1–2 min for State: `available` ✅

**Step 7 — Create Private Route Table**
1. **Route tables** → **Create route table**
2. Name: `lab-pri-rt` | VPC: `lab-vpc` → **Create**
3. **Routes** → **Edit routes** → **Add route**
   - Destination: `0.0.0.0/0` | Target: **NAT Gateway** → `lab-nat`
   - **Save changes**
4. **Subnet associations** → **Edit** → select `lab-pri-a` and `lab-pri-b` → **Save**

#### Validation Checklist
- [ ] VPC shows 4 subnets
- [ ] Public subnets have route to IGW (0.0.0.0/0 → igw-xxx)
- [ ] Private subnets have route to NAT gateway
- [ ] EC2 launched in private subnet can reach the internet through NAT

> ⚠️ **Warning:** NAT Gateway costs ~$0.045/hr. **Delete it immediately after this lab** — go to NAT Gateways → select → Actions → Delete.

---

### LAB 05 — Security Groups Deep Dive

**Objective:** Create tiered security groups for the ALB → App → DB pattern via Console.

#### Security Group Architecture

| SG Name | Inbound Rules | Attached To |
|---|---|---|
| `sg-alb` | TCP 80 from `0.0.0.0/0`, TCP 443 from `0.0.0.0/0` | ALB |
| `sg-app` | TCP 8080 from `sg-alb`, TCP 22 from `sg-bastion` | EC2 App instances |
| `sg-db` | TCP 5432 from `sg-app` | RDS / DB instances |
| `sg-bastion` | TCP 22 from `your-office-ip/32` | Bastion EC2 |

#### Step-by-Step (Console UI)

1. EC2 Console → left sidebar → **Security Groups** → **Create security group**

**Create sg-alb:**
- Security group name: `sg-alb`
- Description: `ALB Security Group`
- VPC: `lab-vpc`
- Inbound rules → **Add rule**:
  - Type: HTTP | Port: 80 | Source: Anywhere IPv4 (`0.0.0.0/0`)
  - Type: HTTPS | Port: 443 | Source: Anywhere IPv4 (`0.0.0.0/0`)
- Click **Create security group** → note the SG ID

**Create sg-bastion:**
- Name: `sg-bastion` | VPC: `lab-vpc`
- Inbound: Type: SSH | Port: 22 | Source: **My IP**
- Click **Create security group**

**Create sg-app:**
- Name: `sg-app` | VPC: `lab-vpc`
- Inbound rules:
  - Type: Custom TCP | Port: 8080 | Source: **Custom** → select `sg-alb` from dropdown
  - Type: SSH | Port: 22 | Source: **Custom** → select `sg-bastion`
- Click **Create security group**

**Create sg-db:**
- Name: `sg-db` | VPC: `lab-vpc`
- Inbound rules:
  - Type: PostgreSQL | Port: 5432 | Source: **Custom** → select `sg-app`
- Click **Create security group**

> 💡 **Best Practice:** Use **AWS Systems Manager Session Manager** instead of bastion hosts — this eliminates port 22 entirely (see Lab 06).

#### Validation Checklist
- [ ] All 4 security groups created in `lab-vpc`
- [ ] `sg-app` allows traffic only from `sg-alb` on port 8080
- [ ] `sg-db` allows traffic only from `sg-app` on port 5432
- [ ] No security group has inbound `0.0.0.0/0` on port 22

---

### LAB 06 — SSM Session Manager (No SSH Required)

**Objective:** Access EC2 instances securely without key pairs or open SSH ports — 100% via Console.

#### Step-by-Step (Console UI)

**Step 1 — Create IAM Role for SSM**
1. Search → **IAM** → left sidebar → **Roles** → **Create role**
2. **Trusted entity type:** AWS service | **Use case:** EC2 → **Next**
3. **Permissions:** search for and select:
   - `AmazonSSMManagedInstanceCore`
   - `CloudWatchAgentServerPolicy` (for later labs)
4. → **Next**
5. Role name: `EC2SSMRole` → **Create role**

**Step 2 — Create Instance Profile & Attach Role**
1. Search → **EC2** → **Instances** → select your `lab01-first-instance`
2. **Actions** → **Security** → **Modify IAM role**
3. IAM role dropdown: select `EC2SSMRole` → **Update IAM role**
4. Wait ~1 minute for the SSM agent to register

**Step 3 — Verify SSM Connection**
1. Search → **Systems Manager**
2. Left sidebar → **Fleet Manager** (or **Session Manager**)
3. Your instance should appear as **Online** in Fleet Manager
4. Left sidebar → **Session Manager** → **Start session**
5. Select your instance → **Start session**
6. A browser-based terminal opens — no SSH or key pair needed! ✅

**Step 4 — Launch a New Instance Without SSH**
1. EC2 → **Launch instances**
2. Name: `lab06-ssm-instance`
3. AMI: Amazon Linux 2023 | Instance type: t3.micro
4. Key pair: **Proceed without a key pair** ← no key pair needed with SSM
5. Network: **Do NOT add** an SSH rule to the security group
6. **Advanced details** → IAM instance profile: select `EC2SSMRole`
7. Launch → connect via Session Manager ✅

#### Validation Checklist
- [ ] Instance reachable via Systems Manager → Session Manager
- [ ] No port 22 open in security group
- [ ] No key pair required for the connection
- [ ] Can run shell commands in the browser terminal

---

### LAB 07 — User Data & IMDSv2 (Instance Metadata)

**Objective:** Master user data scripts for automated bootstrapping and use IMDSv2 securely.

#### Step-by-Step (Console UI)

**Step 1 — Launch with User Data**
1. EC2 → **Launch instances**
2. Name: `lab07-userdata`
3. AMI: Amazon Linux 2023 | Type: t3.micro
4. Key pair: use `MyLabKeyPair` or **Proceed without key pair** + SSM role
5. Security group: add HTTP (80) from Anywhere
6. **Advanced details → User data** — paste this script:

```
#!/bin/bash
set -ex
exec > >(tee /var/log/user-data.log | logger -t user-data -s 2>/dev/console) 2>&1

yum update -y
yum install -y httpd awscli jq

# IMDSv2 — token-based metadata (always use this, never IMDSv1)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)
REGION=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/region)
INSTANCE_TYPE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-type)
PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/local-ipv4)

cat <<EOF > /var/www/html/index.html
<h1>Lab 07 — IMDSv2 Demo</h1>
<p>Instance ID: $INSTANCE_ID</p>
<p>AZ: $AZ | Region: $REGION | Type: $INSTANCE_TYPE</p>
<p>Private IP: $PRIVATE_IP</p>
EOF

systemctl start httpd && systemctl enable httpd
```

**Step 2 — Enforce IMDSv2 (Disable IMDSv1)**
1. After launch, select your instance
2. **Actions** → **Instance settings** → **Modify instance metadata options**
3. Set **IMDSv2** to: **Required** (this disables IMDSv1)
4. HTTP PUT response hop limit: **2** (allows containers to use metadata)
5. Click **Save**

**Step 3 — Verify**
1. Browse to the public IP — you should see the metadata page ✅
2. Connect via Session Manager and check user data log:
   ```
   cat /var/log/user-data.log
   ```

> ⚠️ **Warning:** Always set IMDSv2 to **Required**. IMDSv1 is vulnerable to SSRF attacks. Enforce this in all Launch Templates.

#### Validation Checklist
- [ ] Web page shows correct Instance ID, AZ, Region, and Instance Type
- [ ] User data log at `/var/log/user-data.log` shows successful execution
- [ ] Instance metadata options show **IMDSv2 = Required**

---

## 6. 🟡 Intermediate Labs (8–15)

---

### LAB 08 — Launch Templates: Full Configuration

**Objective:** Create a production-grade Launch Template with all recommended settings via Console.

#### Recommended Parameters

| Parameter | Value | Why |
|---|---|---|
| AMI ID | Custom AMI from Lab 03 | Pre-baked — faster boot, consistent config |
| Instance type | t3.medium | Baseline, can override in ASG |
| Key pair | None (use SSM) | SSM preferred for security |
| IAM instance profile | EC2SSMRole | Enables SSM, CloudWatch, S3 access |
| EBS root volume | gp3, 20GB, encrypted | Encryption at rest, better performance |
| Metadata options | HttpTokens=required | IMDSv2 mandatory for security |
| Detailed monitoring | Enabled | 1-minute CloudWatch metrics |

#### Step-by-Step (Console UI)

**Step 1 — Create the Launch Template**
1. EC2 Console → left sidebar → **Launch Templates** → **Create launch template**
2. **Name:** `lab-app-lt`
3. **Template version description:** `v1.0 initial`
4. Check **Provide guidance to help me set up a template that I can use with EC2 Auto Scaling**

**Step 2 — AMI**
1. Under **Application and OS Images** → click **My AMIs**
2. Select your `lab03-custom-ami-v1.0`

**Step 3 — Instance Type**
1. Instance type: `t3.medium`

**Step 4 — Key Pair**
1. Key pair: **Don't include in launch template** (we use SSM)

**Step 5 — Network Settings**
1. **Firewall (security groups)** → **Select existing security group** → choose `sg-app`

**Step 6 — Storage**
1. **Storage (volumes)** → Volume 1 (root):
   - Size: `20` GiB
   - Volume type: `gp3`
   - IOPS: `3000` (default gp3)
   - Throughput: `125` MB/s
   - Check **Encrypted** → KMS key: `aws/ebs` (default)
   - **Delete on termination:** Yes

**Step 7 — Advanced Details**
1. Expand **Advanced details**
2. **IAM instance profile:** `EC2SSMRole`
3. **Detailed CloudWatch monitoring:** Enable
4. **Metadata accessible:** Enabled
5. **Metadata version:** **V2 only (token required)** ← IMDSv2 enforced
6. **Metadata response hop limit:** `2`
7. **User data:** paste your Lab 07 bootstrap script

**Step 8 — Create**
1. Review all settings → **Create launch template**
2. Confirm it appears in the list with version number **1**

**Step 9 — Create Version 2**
1. Select `lab-app-lt` → **Actions** → **Create template version**
2. Source version: **1**
3. Version description: `v2.0 updated AMI`
4. Change only the AMI field to a newer AMI ID (or same for practice)
5. **Create launch template version**
6. **Actions** → **Set default version** → select **2** → **Set as default version**

#### Validation Checklist
- [ ] Launch Template created with 2 versions
- [ ] Version 2 is set as default
- [ ] Template shows IMDSv2 = required, monitoring = enabled, root volume encrypted

---

### LAB 09 — Application Load Balancer (ALB) Setup

**Objective:** Create an ALB with target group, health checks, and HTTP→HTTPS redirect via Console.

#### Step-by-Step (Console UI)

**Step 1 — Create Target Group**
1. EC2 Console → left sidebar → **Target Groups** → **Create target group**
2. Settings:
   - **Target type:** Instances
   - **Target group name:** `lab-app-tg`
   - **Protocol:** HTTP | **Port:** 80
   - **VPC:** `lab-vpc`
3. **Health checks:**
   - Protocol: HTTP
   - Path: `/health`
   - Port: Traffic port
   - **Advanced health check settings:**
     - Healthy threshold: `2`
     - Unhealthy threshold: `3`
     - Timeout: `5` seconds
     - Interval: `30` seconds
     - Success codes: `200`
4. Click **Next** → skip registering targets for now (ASG does this) → **Create target group**

**Step 2 — Create the ALB**
1. EC2 → left sidebar → **Load Balancers** → **Create load balancer**
2. Select **Application Load Balancer** → **Create**
3. Settings:
   - Name: `lab-alb`
   - Scheme: **Internet-facing**
   - IP address type: IPv4
4. **Network mapping:**
   - VPC: `lab-vpc`
   - Mappings: select **us-east-1a** (subnet: `lab-pub-a`) and **us-east-1b** (subnet: `lab-pub-b`)
5. **Security groups:** select `sg-alb` (remove the default SG)
6. **Listeners and routing:**
   - Listener: HTTP:80 | Default action: **Forward to** `lab-app-tg`
7. Click **Create load balancer**
8. Wait for State to change to **Active** (~1–2 min) — copy the **DNS name** ✅

**Step 3 — Add HTTP → HTTPS Redirect**
1. Select `lab-alb` → **Listeners** tab
2. Select the **HTTP:80** listener → **Actions** → **Edit listener**
3. Default actions: **Remove** the Forward rule → **Add action** → **Redirect**
   - Protocol: HTTPS | Port: 443 | Status code: **301 - Permanently moved**
4. Click **Save changes**

**Step 4 — Manually Register Test Instances**
1. **Target Groups** → select `lab-app-tg` → **Targets** tab → **Register targets**
2. Select your running lab instances → **Include as pending below** → **Register pending targets**
3. Wait for Health status to show **healthy** ✅

#### Validation Checklist
- [ ] ALB state shows Active
- [ ] HTTP:80 listener redirects to HTTPS:443
- [ ] Target group health check shows 200 OK
- [ ] Browsing to ALB DNS name returns your app page

---

### LAB 10 — Auto Scaling Groups (ASG) — Full Setup

**Objective:** Create an ASG with Launch Template, configure multiple scaling policies, and integrate with ALB.

#### Step-by-Step (Console UI)

**Step 1 — Create the ASG**
1. EC2 Console → left sidebar → **Auto Scaling Groups** → **Create Auto Scaling group**
2. **Step 1 — Choose launch template:**
   - Name: `lab-app-asg`
   - Launch template: `lab-app-lt` | Version: **Default ($Default)**
   - Click **Next**

**Step 2 — Instance Launch Options**
1. VPC: `lab-vpc`
2. Availability Zones and subnets: select **`lab-pri-a`** and **`lab-pri-b`**
3. Instance type requirements: override → add `t3.medium`, `t3a.medium`, `m5.large` for diversity
4. Click **Next**

**Step 3 — Configure ALB Integration**
1. **Load balancing:** Attach to an existing load balancer
2. Choose from your existing target groups: select `lab-app-tg`
3. **Health checks:**
   - Turn on **Elastic Load Balancing health checks**
   - Health check grace period: `300` seconds
4. **Additional settings:** Enable **Group metrics collection within CloudWatch**
5. Click **Next**

**Step 4 — Group Size and Scaling**
1. **Group size:**
   - Desired capacity: `2`
   - Minimum capacity: `2`
   - Maximum capacity: `10`
2. **Scaling policies:** Select **Target tracking scaling policy**
   - Policy name: `cpu-target-tracking`
   - Metric type: **Average CPU utilization**
   - Target value: `50`
   - Scale-in cooldown: `300` seconds
   - Scale-out cooldown: `60` seconds
3. Click **Next**

**Step 5 — Notifications (Optional)**
1. **Add notification** → create or select an SNS topic → event types: Launch, Terminate, Fail
2. Click **Next**

**Step 6 — Tags**
1. Add tag: Key = `Name` | Value = `asg-app-server` | Propagate to instances: ✅
2. Click **Next** → Review → **Create Auto Scaling group**

**Step 7 — Add a Second Scaling Policy (Request Count)**
1. Select `lab-app-asg` → **Automatic scaling** tab → **Create dynamic scaling policy**
2. Policy type: **Target tracking scaling**
3. Scaling policy name: `alb-request-count`
4. Metric type: **Application Load Balancer request count per target**
5. Target group: select `lab-app-tg`
6. Target value: `1000`
7. Click **Create**

**Step 8 — Add Scheduled Scaling**
1. **Automatic scaling** tab → **Create scheduled action**
2. Scale-down at night:
   - Name: `scale-down-night`
   - Recurrence: `0 20 * * *` (8 PM UTC)
   - Min: `1` | Max: `4` | Desired: `1`
3. Scale-up in morning:
   - Name: `scale-up-morning`
   - Recurrence: `0 6 * * *` (6 AM UTC)
   - Min: `2` | Max: `10` | Desired: `2`

#### Validation Checklist
- [ ] ASG shows 2 instances in-service in `Activity` tab
- [ ] Instances appear as **healthy** in ALB target group
- [ ] Three scaling policies created: CPU target tracking, ALB request count, 2 scheduled actions

---

### LAB 11 — CloudWatch Monitoring & Alarms

**Objective:** Create CloudWatch dashboards, alarms for ASG metrics, and notifications via SNS.

#### Step-by-Step (Console UI)

**Step 1 — Create SNS Topic for Alerts**
1. Search → **SNS** → **Topics** → **Create topic**
2. Type: **Standard** | Name: `lab-alerts`
3. **Create topic** → click **Create subscription**
4. Protocol: **Email** | Endpoint: your email address
5. **Create subscription** → check your email and click **Confirm subscription** ✅

**Step 2 — Create CPU Alarm**
1. Search → **CloudWatch** → left sidebar → **Alarms** → **All alarms** → **Create alarm**
2. **Select metric** → **EC2** → **By Auto Scaling Group** → `CPUUtilization` for `lab-app-asg` → **Select metric**
3. Settings:
   - Statistic: **Average** | Period: **5 minutes**
   - Threshold: Greater than **80**
4. Click **Next**
5. **Notification:**
   - Alarm state trigger: **In alarm**
   - SNS topic: `lab-alerts`
   - Also add **OK** state → same topic
6. Alarm name: `high-cpu-alarm` | Click **Next** → **Create alarm**

**Step 3 — Create Unhealthy Hosts Alarm**
1. **Create alarm** → **Select metric** → **ApplicationELB** → **Per AppELB, per TG Metrics** → `UnHealthyHostCount` for your target group → **Select metric**
2. Threshold: Greater than **0**
3. SNS: `lab-alerts` (In alarm state)
4. Name: `unhealthy-hosts` → **Create alarm**

**Step 4 — Create CloudWatch Dashboard**
1. Left sidebar → **Dashboards** → **Create dashboard**
2. Name: `EC2-Lab-Dashboard` → **Create dashboard**
3. Add widgets:
   - **Add widget** → **Line** → **Metrics** → EC2 → ASG: `CPUUtilization`
   - **Add widget** → **Number** → ALB: `HealthyHostCount` and `UnHealthyHostCount`
   - **Add widget** → **Line** → ALB: `TargetResponseTime`
   - **Add widget** → **Line** → ASG: `GroupInServiceInstances`
4. **Save dashboard** ✅

**Step 5 — Enable CloudWatch Logs for EC2**
1. In your Launch Template user data, the CloudWatch agent sends logs automatically if `EC2SSMRole` has `CloudWatchAgentServerPolicy` attached
2. CloudWatch → **Log groups** → verify `/var/log/messages` and `/var/log/user-data.log` groups appear

#### Key Metrics to Monitor

| Metric | Namespace | Warning / Critical | Action |
|---|---|---|---|
| CPUUtilization | AWS/EC2 | 60% / 80% | SNS alert + scale-out |
| HealthyHostCount | AWS/ApplicationELB | < 2 | PagerDuty P1 |
| TargetResponseTime | AWS/ApplicationELB | > 1s / > 3s | Alert + investigate |
| GroupInServiceInstances | AWS/AutoScaling | < 2 | Alert immediately |

#### Validation Checklist
- [ ] SNS subscription confirmed via email
- [ ] Two alarms created (CPU + unhealthy hosts)
- [ ] Dashboard shows 4 widgets with live metrics

---

### LAB 12 — Instance Refresh & Rolling Updates

**Objective:** Update all ASG instances to a new Launch Template version with zero downtime.

#### Step-by-Step (Console UI)

**Step 1 — Create New AMI (Updated App)**
1. Start a running instance → apply your app update via Session Manager
2. Stop the instance → **Actions → Image and templates → Create image**
3. Name: `lab-app-ami-v2.0` → **Create image** → wait for `available` state

**Step 2 — Create New Launch Template Version**
1. EC2 → **Launch Templates** → select `lab-app-lt`
2. **Actions** → **Modify template (Create new version)**
3. Source version: `$Default`
4. Version description: `v3.0 app v2.0 updated AMI`
5. **Application and OS Images** → **My AMIs** → select `lab-app-ami-v2.0`
6. All other settings remain unchanged → **Create launch template version**

**Step 3 — Update ASG to Use New Version**
1. **Auto Scaling Groups** → select `lab-app-asg`
2. **Details** tab → **Launch template** section → **Edit**
3. Version: select the new version number → **Update**

**Step 4 — Start Instance Refresh**
1. `lab-app-asg` → **Instance refresh** tab → **Start instance refresh**
2. Settings:
   - Minimum healthy percentage: `90`
   - Instance warmup: `300` seconds
   - **Enable checkpoints:** ✅
     - Checkpoint 1: 20% | Wait: 10 min
     - Checkpoint 2: 50% | Wait: 10 min
3. Click **Start instance refresh**

**Step 5 — Monitor Progress**
1. **Instance refresh** tab → watch the **Percentage complete** counter
2. Instances are replaced one at a time, ensuring 90% healthy throughout
3. Check ALB → **Target groups** → health stays **healthy** during replacement

**Step 6 — Cancel if Something Goes Wrong**
1. **Instance refresh** tab → **Cancel refresh** → instances revert to previous state

#### Validation Checklist
- [ ] New LT version created with updated AMI
- [ ] Instance refresh reaches 100% complete
- [ ] ALB target health never drops to 0 during refresh
- [ ] New instances show the updated app version

---

### LAB 13 — Spot Instances & Mixed Instances Policy

**Objective:** Reduce costs 60–80% using Spot Instances with intelligent fallback via ASG Console.

#### Step-by-Step (Console UI)

**Step 1 — Create Spot-Optimized ASG**
1. EC2 → **Auto Scaling Groups** → **Create Auto Scaling group**
2. Name: `lab-spot-asg`
3. Launch template: `lab-app-lt` | Version: `$Default` → **Next**
4. VPC: `lab-vpc` | Subnets: `lab-pri-a`, `lab-pri-b` → **Next**
5. Load balancer: attach to `lab-app-tg`

**Step 2 — Configure Mixed Instances Policy**
1. **Instance type requirements:** Choose **Manually add instance types**
2. Add these types: `t3.medium`, `t3a.medium`, `t2.medium`, `m5.large`, `m4.large`
3. Scroll to **Instance purchase options:**
   - **On-Demand base capacity:** `1` (always keep 1 On-Demand minimum)
   - **On-Demand percentage above base:** `20` (20% On-Demand, 80% Spot above base)
   - **Spot allocation strategy:** **Capacity optimized** ← picks least-likely-interrupted pools

**Step 3 — Configure Size**
- Min: `2` | Max: `20` | Desired: `4`
- Scaling: Target tracking CPU at 50%

**Step 4 — Handle Spot Interruption Notices**
Add this to your Launch Template user data to poll for interruption:

```
# Add to user data or run as a cron job
while true; do
  TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  SPOT_ACTION=$(curl -sf -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/spot/instance-action 2>/dev/null)
  if [ ! -z "$SPOT_ACTION" ]; then
    echo "Spot interruption received! Draining..."
    # flush queues, stop accepting new requests
  fi
  sleep 5
done
```

**Step 5 — Verify Spot vs On-Demand Mix**
1. `lab-spot-asg` → **Instance management** tab
2. Check the **Lifecycle** column — you'll see `Spot` and `Normal` (On-Demand) instances
3. **Activity** tab → scaling history shows which pools were selected

> 💡 **Tip:** `capacity-optimized` picks Spot pools least likely to be interrupted. Always prefer this over `lowest-price`.

#### Validation Checklist
- [ ] ASG shows mix of Spot and On-Demand instances in Instance management tab
- [ ] 1 On-Demand base always running
- [ ] Spot instances show `capacity-optimized` allocation strategy

---

### LAB 14 — ASG Lifecycle Hooks

**Objective:** Run custom actions before instances enter or leave service — configured via Console.

#### Step-by-Step (Console UI)

**Step 1 — Create IAM Role for Lifecycle Notifications**
1. IAM → **Roles** → **Create role**
2. Trusted entity: AWS service | Use case: **Auto Scaling**
3. Find and attach: `AmazonSNSFullAccess` (scoped down in production)
4. Name: `AutoScalingLifecycleRole` → **Create role**

**Step 2 — Create Launch Lifecycle Hook**
1. EC2 → **Auto Scaling Groups** → `lab-app-asg`
2. **Instance management** tab → **Lifecycle hooks** section → **Create lifecycle hook**
3. Settings:
   - **Lifecycle hook name:** `app-ready-hook`
   - **Lifecycle transition:** **Instance launch**
   - **Heartbeat timeout:** `300` seconds
   - **Default result:** `CONTINUE`
   - **Notification metadata:** `app_ready_check`
4. Click **Create lifecycle hook**

**Step 3 — Create Termination Lifecycle Hook**
1. **Create lifecycle hook** again:
   - Name: `graceful-shutdown-hook`
   - Lifecycle transition: **Instance termination**
   - Heartbeat timeout: `60` seconds
   - Default result: `CONTINUE`
2. Click **Create lifecycle hook**

**Step 4 — Complete the Hook in User Data**
Add to your Launch Template user data (instances wait in `Pending:Wait` until the hook is completed):

```
# Wait for application to be ready
until curl -sf http://localhost:8080/health; do sleep 5; done

# Get instance ID and ASG name
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

# Signal ASG that we're ready
aws autoscaling complete-lifecycle-action \
  --lifecycle-hook-name app-ready-hook \
  --auto-scaling-group-name lab-app-asg \
  --lifecycle-action-result CONTINUE \
  --instance-id $INSTANCE_ID \
  --region us-east-1
```

**Step 5 — Verify Hooks Working**
1. Trigger a scale-out event (set desired capacity to 3)
2. **Instance management** → new instance will show `Pending:Wait` state
3. After the hook completes, instance transitions to `InService` ✅

#### Validation Checklist
- [ ] Two lifecycle hooks created (Launch + Termination)
- [ ] New instances show `Pending:Wait` before becoming `InService`
- [ ] Terminating instances show `Terminating:Wait` before final termination

---

### LAB 15 — ALB Advanced: HTTPS, WAF, and Access Logs

**Objective:** Configure SSL/TLS via ACM, enable WAF, and set up access logs to S3.

#### Step-by-Step (Console UI)

**Step 1 — Request SSL Certificate via ACM**
1. Search → **Certificate Manager** (ACM)
2. **Request a certificate** → **Request a public certificate** → **Next**
3. **Fully qualified domain name:** `app.yourdomain.com`
4. Add another name: `*.yourdomain.com`
5. **Validation method:** DNS validation (recommended)
6. Click **Request**
7. In the certificate list, click your new cert → **Create records in Route 53** (auto-creates DNS entries)
8. Wait ~5 minutes for **Status** to change from `Pending validation` → **Issued** ✅

**Step 2 — Add HTTPS Listener to ALB**
1. EC2 → **Load Balancers** → `lab-alb` → **Listeners** tab → **Add listener**
2. Protocol: **HTTPS** | Port: **443**
3. Default action: **Forward** → select `lab-app-tg`
4. **Secure listener settings:**
   - Security policy: `ELBSecurityPolicy-TLS13-1-2-2021-06` (TLS 1.3 only)
   - Certificate source: From ACM → select your certificate
5. Click **Add**

**Step 3 — Enable WAF**
1. Search → **WAF & Shield**
2. **Create Web ACL**
3. Resource type: **Regional resources** | Region: us-east-1
4. Name: `lab-waf`
5. **Add AWS resources:** select `lab-alb`
6. **Add rules** → **Add managed rule groups**:
   - `AWSManagedRulesCommonRuleSet` (OWASP Top 10 protection)
   - `AWSManagedRulesKnownBadInputsRuleSet`
7. Default action: **Allow** → **Create Web ACL**

**Step 4 — Enable ALB Access Logs to S3**
1. Create S3 bucket first:
   - Search → **S3** → **Create bucket**
   - Name: `my-alb-logs-<your-account-id>` | Region: us-east-1
   - Block all public access: ✅ → **Create bucket**
2. Back to EC2 → **Load Balancers** → `lab-alb` → **Attributes** tab → **Edit**
3. **Monitoring:**
   - Access logs: Enable
   - S3 URI: `s3://my-alb-logs-<account-id>/lab-alb`
   - Deletion protection: **Enable** ✅
4. **Save changes**
5. Send some traffic to ALB → check S3 bucket for log files after ~5 min

> ⚠️ **Warning:** Always use TLS 1.3 policy (`ELBSecurityPolicy-TLS13-1-2-2021-06`). TLS 1.0/1.1 are deprecated and insecure.

#### Validation Checklist
- [ ] HTTPS listener shows `active` on port 443
- [ ] Browser shows green padlock (valid SSL cert) when visiting ALB
- [ ] HTTP traffic redirects to HTTPS
- [ ] WAF Web ACL associated with ALB
- [ ] Access logs appear in S3 bucket after traffic

---

## 7. 🔴 Advanced Labs (16–25)

---

### LAB 16 — EC2 Image Builder: Automated AMI Pipeline

**Objective:** Create a fully automated, versioned AMI pipeline using EC2 Image Builder (replaces manual Packer setup).

#### Step-by-Step (Console UI)

**Step 1 — Create an Image Builder Component (Install Script)**
1. Search → **EC2 Image Builder** → left sidebar → **Components** → **Create component**
2. Settings:
   - Type: **Build**
   - Name: `lab-app-install`
   - Version: `1.0.0`
   - Operating system: **Amazon Linux**
3. **Definition document** (YAML):

```yaml
name: InstallLabApp
description: Install httpd, git, jq, python3 and app dependencies
schemaVersion: 1.0

phases:
  - name: build
    steps:
      - name: InstallPackages
        action: ExecuteBash
        inputs:
          commands:
            - sudo yum update -y
            - sudo yum install -y httpd git jq python3 python3-pip
            - sudo pip3 install boto3 flask
            - sudo systemctl enable httpd
            - echo "Built by EC2 Image Builder on $(date)" | sudo tee /etc/build-info
  - name: validate
    steps:
      - name: CheckHttpd
        action: ExecuteBash
        inputs:
          commands:
            - httpd -v
```
4. Click **Create component**

**Step 2 — Create an Image Recipe**
1. Left sidebar → **Image recipes** → **Create image recipe**
2. Settings:
   - Name: `lab-app-recipe`
   - Version: `1.0.0`
   - Base image: **Amazon Linux 2023 x86_64** (select latest)
3. **Components:** Build components → **Browse build components** → select `lab-app-install`
4. **Storage (volumes):**
   - Root volume: Size `20` GiB | Type `gp3` | Encrypted ✅
5. Click **Create recipe**

**Step 3 — Create IAM Role for Image Builder**
1. IAM → **Roles** → **Create role**
2. Trusted entity: AWS service | Use case: **EC2**
3. Attach policies: `EC2InstanceProfileForImageBuilder`, `AmazonSSMManagedInstanceCore`
4. Name: `EC2ImageBuilderRole` → **Create role**
5. IAM → **Instance profiles** → create profile `EC2ImageBuilderProfile` → attach role

**Step 4 — Create Infrastructure Configuration**
1. Image Builder → **Infrastructure configurations** → **Create infrastructure configuration**
2. Name: `lab-infra-config`
3. IAM role: `EC2ImageBuilderRole`
4. Instance type: `t3.medium`
5. VPC, subnet: choose a **public** subnet (builder needs internet) | Security group: allow outbound HTTPS
6. Click **Create infrastructure configuration**

**Step 5 — Create Image Pipeline**
1. Left sidebar → **Image pipelines** → **Create image pipeline**
2. **Pipeline details:**
   - Name: `lab-app-pipeline`
   - Build schedule: **Weekly** (or Manual for labs)
3. **Recipe:** select `lab-app-recipe v1.0.0`
4. **Infrastructure:** select `lab-infra-config`
5. **Distribution:** select us-east-1 (default) → add us-west-2 for multi-region distribution
6. Click **Create pipeline**

**Step 6 — Run the Pipeline**
1. Select `lab-app-pipeline` → **Actions** → **Run pipeline**
2. Go to **Image pipelines** → click your pipeline → **Output images** tab
3. Watch the status: `Building` → `Testing` → `Distributing` → `Available` (~15–20 min)
4. Copy the AMI ID from **Output images** — update your Launch Template to use it

#### Validation Checklist
- [ ] Pipeline completes with status **Available**
- [ ] New AMI appears in EC2 → AMIs (My AMIs)
- [ ] Launch instance from the new AMI and verify `/etc/build-info` shows the builder timestamp

---

### LAB 17 — CloudFormation: IaC via Console

**Objective:** Deploy the full EC2 + ASG + ALB stack using AWS CloudFormation entirely via Console.

#### Step-by-Step (Console UI)

**Step 1 — Create the CloudFormation Template**
Create a file named `lab17-stack.yaml` on your local machine:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Lab 17 - EC2 + ASG + ALB Stack

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues: [t3.micro, t3.medium, t3.large]
  DesiredCapacity:
    Type: Number
    Default: 2
  AMIId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64

Resources:
  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: App Security Group
      VpcId: !ImportValue lab-vpc-id
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  AppLaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: cfn-lab-app-lt
      LaunchTemplateData:
        ImageId: !Ref AMIId
        InstanceType: !Ref InstanceType
        SecurityGroupIds:
          - !Ref AppSecurityGroup
        MetadataOptions:
          HttpTokens: required
          HttpEndpoint: enabled
        Monitoring:
          Enabled: true

  AppASG:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: cfn-lab-app-asg
      LaunchTemplate:
        LaunchTemplateId: !Ref AppLaunchTemplate
        Version: !GetAtt AppLaunchTemplate.LatestVersionNumber
      MinSize: '1'
      MaxSize: '6'
      DesiredCapacity: !Ref DesiredCapacity
      VPCZoneIdentifier:
        - !ImportValue lab-pri-subnet-a
        - !ImportValue lab-pri-subnet-b

Outputs:
  ASGName:
    Value: !Ref AppASG
    Export:
      Name: lab-asg-name
```

**Step 2 — Deploy via CloudFormation Console**
1. Search → **CloudFormation** → **Stacks** → **Create stack**
2. **Prerequisite - Prepare template:** Upload a template file → **Choose file** → select `lab17-stack.yaml`
3. Click **Next**
4. Stack name: `lab17-ec2-asg-stack`
5. Parameters:
   - InstanceType: `t3.micro`
   - DesiredCapacity: `2`
6. Click **Next** → accept IAM capabilities checkbox → **Submit**

**Step 3 — Monitor Deployment**
1. On the **Events** tab, watch resources being created in order
2. Status transitions: `CREATE_IN_PROGRESS` → `CREATE_COMPLETE` ✅
3. **Resources** tab shows all created resources with links
4. **Outputs** tab shows exported values

**Step 4 — Update the Stack**
1. Select `lab17-ec2-asg-stack` → **Update**
2. Select **Use existing template** → **Next**
3. Change DesiredCapacity to `3` → **Next** → **Next** → **Submit**
4. CloudFormation performs rolling update with no downtime

**Step 5 — Delete the Stack**
1. Select stack → **Delete** → **Delete stack**
2. All resources are cleaned up automatically in correct dependency order

#### Validation Checklist
- [ ] Stack reaches `CREATE_COMPLETE`
- [ ] EC2 instances launched via ASG appear in Instances list
- [ ] Stack update (DesiredCapacity change) completes without errors
- [ ] Stack deletion removes all resources cleanly

---

### LAB 18 — CodePipeline for Automated AMI Refresh

**Objective:** Build a CI/CD pipeline that auto-updates the Launch Template when a new AMI is published.

#### Step-by-Step (Console UI)

**Step 1 — Create an S3 Bucket for Pipeline Artifacts**
1. S3 → **Create bucket**
2. Name: `lab18-pipeline-artifacts-<account-id>` | Region: us-east-1
3. Versioning: Enable → **Create bucket**

**Step 2 — Create the CodePipeline**
1. Search → **CodePipeline** → **Create pipeline**
2. Name: `lab-ami-refresh-pipeline`
3. Service role: **New service role** (auto-created) → **Next**
4. **Source stage:**
   - Source provider: **GitHub (Version 2)** (or **CodeCommit** for AWS-only)
   - Connect to GitHub → select your repo and branch (e.g., `main`)
   - Click **Next**
5. **Build stage** (using CodeBuild):
   - Provider: **AWS CodeBuild** → **Create project** (opens new tab)

**Step 3 — Create CodeBuild Project**
1. Project name: `lab-ami-builder`
2. Environment:
   - Managed image | OS: **Amazon Linux** | Runtime: **Standard** | Image: latest
   - Privileged: ✅ (needed for Docker-based builds if any)
3. Service role: **New service role**
4. **Buildspec**: Use a buildspec file (`buildspec.yml` in your repo root):

```yaml
version: 0.2
phases:
  install:
    commands:
      - echo "Starting AMI build..."
  build:
    commands:
      - aws ec2 create-launch-template-version
          --launch-template-name lab-app-lt
          --version-description "Pipeline build $CODEBUILD_BUILD_NUMBER"
          --source-version '$Latest'
          --launch-template-data "{\"UserData\":\"$(base64 -w0 scripts/bootstrap.sh)\"}"
  post_build:
    commands:
      - aws autoscaling start-instance-refresh
          --auto-scaling-group-name lab-app-asg
          --preferences '{"MinHealthyPercentage":90}'
      - echo "Instance refresh started"
```
5. Click **Continue to CodePipeline**

6. Back in CodePipeline → **Skip deploy stage** (our build stage handles deployment)
7. Review → **Create pipeline**

**Step 4 — Trigger and Monitor**
1. Push a commit to your GitHub repo → pipeline triggers automatically
2. CodePipeline console → watch each stage: Source → Build
3. Green checkmarks = success ✅
4. Back in EC2 → ASG → **Instance refresh** tab → shows an active refresh

#### Validation Checklist
- [ ] Pipeline created with Source + Build stages
- [ ] Commit to repo triggers pipeline automatically
- [ ] Build stage creates new LT version and starts instance refresh
- [ ] Instances show updated configuration after refresh

---

### LAB 19 — EBS Multi-Attach & Shared Storage Patterns

**Objective:** Create io2 Multi-Attach volumes and set up EFS for cross-AZ shared storage via Console.

#### Step-by-Step (Console UI)

**Part A — EBS io2 Multi-Attach**

**Step 1 — Create io2 Multi-Attach Volume**
1. EC2 → **Volumes** → **Create volume**
2. Settings:
   - Volume type: **Provisioned IOPS SSD (io2)**
   - Size: `100` GiB
   - IOPS: `10,000`
   - Availability Zone: `us-east-1a` ← all instances using this must be in 1a
   - **Multi-Attach enabled:** ✅
   - Encrypted: ✅
3. Click **Create volume**

**Step 2 — Attach to Multiple Instances**
1. Launch 2 EC2 instances in **us-east-1a** only
2. Select volume → **Actions** → **Attach volume** → Instance 1 → device `/dev/sdf`
3. Select volume → **Actions** → **Attach volume** → Instance 2 → device `/dev/sdf`
4. Verify volume **State** changes to `in-use` with 2 attachments

> ⚠️ **Warning:** Multi-Attach requires a **cluster-aware filesystem** (GFS2, OCFS2). Using ext4 on Multi-Attach causes data corruption. For most shared storage needs, use EFS instead.

**Part B — Amazon EFS (Recommended for Shared Storage)**

**Step 3 — Create EFS File System**
1. Search → **EFS** → **Create file system**
2. Name: `lab-shared-efs`
3. VPC: `lab-vpc`
4. **Customize:**
   - Performance mode: **General Purpose**
   - Throughput mode: **Elastic** (scales automatically)
   - Encryption at rest: ✅ (KMS: default)
5. **Network:** Add mount targets in both `lab-pri-a` and `lab-pri-b`
   - Security group: create `sg-efs` that allows NFS (port 2049) from `sg-app`
6. Click **Create** → note the File system ID (e.g., `fs-0abc123`)

**Step 4 — Mount EFS on EC2 Instances**
Connect to each instance via Session Manager, then:

```
sudo yum install -y amazon-efs-utils
sudo mkdir -p /mnt/efs
sudo mount -t efs fs-XXXXXXXX:/ /mnt/efs
echo "Hello from $(hostname)" | sudo tee /mnt/efs/shared.txt
cat /mnt/efs/shared.txt   # verify from another instance
```

#### Storage Comparison

| Feature | EBS Single (gp3) | EBS Multi-Attach (io2) | EFS |
|---|---|---|---|
| # Instances | 1 per volume | Up to 16, same AZ | Thousands |
| Cross-AZ | No | No | Yes (regional) |
| Protocol | Block (NVMe) | Block (NVMe) | NFS v4 |
| Cost | $0.08/GB-month | $0.125/GB-month | $0.30/GB-month |
| Use Case | Most workloads | Clustered DBs | Shared app files |

#### Validation Checklist
- [ ] io2 volume shows Multi-Attach enabled with 2 attachments
- [ ] EFS file system has mount targets in both AZs
- [ ] File written on one EC2 is visible from another EC2

---

### LAB 20 — Warm Pools & Predictive Scaling

**Objective:** Reduce scale-out latency with Warm Pools and enable ML-based Predictive Scaling via Console.

#### Step-by-Step (Console UI)

**Part A — Warm Pools**

**Step 1 — Configure Warm Pool**
1. EC2 → **Auto Scaling Groups** → `lab-app-asg`
2. **Advanced configurations** tab → **Warm pool** section → **Create warm pool**
3. Settings:
   - Warm pool size: **Minimum pool size** = `2`
   - **Maximum prepared capacity:** Limit to group maximum capacity (checked)
   - Instance reuse policy: **Return instances to warm pool on scale in**
   - **Instances in warm pool state:** Stopped ← cheapest; pre-initialized but no CPU billing
4. Click **Create**
5. **Instance management** tab → you'll now see warm pool instances with lifecycle state `Warmed:Stopped`

**Step 2 — Verify Scale-Out Speed**
1. Increase desired capacity from 2 → 4
2. Watch the **Instance management** tab — warm pool instances transition `Warmed:Stopped` → `Pending` → `InService` (much faster than cold launch, ~30s vs ~3min)

**Part B — Predictive Scaling**

**Step 3 — Enable Predictive Scaling Policy**
1. `lab-app-asg` → **Automatic scaling** tab → **Create dynamic scaling policy**
2. Policy type: **Predictive scaling**
3. Scaling policy name: `predictive-scaling`
4. Metrics:
   - **Predefined metric pair:** CPU utilization
   - Target value: `50`
5. **Scaling mode:** Forecast and scale (fully automatic)
6. **Pre-launch instances:** `5` minutes before predicted load
7. Click **Create**

**Step 4 — View the Forecast**
1. Select `lab-app-asg` → **Automatic scaling** → **Predictive scaling** tab
2. After 24+ hours of traffic data, a forecast graph appears showing predicted load
3. The ASG pre-scales before peak load arrives ✅

> 💡 **Note:** Predictive Scaling needs at least 24 hours of CloudWatch data to generate forecasts. Run some synthetic load first.

#### Validation Checklist
- [ ] Warm pool shows instances in `Warmed:Stopped` state
- [ ] Scale-out event pulls from warm pool (faster than cold launch)
- [ ] Predictive scaling policy created (forecast visible after 24h)

---

### LAB 21 — Cost Optimization Deep Dive

**Objective:** Reduce EC2 costs using Compute Optimizer, Savings Plans, Spot, and right-sizing via Console.

#### Strategy Overview

| Strategy | Savings vs On-Demand | Commitment | Best For |
|---|---|---|---|
| On-Demand | 0% (baseline) | None | Unpredictable, short-term |
| Savings Plans (Compute) | Up to 66% | 1 or 3 year | Flexible — any family/region |
| Reserved Instances (Standard) | Up to 72% | 1 or 3 year | Steady-state, specific type |
| Spot Instances | Up to 90% | None (interruptible) | Batch, stateless, fault-tolerant |
| Graviton (ARM) | Up to 40% vs x86 | None | Most Linux workloads |

#### Step-by-Step (Console UI)

**Step 1 — Enable AWS Compute Optimizer**
1. Search → **Compute Optimizer** → **Get started** or **Opt in**
2. Select **All accounts** (if using AWS Organizations) or **Current account**
3. Click **Opt in**
4. Wait 24–48 hours for analysis → then view recommendations under **EC2 instances**

**Step 2 — Review Right-Sizing Recommendations**
1. Compute Optimizer → **EC2 instances**
2. Look for findings: **Over-provisioned** (can downsize) or **Under-provisioned** (needs upgrade)
3. Click any instance → see recommended type with projected savings

**Step 3 — Find Cost Waste with Cost Explorer**
1. Search → **Cost Explorer** → **Launch Cost Explorer**
2. **Service filter:** EC2-Instances → group by **Instance Type**
3. Look for large spend on instance types you might replace with Spot or Graviton

**Step 4 — Find Unused Resources**
1. EC2 → **Volumes** → filter Status = **available** (unattached volumes, still costing money!)
   - Select unneeded ones → **Actions** → **Delete volume**
2. EC2 → **Elastic IPs** → any unassociated EIPs cost $0.005/hr
   - Select unneeded → **Actions** → **Release Elastic IP address**
3. EC2 → **Snapshots** → filter by creation date to find old snapshots
   - Delete snapshots older than 90 days that have no associated AMI

**Step 5 — Switch gp2 Volumes to gp3**
1. EC2 → **Volumes** → filter by Type = `gp2`
2. Select a gp2 volume → **Actions** → **Modify volume**
3. Volume type: **gp3** | IOPS: `3000` | Throughput: `125 MB/s`
4. Click **Modify** ← no downtime required, online modification ✅
5. gp3 is ~20% cheaper than gp2 with better baseline performance

**Step 6 — Purchase a Savings Plan**
1. Search → **Savings Plans** → **Purchase Savings Plans**
2. **Plan type:** Compute Savings Plan (most flexible, applies to any region/family)
3. **Term:** 1 Year | **Payment option:** No Upfront (or Partial/All Upfront for more savings)
4. **Hourly commitment:** enter the baseline EC2 spend you're confident about
5. Click **Add to cart** → **Review and purchase**

#### Cost Optimization Checklist
- [ ] Compute Optimizer enabled and showing recommendations
- [ ] No unattached EBS volumes (all `available` volumes deleted or snapshotted)
- [ ] No unassociated Elastic IPs
- [ ] All gp2 volumes converted to gp3
- [ ] Savings Plan purchased for baseline compute spend
- [ ] Spot instances used for ASG burst capacity (Lab 13)

---

### LAB 22 — Multi-Region Active-Active Architecture

**Objective:** Deploy identical stacks in 2 regions with Route 53 latency-based routing.

#### Architecture
```
Users (Global)
    │
    ▼
Route 53 (Latency-Based Routing)
    ├── us-east-1 → ALB → ASG → Aurora (Primary)
    └── ap-south-1 → ALB → ASG → Aurora (Replica)
```

#### Step-by-Step (Console UI)

**Step 1 — Deploy Second Region Stack**
1. Switch region to **Asia Pacific (Mumbai) ap-south-1** using the top-right dropdown
2. Repeat Labs 04, 05, 08, 09, 10 in ap-south-1 to create identical VPC + ASG + ALB
3. Use the same naming convention with suffix: `lab-vpc-mumbai`, `lab-alb-mumbai`
4. Copy your AMI to Mumbai first:
   - EC2 (us-east-1) → **AMIs** → select your AMI → **Actions** → **Copy AMI** → Region: ap-south-1

**Step 2 — Set Up Route 53 Hosted Zone**
1. Search → **Route 53** → **Hosted zones** → **Create hosted zone**
2. Domain name: `yourdomain.com` | Type: **Public hosted zone**
3. **Create hosted zone** → note the 4 nameservers (update your domain registrar)

**Step 3 — Create Latency Records for Both Regions**
1. **Hosted zones** → select your zone → **Create record**
2. Record for us-east-1:
   - Record name: `app`
   - Record type: **A**
   - **Routing policy:** Latency
   - Region: **us-east-1**
   - Value/Route traffic to: **Alias to Application and Classic Load Balancer** → us-east-1 → select `lab-alb`
   - Health check: **Create health check** → monitor ALB DNS on port 80 path `/health`
   - Record ID: `us-east-1-primary`
3. Create record for ap-south-1:
   - Same settings, Region: **ap-south-1**, target: Mumbai ALB
   - Health check: same setup for Mumbai ALB
   - Record ID: `ap-south-1-secondary`
4. Click **Create records**

**Step 4 — Test Latency Routing**
1. Route 53 → **Health checks** → verify both show **Healthy** ✅
2. Use an online tool like `dnschecker.org` to query `app.yourdomain.com` from different regions
3. Users in India resolve to Mumbai ALB; users in US resolve to us-east-1 ALB

**Step 5 — Test Failover**
1. Stop all instances in us-east-1 ASG (desired = 0)
2. Health check for us-east-1 should flip to **Unhealthy**
3. All traffic automatically routes to Mumbai ✅ (verify with DNS checker)

#### Validation Checklist
- [ ] Identical ASG + ALB stack running in both regions
- [ ] Route 53 latency records created with health checks
- [ ] Health checks show Healthy for both regions
- [ ] Disabling us-east-1 causes traffic to route to Mumbai

---

### LAB 23 — Security Hardening & Compliance

**Objective:** Harden EC2 to CIS Benchmark Level 1. Implement Inspector, GuardDuty, and automated patching.

#### Security Control Map

| Control | Implementation | Service |
|---|---|---|
| OS Patching | SSM Patch Manager — auto-approval, maintenance window | Systems Manager |
| Vulnerability Scanning | Inspector v2 — continuous CVE scanning | Inspector |
| Threat Detection | GuardDuty — ML on VPC flow logs + DNS | GuardDuty |
| Config Compliance | Config rules — SG encryption, IMDSv2 checks | AWS Config |
| Secret Rotation | Secrets Manager — auto-rotate DB creds every 30 days | Secrets Manager |
| Audit Trail | CloudTrail — all API calls logged to S3 | CloudTrail |

#### Step-by-Step (Console UI)

**Step 1 — Enable GuardDuty**
1. Search → **GuardDuty** → **Get started** → **Enable GuardDuty**
2. Findings publishing frequency: **Six hours** → **Enable**
3. After ~10 minutes, GuardDuty begins analyzing VPC Flow Logs, DNS logs, and CloudTrail

**Step 2 — Enable Amazon Inspector v2**
1. Search → **Inspector** → **Get started** → **Enable Inspector**
2. Scan types: Select **EC2 scanning** and **ECR container image scanning**
3. Click **Enable** → Inspector begins scanning all running EC2 instances for CVEs
4. View findings under **Findings** → filter by Severity: Critical, High

**Step 3 — Create SSM Patch Baseline**
1. Search → **Systems Manager** → **Patch Manager** → **Patch baselines** → **Create patch baseline**
2. Settings:
   - Name: `AmazonLinux2023-SecurityBaseline`
   - Operating system: **Amazon Linux 2023**
3. **Auto-approval rules:**
   - Products: All | Severity: **Critical** and **High** | Auto-approve after: `0` days
4. Click **Create patch baseline**
5. **Actions** → **Set as default baseline** for Amazon Linux 2023

**Step 4 — Create Maintenance Window**
1. Systems Manager → **Maintenance Windows** → **Create maintenance window**
2. Settings:
   - Name: `Sunday-Security-Patching`
   - Schedule: `cron(0 2 ? * SUN *)` (Sundays at 2 AM UTC)
   - Duration: `4` hours | Stop initiating: `1` hour before end
3. **Create maintenance window**
4. **Targets** tab → **Register targets** → target type: **Resource group** or **Tag** (select your EC2 instances by tag `Name=asg-app-server`)
5. **Tasks** tab → **Register tasks** → **Register Run command task**:
   - Document: `AWS-RunPatchBaseline` | Operation: **Install**
6. **Register task**

**Step 5 — Enforce IMDSv2 on Existing Instances**
1. EC2 → **Instances** → select all running instances
2. **Actions** → **Instance settings** → **Modify instance metadata options**
3. IMDSv2: **Required** → **Save** (applies to all selected at once)

**Step 6 — Enable AWS Config**
1. Search → **Config** → **Get started** → **1-click setup** → **Confirm**
2. Config → **Rules** → **Add rule** → search for:
   - `ec2-imdsv2-check` → Add (flags instances not using IMDSv2)
   - `encrypted-volumes` → Add (flags unencrypted EBS volumes)
   - `restricted-ssh` → Add (flags SGs with 0.0.0.0/0 on port 22)
3. Each rule shows compliant/non-compliant resources

#### Validation Checklist
- [ ] GuardDuty enabled and showing no critical findings
- [ ] Inspector active and scanning — check for findings
- [ ] Patch baseline created and set as default for Amazon Linux 2023
- [ ] Maintenance window scheduled for Sundays
- [ ] All instances show IMDSv2 = Required in console
- [ ] AWS Config shows 3 rules with compliance status

---

### LAB 24 — Chaos Engineering with AWS FIS

**Objective:** Deliberately inject failures and verify system resilience using Fault Injection Simulator.

#### Experiment Design

| Experiment | Fault Injected | Expected Behavior | Success Criteria |
|---|---|---|---|
| Terminate random instance | Terminate 1 EC2 in ASG | ASG replaces within 5 min | HealthyHostCount never drops to 0 |
| CPU stress | 80% CPU for 10 min | Scale-out at 50% avg CPU | New instances healthy within 7 min |
| AZ failure | Stop all instances in 1 AZ | Traffic shifts to other AZ | Zero downtime, < 30s reroute |

#### Step-by-Step (Console UI)

**Step 1 — Create IAM Role for FIS**
1. IAM → **Roles** → **Create role**
2. Trusted entity: AWS service | Service: **FIS (Fault Injection Simulator)**
3. Attach policy: `PowerUserAccess` (scope down in production)
4. Name: `FISRole` → **Create role**

**Step 2 — Create Experiment: Terminate 1 Instance**
1. Search → **FIS** → **Experiment templates** → **Create experiment template**
2. Description: `Terminate 1 EC2 in ASG`
3. **Actions:**
   - Action name: `terminate-ec2-instance`
   - Action type: `aws:ec2:terminate-instances`
4. **Targets:**
   - Name: `ec2-instances`
   - Resource type: `aws:ec2:instance`
   - Resource tags: `aws:autoscaling:groupName = lab-app-asg`
   - Selection mode: `COUNT(1)` ← terminates exactly 1 instance
5. **Stop conditions:**
   - Source: CloudWatch alarm
   - Value: ARN of your `unhealthy-hosts` alarm (from Lab 11) — stops experiment if too many hosts fail
6. **Role:** `FISRole`
7. Click **Create experiment template**

**Step 3 — Run the Experiment**
1. Open two browser tabs:
   - Tab 1: FIS → your experiment
   - Tab 2: CloudWatch → `EC2-Lab-Dashboard` (from Lab 11)
2. FIS → select template → **Actions** → **Start experiment** → type `start` to confirm
3. Watch Tab 2: one instance goes unhealthy → ASG launches replacement → health restores ✅

**Step 4 — Create Experiment: CPU Stress**
1. **Create experiment template**
2. Action: `aws:ssm:send-command`
3. Document ARN: `arn:aws:ssm:us-east-1::document/AWSFIS-Run-CPU-Stress`
4. Parameters: `{"DurationSeconds": ["600"], "CPU": ["0"]}` (0 = all CPUs)
5. Target: `COUNT(2)` instances in the ASG
6. Run → watch CloudWatch CPU metric spike → ASG scale-out triggers ✅

**Step 5 — Create Experiment: AZ Failure Simulation**
1. Action: `aws:ec2:stop-instances`
2. Target: instances with tag `aws:autoscaling:availability-zone = us-east-1a`
3. Run → verify:
   - ALB stops routing to 1a instances
   - 1b instances continue serving traffic
   - ASG launches replacements in 1b
4. **Duration:** 5 minutes → stop → 1a instances resume

#### Validation Checklist
- [ ] Instance termination experiment: ASG replaces within 5 minutes
- [ ] CPU stress experiment: scale-out triggers and new instances become healthy
- [ ] AZ failure experiment: zero 5xx errors visible in ALB metrics
- [ ] All experiments show Stop conditions working correctly

---

### LAB 25 — Full Production Stack (Capstone Project)

**Objective:** Deploy a complete, production-grade 3-tier application using all skills from Labs 1–24.

#### Architecture

| Tier | Services | Configuration |
|---|---|---|
| CDN / DNS | Route 53 + CloudFront | Latency routing, SSL, cache behaviors |
| Load Balancer | ALB + WAF + Shield Standard | HTTPS only, WAF rules, DDoS protection |
| Web / App Tier | EC2 (c6g.large Graviton) + ASG | Min 2 / Max 20 per AZ, Spot 80% / OD 20% |
| Database | Aurora MySQL Serverless v2 | Writer in 1a, reader in 1b |
| Cache | ElastiCache Redis | Multi-AZ, session storage |
| Storage | EFS (shared) + S3 (assets) | EFS for uploads, S3 for static |
| Secrets | Secrets Manager + SSM | All credentials in Secrets Manager |
| Monitoring | CloudWatch + Grafana Cloud | Unified dashboards, PagerDuty on-call |
| Security | Inspector + GuardDuty + Config | Continuous compliance and threat detection |
| IaC | CloudFormation | GitOps style review and apply |

#### Step-by-Step (Console UI — High Level)

**Step 1 — Foundation**
- Complete Lab 04 VPC (custom VPC with 4 subnets)
- Complete Lab 05 Security Groups (all 4 SGs)
- Complete Lab 06 SSM Role

**Step 2 — Compute**
- Complete Lab 08 Launch Template (Graviton: change instance type to `c6g.large`, AMI to ARM AMI)
- Complete Lab 13 Spot ASG (80% Spot, 20% On-Demand)
- Complete Lab 14 Lifecycle Hooks

**Step 3 — Traffic**
- Complete Lab 09 ALB
- Complete Lab 15 HTTPS + WAF + Access Logs

**Step 4 — Automation**
- Complete Lab 16 EC2 Image Builder pipeline
- Complete Lab 20 Warm Pools + Predictive Scaling

**Step 5 — Observability**
- Complete Lab 11 CloudWatch Alarms + Dashboard
- Add CloudWatch Logs Insights queries for error analysis

**Step 6 — Security**
- Complete Lab 23 GuardDuty + Inspector + Patch Manager + Config

**Step 7 — Resilience Testing**
- Complete Lab 24 FIS experiments
- Verify: AZ failure → zero downtime, < 30s reroute

#### Delivery Checklist
- [ ] All infrastructure deployed via Console (documented and reproducible via CloudFormation)
- [ ] AMI baking automated via EC2 Image Builder
- [ ] ASG uses Spot + On-Demand mix with warm pools
- [ ] Zero-downtime deployment via Instance Refresh (Lab 12)
- [ ] CloudWatch dashboard covering all key metrics
- [ ] Load test: 1,000 concurrent users, P99 < 500ms
- [ ] Chaos experiment passed: AZ failure with zero downtime
- [ ] Monthly cost estimate documented with optimization plan
- [ ] Security scan clean: Inspector shows no critical CVEs
- [ ] Disaster recovery tested: full stack recovery < 30 minutes

---

## 8. ⚫ Expert Labs (26–40)

---

### LAB 26 — EC2 Instance Connect Endpoint

**Objective:** Connect to EC2 instances in private subnets without a bastion host, NAT gateway, or open security group ports — using EC2 Instance Connect Endpoint.

#### What You Will Learn
- Create an Instance Connect Endpoint in a private subnet
- Connect to private EC2 instances directly from the browser console
- Remove all inbound SSH rules from security groups

#### Step-by-Step (Console UI)

**Step 1 — Create the Instance Connect Endpoint**
1. EC2 Console → left sidebar → **Network & Security** → **EC2 Instance Connect Endpoints**
2. Click **Create EC2 Instance Connect Endpoint**
3. Settings:
   - VPC: `lab-vpc`
   - Subnet: `lab-pri-a` ← choose one private subnet
   - Security group: **Create new security group**:
     - Name: `sg-eice` | Description: `EC2 Instance Connect Endpoint`
     - Inbound: **none** ← no inbound needed
     - Outbound: TCP 22 to `sg-app` (allow EICE to reach app instances)
   - Preserve client IP: **Enable** (optional, for audit logging)
4. Click **Create endpoint**
5. Wait for Status: `create-complete` ✅

**Step 2 — Update App Security Group**
1. EC2 → **Security Groups** → `sg-app`
2. **Inbound rules** → **Edit inbound rules**
3. **Delete** the SSH rule that allowed port 22 from `sg-bastion`
4. Add: **SSH (22)** | Source: `sg-eice` ← only from the EICE
5. **Save rules**
6. ✅ You can now delete the bastion host entirely

**Step 3 — Connect to a Private Instance**
1. EC2 → **Instances** → select a private instance (one in `lab-pri-a` or `lab-pri-b`)
2. Click **Connect** → **EC2 Instance Connect** tab
3. Connection type: **Connect using EC2 Instance Connect Endpoint**
4. Instance Connect Endpoint: select your endpoint
5. Username: `ec2-user`
6. Click **Connect** ← browser terminal opens to your private instance ✅

**Step 4 — Audit Connections**
1. CloudTrail → **Event history** → filter by Event name: `OpenTunnel`
2. See who connected, when, and which instance

#### Validation Checklist
- [ ] EICE status shows `create-complete`
- [ ] No public IP required on instance
- [ ] No bastion host needed
- [ ] Browser SSH connection opens to private instance

---

### LAB 27 — Placement Groups: Cluster, Spread, and Partition

**Objective:** Understand and configure EC2 Placement Groups for different performance and availability requirements.

#### Placement Group Types

| Type | Network Performance | Fault Isolation | Use Case |
|---|---|---|---|
| Cluster | 10–25 Gbps (lowest latency) | Low — single rack | HPC, ML training, tightly-coupled |
| Spread | Standard | High — separate racks (max 7/AZ) | Small critical instances, HA databases |
| Partition | High | Medium — separate partitions | Kafka, Cassandra, HDFS, large distributed |

#### Step-by-Step (Console UI)

**Step 1 — Create a Cluster Placement Group**
1. EC2 → left sidebar → **Network & Security** → **Placement Groups**
2. **Create placement group**
3. Name: `lab-cluster-pg`
4. Strategy: **Cluster**
5. **Create group**

**Step 2 — Create a Spread Placement Group**
1. **Create placement group**
2. Name: `lab-spread-pg`
3. Strategy: **Spread**
4. Spread level: **Rack** (host-level available for Dedicated Hosts)
5. **Create group**

**Step 3 — Create a Partition Placement Group**
1. **Create placement group**
2. Name: `lab-partition-pg`
3. Strategy: **Partition**
4. Number of partitions: `3` (maps to independent server racks)
5. **Create group**

**Step 4 — Launch Instances into Each Group**
1. EC2 → **Launch instances** → configure normally
2. In **Advanced details** → **Placement group name**: select `lab-cluster-pg`
3. Launch 3 instances into the cluster group
4. Repeat for spread group (max 7 per AZ, each on separate rack hardware)
5. Repeat for partition group — assign instances to partitions 1, 2, 3

**Step 5 — View Partition Information**
1. Instances → select an instance in partition group
2. **Details** tab → **Placement group** → shows group name and **Partition number**
3. This tells you which hardware partition (rack) the instance runs on

**Step 6 — Add Existing Instance to Placement Group**
1. Stop the instance first
2. **Actions** → **Instance settings** → **Change placement group** → select group → **Apply**
3. Start the instance

#### Validation Checklist
- [ ] 3 placement groups created (cluster, spread, partition)
- [ ] Cluster group instances have lower latency between them than across AZs
- [ ] Spread group shows each instance on a different host in the details
- [ ] Partition group shows partition number 1, 2, 3 per instance

---

### LAB 28 — EBS Elastic Volumes: Online Resize & Type Change

**Objective:** Resize EBS volumes and change volume types while the instance is running — zero downtime.

#### Step-by-Step (Console UI)

**Step 1 — Increase Volume Size (Online)**
1. EC2 → **Volumes** → select your root or data volume
2. **Actions** → **Modify volume**
3. Change:
   - Volume type: `gp3` (if not already)
   - Size: Increase from `20` → `50` GiB
   - IOPS: `3000` → `6000` (optional)
   - Throughput: `125` → `250` MB/s (optional)
4. Click **Modify** → **Modify** again to confirm
5. Volume state shows `optimizing` → then `in-use` (no downtime!)

**Step 2 — Extend the Filesystem (Required After Resize)**
Connect to your instance via Session Manager:

```
lsblk                                    # confirm new volume size visible
sudo growpart /dev/nvme0n1 1             # extend the partition
sudo resize2fs /dev/nvme0n1p1            # extend ext4 filesystem
df -h /                                  # confirm new size
```
For XFS filesystem (Amazon Linux 2023):
```
sudo xfs_growfs /
```

**Step 3 — Change Volume Type (gp2 → gp3)**
1. Select a gp2 volume → **Actions** → **Modify volume**
2. Volume type: **gp3**
3. Keep size the same; set IOPS = `3000`, Throughput = `125`
4. **Modify** ← no downtime, filesystem stays mounted ✅
5. Savings: gp3 is ~20% cheaper than gp2

**Step 4 — Upgrade to Provisioned IOPS**
1. For a database volume: **Modify volume** → type: `io2`
2. Set IOPS: `10,000` for consistent DB performance
3. **Modify** — while volume is in `optimizing` state, it still works normally

**Step 5 — Monitor Modification Progress**
1. In the Volumes list, the **State** column shows `optimizing (17%)`
2. Can take minutes to hours for large volumes
3. CloudWatch metric: `VolumeModificationState` tracks progress

#### Validation Checklist
- [ ] Volume resized from 20 GiB to 50 GiB with no instance downtime
- [ ] Filesystem extended to use new space (`df -h` shows new size)
- [ ] gp2 volume converted to gp3 while mounted
- [ ] IOPS increased without detaching volume

---

### LAB 29 — EC2 Fleet with Mixed Instances

**Objective:** Create an EC2 Fleet to launch diverse instance types across Spot and On-Demand pools in a single request.

#### Step-by-Step (Console UI)

**Step 1 — Create an EC2 Fleet**
1. EC2 Console → left sidebar → **Fleet** → **Create fleet**
2. **Fleet settings:**
   - Fleet type: **request** (one-time) or **maintain** (keeps target capacity)
   - Select **maintain** to keep fleet capacity over time
3. **Instance type requirements:**
   - Click **Set instance type requirements**
   - vCPUs: min `2`, max `8`
   - Memory: min `4` GiB, max `16` GiB
   - Click **Preview instance types** → AWS shows eligible types matching requirements

**Step 2 — Configure Purchase Options**
1. **Target capacity:** `10` instances
2. **On-Demand and Spot allocation:**
   - On-Demand base: `2`
   - Additional capacity: `80%` Spot, `20%` On-Demand
3. **Spot settings:**
   - Allocation strategy: **Capacity optimized** ← least interruption
4. **On-Demand settings:**
   - Allocation strategy: **Lowest price**

**Step 3 — Configure Launch Template**
1. **Launch template:** select `lab-app-lt`
2. Version: `$Default`
3. **Overrides:** add multiple instance types:
   - `t3.medium` | `t3a.medium` | `m5.large` | `m4.large` | `m5a.large`
4. **Subnets:** select `lab-pri-a` and `lab-pri-b`

**Step 4 — Additional Settings**
1. **Auto Scaling:** Link to ASG (optional)
2. **Replace unhealthy instances:** ✅
3. **Terminate instances on deletion:** ✅ (clean up when fleet is deleted)
4. **Tags:** Name = `fleet-instance`
5. Click **Create fleet** → Fleet ID shown ✅

**Step 5 — Monitor Fleet**
1. **Fleets** list → click your fleet ID
2. **Fleet summary** tab: shows fulfilled On-Demand + Spot counts
3. **Instance** tab: all instances with type and lifecycle (Spot/On-Demand)
4. **History** tab: events when instances were launched or replaced

#### Validation Checklist
- [ ] Fleet created with 10 instances across multiple types
- [ ] Mix of Spot and On-Demand instances shown in Instance tab
- [ ] Fleet replaces terminated Spot instances automatically (maintain fleet type)

---

### LAB 30 — AWS Backup for EC2

**Objective:** Create automated, policy-driven backup plans for EC2 and EBS using AWS Backup.

#### Step-by-Step (Console UI)

**Step 1 — Create a Backup Vault**
1. Search → **AWS Backup** → left sidebar → **Backup vaults** → **Create backup vault**
2. Backup vault name: `lab-ec2-vault`
3. Encryption key: **Default (aws/backup)** ← KMS encrypted
4. Click **Create backup vault**

**Step 2 — Create a Backup Plan**
1. Left sidebar → **Backup plans** → **Create backup plan**
2. Select **Build a new plan**
3. Backup plan name: `lab-ec2-backup-plan`

**Configure Backup Rule 1 — Daily:**
1. **Add backup rule**
2. Rule name: `daily-backup`
3. Backup vault: `lab-ec2-vault`
4. Backup frequency: **Daily**
5. Backup window: Start time `02:00 UTC`, Duration `2` hours
6. Retention period: **30 days**
7. **Enable** continuous backup for point-in-time recovery (EC2 AMI-based — note: limited feature)

**Configure Backup Rule 2 — Weekly:**
1. **Add another backup rule**
2. Rule name: `weekly-backup`
3. Frequency: **Weekly** | Day: **Sunday**
4. Backup window: `03:00 UTC`
5. Retention period: **90 days**
6. Copy to vault: Create a vault in `us-west-2` for cross-region DR
7. Click **Create plan**

**Step 3 — Assign Resources**
1. **Backup plans** → select `lab-ec2-backup-plan` → **Assign resources**
2. Resource assignment name: `lab-ec2-instances`
3. IAM role: **Default role** (auto-creates `AWSBackupDefaultServiceRole`)
4. **Assign by tags:**
   - Key: `Backup` | Value: `true`
5. Tag your EC2 instances: EC2 → select instance → **Tags** → **Manage tags** → add `Backup = true`
6. Click **Assign resources**

**Step 4 — Run an On-Demand Backup**
1. AWS Backup → left sidebar → **Protected resources**
2. Select your EC2 instance ARN → **Create on-demand backup**
3. Backup vault: `lab-ec2-vault` | Retention: `7` days
4. Click **Create on-demand backup**
5. Left sidebar → **Jobs** → **Backup jobs** → watch status: `Running` → `Completed` ✅

**Step 5 — Restore from Backup**
1. AWS Backup → **Backup vaults** → `lab-ec2-vault` → click into a recovery point
2. **Actions** → **Restore**
3. Select subnet, security group, instance type
4. IAM role: `AWSBackupDefaultServiceRole`
5. Click **Restore backup** → new instance launched from backup ✅

#### Validation Checklist
- [ ] Backup vault created with encryption
- [ ] Backup plan with daily (30d) and weekly (90d) rules
- [ ] Resources assigned via tag `Backup=true`
- [ ] On-demand backup completes with `Completed` status
- [ ] Restore creates new working EC2 instance

---

### LAB 31 — VPC Endpoints for Private EC2 Access

**Objective:** Allow private EC2 instances to access AWS services (S3, SSM, ECR) without internet/NAT gateway, reducing cost and improving security.

#### Types of VPC Endpoints

| Type | Use Case | Cost |
|---|---|---|
| Gateway Endpoint | S3 and DynamoDB only | Free |
| Interface Endpoint (PrivateLink) | All other AWS services (SSM, ECR, etc.) | ~$0.01/hr + data |

#### Step-by-Step (Console UI)

**Step 1 — Create S3 Gateway Endpoint (Free)**
1. VPC Console → left sidebar → **Endpoints** → **Create endpoint**
2. Service category: **AWS services**
3. Search: `com.amazonaws.us-east-1.s3`
4. Type: **Gateway**
5. VPC: `lab-vpc`
6. Route tables: select **both** private route tables (`lab-pri-rt`)
7. Policy: **Full access** (or custom to restrict to specific buckets)
8. Click **Create endpoint**
9. Verify: VPC → **Route tables** → `lab-pri-rt` → **Routes** — new route to `pl-XXXXX` (S3 prefix list) ✅

**Step 2 — Create SSM Interface Endpoints (Required for Session Manager in Private Subnets Without NAT)**
Create 3 interface endpoints (all required for SSM Session Manager):
1. **Create endpoint** for `com.amazonaws.us-east-1.ssm`
   - Type: **Interface**
   - VPC: `lab-vpc`
   - Subnets: `lab-pri-a`, `lab-pri-b`
   - Security group: create `sg-vpc-endpoints` → allow HTTPS (443) from `10.0.0.0/16`
   - Enable private DNS: ✅
2. Repeat for `com.amazonaws.us-east-1.ssmmessages`
3. Repeat for `com.amazonaws.us-east-1.ec2messages`

**Step 3 — Verify SSM Works Without NAT**
1. Delete the NAT Gateway (or set route to local only)
2. EC2 → **Instances** → select a private instance
3. Systems Manager → **Session Manager** → **Start session** → **Connect** ✅
4. If it connects, SSM traffic flows through the interface endpoint — no internet needed

**Step 4 — Create ECR Endpoint (For Private Container Pulls)**
1. Create interface endpoints for:
   - `com.amazonaws.us-east-1.ecr.api`
   - `com.amazonaws.us-east-1.ecr.dkr`
   - `com.amazonaws.us-east-1.s3` (already created as Gateway)
2. Private EC2 instances can now pull Docker images from ECR without internet ✅

**Step 5 — View Endpoint Metrics**
1. VPC → **Endpoints** → select an interface endpoint
2. **Monitoring** tab → view bytes processed, connection count

#### Validation Checklist
- [ ] S3 Gateway endpoint shows in private route tables as prefix list route
- [ ] 3 SSM interface endpoints created with private DNS enabled
- [ ] Session Manager connects to private instance without NAT gateway
- [ ] S3 access works from private instance without NAT (test: `aws s3 ls` from Session Manager)

---

### LAB 32 — Network Load Balancer (NLB) Setup

**Objective:** Create an NLB for TCP/UDP workloads requiring ultra-low latency, static IPs, and TLS pass-through.

#### ALB vs NLB Comparison

| Feature | ALB | NLB |
|---|---|---|
| Protocol | HTTP/HTTPS/gRPC | TCP/UDP/TLS |
| Latency | ~5–20ms | ~100μs (ultra-low) |
| Static IP | No (dynamic DNS) | Yes (Elastic IP per AZ) |
| TLS termination | Yes | Yes + TLS pass-through |
| WebSockets | Yes | Yes |
| Best for | Web apps, microservices | Gaming, IoT, real-time, gRPC |

#### Step-by-Step (Console UI)

**Step 1 — Create NLB Target Group**
1. EC2 → **Target groups** → **Create target group**
2. Target type: **Instances**
3. Name: `lab-nlb-tg`
4. Protocol: **TCP** | Port: **80**
5. VPC: `lab-vpc`
6. Health check:
   - Protocol: **TCP** (or HTTP if your app supports it)
   - Port: Traffic port
   - Healthy threshold: `3` | Unhealthy threshold: `3`
7. Click **Next** → Register instances → **Create target group**

**Step 2 — Create the NLB**
1. EC2 → **Load Balancers** → **Create load balancer** → **Network Load Balancer** → **Create**
2. Settings:
   - Name: `lab-nlb`
   - Scheme: **Internet-facing**
   - IP address type: **IPv4**
3. **Network mapping:**
   - VPC: `lab-vpc`
   - us-east-1a → `lab-pub-a` | Assign Elastic IP (click **Allocate Elastic IP**)
   - us-east-1b → `lab-pub-b` | Assign Elastic IP
4. **Listeners:**
   - Protocol: **TCP** | Port: **80** | Default action: Forward to `lab-nlb-tg`
   - Add listener: **TLS** | Port: **443** | SSL certificate: (same ACM cert from Lab 15)
5. Click **Create load balancer**
6. Note the two **static Elastic IPs** — these never change, safe to use in DNS/firewall whitelists

**Step 3 — Security Group Note**
NLBs do not have security groups attached. Traffic control is done at:
- The EC2 instance's security group (allow traffic from the NLB's Elastic IPs and VPC CIDR)
- Update `sg-app`: allow TCP 80 from `0.0.0.0/0` (NLB preserves source IP, so you can filter at app layer)

**Step 4 — Enable Cross-Zone Load Balancing**
1. Select `lab-nlb` → **Attributes** tab → **Edit**
2. **Cross-zone load balancing:** Enable
3. **Save changes**

**Step 5 — Test**
1. NLB DNS name or Elastic IPs → test with `curl http://<EIP>` or `nc -zv <EIP> 80`
2. Target group → **Targets** → health should show **healthy**

#### Validation Checklist
- [ ] NLB created with static Elastic IPs per AZ
- [ ] TCP:80 and TLS:443 listeners active
- [ ] Target group health check shows healthy
- [ ] Cross-zone load balancing enabled
- [ ] Existing ALB still serves HTTP/HTTPS workloads; NLB serves TCP workloads

---

### LAB 33 — ALB Advanced Routing: Path, Host & Query String

**Objective:** Configure sophisticated listener rules on ALB to route traffic based on URL path, hostname, headers, and query strings.

#### Step-by-Step (Console UI)

**Step 1 — Create Additional Target Groups**
Create 3 target groups (same steps as Lab 09):
1. `lab-api-tg` — port 80, path `/api/health`
2. `lab-static-tg` — port 80, path `/health`
3. `lab-admin-tg` — port 80, path `/admin/health`
Register appropriate instances to each group.

**Step 2 — Add Path-Based Routing Rules**
1. EC2 → **Load Balancers** → `lab-alb` → **Listeners** tab
2. Select the **HTTPS:443** listener → click the listener → **Manage rules**
3. Click **Add rule** → Rule name: `api-path-rule`
4. **Conditions** → **Add condition** → **Path**
   - Values: `/api/*` → **Confirm**
5. **Actions** → **Add action** → **Forward to target group** → `lab-api-tg`
6. Priority: `10` → **Save**

**Step 3 — Add Host-Based Routing**
1. **Add rule** → Rule name: `admin-host-rule`
2. **Conditions** → **Add condition** → **Host header**
   - Values: `admin.yourdomain.com` → **Confirm**
3. Action: Forward to `lab-admin-tg`
4. Priority: `5` (higher priority than path rules)
5. **Save**

**Step 4 — Add Query String Routing**
1. **Add rule** → Rule name: `query-string-rule`
2. **Conditions** → **Add condition** → **Query string**
   - Key: `version` | Value: `v2`
3. Action: Forward to `lab-api-tg` (route v2 queries to API target group)
4. Priority: `15`
5. **Save**

**Step 5 — Add Fixed Response Rule**
1. **Add rule** → Rule name: `health-endpoint`
2. **Conditions**: Path = `/ping`
3. Action: **Return fixed response**
   - Response code: `200`
   - Content-Type: `text/plain`
   - Response body: `pong`
4. Priority: `1` (highest priority)
5. **Save**

**Step 6 — Add Weighted Routing (Blue/Green Canary)**
1. **Add rule** → Rule name: `canary-release`
2. Condition: Path = `/app/*`
3. Action: **Forward to multiple target groups**:
   - `lab-app-tg` → weight `90` (stable version)
   - `lab-api-tg` → weight `10` (new version canary)
4. Priority: `20`
5. **Save** → now 10% of `/app/*` traffic goes to the new version ✅

**Step 7 — View and Reorder Rules**
1. **Listeners** → **Manage rules** → drag to reorder priorities
2. Lower number = higher priority (rule 1 is checked first)

#### Validation Checklist
- [ ] `/api/*` routes to API target group
- [ ] `admin.yourdomain.com` routes to admin target group
- [ ] `/ping` returns `200 pong` (fixed response, no instances involved)
- [ ] `?version=v2` routes to API target group
- [ ] Canary: exactly ~10% of `/app/*` traffic goes to new target group

---

### LAB 34 — EC2 Auto Scaling with Step Scaling Policies

**Objective:** Create step scaling policies that add/remove instances in steps based on CloudWatch alarm thresholds.

#### Step vs Target Tracking

| Feature | Target Tracking | Step Scaling |
|---|---|---|
| Complexity | Simple — set a target | Complex — define alarm brackets |
| Behavior | Gradually adjusts | Jumps by defined steps |
| Fastest scale-out | Medium | Very fast for extreme spikes |
| Best for | Normal workloads | Variable/unpredictable spikes |

#### Step-by-Step (Console UI)

**Step 1 — Create CloudWatch Alarms for Step Scaling**
1. CloudWatch → **Alarms** → **Create alarm**

**Alarm 1 — Scale-Out (CPU > 60%):**
- Metric: EC2 → ASG → CPUUtilization for `lab-app-asg`
- Threshold: Greater than `60` for 2 consecutive 1-minute periods
- Name: `scale-out-alarm-60`

**Alarm 2 — Scale-Out Aggressive (CPU > 80%):**
- Threshold: Greater than `80`
- Name: `scale-out-alarm-80`

**Alarm 3 — Scale-In (CPU < 30%):**
- Threshold: Less than `30` for 5 consecutive 1-minute periods
- Name: `scale-in-alarm-30`

**Step 2 — Create Step Scaling Policy (Scale-Out)**
1. EC2 → **Auto Scaling Groups** → `lab-app-asg`
2. **Automatic scaling** → **Create dynamic scaling policy**
3. Policy type: **Step scaling**
4. Policy name: `step-scale-out`
5. CloudWatch alarm: `scale-out-alarm-60`
6. **Add step adjustments:**
   - Lower bound: `0` | Upper bound: `20` | Add: `1` instance
     (CPU 60–80% → add 1 instance)
   - Lower bound: `20` | Upper bound: `No upper bound` | Add: `3` instances
     (CPU > 80% → add 3 instances at once)
7. Warmup: `300` seconds
8. Click **Create**

**Step 3 — Create Step Scaling Policy (Scale-In)**
1. **Create dynamic scaling policy**
2. Policy type: **Step scaling**
3. Name: `step-scale-in`
4. Alarm: `scale-in-alarm-30`
5. **Step adjustments:**
   - Lower bound: `No lower bound` | Upper bound: `0` | Remove: `1` instance
6. Click **Create**

**Step 4 — Test the Policies**
1. Connect to 2 ASG instances via Session Manager
2. Run CPU stress on both:
   ```
   sudo yum install -y stress
   stress --cpu 4 --timeout 600 &
   ```
3. CloudWatch → Alarms → watch `scale-out-alarm-60` go to ALARM state
4. ASG → **Activity** tab → see new instances launching ✅
5. Kill stress (`kill %1`) → after 5 min, `scale-in-alarm-30` → scale-in triggers

#### Validation Checklist
- [ ] 3 CloudWatch alarms created (2 scale-out, 1 scale-in)
- [ ] Step policy: CPU 60–80% adds 1 instance; CPU >80% adds 3 instances
- [ ] Stress test causes scale-out event visible in ASG Activity tab
- [ ] Stopping stress causes scale-in after cooldown period

---

### LAB 35 — EC2 Hibernate Feature

**Objective:** Use EC2 hibernate to save instance RAM state and resume instantly without re-bootstrapping.

#### Hibernate vs Stop vs Reboot

| Feature | Stop | Hibernate | Reboot |
|---|---|---|---|
| RAM preserved | ❌ | ✅ | ✅ |
| Root volume | Preserved | Preserved (RAM written to EBS) | Preserved |
| Processes resume | ❌ | ✅ | ✅ |
| Boot time | Full boot (1–5 min) | Fast (~30s) | Fast |
| Instance ID | Same | Same | Same |
| Cost when paused | Volume only | Volume only | N/A |

#### Step-by-Step (Console UI)

**Step 1 — Launch Instance with Hibernate Support**
1. EC2 → **Launch instances**
2. Name: `lab35-hibernate`
3. AMI: **Amazon Linux 2023** (hibernate supported)
4. Instance type: `t3.medium` (hibernate requires instances with instance store OR EBS)
5. **Advanced details** → **Stop - Hibernate behavior:** **Enable**
6. **Storage:**
   - Root volume size: `30` GiB (must be large enough to hold RAM content — add RAM size)
   - Volume type: `gp3` | **Encrypted:** ✅ ← **Required for hibernate**
7. Launch

**Step 2 — Verify Hibernate is Available**
1. Select `lab35-hibernate` instance
2. **Instance state** menu → you should see **Hibernate** option ✅ (only appears when supported)

**Step 3 — Set Up a Long-Running Process**
1. Connect via Session Manager
2. Run a process that would normally take time to restart:
   ```
   # Simulate a long-running computation (watch it survive hibernate)
   python3 -c "
   import time
   count = 0
   while True:
       count += 1
       print(f'Processed {count} items at {time.ctime()}')
       time.sleep(10)
   " &
   echo $! > /tmp/process.pid
   cat /tmp/process.pid
   ```

**Step 4 — Hibernate the Instance**
1. EC2 Console → select the instance
2. **Instance state** → **Hibernate instance** → **Hibernate**
3. Instance state goes: `running` → `stopping` → `stopped` (with hibernate flag)
4. Note: your EIP remains, EBS volumes stay, but no CPU/memory billing ✅

**Step 5 — Resume the Instance**
1. Select the hibernated instance → **Instance state** → **Start instance**
2. Instance starts and loads RAM from EBS — much faster than cold boot
3. Connect via Session Manager → check the process:
   ```
   cat /tmp/process.pid
   ps aux | grep python3   # process still running!
   ```
4. Your long-running process resumed exactly where it left off ✅

#### Validation Checklist
- [ ] Instance launched with hibernate enabled and encrypted root volume
- [ ] **Hibernate instance** option visible in Instance state menu
- [ ] After hibernate + resume, process continues from where it left off
- [ ] Boot time noticeably faster than cold start

---

### LAB 36 — Cross-Account & Cross-Region AMI Sharing

**Objective:** Share custom AMIs between AWS accounts and across regions securely.

#### Step-by-Step (Console UI)

**Part A — Cross-Account AMI Sharing**

**Step 1 — Share AMI with Another Account**
1. EC2 → **AMIs** → select your `lab03-custom-ami-v1.0`
2. **Actions** → **Edit AMI permissions**
3. AMI availability: **Private**
4. **Shared accounts:** click **Add account ID**
5. Enter the **12-digit Account ID** of the target account
6. Click **Save changes**

**Step 2 — Accept in Destination Account**
1. Switch to the destination AWS account
2. EC2 → **AMIs** → left dropdown: **Private images** → filter by: **Images shared with me**
3. The shared AMI appears — it can be used to launch instances now ✅
4. To copy the AMI to own the copy: select it → **Actions** → **Copy AMI** → select region → **Copy**

**Part B — Encrypt a Shared AMI**
If the source AMI is unencrypted, encrypt it during copy:
1. **AMIs** → **Actions** → **Copy AMI**
2. Check **Encrypt target EBS snapshots**
3. KMS key: your account's CMK or `aws/ebs`
4. **Copy AMI** ← now you own an encrypted copy ✅

**Part C — Cross-Region AMI Copy**
1. EC2 (source region) → **AMIs** → select AMI
2. **Actions** → **Copy AMI**
3. Destination region: `ap-south-1` (Mumbai)
4. AMI name: `lab03-custom-ami-v1.0-mumbai`
5. Enable encryption: ✅
6. **Copy AMI** → switch to ap-south-1 to see it appear

**Step 3 — Automate AMI Deregistration (Lifecycle)**
1. EC2 Image Builder → **Distribution settings** → configure deprecation rules
2. Alternatively: EC2 → **AMIs** → old AMIs → **Actions** → **Deprecate AMI** → set date
3. Deprecated AMIs don't appear in launch wizard but still exist for running instances

**Step 4 — Make AMI Public (for distribution)**
1. Select AMI → **Actions** → **Edit AMI permissions**
2. AMI availability: **Public** ← anyone in any account can launch
3. Use carefully — verify no sensitive data in the AMI first

#### Validation Checklist
- [ ] AMI shared with specific account ID
- [ ] Shared AMI visible in destination account under "shared with me"
- [ ] Cross-region copy creates encrypted AMI in ap-south-1
- [ ] AMI deprecation date set on old versions

---

### LAB 37 — CloudWatch Agent: Custom Metrics & Memory Monitoring

**Objective:** Install the CloudWatch Agent to collect memory, disk, and custom application metrics — not natively available in EC2.

#### Why CloudWatch Agent?

Default EC2 metrics (CPU, network, disk I/O) do NOT include:
- RAM/memory utilization
- Disk space usage (%)
- Process-level metrics
- Custom application metrics

#### Step-by-Step (Console UI)

**Step 1 — Update IAM Role**
1. IAM → **Roles** → `EC2SSMRole` → **Add permissions** → **Attach policies**
2. Add: `CloudWatchAgentServerPolicy`
3. Verify the role now has both `AmazonSSMManagedInstanceCore` and `CloudWatchAgentServerPolicy`

**Step 2 — Install CloudWatch Agent via Systems Manager**
1. Search → **Systems Manager** → **Run Command** → **Run command**
2. Command document: search `AWS-ConfigureAWSPackage`
3. Parameters:
   - Action: **Install**
   - Name: `AmazonCloudWatchAgent`
4. Targets: **Specify instance tags** → `Name = asg-app-server`
5. Click **Run** → watch command status: **Success** ✅

**Step 3 — Create Agent Configuration via Parameter Store**
1. Systems Manager → **Parameter Store** → **Create parameter**
2. Name: `/AmazonCloudWatch/config`
3. Type: **String** | Data type: **text**
4. Value (paste this JSON):

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
  },
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/", "/data"],
        "metrics_collection_interval": 60
      },
      "cpu": {
        "totalcpu": true,
        "measurement": ["cpu_usage_active", "cpu_usage_user", "cpu_usage_system"]
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/ec2/system/messages",
            "log_stream_name": "{instance_id}"
          },
          {
            "file_path": "/var/log/user-data.log",
            "log_group_name": "/ec2/user-data",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  }
}
```
5. Click **Create parameter**

**Step 4 — Start the Agent via Run Command**
1. Systems Manager → **Run Command** → **Run command**
2. Document: `AmazonCloudWatch-ManageAgent`
3. Parameters:
   - Action: **configure**
   - Optional Configuration Source: **ssm**
   - Optional Configuration Location: `/AmazonCloudWatch/config`
   - Optional Restart: **yes**
4. Targets: your ASG instances
5. Click **Run** → Success ✅

**Step 5 — Verify Custom Metrics in CloudWatch**
1. CloudWatch → **Metrics** → **All metrics** → **CWAgent** namespace
2. You should see:
   - `mem_used_percent` per instance ✅
   - `disk_used_percent` for `/` and `/data` ✅
3. **Add to dashboard:** EC2-Lab-Dashboard → **Add widget** → Line → CWAgent → mem_used_percent

**Step 6 — Create Memory Alarm**
1. CloudWatch → **Alarms** → **Create alarm**
2. Metric: CWAgent → Per-Instance Metrics → `mem_used_percent`
3. Threshold: Greater than `85`
4. SNS: `lab-alerts`
5. Name: `high-memory-alarm`

#### Validation Checklist
- [ ] CloudWatch Agent installed on all ASG instances via SSM Run Command
- [ ] `mem_used_percent` metric visible in CloudWatch → CWAgent namespace
- [ ] `disk_used_percent` metric visible for `/` and `/data`
- [ ] Log groups created: `/ec2/system/messages`, `/ec2/user-data`
- [ ] Memory alarm created and shows OK state

---

### LAB 38 — ASG Suspension, Standby & Instance Detach

**Objective:** Learn how to safely suspend ASG processes, put instances in standby for maintenance, and detach instances.

#### Step-by-Step (Console UI)

**Part A — Suspend ASG Processes**

**Step 1 — Suspend Scaling Temporarily**
1. EC2 → **Auto Scaling Groups** → `lab-app-asg`
2. **Details** tab → **Advanced configurations** → **Edit**
3. **Suspended processes:** click in the box → add:
   - `Launch` (prevents new instances)
   - `Terminate` (prevents instance removal)
   - `HealthCheck` (stops replacing unhealthy instances during maintenance)
4. **Update**
5. ⚠️ ASG will NOT scale or replace instances while these are suspended

**Step 2 — Resume All Processes**
1. **Edit** → **Suspended processes** → remove all → **Update**
2. ASG resumes normal operation ✅

**Part B — Put Instance in Standby**

**Step 3 — Move Instance to Standby**
1. `lab-app-asg` → **Instance management** tab
2. Select one instance → **Actions** → **Set to Standby**
3. Check **Decrement the desired capacity** ← reduces desired by 1 so ASG doesn't launch a replacement
4. **Set to Standby** → instance state changes to `Standby`
5. ALB removes the instance from rotation — it no longer receives traffic ✅

**Step 4 — Perform Maintenance on Standby Instance**
1. Connect to the standby instance via Session Manager
2. Apply patches, update config, restart services as needed
3. Test the instance manually

**Step 5 — Return to Service**
1. `lab-app-asg` → **Instance management** → select your standby instance
2. **Actions** → **Set to InService**
3. Instance re-joins the ASG and ALB target group → traffic resumes ✅

**Part C — Detach Instance**

**Step 6 — Detach an Instance from ASG**
1. **Instance management** → select an instance
2. **Actions** → **Detach**
3. Options:
   - **Replace with a new instance:** ✅ (ASG launches replacement immediately)
4. **Detach** → instance leaves ASG and becomes a standalone EC2 instance
5. Use case: move instance to a different ASG, or promote it to standalone permanent server

**Step 7 — Re-attach an Instance**
1. From the ASG detail page → **Instance management** → **Attach instances**
2. Select the previously detached instance
3. **Attach instance** → it rejoins the ASG and appears in the ALB target group ✅

#### Validation Checklist
- [ ] Suspending Launch + Terminate prevents scaling during maintenance window
- [ ] Standby instance removed from ALB target group immediately
- [ ] Returning instance from standby restores it to ALB rotation
- [ ] Detached instance becomes standalone EC2 (not in ASG anymore)

---

### LAB 39 — Full DR Drill: Snapshot-Based Recovery

**Objective:** Simulate a complete region failure and recover the full EC2 stack using EBS snapshots and AMI copies.

#### DR Strategy Overview

```
RTO: Recovery Time Objective  = < 30 minutes
RPO: Recovery Point Objective = < 24 hours (daily snapshots)

Strategy: Snapshot & Restore (Pilot Light — minimum cost)

Primary Region (us-east-1): Full ASG + ALB + EC2 stack
DR Region    (us-west-2):   Snapshots + AMI copies ready
                             ASG exists with desired=0 (no running instances)
```

#### Step-by-Step (Console UI)

**Step 1 — Create DR Snapshots**
1. EC2 → **Snapshots** → **Create snapshot**
2. Resource type: **Instance**
3. Instance ID: select your app instance
4. Description: `DR-snapshot-$(date)` (just note the date)
5. **Create snapshot** → wait for **completed** state

**Step 2 — Copy Snapshot to DR Region**
1. Select the snapshot → **Actions** → **Copy snapshot**
2. Destination region: **US West (Oregon) us-west-2**
3. Encryption: ✅
4. Description: `DR copy from us-east-1`
5. **Copy snapshot** → switch to us-west-2 and verify it appears

**Step 3 — Create AMI in DR Region**
1. In us-west-2 → **Snapshots** → select the copied snapshot
2. **Actions** → **Create image from snapshot**
3. Image name: `DR-lab-app-ami`
4. Architecture: `x86_64` | Root device: `/dev/xvda`
5. **Create image** → note the AMI ID in us-west-2

**Step 4 — Pre-Build DR Infrastructure (Pilot Light)**
In us-west-2, create:
1. VPC + Subnets + IGW + NAT (Lab 04 steps) — named with `-dr` suffix
2. Security Groups (Lab 05 steps)
3. ALB + Target Group (Lab 09 steps) — state Active, no targets yet
4. Launch Template pointing to DR AMI (Lab 08 steps)
5. ASG with **min=0, max=10, desired=0** — exists but no instances running

**Step 5 — Simulate Disaster (DR Drill)**
1. Switch to us-east-1 → **Auto Scaling Groups** → `lab-app-asg`
2. **Edit group** → Desired: `0` → **Update** (simulates all instances gone)
3. Route 53: flip health check to use us-west-2 ALB DNS

**Step 6 — Execute Recovery in DR Region**
1. Switch to us-west-2
2. **Auto Scaling Groups** → `lab-app-asg-dr`
3. **Edit group** → Desired: `2` → **Update**
4. Wait 3–5 minutes → instances launch → ALB target group shows **healthy** ✅
5. Route 53 → verify DNS resolves to us-west-2 ALB

**Step 7 — Measure RTO**
1. Record time from: "DR drill started" → "ALB shows healthy targets"
2. Target: < 30 minutes ✅

**Step 8 — Restore Primary (After DR Drill)**
1. us-east-1 → ASG desired back to 2
2. Route 53 → flip DNS back to us-east-1
3. us-west-2 → ASG desired back to 0 (Pilot Light off)

#### DR Runbook (Console Steps Summary)
| Step | Action | Location | Expected Time |
|---|---|---|---|
| 1 | Set us-east-1 ASG desired = 0 | us-east-1 Console | 1 min |
| 2 | Switch Route 53 to us-west-2 ALB | Route 53 Console | 2 min |
| 3 | Set us-west-2 ASG desired = 2 | us-west-2 Console | 1 min |
| 4 | Wait for instances to pass health checks | us-west-2 EC2 | 5–10 min |
| 5 | Verify application accessible | Browser | 1 min |
| **Total** | | | **~10–15 min** |

#### Validation Checklist
- [ ] Snapshot exists and copied to us-west-2
- [ ] DR AMI available in us-west-2
- [ ] Pilot Light ASG pre-created with desired=0
- [ ] DR drill: ASG scaled to desired=2 in us-west-2 within 15 minutes
- [ ] ALB health checks pass in us-west-2
- [ ] Route 53 successfully rerouted to DR region

---

### LAB 40 — Advanced Observability: CloudWatch Logs Insights & Container Insights

**Objective:** Use CloudWatch Logs Insights for intelligent log analysis and set up anomaly detection on EC2 metrics.

#### Step-by-Step (Console UI)

**Part A — CloudWatch Logs Insights**

**Step 1 — Query Application Logs**
1. CloudWatch → left sidebar → **Logs** → **Logs Insights**
2. Select log group: `/ec2/system/messages`
3. Time range: **Last 1 hour**
4. Run these example queries:

**Find all errors:**
```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50
```

**Count errors by hour:**
```
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as error_count by bin(1h)
| sort @timestamp desc
```

**Top 10 slowest requests (if Apache access log):**
```
fields @timestamp, request, response_time
| filter ispresent(response_time)
| sort response_time desc
| limit 10
```

5. Click **Run query** → view results in table or visualization format

**Step 2 — Create Saved Query**
1. After running a useful query → **Save** → name: `error-count-by-hour`
2. Saved queries appear in left sidebar under **Saved queries**

**Step 3 — Create Log Metric Filter**
1. CloudWatch → **Log groups** → `/ec2/system/messages`
2. **Metric filters** tab → **Create metric filter**
3. Filter pattern: `ERROR` (or `[ip, id, user, timestamp, request, status=5*, ...]`)
4. **Test pattern** against sample log data
5. Filter name: `application-errors`
6. Metric name: `AppErrorCount` | Namespace: `CustomApp` | Value: `1`
7. **Create metric filter**
8. Now create an alarm on this custom metric: CloudWatch → Alarms → Create → CustomApp → AppErrorCount → threshold > 5 in 5 min

**Part B — Metric Anomaly Detection**

**Step 4 — Enable Anomaly Detection**
1. CloudWatch → **Alarms** → **Create alarm**
2. **Select metric** → EC2 → ASG → CPUUtilization for `lab-app-asg`
3. Instead of Static threshold, select **Anomaly detection**
4. Model training: Automatic (CloudWatch learns normal patterns from 2 weeks of data)
5. Threshold type: **Greater than the band**
6. Band width: `2` standard deviations
7. This alarm fires when CPU deviates significantly from normal patterns ✅
8. Name: `cpu-anomaly-alarm` → **Create alarm**

**Part C — CloudWatch Contributor Insights**

**Step 5 — Enable Contributor Insights for ALB Logs**
1. CloudWatch → left sidebar → **Contributor Insights** → **Create rule**
2. Log group: select your ALB access logs group (from Lab 15)
3. Contributor Insights rule (JSON):

```json
{
  "Schema": {
    "Name": "CloudWatchLogRule",
    "Version": 1
  },
  "AggregateOn": "Count",
  "Contribution": {
    "Keys": ["$.clientip"],
    "Filters": []
  },
  "LogFormat": "JSON",
  "LogGroupNames": ["/aws/alb/lab-alb"]
}
```
4. **Create rule**
5. View: **Top contributors** → see which client IPs are sending the most requests ← useful for detecting DDoS

**Part D — CloudWatch Dashboard with Insights Widget**

**Step 6 — Add Logs Insights Widget to Dashboard**
1. CloudWatch → **Dashboards** → `EC2-Lab-Dashboard` → **Add widget**
2. Widget type: **Logs table**
3. Log group: `/ec2/system/messages`
4. Query:
```
fields @timestamp, @message
| filter @message like /ERROR|WARN/
| sort @timestamp desc
| limit 20
```
5. **Create widget** → dashboard now shows live error log table ✅

#### Validation Checklist
- [ ] Logs Insights query returns results from EC2 log groups
- [ ] Saved query stored for reuse
- [ ] Log metric filter creates `AppErrorCount` custom metric
- [ ] Anomaly detection alarm created on CPU metric
- [ ] Contributor Insights rule shows top client IPs in ALB logs
- [ ] Dashboard widget shows live error logs inline

---

## 9. Real-World Scenario: Startup to Scale

> **ShopSwift** — A high-growth e-commerce platform. Here is exactly how they would scale from Day 1 to 500K users.

### Stage 1: MVP (0 → 1,000 Users)

| Component | Setup | Cost |
|---|---|---|
| EC2 | 1× t3.medium, single AZ | ~$30/mo |
| EBS | gp3 20GB | ~$2/mo |
| Load Balancer | None (direct EC2) | $0 |
| RDS | t3.micro MySQL | ~$15/mo |
| **Total** | | **~$50/month** |

**Console Setup:** Lab 01 (launch EC2) + Lab 02 (EBS) + Lab 03 (custom AMI)

### Stage 2: Growth (1K → 50K Users)

| Component | Setup | Cost |
|---|---|---|
| EC2 + ASG | 2–8× t3.medium, 2 AZs, Spot mix | ~$100–200/mo |
| ALB | Application Load Balancer + SSL | ~$25/mo |
| RDS | db.m5.large Multi-AZ Aurora MySQL | ~$200/mo |
| CloudWatch | Metrics + Logs + Alarms | ~$15/mo |
| **Total** | | **~$400–600/month** |

**Console Setup:** Labs 04–15 (full VPC + SG + ALB + ASG + monitoring)

### Stage 3: Scale (50K → 500K Users)

| Component | Setup | Cost |
|---|---|---|
| EC2 + ASG | 10–50× c6g.xlarge Graviton, Predictive Scaling | ~$800–1500/mo |
| ALB | 2 ALBs (web + api), WAF, Shield | ~$200/mo |
| CloudFront + S3 | CDN for static assets | ~$100/mo |
| Aurora Global | Primary + 1 global replica | ~$600/mo |
| ElastiCache | Redis 2× r6g.large Multi-AZ | ~$250/mo |
| Compute Savings Plan | 1-year commitment | -$800/mo savings |
| **Total** | | **~$1,500–2,500/month** |

**Console Setup:** Labs 20–25 + Labs 31–40 (advanced patterns)

---

### Production Runbook — Incident Response: Instance Health Degraded

**All steps performed in AWS Console:**

1. **Check ALB Target Health:**
   EC2 → Load Balancers → `lab-alb` → Target groups → `lab-app-tg` → Targets tab
   → Look for instances in `unhealthy` state and their reason codes

2. **Check ASG Activity:**
   EC2 → Auto Scaling Groups → `lab-app-asg` → Activity tab
   → Review recent scaling events and their cause

3. **Check Instance Status Checks:**
   EC2 → Instances → filter by ASG tag → Status check column
   → Look for `1/2` or `0/2` checks — indicates hardware or OS issues

4. **Check Application Logs:**
   CloudWatch → Log groups → `/ec2/system/messages` → Log Insights
   → Run error query for last 30 minutes

5. **Manual Scale if ASG Stuck:**
   EC2 → Auto Scaling Groups → `lab-app-asg` → Edit group → Desired: 4 → Update

6. **Rollback if Bad AMI:**
   EC2 → Launch Templates → `lab-app-lt` → Actions → Set default version → select previous stable version
   EC2 → Auto Scaling Groups → `lab-app-asg` → Instance refresh → Start refresh (90% min healthy)

---

## 10. Troubleshooting Guide

| Symptom | Likely Cause | Console Resolution |
|---|---|---|
| EC2 stuck in `pending` | Wrong subnet/SG, AMI not in region | EC2 → select instance → Actions → Instance settings → check System Log |
| Cannot connect via SSM | SSM agent not running, wrong IAM role | Systems Manager → Fleet Manager → check instance appears there |
| ALB health check failing | `/health` not returning 200, SG blocking | Target Groups → Targets → hover over ❌ to see reason code |
| ASG not scaling out | Cooldown active, alarm not in ALARM state | CloudWatch → Alarms → check if alarm is in OK vs ALARM state |
| User data not running | Script syntax error, wrong shebang | EC2 → Connect → Session Manager → `sudo cat /var/log/cloud-init-output.log` |
| Metadata 401 error | IMDSv2 required, app using IMDSv1 | EC2 → Instance settings → Modify instance metadata options → check settings |
| EBS not mounting after reboot | Missing fstab entry | Connect → `cat /etc/fstab` → verify UUID entry with nofail option |
| ALB 504 Gateway Timeout | Backend slow, grace period too short | EC2 → ASG → Edit → increase health check grace period |
| Spot Instance terminated | EC2 reclaiming capacity | ASG → Instance management → check lifecycle shows Spot type; handle SIGTERM in app |
| Scale-out loop | App bug causing high CPU | Suspend Launch process → investigate app → unsuspend when fixed |
| EICE connection fails | Security group missing outbound to sg-app | VPC → Endpoints → check EICE SG has TCP 22 outbound to sg-app |
| CloudWatch Agent not sending metrics | Missing IAM policy | IAM → EC2SSMRole → verify CloudWatchAgentServerPolicy attached |

---

## 11. Quick Reference Cheatsheet

### Console Navigation: Common Paths

| Task | Console Path |
|---|---|
| Launch EC2 | EC2 → Instances → Launch instances |
| Create AMI | EC2 → Instances → select instance → Actions → Image and templates → Create image |
| Create EBS Snapshot | EC2 → Volumes → select volume → Actions → Create snapshot |
| Create Target Group | EC2 → Target groups → Create target group |
| Create ALB | EC2 → Load Balancers → Create load balancer → Application |
| Create NLB | EC2 → Load Balancers → Create load balancer → Network |
| Create ASG | EC2 → Auto Scaling Groups → Create Auto Scaling group |
| Start SSM Session | Systems Manager → Session Manager → Start session |
| Run Command | Systems Manager → Run Command → Run command |
| Create VPC | VPC → Your VPCs → Create VPC |
| Create VPC Endpoint | VPC → Endpoints → Create endpoint |
| Enable GuardDuty | GuardDuty → Get started → Enable |
| Enable Inspector | Inspector → Get started → Enable |
| View Logs Insights | CloudWatch → Logs → Logs Insights |
| Create Backup Plan | AWS Backup → Backup plans → Create backup plan |
| Create FIS Experiment | FIS → Experiment templates → Create |

### Port Reference

| Port | Protocol | Service | Security Group Rule |
|---|---|---|---|
| 22 | TCP | SSH | Source: EICE security group only (Lab 26) |
| 80 | TCP | HTTP | Source: `0.0.0.0/0` (ALB SG only) |
| 443 | TCP | HTTPS | Source: `0.0.0.0/0` (ALB SG only) |
| 2049 | TCP | NFS (EFS) | Source: `sg-app` only |
| 8080 | TCP | App server | Source: `sg-alb` only |
| 5432 | TCP | PostgreSQL | Source: `sg-app` only |
| 3306 | TCP | MySQL / Aurora | Source: `sg-app` only |
| 6379 | TCP | Redis / ElastiCache | Source: `sg-app` only |

### Free Tier Limits

```
EC2 Free Tier:    750 hrs/month of t2.micro or t3.micro (Linux)
EBS Free Tier:    30 GB of gp2/gp3 storage
S3 Free Tier:     5 GB standard storage
Data Transfer:    1 GB outbound free per month
CloudWatch:       10 custom metrics, 10 alarms free
Systems Manager:  Session Manager always free
```

### Instance Type Quick Reference

| vCPU | RAM | Recommended Type | Use Case |
|---|---|---|---|
| 2 | 1 GB | t3.micro | Labs, dev, very low traffic |
| 2 | 4 GB | t3.medium | Web servers, APIs, small apps |
| 4 | 8 GB | t3.xlarge | Medium apps, small DBs |
| 4 | 8 GB | c6g.xlarge (Graviton) | CPU-intensive, 40% cheaper than x86 |
| 8 | 16 GB | m6i.2xlarge | General production workloads |
| 8 | 64 GB | r6i.2xlarge | Memory-intensive, caching, Elasticsearch |

---

## 📚 Additional Resources

| Resource | URL |
|---|---|
| AWS EC2 Documentation | https://docs.aws.amazon.com/ec2 |
| AWS Well-Architected Framework | https://aws.amazon.com/architecture/well-architected |
| AWS Pricing Calculator | https://calculator.aws |
| EC2 Instance Types Comparison | https://instances.vantage.sh |
| AWS Console Sign In | https://console.aws.amazon.com |
| AWS Free Tier Details | https://aws.amazon.com/free |
| AWS Compute Optimizer | https://aws.amazon.com/compute-optimizer |
| CIS Amazon Linux 2023 Benchmark | https://www.cisecurity.org/benchmark/amazon_linux |
| EC2 Image Builder Docs | https://docs.aws.amazon.com/imagebuilder |
| AWS FIS Docs | https://docs.aws.amazon.com/fis |

---

> **Built for:** Developers, DevOps Engineers, Cloud Architects, and SREs learning AWS Compute at every level.
>
> *Start with Lab 01. Work through sequentially. By Lab 40, you'll have production-grade AWS skills — all via the AWS Management Console.*
>
> **40 Labs | 100% Console UI | No CLI Required | Free Tier Friendly (Labs 1–7)**
