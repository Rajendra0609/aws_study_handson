# ☁️ AWS Console Hands-On Master Guide — DevOps L3 Engineer Reference

> **Audience:** L3 DevOps Engineers | **Approach:** Console-first, CLI-paired, production-grade  
> **Topics:** IAM · EC2 & Storage · S3 · VPC · Route 53 · Containers (ECR · ECS · EKS)  
> **Region default:** `ap-south-1` (Mumbai) — swap to your nearest region throughout

---

## 📋 Table of Contents

| # | Domain | Key Topics |
|---|--------|-----------|
| 1 | [IAM — Identity & Access Management](#1--iam--identity--access-management) | Users, Groups, Policies, Roles, MFA, CLI |
| 2 | [EC2 & Storage](#2--ec2--storage) | Instance types, User Data, EBS, EFS, Snapshots, AMI, Spot, Cost |
| 3 | [S3 — Object Storage](#3--s3--object-storage) | Buckets, Policies, Versioning, Lifecycle, Encryption, Access Points |
| 4 | [VPC — Networking](#4--vpc--networking) | Subnets, IGW, NAT, Security Groups, NACLs, Peering, TGW, Endpoints |
| 5 | [Route 53 — DNS](#5--route-53--dns) | Records, Routing Policies, Health Checks, Hybrid DNS |
| 6 | [Containers — ECR · ECS · EKS](#6--containers--ecr--ecs--eks) | Image registry, Fargate, Kubernetes, GitOps, Security |
| 7 | [Cross-Domain Self-Practice Questions](#7--cross-domain-self-practice-questions) | 80+ scenario-based challenges |

---

## 1 · IAM — Identity & Access Management

### Core Architecture

```
AWS Account
├── Root Account        ← Lock away immediately. Enable MFA. Never use day-to-day.
├── Group: Admins       ← Policy: AdministratorAccess
│   └── User: alice
├── Group: DevOps       ← Policy: PowerUserAccess + custom policies
│   ├── User: bob
│   └── User: carol
└── Role: EC2-S3-Reader ← Assumed by EC2/Lambda — never use access keys on services
```

### Key Concepts Quick Reference

| Concept | What it is | L3 Insight |
|---------|-----------|-----------|
| **User** | Long-term credentials; password + access keys | Max 2 access keys; rotate every 90 days |
| **Group** | Collection of users sharing permissions | Assign policies to groups, never to users directly |
| **Policy** | JSON Allow/Deny for Actions on Resources | Explicit Deny always wins over any Allow |
| **Role** | Temporary credentials assumed by services or users | Preferred over access keys for EC2/Lambda/ECS |
| **MFA** | Second factor for console login | Mandatory on root and all admin users |
| **Credentials Report** | Account-wide audit CSV | Run monthly; catch stale keys and missing MFA |
| **Access Advisor** | Per-user service last-used data | Use to apply least-privilege — remove unused services |

### Console: Create Admin User (Day 1 Task)

```
IAM → User Groups → Create Group
  Name: Admins
  Policy: AdministratorAccess

IAM → Users → Create User
  Username: admin-yourname
  Enable Console Access → Custom Password
  Add to group: Admins
  Download .csv (has sign-in URL, credentials)
```

> **After this:** Sign in as the IAM user. Never use root for daily work again.

### Policy Structure & Evaluation

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:Describe*", "ec2:List*"],
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": "ec2:TerminateInstances",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {"aws:RequestedRegion": "ap-south-1"}
      }
    }
  ]
}
```

**Evaluation logic:** Explicit DENY → wins. Explicit ALLOW → permitted. No statement → implicit DENY.

### Custom Policy Hands-On

```bash
# Create policy from file
aws iam create-policy \
  --policy-name EC2-ReadOnly-NoTerminate \
  --policy-document file://policy.json

# Attach to a group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::<account-id>:policy/EC2-ReadOnly-NoTerminate

# Test with Policy Simulator before applying
# IAM → Policy Simulator → select user → select service → Run Simulation
```

### IAM Roles — The Right Way to Give Services Access

```bash
# Create role for EC2 to access S3
aws iam create-role \
  --role-name EC2-S3-ReadOnly-Role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name EC2-S3-ReadOnly-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Attach to running instance
aws ec2 associate-iam-instance-profile \
  --instance-id i-xxxx \
  --iam-instance-profile Name=EC2-S3-ReadOnly-Role
```

From inside the EC2: `aws s3 ls` — works without any credentials configured.

### CLI Setup & Multi-Profile

```bash
aws configure                          # Default profile
aws configure --profile production     # Named profile
aws configure --profile staging

# Use a profile
aws s3 ls --profile production

# Verify who you are
aws sts get-caller-identity

# Quick commands
aws iam list-users
aws iam list-attached-user-policies --user-name alice
aws iam generate-credential-report
aws iam get-credential-report --output text --query Content | base64 -d
```

### Access Key Rotation (Zero-Downtime Pattern)

```
Step 1: Create Key 2 (now have Key 1 + Key 2)
Step 2: Update all apps/scripts to use Key 2
Step 3: Verify Key 2 works everywhere
Step 4: Deactivate Key 1
Step 5: Wait 24–48 hours
Step 6: Delete Key 1
```

### IAM Security Checklist

- [ ] Root account has MFA and zero access keys
- [ ] All humans have individual IAM users (never share)
- [ ] Admin users have MFA
- [ ] Access keys rotated every 90 days (verify via Credentials Report)
- [ ] All services use IAM Roles — zero hardcoded access keys
- [ ] Least-privilege: review Access Advisor monthly
- [ ] CloudTrail enabled for API audit logging

---

## 2 · EC2 & Storage

### Instance Type Families

| Family | Use Case | Example Types |
|--------|----------|--------------|
| `t3`, `t4g` | Burstable, dev/test, web servers | `t3.micro`, `t3.medium` |
| `m6i`, `m7g` | General-purpose app servers | `m6i.large`, `m7g.xlarge` |
| `c6i`, `c7g` | Compute-heavy, batch, HPC | `c6i.2xlarge` |
| `r6i`, `r7g` | Memory-heavy, databases, caches | `r6i.4xlarge` |
| `i3`, `i4i` | Storage-optimized, NoSQL | `i3.xlarge` |
| `p3`, `g4dn` | GPU, ML training, video | `g4dn.xlarge` |

> **L3 Rule:** Use Graviton (`t4g`, `m7g`, `c7g`) — 20% cheaper, often faster. Always valid unless GPU or Windows required.

### Launch EC2 with User Data (Auto-Bootstrap)

```
EC2 → Launch Instance
  AMI: Amazon Linux 2023
  Instance Type: t3.micro
  Key Pair: create or select
  Security Group: allow SSH (22) from your IP, HTTP (80) from 0.0.0.0/0
  Advanced Details → User Data:
```

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd && systemctl enable httpd
echo "<h1>$(hostname -f) — $(date)</h1>" > /var/www/html/index.html
```

```bash
# Tail user data log from inside instance
tail -f /var/log/cloud-init-output.log

# Instance metadata (IMDSv2)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/public-ipv4
```

### Networking: IP Types

| Type | Persists on Stop? | Internet? | Cost |
|------|-------------------|-----------|------|
| Private IP | ✅ Yes | VPC-only | Free |
| Public IP | ❌ Changes | Yes | Free while running |
| Elastic IP | ✅ Yes (static) | Yes | ~$0.005/hr when unattached |

> **L3 Tip:** Avoid Elastic IPs. Use DNS (Route 53) or a Load Balancer instead. Elastic IPs are charged when idle and clutter your account.

### SSH & EC2 Instance Connect

```bash
# Linux/Mac
chmod 400 my-key.pem
ssh -i my-key.pem ec2-user@<PUBLIC-IP>

# Via bastion (ProxyJump)
ssh -i my-key.pem \
  -o "ProxyJump ec2-user@<BASTION-IP>" \
  ec2-user@<PRIVATE-IP>

# No key needed — browser-based (port 22 must be open)
# EC2 Console → Connect → EC2 Instance Connect
```

### EBS Volume Types

| Type | IOPS | Use Case | Boot? |
|------|------|----------|-------|
| `gp3` | Up to 16,000 | General purpose (default) | ✅ |
| `gp2` | Up to 16,000 | Legacy (switch to gp3) | ✅ |
| `io2` | Up to 256,000 | High-performance DBs | ✅ |
| `st1` | 500 max | Big data, streaming | ❌ |
| `sc1` | 250 max | Cold archive | ❌ |

> **L3 Rule:** Always use `gp3` over `gp2` — same IOPS, more throughput, ~20% cheaper.

### Attach & Mount an EBS Volume

```bash
# Console: EC2 → Volumes → Create Volume (same AZ as instance) → Attach

# Inside the instance:
lsblk                          # List block devices
sudo mkfs -t ext4 /dev/xvdf    # Format (only first time!)
sudo mkdir /data
sudo mount /dev/xvdf /data     # Mount
df -h                          # Confirm

# Persist mount across reboots (add to /etc/fstab)
echo "/dev/xvdf /data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

### EBS Snapshots — Cross-AZ & Cross-Region Migration

```bash
# Create snapshot
aws ec2 create-snapshot --volume-id vol-xxxx --description "Before deployment"

# Copy snapshot to another region (enables cross-region migration)
aws ec2 copy-snapshot \
  --source-region ap-south-1 \
  --source-snapshot-id snap-xxxx \
  --region us-east-1

# Create volume from snapshot in a different AZ
aws ec2 create-volume \
  --snapshot-id snap-xxxx \
  --availability-zone ap-south-1b \
  --volume-type gp3

# Automate with Data Lifecycle Manager
# EC2 → Lifecycle Manager → Create Policy → Snapshot schedule
```

### AMI — Build a Custom Image

```
Console:
EC2 → Select running instance → Actions → Image and Templates → Create Image
  Image name: my-nginx-app-v1.0
  No Reboot: ✅ (avoid downtime)
→ Create Image

Launch new instance from AMI:
EC2 → Launch Instance → My AMIs → select your image
```

> **L3 Use:** Bake-in agents, config, security hardening, app binaries → faster ASG scale-out, consistent environments.

### EFS — Shared File System (Linux Only)

```bash
# Console: EFS → Create File System → VPC → Multi-AZ

# On multiple EC2 instances (same or different AZs)
sudo yum install -y amazon-efs-utils
sudo mkdir /efs
sudo mount -t efs -o tls <EFS-DNS-NAME>:/ /efs

# Instance 1: write
echo "Hello from Instance 1" | sudo tee /efs/shared.txt

# Instance 2: read (same content instantly)
cat /efs/shared.txt
```

**EBS vs EFS decision:**

```
Need shared access across instances?   → EFS (ReadWriteMany)
Single instance, max IOPS, persistent? → io2 EBS
Temporary scratch / cache?             → Instance Store (lost on stop)
Default general-purpose?               → gp3 EBS
```

### EC2 Purchasing Options

| Model | Savings | When to Use |
|-------|---------|-------------|
| On-Demand | Baseline | Short-term, unpredictable |
| Reserved (1 or 3yr) | Up to 72% | Steady-state (web servers, DBs) |
| Savings Plans | Up to 72% | Flexible commit — best for mixed fleets |
| Spot | Up to 90% | Stateless, fault-tolerant, batch jobs |
| Dedicated Host | — | BYOL licensing, compliance |

### Spot Instance — Interruption Handler

```bash
# Poll instance metadata for 2-minute warning
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INTERRUPTION=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/spot/termination-time)
[ -n "$INTERRUPTION" ] && echo "SPOT INTERRUPTION at $INTERRUPTION — begin graceful shutdown"
```

### Cost Optimization Checklist

```bash
# Find idle EC2 instances (Compute Optimizer)
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[?finding==`OVER_PROVISIONED`].[instanceArn,recommendationOptions[0].instanceType]' \
  --output table

# Find unattached EBS volumes (bleeding money)
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]' --output table

# Find unattached Elastic IPs
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].[PublicIp,AllocationId]' --output table
```

- [ ] Unattached EBS volumes deleted
- [ ] Old snapshots on lifecycle policy (Data Lifecycle Manager)
- [ ] Over-provisioned instances right-sized (Compute Optimizer)
- [ ] Spot or Savings Plans purchased for steady workloads
- [ ] EFS Lifecycle Management enabled (auto-move to IA after 30 days)
- [ ] Budget alert configured

---

## 3 · S3 — Object Storage

### Core Concepts

- **Bucket:** Globally unique name, region-scoped, flat key-value store
- **Object:** Key + Value + Metadata + Version ID; max size 5 TB (multipart upload required >5 GB)
- **URL format:** `https://<bucket>.s3.<region>.amazonaws.com/<key>`
- **Folders are simulated** via `/` in key names — S3 is flat

