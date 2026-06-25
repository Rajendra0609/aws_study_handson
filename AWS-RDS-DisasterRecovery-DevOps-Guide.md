# AWS RDS, Disaster Recovery & DevOps — Complete Hands-On Training Guide

> A zero-to-production guide written for a beginner. Every topic explains **what it is**, **why it matters in real projects**, **the approach**, and gives **Console (UI)** and **CLI** steps.
>
> Read top-to-bottom the first time. Later, use it as a reference runbook.

---

## Table of Contents

1. [How to Use This Guide](#1-how-to-use-this-guide)
2. [Prerequisites & Environment Setup](#2-prerequisites--environment-setup)
3. [Core Concepts You Must Know First](#3-core-concepts-you-must-know-first)
4. [Amazon RDS Fundamentals](#4-amazon-rds-fundamentals)
5. [Networking & Security for RDS](#5-networking--security-for-rds)
6. [Creating Your First RDS Instance](#6-creating-your-first-rds-instance)
7. [Connecting to the Database](#7-connecting-to-the-database)
8. [High Availability: Multi-AZ](#8-high-availability-multi-az)
9. [Scaling: Read Replicas](#9-scaling-read-replicas)
10. [Backups: Automated & Manual Snapshots](#10-backups-automated--manual-snapshots)
11. [Point-in-Time Recovery (PITR)](#11-point-in-time-recovery-pitr)
12. [Encryption, Secrets & Parameter/Option Groups](#12-encryption-secrets--parameteroption-groups)
13. [Monitoring & Alerting](#13-monitoring--alerting)
14. [Maintenance, Patching & Cost Control](#14-maintenance-patching--cost-control)
15. [Disaster Recovery (DR) — Concepts](#15-disaster-recovery-dr--concepts)
16. [DR Strategy 1: Backup & Restore](#16-dr-strategy-1-backup--restore)
17. [DR Strategy 2: Pilot Light](#17-dr-strategy-2-pilot-light)
18. [DR Strategy 3: Warm Standby](#18-dr-strategy-3-warm-standby)
19. [DR Strategy 4: Multi-Site Active/Active](#19-dr-strategy-4-multi-site-activeactive)
20. [Cross-Region DR for RDS (Hands-On)](#20-cross-region-dr-for-rds-hands-on)
21. [Aurora Global Database (Advanced DR)](#21-aurora-global-database-advanced-dr)
22. [AWS Backup — Centralized DR](#22-aws-backup--centralized-dr)
23. [DevOps: Infrastructure as Code (IaC)](#23-devops-infrastructure-as-code-iac)
24. [DevOps: CI/CD Pipeline for DB Changes](#24-devops-cicd-pipeline-for-db-changes)
25. [DevOps: Automation & Runbooks](#25-devops-automation--runbooks)
26. [DR Testing & Game Days](#26-dr-testing--game-days)
27. [RDS Proxy & Connection Management](#27-rds-proxy--connection-management)
28. [IAM Database Authentication](#28-iam-database-authentication)
29. [AWS DMS — Database Migration Service](#29-aws-dms--database-migration-service)
30. [Aurora Deep-Dive: Cloning, Backtrack, Serverless v2](#30-aurora-deep-dive-cloning-backtrack-serverless-v2)
31. [Cross-Region Network Connectivity](#31-cross-region-network-connectivity)
32. [Events, Notifications, Tagging & Governance](#32-events-notifications-tagging--governance)
33. [Compliance & Well-Architected Reliability](#33-compliance--well-architected-reliability)
34. [Troubleshooting Common Issues](#34-troubleshooting-common-issues)
35. [Real-Time Project Blueprint](#35-real-time-project-blueprint)
36. [Cleanup (Avoid Surprise Bills)](#36-cleanup-avoid-surprise-bills)
37. [Glossary & Cheat Sheet](#37-glossary--cheat-sheet)

---

## 1. How to Use This Guide

- **Console steps** = click-by-click in the AWS web console. Best for learning and one-off tasks.
- **CLI steps** = repeatable commands. Best for automation, documentation, and real projects.
- In real projects you rarely click in the console for production changes — you use **IaC** (Terraform/CloudFormation) and **CI/CD**. We cover all three so you understand the full picture.

**Recommended learning order:** Read concepts → do Console once → repeat with CLI → then automate with IaC.

---

## 2. Prerequisites & Environment Setup

### 2.1 What you need
- An AWS account (use a **non-root IAM user** for daily work).
- A credit/debit card on the account (RDS has a free tier: `db.t3.micro` / `db.t2.micro`, 750 hrs/month for 12 months).
- A computer with internet access.

### 2.2 Create an IAM admin user (Console)
1. Sign in as root → search **IAM** → **Users** → **Create user**.
2. Name: `admin-yourname`. Enable **console access**.
3. Attach policy **AdministratorAccess** (for learning; tighten later).
4. Create user, save the sign-in URL and password.
5. **Enable MFA** on this user (Security credentials tab).

> In real projects, never use `AdministratorAccess` for apps. Use **least-privilege** roles. More in [Section 5](#5-networking--security-for-rds).

### 2.3 Install the AWS CLI v2
**Windows (PowerShell):**
```powershell
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
aws --version
```
**macOS:**
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```
**Linux:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws --version
```

### 2.4 Create access keys & configure the CLI
1. IAM → your user → **Security credentials** → **Create access key** → choose **CLI**.
2. Configure:
```bash
aws configure
# AWS Access Key ID:     <paste>
# AWS Secret Access Key: <paste>
# Default region name:   us-east-1
# Default output format:  json
```
3. Verify:
```bash
aws sts get-caller-identity
```

### 2.5 Optional but recommended tools
- A SQL client: **DBeaver** (free, all engines) or `mysql` / `psql` CLI.
- **Terraform** (`https://developer.hashicorp.com/terraform/downloads`).
- **Git** for version control.

---

## 3. Core Concepts You Must Know First

| Term | Plain-English meaning |
|------|----------------------|
| **Region** | A geographic area (e.g., `us-east-1` = N. Virginia). Your data lives here. |
| **Availability Zone (AZ)** | An isolated data center within a Region. A Region has 3+ AZs. |
| **VPC** | Your private network in AWS. |
| **Subnet** | A slice of the VPC, tied to one AZ. **Public** = internet-reachable, **Private** = not. |
| **Security Group (SG)** | A virtual firewall controlling traffic to a resource (stateful). |
| **RDS** | Managed relational database service (AWS runs the OS, patching, backups). |
| **Snapshot** | A point-in-time backup of a DB volume. |
| **RPO** | *Recovery Point Objective* — how much data (time) you can afford to lose. |
| **RTO** | *Recovery Time Objective* — how fast you must be back online. |

> **RPO and RTO drive every DR decision.** Low RPO/RTO = more money. Match them to business needs.

---

## 4. Amazon RDS Fundamentals

### 4.1 What is RDS?
Amazon RDS (Relational Database Service) is a **managed** service. AWS handles: provisioning, OS patching, DB engine patching, automated backups, failover, and monitoring. You handle: schema, queries, indexes, and connection management.

### 4.2 Supported engines
- **Amazon Aurora** (MySQL- & PostgreSQL-compatible, AWS-built, fastest/most scalable)
- **MySQL**
- **PostgreSQL**
- **MariaDB**
- **Oracle**
- **Microsoft SQL Server**

### 4.3 RDS vs. Aurora vs. EC2-hosted DB
| Option | You manage | Best for |
|--------|-----------|----------|
| **DB on EC2** | Everything (OS, DB, backups, HA) | Full control, niche needs |
| **RDS** | Schema & tuning only | Most workloads |
| **Aurora** | Schema & tuning only | High scale, low-latency, advanced DR |
| **Aurora Serverless v2** | Schema only, auto-scales capacity | Variable/unpredictable load |

### 4.4 Key building blocks
- **DB Instance** — the database server (compute + storage).
- **Instance class** — size, e.g. `db.t3.micro`, `db.m6g.large` (memory/CPU).
- **Storage type** — `gp3` (general SSD, default), `io1/io2` (provisioned IOPS), `magnetic` (legacy).
- **DB Subnet Group** — the set of subnets RDS can place the DB in (needs ≥2 AZs).
- **Parameter Group** — engine configuration (like `my.cnf`).
- **Option Group** — extra engine features (e.g., Oracle TDE, SQL Server audit).

---

## 5. Networking & Security for RDS

### 5.1 The golden rule
**Never expose a production database to the public internet.** Put it in **private subnets**. Apps connect from inside the VPC.

### 5.2 Create a VPC layout (real-project pattern)
A typical 3-tier setup across 2+ AZs:
- Public subnets → Load Balancer / NAT Gateway
- Private app subnets → application servers (EC2/ECS/Lambda)
- Private data subnets → RDS

**Console:** VPC → **Create VPC** → **VPC and more** → choose 2 AZs, 2 public + 2 private subnets → Create.

**CLI (minimal example):**
```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]'

# (Repeat create-subnet for each AZ; create route tables, IGW, NAT as needed)
```
> For real projects, create the network with **Terraform/CloudFormation** (Section 23) — manual subnet wiring is error-prone.

### 5.3 Security Group for RDS
Allow only the **app tier** to reach the DB port (MySQL 3306, PostgreSQL 5432).

**Console:** EC2 → **Security Groups** → Create → Inbound rule: Type=MySQL/Aurora, Source = the **app security group** (not `0.0.0.0/0`).

**CLI:**
```bash
# Create SG for the database
aws ec2 create-security-group \
  --group-name rds-sg --description "RDS access" --vpc-id vpc-0abc123

# Allow MySQL only from the app security group (sg-app123)
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds456 --protocol tcp --port 3306 \
  --source-group sg-app123
```

### 5.4 IAM & least privilege
- Use **IAM database authentication** (token-based) instead of static passwords where possible.
- Store passwords in **AWS Secrets Manager** (auto-rotation supported). See [Section 12](#12-encryption-secrets--parameteroption-groups).

### 5.5 DB Subnet Group
RDS requires a subnet group spanning **≥2 AZs** (for Multi-AZ failover).

**Console:** RDS → **Subnet groups** → Create → add the private data subnets from each AZ.

**CLI:**
```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name prod-db-subnets \
  --db-subnet-group-description "Private data subnets" \
  --subnet-ids subnet-aaa subnet-bbb
```

---

## 6. Creating Your First RDS Instance

We'll create a **MySQL** instance (swap engine as needed).

### 6.1 Console
1. RDS → **Databases** → **Create database**.
2. **Standard create** → Engine: **MySQL**.
3. Template: **Free tier** (learning) or **Production** (real).
4. Settings:
   - DB instance identifier: `prod-mysql-1`
   - Master username: `admin`
   - Credentials: choose **Managed in AWS Secrets Manager** (recommended) or set a password.
5. Instance class: `db.t3.micro` (free tier) or `db.m6g.large` (prod).
6. Storage: `gp3`, 20 GB, **enable storage autoscaling**.
7. Availability: **Multi-AZ** = Yes (prod) / No (learning).
8. Connectivity: choose your **VPC**, **DB subnet group**, **Public access = No**, attach **rds-sg**.
9. Additional: enable **automated backups** (retention 7–35 days), **encryption**, **deletion protection**.
10. **Create database**. Wait ~5–10 min until status = **Available**.

### 6.2 CLI
```bash
aws rds create-db-instance \
  --db-instance-identifier prod-mysql-1 \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.39 \
  --master-username admin \
  --manage-master-user-password \
  --allocated-storage 20 \
  --storage-type gp3 \
  --max-allocated-storage 100 \
  --db-subnet-group-name prod-db-subnets \
  --vpc-security-group-ids sg-rds456 \
  --no-publicly-accessible \
  --multi-az \
  --backup-retention-period 7 \
  --storage-encrypted \
  --deletion-protection
```
Check status:
```bash
aws rds describe-db-instances \
  --db-instance-identifier prod-mysql-1 \
  --query "DBInstances[0].DBInstanceStatus"
```

---

## 7. Connecting to the Database

### 7.1 Get the endpoint
**Console:** RDS → Databases → `prod-mysql-1` → **Connectivity & security** → copy **Endpoint** and **Port**.

**CLI:**
```bash
aws rds describe-db-instances --db-instance-identifier prod-mysql-1 \
  --query "DBInstances[0].Endpoint.[Address,Port]" --output text
```

### 7.2 Get the password from Secrets Manager
```bash
# Find the secret ARN
aws rds describe-db-instances --db-instance-identifier prod-mysql-1 \
  --query "DBInstances[0].MasterUserSecret.SecretArn" --output text

# Retrieve the password
aws secretsmanager get-secret-value --secret-id <secret-arn> \
  --query "SecretString" --output text
```

### 7.3 Connect (from inside the VPC, e.g., a bastion or app server)
```bash
mysql -h prod-mysql-1.abc123.us-east-1.rds.amazonaws.com -P 3306 -u admin -p
```
> Because the DB is **private**, connect from an EC2 in the same VPC, via **SSM Session Manager**, or a **bastion host**. Avoid making the DB public.

---

## 8. High Availability: Multi-AZ

### 8.1 What it is
Multi-AZ keeps a **synchronous standby** in a second AZ. On failure, RDS **automatically fails over** (60–120s) by switching the DNS endpoint. Your app reconnects to the same endpoint.

- **Multi-AZ instance** (1 standby, no read traffic).
- **Multi-AZ DB cluster** (2 readable standbys, faster failover ~35s) — for MySQL/PostgreSQL.

### 8.2 Why it matters
Protects against **AZ failure** and reduces downtime during maintenance/patching. This is **HA, not DR** — it's within one Region.

### 8.3 Enable on an existing instance
**Console:** Databases → select instance → **Modify** → Availability = **Multi-AZ** → Apply immediately or in maintenance window.

**CLI:**
```bash
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-1 \
  --multi-az --apply-immediately
```

### 8.4 Test failover (safe)
**Console:** Databases → instance → **Actions** → **Reboot** → check **Reboot with failover**.

**CLI:**
```bash
aws rds reboot-db-instance \
  --db-instance-identifier prod-mysql-1 --force-failover
```

---

## 9. Scaling: Read Replicas

### 9.1 What it is
A **read replica** is an **asynchronous** copy that serves **read** traffic. Offloads reporting/analytics from the primary. Can be in the **same Region** or **cross-Region** (cross-Region replicas double as DR — see Section 20).

### 9.2 Create a read replica
**Console:** Databases → primary → **Actions** → **Create read replica** → choose AZ/Region, instance class.

**CLI (same region):**
```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier prod-mysql-rr1 \
  --source-db-instance-identifier prod-mysql-1 \
  --db-instance-class db.t3.micro
```
**CLI (cross-region DR replica):**
```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier dr-mysql-rr \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:prod-mysql-1 \
  --region us-west-2 \
  --kms-key-id <dr-region-kms-key>
```

### 9.3 Promote a replica (failover/DR)
A replica can be **promoted** to a standalone primary (breaks replication, becomes writable).
```bash
aws rds promote-read-replica \
  --db-instance-identifier dr-mysql-rr --region us-west-2
```

> **Read replica ≠ standby.** Replicas are async (possible data lag) and don't auto-failover. Multi-AZ standby is sync and auto-fails over.

---

## 10. Backups: Automated & Manual Snapshots

### 10.1 Automated backups
- Enabled by setting **retention period** (1–35 days; `0` disables — never do this in prod).
- Backs up the whole instance + **transaction logs** (enables PITR).
- Stored in S3, encrypted if the DB is encrypted.
- Taken during the **backup window**; minimal performance impact with Multi-AZ.

**Set retention (CLI):**
```bash
aws rds modify-db-instance \
  --db-instance-identifier prod-mysql-1 \
  --backup-retention-period 14 \
  --preferred-backup-window 03:00-04:00 \
  --apply-immediately
```

### 10.2 Manual snapshots
- Kept **until you delete them** (automated ones expire with retention).
- Take before risky changes (schema migration, version upgrade).

**Console:** Databases → instance → **Actions** → **Take snapshot**.

**CLI:**
```bash
aws rds create-db-snapshot \
  --db-instance-identifier prod-mysql-1 \
  --db-snapshot-identifier prod-mysql-1-before-migration
```
List snapshots:
```bash
aws rds describe-db-snapshots --db-instance-identifier prod-mysql-1 \
  --query "DBSnapshots[].DBSnapshotIdentifier"
```

### 10.3 Restore from a snapshot
Restoring **always creates a NEW instance** (you then update app connection strings).

**CLI:**
```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier prod-mysql-restored \
  --db-snapshot-identifier prod-mysql-1-before-migration \
  --db-subnet-group-name prod-db-subnets \
  --vpc-security-group-ids sg-rds456
```

### 10.4 Copy / share snapshots (cross-region DR)
```bash
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123456789012:snapshot:prod-mysql-1-before-migration \
  --target-db-snapshot-identifier prod-mysql-1-dr-copy \
  --source-region us-east-1 --region us-west-2 \
  --kms-key-id <dr-region-kms-key>
```

---

## 11. Point-in-Time Recovery (PITR)

### 11.1 What it is
Restore the database to **any second** within the retention window (uses automated backups + transaction logs). Perfect for "someone ran `DELETE` without a `WHERE` 10 minutes ago."

### 11.2 Find the latest restorable time
```bash
aws rds describe-db-instances --db-instance-identifier prod-mysql-1 \
  --query "DBInstances[0].LatestRestorableTime"
```

### 11.3 Restore to a point in time
**Console:** Databases → instance → **Actions** → **Restore to point in time** → pick custom time → set new identifier.

**CLI:**
```bash
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier prod-mysql-1 \
  --target-db-instance-identifier prod-mysql-pitr \
  --restore-time 2026-06-25T08:45:00Z \
  --db-subnet-group-name prod-db-subnets \
  --vpc-security-group-ids sg-rds456
```
> After restore, validate data, then **repoint the app** (update endpoint / Route 53 record / Secrets Manager).

---

## 12. Encryption, Secrets & Parameter/Option Groups

### 12.1 Encryption at rest (KMS)
- Enable at **creation** (`--storage-encrypted`). You **cannot** encrypt an existing unencrypted instance directly — instead: snapshot → **copy snapshot with encryption** → restore.
```bash
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier prod-mysql-1-snap \
  --target-db-snapshot-identifier prod-mysql-1-snap-enc \
  --kms-key-id <kms-key-id>
```

### 12.2 Encryption in transit (SSL/TLS)
Download the RDS CA bundle and require SSL:
```bash
mysql -h <endpoint> -u admin -p \
  --ssl-ca=global-bundle.pem --ssl-mode=REQUIRED
```

### 12.3 Secrets Manager with rotation
**CLI (create + attach rotation):**
```bash
aws secretsmanager create-secret --name prod/mysql/admin \
  --secret-string '{"username":"admin","password":"<pw>"}'

aws secretsmanager rotate-secret --secret-id prod/mysql/admin \
  --rotation-lambda-arn <rotation-lambda-arn> \
  --rotation-rules AutomaticallyAfterDays=30
```

### 12.4 Parameter Groups (engine config)
```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name prod-mysql8 \
  --db-parameter-group-family mysql8.0 \
  --description "Prod MySQL 8 params"

aws rds modify-db-parameter-group \
  --db-parameter-group-name prod-mysql8 \
  --parameters "ParameterName=max_connections,ParameterValue=500,ApplyMethod=pending-reboot"

aws rds modify-db-instance --db-instance-identifier prod-mysql-1 \
  --db-parameter-group-name prod-mysql8 --apply-immediately
```

### 12.5 Option Groups (extra features)
Used for engine-specific add-ons (e.g., SQL Server `SQLSERVER_AUDIT`, Oracle `OEM`, MySQL `MEMCACHED`). Create and attach similarly via `create-option-group` / `modify-db-instance --option-group-name`.

---

## 13. Monitoring & Alerting

### 13.1 What to watch
| Metric | Why |
|--------|-----|
| `CPUUtilization` | Overload detection |
| `FreeableMemory` | Memory pressure |
| `FreeStorageSpace` | Disk full = outage |
| `DatabaseConnections` | Connection exhaustion |
| `ReadLatency` / `WriteLatency` | Performance |
| `ReplicaLag` | DR/replica health |

### 13.2 CloudWatch alarm (CLI)
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name rds-low-storage \
  --namespace AWS/RDS \
  --metric-name FreeStorageSpace \
  --dimensions Name=DBInstanceIdentifier,Value=prod-mysql-1 \
  --statistic Average --period 300 \
  --threshold 5000000000 --comparison-operator LessThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions <sns-topic-arn>
```

### 13.3 Enhanced Monitoring & Performance Insights
- **Enhanced Monitoring**: OS-level metrics (1–60s granularity).
- **Performance Insights**: visual query/load analysis — invaluable for tuning.

**Enable (CLI):**
```bash
aws rds modify-db-instance --db-instance-identifier prod-mysql-1 \
  --monitoring-interval 60 \
  --monitoring-role-arn <enhanced-monitoring-role-arn> \
  --enable-performance-insights \
  --apply-immediately
```

### 13.4 Logs
Publish DB logs to CloudWatch Logs:
```bash
aws rds modify-db-instance --db-instance-identifier prod-mysql-1 \
  --cloudwatch-logs-export-configuration '{"EnableLogTypes":["error","slowquery","general"]}' \
  --apply-immediately
```

---

## 14. Maintenance, Patching & Cost Control

### 14.1 Maintenance window
AWS applies patches during your chosen window. Multi-AZ patches the standby first to minimize downtime.
```bash
aws rds modify-db-instance --db-instance-identifier prod-mysql-1 \
  --preferred-maintenance-window "sun:05:00-sun:06:00" --apply-immediately
```

### 14.2 Engine version upgrades
- **Minor**: low risk; can auto-apply.
- **Major**: test first! Snapshot, restore to a test instance, run app tests, then upgrade prod.
```bash
aws rds modify-db-instance --db-instance-identifier prod-mysql-1 \
  --engine-version 8.0.40 --apply-immediately
```

### 14.3 Cost control
- Use **Reserved Instances** / **Savings Plans** for steady workloads (up to ~60% off).
- **Stop** non-prod instances when idle (max 7 days, then auto-starts):
```bash
aws rds stop-db-instance --db-instance-identifier dev-mysql-1
aws rds start-db-instance --db-instance-identifier dev-mysql-1
```
- Right-size with Performance Insights; delete old manual snapshots; use `gp3` over `io1` where possible.
- Use **Aurora Serverless v2** for spiky/low-usage workloads.

---

## 15. Disaster Recovery (DR) — Concepts

### 15.1 HA vs. DR
- **High Availability (HA):** survive component/AZ failure **within a Region** (e.g., Multi-AZ). Automatic, seconds.
- **Disaster Recovery (DR):** survive **Region-wide** failure or major incident. Often cross-Region, may be manual.

### 15.2 RPO & RTO drive the strategy
```
Cheaper / slower  <------------------------------------>  Costlier / faster
Backup & Restore   →   Pilot Light   →   Warm Standby   →   Multi-Site Active/Active
RTO: hours              RTO: ~10s–min     RTO: minutes        RTO: ~0
RPO: hours              RPO: minutes      RPO: seconds        RPO: ~0
```

### 15.3 The four AWS DR strategies (official)
| Strategy | What runs in DR region | RTO | RPO | Cost |
|----------|------------------------|-----|-----|------|
| **Backup & Restore** | Nothing (just backups) | Hours | Hours | $ |
| **Pilot Light** | Core data replicated, servers off | 10s of min | Minutes | $$ |
| **Warm Standby** | Scaled-down full stack running | Minutes | Seconds | $$$ |
| **Multi-Site Active/Active** | Full stack, serving traffic | Near zero | Near zero | $$$$ |

### 15.4 How to choose (real-project guidance)
- Internal/dev tools → **Backup & Restore**.
- Standard business apps → **Pilot Light** or **Warm Standby**.
- Mission-critical (banking, e-commerce checkout) → **Warm Standby** or **Active/Active**.

---

## 16. DR Strategy 1: Backup & Restore

### 16.1 Idea
Keep backups/snapshots **copied to a second Region**. On disaster, **restore** there. Cheapest, slowest.

### 16.2 Implementation
1. Automated backups + manual snapshots in primary Region.
2. **Copy snapshots cross-Region** (manual, scheduled via Lambda/EventBridge, or **AWS Backup**).
3. On disaster: restore snapshot in DR Region → update DNS/app config.

**CLI — automate cross-region snapshot copy (run on a schedule):**
```bash
LATEST=$(aws rds describe-db-snapshots --db-instance-identifier prod-mysql-1 \
  --snapshot-type automated --query "reverse(sort_by(DBSnapshots,&SnapshotCreateTime))[0].DBSnapshotIdentifier" --output text)

aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123456789012:snapshot:$LATEST \
  --target-db-snapshot-identifier dr-copy-$LATEST \
  --source-region us-east-1 --region us-west-2 \
  --kms-key-id <dr-kms-key>
```

### 16.3 Recovery
```bash
aws rds restore-db-instance-from-db-snapshot --region us-west-2 \
  --db-instance-identifier prod-mysql-dr \
  --db-snapshot-identifier dr-copy-<snap> \
  --db-subnet-group-name dr-db-subnets \
  --vpc-security-group-ids sg-dr-rds
```

---

## 17. DR Strategy 2: Pilot Light

### 17.1 Idea
Keep the **data layer always replicated** and **alive** in the DR Region (e.g., a **cross-Region read replica**), but keep app/compute **switched off**. On disaster: **promote the replica**, then spin up compute.

### 17.2 Implementation
1. Create a **cross-Region read replica** of RDS (Section 9.2).
2. Pre-create (but keep stopped/minimal) compute via IaC (Auto Scaling at 0, Lambda, or stopped EC2/ECS task defs).
3. Keep AMIs, container images, and IaC templates ready in DR Region.

### 17.3 Failover runbook
```bash
# 1. Promote the DR replica to a writable primary
aws rds promote-read-replica --region us-west-2 \
  --db-instance-identifier dr-mysql-rr

# 2. Scale up compute (example: ASG desired count)
aws autoscaling set-desired-capacity --region us-west-2 \
  --auto-scaling-group-name dr-app-asg --desired-capacity 4

# 3. Update Route 53 to point app DNS to the DR region (see Section 20.4)
```
- **RTO:** 10s of minutes. **RPO:** seconds–minutes (async replica lag).

---

## 18. DR Strategy 3: Warm Standby

### 18.1 Idea
A **fully functional but scaled-down** copy of the stack runs in DR Region at all times. On disaster, **scale it up** to full capacity. Faster than pilot light.

### 18.2 Implementation
- Cross-Region read replica (or Aurora Global DB) **always running**.
- App tier running at **minimal capacity** (e.g., 1 small instance/ASG) and actually serving health checks.
- Load balancer + DNS already configured.

### 18.3 Failover
```bash
# Promote DB (or Aurora global failover), then scale compute up
aws autoscaling set-desired-capacity --region us-west-2 \
  --auto-scaling-group-name dr-app-asg --desired-capacity 10
# Shift traffic via Route 53 weighted/failover routing
```
- **RTO:** minutes. **RPO:** seconds.

---

## 19. DR Strategy 4: Multi-Site Active/Active

### 19.1 Idea
**Both Regions serve live traffic** simultaneously. If one fails, the other absorbs all traffic. Lowest RTO/RPO, highest cost and complexity.

### 19.2 Implementation
- Database: **Aurora Global Database** (with write-forwarding) or app-level multi-region writes (careful with conflicts), or **DynamoDB Global Tables** for NoSQL.
- Route 53 **latency-based** or **geolocation** routing across both Regions.
- Data consistency strategy is the hard part — design for it.

### 19.3 Failover
Mostly **automatic** via Route 53 health checks; for Aurora, promote the secondary Region (typically <1 min RTO).
- **RTO/RPO:** near zero.

---

## 20. Cross-Region DR for RDS (Hands-On)

A complete, practical cross-Region DR setup (Pilot Light / Warm Standby pattern).

### 20.1 Prepare the DR Region
- Create VPC, subnets, subnet group, security groups (mirror primary). Use IaC.
- Create a **KMS key** in DR Region (encrypted cross-Region copies need a DR-region key).

### 20.2 Replicate the database
**Option A — Cross-Region read replica (continuous):**
```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier dr-mysql-rr \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:prod-mysql-1 \
  --region us-west-2 \
  --db-subnet-group-name dr-db-subnets \
  --vpc-security-group-ids sg-dr-rds \
  --kms-key-id <dr-kms-key>
```
**Option B — Scheduled cross-Region snapshot copy (cheaper):** see Section 16.2.

### 20.3 Monitor replica lag
```bash
aws cloudwatch get-metric-statistics --region us-west-2 \
  --namespace AWS/RDS --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=dr-mysql-rr \
  --start-time 2026-06-25T00:00:00Z --end-time 2026-06-25T01:00:00Z \
  --period 300 --statistics Average
```

### 20.4 DNS failover with Route 53
Use a **CNAME** (e.g., `db.myapp.com`) that apps use instead of the raw RDS endpoint, with **failover routing** + health checks.
```bash
# Simplified: update the record to point to the DR endpoint on failover
aws route53 change-resource-record-sets --hosted-zone-id <zone-id> \
  --change-batch '{
    "Changes":[{"Action":"UPSERT","ResourceRecordSet":{
      "Name":"db.myapp.com","Type":"CNAME","TTL":60,
      "ResourceRecords":[{"Value":"dr-mysql-rr.xyz.us-west-2.rds.amazonaws.com"}]
    }}]}'
```
> Apps should connect to `db.myapp.com`, **not** the raw endpoint — so failover = one DNS change.

### 20.5 Failover procedure (summary)
1. Confirm primary is truly down (avoid split-brain).
2. Promote DR replica → writable.
3. Scale up DR compute.
4. Update Route 53 → DR endpoint.
5. Validate app + data.
6. After recovery, rebuild primary and **fail back** during a maintenance window.

---

## 21. Aurora Global Database (Advanced DR)

### 21.1 What it is
Aurora Global Database links one **primary** Region with up to **5 secondary** Regions. Replication uses dedicated infrastructure with **typical lag < 1 second** and **RPO ~1s, RTO < 1 min**. Best-in-class managed DR for relational data.

### 21.2 Create (CLI)
```bash
# 1. Create the global cluster
aws rds create-global-cluster \
  --global-cluster-identifier myapp-global \
  --engine aurora-mysql --engine-version 8.0.mysql_aurora.3.05.2

# 2. Create the primary cluster in the global cluster
aws rds create-db-cluster \
  --db-cluster-identifier myapp-primary \
  --engine aurora-mysql --region us-east-1 \
  --global-cluster-identifier myapp-global \
  --master-username admin --manage-master-user-password \
  --db-subnet-group-name prod-db-subnets

aws rds create-db-instance \
  --db-instance-identifier myapp-primary-1 \
  --db-cluster-identifier myapp-primary \
  --engine aurora-mysql --db-instance-class db.r6g.large --region us-east-1

# 3. Add a secondary (DR) cluster in another region
aws rds create-db-cluster \
  --db-cluster-identifier myapp-secondary \
  --engine aurora-mysql --region us-west-2 \
  --global-cluster-identifier myapp-global \
  --db-subnet-group-name dr-db-subnets

aws rds create-db-instance \
  --db-instance-identifier myapp-secondary-1 \
  --db-cluster-identifier myapp-secondary \
  --engine aurora-mysql --db-instance-class db.r6g.large --region us-west-2
```

### 21.3 Managed failover (planned)
```bash
aws rds failover-global-cluster \
  --global-cluster-identifier myapp-global \
  --target-db-cluster-identifier arn:aws:rds:us-west-2:123456789012:cluster:myapp-secondary
```

### 21.4 Unplanned failover (detach & promote)
If the primary Region is unreachable, detach the secondary and promote it:
```bash
aws rds remove-from-global-cluster --region us-west-2 \
  --global-cluster-identifier myapp-global \
  --db-cluster-identifier arn:aws:rds:us-west-2:123456789012:cluster:myapp-secondary
```

---

## 22. AWS Backup — Centralized DR

### 22.1 What it is
**AWS Backup** centrally manages backups across RDS, EBS, DynamoDB, EFS, EC2, etc., with **policies (backup plans)**, **cross-Region/cross-account copy**, and **compliance reporting**. Preferred in real projects over hand-rolled snapshot scripts.

### 22.2 Set up (Console)
1. AWS Backup → **Backup plans** → **Create** → choose a template or build custom.
2. Define **backup rules**: frequency, retention, **cross-Region copy** to DR Region.
3. **Assign resources** by tag (e.g., `Backup=true`) or by ARN.

### 22.3 Set up (CLI)
```bash
# Create a vault in the DR region
aws backup create-backup-vault --backup-vault-name dr-vault --region us-west-2

# Create a backup plan (JSON file plan.json defines rules + cross-region copy)
aws backup create-backup-plan --backup-plan file://plan.json

# Assign resources by tag
aws backup create-backup-selection \
  --backup-plan-id <plan-id> \
  --backup-selection '{
    "SelectionName":"rds-tagged",
    "IamRoleArn":"arn:aws:iam::123456789012:role/AWSBackupDefaultServiceRole",
    "ListOfTags":[{"ConditionType":"STRINGEQUALS","ConditionKey":"Backup","ConditionValue":"true"}]
  }'
```
Tag your RDS instance so it's picked up:
```bash
aws rds add-tags-to-resource \
  --resource-name arn:aws:rds:us-east-1:123456789012:db:prod-mysql-1 \
  --tags Key=Backup,Value=true
```

---

## 23. DevOps: Infrastructure as Code (IaC)

### 23.1 Why IaC
Manual console clicks aren't repeatable or reviewable. IaC makes infra **versioned, peer-reviewed, and reproducible** across Regions — essential for DR (you can rebuild the DR Region from code).

### 23.2 Terraform example — RDS instance
```hcl
# main.tf
provider "aws" {
  region = "us-east-1"
}

resource "aws_db_subnet_group" "prod" {
  name       = "prod-db-subnets"
  subnet_ids = var.private_subnet_ids
}

resource "aws_security_group" "rds" {
  name   = "rds-sg"
  vpc_id = var.vpc_id
  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [var.app_sg_id]   # only app tier
  }
  egress {
    from_port = 0; to_port = 0; protocol = "-1"; cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_db_instance" "prod" {
  identifier                  = "prod-mysql-1"
  engine                      = "mysql"
  engine_version              = "8.0.39"
  instance_class              = "db.m6g.large"
  allocated_storage           = 20
  max_allocated_storage       = 100
  storage_type                = "gp3"
  username                    = "admin"
  manage_master_user_password = true
  db_subnet_group_name        = aws_db_subnet_group.prod.name
  vpc_security_group_ids      = [aws_security_group.rds.id]
  multi_az                    = true
  storage_encrypted           = true
  backup_retention_period     = 14
  deletion_protection         = true
  skip_final_snapshot         = false
  final_snapshot_identifier   = "prod-mysql-1-final"
}
```
**Run it:**
```bash
terraform init
terraform plan
terraform apply
```

### 23.3 CloudFormation example (snippet)
```yaml
Resources:
  ProdDB:
    Type: AWS::RDS::DBInstance
    Properties:
      DBInstanceIdentifier: prod-mysql-1
      Engine: mysql
      EngineVersion: "8.0.39"
      DBInstanceClass: db.m6g.large
      AllocatedStorage: "20"
      MultiAZ: true
      StorageEncrypted: true
      BackupRetentionPeriod: 14
      DeletionProtection: true
      DBSubnetGroupName: !Ref ProdSubnetGroup
      VPCSecurityGroups: [!Ref RdsSG]
      ManageMasterUserPassword: true
      MasterUsername: admin
```
```bash
aws cloudformation deploy --template-file rds.yaml --stack-name prod-rds \
  --capabilities CAPABILITY_NAMED_IAM
```

### 23.4 Best practices
- Keep IaC in **Git**; require **pull-request reviews**.
- Separate **state per environment** (dev/stage/prod) and per Region.
- Store secrets in **Secrets Manager**, never in code.
- Parameterize Region so the **same code builds the DR Region**.

---

## 24. DevOps: CI/CD Pipeline for DB Changes

### 24.1 The problem
App code is deployed via CI/CD — but **database schema changes** (migrations) need the same discipline: versioned, tested, repeatable, reversible.

### 24.2 Schema migration tools
- **Flyway** or **Liquibase** (SQL/Java), **Alembic** (Python), **Knex/Prisma** (Node), **EF Core** (.NET).
- Migrations are **versioned files** checked into Git and applied in order.

### 24.3 Example pipeline (GitHub Actions / CodePipeline)
```yaml
# .github/workflows/db-migrate.yml
name: DB Migrate
on:
  push:
    branches: [ main ]
    paths: [ "db/migrations/**" ]
jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/ci-db-migrate
          aws-region: us-east-1
      - name: Get DB password from Secrets Manager
        run: echo "DB_PASS=$(aws secretsmanager get-secret-value --secret-id prod/mysql/admin --query SecretString --output text)" >> $GITHUB_ENV
      - name: Run Flyway migrations
        run: |
          flyway -url=jdbc:mysql://db.myapp.com:3306/appdb \
                 -user=admin -password="$DB_PASS" \
                 -locations=filesystem:db/migrations migrate
```

### 24.4 Safe migration practices (real projects)
- **Take a snapshot before migrating** (automate it as a pipeline step).
- Make migrations **backward-compatible** (expand-then-contract): add columns first, deploy app, then remove old columns later.
- Run on a **restored copy** first (test environment from prod snapshot).
- Use **blue/green deployments** (RDS Blue/Green) for risky schema/version changes:
```bash
aws rds create-blue-green-deployment \
  --blue-green-deployment-name prod-bg \
  --source arn:aws:rds:us-east-1:123456789012:db:prod-mysql-1 \
  --target-engine-version 8.0.40
# Test green, then switch over:
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier <id> --switchover-timeout 300
```

---

## 25. DevOps: Automation & Runbooks

### 25.1 Scheduled tasks with EventBridge + Lambda
Automate snapshot copies, replica-lag checks, nightly stop/start of dev DBs.

**Example: EventBridge rule → Lambda that copies snapshots cross-Region nightly.**
```bash
aws events put-rule --name nightly-snapshot-copy \
  --schedule-expression "cron(0 4 * * ? *)"

aws events put-targets --rule nightly-snapshot-copy \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:CopySnapshotToDR"
```

### 25.2 Systems Manager Automation runbooks
Encode your **failover runbook** as an SSM Automation document so recovery is one click/command (promote replica → scale ASG → update Route 53). This reduces human error and RTO.

### 25.3 Incident response basics
- Alarms (Section 13) → **SNS** → email/Slack/PagerDuty.
- Maintain a written, tested **DR runbook** (who does what, in order).
- Use **AWS Health Dashboard** to detect Region-level events.

---

## 26. DR Testing & Game Days

### 26.1 Why test
**An untested DR plan is not a plan.** Backups can be corrupt; runbooks go stale; people forget steps.

### 26.2 What to test (quarterly recommended)
1. **Restore drill** — restore the latest snapshot in DR Region; verify data integrity & row counts.
2. **Failover drill** — promote replica, point a test app at it, run smoke tests; measure actual RTO/RPO.
3. **Multi-AZ failover** — reboot-with-failover; confirm app reconnects.
4. **Fail-back** — practice returning to the primary Region.

### 26.3 Measure & improve
- Record **actual** RTO/RPO vs. targets.
- File action items for gaps.
- Update runbooks and IaC after every drill.

### 26.4 Chaos / fault injection
Use **AWS Fault Injection Simulator (FIS)** to inject failures (e.g., reboot DB, fail an AZ) in a controlled way and validate resilience.

---

## 27. RDS Proxy & Connection Management

### 27.1 The problem it solves
Apps (especially **serverless/Lambda** and high-traffic services) open many short-lived DB connections. Databases have a **connection limit**; too many connections cause errors and exhaust memory. During a **failover**, raw connections break and apps may take a long time to reconnect.

### 27.2 What RDS Proxy is
A **fully managed connection pooler** that sits between your app and RDS/Aurora. Benefits:
- **Pools and reuses** connections → handles connection spikes.
- **Faster failover** (up to ~66% faster) — the proxy holds the app connection and reconnects to the new primary transparently.
- **IAM authentication** and **Secrets Manager** integration (no DB passwords in app).
- Improves security: the app talks to the proxy, not the DB directly.

### 27.3 When to use it (real projects)
- Lambda / serverless apps.
- Microservices with bursty traffic.
- Anything where connection storms or failover speed matter.

### 27.4 Create an RDS Proxy
**Console:** RDS → **Proxies** → **Create proxy** → select engine-compatible DB → choose the **Secrets Manager** secret → assign IAM role → select VPC subnets + security group → Create.

**CLI:**
```bash
aws rds create-db-proxy \
  --db-proxy-name prod-mysql-proxy \
  --engine-family MYSQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"<secret-arn>","IAMAuth":"REQUIRED"}]' \
  --role-arn <proxy-iam-role-arn> \
  --vpc-subnet-ids subnet-aaa subnet-bbb \
  --vpc-security-group-ids sg-rds456 \
  --require-tls

# Register the target database with the proxy
aws rds register-db-proxy-targets \
  --db-proxy-name prod-mysql-proxy \
  --db-instance-identifiers prod-mysql-1
```
Get the proxy endpoint and point your app at it:
```bash
aws rds describe-db-proxies --db-proxy-name prod-mysql-proxy \
  --query "DBProxies[0].Endpoint"
```

### 27.5 Connection best practices (without/with proxy)
- Use **connection pooling** in the app (HikariCP, pgbouncer, SQLAlchemy pool) even with RDS Proxy.
- Set sensible **pool sizes**; don't open a connection per request.
- Always connect via a **stable DNS name** (proxy endpoint or Route 53 CNAME), never a hard-coded raw endpoint.

---

## 28. IAM Database Authentication

### 28.1 What it is
Instead of a static DB password, the app requests a **short-lived (15-min) auth token** from AWS IAM and uses it to log in. No passwords stored in app config. Great for **least privilege** and audit.

> Supported on MySQL, PostgreSQL, MariaDB, and Aurora. Has a connection-rate limit, so it's ideal for moderate connection rates or behind **RDS Proxy**.

### 28.2 Enable IAM auth on the instance
```bash
aws rds modify-db-instance --db-instance-identifier prod-mysql-1 \
  --enable-iam-database-authentication --apply-immediately
```

### 28.3 Create a DB user mapped to IAM (MySQL example)
```sql
CREATE USER 'appuser' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
GRANT SELECT, INSERT, UPDATE, DELETE ON appdb.* TO 'appuser';
```

### 28.4 Attach an IAM policy to the app role
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "rds-db:connect",
    "Resource": "arn:aws:rds-db:us-east-1:123456789012:dbuser:db-ABCDEFGHIJKL/appuser"
  }]
}
```

### 28.5 Connect using a token (CLI + mysql)
```bash
TOKEN=$(aws rds generate-db-auth-token \
  --hostname prod-mysql-1.abc123.us-east-1.rds.amazonaws.com \
  --port 3306 --username appuser --region us-east-1)

mysql -h prod-mysql-1.abc123.us-east-1.rds.amazonaws.com -P 3306 \
  -u appuser --password="$TOKEN" \
  --ssl-ca=global-bundle.pem --enable-cleartext-plugin
```

---

## 29. AWS DMS — Database Migration Service

### 29.1 What it is
**AWS Database Migration Service (DMS)** moves data **into** RDS/Aurora from on-premises databases, EC2 databases, or other clouds — with **minimal downtime**. It can also do **continuous replication** (CDC = Change Data Capture) and **heterogeneous** migrations (e.g., Oracle → PostgreSQL, paired with **AWS SCT** for schema conversion).

### 29.2 When you'll use it (real projects)
- Lift-and-shift an existing on-prem DB into RDS.
- Migrate between engines (Oracle → Aurora PostgreSQL to cut licensing cost).
- Keep a target continuously in sync during a phased cutover.
- Ongoing replication into a data lake / analytics target.

### 29.3 Core components
- **Replication instance** — the compute that runs the migration.
- **Source & target endpoints** — connection details for each DB.
- **Migration task** — full-load, CDC, or full-load + CDC.
- **AWS SCT** — converts schema/code for cross-engine moves.

### 29.4 High-level steps (CLI)
```bash
# 1. Create a replication instance
aws dms create-replication-instance \
  --replication-instance-identifier prod-dms \
  --replication-instance-class dms.t3.medium \
  --allocated-storage 50 \
  --replication-subnet-group-identifier dms-subnet-group \
  --no-publicly-accessible

# 2. Create source endpoint (e.g., on-prem MySQL)
aws dms create-endpoint \
  --endpoint-identifier src-onprem-mysql \
  --endpoint-type source --engine-name mysql \
  --server-name 10.1.2.3 --port 3306 \
  --username admin --password '<pw>'

# 3. Create target endpoint (RDS)
aws dms create-endpoint \
  --endpoint-identifier tgt-rds-mysql \
  --endpoint-type target --engine-name mysql \
  --server-name prod-mysql-1.abc123.us-east-1.rds.amazonaws.com \
  --port 3306 --username admin --password '<pw>'

# 4. Create + start a migration task (full load + ongoing CDC)
aws dms create-replication-task \
  --replication-task-identifier onprem-to-rds \
  --source-endpoint-arn <src-arn> \
  --target-endpoint-arn <tgt-arn> \
  --replication-instance-arn <ri-arn> \
  --migration-type full-load-and-cdc \
  --table-mappings file://table-mappings.json

aws dms start-replication-task \
  --replication-task-arn <task-arn> \
  --start-replication-task-type start-replication
```

### 29.5 Cutover checklist
1. Full load completes, CDC keeps target in sync.
2. Validate row counts and run **DMS data validation**.
3. Freeze writes on source (brief).
4. Let CDC drain, then switch the app to RDS.
5. Stop the task, keep source as rollback for a while.

---

## 30. Aurora Deep-Dive: Cloning, Backtrack, Serverless v2

### 30.1 Why Aurora differs
Aurora separates **compute** from a distributed **storage layer** that auto-replicates **6 copies across 3 AZs**. This unlocks features standard RDS doesn't have.

### 30.2 Fast database cloning (copy-on-write)
Create a near-instant, space-efficient clone of a cluster for testing/migrations — only changed pages consume new storage.
```bash
aws rds restore-db-cluster-to-point-in-time \
  --db-cluster-identifier appdb-clone \
  --source-db-cluster-identifier appdb-primary \
  --restore-type copy-on-write --use-latest-restorable-time
```
> Real use: clone prod to test a risky migration on **real data** without affecting prod and without a full copy's cost/time.

### 30.3 Backtrack (Aurora MySQL)
"Rewind" the cluster to a previous point in time **in seconds**, without restoring a new instance — great for undoing a bad batch job.
```bash
# Enable backtrack at cluster create with --backtrack-window 86400 (seconds)
aws rds backtrack-db-cluster \
  --db-cluster-identifier appdb-primary \
  --backtrack-to 2026-06-25T08:30:00Z
```

### 30.4 Aurora Serverless v2
Auto-scales capacity (in **ACUs**) up/down with load, in fine-grained steps — pay for what you use. Ideal for variable, spiky, or unpredictable workloads.
```bash
aws rds create-db-cluster \
  --db-cluster-identifier appdb-serverless \
  --engine aurora-postgresql \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
  --master-username admin --manage-master-user-password \
  --db-subnet-group-name prod-db-subnets

aws rds create-db-instance \
  --db-instance-identifier appdb-serverless-1 \
  --db-cluster-identifier appdb-serverless \
  --engine aurora-postgresql \
  --db-instance-class db.serverless
```

### 30.5 Aurora reader endpoints & auto-scaling replicas
- **Cluster (writer) endpoint** for writes; **reader endpoint** load-balances across replicas.
- Add **Aurora Auto Scaling** to add/remove read replicas based on load.

---

## 31. Cross-Region Network Connectivity

> DR isn't just data — the **DR Region's network** must be reachable and the app tier must be able to talk to the database. Plan this up front.

### 31.1 Options to connect Regions/VPCs
| Option | Use case |
|--------|----------|
| **VPC Peering** | Simple 1:1 VPC connectivity (incl. inter-Region peering) |
| **Transit Gateway** | Hub-and-spoke connecting many VPCs/Regions/on-prem |
| **Site-to-Site VPN** | Encrypted tunnel to on-prem data center |
| **Direct Connect** | Dedicated private link to on-prem (low latency, high throughput) |

### 31.2 Inter-Region VPC peering (CLI)
```bash
aws ec2 create-vpc-peering-connection \
  --vpc-id vpc-primary --peer-vpc-id vpc-dr \
  --peer-region us-west-2

# Accept in the DR region, then add routes on both sides
aws ec2 accept-vpc-peering-connection --region us-west-2 \
  --vpc-peering-connection-id pcx-123
```

### 31.3 Design rules
- Use **non-overlapping CIDR ranges** across Regions (overlap blocks peering/TGW).
- Mirror **subnet groups and security groups** in the DR Region.
- Keep **Route 53 private hosted zones** consistent so internal names resolve in both Regions.

---

## 32. Events, Notifications, Tagging & Governance

### 32.1 RDS Event Subscriptions
Get notified about failovers, low storage, backups, deletions, parameter changes, etc., via **SNS**.

**CLI:**
```bash
aws rds create-event-subscription \
  --subscription-name rds-critical-events \
  --sns-topic-arn <sns-topic-arn> \
  --source-type db-instance \
  --event-categories '["failover","failure","low storage","maintenance"]' \
  --source-ids prod-mysql-1
```

### 32.2 Tagging strategy (do this from day one)
Consistent tags drive **cost allocation**, **automation**, **backups (AWS Backup selection)**, and **governance**.

Recommended tags: `Environment` (prod/stage/dev), `Owner`, `Application`, `CostCenter`, `Backup` (true/false), `DataClassification`.
```bash
aws rds add-tags-to-resource \
  --resource-name arn:aws:rds:us-east-1:123456789012:db:prod-mysql-1 \
  --tags Key=Environment,Value=prod Key=Owner,Value=team-payments \
         Key=Backup,Value=true Key=CostCenter,Value=CC1234
```

### 32.3 Governance & guardrails
- **AWS Config** rules: enforce "RDS must be encrypted", "RDS not publicly accessible", "backups enabled", "Multi-AZ for prod".
- **Service Control Policies (SCPs)** in AWS Organizations to block non-compliant actions.
- **IAM permission boundaries** to limit what teams can do.
- **Cost allocation tags** + **Budgets** + **billing alarms** for spend visibility.

```bash
# Example: ensure a billing alarm exists (via CloudWatch in us-east-1)
aws cloudwatch put-metric-alarm --alarm-name monthly-budget \
  --namespace AWS/Billing --metric-name EstimatedCharges \
  --dimensions Name=Currency,Value=USD \
  --statistic Maximum --period 21600 --threshold 500 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 1 \
  --alarm-actions <sns-topic-arn>
```

---

## 33. Compliance & Well-Architected Reliability

### 33.1 Compliance building blocks
- **Encryption at rest** (KMS) + **in transit** (TLS) — often mandatory (PCI-DSS, HIPAA, GDPR).
- **Audit logging**: enable engine audit logs, export to **CloudWatch Logs**, and centralize with **CloudTrail** (API activity).
- **Secrets rotation** via Secrets Manager.
- **Backup retention** + **immutable backups** (AWS Backup Vault Lock) for ransomware/compliance.
- Map controls with **AWS Artifact** (compliance reports) and **AWS Audit Manager**.

### 33.2 Well-Architected — Reliability pillar (DB lens)
- **Multi-AZ** for HA; **cross-Region** for DR.
- **Automated backups + tested restores** (untested = unreliable).
- **Defined RTO/RPO** with strategy matched to them.
- **Automated failover runbooks** (SSM) to remove human error.
- **Monitoring + alarms** on the leading indicators (storage, connections, replica lag).
- **Capacity planning**: storage autoscaling, right-sized instance classes, Serverless v2 for variable load.

### 33.3 Security pillar quick wins
- Private subnets only; no public DB access.
- Least-privilege SGs (source = app SG, not CIDR).
- IAM auth / Secrets Manager (no static passwords in code).
- Deletion protection + final snapshots on prod.

---

## 34. Troubleshooting Common Issues

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| **Can't connect** (timeout) | SG doesn't allow your source; DB in private subnet | Add inbound rule for app SG; connect from inside VPC / bastion / SSM |
| **Can't connect** (auth failed) | Wrong password / IAM token expired | Re-fetch from Secrets Manager; tokens expire in 15 min |
| **Too many connections** | Connection leak / no pooling | Add RDS Proxy / app pool; raise `max_connections` (param group) |
| **Storage full** | Growth without autoscaling | Enable storage autoscaling; clean up; alarm on `FreeStorageSpace` |
| **High CPU / slow queries** | Missing indexes, bad queries | Use **Performance Insights** + slow query log; add indexes |
| **High replica lag** | Heavy writes / undersized replica | Bigger replica class; reduce write bursts; check network |
| **Failover took long / app didn't reconnect** | App caches DNS / no retry logic | Lower TTL, use proxy, add reconnect/retry in driver |
| **Can't delete instance** | Deletion protection on | `modify-db-instance --no-deletion-protection` first |
| **Restore created new endpoint, app broke** | App used raw endpoint | Use stable Route 53 CNAME; update Secrets Manager |
| **Encryption can't be enabled in place** | Existing unencrypted DB | Snapshot → copy with KMS → restore |

### 34.1 Useful diagnostics
```bash
# Recent events for an instance
aws rds describe-events --source-identifier prod-mysql-1 \
  --source-type db-instance --duration 1440

# Pending modifications / status
aws rds describe-db-instances --db-instance-identifier prod-mysql-1 \
  --query "DBInstances[0].[DBInstanceStatus,PendingModifiedValues]"

# Check parameter values actually applied
aws rds describe-db-parameters --db-parameter-group-name prod-mysql8 \
  --query "Parameters[?ParameterName=='max_connections']"
```

---

## 35. Real-Time Project Blueprint

A reference architecture tying everything together for a typical production app needing **RTO ≈ 15 min, RPO ≈ 1 min**.

### 27.1 Architecture
```
Region A (Primary: us-east-1)            Region B (DR: us-west-2)
┌───────────────────────────┐           ┌───────────────────────────┐
│ Route 53 (failover + HC)  │◄─────────►│ (same DNS, failover target)│
│ ALB → App (ASG, Multi-AZ) │           │ ALB → App (ASG min size)   │
│ RDS MySQL Multi-AZ        │──async──► │ Cross-Region Read Replica  │
│ Secrets Manager           │           │ Secrets (replicated)       │
│ Automated backups (14d)   │──copy───► │ AWS Backup vault (cross-rgn)│
└───────────────────────────┘           └───────────────────────────┘
        IaC (Terraform) builds BOTH regions from the same code
        CI/CD deploys app + runs DB migrations (Flyway) safely
        CloudWatch alarms → SNS → on-call
```

### 27.2 Component checklist
- [ ] VPC with public + private(app) + private(data) subnets across **2+ AZs**.
- [ ] RDS **Multi-AZ**, encrypted, deletion protection, 14-day backups.
- [ ] **Secrets Manager** with rotation.
- [ ] **Cross-Region read replica** (Pilot Light/Warm Standby).
- [ ] **AWS Backup** plan with cross-Region copy.
- [ ] **Route 53** failover record (`db.myapp.com`, low TTL).
- [ ] **CloudWatch** alarms (CPU, storage, connections, replica lag) → SNS.
- [ ] **Terraform** repo builds both Regions; state per env.
- [ ] **CI/CD** for app + DB migrations (snapshot-before-migrate, blue/green).
- [ ] **SSM Automation** failover runbook.
- [ ] **Quarterly DR game days** documented.

### 27.3 Decision guide: which DB & DR for the project
| Need | Choose |
|------|--------|
| Standard SQL, moderate scale | RDS MySQL/PostgreSQL Multi-AZ |
| High scale, low latency, best DR | **Aurora** + **Global Database** |
| Spiky/unpredictable load | Aurora **Serverless v2** |
| Cheapest DR, hours OK | Backup & Restore (cross-Region snapshots) |
| RTO minutes, RPO seconds | Cross-Region replica (Warm Standby) |
| RTO/RPO near zero | Aurora Global DB / Active-Active |

---

## 36. Cleanup (Avoid Surprise Bills)

> Do this after labs. Order matters (replicas before primary).

```bash
# Delete read replicas / DR replicas first
aws rds delete-db-instance --db-instance-identifier dr-mysql-rr \
  --region us-west-2 --skip-final-snapshot

# Disable deletion protection, then delete the primary (keep a final snapshot)
aws rds modify-db-instance --db-instance-identifier prod-mysql-1 \
  --no-deletion-protection --apply-immediately

aws rds delete-db-instance --db-instance-identifier prod-mysql-1 \
  --final-db-snapshot-identifier prod-mysql-1-final

# Delete old snapshots you no longer need
aws rds delete-db-snapshot --db-snapshot-identifier prod-mysql-1-before-migration

# Delete subnet group, security group, and (if created) the VPC
aws rds delete-db-subnet-group --db-subnet-group-name prod-db-subnets
```
Also check: **NAT Gateways**, **Elastic IPs**, **EC2/ASG**, **KMS keys**, **Backup vaults** — these cost money too. Use **Cost Explorer** and **Billing alarms**.

---

## 37. Glossary & Cheat Sheet

### 29.1 Glossary
- **AZ** — Availability Zone (isolated data center).
- **Multi-AZ** — synchronous standby in another AZ for HA (auto-failover).
- **Read Replica** — async read-only copy; cross-Region ones aid DR.
- **PITR** — Point-in-Time Recovery using backups + transaction logs.
- **RPO** — max acceptable data loss (time).
- **RTO** — max acceptable downtime.
- **Pilot Light / Warm Standby / Active-Active** — DR strategies by cost/speed.
- **Aurora Global Database** — cross-Region managed replication, sub-second lag.
- **IaC** — Infrastructure as Code (Terraform/CloudFormation).
- **Blue/Green** — deploy changes to a parallel copy, then switch.

### 29.2 Most-used RDS CLI commands
```bash
# List / describe
aws rds describe-db-instances --query "DBInstances[].DBInstanceIdentifier"
aws rds describe-db-snapshots --db-instance-identifier prod-mysql-1

# Create / modify / delete
aws rds create-db-instance ...
aws rds modify-db-instance --db-instance-identifier X --apply-immediately ...
aws rds delete-db-instance --db-instance-identifier X --final-db-snapshot-identifier X-final

# Backup / restore
aws rds create-db-snapshot --db-instance-identifier X --db-snapshot-identifier X-snap
aws rds restore-db-instance-from-db-snapshot --db-instance-identifier X-new --db-snapshot-identifier X-snap
aws rds restore-db-instance-to-point-in-time --source-db-instance-identifier X --target-db-instance-identifier X-pitr --restore-time <ts>

# Replicas / failover
aws rds create-db-instance-read-replica --db-instance-identifier RR --source-db-instance-identifier X
aws rds promote-read-replica --db-instance-identifier RR
aws rds reboot-db-instance --db-instance-identifier X --force-failover

# Aurora global
aws rds failover-global-cluster --global-cluster-identifier G --target-db-cluster-identifier <arn>
```

### 29.3 Quick mental model
1. **RDS** = managed DB.
2. **Multi-AZ** = HA inside a Region (automatic).
3. **Backups/Snapshots/PITR** = recover from mistakes/corruption.
4. **Cross-Region replica / Aurora Global / AWS Backup** = survive a Region disaster.
5. **IaC + CI/CD + runbooks + game days** = the DevOps discipline that makes it reliable and repeatable.

---

*End of guide. Start with Sections 2–7 hands-on, then layer in HA (8), backups (10–11), and DR (15–22). Automate everything with IaC (23) once you're comfortable.*
