# 🌐 AWS VPC Deep Dive — Lab Setup & Real-World Guide

> **Project Type:** AWS Networking Foundation  
> **Difficulty:** Intermediate  
> **Estimated Lab Time:** 3–4 Hours  
> **AWS Services Used:** VPC, Subnets, Route Tables, Internet Gateway, NAT Gateway, Security Groups, Network ACLs, Elastic IP, Availability Zones

---

## 📋 Table of Contents

1. [What Is a VPC and Why Does It Matter?](#1-what-is-a-vpc-and-why-does-it-matter)
2. [Architecture Overview](#2-architecture-overview)
3. [Prerequisites](#3-prerequisites)
4. [Lab Setup — Step-by-Step (AWS UI)](#4-lab-setup--step-by-step-aws-ui)
   - Step 1: Create the VPC
   - Step 2: Create Subnets
   - Step 3: Create and Attach Internet Gateway
   - Step 4: Create NAT Gateway + Elastic IP
   - Step 5: Configure Route Tables
   - Step 6: Configure Network ACLs
   - Step 7: Configure Security Groups
   - Step 8: Launch EC2 Instances to Validate
5. [Startup Team Structure](#5-startup-team-structure)
6. [Real-World Scenario](#6-real-world-scenario)
7. [Resources Required](#7-resources-required)
8. [Common Pitfalls & How to Avoid Them](#8-common-pitfalls--how-to-avoid-them)
9. [How to Learn This — Study Roadmap](#9-how-to-learn-this--study-roadmap)
10. [Cleanup (Avoid Billing)](#10-cleanup-avoid-billing)
11. [Glossary of Key Terms](#11-glossary-of-key-terms)

---

## 1. What Is a VPC and Why Does It Matter?

A **Virtual Private Cloud (VPC)** is your own logically isolated, private network inside AWS. Think of it as building your own data center inside the AWS cloud — you control:

- **IP address ranges** (CIDR blocks)
- **Subnets** (public-facing vs. private)
- **Routing** (how traffic flows in and out)
- **Firewall rules** (Security Groups and NACLs)
- **Internet access controls** (Internet Gateway, NAT Gateway)

Without a VPC, your servers would be either fully exposed to the internet or completely isolated with no connectivity. VPC gives you the right balance — **secure, controlled, scalable networking.**

### Key Components at a Glance

| Component | Purpose | Scope |
|---|---|---|
| VPC | Isolated virtual network | Regional |
| Subnet | Segment of the VPC's IP range | Per AZ |
| Internet Gateway (IGW) | Public internet access for public subnets | Per VPC |
| NAT Gateway | Outbound-only internet for private subnets | Per AZ |
| Elastic IP | Static public IP for NAT Gateway | Per AZ |
| Route Table | Controls where traffic is directed | Per subnet |
| Security Group | Instance-level stateful firewall | Per resource |
| Network ACL (NACL) | Subnet-level stateless firewall | Per subnet |
| Availability Zone | Physical data center for HA design | Per Region |

---

## 2. Architecture Overview

```
AWS Region: ap-south-1 (Mumbai)
┌──────────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                            │
│                                                              │
│  ┌─────────────────────┐  ┌─────────────────────┐           │
│  │   AZ-1a             │  │   AZ-1b             │           │
│  │                     │  │                     │           │
│  │ ┌─────────────────┐ │  │ ┌─────────────────┐ │           │
│  │ │ Public Subnet   │ │  │ │ Public Subnet   │ │           │
│  │ │ 10.0.1.0/24     │ │  │ │ 10.0.2.0/24     │ │           │
│  │ │  [Web/Bastion]  │ │  │ │  [Web/LB]       │ │           │
│  │ └────────┬────────┘ │  │ └────────┬────────┘ │           │
│  │          │          │  │          │           │           │
│  │ ┌────────▼────────┐ │  │ ┌────────▼────────┐ │           │
│  │ │ Private Subnet  │ │  │ │ Private Subnet  │ │           │
│  │ │ 10.0.3.0/24     │ │  │ │ 10.0.4.0/24     │ │           │
│  │ │  [App/DB/EKS]   │ │  │ │  [App/DB/EKS]   │ │           │
│  │ └─────────────────┘ │  │ └─────────────────┘ │           │
│  └─────────────────────┘  └─────────────────────┘           │
│                                                              │
│  Internet Gateway (IGW) ←──── Internet                      │
│  NAT Gateway (AZ-1a) ←── Elastic IP                         │
│  NAT Gateway (AZ-1b) ←── Elastic IP  [For HA]               │
└──────────────────────────────────────────────────────────────┘

Traffic Rules:
  Public Subnet → Route Table → IGW → Internet (both directions)
  Private Subnet → Route Table → NAT GW → IGW → Internet (outbound only)
  NACL → Subnet-level filters (stateless)
  Security Group → Instance-level filters (stateful)
```

---

## 3. Prerequisites

### AWS Account
- An active AWS account (Free Tier works for this lab)
- IAM user or role with the following permissions:
  - `AmazonVPCFullAccess`
  - `AmazonEC2FullAccess`
- **Recommended:** Enable MFA on your AWS account before starting

### Cost Estimate
| Resource | Cost |
|---|---|
| VPC, Subnets, IGW, Route Tables, NACLs, Security Groups | **Free** |
| NAT Gateway | ~$0.045/hour + $0.045/GB data transfer |
| Elastic IP (unattached) | $0.005/hour |
| EC2 (t2.micro) for testing | Free Tier eligible (750 hrs/month) |
| **Total for a 4-hour lab** | **~$0.20–$0.50 USD** |

> ⚠️ **Always delete NAT Gateway and release Elastic IPs after the lab to avoid ongoing charges.**

### Local Machine
- Web browser (Chrome/Firefox) to access AWS Console
- AWS CLI installed (optional, for validation)
- SSH client (Terminal on Mac/Linux, PuTTY or Windows Terminal on Windows)

---

## 4. Lab Setup — Step-by-Step (AWS UI)

> **Region Used in This Lab:** `ap-south-1` (Mumbai). All steps work identically in any region.

---

### Step 1: Create the VPC

1. Log into **AWS Management Console** → Search for **VPC** in the top search bar → Click **VPC**
2. In the left sidebar, click **Your VPCs** → Click **Create VPC**
3. Fill in the following:

| Field | Value |
|---|---|
| Resources to create | **VPC only** (not VPC and more) |
| Name tag | `lab-vpc` |
| IPv4 CIDR block | `10.0.0.0/16` |
| IPv6 CIDR block | No IPv6 CIDR block |
| Tenancy | Default |

4. Click **Create VPC**

**What just happened?** You created an isolated network with 65,536 possible IP addresses (10.0.0.0 → 10.0.255.255). Nothing is connected to the internet yet.

> 💡 **Tip:** AWS reserves 5 IP addresses in every subnet (first 4 and last 1). Plan your CIDR blocks accordingly.

---

### Step 2: Create Subnets

You'll create **4 subnets** — 2 public and 2 private, spread across 2 Availability Zones.

#### Enable DNS Settings First
1. Select `lab-vpc` from **Your VPCs** list
2. Click **Actions** → **Edit VPC settings**
3. Enable **DNS hostnames** ✅ and **DNS resolution** ✅
4. Click **Save**

#### Create Public Subnet — AZ-1a
1. Left sidebar → **Subnets** → **Create subnet**
2. VPC ID: Select `lab-vpc`
3. Click **Add new subnet** and fill in:

| Field | Value |
|---|---|
| Subnet name | `lab-public-subnet-1a` |
| Availability Zone | `ap-south-1a` |
| IPv4 CIDR block | `10.0.1.0/24` |

#### Create Public Subnet — AZ-1b
Click **Add new subnet** again:

| Field | Value |
|---|---|
| Subnet name | `lab-public-subnet-1b` |
| Availability Zone | `ap-south-1b` |
| IPv4 CIDR block | `10.0.2.0/24` |

#### Create Private Subnet — AZ-1a
Click **Add new subnet** again:

| Field | Value |
|---|---|
| Subnet name | `lab-private-subnet-1a` |
| Availability Zone | `ap-south-1a` |
| IPv4 CIDR block | `10.0.3.0/24` |

#### Create Private Subnet — AZ-1b
Click **Add new subnet** again:

| Field | Value |
|---|---|
| Subnet name | `lab-private-subnet-1b` |
| Availability Zone | `ap-south-1b` |
| IPv4 CIDR block | `10.0.4.0/24` |

5. Click **Create subnet** (all 4 are created together)

#### Enable Auto-Assign Public IP for Public Subnets
Repeat for both public subnets:
1. Select `lab-public-subnet-1a` → **Actions** → **Edit subnet settings**
2. Check ✅ **Enable auto-assign public IPv4 address**
3. Click **Save** → Repeat for `lab-public-subnet-1b`

> 💡 **Why?** Without this, EC2 instances in public subnets won't get a public IP by default, and you won't be able to reach them.

---

### Step 3: Create and Attach Internet Gateway (IGW)

1. Left sidebar → **Internet Gateways** → **Create internet gateway**
2. Name tag: `lab-igw`
3. Click **Create internet gateway**
4. After creation, click **Actions** → **Attach to VPC**
5. Select `lab-vpc` → Click **Attach internet gateway**

**What just happened?** The IGW is like plugging your VPC into the internet. However, just creating it doesn't route traffic — you still need to update route tables in Step 5.

> ⚠️ **Only 1 IGW can be attached to a VPC at a time.**

---

### Step 4: Create NAT Gateway + Elastic IP

NAT Gateway allows your private subnet instances to reach the internet (for updates, API calls, etc.) without being directly reachable from the internet.

#### Create Elastic IP — AZ-1a
1. Left sidebar → **Elastic IPs** → **Allocate Elastic IP address**
2. Leave defaults → Click **Allocate**
3. Note the IP address — you'll use this for NAT Gateway

#### Create NAT Gateway — AZ-1a
1. Left sidebar → **NAT Gateways** → **Create NAT gateway**
2. Fill in:

| Field | Value |
|---|---|
| Name | `lab-nat-gateway-1a` |
| Subnet | `lab-public-subnet-1a` ← **Must be a PUBLIC subnet** |
| Connectivity type | Public |
| Elastic IP allocation ID | Select the IP you just created |

3. Click **Create NAT gateway**

> ⏳ **NAT Gateway takes 2–3 minutes to become available.** Wait until the Status shows **Available** before proceeding.

> 💡 **For High Availability (Production):** Create a second NAT Gateway in `lab-public-subnet-1b` with its own Elastic IP. This ensures private subnets in AZ-1b don't lose internet access if AZ-1a fails.

---

### Step 5: Configure Route Tables

Route tables tell traffic where to go. You'll create **3 route tables:**
- 1 Public Route Table (for both public subnets)
- 2 Private Route Tables (one per AZ, pointing to the NAT Gateway)

#### Create Public Route Table
1. Left sidebar → **Route Tables** → **Create route table**

| Field | Value |
|---|---|
| Name | `lab-public-rt` |
| VPC | `lab-vpc` |

2. Click **Create route table**
3. Select `lab-public-rt` → **Routes tab** → **Edit routes** → **Add route**

| Destination | Target |
|---|---|
| `0.0.0.0/0` | Select **Internet Gateway** → `lab-igw` |

4. Click **Save changes**
5. Go to **Subnet associations tab** → **Edit subnet associations**
6. Check ✅ `lab-public-subnet-1a` and `lab-public-subnet-1b`
7. Click **Save associations**

#### Create Private Route Table — AZ-1a
1. **Create route table** → Name: `lab-private-rt-1a`, VPC: `lab-vpc`
2. **Edit routes** → Add route:

| Destination | Target |
|---|---|
| `0.0.0.0/0` | Select **NAT Gateway** → `lab-nat-gateway-1a` |

3. **Subnet associations** → Associate `lab-private-subnet-1a`

#### Create Private Route Table — AZ-1b
1. **Create route table** → Name: `lab-private-rt-1b`, VPC: `lab-vpc`
2. **Edit routes** → Add route:

| Destination | Target |
|---|---|
| `0.0.0.0/0` | Select **NAT Gateway** → `lab-nat-gateway-1a` *(or 1b if you created one)* |

3. **Subnet associations** → Associate `lab-private-subnet-1b`

> 💡 **Verify:** Select each route table and confirm the subnet associations are correct. A subnet can only be associated with ONE route table at a time.

---

### Step 6: Configure Network ACLs (NACLs)

NACLs are **stateless** — you must explicitly allow both inbound and outbound rules. They act as an extra layer of security at the subnet level.

> AWS creates a **default NACL** that allows all traffic. For this lab, you'll create custom NACLs for better control.

#### Create Custom NACL for Public Subnets
1. Left sidebar → **Network ACLs** → **Create network ACL**

| Field | Value |
|---|---|
| Name | `lab-public-nacl` |
| VPC | `lab-vpc` |

2. Select `lab-public-nacl` → **Inbound rules** → **Edit inbound rules** → **Add new rule**

| Rule # | Type | Protocol | Port | Source | Allow/Deny |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | Allow |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | Allow |
| 120 | SSH | TCP | 22 | *Your IP*/32 | Allow |
| 130 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

3. **Outbound rules** → **Edit outbound rules** → **Add new rule**

| Rule # | Type | Protocol | Port | Destination | Allow/Deny |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | Allow |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | Allow |
| 120 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

4. **Subnet associations** → Associate `lab-public-subnet-1a` and `lab-public-subnet-1b`

#### Create Custom NACL for Private Subnets
1. Create another NACL: `lab-private-nacl`, VPC: `lab-vpc`
2. **Inbound rules:**

| Rule # | Type | Protocol | Port | Source | Allow/Deny |
|---|---|---|---|---|---|
| 100 | All traffic | All | All | 10.0.0.0/16 | Allow |
| 110 | Custom TCP | TCP | 1024-65535 | 0.0.0.0/0 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

3. **Outbound rules:**

| Rule # | Type | Protocol | Port | Destination | Allow/Deny |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | Allow |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | Allow |
| 120 | All traffic | All | All | 10.0.0.0/16 | Allow |
| * | All traffic | All | All | 0.0.0.0/0 | Deny |

4. Associate `lab-private-subnet-1a` and `lab-private-subnet-1b`

> ⚠️ **NACL vs. Security Group Differences:**
> - NACLs are **stateless** — return traffic must be explicitly allowed (hence the ephemeral port rules 1024–65535)
> - Security Groups are **stateful** — allowed inbound traffic automatically allows the return
> - NACLs process rules in **order** (lowest rule number wins first match)

---

### Step 7: Configure Security Groups

Security Groups act as virtual firewalls at the **instance level**. Unlike NACLs, they are stateful.

#### Create Web Server Security Group (Public)
1. Left sidebar → **Security Groups** → **Create security group**

| Field | Value |
|---|---|
| Name | `lab-sg-web` |
| Description | Allow HTTP, HTTPS, SSH from internet |
| VPC | `lab-vpc` |

2. **Inbound rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | My IP *(your current IP auto-fills)* |
| HTTP | TCP | 80 | 0.0.0.0/0 |
| HTTPS | TCP | 443 | 0.0.0.0/0 |

3. **Outbound rules:** Leave default (Allow all outbound)
4. Click **Create security group**

#### Create Application Server Security Group (Private)
1. **Create security group**

| Field | Value |
|---|---|
| Name | `lab-sg-app` |
| Description | Allow traffic only from web layer |
| VPC | `lab-vpc` |

2. **Inbound rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| Custom TCP | TCP | 8080 | `lab-sg-web` *(select the SG, not an IP)* |
| SSH | TCP | 22 | `lab-sg-web` *(SSH only from bastion/web)* |

3. Click **Create security group**

#### Create Database Security Group (Private)
1. **Create security group:** `lab-sg-db`
2. **Inbound rules:**

| Type | Protocol | Port | Source |
|---|---|---|---|
| MySQL/Aurora | TCP | 3306 | `lab-sg-app` |

3. Click **Create security group**

> 💡 **Best Practice:** Reference Security Group IDs as sources instead of IP ranges wherever possible. This creates a dynamic rule — if you add more app servers, they automatically inherit database access.

---

### Step 8: Launch EC2 Instances to Validate

#### Launch Bastion Host (Public Subnet)
1. Go to **EC2** → **Launch Instances**
2. Name: `lab-bastion-host`
3. AMI: Amazon Linux 2023 (Free Tier eligible)
4. Instance type: `t2.micro`
5. Create a new key pair: `lab-key-pair` → Download the `.pem` file
6. **Network settings:**
   - VPC: `lab-vpc`
   - Subnet: `lab-public-subnet-1a`
   - Auto-assign public IP: Enable
   - Security group: Select `lab-sg-web`
7. Click **Launch instance**

#### Launch Private Instance (Private Subnet)
1. **Launch Instance** → Name: `lab-private-app`
2. AMI: Amazon Linux 2023, Instance type: `t2.micro`
3. **Network settings:**
   - VPC: `lab-vpc`
   - Subnet: `lab-private-subnet-1a`
   - Auto-assign public IP: Disable
   - Security group: Select `lab-sg-app`
4. Key pair: `lab-key-pair` (same key)
5. Click **Launch instance**

#### Validate Connectivity
```bash
# 1. SSH into Bastion Host from your machine
ssh -i lab-key-pair.pem ec2-user@<BASTION-PUBLIC-IP>

# 2. From Bastion, SSH into the Private Instance
ssh -i lab-key-pair.pem ec2-user@<PRIVATE-INSTANCE-PRIVATE-IP>
# (e.g., 10.0.3.xxx)

# 3. From Private Instance, test internet access via NAT Gateway
curl https://checkip.amazonaws.com
# You should see the NAT Gateway's Elastic IP — NOT the private instance IP
# This confirms: Private Subnet → NAT GW → Internet is working ✅

# 4. Confirm private instance has NO direct public IP
curl https://checkip.amazonaws.com  # Should show NAT GW Elastic IP, not instance IP
ping google.com  # Should work (via NAT)
```

**Expected Results:**
- ✅ Bastion has a public IP and you can SSH into it from your machine
- ✅ Private instance has only a private IP (10.0.x.x)
- ✅ Private instance can reach the internet through NAT Gateway
- ✅ No one can directly SSH into the private instance from the internet

---

## 5. Startup Team Structure

> **Scenario:** A seed-stage startup building a SaaS product is setting up its cloud infrastructure on AWS. The CTO has assigned a small team to design and implement the foundational VPC architecture.

### Team Composition (5 People)

---

#### 👤 Role 1: Cloud/Infrastructure Lead (1 Person)
**Seniority:** Senior (5+ years)  
**Responsibilities:**
- Owns the overall VPC architecture design
- Defines CIDR block strategy (accounting for future growth and VPC peering)
- Sets organizational standards: tagging policies, naming conventions, subnet strategy
- Reviews and approves all Terraform/CDK code before it's applied
- Makes final decisions on HA vs. cost tradeoffs (e.g., 1 NAT GW vs. 2)
- Writes the Architecture Decision Records (ADRs)

**In this project:**
- Designs the `10.0.0.0/16` VPC with future-proof subnetting
- Decides on 2 AZs with room for a 3rd later
- Approves the NACL and Security Group rulesets

---

#### 👤 Role 2: DevOps/Platform Engineer (1 Person — Raja's Role)
**Seniority:** Mid-level (2–4 years)  
**Responsibilities:**
- Implements the VPC infrastructure using **Terraform** (Infrastructure as Code)
- Manages the CI/CD pipeline that applies infra changes (Jenkins or GitHub Actions)
- Handles the day-2 operations: monitoring, cost optimization, patching NAT GW configs
- Sets up VPC Flow Logs → CloudWatch/S3 for audit and debugging
- Writes runbooks and lab documentation
- Troubleshoots connectivity issues (routing, NACL conflicts, SG rules)

**In this project:**
- Writes all Terraform modules: `vpc`, `subnets`, `igw`, `nat`, `routes`, `sg`, `nacl`
- Runs `terraform plan` and `terraform apply` through the CI/CD pipeline
- Sets up VPC Flow Logs to S3 for the security team

---

#### 👤 Role 3: Security Engineer (1 Person)
**Seniority:** Mid-level  
**Responsibilities:**
- Reviews and signs off on Security Group rules and NACL rulesets
- Enforces least privilege — no `0.0.0.0/0` on SSH except via bastion
- Ensures private subnets have no IGW routes (pure outbound via NAT only)
- Configures AWS Security Hub, GuardDuty, and VPC Flow Log alerts
- Scans IaC code with **Checkov** or **tfsec** for misconfigurations

**In this project:**
- Reviews all SG and NACL rules against the CIS AWS Benchmark
- Flags and blocks any "allow all inbound" rules
- Sets up GuardDuty to detect anomalous VPC traffic

---

#### 👤 Role 4: Backend Developer (1 Person)
**Seniority:** Mid-level  
**Responsibilities:**
- Specifies which ports the application needs open (e.g., 8080 for API, 5432 for PostgreSQL)
- Tests connectivity between the application tier and database tier after subnets are provisioned
- Works with the DevOps engineer to configure the correct Security Group ingress rules
- Validates that the app can reach external APIs through the NAT Gateway

**In this project:**
- Tests that the Node.js app in the private subnet can reach external services
- Confirms database connections are only accessible from the app SG

---

#### 👤 Role 5: Project Manager / Technical Program Manager (1 Person)
**Seniority:** Mid-level  
**Responsibilities:**
- Creates and tracks tasks in Jira/Linear
- Facilitates daily standups, sprint planning, and retrospectives
- Manages dependencies (e.g., security sign-off must happen before production deploy)
- Communicates progress and blockers to the CTO
- Maintains the project timeline and budget (monitors AWS costs)

**In this project:**
- Tracks a 2-week sprint: Day 1–3 design, Day 4–7 implementation, Day 8–10 testing/review

### Execution Timeline

```
Week 1:
  Day 1-2  │ Architecture Review & CIDR Planning (Lead + Security)
  Day 3    │ Terraform module skeleton written (DevOps Engineer)
  Day 4-5  │ VPC, Subnets, IGW, NAT GW provisioned via Terraform (DevOps)
  Day 5    │ Security review of NACL & SG rules (Security Engineer)

Week 2:
  Day 6-7  │ Route Tables, NACLs, Security Groups finalized (DevOps + Security)
  Day 8    │ EC2 test instances launched, connectivity validated (DevOps + Backend)
  Day 9    │ VPC Flow Logs, CloudWatch alarms configured (DevOps + Security)
  Day 10   │ Documentation, retrospective, handoff (PM + All)
```

---

## 6. Real-World Scenario

### The Problem (Without VPC Design)
A real startup without proper VPC design might:
- Deploy all EC2 instances with public IPs for "convenience"
- Have databases directly reachable from the internet
- Use a single AZ (losing everything in an AZ outage)
- Forget outbound internet access for private instances (software updates fail)

### The Solution (What This Lab Builds)
This architecture is used by thousands of companies. Here's how it maps to real business needs:

#### Tier Isolation
```
Internet → Load Balancer (Public Subnet)
         → Application Servers (Private Subnet, AZ-1a and AZ-1b)
         → Database (Private Subnet, AZ-1a — RDS Multi-AZ handles its own replication)
```
- **Public Subnet:** Only your load balancers and bastion hosts live here
- **Private Subnet:** Application servers, EKS worker nodes, Lambda, RDS — anything that shouldn't be directly internet-facing
- **Result:** Even if someone compromises your web tier, they cannot directly reach your database

#### High Availability
- Spreading subnets across 2 AZs means if `ap-south-1a` experiences an outage, your `ap-south-1b` resources continue serving traffic
- Auto Scaling Groups and EKS automatically distribute workloads across AZs

#### Secure Outbound Updates
- Private EC2 instances need to `yum update` or `pip install` packages from the internet
- NAT Gateway provides this outbound access without exposing the instances to inbound connections

#### Compliance & Audit
- VPC Flow Logs capture all network traffic metadata → S3 or CloudWatch
- This satisfies auditors for standards like **SOC 2**, **ISO 27001**, and **PCI-DSS**
- NACLs provide an additional documented control layer that auditors love

#### Cost Optimization (Real World)
- **Dev environment:** 1 NAT Gateway (cross-AZ traffic is cheap and AZ failure acceptable)
- **Production:** 2 NAT Gateways (one per AZ) to eliminate cross-AZ charges and ensure HA
- Use **VPC Endpoints** for S3 and DynamoDB to avoid NAT Gateway data processing costs

---

## 7. Resources Required

### AWS Resources (Lab)

| Resource | Count | Notes |
|---|---|---|
| VPC | 1 | |
| Subnets | 4 | 2 public, 2 private |
| Internet Gateway | 1 | |
| NAT Gateway | 1–2 | 1 for lab, 2 for HA prod |
| Elastic IP | 1–2 | One per NAT Gateway |
| Route Tables | 3 | 1 public, 2 private |
| Network ACLs | 2 | 1 public, 1 private |
| Security Groups | 3 | web, app, db |
| EC2 Instances | 2 | For validation only |
| Key Pair | 1 | For SSH |

### Learning Resources

| Resource | Type | Link |
|---|---|---|
| AWS VPC Official Docs | Documentation | docs.aws.amazon.com/vpc |
| AWS Well-Architected Framework | Best Practices | aws.amazon.com/architecture/well-architected |
| Stephane Maarek — AWS SAA Course | Video Course | Udemy |
| Adrian Cantrill — AWS SAA Course | Video Course | learn.cantrill.io |
| TechWorld with Nana — Networking Basics | YouTube | Free |
| AWS Skill Builder | Labs & Quizzes | skillbuilder.aws (Free + Paid) |
| Terraform AWS VPC Module | IaC Reference | registry.terraform.io |

### Tools for Real-World Implementation

| Tool | Purpose |
|---|---|
| **Terraform** | Infrastructure as Code for VPC provisioning |
| **AWS CloudFormation** | AWS-native IaC alternative |
| **Checkov / tfsec** | Scan IaC for VPC misconfigurations |
| **AWS VPC Reachability Analyzer** | Visually test connectivity between resources |
| **VPC Flow Logs** | Network traffic auditing and anomaly detection |
| **AWS Network Manager** | Visualize multi-VPC, multi-region topologies |
| **draw.io / Lucidchart** | Document your VPC architecture diagrams |

---

## 8. Common Pitfalls & How to Avoid Them

| Pitfall | Symptom | Fix |
|---|---|---|
| Private subnet instance can't reach internet | `curl` times out from private EC2 | Check private route table points to NAT GW, not IGW |
| NAT Gateway in wrong subnet | No outbound from private instances | NAT GW **must** be in a PUBLIC subnet |
| Forgot to attach IGW to VPC | Public instances unreachable | Verify IGW is attached to VPC (only 1 allowed) |
| Public subnet not auto-assigning IPs | EC2 has no public IP | Enable auto-assign public IP on public subnet |
| NACL blocking return traffic | Connections hang/timeout | Add ephemeral port range (1024-65535) to NACL outbound rules |
| Security Group references are missing | App can't reach DB | Use SG references, not hardcoded IPs |
| Wrong route table association | Traffic going nowhere | A subnet can only have 1 route table — verify associations |
| Elastic IP left unattached after NAT deletion | Ongoing billing charges | Release EIPs when not in use |
| No VPC Flow Logs | Can't debug network issues | Enable Flow Logs to S3/CloudWatch from day 1 |
| Single AZ design | Regional outage takes down app | Always use at least 2 AZs in production |

---

## 9. How to Learn This — Study Roadmap

### Phase 1 — Foundations (Week 1–2)
- [ ] Understand IPv4 CIDR notation: what does `/16`, `/24`, `/32` mean?
- [ ] Learn subnetting basics: how many hosts per subnet?
- [ ] Understand OSI Model layers 3 (Network) and 4 (Transport)
- [ ] Read AWS VPC FAQ and the VPC User Guide Introduction

### Phase 2 — Core VPC Concepts (Week 3–4)
- [ ] Complete this lab manually through the AWS UI
- [ ] Draw your architecture on paper before you build it
- [ ] Understand the difference between Security Groups (stateful) and NACLs (stateless)
- [ ] Practice: delete and rebuild the entire VPC from scratch 3 times

### Phase 3 — Infrastructure as Code (Week 5–6)
- [ ] Redo this entire lab using **Terraform**
- [ ] Use the `terraform-aws-modules/vpc/aws` module
- [ ] Add VPC Flow Logs via Terraform
- [ ] Push your Terraform code to GitHub, use a CI/CD pipeline to apply it

### Phase 4 — Advanced Networking (Week 7–8)
- [ ] VPC Peering — connect two VPCs
- [ ] AWS Transit Gateway — hub-and-spoke for many VPCs
- [ ] VPC Endpoints — private connectivity to S3, DynamoDB without NAT
- [ ] AWS PrivateLink — expose services privately to other VPCs
- [ ] Site-to-Site VPN and Direct Connect concepts

### Phase 5 — Certification Alignment
- **AWS Solutions Architect Associate (SAA-C03):** VPC is ~20% of the exam
- **AWS Advanced Networking Specialty:** Deep dive into all networking topics
- Study with practice exams on **Tutorials Dojo** (Jon Bonso)

---

## 10. Cleanup (Avoid Billing)

**Delete resources in this exact order to avoid dependency errors:**

```
1. Terminate EC2 Instances
   EC2 → Instances → Select both → Instance State → Terminate

2. Delete NAT Gateway
   VPC → NAT Gateways → Select lab-nat-gateway-1a → Actions → Delete
   ⏳ Wait for Status: Deleted (takes 1–2 minutes)

3. Release Elastic IP
   VPC → Elastic IPs → Select the IP → Actions → Release Elastic IP address

4. Detach and Delete Internet Gateway
   VPC → Internet Gateways → Select lab-igw → Actions → Detach from VPC
   Then → Actions → Delete Internet Gateway

5. Delete Custom Route Tables
   VPC → Route Tables → Delete lab-public-rt, lab-private-rt-1a, lab-private-rt-1b
   (Cannot delete the main route table — that's deleted with the VPC)

6. Delete Custom NACLs
   VPC → Network ACLs → Delete lab-public-nacl, lab-private-nacl

7. Delete Security Groups
   VPC → Security Groups → Delete lab-sg-db first, then lab-sg-app, then lab-sg-web
   (Delete in dependency order — DB references App, App references Web)

8. Delete Subnets
   VPC → Subnets → Select all 4 → Actions → Delete subnet

9. Delete the VPC
   VPC → Your VPCs → Select lab-vpc → Actions → Delete VPC
```

> ✅ After cleanup, verify in **AWS Cost Explorer** or the **Billing Dashboard** that no VPC-related charges are accruing.

---

## 11. Glossary of Key Terms

| Term | Definition |
|---|---|
| **VPC** | Virtual Private Cloud — isolated network in AWS |
| **CIDR** | Classless Inter-Domain Routing — notation like `10.0.0.0/16` defining IP ranges |
| **Subnet** | Subdivision of a VPC's IP range, confined to one AZ |
| **Public Subnet** | Subnet with a route to the Internet Gateway |
| **Private Subnet** | Subnet with no direct route to the internet |
| **IGW** | Internet Gateway — allows two-way internet access for public subnets |
| **NAT Gateway** | Network Address Translation — allows outbound-only internet for private subnets |
| **Elastic IP** | Static public IPv4 address, allocated to your account |
| **Route Table** | Rules that determine where network traffic is directed |
| **Security Group** | Stateful virtual firewall at the instance/resource level |
| **NACL** | Network Access Control List — stateless firewall at the subnet level |
| **AZ** | Availability Zone — physically separate data center within a region |
| **Bastion Host** | Jump server in a public subnet used to SSH into private instances |
| **VPC Flow Logs** | Capture IP traffic metadata going in/out of your VPC |
| **Stateful** | Return traffic is automatically allowed (Security Groups) |
| **Stateless** | Return traffic must be explicitly allowed (NACLs) |
| **Ephemeral Ports** | Short-lived ports (1024–65535) used for return traffic in TCP |

---

## ✅ Lab Completion Checklist

- [ ] VPC created with `10.0.0.0/16` CIDR
- [ ] 4 subnets created (2 public, 2 private across 2 AZs)
- [ ] DNS Hostnames and DNS Resolution enabled on VPC
- [ ] Auto-assign public IP enabled on public subnets
- [ ] Internet Gateway created and attached to VPC
- [ ] NAT Gateway created in public subnet with Elastic IP
- [ ] Public Route Table: `0.0.0.0/0 → IGW`, associated with both public subnets
- [ ] Private Route Tables: `0.0.0.0/0 → NAT GW`, each associated with private subnet
- [ ] Custom NACLs created for public and private subnets
- [ ] Security Groups created for web, app, and database tiers
- [ ] EC2 bastion host launched in public subnet — SSH works from internet
- [ ] EC2 private instance launched in private subnet — only reachable via bastion
- [ ] From private instance, `curl checkip.amazonaws.com` returns NAT Gateway EIP
- [ ] VPC Flow Logs enabled (optional but recommended)
- [ ] All resources deleted after lab to avoid billing

---

*Prepared for: Raja — DevOps Engineer Lab Reference*  
*Version: 1.0 | Services: AWS VPC, EC2, Networking*  
*Region: ap-south-1 (Mumbai) | Estimated Cost: < $1 USD for full lab*