### Create Bucket & Upload

```
S3 → Create Bucket
  Name: my-app-assets-2024 (globally unique)
  Region: ap-south-1
  Block all public access: ✅ (keep on by default)
→ Create Bucket → Upload files
```

### Bucket Policy — Public Read (for static sites)

```
S3 → Bucket → Permissions → Block Public Access → uncheck all → Save
Permissions → Bucket Policy → Edit → paste:
```

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
  }]
}
```

### Enforce Encryption at Upload (Deny Unencrypted PUT)

```json
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::YOUR-BUCKET/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

### Static Website Hosting

```
S3 → Bucket → Properties → Static website hosting → Enable
  Index document: index.html
  Error document: error.html
→ Save

# HTTP only. For HTTPS, put CloudFront in front.
# Website URL: http://<bucket>.s3-website-<region>.amazonaws.com
```

### Versioning

```bash
# Enable via console: Bucket → Properties → Bucket Versioning → Enable

# List all versions
aws s3api list-object-versions --bucket my-bucket

# Delete specific version (permanent delete)
aws s3api delete-object --bucket my-bucket --key file.txt --version-id <version-id>

# Restore: delete the Delete Marker to undelete an object
```

### Lifecycle Rules — Auto-Tier & Expire

```
S3 → Bucket → Management → Lifecycle rules → Create lifecycle rule
  Name: archive-old-logs
  Apply to: prefix logs/
  Actions:
    Transition: after 30 days → Standard-IA
    Transition: after 90 days → Glacier Flexible Retrieval
    Expire: after 365 days
    Delete incomplete multipart uploads: after 7 days ← always add this
```

