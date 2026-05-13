# 🌐 AWS VPC — Master Guide: Complete Deep Dive + Advanced Labs
> **Domain:** Networking | **Level:** Beginner → Expert  
> **Estimated Time:** Foundation Labs: 4–6 hrs | Advanced Labs: 8–12 hrs | Expert Labs: 4–6 hrs  
> **AWS Services:** VPC, Subnets, IGW, NAT Gateway, Security Groups, NACLs, VPC Peering, VPC Endpoints, Transit Gateway, ALB, Route 53, Network Firewall, GuardDuty, Flow Logs, Client VPN, PrivateLink, Traffic Mirroring, VPC Lattice, IPAM, Network Monitor, Terraform  
> **Region:** ap-south-1 (Mumbai) | **Estimated Cost:** < $5 USD for all labs

> *This is a consolidated guide merging all VPC lab materials — Foundation through Expert (30 Labs Total). Duplications have been removed and unique content from all source documents has been preserved.*

---

## 📋 Table of Contents

**Foundation**
1. [What is a VPC?](#1-what-is-a-vpc)
2. [Architecture Overview](#2-architecture-overview)
3. [Key Concepts & Glossary](#3-key-concepts--glossary)
4. [Prerequisites & Cost Estimate](#4-prerequisites--cost-estimate)
5. [CIDR Plan (Reference Throughout)](#5-cidr-plan)

**Foundation Labs (Do These First)**
- [LAB 1 — Create the VPC](#lab-1--create-the-vpc)
- [LAB 2 — Create Subnets (6 Subnets Across 2 AZs)](#lab-2--create-subnets)
- [LAB 3 — Internet Gateway](#lab-3--internet-gateway)
- [LAB 4 — NAT Gateway + Elastic IP](#lab-4--nat-gateway--elastic-ip)
- [LAB 5 — Route Tables](#lab-5--route-tables)
- [LAB 6 — Security Groups](#lab-6--security-groups)
- [LAB 7 — Network ACLs (NACLs)](#lab-7--network-acls)
- [LAB 8 — VPC Peering (Prod ↔ Dev)](#lab-8--vpc-peering)
- [LAB 9 — Launch EC2 & Validate Connectivity](#lab-9--launch-ec2--validate-connectivity)

**Advanced Labs (Build Real-World Skills)**
- [LAB 10 — VPC Flow Logs (Network Audit)](#lab-10--vpc-flow-logs)
- [LAB 11 — VPC Gateway Endpoint for S3](#lab-11--vpc-gateway-endpoint-for-s3)
- [LAB 12 — VPC Interface Endpoints (PrivateLink)](#lab-12--vpc-interface-endpoints-privatelink)
- [LAB 13 — Application Load Balancer (ALB) + Target Groups](#lab-13--application-load-balancer-alb)
- [LAB 14 — Route 53 Private Hosted Zone (Internal DNS)](#lab-14--route-53-private-hosted-zone)
- [LAB 15 — AWS Transit Gateway (Multi-VPC Hub-and-Spoke)](#lab-15--aws-transit-gateway)
- [LAB 16 — VPC Reachability Analyzer (Debug Connectivity)](#lab-16--vpc-reachability-analyzer)
- [LAB 17 — AWS Network Firewall (Deep Packet Inspection)](#lab-17--aws-network-firewall)
- [LAB 18 — GuardDuty + Flow Log Threat Detection](#lab-18--guardduty--flow-log-threat-detection)
- [LAB 19 — Site-to-Site VPN Gateway (Hybrid Cloud)](#lab-19--site-to-site-vpn-gateway)
- [LAB 20 — Terraform IaC (Rebuild Everything as Code)](#lab-20--terraform-iac)

**Expert Labs (Production-Grade Scenarios)**
- [LAB 21 — RDS Multi-AZ with DB Subnet Groups](#lab-21--rds-multi-az-with-db-subnet-groups)
- [LAB 22 — VPC IPAM (IP Address Manager)](#lab-22--vpc-ipam-ip-address-manager)
- [LAB 23 — AWS Client VPN (Remote Employee Access)](#lab-23--aws-client-vpn)
- [LAB 24 — PrivateLink — Expose Your Own Service](#lab-24--privatelink--expose-your-own-service)
- [LAB 25 — Traffic Mirroring (Network Forensics)](#lab-25--traffic-mirroring)
- [LAB 26 — Egress-Only Internet Gateway (IPv6)](#lab-26--egress-only-internet-gateway-ipv6)
- [LAB 27 — VPC Lattice (Service-to-Service Networking)](#lab-27--vpc-lattice)
- [LAB 28 — CloudWatch Network Monitor](#lab-28--cloudwatch-network-monitor)
- [LAB 29 — AWS Network Access Analyzer (Blast Radius Assessment)](#lab-29--aws-network-access-analyzer)
- [LAB 30 — Prefix Lists + Managed Security Group Rules at Scale](#lab-30--prefix-lists--managed-security-group-rules)

**Reference**
- [Common Pitfalls & Fixes](#common-pitfalls--fixes)
- [Interview Questions](#interview-questions)
- [Cleanup (Avoid Billing)](#cleanup)
- [Lab Completion Checklist](#lab-completion-checklist)

---

## 1. What is a VPC?

A **Virtual Private Cloud (VPC)** is your own logically isolated, private network inside AWS. Think of it as building your own data center inside the AWS cloud — you control:

- **IP address ranges** (CIDR blocks)
- **Subnets** (public-facing vs. private)
- **Routing** (how traffic flows in and out)
- **Firewall rules** (Security Groups and NACLs)
- **Internet access controls** (Internet Gateway, NAT Gateway)

Without a VPC, your servers would be either fully exposed to the internet or completely isolated. VPC gives you the right balance — **secure, controlled, scalable networking.**

### Why VPC Matters

- Every production workload on AWS lives inside a VPC
- Provides network-level isolation, security, and compliance controls
- Enables hybrid connectivity (connect your on-premises DC to AWS via VPN or Direct Connect)
- Mandatory knowledge for any AWS role — DevOps, Cloud Engineer, Solutions Architect

---

## 2. Architecture Overview

```
AWS Region: ap-south-1 (Mumbai)
┌──────────────────────────────────────────────────────────────────┐
│  VPC: prod-vpc  (10.0.0.0/16)                                    │
│                                                                  │
│  ┌──────── AZ-1a ──────────────────┐  ┌───── AZ-1b ───────────┐ │
│  │                                 │  │                        │ │
│  │  Public Subnet 10.0.1.0/24      │  │  Public Subnet         │ │
│  │  [Bastion Host] [NAT Gateway]   │  │  10.0.2.0/24 [ALB]    │ │
│  │  [ALB Target]                   │  │                        │ │
│  │                                 │  │                        │ │
│  │  Private App 10.0.3.0/24        │  │  Private App           │ │
│  │  [App Servers / EKS]            │  │  10.0.4.0/24           │ │
│  │                                 │  │                        │ │
│  │  Private DB 10.0.5.0/24         │  │  Private DB            │ │
│  │  [RDS / Aurora Primary]         │  │  10.0.6.0/24 [Standby] │ │
│  └─────────────────────────────────┘  └────────────────────────┘ │
│                                                                  │
│  Internet Gateway (IGW) ←────────────── Public Internet         │
│  NAT Gateway (AZ-1a) ←── Elastic IP                             │
│  VPC Peering ←──────────────────────── Dev VPC (10.1.0.0/16)    │
│  Transit Gateway ←──────────────────── Mgmt / Shared VPC        │
│  VPC Endpoints ←────────────────────── S3, SSM, Secrets Manager │
│  Network Firewall ←─────────────────── Egress inspection        │
└──────────────────────────────────────────────────────────────────┘

Traffic Flows:
  Internet → IGW → Public Subnet → Security Group → EC2
  Private EC2 → NAT GW → IGW → Internet (outbound only)
  App → DB Subnet (no internet route — fully isolated)
  App → S3 via VPC Gateway Endpoint (never leaves AWS network)
  Cross-VPC → VPC Peering or Transit Gateway
```

---

## 3. Key Concepts & Glossary

| Term | Definition |
|---|---|
| **VPC** | Virtual Private Cloud — isolated network in AWS |
| **CIDR** | `10.0.0.0/16` — defines your IP address range |
| **Subnet** | Subdivision of VPC IP range, confined to one AZ |
| **Public Subnet** | Has a route to the Internet Gateway — resources can get public IPs |
| **Private Subnet** | No direct route to internet — resources only reachable internally |
| **IGW** | Internet Gateway — allows two-way internet access for public subnets |
| **NAT Gateway** | Allows outbound-only internet for private subnets |
| **Elastic IP** | Static public IPv4 address attached to your account |
| **Route Table** | Rules that determine where network traffic is directed |
| **Security Group** | **Stateful** firewall at the instance/resource level |
| **NACL** | **Stateless** firewall at the subnet level — must allow both directions |
| **Ephemeral Ports** | 1024–65535 — response ports used by TCP; must be allowed in NACLs |
| **AZ** | Availability Zone — physically separate data center within a region |
| **Bastion Host** | Jump server in a public subnet to SSH into private instances |
| **VPC Flow Logs** | Capture IP traffic metadata going in/out of your VPC |
| **VPC Peering** | Private routing between two VPCs |
| **VPC Endpoint** | Private connection to AWS services without using the internet |
| **Transit Gateway** | Hub-and-spoke routing between many VPCs and on-premises |
| **PrivateLink** | Expose services privately to other VPCs without peering |

---

## 4. Prerequisites & Cost Estimate

### Requirements
- AWS account (Free Tier works for most of this guide)
- IAM user with `AmazonVPCFullAccess` + `AmazonEC2FullAccess`
- Basic understanding of IP addresses and CIDR notation
- Terminal/SSH client for connectivity testing

### IAM Permissions (Minimum Required)

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

### Cost Estimate

| Resource | Lab (8 hours) | Production (monthly) |
|---|---|---|
| VPC, Subnets, IGW, Route Tables, NACLs, Security Groups | **Free** | **Free** |
| NAT Gateway | ~$0.36 | ~$32 + $0.045/GB data |
| Elastic IP (unattached) | $0.005/hr | $0 when attached |
| EC2 t2.micro (Free Tier) | $0 | Varies |
| Transit Gateway | $0.05/hr + $0.02/GB | ~$36+ |
| Network Firewall | $0.395/hr per AZ | ~$285/AZ |
| VPC Flow Logs → CloudWatch | Minimal | ~$5–20 |
| **Foundation Labs (4 hours)** | **~$0.25** | — |
| **All 30 Labs (12 hours)** | **~$2.50–$5.00** | — |

> ⚠️ **Always delete NAT Gateway, Transit Gateway, and Network Firewall after labs — these generate significant ongoing charges.**

---

## 4a. Team Structure (Startup Scenario)

> **Scenario:** A Series-A startup migrating from shared hosting to AWS. The team has been tasked with setting up the foundational VPC architecture in 2 weeks before the app team starts deploying.

### Team Composition

| Role | Count | Responsibility |
|---|---|---|
| **Cloud/Infrastructure Lead** | 1 | Overall architecture, CIDR planning, design decisions, ADRs, final review |
| **DevOps/Platform Engineer** | 1–2 | Hands-on VPC implementation, Terraform IaC, CI/CD, Flow Logs, runbooks |
| **Security Engineer** | 1 | Security Groups, NACLs policy, compliance review, GuardDuty, tfsec/Checkov |
| **Backend Developer** | 1 | Specifies port requirements, validates app-to-DB connectivity |
| **Project Manager / TPM** | 1 | Sprint planning, timeline, stakeholder updates, cost monitoring |

> In a small startup (3–5 person team), the DevOps engineer often doubles as Network + Security. In a scale-up (10+ people), these roles split.

### Sprint Breakdown (2-Week Sprint)

**Week 1**

| Day | Task | Owner |
|---|---|---|
| Day 1–2 | CIDR planning, AZ selection, architecture diagram, ADRs | Cloud Lead + Security |
| Day 3 | Terraform module skeleton; VPC, Subnets, IGW | DevOps Engineer |
| Day 4–5 | NAT GW, Elastic IP, Route Tables, VPC Peering | DevOps Engineer |
| Day 5 | Security Group and NACL rule review against CIS Benchmark | Security Engineer |

**Week 2**

| Day | Task | Owner |
|---|---|---|
| Day 6–7 | Deploy test EC2 instances, validate all connectivity | DevOps + QA |
| Day 8 | Fix issues, harden NACLs, scan IaC with Checkov | Security + DevOps |
| Day 9 | VPC Flow Logs, CloudWatch alarms, tagging, cost review | All |
| Day 10 | Documentation, retrospective, handoff to App team | PM + All |

---

## 4b. Real-World Scenario: E-Commerce Platform Launch

**Company:** A funded e-commerce startup launching in India.

**Requirements:** High availability (≥99.9% uptime), PCI-DSS compliance (payment processing), auto-scaling app tier, isolated DB tier, monitoring + audit logging, environment separation (Prod / Staging / Dev).

**Execution:**

- **Week 1 — Network Design:** Cloud Architect produces network design document. Security team reviews for PCI-DSS compliance (requires network segmentation). Dev team reviews application connectivity requirements.
- **Week 2 — IaC Development:** DevOps writes Terraform modules. Code reviewed via GitLab CI/CD with `terraform plan` in pipeline. Deployed to Staging first.
- **Week 3 — Hardening & Monitoring:** Flow Logs → CloudWatch with traffic alerts. NACLs tightened from real traffic patterns. GuardDuty enabled. Pen testing on public endpoints.
- **Week 4 — Production Deployment:** Blue/Green deployment. App team onboarded. Runbooks and incident response procedures documented.

**Production Additions Beyond Basic Lab:**

| Feature | Purpose |
|---|---|
| **VPC Flow Logs** | Capture all IP traffic for security audit (SOC 2, PCI-DSS, ISO 27001) |
| **AWS Network Firewall** | Deep packet inspection for egress control |
| **Transit Gateway** | Connect 10+ VPCs at scale instead of mesh peering |
| **VPC Endpoints** | Private access to S3, DynamoDB without internet or NAT costs |
| **PrivateLink** | Expose internal services privately to other VPCs |
| **AWS WAF on ALB** | Layer 7 protection for web applications |
| **Route 53 Private Hosted Zones** | Internal DNS within the VPC |

---

## 5. CIDR Plan

> Reference this throughout all labs.

```
prod-vpc CIDR:          10.0.0.0/16   (65,536 IPs)

Public Subnet AZ-1a:    10.0.1.0/24   (254 usable IPs)  → Bastion, NAT GW, ALB
Public Subnet AZ-1b:    10.0.2.0/24   (254 usable IPs)  → ALB
Private App AZ-1a:      10.0.3.0/24                      → App Servers
Private App AZ-1b:      10.0.4.0/24                      → App Servers
Private DB AZ-1a:       10.0.5.0/24                      → RDS Primary
Private DB AZ-1b:       10.0.6.0/24                      → RDS Standby

dev-vpc CIDR:           10.1.0.0/16   (for VPC peering — must NOT overlap)
mgmt-vpc CIDR:          10.2.0.0/16   (for Transit Gateway lab)

Note: AWS reserves 5 IPs per subnet:
  .0 = network address
  .1 = VPC router
  .2 = AWS DNS
  .3 = reserved for future use
  .255 = broadcast
```

---

## LAB 1 — Create the VPC

### 🖥️ Console Steps

1. **AWS Console** → search for **VPC** → click **VPC**
2. Left sidebar → **Your VPCs** → **Create VPC**
3. Fill in:

| Field | Value |
|---|---|
| Resources to create | **VPC only** |
| Name tag | `prod-vpc` |
| IPv4 CIDR block | `10.0.0.0/16` |
| IPv6 CIDR block | No IPv6 |
| Tenancy | Default |

4. Click **Create VPC**
5. Select `prod-vpc` → **Actions → Edit VPC settings**
6. Enable ✅ **DNS hostnames** and ✅ **DNS resolution** → **Save**

### 💻 CLI Commands

```bash
# Create the VPC
VPC_ID=$(aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc},{Key=Environment,Value=Production}]' \
  --query 'Vpc.VpcId' --output text)

echo "VPC created: $VPC_ID"

# Enable DNS hostnames and resolution
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support

# Verify
aws ec2 describe-vpcs --vpc-ids $VPC_ID \
  --query 'Vpcs[0].{CIDR:CidrBlock,DNS:EnableDnsHostnames,State:State}' \
  --output table
```

> ✅ **What happened:** You created an isolated network with 65,536 possible IPs. Nothing is connected to the internet yet.

---

## LAB 2 — Create Subnets

Create **6 subnets** across 2 Availability Zones — 2 public, 2 private app, 2 private DB.

### 🖥️ Console Steps

1. Left sidebar → **Subnets** → **Create subnet**
2. VPC ID: Select `prod-vpc`
3. Click **Add new subnet** and create all 6:

| Subnet Name | AZ | CIDR Block | Purpose |
|---|---|---|---|
| `pub-subnet-1a` | ap-south-1a | `10.0.1.0/24` | Bastion, NAT GW, ALB |
| `pub-subnet-1b` | ap-south-1b | `10.0.2.0/24` | ALB |
| `priv-app-subnet-1a` | ap-south-1a | `10.0.3.0/24` | App Servers |
| `priv-app-subnet-1b` | ap-south-1b | `10.0.4.0/24` | App Servers |
| `priv-db-subnet-1a` | ap-south-1a | `10.0.5.0/24` | RDS Primary |
| `priv-db-subnet-1b` | ap-south-1b | `10.0.6.0/24` | RDS Standby |

4. Click **Create subnet**

**Enable Auto-Assign Public IP on public subnets:**
- Select `pub-subnet-1a` → **Actions → Edit subnet settings**
- Check ✅ **Enable auto-assign public IPv4 address** → **Save**
- Repeat for `pub-subnet-1b`

### 💻 CLI Commands

```bash
# Public Subnets
PUB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=pub-subnet-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PUB_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.2.0/24 --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=pub-subnet-1b}]' \
  --query 'Subnet.SubnetId' --output text)

# Enable auto-assign public IP
aws ec2 modify-subnet-attribute --subnet-id $PUB_1A --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $PUB_1B --map-public-ip-on-launch

# Private App Subnets
PRIV_APP_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.3.0/24 --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=priv-app-subnet-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PRIV_APP_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.4.0/24 --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=priv-app-subnet-1b}]' \
  --query 'Subnet.SubnetId' --output text)

# Private DB Subnets
PRIV_DB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.5.0/24 --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=priv-db-subnet-1a}]' \
  --query 'Subnet.SubnetId' --output text)

PRIV_DB_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.6.0/24 --availability-zone ap-south-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=priv-db-subnet-1b}]' \
  --query 'Subnet.SubnetId' --output text)

echo "Subnets created:"
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[].{Name:Tags[0].Value,CIDR:CidrBlock,AZ:AvailabilityZone}' \
  --output table
```

---

## LAB 3 — Internet Gateway

### 🖥️ Console Steps

1. Left sidebar → **Internet Gateways** → **Create internet gateway**
2. Name tag: `prod-igw` → **Create internet gateway**
3. After creation → **Actions → Attach to VPC** → select `prod-vpc` → **Attach**

### 💻 CLI Commands

```bash
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway \
  --internet-gateway-id $IGW_ID \
  --vpc-id $VPC_ID

echo "✅ IGW created and attached: $IGW_ID"
```

> ✅ **What happened:** VPC now has an internet connection point. But nothing routes to it yet — that's the route table's job.

---

## LAB 4 — NAT Gateway + Elastic IP

NAT Gateway lets private subnet instances reach the internet for outbound traffic (package updates, API calls) without being directly reachable from the internet.

> ⚠️ NAT Gateway **must** be placed in a **public subnet**.

### 🖥️ Console Steps

1. Left sidebar → **Elastic IPs** → **Allocate Elastic IP address** → **Allocate**
2. Note the Allocation ID
3. Left sidebar → **NAT Gateways** → **Create NAT gateway**

| Field | Value |
|---|---|
| Name | `prod-nat-gw` |
| Subnet | `pub-subnet-1a` ← Must be PUBLIC |
| Connectivity type | Public |
| Elastic IP | Select the one you just allocated |

4. Click **Create NAT gateway**
5. ⏳ Wait 2–3 minutes until Status = **Available**

### 💻 CLI Commands

```bash
# Allocate Elastic IP
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc \
  --query 'AllocationId' --output text)

echo "Elastic IP allocated: $EIP_ALLOC"

# Create NAT Gateway in the public subnet
NAT_GW_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_1A \
  --allocation-id $EIP_ALLOC \
  --tag-specifications 'ResourceType=natgateway,Tags=[{Key=Name,Value=prod-nat-gw}]' \
  --query 'NatGateway.NatGatewayId' --output text)

echo "NAT Gateway created: $NAT_GW_ID"
echo "Waiting for NAT Gateway to become available..."

aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_ID
echo "✅ NAT Gateway is now available"
```

---

## LAB 5 — Route Tables

Route tables tell traffic where to go. Each subnet is associated with exactly one route table.

### 🖥️ Console Steps

**Public Route Table:**
1. Left sidebar → **Route Tables** → **Create route table**
   - Name: `pub-rt` | VPC: `prod-vpc` → **Create**
2. Select `pub-rt` → **Routes tab → Edit routes → Add route**
   - Destination: `0.0.0.0/0` | Target: **Internet Gateway → prod-igw** → **Save**
3. **Subnet associations tab → Edit subnet associations**
   - Check ✅ `pub-subnet-1a` and `pub-subnet-1b` → **Save**

**Private App Route Table:**
1. Create route table: `priv-app-rt` | VPC: `prod-vpc`
2. Add route: Destination `0.0.0.0/0` | Target: **NAT Gateway → prod-nat-gw**
3. Associate: `priv-app-subnet-1a` and `priv-app-subnet-1b`

**Private DB Route Table (NO internet):**
1. Create route table: `priv-db-rt` | VPC: `prod-vpc`
2. **Do NOT add a 0.0.0.0/0 route** — DB subnets must be fully isolated
3. Associate: `priv-db-subnet-1a` and `priv-db-subnet-1b`

### 💻 CLI Commands

```bash
# ---- Public Route Table ----
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=pub-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PUB_RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_1A
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_1B

# ---- Private App Route Table ----
PRIV_APP_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=priv-app-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PRIV_APP_RT \
  --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_ID

aws ec2 associate-route-table --route-table-id $PRIV_APP_RT --subnet-id $PRIV_APP_1A
aws ec2 associate-route-table --route-table-id $PRIV_APP_RT --subnet-id $PRIV_APP_1B

# ---- Private DB Route Table (NO internet route) ----
PRIV_DB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=priv-db-rt}]' \
  --query 'RouteTable.RouteTableId' --output text)

aws ec2 associate-route-table --route-table-id $PRIV_DB_RT --subnet-id $PRIV_DB_1A
aws ec2 associate-route-table --route-table-id $PRIV_DB_RT --subnet-id $PRIV_DB_1B

echo "✅ Route tables configured:"
echo "  pub-rt: 0.0.0.0/0 → IGW"
echo "  priv-app-rt: 0.0.0.0/0 → NAT GW"
echo "  priv-db-rt: local only (no internet)"
```

**Route Table Summary:**

| Route Table | Destination | Target | Subnets |
|---|---|---|---|
| `pub-rt` | `10.0.0.0/16` | local | Public 1a, 1b |
| `pub-rt` | `0.0.0.0/0` | IGW | Public 1a, 1b |
| `priv-app-rt` | `10.0.0.0/16` | local | App 1a, 1b |
| `priv-app-rt` | `0.0.0.0/0` | NAT GW | App 1a, 1b |
| `priv-db-rt` | `10.0.0.0/16` | local | DB 1a, 1b |

---

## LAB 6 — Security Groups

Security Groups are **stateful** instance-level firewalls. Allowed inbound = return traffic automatically allowed.

### 🖥️ Console Steps

1. Left sidebar → **Security Groups** → **Create security group**
2. Create the 4 groups below. Set VPC to `prod-vpc` for all.

**SG-1: Bastion Host**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | SSH | 22 | Your IP (e.g., `203.x.x.x/32`) |
| Outbound | All | All | `0.0.0.0/0` |

**SG-2: Application Load Balancer**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | `0.0.0.0/0` |
| Inbound | HTTPS | 443 | `0.0.0.0/0` |
| Outbound | All | All | `0.0.0.0/0` |

**SG-3: App Servers (Private Subnet)**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | Custom TCP | 8080 | SG-2 (ALB SG — reference by SG ID) |
| Inbound | SSH | 22 | SG-1 (Bastion SG) |
| Outbound | All | All | `0.0.0.0/0` |

**SG-4: Database**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | MySQL/Aurora | 3306 | SG-3 (App Server SG) |
| Outbound | All | All | `0.0.0.0/0` |

### 💻 CLI Commands

```bash
# SG-1: Bastion
BASTION_SG=$(aws ec2 create-security-group \
  --group-name bastion-sg --description "Bastion Host" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG --protocol tcp --port 22 --cidr $(curl -s ifconfig.me)/32

# SG-2: ALB
ALB_SG=$(aws ec2 create-security-group \
  --group-name alb-sg --description "Application Load Balancer" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --ip-permissions '[{"IpProtocol":"tcp","FromPort":80,"ToPort":80,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]},{"IpProtocol":"tcp","FromPort":443,"ToPort":443,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]}]'

# SG-3: App Servers
APP_SG=$(aws ec2 create-security-group \
  --group-name app-sg --description "App Servers" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $APP_SG \
  --protocol tcp --port 8080 --source-group $ALB_SG
aws ec2 authorize-security-group-ingress --group-id $APP_SG \
  --protocol tcp --port 22 --source-group $BASTION_SG

# SG-4: Database
DB_SG=$(aws ec2 create-security-group \
  --group-name db-sg --description "Database" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $DB_SG \
  --protocol tcp --port 3306 --source-group $APP_SG

echo "✅ Security Groups created"
echo "  Bastion SG: $BASTION_SG"
echo "  ALB SG:     $ALB_SG"
echo "  App SG:     $APP_SG"
echo "  DB SG:      $DB_SG"
```

---

## LAB 7 — Network ACLs

NACLs are **stateless** subnet-level firewalls. You must allow **both inbound AND outbound** explicitly, including ephemeral return ports (1024–65535).

> **NACL vs Security Group:**
> - NACL: Stateless, processes rules by number order, supports DENY rules, applies to entire subnet
> - Security Group: Stateful, return traffic auto-allowed, allow-only, applies per instance

### 🖥️ Console Steps

**Public Subnet NACL:**
1. Left sidebar → **Network ACLs** → **Create network ACL**
   - Name: `pub-nacl` | VPC: `prod-vpc` → **Create**
2. **Inbound rules → Edit inbound rules:**

| Rule # | Type | Protocol | Port | Source | Action |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | `0.0.0.0/0` | Allow |
| 110 | HTTPS | TCP | 443 | `0.0.0.0/0` | Allow |
| 120 | SSH | TCP | 22 | Your IP/32 | Allow |
| 130 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

3. **Outbound rules → Edit outbound rules:**

| Rule # | Type | Protocol | Port | Destination | Action |
|---|---|---|---|---|---|
| 100 | HTTP | TCP | 80 | `0.0.0.0/0` | Allow |
| 110 | HTTPS | TCP | 443 | `0.0.0.0/0` | Allow |
| 120 | Custom TCP | TCP | 1024-65535 | `0.0.0.0/0` | Allow |
| * | All traffic | All | All | `0.0.0.0/0` | Deny |

4. **Subnet associations → Edit → Select `pub-subnet-1a` and `pub-subnet-1b`**

**Private Subnet NACL:**
- Create `priv-nacl` for VPC `prod-vpc`
- Inbound: Allow all from `10.0.0.0/16`, allow ephemeral ports from anywhere
- Outbound: Allow HTTP/HTTPS to internet (via NAT), allow all to VPC CIDR
- Associate with all 4 private subnets

### 💻 CLI Commands

```bash
# Public NACL
PUB_NACL=$(aws ec2 create-network-acl --vpc-id $VPC_ID \
  --tag-specifications 'ResourceType=network-acl,Tags=[{Key=Name,Value=pub-nacl}]' \
  --query 'NetworkAcl.NetworkAclId' --output text)

# Inbound rules
aws ec2 create-network-acl-entry --network-acl-id $PUB_NACL \
  --rule-number 100 --protocol tcp --port-range From=80,To=80 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress
aws ec2 create-network-acl-entry --network-acl-id $PUB_NACL \
  --rule-number 110 --protocol tcp --port-range From=443,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress
aws ec2 create-network-acl-entry --network-acl-id $PUB_NACL \
  --rule-number 120 --protocol tcp --port-range From=22,To=22 \
  --cidr-block $(curl -s ifconfig.me)/32 --rule-action allow --ingress
aws ec2 create-network-acl-entry --network-acl-id $PUB_NACL \
  --rule-number 130 --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow --ingress

# Outbound rules
aws ec2 create-network-acl-entry --network-acl-id $PUB_NACL \
  --rule-number 100 --protocol tcp --port-range From=80,To=443 \
  --cidr-block 0.0.0.0/0 --rule-action allow --egress
aws ec2 create-network-acl-entry --network-acl-id $PUB_NACL \
  --rule-number 110 --protocol tcp --port-range From=1024,To=65535 \
  --cidr-block 0.0.0.0/0 --rule-action allow --egress

# Associate with public subnets
aws ec2 replace-network-acl-association \
  --network-acl-id $PUB_NACL \
  --association-id $(aws ec2 describe-network-acls \
    --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.subnet-id,Values=$PUB_1A" \
    --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' --output text)

echo "✅ NACLs created — two firewall layers now active"
```

---

## LAB 8 — VPC Peering (Prod ↔ Dev)

VPC Peering creates a private connection between two VPCs. Traffic stays on the AWS backbone — no internet, no VPN.

> ⚠️ **CIDR ranges must NOT overlap.** `prod-vpc` = `10.0.0.0/16`, `dev-vpc` = `10.1.0.0/16` — these are fine.

### 🖥️ Console Steps

1. Create a dev VPC: **VPC → Your VPCs → Create VPC**
   - Name: `dev-vpc` | CIDR: `10.1.0.0/16`
2. Go to **VPC → Peering Connections → Create Peering Connection**
   - Name: `prod-dev-peering`
   - Requester VPC: `prod-vpc`
   - Accepter VPC: `dev-vpc` (same account, same region)
3. After creation → select the peering → **Actions → Accept Request**
4. **Update route tables in BOTH VPCs:**
   - In `pub-rt` (prod): Add route `10.1.0.0/16 → peering connection`
   - In dev-vpc route table: Add route `10.0.0.0/16 → peering connection`
5. **Update Security Groups** to allow traffic from the other VPC's CIDR

### 💻 CLI Commands

```bash
# Create Dev VPC
DEV_VPC=$(aws ec2 create-vpc --cidr-block 10.1.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=dev-vpc}]' \
  --query 'Vpc.VpcId' --output text)

# Create peering connection
PEERING_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $VPC_ID --peer-vpc-id $DEV_VPC \
  --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=prod-dev-peering}]' \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# Accept the peering request
aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_ID

# Add routes in prod-vpc pub-rt
aws ec2 create-route --route-table-id $PUB_RT \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID

# Add routes in prod-vpc priv-app-rt
aws ec2 create-route --route-table-id $PRIV_APP_RT \
  --destination-cidr-block 10.1.0.0/16 \
  --vpc-peering-connection-id $PEERING_ID

echo "✅ VPC Peering created: $PEERING_ID"
echo "   Prod (10.0.0.0/16) ↔ Dev (10.1.0.0/16)"
```

---

## LAB 9 — Launch EC2 & Validate Connectivity

### 🖥️ Console Steps

**Launch Bastion Host:**
1. **EC2 → Launch Instance**
   - Name: `bastion-host` | AMI: Amazon Linux 2023 | Type: `t2.micro`
   - VPC: `prod-vpc` | Subnet: `pub-subnet-1a` | Auto-assign public IP: **Enable**
   - Security Group: `bastion-sg` | Key pair: create `lab-key-pair`

**Launch Private App Instance:**
1. **EC2 → Launch Instance**
   - Name: `private-app` | AMI: Amazon Linux 2023 | Type: `t2.micro`
   - VPC: `prod-vpc` | Subnet: `priv-app-subnet-1a` | Auto-assign public IP: **Disable**
   - Security Group: `app-sg` | Key pair: `lab-key-pair`

### 💻 Validation Commands

```bash
# 1. SSH into Bastion Host
ssh -i lab-key-pair.pem ec2-user@<BASTION_PUBLIC_IP>

# 2. From Bastion — verify it has internet
curl https://ifconfig.me         # Shows bastion's public IP ✅
ping 8.8.8.8                     # Should work ✅

# 3. SSH into Private Instance via Bastion (ProxyJump)
ssh -i lab-key-pair.pem \
    -o "ProxyJump ec2-user@<BASTION_PUBLIC_IP>" \
    ec2-user@<PRIVATE_INSTANCE_IP>

# 4. From Private Instance — verify NAT Gateway is working
curl https://ifconfig.me         # Should show NAT GW's Elastic IP (not instance IP) ✅
sudo yum update -y               # Package update via NAT ✅
ping 8.8.8.8                     # Internet via NAT ✅

# 5. Verify no direct public IP on private instance
echo "My private IP:"
hostname -I                      # Shows only 10.0.3.x — no public IP ✅

# 6. From private instance — DB subnet should be unreachable from internet
# Launch a DB subnet instance, then from that instance:
curl https://google.com          # Should FAIL — expected ✅
ping 8.8.8.8                     # Should FAIL — expected ✅
```

**Expected Results:**

| Test | Expected | Reason |
|---|---|---|
| Bastion → internet | ✅ Works | Public subnet + IGW |
| Private app → internet | ✅ Works (via NAT) | Private subnet + NAT GW |
| Internet → private app directly | ❌ Fails | No public IP, SG blocks |
| DB instance → internet | ❌ Fails | No route in priv-db-rt |
| VPC peering ping | ✅ Works | Route + SG allow cross-VPC |

---

# ADVANCED LABS

> The following labs represent real-world skills used in production environments.
> Complete the Foundation Labs (1–9) before starting these.

---

## LAB 10 — VPC Flow Logs

VPC Flow Logs capture **metadata** of every packet flowing through your VPC — source IP, destination IP, port, protocol, bytes, and whether it was accepted or rejected. Essential for security audits, debugging, and compliance (SOC 2, PCI-DSS).

> Flow Logs capture metadata, NOT packet contents. They are not a packet sniffer.

### What Flow Logs Tell You
- Which Security Group is blocking traffic
- Who is trying to connect to your instances from suspicious IPs
- Data transfer patterns for cost optimization
- Evidence for compliance auditors

### 🖥️ Console Steps

**Step 1 — Create an S3 bucket for logs:**
1. Go to **S3 → Create bucket**
   - Name: `prod-vpc-flow-logs-YOURACCOUNT` | Region: `ap-south-1`
   - Block all public access: ✅ (keep on) → **Create bucket**

**Step 2 — Enable Flow Logs on VPC:**
1. Go to **VPC → Your VPCs → select `prod-vpc`**
2. **Actions → Create flow log**
3. Fill in:

| Field | Value |
|---|---|
| Filter | **All** (Accept + Reject) |
| Maximum aggregation interval | 1 minute (more granular) |
| Destination | Send to an Amazon S3 bucket |
| S3 bucket ARN | `arn:aws:s3:::prod-vpc-flow-logs-YOURACCOUNT` |
| Log record format | AWS default format |

4. Click **Create flow log**

**Step 3 — Also enable on a specific subnet for finer control:**
1. **VPC → Subnets → select `pub-subnet-1a`**
2. **Actions → Create flow log** → same settings

**Step 4 — Read the logs (Console):**
1. Go to **S3 → prod-vpc-flow-logs-YOURACCOUNT**
2. Navigate: `AWSLogs/ACCOUNT_ID/vpcflowlogs/ap-south-1/YEAR/MONTH/DAY/`
3. Download a log file → open in text editor
4. Each line = one traffic record

**Log Record Format:**
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes windowstart windowend action flowlogstatus

Example rejected connection:
2 123456789012 eni-abc12345 185.234.219.68 10.0.3.22 45231 22 6 1 40 1619000000 1619000060 REJECT OK
                ^^^           ^^^            ^^^       ^^^
                ENI           Source IP      Attacker  Your SSH port — BLOCKED ✅
```

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create S3 bucket for flow logs
aws s3api create-bucket \
  --bucket prod-vpc-flow-logs-${ACCOUNT_ID} \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Enable VPC Flow Logs to S3
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::prod-vpc-flow-logs-${ACCOUNT_ID} \
  --max-aggregation-interval 60

# Also send to CloudWatch for real-time alerting
aws logs create-log-group --log-group-name /vpc/prod-flowlogs

aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /vpc/prod-flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::${ACCOUNT_ID}:role/flowlogs-role

# Query flow logs with Athena (after 5 minutes of data)
# Go to Athena → create a table pointing to your S3 bucket
# SQL: SELECT srcaddr, dstaddr, action, count(*) FROM vpc_flow_logs
#      WHERE action='REJECT' GROUP BY 1,2,3 ORDER BY 4 DESC LIMIT 20

echo "✅ VPC Flow Logs enabled — all traffic metadata is being captured"
```

### Analyze Flow Logs with CloudWatch Insights

```bash
# Find who is trying to SSH (port 22) and being blocked
aws logs start-query \
  --log-group-name /vpc/prod-flowlogs \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, srcaddr, dstport, action
                  | filter dstport = 22 and action = "REJECT"
                  | stats count(*) as attempts by srcaddr
                  | sort attempts desc
                  | limit 20'
```

---

## LAB 11 — VPC Gateway Endpoint for S3

By default, private EC2 instances reach S3 through the NAT Gateway — meaning you pay for NAT data transfer charges. A **Gateway Endpoint** creates a direct private path to S3, completely bypassing the NAT Gateway and public internet. It's **free** and much faster.

> **Use case:** Your app servers upload/download large files to S3. Without an endpoint, all that traffic goes through NAT Gateway and you pay $0.045/GB for every byte. With a Gateway Endpoint: $0/GB.

### 🖥️ Console Steps

1. **VPC → Endpoints → Create Endpoint**
2. Fill in:

| Field | Value |
|---|---|
| Service category | AWS services |
| Service name | Search `com.amazonaws.ap-south-1.s3` → select **Gateway** type |
| VPC | `prod-vpc` |
| Route tables | Select **all route tables** (pub-rt, priv-app-rt, priv-db-rt) |
| Policy | Full access (or restrict to specific S3 buckets) |

3. Click **Create endpoint**

**Verify it worked:**
1. SSH into a private app instance
2. Try: `aws s3 ls` — should work without internet access!
3. Check route tables — you'll see a new route: `pl-xxxxx (S3 prefix list) → vpce-xxxxxxxx`

### 💻 CLI Commands

```bash
# Create Gateway Endpoint for S3
S3_ENDPOINT=$(aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids $PUB_RT $PRIV_APP_RT $PRIV_DB_RT \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=s3-gateway-endpoint}]' \
  --query 'VpcEndpoint.VpcEndpointId' --output text)

echo "✅ S3 Gateway Endpoint created: $S3_ENDPOINT"

# Verify — check that route tables now have the S3 prefix list route
aws ec2 describe-route-tables --route-table-ids $PRIV_APP_RT \
  --query 'RouteTables[0].Routes[].{Destination:DestinationPrefixListId, Target:GatewayId}' \
  --output table

# Test from private EC2 (S3 access should work WITHOUT internet)
# From private EC2:
# aws s3 ls --region ap-south-1     ← Should work even if NAT GW is deleted!
# aws s3 cp /tmp/test.txt s3://your-bucket/  ← Direct S3 access, free!

# Create restrictive endpoint policy (only allow specific bucket)
cat > endpoint-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
    "Resource": [
      "arn:aws:s3:::my-app-bucket",
      "arn:aws:s3:::my-app-bucket/*"
    ]
  }]
}
EOF
# Attach this policy to restrict which S3 buckets can be accessed via endpoint
```

---

## LAB 12 — VPC Interface Endpoints (PrivateLink)

Interface Endpoints use **AWS PrivateLink** to give private instances access to AWS services (SSM, Secrets Manager, CloudWatch, ECR, etc.) without using NAT Gateway or the internet. A private IP is created in your subnet that routes to the AWS service.

> **Why this matters:** Without Interface Endpoints, you need a NAT Gateway for private instances to reach SSM, pull Docker images from ECR, or fetch secrets. Interface Endpoints are cheaper at scale and more secure.

### Key Services Where Interface Endpoints Are Commonly Used

| Service | Endpoint Name |
|---|---|
| AWS Systems Manager (SSM) | `com.amazonaws.ap-south-1.ssm` |
| SSM Messages (Session Manager) | `com.amazonaws.ap-south-1.ssmmessages` |
| EC2 Messages | `com.amazonaws.ap-south-1.ec2messages` |
| Secrets Manager | `com.amazonaws.ap-south-1.secretsmanager` |
| ECR (Docker images) | `com.amazonaws.ap-south-1.ecr.dkr` + `ecr.api` |
| CloudWatch Logs | `com.amazonaws.ap-south-1.logs` |
| STS (IAM) | `com.amazonaws.ap-south-1.sts` |

### 🖥️ Console Steps (SSM Interface Endpoint)

1. **VPC → Endpoints → Create Endpoint**
2. Service name: search `ssm` → select `com.amazonaws.ap-south-1.ssm` (type: **Interface**)
3. VPC: `prod-vpc`
4. Subnets: select `priv-app-subnet-1a` and `priv-app-subnet-1b`
5. Enable private DNS: ✅ (routes all SSM calls automatically to endpoint)
6. Security Group: create a new SG — allow **HTTPS (443) from the VPC CIDR (10.0.0.0/16)**
7. Click **Create endpoint**
8. Repeat for `ssmmessages` and `ec2messages` (required for Session Manager without SSH)

### 💻 CLI Commands

```bash
# Create SG for endpoints (allow HTTPS from within VPC)
ENDPOINT_SG=$(aws ec2 create-security-group \
  --group-name endpoint-sg \
  --description "VPC Interface Endpoints" \
  --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $ENDPOINT_SG --protocol tcp --port 443 --cidr 10.0.0.0/16

# Create SSM Interface Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids $PRIV_APP_1A $PRIV_APP_1B \
  --security-group-ids $ENDPOINT_SG \
  --private-dns-enabled \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=ssm-endpoint}]'

# Create SSM Messages Endpoint (required for Session Manager)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.ssmmessages \
  --vpc-endpoint-type Interface \
  --subnet-ids $PRIV_APP_1A $PRIV_APP_1B \
  --security-group-ids $ENDPOINT_SG \
  --private-dns-enabled \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=ssmmessages-endpoint}]'

# Create Secrets Manager Endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.secretsmanager \
  --vpc-endpoint-type Interface \
  --subnet-ids $PRIV_APP_1A $PRIV_APP_1B \
  --security-group-ids $ENDPOINT_SG \
  --private-dns-enabled \
  --tag-specifications 'ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=secretsmanager-endpoint}]'

echo "✅ Interface Endpoints created"
echo "   Private instances can now reach SSM & Secrets Manager without NAT GW"

# Test: From a private EC2 with SSM role attached
# aws ssm start-session --target <instance-id>   ← Works without internet or SSH!
```

---

## LAB 13 — Application Load Balancer (ALB)

An ALB distributes HTTP/HTTPS traffic across multiple EC2 instances in private subnets. It sits in the public subnet and forwards to private app servers — the app servers never need public IPs.

### Architecture After This Lab
```
Internet → ALB (public subnets) → App Servers (private subnets, no public IPs)
```

### 🖥️ Console Steps

**Step 1 — Create Target Group:**
1. **EC2 → Target Groups → Create target group**
   - Target type: **Instances**
   - Target group name: `app-tg`
   - Protocol: HTTP | Port: 8080 | VPC: `prod-vpc`
   - Health check: HTTP | Path: `/health`
2. Register your private app EC2 instances → **Create**

**Step 2 — Create Application Load Balancer:**
1. **EC2 → Load Balancers → Create Load Balancer → Application Load Balancer**
2. Fill in:
   - Name: `prod-alb`
   - Scheme: **Internet-facing**
   - VPC: `prod-vpc`
   - Subnets: Select `pub-subnet-1a` and `pub-subnet-1b`
3. Security Group: `alb-sg`
4. Listener: HTTP (80) → Default action: **Forward to `app-tg`**
5. Click **Create load balancer**

**Step 3 — Add HTTPS Listener (Production Requirement):**
1. **EC2 → Load Balancers → select `prod-alb`**
2. **Listeners tab → Add listener**
   - Protocol: HTTPS | Port: 443
   - Certificate: Upload your ACM certificate or request a free one
   - Default action: Forward to `app-tg`
3. Add redirect: HTTP (80) → HTTPS (443)

### 💻 CLI Commands

```bash
# Step 1: Create Target Group
TG_ARN=$(aws elbv2 create-target-group \
  --name app-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

echo "Target Group: $TG_ARN"

# Step 2: Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name prod-alb \
  --subnets $PUB_1A $PUB_1B \
  --security-groups $ALB_SG \
  --scheme internet-facing \
  --type application \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)

echo "ALB DNS: $ALB_DNS"

# Step 3: Create Listener (HTTP → forward to target group)
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Step 4: Register your app instances in the target group
# Replace i-xxxx with your actual private instance IDs
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=i-1234567890abcdef0 Id=i-0987654321fedcba0

# Step 5: Test the ALB
curl http://$ALB_DNS/health   # Should return 200 from your app servers

echo "✅ ALB created — traffic flows: Internet → ALB → Private App Servers"
```

---

## LAB 14 — Route 53 Private Hosted Zone

Instead of hardcoding IP addresses (like `10.0.3.45`) in your apps, use internal DNS names (like `db.internal.technova.com`). Route 53 Private Hosted Zones resolve DNS names only within your VPC — they never go to the public internet.

> **Real-world use:** Your app config says `DB_HOST=db.internal.technova.com`. When the DB migrates to a new instance, you just update DNS — zero app code changes.

### 🖥️ Console Steps

1. **Route 53 → Hosted zones → Create hosted zone**
2. Fill in:

| Field | Value |
|---|---|
| Domain name | `internal.technova.com` |
| Type | **Private hosted zone** |
| VPCs to associate | Select `prod-vpc` |

3. Click **Create hosted zone**

4. **Create DNS records:**
   - **Record → Create record**
   - Record name: `db` | Type: `A` | Value: `10.0.5.10` (your RDS private IP) → **Create**
   - Record name: `app` | Type: `A` | Value: `10.0.3.22` → **Create**
   - Record name: `bastion` | Type: `A` | Value: `10.0.1.15` → **Create**

5. **Test from private EC2:**
   ```bash
   nslookup db.internal.technova.com      # Should resolve to 10.0.5.10
   curl http://app.internal.technova.com:8080/health
   ```

### 💻 CLI Commands

```bash
# Create Private Hosted Zone
HOSTED_ZONE_ID=$(aws route53 create-hosted-zone \
  --name internal.technova.com \
  --caller-reference $(date +%s) \
  --hosted-zone-config PrivateZone=true,Comment="Internal DNS" \
  --vpc VPCRegion=ap-south-1,VPCId=$VPC_ID \
  --query 'HostedZone.Id' --output text)

echo "Hosted Zone created: $HOSTED_ZONE_ID"

# Create DNS record for DB
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "db.internal.technova.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "10.0.5.10"}]
      }
    }, {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.internal.technova.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "10.0.3.22"}]
      }
    }]
  }'

# Associate with additional VPCs (e.g., dev-vpc can also use this DNS)
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --vpc VPCRegion=ap-south-1,VPCId=$DEV_VPC

# Test from inside the VPC (run on a private EC2)
# nslookup db.internal.technova.com     → 10.0.5.10
# nslookup app.internal.technova.com    → 10.0.3.22

echo "✅ Internal DNS working — no more hardcoded IPs in configs"
```

---

## LAB 15 — AWS Transit Gateway (Multi-VPC Hub-and-Spoke)

When you have **3+ VPCs**, VPC peering becomes a mesh — each VPC needs a peering connection to every other. With Transit Gateway (TGW), all VPCs connect to a central hub. Much simpler to manage.

```
Without TGW (mesh peering):          With TGW (hub-and-spoke):
VPC-A ←→ VPC-B                       VPC-A ──┐
VPC-A ←→ VPC-C                       VPC-B ──┤─→ Transit Gateway
VPC-B ←→ VPC-C                       VPC-C ──┘
(3 peering connections)               (3 attachments to 1 TGW)
```

> ⚠️ **Cost:** Transit Gateway = $0.05/hr per attachment + $0.02/GB. Delete after lab.

### 🖥️ Console Steps

1. **VPC → Transit Gateways → Create Transit Gateway**
   - Name: `prod-tgw`
   - ASN: leave default (64512)
   - DNS support: Enable ✅
   - VPN ECMP support: Enable ✅
   - Click **Create transit gateway** → wait for **Available**

2. **Attach prod-vpc:**
   - **VPC → Transit Gateway Attachments → Create Transit Gateway Attachment**
   - Transit Gateway: `prod-tgw` | Attachment type: VPC
   - VPC: `prod-vpc` | Subnets: select one subnet per AZ (e.g., `priv-app-subnet-1a`, `priv-app-subnet-1b`)
   - Click **Create**

3. **Attach dev-vpc** (same steps with `dev-vpc` and its subnets)

4. **Attach mgmt-vpc** (create a third VPC: `10.2.0.0/16`, then attach)

5. **Update route tables in each VPC:**
   - In `prod-vpc priv-app-rt`: Add route `10.1.0.0/16 → tgw-xxxxxxx` (dev-vpc)
   - In `prod-vpc priv-app-rt`: Add route `10.2.0.0/16 → tgw-xxxxxxx` (mgmt-vpc)
   - Same for the other VPCs toward prod-vpc and each other

### 💻 CLI Commands

```bash
# Create Transit Gateway
TGW_ID=$(aws ec2 create-transit-gateway \
  --description "TechNova Central TGW" \
  --options AmazonSideAsn=64512,DnsSupport=enable,VpnEcmpSupport=enable \
  --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=prod-tgw}]' \
  --query 'TransitGateway.TransitGatewayId' --output text)

echo "Transit Gateway: $TGW_ID"
echo "Waiting for TGW to become available (2-3 minutes)..."
aws ec2 wait transit-gateway-available --filters "Name=transit-gateway-id,Values=$TGW_ID"

# Create mgmt VPC for the lab
MGMT_VPC=$(aws ec2 create-vpc --cidr-block 10.2.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=mgmt-vpc}]' \
  --query 'Vpc.VpcId' --output text)

# Attach prod-vpc to TGW
PROD_ATTACH=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $VPC_ID \
  --subnet-ids $PRIV_APP_1A $PRIV_APP_1B \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=prod-attach}]' \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

# Attach dev-vpc to TGW
DEV_ATTACH=$(aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $DEV_VPC \
  --subnet-ids <dev-subnet-id> \
  --tag-specifications 'ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=dev-attach}]' \
  --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

# Add TGW route in prod-vpc private route table (reach dev-vpc)
aws ec2 create-route --route-table-id $PRIV_APP_RT \
  --destination-cidr-block 10.1.0.0/16 \
  --transit-gateway-id $TGW_ID

# Add TGW route in prod-vpc private route table (reach mgmt-vpc)
aws ec2 create-route --route-table-id $PRIV_APP_RT \
  --destination-cidr-block 10.2.0.0/16 \
  --transit-gateway-id $TGW_ID

echo "✅ Transit Gateway connecting 3 VPCs:"
echo "   prod-vpc (10.0.0.0/16) ↔ dev-vpc (10.1.0.0/16) ↔ mgmt-vpc (10.2.0.0/16)"

# Test connectivity (from prod private instance):
# ping 10.1.x.x   → dev-vpc instance (should succeed)
# ping 10.2.x.x   → mgmt-vpc instance (should succeed)
```

---

## LAB 16 — VPC Reachability Analyzer

Reachability Analyzer lets you **test network paths without sending actual traffic**. It analyzes your route tables, security groups, and NACLs and tells you exactly what's blocking connectivity — perfect for debugging "why can't A reach B?"

> No traffic is sent. It's a pure configuration analysis tool. Free for analysis, $0.10 per analysis run.

### 🖥️ Console Steps

**Test 1 — Can Bastion reach the private app instance?**
1. **VPC → Reachability Analyzer → Create and analyze path**
2. Fill in:
   - Source type: Instance | Source: `bastion-host`
   - Destination type: Instance | Destination: `private-app`
   - Protocol: TCP | Destination port: 22
3. Click **Create and analyze path**
4. Expected result: **Reachable ✅** — path: Bastion SG → app-sg allows port 22 from bastion SG

**Test 2 — Can the internet reach the private app directly? (Should FAIL)**
1. Create path:
   - Source type: Internet Gateway | Source: `prod-igw`
   - Destination type: Instance | Destination: `private-app`
   - Protocol: TCP | Port: 22
2. Expected result: **Not reachable ✅** — analyzer shows exactly WHICH rule blocks it (private subnet has no public route, SG doesn't allow internet)

**Test 3 — Can private app reach S3 endpoint?**
1. Source: `private-app` instance
2. Destination type: VPC Endpoint | Destination: your S3 endpoint
3. Expected: **Reachable** via VPC endpoint (not internet)

### 💻 CLI Commands

```bash
# Create a reachability analysis path
PATH_ID=$(aws ec2 create-network-insights-path \
  --source <bastion-instance-id> \
  --destination <private-app-instance-id> \
  --protocol tcp \
  --destination-port 22 \
  --tag-specifications 'ResourceType=network-insights-path,Tags=[{Key=Name,Value=bastion-to-app}]' \
  --query 'NetworkInsightsPath.NetworkInsightsPathId' --output text)

# Run the analysis
ANALYSIS_ID=$(aws ec2 start-network-insights-analysis \
  --network-insights-path-id $PATH_ID \
  --query 'NetworkInsightsAnalysis.NetworkInsightsAnalysisId' --output text)

echo "Analysis running: $ANALYSIS_ID"
sleep 30   # Wait for analysis to complete

# Get the result
aws ec2 describe-network-insights-analyses \
  --network-insights-analysis-ids $ANALYSIS_ID \
  --query 'NetworkInsightsAnalyses[0].{Status:Status,Reachable:NetworkPathFound,Explanation:ExplanationCodes}' \
  --output table

# If NOT reachable, get the full explanation (shows exact blocking rule)
aws ec2 describe-network-insights-analyses \
  --network-insights-analysis-ids $ANALYSIS_ID \
  --query 'NetworkInsightsAnalyses[0].Explanations'
```

---

## LAB 17 — AWS Network Firewall

AWS Network Firewall provides **stateful deep packet inspection** and allows you to block traffic based on domain names, IP reputation lists, and protocol signatures — things Security Groups and NACLs cannot do.

> **Use case:** Block all egress from your VPC except to approved domains. Even if a private server is compromised, it can't exfiltrate data to unknown IPs.

### Architecture Change for Network Firewall
```
Private EC2 → Private Subnet → Firewall Subnet → Network Firewall → NAT GW → Internet
(All egress traffic is inspected before leaving the VPC)
```

### 🖥️ Console Steps

**Step 1 — Create Firewall Subnets (one per AZ):**
1. **VPC → Subnets → Create subnet**
   - Name: `firewall-subnet-1a` | AZ: `ap-south-1a` | CIDR: `10.0.7.0/24`
   - Name: `firewall-subnet-1b` | AZ: `ap-south-1b` | CIDR: `10.0.8.0/24`

**Step 2 — Create Firewall Rule Group:**
1. **VPC → Network Firewall → Network Firewall rule groups → Create rule group**
2. Type: Stateful rule group | Capacity: 100
3. Name: `block-bad-domains`
4. Add rules — Domain list:
   - Action: **Deny** 
   - Protocols: HTTP, HTTPS
   - Domains: `malware.example.com`, `exfiltration.evil.com`
   - Alternatively: Set to ALLOW only `*.amazonaws.com`, `*.github.com`, `packages.python.org`

**Step 3 — Create Firewall Policy:**
1. **VPC → Network Firewall → Firewall policies → Create firewall policy**
2. Name: `prod-fw-policy`
3. Add your rule group under **Stateful rule groups**
4. Default actions: Stateful — Drop established, Alert established

**Step 4 — Create the Network Firewall:**
1. **VPC → Network Firewall → Firewalls → Create firewall**
2. Name: `prod-network-firewall` | VPC: `prod-vpc`
3. Subnets: `firewall-subnet-1a` (ap-south-1a), `firewall-subnet-1b` (ap-south-1b)
4. Firewall policy: `prod-fw-policy`

**Step 5 — Update Route Tables (redirect traffic through firewall):**
- Modify `priv-app-rt`: Change `0.0.0.0/0 → NAT GW` to `0.0.0.0/0 → Firewall endpoint`
- Add a new route in the public/firewall subnet back to NAT GW

### 💻 CLI Commands

```bash
# Step 1: Create firewall rule group (domain-based blocking)
aws network-firewall create-rule-group \
  --rule-group-name block-bad-domains \
  --type STATEFUL \
  --capacity 100 \
  --rule-group '{
    "RulesSource": {
      "RulesSourceList": {
        "Targets": [".malware.example.com", ".exfiltration.evil.com"],
        "TargetTypes": ["TLS_SNI", "HTTP_HOST"],
        "GeneratedRulesType": "DENYLIST"
      }
    }
  }'

# Step 2: Create allowlist rule group (allow only approved domains)
aws network-firewall create-rule-group \
  --rule-group-name allow-approved-domains \
  --type STATEFUL \
  --capacity 100 \
  --rule-group '{
    "RulesSource": {
      "RulesSourceList": {
        "Targets": [".amazonaws.com", ".github.com", ".pypi.org", ".npmjs.com"],
        "TargetTypes": ["TLS_SNI", "HTTP_HOST"],
        "GeneratedRulesType": "ALLOWLIST"
      }
    }
  }'

RULE_ARN=$(aws network-firewall describe-rule-group \
  --rule-group-name block-bad-domains --type STATEFUL \
  --query 'RuleGroupMetadata.Arn' --output text)

# Step 3: Create Firewall Policy
POLICY_ARN=$(aws network-firewall create-firewall-policy \
  --firewall-policy-name prod-fw-policy \
  --firewall-policy '{
    "StatelessDefaultActions": ["aws:forward_to_sfe"],
    "StatelessFragmentDefaultActions": ["aws:forward_to_sfe"],
    "StatefulRuleGroupReferences": [{"ResourceArn": "'$RULE_ARN'"}],
    "StatefulDefaultActions": ["aws:drop_established","aws:alert_established"]
  }' \
  --query 'FirewallPolicyMetadata.Arn' --output text)

# Step 4: Create the Network Firewall
aws network-firewall create-firewall \
  --firewall-name prod-network-firewall \
  --vpc-id $VPC_ID \
  --subnet-mappings SubnetId=$PRIV_APP_1A SubnetId=$PRIV_APP_1B \
  --firewall-policy-arn $POLICY_ARN

echo "✅ Network Firewall deployed — egress traffic is now inspected"
echo "   Any connection to blocked domains will be dropped"

# Test (from private EC2):
# curl https://amazonaws.com     → Should succeed ✅ (in allowlist)
# curl https://malware.example.com → Should FAIL ✅ (blocked)
```

---

## LAB 18 — GuardDuty + Flow Log Threat Detection

Amazon GuardDuty uses machine learning to analyze VPC Flow Logs, CloudTrail, and DNS logs to detect threats like: port scanning, cryptocurrency mining, unusual data exfiltration, and known malicious IP communication.

### 🖥️ Console Steps

**Enable GuardDuty:**
1. Search **GuardDuty** → **Get started → Enable GuardDuty**
2. Select **Enable** (free 30-day trial, then ~$1–$4/month depending on data volume)

**Create CloudWatch Alarms for GuardDuty Findings:**
1. **GuardDuty → Settings → Findings export options**
   - Export to S3: Set up S3 bucket for findings archive
2. **GuardDuty → Settings → Enable CloudWatch Events**
   - This sends all findings to EventBridge automatically
3. **EventBridge → Rules → Create rule**
   - Event source: AWS services → GuardDuty → GuardDuty Finding
   - Target: SNS topic (to send email alerts)

### 💻 CLI Commands

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Enable GuardDuty
DETECTOR_ID=$(aws guardduty create-detector --enable \
  --query 'DetectorId' --output text)

echo "GuardDuty enabled: $DETECTOR_ID"

# Create SNS topic for security alerts
ALERT_TOPIC=$(aws sns create-topic --name guardduty-alerts \
  --query 'TopicArn' --output text)

aws sns subscribe \
  --topic-arn $ALERT_TOPIC \
  --protocol email \
  --notification-endpoint admin@technova.com

# Create EventBridge rule to send HIGH/CRITICAL findings to SNS
aws events put-rule \
  --name guardduty-high-severity \
  --event-pattern '{
    "source": ["aws.guardduty"],
    "detail-type": ["GuardDuty Finding"],
    "detail": {
      "severity": [{"numeric": [">=", 7]}]
    }
  }' \
  --state ENABLED

aws events put-targets \
  --rule guardduty-high-severity \
  --targets Id=notify-security,Arn=$ALERT_TOPIC

# List current GuardDuty findings (if any)
aws guardduty list-findings --detector-id $DETECTOR_ID \
  --finding-criteria '{"Criterion":{"severity":{"Gte":7}}}' \
  --query 'FindingIds' --output table

echo "✅ GuardDuty enabled with email alerts for HIGH/CRITICAL findings"
echo "   GuardDuty analyzes Flow Logs + CloudTrail + DNS automatically"
```

**Simulate a GuardDuty Finding (for testing):**
```bash
# GuardDuty provides sample findings for testing
aws guardduty create-sample-findings \
  --detector-id $DETECTOR_ID \
  --finding-types "UnauthorizedAccess:EC2/SSHBruteForce" \
                  "Trojan:EC2/BlackholeTraffic" \
                  "Recon:EC2/PortProbeUnprotectedPort"

# Then check: GuardDuty → Findings — you'll see the sample threats
# You should also receive an email alert via SNS
```

---

## LAB 19 — Site-to-Site VPN Gateway

A Site-to-Site VPN connects your on-premises data center (or office network) to your AWS VPC over an encrypted IPsec tunnel. This is how companies with hybrid cloud setups enable their on-premise servers to communicate with AWS resources.

### Architecture
```
On-Premise Network (192.168.0.0/24)
        │
        │ IPsec VPN Tunnel (over internet)
        │
Virtual Private Gateway (VGW) → prod-vpc private subnets
```

> For this lab, you can simulate the "on-premises" side using a pfSense, StrongSwan, or OpenSwan EC2 instance in a different VPC/region acting as the customer gateway.

### 🖥️ Console Steps

**Step 1 — Create Customer Gateway (represents your on-premise router):**
1. **VPC → Customer Gateways → Create customer gateway**
   - Name: `on-prem-cgw`
   - BGP ASN: `65000`
   - IP address: `<your_on_premise_public_IP>` (or simulated EC2 public IP)
   - Type: ipsec.1

**Step 2 — Create Virtual Private Gateway:**
1. **VPC → Virtual Private Gateways → Create virtual private gateway**
   - Name: `prod-vgw` | ASN: Amazon default
2. After creation → **Actions → Attach to VPC** → `prod-vpc`

**Step 3 — Create Site-to-Site VPN Connection:**
1. **VPC → Site-to-Site VPN Connections → Create VPN connection**
   - Name: `prod-to-onprem-vpn`
   - Virtual private gateway: `prod-vgw`
   - Customer gateway: `on-prem-cgw` (existing)
   - Routing: Static
   - Static IP prefixes: `192.168.0.0/24` (your on-prem network)
2. Download the VPN configuration file after creation (has pre-shared keys and tunnel IPs)

**Step 4 — Update Route Tables:**
- In `priv-app-rt`: Add route `192.168.0.0/24 → vgw-xxxxxxxx`
- Enable **Route Propagation** on route tables (VGW can auto-populate routes)

### 💻 CLI Commands

```bash
# Step 1: Create Customer Gateway
CGW_ID=$(aws ec2 create-customer-gateway \
  --type ipsec.1 \
  --public-ip 203.0.113.100 \
  --bgp-asn 65000 \
  --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=on-prem-cgw}]' \
  --query 'CustomerGateway.CustomerGatewayId' --output text)

# Step 2: Create Virtual Private Gateway
VGW_ID=$(aws ec2 create-vpn-gateway \
  --type ipsec.1 \
  --amazon-side-asn 64512 \
  --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=prod-vgw}]' \
  --query 'VpnGateway.VpnGatewayId' --output text)

# Attach VGW to VPC
aws ec2 attach-vpn-gateway \
  --vpn-gateway-id $VGW_ID \
  --vpc-id $VPC_ID

echo "Waiting for VGW to attach..."
aws ec2 wait vpc-available --filters "Name=vpc-id,Values=$VPC_ID"

# Step 3: Create VPN Connection
VPN_ID=$(aws ec2 create-vpn-connection \
  --type ipsec.1 \
  --customer-gateway-id $CGW_ID \
  --vpn-gateway-id $VGW_ID \
  --options StaticRoutesOnly=true \
  --tag-specifications 'ResourceType=vpn-connection,Tags=[{Key=Name,Value=prod-to-onprem}]' \
  --query 'VpnConnection.VpnConnectionId' --output text)

# Step 4: Add static route (on-premise network)
aws ec2 create-vpn-connection-route \
  --vpn-connection-id $VPN_ID \
  --destination-cidr-block 192.168.0.0/24

# Enable route propagation on route tables
aws ec2 enable-vgw-route-propagation \
  --route-table-id $PRIV_APP_RT \
  --gateway-id $VGW_ID

# Download VPN configuration for the customer gateway device
aws ec2 describe-vpn-connections \
  --vpn-connection-ids $VPN_ID \
  --query 'VpnConnections[0].CustomerGatewayConfiguration' \
  --output text > vpn-config.xml

echo "✅ VPN created: $VPN_ID"
echo "   Download and apply vpn-config.xml to your on-premise router"
echo "   Once configured: on-premise (192.168.0.0/24) ↔ prod-vpc (10.0.0.0/16)"
```

---

## LAB 20 — Terraform IaC (Rebuild Everything as Code)

Infrastructure as Code is the professional standard. Once you've built the VPC manually in the console, rebuild it entirely with Terraform. This is the skill that makes you production-ready.

### Why Terraform?
- Version control your infrastructure (Git)
- Review changes before applying (`terraform plan`)
- Reproducibly create identical environments (dev, staging, prod)
- Destroy and recreate infrastructure safely

### Project Structure

```
vpc-terraform/
├── main.tf           # Provider config
├── vpc.tf            # VPC + subnets
├── gateways.tf       # IGW + NAT GW + Elastic IP
├── routes.tf         # Route tables + associations
├── security.tf       # Security Groups + NACLs
├── endpoints.tf      # VPC Endpoints
├── variables.tf      # Input variables
├── outputs.tf        # Output values (IDs, IPs, DNS)
└── terraform.tfvars  # Your variable values
```

### `variables.tf`

```hcl
variable "region" {
  default = "ap-south-1"
}

variable "vpc_cidr" {
  default = "10.0.0.0/16"
}

variable "environment" {
  default = "production"
}

variable "project" {
  default = "technova"
}

variable "azs" {
  default = ["ap-south-1a", "ap-south-1b"]
}

variable "public_subnets" {
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_app_subnets" {
  default = ["10.0.3.0/24", "10.0.4.0/24"]
}

variable "private_db_subnets" {
  default = ["10.0.5.0/24", "10.0.6.0/24"]
}
```

### `main.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "technova-terraform-state"
    key    = "vpc/terraform.tfstate"
    region = "ap-south-1"
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "Terraform"
      Owner       = "platform-team"
    }
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

### `vpc.tf`

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project}-${var.environment}-vpc"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = length(var.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.project}-pub-subnet-${substr(var.azs[count.index], -2, 2)}"
    Type = "public"
  }
}

# Private App Subnets
resource "aws_subnet" "private_app" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_app_subnets[count.index]
  availability_zone = var.azs[count.index]

  tags = {
    Name = "${var.project}-priv-app-subnet-${substr(var.azs[count.index], -2, 2)}"
    Type = "private-app"
  }
}

# Private DB Subnets
resource "aws_subnet" "private_db" {
  count             = length(var.azs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_db_subnets[count.index]
  availability_zone = var.azs[count.index]

  tags = {
    Name = "${var.project}-priv-db-subnet-${substr(var.azs[count.index], -2, 2)}"
    Type = "private-db"
  }
}
```

### `gateways.tf`

```hcl
# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "${var.project}-igw" }
}

# Elastic IP for NAT Gateway
resource "aws_eip" "nat" {
  domain     = "vpc"
  depends_on = [aws_internet_gateway.main]
  tags = { Name = "${var.project}-nat-eip" }
}

# NAT Gateway (in first public subnet)
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
  depends_on    = [aws_internet_gateway.main]
  tags = { Name = "${var.project}-nat-gw" }
}
```

### `routes.tf`

```hcl
# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = { Name = "${var.project}-pub-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private App Route Table
resource "aws_route_table" "private_app" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = { Name = "${var.project}-priv-app-rt" }
}

resource "aws_route_table_association" "private_app" {
  count          = length(aws_subnet.private_app)
  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private_app.id
}

# Private DB Route Table (NO internet route)
resource "aws_route_table" "private_db" {
  vpc_id = aws_vpc.main.id
  tags = { Name = "${var.project}-priv-db-rt" }
}

resource "aws_route_table_association" "private_db" {
  count          = length(aws_subnet.private_db)
  subnet_id      = aws_subnet.private_db[count.index].id
  route_table_id = aws_route_table.private_db.id
}

# S3 Gateway Endpoint (free, faster than NAT)
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = aws_vpc.main.id
  service_name      = "com.amazonaws.${var.region}.s3"
  vpc_endpoint_type = "Gateway"
  route_table_ids   = [
    aws_route_table.public.id,
    aws_route_table.private_app.id,
    aws_route_table.private_db.id
  ]
  tags = { Name = "${var.project}-s3-endpoint" }
}
```

### `security.tf`

```hcl
# Bastion Security Group
resource "aws_security_group" "bastion" {
  name        = "${var.project}-bastion-sg"
  description = "Bastion host access"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "SSH from admin"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["${chomp(data.http.my_ip.response_body)}/32"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project}-bastion-sg" }
}

# App Server Security Group
resource "aws_security_group" "app" {
  name        = "${var.project}-app-sg"
  description = "App server access"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "App traffic from ALB"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  ingress {
    description     = "SSH from bastion only"
    from_port       = 22
    to_port         = 22
    protocol        = "tcp"
    security_groups = [aws_security_group.bastion.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project}-app-sg" }
}

# Database Security Group
resource "aws_security_group" "db" {
  name        = "${var.project}-db-sg"
  description = "Database access"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "MySQL from app servers only"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  tags = { Name = "${var.project}-db-sg" }
}
```

### `outputs.tf`

```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_app_subnet_ids" {
  value = aws_subnet.private_app[*].id
}

output "private_db_subnet_ids" {
  value = aws_subnet.private_db[*].id
}

output "nat_gateway_ip" {
  value = aws_eip.nat.public_ip
}

output "bastion_sg_id" {
  value = aws_security_group.bastion.id
}

output "app_sg_id" {
  value = aws_security_group.app.id
}

output "db_sg_id" {
  value = aws_security_group.db.id
}
```

### Running Terraform

```bash
# Install Terraform
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform

# Initialize (downloads AWS provider)
terraform init

# Preview what will be created
terraform plan

# Apply (creates all resources)
terraform apply
# Type 'yes' when prompted

# Check what was created
terraform output

# Destroy everything (clean up after lab)
terraform destroy
# Type 'yes' when prompted
```

---

---

# EXPERT LABS

> These labs represent production-level scenarios encountered in real enterprise AWS environments. They build on all previous labs.

---

## LAB 21 — RDS Multi-AZ with DB Subnet Groups

A **DB Subnet Group** tells RDS which subnets (and therefore which AZs) it can use for the primary and standby database instances. This is the professional way to deploy any managed database in a VPC — it enforces network isolation, enables Multi-AZ failover, and gives you full control over which private subnets your database lives in.

> **Why this matters:** Without a DB Subnet Group in private subnets, your RDS instance may end up in a public subnet or in the wrong AZ. This lab shows the correct architecture.

### Architecture After This Lab
```
Internet → ALB → App Server (priv-app-subnet) → RDS Primary (priv-db-subnet-1a)
                                                       ↕ Sync
                                              RDS Standby (priv-db-subnet-1b)
```

### 🖥️ Console Steps

**Step 1 — Create DB Subnet Group:**
1. **AWS Console** → search **RDS** → click **RDS**
2. Left sidebar → **Subnet groups** → **Create DB subnet group**
3. Fill in:

| Field | Value |
|---|---|
| Name | `prod-db-subnet-group` |
| Description | `Private DB subnets for production RDS` |
| VPC | `prod-vpc` |

4. Under **Add subnets**:
   - Availability Zone: `ap-south-1a` → Subnet: `priv-db-subnet-1a (10.0.5.0/24)`
   - Availability Zone: `ap-south-1b` → Subnet: `priv-db-subnet-1b (10.0.6.0/24)`
5. Click **Create**

**Step 2 — Create RDS MySQL Instance (Multi-AZ):**
1. **RDS → Databases → Create database**
2. Fill in:

| Field | Value |
|---|---|
| Creation method | Standard create |
| Engine | MySQL 8.0 |
| Template | **Free tier** (for lab) or **Production** (enables Multi-AZ) |
| DB instance identifier | `prod-mysql` |
| Master username | `admin` |
| Master password | `YourSecurePass123!` |
| DB instance class | `db.t3.micro` |
| Storage type | gp3, 20 GB |
| **Multi-AZ deployment** | ✅ **Create a standby instance** |
| VPC | `prod-vpc` |
| DB subnet group | `prod-db-subnet-group` |
| Public access | ❌ **No** |
| VPC security group | Create new → name: `rds-sg` |
| Availability Zone | `ap-south-1a` |
| Database port | 3306 |

3. Under **Additional configuration**:
   - Initial database name: `appdb`
   - Enable ✅ **Automated backups** (retention: 7 days)
   - Enable ✅ **Enhanced monitoring**
   - Enable ✅ **Performance Insights**
4. Click **Create database** — takes ~5 minutes

**Step 3 — Configure RDS Security Group:**
1. After RDS is created, go to **EC2 → Security Groups**
2. Select `rds-sg` → **Inbound rules → Edit inbound rules → Add rule**
3. Type: **MySQL/Aurora** | Port: `3306` | Source: **Custom** → select `app-sg`
4. This ensures only app servers can reach the database — the internet cannot
5. Click **Save rules**

**Step 4 — Connect from App Server:**
1. SSH into your private app server (via bastion)
2. Install MySQL client:
   ```
   sudo yum install mysql -y
   ```
3. Get the RDS endpoint from: **RDS → Databases → prod-mysql → Connectivity & security → Endpoint**
4. Connect:
   ```
   mysql -h <rds-endpoint>.rds.amazonaws.com -u admin -p
   ```
5. Should connect successfully — database is private, no public IP ✅

**Step 5 — Test Multi-AZ Failover:**
1. **RDS → Databases → prod-mysql → Actions → Reboot**
2. Check ✅ **Reboot with failover** → **Confirm**
3. Watch the event log: **RDS → Events** — failover takes ~60 seconds
4. After failover, the standby in `ap-south-1b` becomes the new primary
5. The endpoint DNS name stays the same — your app reconnects automatically ✅

> ✅ **What you built:** A highly available database that automatically fails over to a different AZ if the primary fails, with no manual intervention and no DNS change required.

---

## LAB 22 — VPC IPAM (IP Address Manager)

**VPC IPAM** is AWS's centralized IP address planning tool. It lets you define top-level IP pools, allocate CIDR ranges to VPCs automatically, and detect overlapping IP ranges across your entire AWS organization before they cause problems. This is the tool used by companies managing dozens or hundreds of VPCs.

> **Why this matters:** Without IPAM, teams manually track VPC CIDRs in spreadsheets. Teams accidentally overlap CIDRs, breaking VPC peering and Transit Gateway. IPAM prevents this at scale.

### 🖥️ Console Steps

**Step 1 — Create an IPAM:**
1. **VPC → IPAM** (find in left sidebar under Network Manager section, or search "IPAM")
2. Click **Create IPAM**
3. Fill in:

| Field | Value |
|---|---|
| Name | `org-ipam` |
| Description | `Centralized IP management for all VPCs` |
| Operating regions | ✅ Add `ap-south-1` (Mumbai) |

4. Click **Create IPAM**

**Step 2 — Create the Top-Level IP Pool (RFC 1918 space):**
1. In IPAM → **Pools** → **Create pool**
2. Fill in:

| Field | Value |
|---|---|
| Name | `global-private-pool` |
| Description | `Top-level RFC 1918 address space` |
| Address family | IPv4 |
| IPAM scope | Private scope |
| Locale | None (top-level) |
| CIDR | `10.0.0.0/8` |

3. Click **Create pool**

**Step 3 — Create a Regional Pool (Mumbai):**
1. **IPAM → Pools → Create pool**
2. Fill in:

| Field | Value |
|---|---|
| Name | `mumbai-pool` |
| Source (parent pool) | `global-private-pool` |
| Locale | `ap-south-1` |
| CIDR | `10.0.0.0/12` (allocated from parent) |
| Allocation min/default/max netmask | `/24` / `/16` / `/8` |

3. Click **Create pool**

**Step 4 — Create Environment Sub-pools:**
1. **Create pool** → Name: `prod-pool`
   - Source: `mumbai-pool`
   - CIDR: `10.0.0.0/14`
   - Tags: `Environment=Production`
2. **Create pool** → Name: `dev-pool`
   - Source: `mumbai-pool`
   - CIDR: `10.4.0.0/14`
   - Tags: `Environment=Development`
3. **Create pool** → Name: `staging-pool`
   - Source: `mumbai-pool`
   - CIDR: `10.8.0.0/14`
   - Tags: `Environment=Staging`

**Step 5 — Allocate a CIDR to a VPC from IPAM:**
1. **VPC → Your VPCs → Create VPC**
2. Resources to create: **VPC only**
3. Name: `new-project-vpc`
4. IPv4 CIDR: **IPAM-allocated IPv4 CIDR**
5. IPAM pool: select `prod-pool`
6. Netmask length: `/16`
7. Click **Create VPC** — IPAM automatically assigns the next available `/16` from the pool

**Step 6 — View IPAM Dashboard (Detect Overlaps):**
1. **IPAM → Resource discovery** → **Discover resources** to scan existing VPCs
2. **IPAM → Pools** → click `prod-pool` → **Allocations tab**
   - Shows every VPC CIDR allocated, which account, which region
   - Overlaps are flagged in red automatically
3. **IPAM → Insights** → view free space, utilization percentage per pool

> ✅ **What you built:** A centralized IP management system that prevents duplicate CIDRs and gives full visibility into your entire organization's IP space.

---

## LAB 23 — AWS Client VPN

**AWS Client VPN** is a managed OpenVPN service that lets your employees securely connect to AWS VPC resources from their laptops — without needing a bastion host or exposing resources to the internet. Think of it as a corporate VPN endpoint managed by AWS.

> **Use case:** Developers working from home need to access internal tools (GitLab, Jira, RDS databases) running in private subnets. With Client VPN, they connect via the VPN client and get a private IP, as if they were on the office network.

### 🖥️ Console Steps

**Step 1 — Generate Server and Client Certificates (using AWS Certificate Manager):**
1. **AWS Certificate Manager (ACM) → Import a certificate**
   - For this lab, use ACM Private CA or use the mutual certificate auth flow
   - Alternatively: In the console go to **ACM → Private certificate authority → Create CA**
   - Name: `client-vpn-ca` | Type: Root | Key algorithm: RSA 2048
   - Click **Create CA** → **Install CA certificate** → **Generate root certificate**

2. After CA is active, **ACM → Request → Request a private certificate**:
   - For server cert: domain `server.vpn.internal` → issued by `client-vpn-ca`
   - For client cert: domain `client1.vpn.internal` → same CA

**Step 2 — Create the Client VPN Endpoint:**
1. **VPC → Client VPN endpoints → Create client VPN endpoint**
2. Fill in:

| Field | Value |
|---|---|
| Name | `prod-client-vpn` |
| Description | `Employee VPN access to prod-vpc` |
| Client IPv4 CIDR | `172.16.0.0/22` (VPN client IP pool — must NOT overlap VPC) |
| Server certificate ARN | Select the `server.vpn.internal` cert from ACM |
| Authentication type | **Mutual authentication** |
| Client certificate ARN | Select `client1.vpn.internal` cert |
| Enable connection logging | ✅ Yes → CloudWatch log group `/vpn/client-connections` |
| Enable split tunnel | ✅ Yes (only VPC traffic goes through VPN — internet stays local) |
| VPC ID | `prod-vpc` |
| Security group | Create new `client-vpn-sg` |
| VPN port | 443 |
| DNS servers | `10.0.0.2` (VPC DNS resolver) |

3. Click **Create client VPN endpoint**

**Step 3 — Associate with Subnets:**
1. Select your new endpoint → **Associations tab → Associate**
2. VPC: `prod-vpc` | Subnet: `priv-app-subnet-1a`
3. Click **Associate** → repeat for `priv-app-subnet-1b`
4. Status changes from `pending-associate` to `available` (~2 min)

**Step 4 — Configure Authorization Rules:**
1. **Authorization rules tab → Add authorization rule**
2. Destination network: `10.0.0.0/16` (entire prod-vpc)
3. Grant access to: **All users**
4. Description: `Allow VPN users to access prod-vpc`
5. Click **Add authorization rule**

**Step 5 — Download and Distribute VPN Config:**
1. Select endpoint → **Download client configuration**
2. A `.ovpn` file is downloaded
3. Open the file in a text editor — add the client certificate and key contents between `<cert>...</cert>` and `<key>...</key>` tags
4. Import the modified `.ovpn` into **AWS VPN Client** (desktop app) or any OpenVPN-compatible client

**Step 6 — Connect and Verify:**
1. Install **AWS VPN Client** on your laptop (download from aws.amazon.com/vpnclient)
2. **File → Manage Profiles → Add Profile** → select your `.ovpn` file
3. Click **Connect**
4. After connecting, open a terminal: run `ping 10.0.3.x` (private app subnet IP) → should succeed ✅
5. You now have private connectivity to all VPC resources without SSH or bastion hosts

> ✅ **What you built:** A managed corporate VPN where employees can securely reach private AWS resources from anywhere, with full connection logging and no server infrastructure to manage.

---

## LAB 24 — PrivateLink — Expose Your Own Service

AWS PrivateLink lets you expose a service running in YOUR VPC to consumers in OTHER VPCs — without peering, without public internet, and without giving consumers access to your entire network. This is the backbone of AWS marketplace services and is used by SaaS companies to offer private connectivity to enterprise customers.

> **Use case:** Your team runs an internal authentication API. Other teams in other VPCs or accounts need to call it — securely, privately, with no internet exposure and no cross-VPC routing.

### Architecture
```
Consumer VPC → Interface Endpoint → PrivateLink → NLB → Your Service (Provider VPC)
(Other team's account)                                    (Your account, your NLB)
```

### 🖥️ Console Steps

**Step 1 — Set Up the Service Side (Provider VPC):**

1. You need a **Network Load Balancer (NLB)** in front of your service:
   - **EC2 → Load Balancers → Create load balancer → Network Load Balancer**
   - Name: `internal-api-nlb` | Scheme: **Internal** | IP type: IPv4
   - VPC: `prod-vpc`
   - Mappings: `priv-app-subnet-1a` (AZ: ap-south-1a) + `priv-app-subnet-1b`
   - Listeners: TCP port `80`
   - Target group: Create new → Target type: Instances → Port: 8080 → register your app servers
   - Click **Create load balancer**

**Step 2 — Create the Endpoint Service:**
1. **VPC → Endpoint services → Create endpoint service**
2. Fill in:

| Field | Value |
|---|---|
| Name | `my-internal-api-service` |
| Load balancer type | Network |
| Available load balancers | Select `internal-api-nlb` |
| Acceptance required | ✅ Yes (you manually approve each consumer) |
| Enable private DNS name | ✅ Yes — enter `api.internal.technova.com` |

3. Click **Create**
4. Note the **Service name** (looks like: `com.amazonaws.vpce.ap-south-1.vpce-svc-xxxxxxxxxx`)

**Step 3 — Allow Specific AWS Accounts (Principals):**
1. Select your endpoint service → **Allow principals** tab
2. Click **Allow principals**
3. Add: `arn:aws:iam::CONSUMER_ACCOUNT_ID:root`
4. Click **Allow principals** — only this account can now request a connection to your service

**Step 4 — Consumer Side (Create Interface Endpoint to Your Service):**
1. In the consumer VPC (could be `dev-vpc` in same account for this lab)
2. **VPC → Endpoints → Create endpoint**
3. Fill in:

| Field | Value |
|---|---|
| Service category | **Other endpoint services** |
| Service name | Paste the service name from Step 2 |

4. Click **Verify service** → should show ✅ Service name verified
5. VPC: `dev-vpc` | Subnets: consumer's private subnets
6. Enable private DNS: ✅ Yes (consumers use your custom DNS name)
7. Security Group: allow TCP 80/443 from consumer VPC CIDR
8. Click **Create endpoint** — status shows `pending acceptance`

**Step 5 — Accept the Connection (Provider Side):**
1. Back in provider account: **VPC → Endpoint services → my-internal-api-service**
2. **Endpoint connections tab** → select the pending connection
3. **Actions → Accept endpoint connection**
4. Status changes to `available`

**Step 6 — Test the Private Connection:**
1. SSH into an EC2 in the consumer VPC
2. Run: `curl http://api.internal.technova.com/health`
3. Traffic goes: Consumer EC2 → Interface Endpoint ENI → PrivateLink → NLB → App Server
4. At no point does traffic leave the AWS network ✅
5. The app server never sees the consumer's IP — it sees the NLB IP ✅

> ✅ **What you built:** A production-grade PrivateLink service. This is how AWS services like S3, SSM, and DynamoDB are consumed privately, and how SaaS companies offer private connectivity to their enterprise customers.

---

## LAB 25 — Traffic Mirroring

**Traffic Mirroring** copies actual network packets from an EC2 instance's ENI and sends them to another EC2 running a network analysis tool (like Suricata, Zeek, or Wireshark). Unlike Flow Logs (which only capture metadata), Traffic Mirroring captures the actual packet contents — enabling deep security analysis and intrusion detection.

> **Use case:** Your security team suspects a compromised EC2 is exfiltrating data. Traffic Mirroring lets you capture all packets from that instance and analyze them in real-time with an IDS (Intrusion Detection System).

### 🖥️ Console Steps

**Step 1 — Launch the Mirror Target (Analysis EC2):**
1. **EC2 → Launch Instance**
   - Name: `ids-analyzer` | AMI: Amazon Linux 2023 | Type: `t3.medium`
   - VPC: `prod-vpc` | Subnet: `priv-app-subnet-1a` | Auto-assign public IP: **Disable**
   - Security Group: Create new `ids-sg` — allow UDP 4789 (VXLAN) from the VPC CIDR `10.0.0.0/16`
   - Key pair: `lab-key-pair`
2. Click **Launch instance**

**Step 2 — Install a Packet Analyzer on the IDS Instance:**
1. SSH into `ids-analyzer` via bastion
2. Install tcpdump:
   ```
   sudo yum install tcpdump -y
   ```
3. Set it listening on the VXLAN interface (mirrored traffic arrives encapsulated):
   ```
   sudo tcpdump -i eth0 udp port 4789 -w /tmp/mirror-capture.pcap
   ```

**Step 3 — Create a Mirror Target:**
1. **VPC → Traffic Mirroring → Mirror targets → Create traffic mirror target**
2. Fill in:

| Field | Value |
|---|---|
| Name | `ids-target` |
| Description | `IDS EC2 analyzer instance` |
| Target type | **Network interface** |
| Network interface | Select the ENI of `ids-analyzer` |

3. Click **Create**

**Step 4 — Create a Mirror Filter (What to Capture):**
1. **VPC → Traffic Mirroring → Mirror filters → Create traffic mirror filter**
2. Name: `capture-http-ssh`
3. Under **Inbound rules → Add rule**:

| Rule # | Traffic direction | Protocol | Source CIDR | Destination Port | Action |
|---|---|---|---|---|---|
| 100 | Inbound | TCP | `0.0.0.0/0` | 22 | Accept |
| 110 | Inbound | TCP | `0.0.0.0/0` | 80 | Accept |
| 120 | Inbound | TCP | `0.0.0.0/0` | 443 | Accept |

4. Under **Outbound rules → Add rule**:
   - Rule 100: Outbound | All traffic | `0.0.0.0/0` | Accept
5. Click **Create**

**Step 5 — Create the Mirror Session (Source → Target):**
1. **VPC → Traffic Mirroring → Mirror sessions → Create traffic mirror session**
2. Fill in:

| Field | Value |
|---|---|
| Name | `monitor-private-app` |
| Mirror source | Select the ENI of your `private-app` EC2 |
| Mirror target | `ids-target` |
| Session number | `1` |
| Mirror filter | `capture-http-ssh` |
| Packet length | `0` (full packets) |
| Virtual network ID | `12345` (VXLAN ID — any number) |

3. Click **Create** — mirroring starts immediately

**Step 6 — Verify Mirror is Working:**
1. SSH into `private-app` (via bastion) and generate some HTTP traffic:
   ```
   curl http://httpbin.org/get
   ```
2. On `ids-analyzer`, check the tcpdump capture:
   ```
   sudo tcpdump -i eth0 udp port 4789 -c 20 -v
   ```
3. You'll see VXLAN-encapsulated packets containing the actual HTTP request ✅
4. To analyze the pcap: download `/tmp/mirror-capture.pcap` and open in Wireshark on your laptop

**Step 7 — Test Intrusion Detection with Suricata (Optional):**
1. On `ids-analyzer`: `sudo yum install suricata -y`
2. Configure Suricata to listen on VXLAN port 4789
3. Download community rules: `sudo suricata-update`
4. Start Suricata: `sudo suricata -c /etc/suricata/suricata.yaml -i eth0`
5. Any suspicious traffic (port scans, known malware signatures) will create alerts in `/var/log/suricata/eve.json`

> ✅ **What you built:** A real-time packet capture and intrusion detection pipeline. This is how security operations centers monitor for threats in production AWS environments.

---

## LAB 26 — Egress-Only Internet Gateway (IPv6)

An **Egress-Only Internet Gateway** is the IPv6 equivalent of a NAT Gateway. It allows IPv6 instances in private subnets to initiate outbound connections to the internet, while preventing the internet from initiating connections back. Since IPv6 addresses are all globally routable (no NAT needed), you need this gateway to enforce private behavior.

> **Why this matters:** AWS is moving toward IPv6. New services like VPC Lattice are IPv6-native. Understanding Egress-Only IGW is essential for modern AWS networking.

### 🖥️ Console Steps

**Step 1 — Add an IPv6 CIDR Block to the VPC:**
1. **VPC → Your VPCs → select `prod-vpc`**
2. **Actions → Edit CIDRs**
3. Under **IPv6 CIDRs** → **Add new IPv6 CIDR**
4. Select: **Amazon-provided IPv6 CIDR block** (AWS assigns a `/56` block from Amazon's range)
5. Click **Save**
6. Note the assigned IPv6 CIDR (looks like: `2406:da1a:xxx::/56`)

**Step 2 — Add IPv6 CIDRs to Subnets:**
1. **VPC → Subnets → select `priv-app-subnet-1a`**
2. **Actions → Edit IPv6 CIDRs**
3. Click **Add IPv6 CIDR** — AWS auto-assigns a `/64` from the VPC's `/56` block
4. Click **Save** → repeat for `priv-app-subnet-1b`
5. For public subnets (`pub-subnet-1a`, `pub-subnet-1b`):
   - Same process — add `/64` CIDRs
   - Also: **Actions → Edit subnet settings** → ✅ Enable **auto-assign IPv6 address**

**Step 3 — Create the Egress-Only Internet Gateway:**
1. **VPC → Egress-only internet gateways → Create egress-only internet gateway**
2. Fill in:

| Field | Value |
|---|---|
| Name tag | `prod-eigw` |
| VPC | `prod-vpc` |

3. Click **Create egress-only internet gateway**

**Step 4 — Update Route Tables for IPv6:**

*Public subnets — allow both inbound and outbound IPv6 (via regular IGW):*
1. **VPC → Route Tables → select `pub-rt`**
2. **Routes → Edit routes → Add route**
   - Destination: `::/0` (all IPv6) | Target: **Internet Gateway → prod-igw**
3. Click **Save changes**

*Private app subnets — outbound IPv6 only (via Egress-Only IGW):*
1. **VPC → Route Tables → select `priv-app-rt`**
2. **Routes → Edit routes → Add route**
   - Destination: `::/0` | Target: **Egress only internet gateway → prod-eigw**
3. Click **Save changes**

*DB subnets — NO IPv6 internet route (stays as-is)*

**Step 5 — Enable IPv6 on Security Groups:**
1. **EC2 → Security Groups → select `app-sg`**
2. **Inbound rules → Edit inbound rules → Add rule**
   - Type: All traffic | Source: `10.0.0.0/16` (keep existing)
3. **Outbound rules → Edit outbound rules → Add rule**
   - Type: All traffic | Destination: `::/0` (allows IPv6 outbound)
4. Save

**Step 6 — Launch an EC2 with IPv6 and Test:**
1. **EC2 → Launch Instance**
   - Name: `ipv6-test` | Subnet: `priv-app-subnet-1a`
   - Network settings → Assign IPv6 address: ✅ Enable
2. SSH in via bastion, then:
   ```
   # Check IPv6 address
   ip -6 addr show eth0

   # Test IPv6 outbound (should work via Egress-Only IGW)
   curl -6 https://ipv6.google.com

   # Verify internet can NOT initiate connection to this instance
   # Try to ping the instance's IPv6 from outside → should fail
   ```
3. Outbound to internet works ✅ | Inbound from internet blocked ✅

> ✅ **What you built:** A dual-stack VPC where public instances have full IPv6 internet access, and private instances have outbound-only IPv6 access — mirroring the NAT Gateway pattern but for IPv6 at no cost.

---

## LAB 27 — VPC Lattice (Service-to-Service Networking)

**AWS VPC Lattice** is a modern application networking service that handles service-to-service communication across VPCs and AWS accounts without requiring VPC peering, Transit Gateway, or PrivateLink. It provides built-in authentication, authorization, observability, and traffic management at the service layer.

> **Use case:** You have a microservices architecture — checkout service, payment service, and inventory service — each in different VPCs or accounts. VPC Lattice connects them with auth policies and traffic routing, without network-level complexity.

### Architecture
```
checkout-service (prod-vpc) → VPC Lattice Service Network → payment-service (finance-vpc)
                                                           → inventory-service (ops-vpc)
```

### 🖥️ Console Steps

**Step 1 — Create a Service Network:**
1. **AWS Console** → search **VPC Lattice** (or find under VPC → VPC Lattice)
2. **Service networks → Create service network**
3. Fill in:

| Field | Value |
|---|---|
| Name | `prod-service-network` |
| Auth policy type | **AWS IAM** (enables request-level auth) |

4. Click **Create service network**

**Step 2 — Create a Lattice Service (Payment API):**
1. **VPC Lattice → Services → Create service**
2. Fill in:

| Field | Value |
|---|---|
| Name | `payment-api` |
| Auth policy type | **AWS IAM** |

3. Click **Next** → **Custom domain configuration** (optional — skip for lab)
4. Click **Next** → **Routing**:
   - Protocol: **HTTP** | Port: `8080`
   - Default action: Forward to target group
5. **Create target group**:
   - Target type: **Instances** | Name: `payment-api-tg`
   - Protocol: HTTP | Port: 8080 | VPC: `prod-vpc`
   - Register targets: select your app EC2 instances
6. Click **Create service**

**Step 3 — Associate the Service with the Service Network:**
1. **VPC Lattice → Service networks → prod-service-network**
2. **Services tab → Associate services → Add**
3. Select `payment-api` → **Associate**

**Step 4 — Associate VPCs with the Service Network:**
1. **prod-service-network → VPC associations tab → Associate VPC**
2. Fill in:

| Field | Value |
|---|---|
| VPC | `prod-vpc` |
| Security groups | Create new `lattice-sg` allowing port 8080 from `10.0.0.0/16` |

3. Click **Associate** → repeat for `dev-vpc` (consumer VPC)

**Step 5 — Set an Auth Policy (IAM Authorization):**
1. **VPC Lattice → Services → payment-api → Auth policy tab**
2. Click **Edit** → paste this policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::YOUR_ACCOUNT_ID:role/checkout-service-role"
    },
    "Action": "vpc-lattice-svcs:Invoke",
    "Resource": "*"
  }]
}
```

3. Click **Save changes** — now only the checkout service's IAM role can call the payment API

**Step 6 — Test Service-to-Service Communication:**
1. SSH into an EC2 in the consumer VPC (`dev-vpc`)
2. Get the Lattice service DNS name: **VPC Lattice → Services → payment-api → DNS name** (looks like: `payment-api-xxxxxxxxxx.xxxxxx.vpc-lattice-svcs.us-east-1.vpce.amazonaws.com`)
3. From the consumer EC2 with the correct IAM role:
   ```
   curl http://<lattice-service-dns>:8080/health
   ```
4. Request is authenticated via SigV4, routed through Lattice, delivered to your service ✅

**Step 7 — View Lattice Access Logs:**
1. **VPC Lattice → Service networks → prod-service-network → Monitoring tab**
2. Enable access logs → destination: **CloudWatch Logs** → log group `/lattice/prod-service-network`
3. All requests are logged with: source VPC, IAM principal, response code, latency ✅

> ✅ **What you built:** A zero-trust service mesh that connects microservices across VPC boundaries with per-request IAM authorization — no VPN, no peering, no managing nginx reverse proxies.

---

## LAB 28 — CloudWatch Network Monitor

**CloudWatch Network Monitor** provides active network health monitoring between your AWS resources and on-premises endpoints. It continuously probes network paths and raises alarms when latency or packet loss exceeds thresholds — giving you early warning of connectivity degradation before users are impacted.

> **Use case:** You have a hybrid cloud setup with a Site-to-Site VPN. You want to be alerted the moment VPN tunnel latency exceeds 50ms or packet loss exceeds 1%, so you can proactively investigate before the on-premises application teams report issues.

### 🖥️ Console Steps

**Step 1 — Create a Network Monitor:**
1. **CloudWatch → Network monitoring → Network monitors → Create monitor** (search "Network Monitor" in CloudWatch sidebar)
2. Fill in:

| Field | Value |
|---|---|
| Monitor name | `prod-network-monitor` |
| Aggregation period | 30 seconds |

3. Click **Next**

**Step 2 — Add Probes (Source → Destination pairs):**

*Probe 1 — Monitor VPC internal latency (App → DB):*
1. Click **Add probe**

| Field | Value |
|---|---|
| Source | `prod-vpc` |
| Source subnet | `priv-app-subnet-1a` |
| Destination | IP address → `10.0.5.10` (RDS private IP) |
| Destination port | `3306` |
| Protocol | **TCP** |
| Packet size | 56 bytes |

2. Click **Add probe**

*Probe 2 — Monitor internet reachability from public subnet:*
1. Click **Add probe**

| Field | Value |
|---|---|
| Source | `prod-vpc` |
| Source subnet | `pub-subnet-1a` |
| Destination | IP address → `8.8.8.8` |
| Destination port | `53` |
| Protocol | **UDP** |

2. Click **Add probe**

*Probe 3 — Monitor VPN tunnel health (if VPN lab completed):*
1. Click **Add probe**

| Field | Value |
|---|---|
| Source | `prod-vpc` |
| Source subnet | `priv-app-subnet-1a` |
| Destination | IP address → `192.168.0.1` (on-prem side) |
| Protocol | **ICMP** |

3. Click **Next → Create monitor**

**Step 3 — View Real-Time Metrics:**
1. **CloudWatch → Network monitoring → Network monitors → prod-network-monitor**
2. **Probes tab** — each probe shows:
   - **Round-trip time (RTT)** in milliseconds
   - **Packet loss** percentage
   - **Jitter** (variation in latency)
3. **Monitor health** tab — overall health: `Healthy` / `Degraded` / `Unavailable`

**Step 4 — Create CloudWatch Alarms for Network Health:**

*Alarm for high latency:*
1. **CloudWatch → Alarms → Create alarm**
2. **Select metric → AWS/NetworkMonitor → PerProbeMetrics → RTT**
3. Select the probe for `priv-app-subnet-1a → 10.0.5.10`
4. Fill in:

| Field | Value |
|---|---|
| Statistic | Average |
| Period | 1 minute |
| Threshold type | Static |
| Condition | **Greater than 50** (milliseconds) |
| Datapoints to alarm | 3 out of 5 |

5. **Next → Actions** → Send notification to SNS topic `network-alerts`
6. Alarm name: `high-db-latency` → **Create alarm**

*Alarm for packet loss:*
1. **Create alarm** → metric: `AWS/NetworkMonitor → PacketLoss`
2. Same probe | Condition: **Greater than 1** (percent)
3. Alarm name: `packet-loss-detected` → **Create alarm**

**Step 5 — Create a Network Monitor Dashboard:**
1. **CloudWatch → Dashboards → Create dashboard** → name: `Network-Health`
2. **Add widget → Line graph** → metrics: NetworkMonitor RTT for all probes
3. **Add widget → Line graph** → metrics: NetworkMonitor PacketLoss for all probes
4. **Add widget → Alarm status** → select both alarms created above
5. **Save dashboard**

**Step 6 — Test the Alarm (Simulate Packet Loss):**
1. Temporarily **remove the route** for DB subnet in `priv-app-rt` (just for test):
   - **VPC → Route Tables → priv-app-rt → Routes → Edit routes**
   - Delete the `10.0.5.0/24` route → **Save**
2. Wait 2–3 minutes — the monitor detects packet loss, the alarm triggers
3. You receive an email notification ✅
4. Restore the route → monitor returns to healthy

> ✅ **What you built:** A proactive network health monitoring system that alerts your team the moment connectivity degrades — before users or applications feel the impact.

---

## LAB 29 — AWS Network Access Analyzer

**Network Access Analyzer** helps you identify unintended network access to your AWS resources. Unlike Reachability Analyzer (which tests specific A→B paths), Network Access Analyzer takes a scope-based approach: you define what access should NOT exist (e.g., "no resource should be directly reachable from the internet without going through a load balancer") and it finds all violations across your entire VPC.

> **Use case:** Before a security audit, run Network Access Analyzer to automatically find any EC2 instances, RDS databases, or ECS tasks that are unintentionally reachable from the internet.

### 🖥️ Console Steps

**Step 1 — Create a Network Access Scope:**
1. **VPC → Network Analysis → Network Access Analyzer → Network access scopes → Create network access scope**
2. Name: `internet-to-ec2-direct-check`
3. Description: `Find EC2 instances reachable from internet without going through a load balancer`
4. Under **Match conditions**:

*Source:*
- Source type: **Internet gateways**
- Select: `prod-igw`

*Destination:*
- Destination type: **EC2 instances**
- Tags filter: `Environment=Production` (or leave blank to check all)

*Exclusion — paths through:*
- ✅ **Exclude paths that traverse:** Elastic Load Balancers
- This means: find instances reachable from internet WITHOUT going through an ALB/NLB

5. Click **Create network access scope**

**Step 2 — Run the Analysis:**
1. Select `internet-to-ec2-direct-check` → **Analyze**
2. Choose: `prod-vpc` | Click **Start analysis**
3. Wait ~2 minutes for results

**Step 3 — Interpret the Results:**
1. **Findings tab** — each finding shows:
   - The EC2 instance that is directly reachable from internet
   - The network path (IGW → Route Table → Subnet → Security Group → Instance)
   - The specific security group rule that allows the access
2. Expected for your lab:
   - `bastion-host` — reachable from internet via SSH ✅ (intentional)
   - If any **private app** or **DB** instances appear → ❌ **Security misconfiguration found!**

**Step 4 — Create a Scope for DB Isolation Check:**
1. **Create network access scope**
2. Name: `no-internet-to-rds`

*Source:* Internet gateways → `prod-igw`

*Destination:* Network interfaces with tag `aws:cloudformation:stack-name=prod-rds` OR destination ports `3306, 5432, 1433` (database ports)

3. **Analyze** → any findings = database is unintentionally exposed

**Step 5 — Create a Scope for Cross-VPC Exposure:**
1. **Create network access scope**
2. Name: `unauthorized-cross-vpc-access`

*Source:* VPC → `dev-vpc` (should only reach limited resources in prod)

*Destination:* VPC → `prod-vpc` subnets `priv-db-subnet-1a`, `priv-db-subnet-1b`

3. Run analysis — any finding means dev-vpc can reach prod databases (policy violation!)

**Step 6 — Automate Periodic Analysis (EventBridge Schedule):**
1. **EventBridge → Rules → Create rule**
2. Name: `weekly-network-access-scan`
3. Event source: **Schedule** → Cron: `0 9 ? * MON *` (every Monday at 9 AM)
4. Target: **AWS API → EC2 → StartNetworkInsightsAnalysis** with your scope ARN
5. Save → every Monday, the analysis runs automatically and findings appear in the console

> ✅ **What you built:** An automated blast-radius assessment that continuously checks for unintended network exposure — the equivalent of a weekly network security audit, running hands-free.

---

## LAB 30 — Prefix Lists + Managed Security Group Rules at Scale

As your AWS environment grows from 1 VPC to 50 VPCs, managing Security Group rules becomes a nightmare. Every time you add a new subnet, you need to update dozens of Security Groups. **Managed Prefix Lists** solve this: define a set of CIDRs once, reference that list in all your Security Groups, and update only the prefix list when IPs change — all Security Groups update automatically.

> **Real scenario:** Your company has 20 office locations. Each office's public IP must be allowed SSH access to bastion hosts in 15 VPCs. Without prefix lists: 20 × 15 = 300 Security Group rules to update every time an office IP changes. With prefix lists: update 1 prefix list → all 15 VPCs update instantly.

### 🖥️ Console Steps

**Step 1 — Create a Customer-Managed Prefix List:**
1. **VPC → Managed prefix lists → Create prefix list**
2. Fill in:

| Field | Value |
|---|---|
| Name | `corporate-office-ips` |
| Max entries | `25` (maximum number of CIDRs this list will ever hold) |
| Address family | IPv4 |

3. Under **Entries → Add entry**:
   - CIDR: `203.0.113.10/32` | Description: `Mumbai HQ`
   - CIDR: `198.51.100.20/32` | Description: `Delhi Office`
   - CIDR: `192.0.2.30/32` | Description: `Bangalore Office`
   - CIDR: `10.200.0.0/24` | Description: `VPN connected clients`
4. Click **Create prefix list**
5. Note the **Prefix list ID** (looks like: `pl-xxxxxxxxxx`)

**Step 2 — Use the Prefix List in a Security Group:**
1. **EC2 → Security Groups → select `bastion-sg`**
2. **Inbound rules → Edit inbound rules**
3. Find the existing rule allowing SSH from your IP → **Delete** it (we'll replace with prefix list)
4. **Add rule**:

| Type | Protocol | Port | Source |
|---|---|---|---|
| SSH | TCP | 22 | **Custom** → paste `pl-xxxxxxxxxx` (your prefix list ID) |

5. Click **Save rules**
6. The rule now shows: `SSH | TCP | 22 | pl-xxxxxxxxxx (corporate-office-ips)`

**Step 3 — Reference the Same Prefix List in Multiple Security Groups and VPCs:**
1. **EC2 → Security Groups → select `app-sg`**
2. **Inbound rules → Edit inbound rules → Add rule**
   - Type: HTTPS | Port: 443 | Source: `pl-xxxxxxxxxx`
3. Repeat for any other Security Groups that need access from corporate IPs
4. All these SGs now share ONE source of truth for corporate IPs ✅

**Step 4 — Add a New Office IP (Watch All SGs Update Instantly):**
1. A new office opens in Chennai
2. **VPC → Managed prefix lists → corporate-office-ips → Actions → Modify**
3. **Add entry**: CIDR: `172.16.50.0/24` | Description: `Chennai Office`
4. **Save changes** → Version increments automatically
5. Check ALL Security Groups that reference this prefix list — they all now allow Chennai IPs ✅
6. **Zero Security Group edits were required in any individual SG**

**Step 5 — Share a Prefix List Across AWS Accounts (Resource Access Manager):**
1. **VPC → Managed prefix lists → corporate-office-ips → Actions → Share**
2. This opens **AWS Resource Access Manager (RAM)**
3. **Create resource share**:
   - Name: `shared-office-ips`
   - Resources: `pl-xxxxxxxxxx`
   - Principals: Add the AWS Account IDs of other teams/accounts
4. Other accounts accept the share → they can now reference your prefix list in their own Security Groups
5. When you update the prefix list, ALL accounts' Security Groups that reference it update instantly ✅

**Step 6 — Use AWS-Managed Prefix Lists (S3, CloudFront, DynamoDB):**
1. AWS provides its own managed prefix lists for services — no configuration needed:
   - **VPC → Managed prefix lists** → look for `com.amazonaws.ap-south-1.s3` (S3 IP ranges)
   - `com.amazonaws.global.cloudfront.origin-facing` (CloudFront IPs)
2. **Use case:** Allow HTTPS only from CloudFront to your ALB:
   - **ALB Security Group → Inbound rules → Add rule**
   - HTTPS | Port 443 | Source: `pl-xxxxxxxxxx` (CloudFront managed prefix list)
   - This ensures ONLY CloudFront can reach your ALB — direct internet access is blocked
   - AWS automatically updates the CloudFront IP ranges in the prefix list ✅

> ✅ **What you built:** A scalable, maintainable Security Group management system. This is how enterprise AWS environments handle hundreds of Security Groups across dozens of VPCs — one source of truth, zero drift.

---

## Common Pitfalls & Fixes

| Pitfall | Symptom | Fix |
|---|---|---|
| NAT Gateway in private subnet | Private instances can't reach internet | NAT GW **must** be in a PUBLIC subnet |
| Forgot to attach IGW | Public instances unreachable | Verify IGW is attached to VPC |
| Public subnet not auto-assigning IPs | EC2 has no public IP | Enable auto-assign on public subnet |
| NACL missing ephemeral ports | Connections timeout (requests go through but responses are blocked) | Add rule for ports 1024-65535 both inbound and outbound |
| Overlapping CIDR for VPC peering | Peering fails | Plan CIDRs upfront — use a CIDR tracker |
| Same route table for public + private | Private instances can route to internet directly | Always separate route tables |
| Security Group referencing IP not SG | SG rules break when instances scale | Always reference Security Group IDs as sources |
| DNS Hostnames disabled | Internal DNS doesn't resolve | Enable on VPC → Actions → Edit VPC settings |
| Unattached Elastic IP after NAT deletion | Ongoing billing | Release EIPs when not in use |
| No VPC Flow Logs | Can't debug or audit network issues | Enable from day 1 |
| Default VPC used in production | Hard to audit, shared with all defaults | Always create a custom VPC for prod workloads |
| Not tagging resources | Impossible to manage at scale | Tag everything: Name, Environment, Owner, Team |
| Interface Endpoint missing security group | HTTPS traffic to endpoint blocked | Create SG allowing 443 from VPC CIDR, attach to endpoint |
| TGW route not added in VPC route table | Cross-VPC traffic doesn't route | Add explicit route for remote VPC CIDR pointing to TGW |
| VPN tunnel shows "DOWN" | IPsec not negotiated | Ensure on-premise router uses correct pre-shared key and cipher suites from config file |

---

## Interview Questions

**Conceptual:**
- What is the difference between a Security Group and a NACL?
- Can a VPC span multiple regions? *(No — region-scoped)*
- What is the difference between an Internet Gateway and a NAT Gateway?
- Why do private instances need NAT and not just an IGW?
- What happens if two peered VPCs have overlapping CIDRs?
- How does VPC Peering differ from Transit Gateway?
- What is a VPC Endpoint and when would you use Gateway vs Interface type?
- What are ephemeral ports and why do they matter in NACLs?

**Scenario-Based:**
- Your private EC2 can't reach the internet. Walk me through troubleshooting steps.
- A developer can't SSH to a private EC2. What do you check?
- How would you design a VPC for a 3-tier web application?
- How do you give an EC2 in a private subnet access to S3 without using NAT?
- Your app is making a large number of API calls to S3 and NAT Gateway costs are high. What do you do?
- You have 10 VPCs that all need to talk to each other. How do you design the connectivity?
- A security audit finds that an S3 bucket is publicly accessible. How does VPC architecture prevent this?

**Advanced / Architecture:**
- When would you use AWS Network Firewall vs Security Groups vs NACLs?
- How do you implement zero-trust networking in AWS VPC?
- Explain PrivateLink vs VPC Peering vs Transit Gateway — when would you use each?
- How do you design a hub-and-spoke VPC architecture for a multi-account AWS organization?
- How would you migrate an on-premises application to AWS with minimal downtime using a VPN?
- How does Route 53 Private Hosted Zone differ from public DNS, and what are the use cases?

---

## Cleanup (Avoid Billing)

**Delete in this exact order to avoid dependency errors:**

```bash
# 1. Terminate EC2 instances first
aws ec2 terminate-instances --instance-ids <instance-ids>

# 2. Delete NAT Gateway (wait for deletion before releasing EIP)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_ID
# Wait: aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_GW_ID

# 3. Release Elastic IP
aws ec2 release-address --allocation-id $EIP_ALLOC

# 4. Delete Network Firewall (if created)
aws network-firewall delete-firewall --firewall-name prod-network-firewall

# 5. Delete Transit Gateway attachments, then Transit Gateway
aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $PROD_ATTACH
aws ec2 delete-transit-gateway --transit-gateway-id $TGW_ID

# 6. Delete VPN Connection and Virtual Private Gateway
aws ec2 delete-vpn-connection --vpn-connection-id $VPN_ID
aws ec2 detach-vpn-gateway --vpn-gateway-id $VGW_ID --vpc-id $VPC_ID
aws ec2 delete-vpn-gateway --vpn-gateway-id $VGW_ID

# 7. Delete VPC Endpoints
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <endpoint-ids>

# 8. Delete VPC Peering Connection
aws ec2 delete-vpc-peering-connection --vpc-peering-connection-id $PEERING_ID

# 9. Detach and delete Internet Gateway
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

# 10. Delete route tables (except main)
aws ec2 delete-route-table --route-table-id $PUB_RT
aws ec2 delete-route-table --route-table-id $PRIV_APP_RT
aws ec2 delete-route-table --route-table-id $PRIV_DB_RT

# 11. Delete Security Groups (DB first, then App, then ALB, then Bastion)
aws ec2 delete-security-group --group-id $DB_SG
aws ec2 delete-security-group --group-id $APP_SG
aws ec2 delete-security-group --group-id $ALB_SG
aws ec2 delete-security-group --group-id $BASTION_SG

# 12. Delete subnets
for SUBNET in $PUB_1A $PUB_1B $PRIV_APP_1A $PRIV_APP_1B $PRIV_DB_1A $PRIV_DB_1B; do
  aws ec2 delete-subnet --subnet-id $SUBNET
done

# 13. Delete VPCs
aws ec2 delete-vpc --vpc-id $VPC_ID
aws ec2 delete-vpc --vpc-id $DEV_VPC

# 14. Disable GuardDuty
aws guardduty delete-detector --detector-id $DETECTOR_ID

# 15. Delete Route 53 Hosted Zone (delete records first, then zone)
aws route53 delete-hosted-zone --id $HOSTED_ZONE_ID
```

> ✅ After cleanup, verify no charges are accruing in **AWS Billing → Cost Explorer**.

---

## Lab Completion Checklist

**Foundation Labs:**
- [ ] VPC created with `10.0.0.0/16` CIDR, DNS hostnames enabled
- [ ] 6 subnets across 2 AZs (public x2, app x2, db x2)
- [ ] Auto-assign public IP enabled on public subnets
- [ ] Internet Gateway created and attached to VPC
- [ ] Elastic IP allocated, NAT Gateway created in public subnet
- [ ] Public Route Table: `0.0.0.0/0 → IGW`, associated with public subnets
- [ ] Private App Route Table: `0.0.0.0/0 → NAT GW`
- [ ] Private DB Route Table: local only (no internet route)
- [ ] Security Groups: bastion, ALB, app, DB with least-privilege rules
- [ ] NACLs configured with ephemeral port rules (1024-65535)
- [ ] VPC Peering prod ↔ dev, routes updated in both VPCs
- [ ] EC2 bastion launched, SSH from internet works
- [ ] Private instance accessible only via bastion
- [ ] `curl ifconfig.me` from private instance shows NAT GW EIP
- [ ] DB instance has NO internet access confirmed

**Expert Labs:**
- [ ] RDS Multi-AZ created in private DB subnets, Multi-AZ failover tested in ~60 seconds
- [ ] VPC IPAM set up with hierarchical pools (global → regional → environment), VPC CIDR auto-allocated
- [ ] Client VPN endpoint created, `.ovpn` downloaded, private VPC resources accessible from laptop
- [ ] PrivateLink service exposed via NLB, consumer endpoint created and connection accepted, cross-VPC call verified
- [ ] Traffic Mirroring session configured, VXLAN packets captured on IDS instance, Wireshark/tcpdump analysis done
- [ ] IPv6 CIDR added to VPC and subnets, Egress-Only IGW created, private instance can reach IPv6 internet (outbound only)
- [ ] VPC Lattice service network created, service associated, auth policy enforced, cross-VPC service call verified
- [ ] CloudWatch Network Monitor probes deployed, latency and packet-loss alarms configured and tested
- [ ] Network Access Analyzer scopes created, internet-to-EC2 and DB-isolation findings reviewed
- [ ] Prefix List created with office IPs, referenced in multiple Security Groups, new IP added to list with zero SG edits


- [ ] VPC Flow Logs enabled → S3 + CloudWatch, log records visible
- [ ] S3 Gateway Endpoint created, private EC2 accesses S3 without NAT
- [ ] SSM + Secrets Manager Interface Endpoints created
- [ ] ALB created in public subnets, forwarding to private app instances
- [ ] Route 53 Private Hosted Zone with internal DNS records, resolving from VPC
- [ ] Transit Gateway connecting 3 VPCs, cross-VPC traffic verified
- [ ] Reachability Analyzer run — both reachable and not-reachable paths confirmed
- [ ] Network Firewall deployed with domain-based blocking rules
- [ ] GuardDuty enabled with sample findings triggered and email alerts received
- [ ] VPN Gateway created and configuration downloaded
- [ ] Entire VPC rebuilt from scratch using Terraform (`terraform apply` succeeds)
- [ ] All resources destroyed to avoid billing

---

## Learning Roadmap

| Phase | Timeline | Focus |
|---|---|---|
| Phase 1 — Foundations | Week 1–2 | CIDR notation, subnetting, OSI Layer 3/4 |
| Phase 2 — Core Labs | Week 3–4 | Complete Labs 1–9 twice (console then CLI) |
| Phase 3 — Advanced Labs | Week 5–6 | Labs 10–16 (Flow Logs, Endpoints, ALB, Route 53) |
| Phase 4 — Expert Labs I | Week 7–8 | Labs 17–20 (Firewall, GuardDuty, VPN, Terraform) |
| Phase 5 — Expert Labs II | Week 9–10 | Labs 21–25 (RDS, IPAM, Client VPN, PrivateLink, Traffic Mirror) |
| Phase 6 — Expert Labs III | Week 11–12 | Labs 26–30 (IPv6, Lattice, Network Monitor, Access Analyzer, Prefix Lists) |
| Phase 7 — Certification | Week 13–14 | AWS SAA-C03 / ANS-C01 (VPC = ~30% of ANS exam) |

### Learning Resources

| Resource | Type | Link/Notes |
|---|---|---|
| AWS VPC Official Docs | Documentation | docs.aws.amazon.com/vpc |
| AWS Well-Architected Framework | Best Practices | aws.amazon.com/architecture/well-architected |
| Stephane Maarek — AWS SAA Course | Video Course | Udemy |
| Adrian Cantrill — AWS SAA Course | Video Course | learn.cantrill.io |
| TechWorld with Nana — Networking Basics | YouTube | Free |
| AWS Skill Builder | Labs & Quizzes | skillbuilder.aws (Free + Paid) |
| Terraform AWS VPC Module | IaC Reference | registry.terraform.io |
| AWS re:Invent VPC Deep Dive | Video | YouTube — search "aws reinvent vpc deep dive" |
| Tutorials Dojo (Jon Bonso) | Practice Exams | For SAA-C03 / ANS-C01 |
| CIDR Calculator | Tool | cidr.xyz |

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

**Certifications That Validate This Knowledge:**
- **AWS Cloud Practitioner** — basic VPC concepts
- **AWS Solutions Architect Associate (SAA-C03)** — full VPC coverage, most relevant
- **AWS Solutions Architect Professional** — complex multi-VPC, hybrid connectivity
- **AWS Advanced Networking Specialty** — expert-level deep dive on all topics

---

---

*Built for: Raja (DevOps Engineer) | Region: ap-south-1 (Mumbai)*  
*Version: 4.0 — Master Consolidated Guide | Foundation + Advanced + Expert Labs (30 Labs Total)*  
*Estimated Cost: < $5 for all labs | Services: VPC, Subnets, IGW, NAT, Endpoints, TGW, ALB, Route 53, Network Firewall, GuardDuty, Client VPN, PrivateLink, Traffic Mirroring, VPC Lattice, IPAM, Network Monitor, Terraform*  
*Merged from: AWS_VPC_Complete_Advanced_Guide.md + AWS_VPC_DeepDive_README.md + AWS_VPC_DeepDive_README_one.md*
