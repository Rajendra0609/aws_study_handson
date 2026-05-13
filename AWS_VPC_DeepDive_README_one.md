# 🌐 AWS VPC Deep Dive — Complete Lab Guide

> **Domain:** Networking | **Level:** Intermediate–Advanced | **Estimated Time:** 4–6 Hours (Lab) | 2–3 Weeks (Real-World Project)

---

## 📋 Table of Contents

1. [What is a VPC?](#1-what-is-a-vpc)
2. [Architecture Overview](#2-architecture-overview)
3. [Services Covered](#3-services-covered)
4. [Prerequisites](#4-prerequisites)
5. [Team Structure (Startup Scenario)](#5-team-structure-startup-scenario)
6. [Resources Required](#6-resources-required)
7. [Lab Setup — Step-by-Step](#7-lab-setup--step-by-step)
   - [Phase 1: Create the VPC](#phase-1-create-the-vpc)
   - [Phase 2: Create Subnets](#phase-2-create-subnets)
   - [Phase 3: Internet Gateway](#phase-3-internet-gateway)
   - [Phase 4: NAT Gateway & Elastic IP](#phase-4-nat-gateway--elastic-ip)
   - [Phase 5: Route Tables](#phase-5-route-tables)
   - [Phase 6: Security Groups](#phase-6-security-groups)
   - [Phase 7: Network ACLs (NACLs)](#phase-7-network-acls-nacls)
   - [Phase 8: VPC Peering](#phase-8-vpc-peering)
   - [Phase 9: Validate & Test Connectivity](#phase-9-validate--test-connectivity)
8. [How to Learn This (Study Path)](#8-how-to-learn-this-study-path)
9. [Real-World Scenario](#9-real-world-scenario)
10. [Common Mistakes to Avoid](#10-common-mistakes-to-avoid)
11. [Cost Estimates](#11-cost-estimates)
12. [Cleanup](#12-cleanup)
13. [Interview Questions](#13-interview-questions)

---

## 1. What is a VPC?

A **Virtual Private Cloud (VPC)** is your own logically isolated section of the AWS cloud. Think of it as your private data center inside AWS — you have full control over IP addressing, subnets, routing, gateways, and security.

### Why VPC Matters

- Every production workload on AWS lives inside a VPC
- Provides network-level isolation, security, and control
- Enables hybrid connectivity (connect your on-premise DC to AWS)
- Mandatory knowledge for any AWS role — DevOps, Cloud, Solutions Architect

### Key Concepts at a Glance

| Concept | What it Does |
|---|---|
| **VPC** | Your isolated private network in AWS |
| **Subnet** | A segment of the VPC's IP space tied to one AZ |
| **Internet Gateway (IGW)** | Allows public subnets to talk to the internet |
| **NAT Gateway** | Lets private subnets reach the internet (outbound only) |
| **Route Table** | Rules that determine where traffic is directed |
| **Security Group** | Instance-level stateful firewall (allow rules only) |
| **NACL** | Subnet-level stateless firewall (allow + deny rules) |
| **Elastic IP** | Static public IPv4 address you own |
| **VPC Peering** | Private routing between two VPCs |

---

## 2. Architecture Overview

```
AWS Region: ap-south-1 (Mumbai)
┌─────────────────────────────────────────────────────────┐
│  VPC: prod-vpc  (10.0.0.0/16)                           │
│                                                         │
│  ┌─── AZ-1a ──────────────┐  ┌─── AZ-1b ────────────┐  │
│  │                        │  │                       │  │
│  │  Public Subnet         │  │  Public Subnet        │  │
│  │  10.0.1.0/24           │  │  10.0.2.0/24          │  │
│  │  [Bastion Host]        │  │  [Load Balancer]      │  │
│  │  [NAT Gateway]         │  │                       │  │
│  │                        │  │                       │  │
│  │  Private Subnet        │  │  Private Subnet       │  │
│  │  10.0.3.0/24           │  │  10.0.4.0/24          │  │
│  │  [App Servers]         │  │  [App Servers]        │  │
│  │                        │  │                       │  │
│  │  DB Private Subnet     │  │  DB Private Subnet    │  │
│  │  10.0.5.0/24           │  │  10.0.6.0/24          │  │
│  │  [RDS / Aurora]        │  │  [RDS Standby]        │  │
│  └────────────────────────┘  └───────────────────────┘  │
│                                                         │
│  Internet Gateway (IGW) ←── Public Internet             │
│  NAT Gateway (in AZ-1a) ←── Private outbound traffic    │
│  VPC Peering ←──────────────── Dev VPC / Mgmt VPC       │
└─────────────────────────────────────────────────────────┘
```

### Traffic Flow Summary

- **Inbound Public Traffic:** Internet → IGW → Public Subnet → Security Group → EC2
- **Private Outbound Traffic:** Private EC2 → NAT Gateway → IGW → Internet
- **DB Access:** App Server (private) → DB Subnet (private, no internet access)
- **Cross-VPC:** VPC Peering → Route Table entry → Target VPC

---

## 3. Services Covered

| AWS Service | Purpose in This Lab |
|---|---|
| **VPC** | Core isolated network |
| **Subnets** | Network segmentation across AZs |
| **Internet Gateway (IGW)** | Internet access for public subnets |
| **NAT Gateway** | Secure outbound access for private subnets |
| **Elastic IP** | Static IP attached to NAT Gateway |
| **Route Tables** | Traffic routing for subnets |
| **Security Groups** | Instance-level stateful firewall |
| **Network ACLs (NACLs)** | Subnet-level stateless firewall |
| **Availability Zones** | High availability across physical locations |
| **VPC Peering** | Private connectivity between two VPCs |

---

## 4. Prerequisites

### Skills Required

- Basic understanding of IP addressing and CIDR notation
- AWS Console access (IAM user or root — not recommended for root in prod)
- Familiarity with EC2 (you'll need it for testing connectivity)
- Basic Linux commands (ping, curl, ssh) for validation

### Tools You Need

- **AWS Account** — Free tier is sufficient for this lab
- **AWS CLI** — For optional CLI-based steps
- **SSH key pair** — To connect to test EC2 instances
- **Terminal / PuTTY** — For SSH access
- **Browser** — AWS Management Console

### AWS CLI Setup (Optional but Recommended)

```bash
# Install AWS CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure
aws configure
# AWS Access Key ID: <your-access-key>
# AWS Secret Access Key: <your-secret-key>
# Default region name: ap-south-1
# Default output format: json
```

---

## 5. Team Structure (Startup Scenario)

> **Scenario:** You've joined a Series-A startup that is migrating from a shared hosting environment to AWS. The team has been asked to set up the foundational AWS network architecture (VPC) in 2 weeks before the app team starts deploying.

### Team Composition

| Role | Count | Responsibility |
|---|---|---|
| **Cloud/DevOps Lead** | 1 | Overall architecture, CIDR planning, design decisions, final review |
| **Network Engineer / Cloud Engineer** | 1–2 | Hands-on VPC, subnets, routing, NACLs, peering setup |
| **Security Engineer** | 1 | Security Groups, NACLs policy, compliance review |
| **QA / Validation Engineer** | 1 | Connectivity testing, audit trails, documentation |
| **Product/Project Manager** | 1 | Timeline, stakeholder updates, sprint planning |

> In a small startup (3–5 person team), the DevOps engineer often doubles as Network + Security. In a scale-up (10+ people), these roles split.

### Sprint Breakdown (2-Week Sprint)

**Week 1**

| Day | Task | Owner |
|---|---|---|
| Day 1 | CIDR planning, AZ selection, architecture diagram | Cloud Lead |
| Day 2 | VPC creation, subnets, IGW | Network Engineer |
| Day 3 | NAT Gateway, Elastic IP, Route Tables | Network Engineer |
| Day 4 | Security Groups, NACLs | Security Engineer |
| Day 5 | VPC Peering (Dev ↔ Prod), review | Cloud Lead + Network Engineer |

**Week 2**

| Day | Task | Owner |
|---|---|---|
| Day 6–7 | Deploy test EC2 instances, validate connectivity | QA Engineer |
| Day 8 | Fix issues, harden NACLs | Security + Network Engineer |
| Day 9 | Documentation, tagging, cost review | All |
| Day 10 | Stakeholder demo, handover to App team | Cloud Lead + PM |

---

## 6. Resources Required

### AWS Resources (Lab — Free Tier)

| Resource | Quantity | Notes |
|---|---|---|
| VPC | 2 | prod-vpc + dev-vpc (for peering) |
| Subnets | 6 (prod) + 2 (dev) | Public, Private App, Private DB per AZ |
| Internet Gateway | 2 | One per VPC |
| NAT Gateway | 1–2 | Chargeable — $0.045/hr per gateway |
| Elastic IP | 1–2 | Free when attached; $0.005/hr when unattached |
| Route Tables | 3–4 | Public RT, Private RT, DB RT |
| Security Groups | 4–5 | Bastion, ALB, App, DB, Internal |
| NACLs | 2 | Public NACL, Private NACL |
| EC2 Instances (test) | 2–3 | t2.micro / t3.micro (free tier) |

### IAM Permissions Required

Your IAM user/role needs at minimum:

```json
{
  "Effect": "Allow",
  "Action": [
    "ec2:CreateVpc",
    "ec2:CreateSubnet",
    "ec2:CreateInternetGateway",
    "ec2:AttachInternetGateway",
    "ec2:CreateNatGateway",
    "ec2:AllocateAddress",
    "ec2:CreateRouteTable",
    "ec2:CreateRoute",
    "ec2:AssociateRouteTable",
    "ec2:CreateSecurityGroup",
    "ec2:AuthorizeSecurityGroupIngress",
    "ec2:AuthorizeSecurityGroupEgress",
    "ec2:CreateNetworkAcl",
    "ec2:CreateNetworkAclEntry",
    "ec2:CreateVpcPeeringConnection",
    "ec2:AcceptVpcPeeringConnection",
    "ec2:DescribeVpcs",
    "ec2:DescribeSubnets",
    "ec2:RunInstances",
    "ec2:TerminateInstances"
  ],
  "Resource": "*"
}
```

### Cost Estimate (Lab)

| Resource | Hourly Cost | 8-Hour Lab Cost |
|---|---|---|
| NAT Gateway | $0.045/hr | ~$0.36 |
| Elastic IP (unattached) | $0.005/hr | ~$0.04 |
| EC2 t2.micro | Free tier | $0 |
| Data Transfer | ~$0.01/GB | Minimal |
| **Total** | | **~$0.50–$1.00** |

> ⚠️ **Always delete NAT Gateway and release Elastic IP when lab is done — these keep charging even when idle.**

---

## 7. Lab Setup — Step-by-Step

### CIDR Plan (Reference This Throughout)

```
VPC CIDR:             10.0.0.0/16  (65,536 IPs)

Public Subnet AZ-1a:  10.0.1.0/24  (254 usable IPs)
Public Subnet AZ-1b:  10.0.2.0/24  (254 usable IPs)

Private App AZ-1a:    10.0.3.0/24
Private App AZ-1b:    10.0.4.0/24

Private DB AZ-1a:     10.0.5.0/24
Private DB AZ-1b:     10.0.6.0/24

Dev VPC CIDR:         10.1.0.0/16  (for VPC peering — must NOT overlap)
```

> AWS reserves 5 IPs per subnet: .0 (network), .1 (router), .2 (DNS), .3 (future), .255 (broadcast)

---

### Phase 1: Create the VPC

**Console Steps:**

1. Go to **AWS Console → VPC → Your VPCs → Create VPC**
2. Fill in:
   - **Name tag:** `prod-vpc`
   - **IPv4 CIDR block:** `10.0.0.0/16`
   - **IPv6 CIDR block:** No IPv6 CIDR block (for now)
   - **Tenancy:** Default
3. Click **Create VPC**

**CLI Alternative:**
```bash
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc},{Key=Environment,Value=Production}]'
```

**Enable DNS Hostnames (Required):**
- Select your VPC → Actions → **Edit VPC settings**
- Check: ✅ Enable DNS hostnames
- Check: ✅ Enable DNS resolution
- Save

```bash
# CLI
aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id <vpc-id> --enable-dns-support
```

**What just happened:** You created an isolated network. Nothing can communicate with the internet yet — no gateway is attached.

---

### Phase 2: Create Subnets

Create all 6 subnets (Public x2, Private App x2, Private DB x2).

**Console Steps (repeat for each subnet):**

1. Go to **VPC → Subnets → Create Subnet**
2. Select your VPC: `prod-vpc`
3. Create each subnet as follows:

| Subnet Name | AZ | CIDR Block | Public? |
|---|---|---|---|
| `pub-subnet-1a` | ap-south-1a | 10.0.1.0/24 | Yes |
| `pub-subnet-1b` | ap-south-1b | 10.0.2.0/24 | Yes |
| `priv-app-subnet-1a` | ap-south-1a | 10.0.3.0/24 | No |
| `priv-app-subnet-1b` | ap-south-1b | 10.0.4.0/24 | No |
| `priv-db-subnet-1a` | ap-south-1a | 10.0.5.0/24 | No |
| `priv-db-subnet-1b` | ap-south-1b | 10.0.6.0/24 | No |

**Enable Auto-Assign Public IP on public subnets:**
- Select `pub-subnet-1a` → Actions → **Edit subnet settings**
- Check: ✅ Enable auto-assign public IPv4 address
- Repeat for `pub-subnet-1b`

**CLI Alternative:**
```bash
# Public Subnet AZ-1a
aws ec2 create-subnet \
  --vpc-id <vpc-id> \
  --cidr-block 10.0.1.0/24 \
  --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=pub-subnet-1a}]'

# Enable Auto-assign Public IP
aws ec2 modify-subnet-attribute \
  --subnet-id <subnet-id> \
  --map-public-ip-on-launch
```

**What just happened:** You divided your VPC into logical segments. Subnets tied to one Availability Zone = High Availability across AZ-1a and AZ-1b.

---

### Phase 3: Internet Gateway

The IGW is the door between your VPC and the public internet.

**Console Steps:**

1. Go to **VPC → Internet Gateways → Create internet gateway**
2. Name: `prod-igw`
3. Create → Then **Actions → Attach to VPC → select prod-vpc**

**CLI Alternative:**
```bash
# Create IGW
aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]'

# Attach to VPC
aws ec2 attach-internet-gateway \
  --internet-gateway-id <igw-id> \
  --vpc-id <vpc-id>
```

**What just happened:** The VPC now has an internet connection point — but nothing is routing traffic to it yet. The route table controls that.

---

### Phase 4: NAT Gateway & Elastic IP

The NAT Gateway lives in the **public subnet** and lets your **private subnet** instances reach the internet for outbound traffic (e.g., downloading packages), without being directly reachable from the internet.

**Step 4a — Allocate an Elastic IP:**

1. Go to **VPC → Elastic IPs → Allocate Elastic IP address**
2. Keep defaults → Click **Allocate**
3. Note the Allocation ID

```bash
aws ec2 allocate-address --domain vpc
```

**Step 4b — Create NAT Gateway:**

1. Go to **VPC → NAT Gateways → Create NAT Gateway**
2. Fill in:
   - **Name:** `prod-nat-gw`
   - **Subnet:** `pub-subnet-1a` ← Must be a PUBLIC subnet
   - **Elastic IP:** Select the one you just allocated
   - **Connectivity type:** Public
3. Click **Create NAT Gateway**

> ⏳ Wait 2–3 minutes for NAT Gateway to become **Available**.

```bash
aws ec2 create-nat-gateway \
  --subnet-id <pub-subnet-1a-id> \
  --allocation-id <eip-allocation-id> \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-nat-gw}]'
```

**What just happened:** Private EC2 instances can now make outbound internet calls. Inbound connections from the internet to private instances are still blocked.

---

### Phase 5: Route Tables

Route tables tell traffic where to go. Each subnet must be associated with exactly one route table.

**Step 5a — Public Route Table:**

1. Go to **VPC → Route Tables → Create route table**
2. Name: `pub-rt`, VPC: `prod-vpc`
3. After creation → **Routes tab → Edit routes → Add route:**
   - Destination: `0.0.0.0/0`
   - Target: Select **Internet Gateway → prod-igw**
4. Save

**Associate Public Subnets:**
- **Subnet associations tab → Edit subnet associations**
- Select: `pub-subnet-1a` and `pub-subnet-1b`
- Save

**Step 5b — Private App Route Table:**

1. Create route table: `priv-app-rt`, VPC: `prod-vpc`
2. Add route:
   - Destination: `0.0.0.0/0`
   - Target: **NAT Gateway → prod-nat-gw**
3. Associate: `priv-app-subnet-1a` and `priv-app-subnet-1b`

**Step 5c — Private DB Route Table:**

1. Create route table: `priv-db-rt`, VPC: `prod-vpc`
2. **No 0.0.0.0/0 route** — DB subnets should have NO internet access
3. Associate: `priv-db-subnet-1a` and `priv-db-subnet-1b`

```bash
# Create public route table
aws ec2 create-route-table --vpc-id <vpc-id> \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=pub-rt}]'

# Add internet route
aws ec2 create-route \
  --route-table-id <pub-rt-id> \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id <igw-id>

# Associate subnet
aws ec2 associate-route-table \
  --route-table-id <pub-rt-id> \
  --subnet-id <pub-subnet-1a-id>
```

**Route Table Summary:**

| Route Table | Destination | Target | Subnets |
|---|---|---|---|
| pub-rt | 10.0.0.0/16 | local | Public 1a, 1b |
| pub-rt | 0.0.0.0/0 | IGW | Public 1a, 1b |
| priv-app-rt | 10.0.0.0/16 | local | App 1a, 1b |
| priv-app-rt | 0.0.0.0/0 | NAT GW | App 1a, 1b |
| priv-db-rt | 10.0.0.0/16 | local | DB 1a, 1b |

**What just happened:** Traffic is now routed correctly. Public subnets → internet via IGW. Private app → internet via NAT. DB subnets → isolated, no internet.

---

### Phase 6: Security Groups

Security Groups are **stateful** instance-level firewalls. If you allow inbound traffic, the response is automatically allowed outbound.

**Create the following Security Groups inside prod-vpc:**

**SG-1: Bastion Host (Jump Server)**

| Direction | Type | Protocol | Port | Source |
|---|---|---|---|---|
| Inbound | SSH | TCP | 22 | Your IP (e.g., 203.x.x.x/32) |
| Outbound | All traffic | All | All | 0.0.0.0/0 |

```bash
aws ec2 create-security-group \
  --group-name bastion-sg \
  --description "Bastion Host SG" \
  --vpc-id <vpc-id>

aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 22 \
  --cidr <your-ip>/32
```

**SG-2: Application Load Balancer (ALB)**

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | 0.0.0.0/0 |
| Inbound | HTTPS | 443 | 0.0.0.0/0 |
| Outbound | All | All | 0.0.0.0/0 |

**SG-3: App Servers (Private)**

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | Custom TCP | 8080 | ALB SG (reference SG-2) |
| Inbound | SSH | 22 | Bastion SG (reference SG-1) |
| Outbound | All | All | 0.0.0.0/0 |

**SG-4: Database (DB Subnet)**

| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | MySQL/Aurora | 3306 | App Server SG (reference SG-3) |
| Outbound | All | All | 0.0.0.0/0 |

> 🔑 **Best Practice:** Reference security groups as sources instead of CIDR blocks wherever possible. This is more secure and dynamically adjusts as instances scale.

---

### Phase 7: Network ACLs (NACLs)

NACLs are **stateless** subnet-level firewalls. You must explicitly allow both inbound AND outbound traffic (including ephemeral/response ports).

> Unlike Security Groups, NACLs support **DENY** rules and process rules in **order by rule number** (lowest number first).

**Ephemeral Port Range:** `1024–65535` — These are the response ports used by clients. NACLs require you to allow these explicitly.

**Public Subnet NACL:**

| Rule # | Type | Protocol | Port Range | Source | Allow/Deny |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | 0.0.0.0/0 | ALLOW |
| 110 | HTTPS | TCP | 443 | 0.0.0.0/0 | ALLOW |
| 120 | SSH | TCP | 22 | Your IP/32 | ALLOW |
| 130 | Custom TCP (ephemeral) | TCP | 1024–65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | All | 0.0.0.0/0 | DENY |

**Outbound — Public NACL:**

| Rule # | Type | Port Range | Destination | Allow/Deny |
|---|---|---|---|---|
| 100 | All TCP | 80, 443 | 0.0.0.0/0 | ALLOW |
| 110 | Custom TCP (ephemeral) | 1024–65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | 0.0.0.0/0 | DENY |

**Private Subnet NACL:**

| Rule # | Type | Protocol | Port | Source | Allow/Deny |
|---|---|---|---|---|---|
| 100 | Custom TCP | TCP | 8080 | 10.0.1.0/24 (Public) | ALLOW |
| 110 | SSH | TCP | 22 | 10.0.1.0/24 (Bastion) | ALLOW |
| 120 | Custom TCP (ephemeral) | TCP | 1024–65535 | 0.0.0.0/0 | ALLOW |
| * | All traffic | All | All | 0.0.0.0/0 | DENY |

**Console Steps:**

1. Go to **VPC → Network ACLs → Create network ACL**
2. Name: `pub-nacl`, VPC: `prod-vpc`
3. Add inbound/outbound rules as above
4. **Subnet associations → Edit → Associate public subnets**
5. Repeat for private NACL

**What just happened:** You now have TWO layers of firewall — NACLs at the subnet level and Security Groups at the instance level.

---

### Phase 8: VPC Peering

VPC Peering creates a private network connection between two VPCs. Traffic stays on AWS backbone — no internet, no VPN needed.

**Use Case:** Connect your `prod-vpc` (10.0.0.0/16) to `dev-vpc` (10.1.0.0/16)

> ⚠️ **CIDR ranges must NOT overlap** between peered VPCs.

**Step 8a — Create Dev VPC (Requester):**

```bash
aws ec2 create-vpc --cidr-block 10.1.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=dev-vpc}]'
```

**Step 8b — Create Peering Connection:**

1. Go to **VPC → Peering Connections → Create Peering Connection**
2. Fill in:
   - **Name:** `prod-dev-peering`
   - **Requester VPC:** `prod-vpc`
   - **Accepter VPC:** `dev-vpc` (same account/region in this lab)
3. Create → Then go to the peering connection → **Actions → Accept Request**

```bash
# Create peering request
aws ec2 create-vpc-peering-connection \
  --vpc-id <prod-vpc-id> \
  --peer-vpc-id <dev-vpc-id> \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=prod-dev-peering}]'

# Accept the peering
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id <pcx-id>
```

**Step 8c — Update Route Tables for Peering:**

In `prod-vpc` → `pub-rt` (and `priv-app-rt`):
- Add route: Destination `10.1.0.0/16` → Target: Peering Connection

In `dev-vpc` route table:
- Add route: Destination `10.0.0.0/16` → Target: Peering Connection

**Step 8d — Update Security Groups:**

Add inbound rule in app server SG:
- Allow traffic from `10.1.0.0/16` (dev-vpc CIDR) on required ports

**What just happened:** Dev and Prod environments can now communicate privately. This is used for shared services like logging, monitoring, or internal APIs.

---

### Phase 9: Validate & Test Connectivity

Deploy test EC2 instances and confirm your routing and security rules work correctly.

**Step 9a — Launch a Bastion Host (Public Subnet)**

1. EC2 → Launch Instance → Amazon Linux 2
2. Instance type: t2.micro
3. Network: `prod-vpc`, Subnet: `pub-subnet-1a`
4. Auto-assign public IP: **Enable**
5. Security Group: `bastion-sg`
6. Launch with your key pair

**Step 9b — Launch a Private App Instance**

1. EC2 → Launch Instance → Amazon Linux 2
2. Instance type: t2.micro
3. Network: `prod-vpc`, Subnet: `priv-app-subnet-1a`
4. Auto-assign public IP: **Disable**
5. Security Group: `app-sg`
6. Same key pair

**Step 9c — Test Public Instance Internet Access**

```bash
# SSH into Bastion
ssh -i your-key.pem ec2-user@<bastion-public-ip>

# Test internet from bastion
curl https://ifconfig.me   # Should return the Elastic IP or public IP
ping 8.8.8.8
```

**Step 9d — SSH into Private Instance via Bastion (Jump Host)**

```bash
# Option 1: SSH ProxyJump (preferred)
ssh -i your-key.pem \
    -o "ProxyJump ec2-user@<bastion-public-ip>" \
    ec2-user@<private-instance-ip>

# Option 2: SSH Agent Forwarding
ssh-add your-key.pem
ssh -A ec2-user@<bastion-public-ip>
# Then from bastion:
ssh ec2-user@<private-instance-ip>
```

**Step 9e — Test Private Instance Internet Access (via NAT)**

```bash
# From the private instance:
curl https://ifconfig.me           # Should show NAT Gateway's Elastic IP
sudo yum update -y                 # Should succeed (outbound internet via NAT)
ping 8.8.8.8                      # Should work
```

**Step 9f — Verify DB Subnet has NO Internet**

```bash
# Launch an instance in DB subnet, SSH to it via Bastion
# From DB instance:
ping 8.8.8.8              # Should FAIL (no route to internet — expected)
curl https://google.com   # Should FAIL (expected)
```

**Step 9g — Test VPC Peering**

```bash
# From prod instance, ping dev instance private IP
ping 10.1.x.x   # Should succeed if routes and SGs are configured correctly
```

### Connectivity Validation Checklist

- [ ] Bastion has internet access (public subnet + IGW)
- [ ] Private app instance can reach internet via NAT (outbound only)
- [ ] Private app instance is NOT reachable directly from internet
- [ ] DB instance has NO internet access
- [ ] VPC peering allows cross-VPC private traffic
- [ ] NACL rules correctly restrict subnet traffic
- [ ] Security Group rules restrict instance-level access

---

## 8. How to Learn This (Study Path)

### Week 1 — Foundation

- [ ] Read AWS VPC documentation (docs.aws.amazon.com/vpc)
- [ ] Understand CIDR notation: practice at [cidr.xyz](https://cidr.xyz)
- [ ] Watch AWS re:Invent VPC networking talks (YouTube)
- [ ] Draw your own VPC architecture before touching console

### Week 2 — Hands-On

- [ ] Complete this lab end-to-end at least **twice** (console first, then CLI)
- [ ] Try destroying one component and observe what breaks
- [ ] Create the same setup using **Terraform** (Infrastructure as Code)
- [ ] Try connecting an EC2 with a load balancer in the public subnet

### Week 3 — Advanced

- [ ] Set up **VPN Gateway** or **Direct Connect** (simulated)
- [ ] Explore **VPC Endpoints** (Gateway for S3/DynamoDB, Interface for others)
- [ ] Implement **VPC Flow Logs** → send to CloudWatch or S3 for analysis
- [ ] Try **Transit Gateway** (hub-spoke VPC peering at scale)
- [ ] Implement **AWS Network Firewall**

### Certifications That Validate This Knowledge

| Cert | Level | Relevance |
|---|---|---|
| AWS Cloud Practitioner | Beginner | Basic VPC concepts |
| AWS Solutions Architect Associate | Intermediate | Full VPC coverage |
| AWS Solutions Architect Professional | Advanced | Complex networking, hybrid |
| AWS Advanced Networking Specialty | Expert | Deep networking focus |

---

## 9. Real-World Scenario

### Scenario: E-Commerce Platform Launch

**Company:** A funded e-commerce startup launching in India.

**Requirements:**
- High availability (minimum 99.9% uptime)
- PCI-DSS compliance (for payment processing)
- Auto-scaling app tier
- Isolated DB tier
- Monitoring + audit logging
- Environment separation (Prod / Staging / Dev)

**How the Team Executes It:**

**Week 1 — Network Design**
- Cloud Architect produces network design document with CIDR plan
- Security team reviews for compliance (PCI-DSS requires network segmentation)
- Dev team reviews for application connectivity requirements

**Week 2 — IaC Development**
- DevOps engineers write Terraform modules for VPC components
- Code reviewed and merged via GitLab CI/CD (with `terraform plan` in pipeline)
- Infra deployed to Staging first for validation

**Week 3 — Hardening & Monitoring**
- VPC Flow Logs enabled → CloudWatch Logs → alerts on unusual traffic
- Network ACLs tightened based on actual traffic patterns
- AWS GuardDuty enabled for threat detection
- Penetration testing on public-facing endpoints

**Week 4 — Production Deployment + Handover**
- Blue/Green deployment of the VPC change
- Application team onboarded
- Runbook created for network changes
- Incident response procedures documented

**Real-World Additions (Not in Basic Lab):**

| Feature | Purpose |
|---|---|
| **VPC Flow Logs** | Capture all IP traffic for security audit |
| **AWS Network Firewall** | Deep packet inspection for production |
| **Transit Gateway** | Connect 10+ VPCs at scale instead of mesh peering |
| **VPC Endpoints** | Private access to S3, DynamoDB without internet |
| **PrivateLink** | Expose internal services privately to other VPCs |
| **AWS WAF on ALB** | Layer 7 protection for web applications |
| **Route 53 Private Hosted Zones** | Internal DNS within the VPC |

**Terraform Example (Real-World IaC):**

```hcl
# vpc.tf
resource "aws_vpc" "prod" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "prod-vpc"
    Environment = "production"
    ManagedBy   = "Terraform"
    Team        = "platform-engineering"
  }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.prod.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "pub-subnet-${count.index + 1}"
    Type = "public"
  }
}

resource "aws_internet_gateway" "prod" {
  vpc_id = aws_vpc.prod.id
  tags = { Name = "prod-igw" }
}

resource "aws_eip" "nat" {
  domain = "vpc"
}

resource "aws_nat_gateway" "prod" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  tags = { Name = "prod-nat-gw" }
  depends_on = [aws_internet_gateway.prod]
}
```

---

## 10. Common Mistakes to Avoid

| Mistake | Impact | Fix |
|---|---|---|
| Overlapping CIDR blocks between VPCs | VPC peering fails | Plan CIDRs upfront; use a CIDR tracker |
| NAT Gateway in private subnet | No internet for private instances | Always place NAT in **public** subnet |
| Forgetting ephemeral ports in NACLs | Connections established but responses dropped | Add rule for ports 1024–65535 |
| Using 0.0.0.0/0 in SG inbound for SSH | Critical security risk | Use specific IP/32 or Bastion SG reference |
| Not enabling DNS hostnames on VPC | Internal DNS doesn't resolve | Enable on VPC creation |
| Same route table for public + private subnets | Private instances get public internet route | Always use separate route tables |
| Leaving NAT Gateway running after lab | Costs ~$32/month | Delete NAT GW and release EIP when done |
| Not tagging resources | Impossible to manage at scale | Tag everything: Name, Environment, Owner |
| Default VPC used in production | Shared with all default resources, hard to audit | Always create a custom VPC for prod |
| Unattached Elastic IP | Costs $0.005/hr for nothing | Release EIPs when not in use |

---

## 11. Cost Estimates

### Lab (8 Hours)

| Item | Cost |
|---|---|
| NAT Gateway | ~$0.36 |
| EC2 t2.micro (free tier) | $0.00 |
| Data processing | ~$0.01 |
| **Total** | **~$0.40** |

### Real-World Production (Monthly)

| Item | Monthly Cost |
|---|---|
| NAT Gateway (1, running 24/7) | ~$32 |
| NAT Gateway data processing ($0.045/GB) | $5–50 depending on traffic |
| Elastic IPs (attached) | $0 |
| VPC Flow Logs → CloudWatch | ~$5–20 |
| EC2 instances (your app servers) | Varies |
| **VPC Infrastructure Base** | **~$40–100/month** |

> For high-traffic production, consider **NAT Instance** (EC2 with NAT AMI) as a cheaper alternative to NAT Gateway — but you lose managed HA.

---

## 12. Cleanup

**Do this in ORDER to avoid dependency errors:**

```bash
# 1. Terminate EC2 instances
aws ec2 terminate-instances --instance-ids <instance-ids>

# 2. Delete NAT Gateway (wait for it to be deleted before releasing EIP)
aws ec2 delete-nat-gateway --nat-gateway-id <nat-gw-id>

# 3. Wait for NAT Gateway status = deleted, then release Elastic IP
aws ec2 release-address --allocation-id <eip-allocation-id>

# 4. Delete VPC Peering Connection
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id <pcx-id>

# 5. Detach and delete Internet Gateway
aws ec2 detach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id>
aws ec2 delete-internet-gateway --internet-gateway-id <igw-id>

# 6. Delete subnets
aws ec2 delete-subnet --subnet-id <subnet-id>   # repeat for all subnets

# 7. Delete route tables (except main)
aws ec2 delete-route-table --route-table-id <rt-id>

# 8. Delete security groups (except default)
aws ec2 delete-security-group --group-id <sg-id>

# 9. Delete NACLs (except default)
aws ec2 delete-network-acl --network-acl-id <nacl-id>

# 10. Delete VPC
aws ec2 delete-vpc --vpc-id <vpc-id>
```

**Console Shortcut:** You can also use **VPC → Actions → Delete VPC** which cascade-deletes most associated resources (except NAT Gateway, EIP, and EC2).

---

## 13. Interview Questions

These are high-probability interview questions for roles involving AWS networking:

**Conceptual**
- What is the difference between a Security Group and a NACL?
- Can a VPC span multiple regions? (No — it is region-scoped)
- What is the difference between an Internet Gateway and a NAT Gateway?
- Why do private instances need a NAT Gateway and not just an IGW?
- What happens if two VPCs have overlapping CIDRs and you try to peer them?
- How does VPC peering differ from Transit Gateway?

**Scenario-Based**
- Your private EC2 instance can't reach the internet. Walk me through the troubleshooting steps.
- A developer can't SSH to an EC2 instance. What do you check?
- How would you design a VPC for a 3-tier web application (Web/App/DB)?
- How do you give an EC2 in a private subnet access to S3 without using the internet?

**Hands-On / Configuration**
- How do you make an S3 bucket accessible from a VPC without going over the internet? (VPC Gateway Endpoint)
- How do you control egress traffic from a VPC at scale? (NAT Gateway / Network Firewall / Egress-only IGW for IPv6)
- What is the purpose of enabling DNS hostnames in a VPC?
- What are ephemeral ports and why do they matter in NACL configuration?

---

## 📚 Additional Resources

| Resource | Link |
|---|---|
| AWS VPC Documentation | https://docs.aws.amazon.com/vpc/latest/userguide/ |
| AWS VPC CIDR Calculator | https://cidr.xyz |
| AWS VPC Limits | https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html |
| Terraform AWS VPC Module | https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws |
| AWS re:Invent VPC Deep Dive | https://www.youtube.com/results?search_query=aws+reinvent+vpc+deep+dive |
| AWS Well-Architected — Networking | https://docs.aws.amazon.com/wellarchitected/latest/framework/ |

---

## ✅ Lab Completion Checklist

- [ ] VPC created with correct CIDR (10.0.0.0/16)
- [ ] 6 subnets created across 2 AZs
- [ ] Public subnets have auto-assign public IP enabled
- [ ] Internet Gateway created and attached
- [ ] Elastic IP allocated
- [ ] NAT Gateway created in public subnet
- [ ] Public route table routes 0.0.0.0/0 to IGW
- [ ] Private app route table routes 0.0.0.0/0 to NAT GW
- [ ] DB route table has NO internet route
- [ ] Security Groups created with least-privilege rules
- [ ] NACLs configured with ephemeral port rules
- [ ] VPC Peering between prod and dev VPCs
- [ ] Route tables updated for peering
- [ ] Connectivity tested (bastion → internet, private → internet via NAT, DB → no internet)
- [ ] Resources tagged properly
- [ ] Cleanup performed after lab

---

*Prepared for: AWS VPC Deep Dive Lab | Environment: Learning / Startup Simulation*  
*Author: Raja (Sri Rajendra Prasad Daggubati) | Stack: AWS | Region: ap-south-1*