```bash
# Via CLI
aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json
```

### S3 Storage Classes

| Class | Retrieval | Min Duration | Use Case |
|-------|-----------|-------------|---------|
| Standard | Instant | None | Frequently accessed |
| Standard-IA | Instant | 30 days | Infrequent, rapid retrieval |
| One Zone-IA | Instant | 30 days | Non-critical, single AZ |
| Intelligent-Tiering | Instant | None | Unknown patterns |
| Glacier Instant | Instant | 90 days | Archives, quarterly access |
| Glacier Flexible | 1 min–12 hr | 90 days | Archives |
| Glacier Deep Archive | 12–48 hr | 180 days | Long-term compliance |

### Replication (CRR / SRR)

```bash
# Both source and destination must have versioning enabled
# Console: Source Bucket → Management → Replication rules → Create

# CLI: create IAM role, then:
aws s3api put-bucket-replication \
  --bucket source-bucket \
  --replication-configuration file://replication.json

# Replicate existing objects (not auto-replicated when rule first created)
# S3 → Batch Operations → Create job → Replication
```

### Encryption

| Method | Key Owner | Notes |
|--------|----------|-------|
| SSE-S3 | AWS | Default since Jan 2023, AES-256 |
| SSE-KMS | You (KMS) | Audit trail in CloudTrail; key rotation |
| SSE-C | You entirely | Pass key with every request; AWS never stores it |
| Client-Side | You | Encrypted before upload |

```bash
# Upload with SSE-KMS
aws s3 cp file.txt s3://my-bucket/ \
  --sse aws:kms \
  --sse-kms-key-id alias/my-key

# Upload with SSE-C
aws s3 cp file.txt s3://my-bucket/ \
  --sse-c AES256 \
  --sse-c-key fileb://encryption-key.bin
```

### Pre-Signed URLs (Temporary Access)

```bash
# Download URL — valid 5 minutes
aws s3 presign s3://my-bucket/private-file.pdf --expires-in 300

# Upload URL — let users upload directly to S3
aws s3 presign s3://my-bucket/target.txt \
  --expires-in 3600 \
  --http-method PUT
```

### S3 Object Lock (WORM Compliance)

```
Enable at bucket creation → cannot be turned off after.

Compliance mode: not even root can delete
Governance mode: users with s3:BypassGovernanceRetention CAN delete
Legal Hold: independent lock — placed/removed by authorized users

Use case: SEC 17a-4, HIPAA, PCI-DSS immutable records
```

### VPC Endpoint for S3 (Free Private Access)

```bash
# Create Gateway Endpoint (free!) — traffic never leaves AWS
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxx \
  --service-name com.amazonaws.ap-south-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-xxxx

# Now private EC2 instances can aws s3 ls without NAT Gateway
```

### S3 Best Practices Checklist

- [ ] Versioning enabled on production buckets
- [ ] Default encryption set (SSE-S3 or SSE-KMS)
- [ ] Lifecycle rules to expire logs and old versions
- [ ] Block Public Access at account level
- [ ] Access logging enabled (to a separate bucket)
- [ ] MFA Delete on critical buckets (CLI only, root only)
- [ ] Incomplete multipart upload cleanup rule (7 days)
- [ ] S3 Storage Lens dashboard enabled for visibility

---

## 4 · VPC — Networking

### CIDR Plan (Reference Architecture)

```
prod-vpc:            10.0.0.0/16   (65,536 IPs)
├── Public  AZ-1a:   10.0.1.0/24  → Bastion, NAT GW, ALB
├── Public  AZ-1b:   10.0.2.0/24  → ALB
├── App     AZ-1a:   10.0.3.0/24  → App Servers / EKS nodes
├── App     AZ-1b:   10.0.4.0/24
├── DB      AZ-1a:   10.0.5.0/24  → RDS Primary (no internet route)
└── DB      AZ-1b:   10.0.6.0/24  → RDS Standby

AWS reserves 5 IPs per subnet: .0 .1 .2 .3 .255
```

> **L3 Rule:** Plan CIDRs before you create anything — they cannot be changed. Use IPAM for multi-VPC environments.

### Create VPC + Subnets + IGW + NAT (CLI)

```bash
# VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=prod-vpc}]' \
  --query 'Vpc.VpcId' --output text)
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support

# Public Subnet
PUB_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 --availability-zone ap-south-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=pub-subnet-1a}]' \
  --query 'Subnet.SubnetId' --output text)
aws ec2 modify-subnet-attribute --subnet-id $PUB_1A --map-public-ip-on-launch

# Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
  --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=prod-igw}]' \
  --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# NAT Gateway (in public subnet)
EIP=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_GW=$(aws ec2 create-nat-gateway --subnet-id $PUB_1A \
  --allocation-id $EIP --query 'NatGateway.NatGatewayId' --output text)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

# Route Tables
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT \
  --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_1A
```

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance (ENI) | Subnet |
| Stateful? | ✅ Yes | ❌ No (both directions needed) |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules | Numbered order (lowest first) |
| Default | Deny all in | Allow all |

### Security Group — Layered Architecture

```bash
# Bastion SG — only you can SSH
BASTION_SG=$(aws ec2 create-security-group \
  --group-name bastion-sg --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG --protocol tcp --port 22 --cidr $(curl -s ifconfig.me)/32

# App SG — only ALB on 8080, only bastion on 22
APP_SG=$(aws ec2 create-security-group \
  --group-name app-sg --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $APP_SG --protocol tcp --port 8080 --source-group $ALB_SG
aws ec2 authorize-security-group-ingress \
  --group-id $APP_SG --protocol tcp --port 22 --source-group $BASTION_SG

# DB SG — only app on 3306
DB_SG=$(aws ec2 create-security-group \
  --group-name db-sg --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress \
  --group-id $DB_SG --protocol tcp --port 3306 --source-group $APP_SG
```

### NACLs — Stateless (L3 Must-Know)

```
Public NACL — Inbound Rules:
  100: TCP 80   0.0.0.0/0 ALLOW   ← HTTP
  110: TCP 443  0.0.0.0/0 ALLOW   ← HTTPS
  120: TCP 22   YOUR-IP/32 ALLOW  ← SSH
  130: TCP 1024-65535 0.0.0.0/0 ALLOW  ← Ephemeral return ports (CRITICAL)
  *:  All DENY

Public NACL — Outbound Rules:
  100: TCP 80   0.0.0.0/0 ALLOW
  110: TCP 443  0.0.0.0/0 ALLOW
  120: TCP 1024-65535 0.0.0.0/0 ALLOW  ← Response ports
  *:  All DENY
```

> **Common mistake:** Forgetting ephemeral ports (1024–65535) in NACLs causes mysterious timeouts. Security Groups are stateful — NACLs are not.

### VPC Flow Logs

```bash
# Send to S3 and CloudWatch
aws ec2 create-flow-logs \
  --resource-type VPC --resource-ids $VPC_ID \
  --traffic-type ALL \
  --log-destination-type s3 \
  --log-destination arn:aws:s3:::my-flow-logs-bucket \
  --max-aggregation-interval 60

# Analyze with CloudWatch Insights
# Filter: find SSH brute-force attempts
# fields @timestamp, srcaddr, action
# | filter dstport = 22 and action = "REJECT"
# | stats count(*) by srcaddr | sort desc | limit 20
```

### VPC Peering vs Transit Gateway

```
Peering (non-transitive):          Transit Gateway (transitive hub-spoke):
A ↔ B (1 connection)               A ─┐
A ↔ C (1 connection)               B ─┤── TGW ── all talk to all
B ↔ C (1 connection)               C ─┘
= 3 connections for 3 VPCs         = 3 attachments (scales to 100+ VPCs)

Use Peering: < 5 VPCs, simple topology
Use TGW: 5+ VPCs, multi-account, need on-prem connectivity
```

### VPC Endpoints (Save NAT Gateway Costs)

```bash
# Gateway Endpoint for S3 — FREE, traffic never leaves AWS
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids $PRIV_RT

# Interface Endpoint for SSM (removes need for NAT or bastion)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.ssm \
  --vpc-endpoint-type Interface \
  --subnet-ids $PRIV_APP_1A \
  --security-group-ids $ENDPOINT_SG \
  --private-dns-enabled
# Repeat for ssmmessages and ec2messages
```

### Route 53 Private Hosted Zone (Internal DNS)

```bash
# Create private hosted zone
aws route53 create-hosted-zone \
  --name internal.yourcompany.com \
  --caller-reference $(date +%s) \
  --hosted-zone-config PrivateZone=true \
  --vpc VPCRegion=ap-south-1,VPCId=$VPC_ID

# Now use db.internal.yourcompany.com instead of 10.0.5.10
# When DB moves — update DNS, zero app changes
```

### Connectivity Validation Tests

| Test | Expected | If failing — check |
|------|----------|-------------------|
| Bastion → internet | ✅ Works | Public subnet has IGW route |
| Private app → internet | ✅ Works (via NAT) | Private RT has NAT GW route |
| Internet → private directly | ❌ Fail | No public IP, SG blocks |
| DB → internet | ❌ Fail | No route in DB route table |
| App → S3 via endpoint | ✅ Works (no NAT cost) | Endpoint route in RT |

```bash
# From private instance — verify NAT GW is used
curl https://ifconfig.me  # Shows NAT GW's Elastic IP, not instance IP

# Reachability Analyzer (no traffic sent — free analysis)
PATH_ID=$(aws ec2 create-network-insights-path \
  --source <bastion-instance-id> \
  --destination <private-instance-id> \
  --protocol tcp --destination-port 22 \
  --query 'NetworkInsightsPath.NetworkInsightsPathId' --output text)
aws ec2 start-network-insights-analysis --network-insights-path-id $PATH_ID
```

### VPC Cleanup Order (Avoid Billing Traps)

```bash
# Always in this order — dependencies break on reverse
1. Terminate EC2 instances
2. Delete NAT Gateway → wait → Release Elastic IP
3. Delete Network Firewall (if used)
4. Delete Transit Gateway Attachments → Transit Gateway
5. Delete VPN Connection → Detach VGW → Delete VGW
6. Delete VPC Endpoints
7. Delete VPC Peering Connections
8. Detach + Delete Internet Gateway
9. Delete Route Tables (custom)
10. Delete Security Groups
11. Delete Subnets
12. Delete VPC
```

---

## 5 · Route 53 — DNS

### DNS Record Types Reference

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Domain → IPv4 | `app.co → 13.234.56.78` |
| **AAAA** | Domain → IPv6 | `app.co → 2001:db8::1` |
| **CNAME** | Domain → Domain (not at root) | `www → app.co` |
| **Alias** | AWS-native; works at root | `co → alb.amazonaws.com` |
| **MX** | Mail routing | Priority + mail server |
| **TXT** | Verification, SPF, DKIM | `"v=spf1 include:..."` |
| **NS** | Name servers for zone | Route 53 NS servers |

### CNAME vs Alias — Critical Difference

| Feature | CNAME | Alias |
|---------|-------|-------|
| Root domain (`@`)? | ❌ No | ✅ Yes |
| Points to AWS resources? | ❌ Use Alias | ✅ ELB, CF, S3, API GW |
| DNS query charge? | ✅ Charged | ❌ Free for AWS resources |
| TTL | You set it | Route 53 manages it |

> **Rule:** Anything pointing to an AWS resource → use Alias. Root domain (`co`) must use Alias.

### All Routing Policies

| Policy | Use Case | Health Checks | Notes |
|--------|---------|---------------|-------|
| **Simple** | Single endpoint | ❌ No | Returns all IPs; client picks randomly |
| **Weighted** | A/B testing, canary deploy | ✅ Optional | Weight 0 = excluded, not deleted |
| **Latency** | Global multi-region | ✅ Optional | Based on network latency data |
| **Failover** | Active-passive HA | ✅ Required on Primary | Exactly 2 records |
| **Geolocation** | Compliance, localization | ✅ Optional | Always create Default record |
| **Geoproximity** | Distance + bias control | ✅ Optional | Requires Traffic Policy |
| **IP-Based** | CIDR-level routing | ✅ Optional | Most granular |
| **Multi-Value** | Client-side LB with HA | ✅ Recommended | Up to 8 healthy records |

### Hands-On: Latency-Based Routing (3 Regions)

```bash
# Create 3 records — same hostname, different regions
# Console: Route 53 → Hosted Zone → Create Record → Latency policy

# CLI example (ap-south-1 record)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234ABCDEF \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "app.yourco.com",
        "Type": "A",
        "SetIdentifier": "latency-mumbai",
        "Region": "ap-south-1",
        "TTL": 60,
        "ResourceRecords": [{"Value": "<mumbai-ip>"}]
      }
    }]
  }'
# Repeat with different Region and IP for eu-west-1, us-east-1
```

### Health Checks

```bash
# Create HTTP health check
aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "IPAddress": "13.234.56.78",
    "Port": 80,
    "Type": "HTTP",
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3,
    "EnableSNI": false
  }'

# Check status
aws route53 get-health-check-status --health-check-id <id>
```

### Failover Routing (Active-Passive)

```
Primary record:  app.co → Mumbai EC2 (health check REQUIRED)
Secondary record: app.co → S3 Static "Maintenance" page (no health check needed)

When health check fails → Route 53 automatically serves secondary
When primary recovers → traffic returns to primary
```

### TTL Strategy for Production Changes

```
Normal operation:   TTL = 3600   ← balance of cost and flexibility
48h before IP change: TTL = 60  ← reduce early so caches drain
After IP change:    TTL = 3600  ← restore once propagated
```

### Hybrid DNS — Route 53 Resolver

```bash
# For on-prem ↔ AWS private DNS resolution

# Inbound endpoint: on-prem can query AWS private DNS
aws route53resolver create-resolver-endpoint \
  --creator-request-id $(date +%s) \
  --direction INBOUND \
  --security-group-ids $ENDPOINT_SG \
  --ip-addresses SubnetId=$PRIV_APP_1A SubnetId=$PRIV_APP_1B

# Outbound endpoint: VPC can query on-prem DNS
aws route53resolver create-resolver-endpoint \
  --creator-request-id $(date +%s) \
  --direction OUTBOUND \
  --security-group-ids $ENDPOINT_SG \
  --ip-addresses SubnetId=$PRIV_APP_1A SubnetId=$PRIV_APP_1B

# Forwarding rule: send *.internal.corp → 192.168.1.53 (on-prem DNS)
aws route53resolver create-resolver-rule \
  --creator-request-id $(date +%s) \
  --rule-type FORWARD \
  --domain-name internal.corp \
  --resolver-endpoint-id <outbound-endpoint-id> \
  --target-ips Ip=192.168.1.53,Port=53
```

### DNS Troubleshooting Commands

```bash
# Basic lookup
dig app.yourco.com
dig +short app.yourco.com
nslookup app.yourco.com

# Check NS records (who controls the domain)
dig NS yourco.com

# Query specific DNS server
dig @8.8.8.8 app.yourco.com

# Trace full resolution path
dig +trace app.yourco.com

# Watch TTL countdown
watch -n 1 "dig +short app.yourco.com"
```

---

## 6 · Containers — ECR · ECS · EKS

### Container Concepts Quick Reference

| Term | Meaning |
|------|---------|
| **Image** | Packaged snapshot of app + dependencies |
| **Container** | Running instance of an image |
| **Registry (ECR)** | AWS private image store |
| **ECS** | AWS container orchestrator (simpler, AWS-native) |
| **EKS** | AWS managed Kubernetes (more powerful, more ops) |
| **Task / Pod** | Unit running one or more containers |
| **Fargate** | Serverless compute — no EC2 to manage |

### ECR — Image Registry

```bash
# Authenticate Docker to ECR (valid 12 hours)
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin \
    <account-id>.dkr.ecr.ap-south-1.amazonaws.com

# Create repository
aws ecr create-repository \
  --repository-name my-app \
  --image-tag-mutability IMMUTABLE   # ← Production safety

# Build, tag, push
docker build -t my-app .
docker tag my-app:latest \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0
docker push \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0

# Verify
aws ecr list-images --repository-name my-app
```

### ECR Lifecycle Policy (Prevent Storage Bloat)

```bash
cat > lifecycle.json << 'EOF'
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Remove untagged images after 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": {"type": "expire"}
    },
    {
      "rulePriority": 2,
      "description": "Keep only 10 tagged images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": {"type": "expire"}
    }
  ]
}
EOF

aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text file://lifecycle.json
```

### Docker Multi-Stage Build (Smaller, Safer Images)

```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Production (lean image)
FROM node:20-alpine
WORKDIR /app
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser                                      # Non-root
COPY --from=builder /app/node_modules ./node_modules
COPY --chown=appuser:appgroup src/ ./src/
EXPOSE 3000
CMD ["node", "src/index.js"]
```

```bash
# Multi-arch for Graviton (ARM) + x86 cost savings
docker buildx create --name multiarch --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag <ecr-uri>/my-app:v1.0 \
  --push .
```

### ECS — Fargate (Serverless Containers)

```
ECS Architecture:
Cluster
└── Service (keeps N tasks running, integrates with ALB, handles rolling deploys)
     └── Task (running group of containers, based on Task Definition)
          └── Container (the Docker container)

Two IAM Roles:
  Task Execution Role → ECS agent uses this to pull image + send logs (ecsTaskExecutionRole)
  Task Role          → Your app code uses this to call AWS APIs (S3, DynamoDB etc.)
```

```bash
# Create cluster
aws ecs create-cluster --cluster-name my-cluster

# Register task definition
cat > task-def.json << 'EOF'
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512", "memory": "1024",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "my-app",
    "image": "<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0",
    "portMappings": [{"containerPort": 80, "protocol": "tcp"}],
    "essential": true,
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/my-app",
        "awslogs-region": "ap-south-1",
        "awslogs-stream-prefix": "ecs"
      }
    }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file://task-def.json
```

### ECS Service + ALB (Production Pattern)

```bash
# Create target group and ALB first, then:
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-app-service \
  --task-definition my-app-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[$SUBNET_ID],
    securityGroups=[$SG_ID],
    assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=my-app,containerPort=80" \
  --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true}"
```

### ECS Operations

```bash
# Force redeploy (new image, same tag)
aws ecs update-service \
  --cluster my-cluster \
  --service my-app-service \
  --force-new-deployment

# Tail logs live
aws logs tail /ecs/my-app --follow

# Shell into running container (ECS Exec)
aws ecs update-service --cluster my-cluster --service my-app-service \
  --enable-execute-command --force-new-deployment
TASK_ARN=$(aws ecs list-tasks --cluster my-cluster \
  --service-name my-app-service --query "taskArns[0]" --output text)
aws ecs execute-command \
  --cluster my-cluster --task $TASK_ARN \
  --container my-app --interactive --command "/bin/sh"
```

### Secrets Management in ECS

```bash
# Store secret
aws secretsmanager create-secret \
  --name /myapp/prod/db-password \
  --secret-string "mypassword123"
SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id /myapp/prod/db-password --query ARN --output text)

# Reference in task definition containerDefinitions:
# "secrets": [{"name": "DB_PASSWORD", "valueFrom": "<secret-arn>"}]
```

### ECS Auto Scaling

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-app-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 --max-capacity 10

aws application-autoscaling put-scaling-policy \
  --policy-name cpu-tracking \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-app-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 50.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 60,
    "ScaleOutCooldown": 60
  }'
```

### ECS Deployment Strategies

| Strategy | How it works | Use case |
|----------|-------------|---------|
| **Rolling update** | Replaces tasks gradually (default) | Most workloads |
| **Blue/Green (CodeDeploy)** | Runs new alongside old; shifts traffic when healthy | Zero-downtime |
| **Circuit breaker** | Auto-rollback if new tasks fail to start | Safety net (always enable) |

### EKS — Kubernetes on AWS

```bash
# Create cluster (takes 15–20 min)
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-south-1 \
  --version 1.30 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2 --nodes-min 1 --nodes-max 4 \
  --managed

# Connect kubectl
aws eks update-kubeconfig --name my-eks-cluster --region ap-south-1

# Verify
kubectl get nodes
kubectl get pods --all-namespaces
```

### Kubernetes Core Objects

| Object | Purpose | L3 Note |
|--------|---------|---------|
| **Pod** | Smallest unit — 1+ containers | Ephemeral — never store state here |
| **Deployment** | Manages replica pods, rolling updates | Always use Deployments, not bare Pods |
| **Service** | Stable network endpoint | ClusterIP (internal) / LoadBalancer (NLB) |
| **ConfigMap** | Non-sensitive config | Mount as env vars or files |
| **Secret** | Sensitive data (base64) | Use External Secrets Operator in prod |
| **Ingress** | HTTP path/host routing | Needs AWS Load Balancer Controller |
| **HPA** | Scale pods on CPU/memory | Requires Metrics Server |
| **PDB** | Protect pods during disruptions | Always set before node drains |

### Deploy App to EKS

```bash
cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 2
  selector:
    matchLabels: {app: my-app}
  template:
    metadata:
      labels: {app: my-app}
    spec:
      containers:
      - name: my-app
        image: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0
        ports:
        - containerPort: 80
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits:   {cpu: 500m, memory: 256Mi}
        livenessProbe:
          httpGet: {path: /health, port: 80}
          periodSeconds: 10
        readinessProbe:
          httpGet: {path: /ready, port: 80}
          periodSeconds: 5
EOF

kubectl apply -f deployment.yaml
kubectl rollout status deployment/my-app

# Rolling update to new image
kubectl set image deployment/my-app my-app=<ecr-uri>:v2.0
kubectl rollout status deployment/my-app

# Rollback
kubectl rollout undo deployment/my-app
```

### IRSA — IAM Roles for Service Accounts (Pods → AWS)

```bash
# Easiest way — eksctl
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace default \
  --name s3-reader \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Use in pod spec
# spec:
#   serviceAccountName: s3-reader
# Pod now has S3 read access via IRSA — zero hardcoded credentials
```

### EKS Cluster Upgrade (L3 Procedure)

```bash
# ALWAYS upgrade one minor version at a time: 1.28 → 1.29 → 1.30

# Step 1: Upgrade control plane
aws eks update-cluster-version \
  --name my-eks-cluster --kubernetes-version 1.30
aws eks wait cluster-active --name my-eks-cluster

# Step 2: Upgrade managed node group
aws eks update-nodegroup-version \
  --cluster-name my-eks-cluster --nodegroup-name workers

# Step 3: Update ALL add-ons
aws eks update-addon \
  --cluster-name my-eks-cluster \
  --addon-name vpc-cni --resolve-conflicts OVERWRITE
# Repeat for coredns, kube-proxy, aws-ebs-csi-driver
```

### ECS vs EKS — Decision Guide

| Factor | ECS | EKS |
|--------|-----|-----|
| Team k8s experience | Low / none | Medium to high |
| Setup time | Minutes (Fargate) | 20–60 min + add-ons |
| Ops overhead | Very low | Higher |
| Ecosystem | AWS-native | Full CNCF (Helm, Argo, Istio) |
| Multi-cloud portability | AWS-only | Portable |
| Service mesh | ECS Service Connect | Istio / Linkerd |

> **Rule of thumb:** Start with ECS Fargate. Migrate to EKS when you need Kubernetes-native features, have k8s-experienced teams, or need cross-cloud portability.

### CI/CD Pipeline: GitHub Actions → ECR → ECS

```yaml
# .github/workflows/deploy.yml
name: Deploy to ECS
on:
  push:
    branches: [main]
env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: my-app
  ECS_SERVICE: my-app-service
  ECS_CLUSTER: my-cluster
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
    - uses: actions/checkout@v4
    - uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::<account-id>:role/github-actions-role
        aws-region: ${{ env.AWS_REGION }}
    - id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2
    - id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
    - run: |
        aws ecs describe-task-definition --task-definition my-app-task \
          --query taskDefinition > task-definition.json
    - id: task-def
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      with:
        task-definition: task-definition.json
        container-name: my-app
        image: ${{ steps.build-image.outputs.image }}
    - uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: ${{ env.ECS_SERVICE }}
        cluster: ${{ env.ECS_CLUSTER }}
        wait-for-service-stability: true
```

### Container Security Checklist

- [ ] Images built with multi-stage (no build tools in production)
- [ ] Non-root user in Dockerfile (`USER appuser`)
- [ ] ECR tag immutability enabled on production repos
- [ ] ECR scan on push enabled; critical CVEs block deployment
- [ ] Lifecycle policy on every ECR repo
- [ ] ECS tasks: `readonlyRootFilesystem: true`
- [ ] ECS tasks: secrets via Secrets Manager `valueFrom`, never env literals
- [ ] EKS pods: `runAsNonRoot: true`, `allowPrivilegeEscalation: false`
- [ ] EKS: Network Policies deny-all, then explicit allow
- [ ] EKS: KMS encryption for secrets at rest
- [ ] EKS: PodDisruptionBudgets on all production Deployments
- [ ] VPC Endpoints for ECR (PrivateLink) on private clusters

---

## 7 · Cross-Domain Self-Practice Questions

> Work through these independently after each domain. Answers require console + CLI practice — not just reading.

---

### IAM Practice Questions

1. Create a custom IAM policy that allows EC2 `Describe*` actions but denies `TerminateInstances` only in `us-east-1`. Test it using the IAM Policy Simulator.
2. A developer says "I can list S3 buckets but can't read objects in `prod-data`." Walk through every layer of the access decision you'd check.
3. Set up MFA enforcement: write a policy that denies ALL actions when MFA is not present. Attach it to a user and verify that actions fail without MFA.
4. An EC2 instance needs to write to DynamoDB. Design the minimal IAM role (trust policy + permission policy). What happens if you attach the wrong trust policy?
5. A contractor needs read-only access for exactly 4 hours. Design the solution using AssumeRole with a session duration. Why not create a permanent IAM user?
6. You inherit an account. Run the Credentials Report and find 3 security gaps. Document what you'd fix and in what order.
7. Create a permissions boundary that caps what a developer can grant — they can create roles but only attach policies within a specific boundary.

---

### EC2 & Storage Practice Questions

8. Launch an EC2 instance via User Data that installs Docker, pulls `nginx:alpine` from ECR, and starts it on port 80. Access the page in your browser.
9. You have a 50 GB `gp2` volume. Migrate it to `gp3` with no downtime and no snapshot (hint: live volume modification). Verify the new type and performance settings.
10. A private subnet EC2 instance can't reach `yum` repositories. Trace every step from the instance's ENI to the internet. Where exactly is the break?
11. Set up a Spot Fleet using `priceCapacityOptimized` strategy across 3 instance types. Deploy a stateless web server and test that it survives a simulated interruption.
12. Using Data Lifecycle Manager, create a policy that takes daily snapshots of all volumes tagged `Backup=true` and retains the last 7. Verify a snapshot was created.
13. Encrypt an existing unencrypted EBS volume without terminating the instance. Document each step and why you need the intermediate snapshot.
14. Set up EFS and mount it on two EC2 instances in different AZs. Write a file from instance A and read it from instance B. Enable EFS IA after 30 days access.

---

### S3 Practice Questions

15. Create a bucket policy that allows `s3:GetObject` only if the request comes from a specific VPC endpoint. Test from inside and outside the VPC.
16. Enable Cross-Region Replication from Mumbai to Singapore. Push 5 objects. Verify they appear in the destination. Then enable delete marker replication and test deletion.
17. Generate a pre-signed PUT URL and use it to upload a file from a machine with zero AWS credentials. What does the upload inherit in terms of permissions?
18. Set up S3 Event Notifications to trigger a Lambda function on every `PUT`. The Lambda should log the object key and size to CloudWatch.
19. Enable S3 Object Lock in Governance mode on a bucket and upload a test file with a 1-day retention. Attempt to delete it as an admin. Then try with `s3:BypassGovernanceRetention`.
20. Build a multi-team bucket: Finance team can only access `finance/*` and Engineering can only access `code/*`. Implement using S3 Access Points.
21. Enable S3 server access logging. Wait for logs. Query them with Athena to find the top 5 most requested objects and any `403` errors.
22. Your bucket has 500 GB of `gp` class objects. Write a Lifecycle rule to move everything not touched in 60 days to Glacier, and delete anything older than 2 years.

---

### VPC Practice Questions

23. Build the full 3-tier VPC from scratch (public/app/db subnets in 2 AZs) using only CLI. Validate: bastion SSH works, private app has internet via NAT, DB has NO internet.
24. Create a VPC Peering connection between `prod-vpc (10.0.0.0/16)` and `dev-vpc (10.1.0.0/16)`. Why would peering fail if you chose `10.0.0.0/16` for both?
25. Enable VPC Flow Logs and intentionally trigger a Security Group REJECT (try SSH to a closed port). Find that REJECT entry in CloudWatch Insights using a Log Insights query.
26. A private EC2 instance is hitting `s3.amazonaws.com` via NAT Gateway — you can see the data charges. Add a Gateway Endpoint and verify the traffic no longer uses NAT (check flow logs).
27. Your app server can't reach the SSM Session Manager. You don't have a NAT Gateway. Create Interface Endpoints for `ssm`, `ssmmessages`, and `ec2messages`. Test with `aws ssm start-session`.
28. Design a NACL for a public subnet that blocks inbound from a specific IP range (simulate a malicious IP). Confirm Security Group still allows your IP, but the blocked CIDR is rejected at subnet level.
29. Set up a Transit Gateway with 3 VPCs. Verify any VPC can ping any other. Then create a Route Table in TGW that blocks dev-vpc from reaching db-vpc.
30. Run Reachability Analyzer to find why an EC2 in `priv-app-subnet-1a` can't reach the RDS in `priv-db-subnet-1a`. Document the exact blocking component it identifies.

---

### Route 53 Practice Questions

31. Register or delegate a domain to Route 53. Create A records for 3 EC2 instances. Verify with `dig` and `nslookup`. Observe TTL countdown in `dig` output.
32. Set up weighted routing: 70% to Mumbai EC2, 30% to Singapore EC2. Run 20 `curl` requests and count how many hit each. Adjust to 90/10 and repeat.
33. Create a failover setup: primary EC2 with a health check on `/health`, secondary pointing to an S3 static page. Stop the primary's web server. How long until Route 53 fails over?
34. Set up latency routing with 3 EC2 instances (Mumbai, Ireland, Virginia). Use a VPN to simulate connections from different geographies. Confirm each region returns the correct endpoint.
35. Set up geolocation routing so Indian users get a specific page, European users get another, and everyone else gets a default. Test with IP-faking tools or a VPN.
36. Create a Route 53 Private Hosted Zone for `internal.yourcompany.com`. Add A records for `db`, `cache`, and `api`. Confirm they resolve from inside the VPC but not from the internet.
37. Configure Route 53 Resolver inbound and outbound endpoints. Test that an EC2 in your VPC can resolve `server.corp.internal` (simulate on-prem with a custom DNS on another EC2).

---

### Container Practice Questions

38. Build a Node.js or Python app, write a multi-stage Dockerfile, push to ECR with tag immutability and scan on push. Find and document any CRITICAL CVEs in the scan results.
39. Deploy to ECS Fargate: create a task definition with a secret from Secrets Manager, a CloudWatch log group, 0.5 vCPU / 1 GB. Verify logs appear and the secret is accessible in the container.
40. Create an ECS service with 2 tasks behind an ALB. Update the image (push v2.0), force a rolling deploy, and watch the deployment in the ECS console. Enable circuit breaker and rollback.
41. Enable ECS Exec on a running service. Shell into a task. Inspect environment variables. Confirm the Secrets Manager secret is present as an env var without being stored in the task definition source.
42. Set up Fargate SPOT for 80% of your ECS service tasks. Configure the circuit breaker. Simulate a task failure by running a container that exits immediately. Confirm rollback occurs.
43. Create an ECR Pull Through Cache rule for Docker Hub. Pull `nginx:alpine` via your ECR pull-through endpoint. Confirm Docker Hub is NOT hit on second pull (cache works).
44. Build a multi-arch image (linux/amd64 + linux/arm64) using `docker buildx`. Push to ECR. Launch both a Graviton (ARM) and x86 Fargate task using the same image URI. Both should run successfully.
45. Create an EKS cluster using eksctl. Deploy your app as a Kubernetes Deployment with 3 replicas. Expose it via an ALB Ingress (requires AWS Load Balancer Controller). Access via the ALB DNS.
46. Set up IRSA: create an IAM role that allows `s3:ListBucket`. Create a ServiceAccount with that role. Run a Pod with that service account and verify `aws s3 ls` works from inside the pod with no credentials.
47. Install the Metrics Server. Create an HPA targeting 50% CPU. Use a load generator (`kubectl run load --image=busybox -- sh -c "while true; do wget -q -O- http://my-app-service; done"`) to trigger scale-out.
48. Write a PodDisruptionBudget requiring at minimum 2 pods available. Drain a node and observe that Kubernetes respects the PDB — it drains one pod, waits for replacement, then drains the next.
49. Create a Network Policy that denies all ingress to the `default` namespace. Then add an explicit allow from the `frontend` pod selector to the `backend` on port 8080. Test both allowed and blocked paths.
50. Perform an EKS minor version upgrade (e.g., 1.29 → 1.30): control plane → node group → all add-ons. Document what would break if you skipped a version.

---

### Cross-Domain Architecture Questions

These require combining multiple services. Design first (whiteboard or diagram), then build.

51. **Static Website + CDN + HTTPS:** Host a React app on S3, serve via CloudFront, use Route 53 for custom domain, use ACM for free HTTPS. Total cost target: < $1/month.

52. **Secure 3-Tier App:** Design and build a VPC with app servers in private subnets, RDS in isolated DB subnets, ALB in public subnets, secrets in Secrets Manager, and no direct internet access for any backend resource.

53. **Containerized Microservices:** Deploy a 2-service app (frontend + API) on ECS Fargate. Frontend calls API via internal DNS. Both store secrets via Secrets Manager. ALB routes `/` to frontend and `/api` to backend. Blue/Green deployment via CodeDeploy.

54. **GitOps EKS Platform:** Create an EKS cluster. Install ArgoCD. Point ArgoCD at a GitHub repo. Commit a Kubernetes manifest. Verify ArgoCD auto-deploys. Manually change a deployment in kubectl — confirm ArgoCD reverts it (selfHeal).

55. **Cost-Optimized Batch Pipeline:** Upload a CSV to S3. S3 event triggers an ECS Fargate Spot task. Task processes the CSV and writes results to another S3 bucket. Estimated cost per run < $0.01.

56. **Hybrid Connectivity:** Simulate on-premises with an EC2 in a separate VPC. Set up a Transit Gateway connecting 3 VPCs. Private instances in each VPC should resolve DNS names of the others via Route 53 Private Hosted Zones.

57. **Observability Stack:** Enable VPC Flow Logs, CloudTrail, ECS Container Insights, and EKS CloudWatch Logs. Create CloudWatch dashboards for: container CPU/memory, VPC rejected traffic, S3 error rates. Set alarms for all.

58. **DR (Disaster Recovery):** Configure S3 Cross-Region Replication, ECR cross-region replication, and Route 53 failover routing. Simulate a region failure by making the primary health check fail. Measure time-to-failover.

59. **Security Hardening Audit:** Given a VPC with known misconfigurations (public RDS, over-permissive SGs, no encryption, no MFA), use IAM Credentials Report, Trusted Advisor, and Network Access Analyzer to find and fix every issue.

60. **Zero-Downtime Deploy:** Deploy v1 of your app. Set up CodeDeploy Blue/Green for ECS with a 10% canary phase and a 5-minute bake period. Deploy v2. Break the `/health` endpoint in v2. Confirm CodeDeploy auto-rolls back to v1.

---

## Quick Reference — Critical CLI Commands

```bash
# ─── IAM ───────────────────────────────────────────────
aws sts get-caller-identity
aws iam list-users
aws iam generate-credential-report
aws iam get-credential-report --query Content --output text | base64 -d

# ─── EC2 ───────────────────────────────────────────────
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PrivateIpAddress]' --output table
aws ec2 describe-volumes --filters Name=status,Values=available   # unattached volumes
aws ec2 describe-addresses --query 'Addresses[?!AssociationId]'   # unattached EIPs

# ─── S3 ────────────────────────────────────────────────
aws s3 ls
aws s3 ls s3://bucket/ --recursive --human-readable
aws s3 presign s3://bucket/file --expires-in 300
aws s3api list-object-versions --bucket my-bucket

# ─── VPC ───────────────────────────────────────────────
aws ec2 describe-vpcs --query 'Vpcs[*].[VpcId,CidrBlock,Tags[?Key==`Name`].Value|[0]]' --output table
aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID --output table
aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID --output table

# ─── Route 53 ──────────────────────────────────────────
aws route53 list-hosted-zones
aws route53 list-resource-record-sets --hosted-zone-id Z1234
aws route53 get-health-check-status --health-check-id <id>

# ─── ECR ───────────────────────────────────────────────
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com
aws ecr describe-repositories
aws ecr list-images --repository-name my-app
aws ecr start-image-scan --repository-name my-app --image-id imageTag=latest
aws ecr describe-image-scan-findings --repository-name my-app --image-id imageTag=latest

# ─── ECS ───────────────────────────────────────────────
aws ecs list-clusters
aws ecs list-tasks --cluster my-cluster
aws ecs describe-tasks --cluster my-cluster --tasks $TASK_ARN
aws ecs update-service --cluster my-cluster --service my-service --force-new-deployment
aws logs tail /ecs/my-app --follow

# ─── EKS / kubectl ─────────────────────────────────────
kubectl get nodes -o wide
kubectl get pods --all-namespaces
kubectl rollout status deployment/my-app
kubectl rollout undo deployment/my-app
kubectl top nodes && kubectl top pods
kubectl get events --sort-by='.lastTimestamp'
kubectl exec -it <pod> -- /bin/sh
aws eks list-clusters
aws eks update-kubeconfig --name my-cluster --region ap-south-1
```

---

## Common Mistakes & Fixes (L3 Survival Guide)

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Root account has access keys | Security risk | Delete immediately, enable MFA |
| `latest` tag in production | Can't rollback | Tag with git SHA + semver; enable ECR tag immutability |
| ECS task fails to start | Service stuck at 0/2 | Check execution role has ECR pull + CloudWatch permissions |
| Private EC2 can't reach internet | NAT timeout | NAT GW in PUBLIC subnet? Route table has `0.0.0.0/0 → NAT GW`? |
| NACL connection timeout | TCP works one way | Missing ephemeral ports (1024–65535) on return direction |
| kubectl Unauthorized | EKS access denied | `aws eks update-kubeconfig` with correct IAM identity |
| EKS pods Pending | Never scheduled | Check node capacity, resource requests, and SG rules |
| S3 access denied | 403 on object | IAM policy + Bucket policy must BOTH allow; explicit deny anywhere wins |
| ECR auth expired | docker push fails | Token valid 12h — re-run `get-login-password` |
| Route 53 geolocation no response | NXDOMAIN for some users | Missing Default record — always create it |
| EKS upgrade failure | Add-on incompatibility | Check deprecated API versions; upgrade add-ons after control plane |
| No resource requests on k8s pods | Pods evicted first under pressure | Always set `requests` and `limits` |
| Secrets in env vars | Plaintext in task definition | Use Secrets Manager `valueFrom` or External Secrets Operator |
| Single AZ ECS service | AZ failure kills app | Spread tasks across AZs with `spread` placement |
| Interface Endpoint not working | Still using NAT | Enable private DNS; SG must allow 443 from VPC CIDR |
| Forgetting add-on update after EKS upgrade | Version mismatch | `aws eks update-addon` for EVERY add-on after control plane upgrade |

---

## 10-Week Learning Path

```
Week 1  → IAM: Users, Groups, Policies, Roles, MFA, CLI, Access Advisor
Week 2  → EC2: Instance types, SSH, User Data, Security Groups, IAM Roles
Week 3  → EC2 Storage: EBS types, mount, snapshot, AMI, EFS, Instance Store
Week 4  → EC2 Advanced: Purchasing options, Spot, Auto Scaling, Cost Optimizer
Week 5  → S3: Buckets, policies, versioning, lifecycle, encryption, pre-signed URLs
Week 6  → VPC Foundation: Subnets, IGW, NAT, Route Tables, SGs, NACLs
Week 7  → VPC Advanced: Peering, TGW, Endpoints, Flow Logs, Reachability Analyzer
Week 8  → Route 53: All routing policies, Health Checks, Hybrid DNS, TTL strategy
Week 9  → Containers Part 1: Docker, ECR, ECS Fargate, ALB, secrets, CI/CD
Week 10 → Containers Part 2: EKS, kubectl, IRSA, HPA, Network Policies, upgrades
```

---

*Built for hands-on L3 DevOps engineers. Practice in the console first, automate with CLI second, codify with Terraform third.*  
*Region: ap-south-1 (Mumbai) | All labs tested on AWS Free Tier + minimal paid resources*  
*Always clean up resources after practice to avoid unexpected charges.*
