# ☁️ AWS Console Hands-On Master Guide — DevOps L3 Engineer Reference

> **Audience:** L3 DevOps Engineers | **Approach:** Console-first, production-grade
> **Topics:** IAM · EC2 & Storage · S3 · VPC · Route 53 · Containers (ECR · ECS · EKS)
> **Region default:** `ap-south-1` (Mumbai) — swap to your nearest region throughout
> **Note:** Docker, kubectl, eksctl, and dig/nslookup are terminal-only tools with no console equivalent — those sections are kept as-is with a terminal label.

---

## 📋 Table of Contents

| # | Domain | Key Topics |
|---|--------|-----------|
| 1 | [IAM — Identity & Access Management](#1--iam--identity--access-management) | Users, Groups, Policies, Roles, MFA |
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
Step 1 — Create Group
  IAM → User groups → Create group
    Group name: Admins
    Attach permissions policy → search: AdministratorAccess → check it
  → Create user group

Step 2 — Create User
  IAM → Users → Create user
    User name: admin-yourname
    ✅ Provide user access to the AWS Management Console
    Select: I want to create an IAM user
    Console password: Custom password (set a strong one)
    ✅ Users must create a new password at next sign-in
  → Next → Add user to group: Admins → Next → Create user
  → Download .csv (contains sign-in URL and credentials)

Step 3 — Enable MFA on the new user
  IAM → Users → admin-yourname → Security credentials
  → Multi-factor authentication (MFA) → Assign MFA device
    Device name: admin-mfa
    MFA device: Authenticator app → Next
    Scan the QR code with Google Authenticator / Authy
    Enter two consecutive codes → Add MFA
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

### Custom Policy — Console Hands-On

```
Step 1 — Create the policy
  IAM → Policies → Create policy
    Select: JSON tab → paste the JSON above (replace as needed)
  → Next → Policy name: EC2-ReadOnly-NoTerminate → Create policy

Step 2 — Attach to a Group
  IAM → User groups → Developers → Permissions tab
  → Add permissions → Attach policies
    Search: EC2-ReadOnly-NoTerminate → check it
  → Attach policies

Step 3 — Test with Policy Simulator
  IAM → Policy Simulator (top-right "Tools" menu or direct link)
  → Select users/roles → select the user
  → Select service: EC2 → select actions → Run Simulation
  → Review Allow / Deny results
```

### IAM Roles — The Right Way to Give Services Access

```
Step 1 — Create the Role
  IAM → Roles → Create role
    Trusted entity type: AWS service
    Use case: EC2 → Next
    Search policy: AmazonS3ReadOnlyAccess → check it → Next
    Role name: EC2-S3-ReadOnly-Role → Create role

Step 2 — Attach Role to a Running EC2 Instance
  EC2 → Instances → select instance
  → Actions → Security → Modify IAM role
  → Select IAM role: EC2-S3-ReadOnly-Role → Update IAM role

Step 3 — Verify from inside EC2
  Connect to the instance (EC2 Instance Connect or SSH)
  Run: aws s3 ls     ← works without any credentials configured
```

### Access Key Management (Console)

```
Create Access Key:
  IAM → Users → select user → Security credentials
  → Access keys → Create access key
    Use case: CLI / Application running outside AWS
  → Download .csv → store securely

Deactivate / Delete Access Key:
  IAM → Users → select user → Security credentials → Access keys
  → Click: Make inactive (deactivate) or Delete

View Credential Report (Account-wide audit):
  IAM → Credential report → Download Report (CSV)
  Review for: stale keys, missing MFA, last used dates

View Access Advisor (per-user):
  IAM → Users → select user → Access Advisor tab
  → Review service last-accessed dates → remove unused permissions
```

### Access Key Rotation (Zero-Downtime Pattern)

```
Step 1: Create Key 2  → IAM → Users → user → Security credentials → Create access key
Step 2: Update all apps/scripts to use Key 2
Step 3: Verify Key 2 works everywhere
Step 4: Deactivate Key 1  → Actions: Make inactive
Step 5: Wait 24–48 hours
Step 6: Delete Key 1  → Actions: Delete
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
  Name: my-web-server
  AMI: Amazon Linux 2023 (Free tier eligible)
  Instance Type: t3.micro
  Key pair: Create new key pair → Name: my-key → RSA → .pem → Create
  Network settings:
    VPC: default (or your VPC)
    Security group: Create new
      Rule 1: SSH — Port 22 — Source: My IP
      Rule 2: HTTP — Port 80 — Source: 0.0.0.0/0
  Advanced details → User data (paste below):
```

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd && systemctl enable httpd
echo "<h1>$(hostname -f) — $(date)</h1>" > /var/www/html/index.html
```

```
→ Launch instance
→ Wait for Instance State: Running, then open Public IP in browser
```

> **Connect via browser (no key needed):**
> EC2 → Instances → select instance → Connect → EC2 Instance Connect → Connect

### Networking: IP Types

| Type | Persists on Stop? | Internet? | Cost |
|------|-------------------|-----------|------|
| Private IP | ✅ Yes | VPC-only | Free |
| Public IP | ❌ Changes | Yes | Free while running |
| Elastic IP | ✅ Yes (static) | Yes | ~$0.005/hr when unattached |

> **L3 Tip:** Avoid Elastic IPs. Use DNS (Route 53) or a Load Balancer instead.

### SSH & EC2 Instance Connect

```
Browser-based (no key needed):
  EC2 → Instances → select instance → Connect → EC2 Instance Connect → Connect

Key-based SSH (Linux/Mac terminal):
  chmod 400 my-key.pem
  ssh -i my-key.pem ec2-user@<PUBLIC-IP>

Via bastion (terminal):
  ssh -i my-key.pem -o "ProxyJump ec2-user@<BASTION-IP>" ec2-user@<PRIVATE-IP>
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

### Attach & Mount an EBS Volume (Console)

```
Step 1 — Create Volume in same AZ as instance
  EC2 → Elastic Block Store → Volumes → Create volume
    Volume type: gp3
    Size: 20 GiB
    Availability Zone: ap-south-1a  ← must match instance AZ
  → Create volume

Step 2 — Attach to Instance
  Select the new volume → Actions → Attach volume
  → Instance: select your instance
  → Device name: /dev/sdf  → Attach volume

Step 3 — Mount inside the instance (via EC2 Instance Connect terminal)
  lsblk                          # confirm device (e.g. xvdf)
  sudo mkfs -t ext4 /dev/xvdf    # format (only first time!)
  sudo mkdir /data
  sudo mount /dev/xvdf /data     # mount
  df -h                          # confirm

Step 4 — Persist across reboots
  echo "/dev/xvdf /data ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
```

### EBS Snapshots — Console (Cross-AZ & Cross-Region Migration)

```
Create Snapshot:
  EC2 → Elastic Block Store → Volumes → select volume
  → Actions → Create snapshot
    Description: Before deployment → Create snapshot

Copy Snapshot to Another Region:
  EC2 → Elastic Block Store → Snapshots → select snapshot
  → Actions → Copy snapshot
    Destination Region: us-east-1 → Copy snapshot

Create Volume from Snapshot in a Different AZ:
  Snapshots → select snapshot → Actions → Create volume from snapshot
    Availability Zone: ap-south-1b
    Volume type: gp3 → Create volume

Automate with Data Lifecycle Manager:
  EC2 → Elastic Block Store → Lifecycle Manager → Create lifecycle policy
    Policy type: EBS snapshot policy
    Target: Instances or Volumes with specific tags
    Schedule: Daily at 02:00 UTC → Retain: 7 snapshots
  → Create policy
```

### AMI — Build a Custom Image (Console)

```
Create AMI:
  EC2 → Instances → select running instance
  → Actions → Image and templates → Create image
    Image name: my-nginx-app-v1.0
    ✅ No reboot (avoid downtime)
  → Create image  (takes 5–15 min, check EC2 → AMIs)

Launch New Instance from your AMI:
  EC2 → Launch instance → My AMIs (left sidebar)
  → Select your AMI → configure and launch as normal
```

> **L3 Use:** Bake-in agents, config, security hardening, app binaries → faster ASG scale-out, consistent environments.

### EFS — Shared File System (Console)

```
Step 1 — Create EFS File System
  EFS → Create file system
    Name: my-efs
    VPC: select your VPC
  → Customize (optional: set lifecycle policy)
  → Create  (note the DNS name: fs-xxxx.efs.ap-south-1.amazonaws.com)

Step 2 — Allow NFS access in Security Group
  EC2 → Security Groups → your app SG
  → Inbound rules → Edit inbound rules
    Add rule: NFS (port 2049) — Source: your app SG (self-referencing)
  → Save rules

Step 3 — Mount on EC2 instances (EC2 Instance Connect terminal)
  sudo yum install -y amazon-efs-utils
  sudo mkdir /efs
  sudo mount -t efs -o tls <EFS-DNS-NAME>:/ /efs

Step 4 — Test shared access
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

### Spot Instance (Console)

```
EC2 → Spot Requests → Request Spot Instances
  Request type: Request (one-time) or Persistent
  Launch template: create or select
  Target capacity: 2 instances
  Instance types: c6i.large, c5.large, m5.large  ← multiple types = fewer interruptions
  Allocation strategy: Price capacity optimized
  Max price: leave blank (use on-demand price as cap)
→ Launch

Monitor interruptions:
  EC2 → Instances → look for "Spot" type
  Instance state → interruption notices appear 2 min before termination
```

### Cost Optimization — Console

```
Find Idle / Over-provisioned Instances (Compute Optimizer):
  AWS Console → Compute Optimizer → EC2 instances
  → Filter: Over-provisioned → review right-sizing recommendations

Find Unattached EBS Volumes (cost leak):
  EC2 → Elastic Block Store → Volumes
  → Filter: State = available  ← these are billable orphans, delete them

Find Unattached Elastic IPs:
  EC2 → Network & Security → Elastic IPs
  → Unassociated IPs show "–" in Instance column → Actions → Release
```

**Cost Optimization Checklist:**
- [ ] Unattached EBS volumes deleted
- [ ] Old snapshots on lifecycle policy (Data Lifecycle Manager)
- [ ] Over-provisioned instances right-sized (Compute Optimizer)
- [ ] Spot or Savings Plans purchased for steady workloads
- [ ] EFS Lifecycle Management enabled (auto-move to IA after 30 days)
- [ ] Budget alert configured (Billing → Budgets → Create budget)

---

## 3 · S3 — Object Storage

### Core Concepts

- **Bucket:** Globally unique name, region-scoped, flat key-value store
- **Object:** Key + Value + Metadata + Version ID; max size 5 TB (multipart upload required >5 GB)
- **URL format:** `https://<bucket>.s3.<region>.amazonaws.com/<key>`
- **Folders are simulated** via `/` in key names — S3 is flat

### Create Bucket & Upload

```
S3 → Create bucket
  Bucket name: my-app-assets-2024  (globally unique)
  AWS Region: ap-south-1
  Block Public Access: ✅ Block all (keep on by default)
→ Create bucket

Upload files:
  S3 → Buckets → my-app-assets-2024 → Upload
  → Add files or Add folder → Upload
```

### Bucket Policy — Public Read (for static sites)

```
Step 1 — Disable Block Public Access
  S3 → Buckets → your-bucket → Permissions tab
  → Block public access → Edit → uncheck all → Save changes
  Type "confirm" → Confirm

Step 2 — Add Bucket Policy
  Permissions tab → Bucket policy → Edit → paste:
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

```
→ Save changes
```

### Enforce Encryption at Upload (Deny Unencrypted PUT)

Add this statement to your bucket policy:

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
S3 → Buckets → your-bucket → Properties tab
→ Static website hosting → Edit → Enable
  Index document: index.html
  Error document: error.html
→ Save changes

Website URL:
  Properties → Static website hosting → note the endpoint URL
  Format: http://<bucket>.s3-website-<region>.amazonaws.com

Note: HTTP only. For HTTPS, put CloudFront in front.
```

### Versioning

```
Enable Versioning (Console):
  S3 → Buckets → your-bucket → Properties
  → Bucket Versioning → Edit → Enable → Save changes

List Versions:
  S3 → Buckets → your-bucket → Objects tab
  → Toggle "Show versions" (top right) — all versions appear with Version ID

Delete Specific Version (permanent):
  Show versions ON → select the specific version → Delete
  → type "permanently delete" → Delete objects

Restore (undelete) an object:
  Show versions ON → find the "Delete marker" for your object
  → select the delete marker → Delete (removes the marker, restores the object)
```

### Lifecycle Rules — Auto-Tier & Expire

```
S3 → Buckets → your-bucket → Management tab
→ Lifecycle rules → Create lifecycle rule
  Rule name: archive-old-logs
  Rule scope: Limit to specific prefix → logs/
  Lifecycle rule actions:
    ✅ Transition current versions between storage classes
       After 30 days → Standard-IA
       After 90 days → Glacier Flexible Retrieval
    ✅ Expire current versions after 365 days
    ✅ Delete incomplete multipart uploads after 7 days  ← always add this
→ Create rule
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

```
Prerequisites: Enable versioning on BOTH source and destination buckets first.

Console: Source Bucket → Management tab → Replication rules → Create replication rule
  Rule name: mumbai-to-singapore
  Status: Enabled
  Source bucket scope: All objects (or prefix filter)
  Destination: choose bucket in this or another account
    → ap-southeast-1 bucket for CRR
  IAM role: Create new role (auto-created)
  Additional options:
    ✅ Replicate delete markers (optional)
→ Save

Replicate existing objects (not auto-replicated when rule is first created):
  S3 → Batch Operations → Create job
    Manifest: S3 Inventory report or CSV
    Operation: Replication → Run job
```

### Encryption

| Method | Key Owner | Notes |
|--------|----------|-------|
| SSE-S3 | AWS | Default since Jan 2023, AES-256 |
| SSE-KMS | You (KMS) | Audit trail in CloudTrail; key rotation |
| SSE-C | You entirely | Pass key with every request; AWS never stores it |
| Client-Side | You | Encrypted before upload |

```
Set Default Encryption:
  S3 → Buckets → your-bucket → Properties
  → Default encryption → Edit
    Encryption type: SSE-KMS
    AWS KMS key: choose key or use AWS managed key
  → Save changes

Upload with specific encryption (per object):
  S3 → Upload → Properties (expand) → Server-side encryption
    Select: SSE-KMS → choose your KMS key
```

### Pre-Signed URLs (Console — Time-Limited Access)

```
Generate Pre-Signed Download URL:
  S3 → Buckets → your-bucket → click on the object
  → Object actions → Share with a presigned URL
    Duration: 5 minutes (or custom)
  → Create presigned URL → Copy the URL

Note: For pre-signed PUT URLs (upload), you must use the AWS CLI or SDK.
```

### S3 Object Lock (WORM Compliance)

```
Enable at bucket creation:
  S3 → Create bucket → Advanced settings
  → Object Lock: Enable → Acknowledge → Create bucket

Apply retention to object:
  S3 → object → Object actions → Edit object lock retention
    Retention mode: Governance (admins can override) or Compliance (no one can delete)
    Retain until date: set date → Save changes

Legal Hold (independent lock):
  Object → Object actions → Edit object lock legal hold: Enable

Use case: SEC 17a-4, HIPAA, PCI-DSS immutable records
```

### VPC Endpoint for S3 (Free Private Access)

```
VPC → Endpoints → Create endpoint
  Service category: AWS services
  Service name: search "s3" → select com.amazonaws.ap-south-1.s3
  Endpoint type: Gateway  ← FREE (not Interface)
  VPC: select your VPC
  Route tables: select private route tables (app, DB subnets)
→ Create endpoint

Now private EC2 instances can access S3 without going through NAT Gateway.
Check: VPC → Route Tables → private RT → Routes → should see a new S3 prefix route.
```

### S3 Best Practices Checklist

- [ ] Versioning enabled on production buckets
- [ ] Default encryption set (SSE-S3 or SSE-KMS)
- [ ] Lifecycle rules to expire logs and old versions
- [ ] Block Public Access at account level
- [ ] Access logging enabled (to a separate bucket)
- [ ] MFA Delete on critical buckets (requires CLI with root credentials — cannot be done via console)
- [ ] Incomplete multipart upload cleanup rule (7 days)
- [ ] S3 Storage Lens dashboard enabled: S3 → Storage Lens → Create dashboard

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

### Create VPC + Subnets + IGW + NAT (Console Step-by-Step)

```
━━━ Step 1: Create VPC ━━━
VPC → Your VPCs → Create VPC
  Resource to create: VPC only
  Name tag: prod-vpc
  IPv4 CIDR: 10.0.0.0/16
→ Create VPC

Enable DNS hostnames (required for many services):
  VPC → Your VPCs → select prod-vpc → Actions → Edit VPC settings
  ✅ Enable DNS hostnames → Save

━━━ Step 2: Create Subnets ━━━
VPC → Subnets → Create subnet
  VPC: prod-vpc
  Add subnet (repeat for each):
    Name: pub-subnet-1a  |  AZ: ap-south-1a  |  CIDR: 10.0.1.0/24
    Name: pub-subnet-1b  |  AZ: ap-south-1b  |  CIDR: 10.0.2.0/24
    Name: app-subnet-1a  |  AZ: ap-south-1a  |  CIDR: 10.0.3.0/24
    Name: app-subnet-1b  |  AZ: ap-south-1b  |  CIDR: 10.0.4.0/24
    Name: db-subnet-1a   |  AZ: ap-south-1a  |  CIDR: 10.0.5.0/24
    Name: db-subnet-1b   |  AZ: ap-south-1b  |  CIDR: 10.0.6.0/24
→ Create subnet

Enable auto-assign public IP on public subnets:
  Subnets → select pub-subnet-1a → Actions → Edit subnet settings
  ✅ Enable auto-assign public IPv4 address → Save
  Repeat for pub-subnet-1b

━━━ Step 3: Create Internet Gateway ━━━
VPC → Internet gateways → Create internet gateway
  Name tag: prod-igw → Create internet gateway
→ Actions → Attach to VPC → select prod-vpc → Attach internet gateway

━━━ Step 4: Create NAT Gateway (in public subnet) ━━━
VPC → NAT gateways → Create NAT gateway
  Name: prod-nat-gw
  Subnet: pub-subnet-1a   ← MUST be a public subnet
  Connectivity type: Public
  Elastic IP: click "Allocate Elastic IP"
→ Create NAT gateway  (takes ~2 minutes to become Available)

━━━ Step 5: Create Route Tables ━━━
A) Public Route Table:
  VPC → Route tables → Create route table
    Name: pub-rt  |  VPC: prod-vpc → Create

  Add internet route:
    pub-rt → Routes tab → Edit routes → Add route
      Destination: 0.0.0.0/0  |  Target: Internet Gateway → prod-igw
    → Save changes

  Associate public subnets:
    pub-rt → Subnet associations → Edit subnet associations
    ✅ pub-subnet-1a  ✅ pub-subnet-1b → Save

B) Private (App) Route Table:
  Create route table: Name: priv-app-rt  |  VPC: prod-vpc → Create
  Add route: 0.0.0.0/0 → NAT Gateway → prod-nat-gw → Save
  Associate: app-subnet-1a, app-subnet-1b

C) DB Route Table (NO internet route):
  Create route table: Name: priv-db-rt  |  VPC: prod-vpc → Create
  ← DO NOT add any 0.0.0.0/0 route (DB must have no internet access)
  Associate: db-subnet-1a, db-subnet-1b
```

### Security Groups vs NACLs

| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance (ENI) | Subnet |
| Stateful? | ✅ Yes | ❌ No (both directions needed) |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules | Numbered order (lowest first) |
| Default | Deny all in | Allow all |

### Security Group — Layered Architecture (Console)

```
━━━ Bastion SG — only your IP can SSH ━━━
EC2 → Security Groups → Create security group
  Name: bastion-sg  |  VPC: prod-vpc
  Inbound rules → Add rule:
    Type: SSH  |  Port: 22  |  Source: My IP  (auto-fills your IP)
→ Create security group

━━━ ALB SG — public web traffic ━━━
Create security group: alb-sg  |  VPC: prod-vpc
Inbound rules:
  HTTP    | 80  | Source: 0.0.0.0/0
  HTTPS   | 443 | Source: 0.0.0.0/0
→ Create

━━━ App SG — only ALB on 8080, only bastion on 22 ━━━
Create security group: app-sg  |  VPC: prod-vpc
Inbound rules:
  Custom TCP | 8080 | Source: alb-sg (select Security Group as source type)
  SSH        | 22   | Source: bastion-sg
→ Create

━━━ DB SG — only app on 3306 ━━━
Create security group: db-sg  |  VPC: prod-vpc
Inbound rules:
  MYSQL/Aurora | 3306 | Source: app-sg
→ Create
```

### NACLs — Stateless (L3 Must-Know)

```
VPC → Network ACLs → Create network ACL
  Name: public-nacl  |  VPC: prod-vpc → Create

Inbound rules → Edit inbound rules:
  Rule 100: TCP | 80       | 0.0.0.0/0 | Allow    ← HTTP
  Rule 110: TCP | 443      | 0.0.0.0/0 | Allow    ← HTTPS
  Rule 120: TCP | 22       | YOUR-IP/32 | Allow   ← SSH
  Rule 130: TCP | 1024-65535 | 0.0.0.0/0 | Allow ← Ephemeral return ports (CRITICAL)
  Rule *:   All traffic    | 0.0.0.0/0 | Deny

Outbound rules → Edit outbound rules:
  Rule 100: TCP | 80       | 0.0.0.0/0 | Allow
  Rule 110: TCP | 443      | 0.0.0.0/0 | Allow
  Rule 120: TCP | 1024-65535 | 0.0.0.0/0 | Allow  ← Response ports
  Rule *:   All traffic    | 0.0.0.0/0 | Deny

Associate with public subnets:
  NACL → Subnet associations → Edit → select pub-subnet-1a, pub-subnet-1b → Save
```

> **Common mistake:** Forgetting ephemeral ports (1024–65535) in NACLs causes mysterious timeouts. Security Groups are stateful — NACLs are not.

### VPC Flow Logs (Console)

```
Send to CloudWatch Logs:
  VPC → Your VPCs → select prod-vpc → Flow logs tab → Create flow log
    Filter: All (accepted + rejected)
    Maximum aggregation interval: 1 minute
    Destination: Send to CloudWatch Logs
    Destination log group: /aws/vpc/flowlogs/prod-vpc
    IAM role: Create new role (auto-prompt) or select existing
  → Create flow log

Send to S3 (cheaper for long-term):
  Same steps → Destination: Send to an Amazon S3 bucket
  → S3 bucket ARN: arn:aws:s3:::my-flow-logs-bucket

Analyze in CloudWatch Logs Insights:
  CloudWatch → Logs Insights → select /aws/vpc/flowlogs/prod-vpc
  Sample query (SSH brute-force):
    fields @timestamp, srcaddr, action
    | filter dstport = 22 and action = "REJECT"
    | stats count(*) by srcaddr
    | sort desc | limit 20
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

### VPC Peering (Console)

```
Step 1 — Create Peering Connection
  VPC → Peering connections → Create peering connection
    Name: prod-dev-peer
    VPC Requester: prod-vpc (10.0.0.0/16)
    Account: My account (or another account ID)
    VPC Accepter: dev-vpc (10.1.0.0/16)
  → Create peering connection

Step 2 — Accept the Request
  Peering connections → select the pending connection
  → Actions → Accept request → Accept

Step 3 — Update Route Tables (both VPCs)
  prod-vpc route table → Routes → Edit routes → Add route:
    Destination: 10.1.0.0/16 → Target: Peering Connection → prod-dev-peer

  dev-vpc route table → Routes → Edit routes → Add route:
    Destination: 10.0.0.0/16 → Target: Peering Connection → prod-dev-peer
```

### Transit Gateway (Console)

```
Step 1 — Create TGW
  VPC → Transit gateways → Create transit gateway
    Name: corp-tgw
    ASN: 64512 (default)
    ✅ DNS support, ✅ VPN ECMP support
  → Create transit gateway (takes ~5 min)

Step 2 — Attach VPCs
  VPC → Transit gateway attachments → Create transit gateway attachment
    TGW: corp-tgw
    Attachment type: VPC
    VPC: prod-vpc → select subnets (one per AZ)
  → Create  (repeat for dev-vpc, staging-vpc)

Step 3 — Update Route Tables in each VPC
  Each VPC's private route table → Add route:
    Destination: 10.0.0.0/8 (covers all internal CIDRs)
    Target: Transit Gateway → corp-tgw
```

### VPC Endpoints — Console (Save NAT Gateway Costs)

```
Gateway Endpoint for S3 (FREE):
  VPC → Endpoints → Create endpoint
    Service category: AWS services
    Service: com.amazonaws.ap-south-1.s3 → Gateway type
    VPC: prod-vpc
    Route tables: select all private route tables
  → Create endpoint

Interface Endpoint for SSM (removes need for NAT or bastion):
  VPC → Endpoints → Create endpoint
    Service: com.amazonaws.ap-south-1.ssm → Interface type
    VPC: prod-vpc
    Subnets: select private app subnets
    Security group: allow port 443 from VPC CIDR
    ✅ Enable private DNS name
  → Create endpoint
  Repeat for: ssmmessages and ec2messages
```

### Route 53 Private Hosted Zone (Internal DNS)

```
Route 53 → Hosted zones → Create hosted zone
  Domain name: internal.yourcompany.com
  Type: ✅ Private hosted zone
  Region: ap-south-1
  VPC ID: select your prod-vpc
→ Create hosted zone

Add internal records:
  Hosted zone → Create record
    Record name: db    |  Type: A  |  Value: 10.0.5.10
    Record name: cache |  Type: A  |  Value: 10.0.3.20
    Record name: api   |  Type: A  |  Value: 10.0.3.30
→ Create records

Now use db.internal.yourcompany.com instead of hardcoded IPs.
When DB moves — update DNS, zero app changes.
```

### Connectivity Validation Tests

| Test | Expected | If failing — check |
|------|----------|-------------------|
| Bastion → internet | ✅ Works | Public subnet has IGW route |
| Private app → internet | ✅ Works (via NAT) | Private RT has NAT GW route |
| Internet → private directly | ❌ Fail | No public IP, SG blocks |
| DB → internet | ❌ Fail | No route in DB route table |
| App → S3 via endpoint | ✅ Works (no NAT cost) | Endpoint route in RT |

### Reachability Analyzer (Console — No Traffic Sent)

```
VPC → Network Manager → Reachability Analyzer → Create and analyze path
  Source type: Instance | Source: bastion-instance
  Destination type: Instance | Destination: private-app-instance
  Protocol: TCP | Destination port: 22
→ Create and analyze path  (takes ~1 min)

→ If "Not reachable" — click the path → expand the blocking component
  It will show exactly which SG rule or route table entry blocks the traffic
```

### VPC Cleanup Order (Avoid Billing Traps)

```
Always in this order — dependencies prevent reverse order:
1.  EC2 → Instances → Terminate all instances
2.  VPC → NAT Gateways → Delete NAT GW → wait Available → Deleted
3.  EC2 → Elastic IPs → Release address
4.  VPC → Transit Gateway Attachments → Delete → then Transit Gateway → Delete
5.  VPC → VPN Connections → Delete → detach VGW → delete VGW
6.  VPC → Endpoints → Delete
7.  VPC → Peering Connections → Delete
8.  VPC → Internet Gateways → Detach from VPC → Delete
9.  VPC → Route Tables → Delete (custom only; main RT auto-deletes with VPC)
10. VPC → Security Groups → Delete (custom only)
11. VPC → Subnets → Delete all subnets
12. VPC → Your VPCs → Delete VPC
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

> **Rule:** Anything pointing to an AWS resource → use Alias. Root domain must use Alias.

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

### Hands-On: Latency-Based Routing (3 Regions) — Console

```
Route 53 → Hosted zones → select your zone → Create record

Record 1 — Mumbai:
  Record name: app.yourco.com
  Record type: A
  Value: <mumbai-EC2-public-IP>
  TTL: 60
  Routing policy: Latency
  Region: ap-south-1
  Record ID: latency-mumbai
→ Create records

Repeat → Create record (Record 2 — Ireland):
  Same record name: app.yourco.com
  Routing policy: Latency | Region: eu-west-1
  Value: <ireland-EC2-public-IP>
  Record ID: latency-ireland

Repeat → Create record (Record 3 — Virginia):
  Routing policy: Latency | Region: us-east-1
  Value: <virginia-EC2-public-IP>
  Record ID: latency-virginia

Route 53 will automatically return the IP with lowest latency to each client.
```

### Health Checks (Console)

```
Route 53 → Health checks → Create health check
  Name: mumbai-app-health
  What to monitor: Endpoint
  Protocol: HTTP
  IP address: 13.234.56.78
  Port: 80
  Path: /health
  Request interval: 30 seconds
  Failure threshold: 3
→ Create health check

Monitor status:
  Health checks → select check → Monitoring tab
  → Status shows: Healthy / Unhealthy with CloudWatch metrics

Get notified on failure:
  Health check → Create alarm
  → SNS topic: send email alert when status = unhealthy
```

### Failover Routing (Active-Passive) — Console

```
Step 1 — Create Health Check for primary (see above)

Step 2 — Create Primary Record
  Route 53 → Hosted zones → Create record
    Record name: app.yourco.com  |  Type: A
    Value: <mumbai-EC2-IP>
    Routing policy: Failover
    Failover record type: Primary
    Health check: mumbai-app-health  ← REQUIRED on primary
    Record ID: failover-primary
  → Create records

Step 3 — Create Secondary Record
  Create record (same name and type)
    Value: <S3-static-website-endpoint> (or S3 alias)
    Routing policy: Failover
    Failover record type: Secondary
    Record ID: failover-secondary
    (No health check needed on secondary)
  → Create records

Result: When health check fails → Route 53 automatically serves secondary.
        When primary recovers → traffic returns to primary.
```

### TTL Strategy for Production Changes

```
Normal operation:    TTL = 3600   ← balance of cost and flexibility
48h before IP change: TTL = 60  ← reduce early so caches drain
After IP change:     TTL = 3600  ← restore once propagated
```

### Hybrid DNS — Route 53 Resolver (Console)

```
VPC → DNS Firewall / Route 53 Resolver → Inbound endpoints → Create inbound endpoint
  Name: corp-inbound-endpoint
  VPC: prod-vpc
  Security group: allow UDP/TCP 53 from on-prem IP range
  IP addresses: select 2 subnets (one per AZ) → auto-assign IPs
→ Create (gives 2 IPs — configure on-prem DNS forwarder to send AWS queries here)

Create Outbound Endpoint:
  Resolver → Outbound endpoints → Create outbound endpoint
    Name: corp-outbound-endpoint
    VPC: prod-vpc
    Security group: allow outbound UDP/TCP 53
    IP addresses: select 2 subnets → auto-assign IPs
  → Create

Create Forwarding Rule (AWS → on-prem DNS):
  Resolver → Rules → Create rule
    Name: forward-to-onprem
    Rule type: Forward
    Domain name: internal.corp
    Target IP addresses: 192.168.1.53 port 53  ← on-prem DNS server
    VPCs to associate: prod-vpc
  → Save
```

### DNS Troubleshooting (Terminal — no console equivalent)

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

### ECR — Image Registry (Console)

```
Create Repository:
  ECR → Repositories → Create repository
    Visibility: Private
    Repository name: my-app
    Tag immutability: Enabled  ← Production safety
    Scan on push: Enabled  ← auto-vulnerability scan
  → Create repository

Note the URI: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app

View Scan Results:
  ECR → Repositories → my-app → Images
  → select image → click Vulnerabilities
  → Review CRITICAL / HIGH findings

Set Lifecycle Policy:
  ECR → Repositories → my-app → Lifecycle policy → Create rule
    Rule priority: 1
    Rule description: Remove untagged images after 7 days
    Image status: Untagged
    Match criteria: Since image pushed — 7 days
    Action: Expire
  → Save (add second rule: keep only 10 tagged v* images)
```

> **Note:** Docker build, tag, push, and pull are terminal operations. ECR console shows repos and scan results; image management requires Docker CLI from your machine or CI/CD pipeline.

```bash
# ── TERMINAL: Authenticate & Push (required — no console equivalent) ──
aws ecr get-login-password --region ap-south-1 \
  | docker login --username AWS --password-stdin \
    <account-id>.dkr.ecr.ap-south-1.amazonaws.com

docker build -t my-app .
docker tag my-app:latest <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0
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
# Multi-arch for Graviton (ARM) + x86 cost savings — terminal only
docker buildx create --name multiarch --use
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag <ecr-uri>/my-app:v1.0 \
  --push .
```

### ECS — Fargate (Console Step-by-Step)

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

```
━━━ Step 1: Create ECS Cluster ━━━
ECS → Clusters → Create cluster
  Cluster name: my-cluster
  Infrastructure: AWS Fargate  ← serverless
→ Create cluster

━━━ Step 2: Create CloudWatch Log Group ━━━
CloudWatch → Log groups → Create log group
  Log group name: /ecs/my-app
  Retention: 30 days
→ Create

━━━ Step 3: Create Task Definition ━━━
ECS → Task definitions → Create new task definition
  Task definition family: my-app-task
  Launch type: AWS Fargate
  OS/Architecture: Linux/X86_64
  CPU: 0.5 vCPU  |  Memory: 1 GB
  Task role: (your app-level AWS role, if app calls AWS APIs)
  Task execution role: ecsTaskExecutionRole  ← allows ECR pull + CloudWatch

Container — Add container:
  Name: my-app
  Image URI: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0
  Port mappings: Container port 80 → TCP
  Log collection: ✅ Use log collection
    Log driver: awslogs
    awslogs-group: /ecs/my-app
    awslogs-region: ap-south-1
    awslogs-stream-prefix: ecs
→ Create (creates revision 1)

━━━ Step 4: Create Secrets (optional) ━━━
Secrets Manager → Store a new secret
  Secret type: Other type of secret
  Key: DB_PASSWORD | Value: mypassword123
  Secret name: /myapp/prod/db-password
→ Store (note the Secret ARN)

Reference in task definition:
  Container → Environment variables → Add:
    Key: DB_PASSWORD
    Value type: ValueFrom
    Value: <secret-arn>
  (Task execution role must have secretsmanager:GetSecretValue permission)
```

### ECS Service + ALB (Console)

```
━━━ Step 5: Create Target Group ━━━
EC2 → Load Balancers → Target groups → Create target group
  Target type: IP addresses  ← required for Fargate
  Target group name: my-app-tg
  Protocol: HTTP | Port: 80
  VPC: prod-vpc
  Health check path: /health
→ Create target group

━━━ Step 6: Create Application Load Balancer ━━━
EC2 → Load Balancers → Create load balancer → Application Load Balancer
  Name: my-app-alb
  Scheme: Internet-facing
  IP type: IPv4
  VPC: prod-vpc
  Subnets: ✅ pub-subnet-1a  ✅ pub-subnet-1b
  Security group: alb-sg (ports 80 + 443)
  Listener: HTTP:80 → Forward to my-app-tg
→ Create load balancer

━━━ Step 7: Create ECS Service ━━━
ECS → Clusters → my-cluster → Services → Create
  Launch type: FARGATE
  Task definition: my-app-task (latest revision)
  Service name: my-app-service
  Desired tasks: 2
  Deployment type: Rolling update
  ✅ Enable deployment circuit breaker  |  ✅ Enable rollback
  VPC: prod-vpc
  Subnets: app-subnet-1a, app-subnet-1b
  Security group: app-sg
  ✅ Load balancing: Application Load Balancer
    Load balancer: my-app-alb
    Listener: 80:HTTP
    Target group: my-app-tg
→ Create service

Verify deployment:
  ECS → Clusters → my-cluster → Services → my-app-service → Deployments tab
  → Wait for Running count = 2
  → Open ALB DNS in browser to verify the app
```

### ECS Operations (Console + Terminal)

```
Force Redeploy (new image, same tag):
  ECS → Clusters → my-cluster → Services → my-app-service → Update service
  ✅ Force new deployment → Update

View Live Logs:
  ECS → Clusters → my-cluster → Services → my-app-service → Logs tab
  (also available in CloudWatch → Log groups → /ecs/my-app)

View Task Details:
  ECS → Clusters → my-cluster → Tasks → select running task
  → Containers tab → view port, IPs, environment, health status

Shell into Running Container (ECS Exec — requires enabling first):
  ECS → Services → my-app-service → Update service
  ✅ Enable Execute Command → Update service → Force new deployment → Update

  Then from terminal:
  TASK_ARN=$(aws ecs list-tasks --cluster my-cluster \
    --service-name my-app-service --query "taskArns[0]" --output text)
  aws ecs execute-command \
    --cluster my-cluster --task $TASK_ARN \
    --container my-app --interactive --command "/bin/sh"
```

### ECS Auto Scaling (Console)

```
ECS → Clusters → my-cluster → Services → my-app-service → Update service

Service auto scaling → Use Service Auto Scaling
  Minimum tasks: 1
  Desired tasks: 2
  Maximum tasks: 10

Scaling policies → Add scaling policy:
  Policy type: Target tracking
  Policy name: cpu-tracking
  ECS service metric: ECSServiceAverageCPUUtilization
  Target value: 50%
  Scale-out cooldown: 60 seconds
  Scale-in cooldown: 60 seconds
→ Update service
```

### ECS Deployment Strategies

| Strategy | How it works | Use case |
|----------|-------------|---------|
| **Rolling update** | Replaces tasks gradually (default) | Most workloads |
| **Blue/Green (CodeDeploy)** | Runs new alongside old; shifts traffic when healthy | Zero-downtime |
| **Circuit breaker** | Auto-rollback if new tasks fail to start | Safety net (always enable) |

### EKS — Kubernetes on AWS

> **Note:** EKS cluster creation, kubectl, and eksctl are terminal operations. The EKS console is used to monitor nodes, pods, workloads, and view add-on status.

```bash
# ── TERMINAL: Create cluster (takes 15–20 min) ──
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

```
Console — Monitor EKS:
  EKS → Clusters → my-eks-cluster
    → Overview: cluster status, Kubernetes version
    → Compute tab: node groups, node status
    → Workloads tab: Deployments, DaemonSets, Pods
    → Configuration → Add-ons: view/update vpc-cni, coredns, kube-proxy

Console — View Pod logs:
  EKS → Clusters → my-eks-cluster → Workloads → Pods → select pod → Logs
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

### Deploy App to EKS (Terminal)

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
# ── TERMINAL ──
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

### EKS Cluster Upgrade — Console + Terminal

```
Console — Check current version:
  EKS → Clusters → my-eks-cluster → Overview → Kubernetes version

Console — Upgrade control plane:
  EKS → Clusters → my-eks-cluster → Update Kubernetes version
  → Select: 1.30 (upgrade ONE minor version at a time: 1.28 → 1.29 → 1.30)
  → Update  (takes 10–20 min)

Console — Upgrade node group:
  EKS → Clusters → my-eks-cluster → Compute → Node groups → workers
  → Update now → Rolling update → Confirm

Console — Update Add-ons:
  EKS → Clusters → my-eks-cluster → Configuration → Add-ons
  → For each add-on (vpc-cni, coredns, kube-proxy, aws-ebs-csi-driver):
    Select → Update → Resolve conflicts: Overwrite → Update
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

### CI/CD Pipeline: GitHub Actions → ECR → ECS (Terminal/YAML)

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

> Work through these independently after each domain. Answers require console practice — not just reading.

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
9. You have a 50 GB `gp2` volume. Migrate it to `gp3` with no downtime and no snapshot (console: Volumes → Modify Volume → change type to gp3). Verify the new type and performance settings.
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

23. Build the full 3-tier VPC from scratch (public/app/db subnets in 2 AZs) using only the console. Validate: bastion SSH works, private app has internet via NAT, DB has NO internet.
24. Create a VPC Peering connection between `prod-vpc (10.0.0.0/16)` and `dev-vpc (10.1.0.0/16)`. Why would peering fail if you chose `10.0.0.0/16` for both?
25. Enable VPC Flow Logs and intentionally trigger a Security Group REJECT (try SSH to a closed port). Find that REJECT entry in CloudWatch Insights using a Log Insights query.
26. A private EC2 instance is hitting `s3.amazonaws.com` via NAT Gateway — you can see the data charges. Add a Gateway Endpoint and verify the traffic no longer uses NAT (check flow logs).
27. Your app server can't reach the SSM Session Manager. You don't have a NAT Gateway. Create Interface Endpoints for `ssm`, `ssmmessages`, and `ec2messages`. Test with AWS Systems Manager → Session Manager → Start session.
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

## Console Navigation Quick Reference

| Task | Console Path |
|------|-------------|
| Create IAM User | IAM → Users → Create user |
| Create IAM Role (for EC2) | IAM → Roles → Create role → EC2 |
| Policy Simulator | IAM → top menu "Tools" → Policy Simulator |
| Credentials Report | IAM → Credential report → Download |
| Access Advisor | IAM → Users → [user] → Access Advisor tab |
| Launch EC2 | EC2 → Instances → Launch instance |
| Create EBS Volume | EC2 → Elastic Block Store → Volumes → Create |
| Create Snapshot | EC2 → Volumes → [vol] → Actions → Create snapshot |
| Create AMI | EC2 → Instances → [i] → Actions → Image and templates → Create image |
| Data Lifecycle Manager | EC2 → Elastic Block Store → Lifecycle Manager |
| Create S3 Bucket | S3 → Create bucket |
| S3 Bucket Policy | S3 → [bucket] → Permissions → Bucket policy → Edit |
| S3 Lifecycle Rules | S3 → [bucket] → Management → Lifecycle rules |
| Create VPC | VPC → Your VPCs → Create VPC |
| Create Subnets | VPC → Subnets → Create subnet |
| Create IGW | VPC → Internet gateways → Create |
| Create NAT GW | VPC → NAT gateways → Create |
| Create Route Table | VPC → Route tables → Create |
| VPC Endpoints | VPC → Endpoints → Create endpoint |
| VPC Flow Logs | VPC → Your VPCs → [vpc] → Flow logs → Create flow log |
| Reachability Analyzer | VPC → Network Manager → Reachability Analyzer |
| Create Security Group | EC2 → Security Groups → Create security group |
| Create NACL | VPC → Network ACLs → Create network ACL |
| Route 53 Hosted Zone | Route 53 → Hosted zones → Create hosted zone |
| Route 53 Record | Route 53 → Hosted zones → [zone] → Create record |
| Route 53 Health Check | Route 53 → Health checks → Create health check |
| Route 53 Resolver | VPC → DNS Firewall / Route 53 Resolver |
| Create ECR Repo | ECR → Repositories → Create repository |
| ECR Lifecycle Policy | ECR → [repo] → Lifecycle policy → Create rule |
| Create ECS Cluster | ECS → Clusters → Create cluster |
| Create Task Definition | ECS → Task definitions → Create new |
| Create ECS Service | ECS → Clusters → [cluster] → Services → Create |
| ECS Force Redeploy | ECS → Services → [service] → Update service → ✅ Force new deployment |
| EKS Create Cluster | (eksctl in terminal — no full console wizard) |
| EKS View Workloads | EKS → Clusters → [cluster] → Workloads |
| EKS Update Add-ons | EKS → Clusters → [cluster] → Configuration → Add-ons |
| Compute Optimizer | AWS Console → Compute Optimizer |
| CloudWatch Logs Insights | CloudWatch → Logs Insights |
| Secrets Manager | Secrets Manager → Store a new secret |
| Budget Alerts | Billing → Budgets → Create budget |

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
| Forgetting add-on update after EKS upgrade | Version mismatch | Update every add-on via EKS Console → Add-ons after control plane upgrade |

---

## 10-Week Learning Path

```
Week 1  → IAM: Users, Groups, Policies, Roles, MFA (Console only)
Week 2  → EC2: Instance types, SSH, User Data, Security Groups, IAM Roles
Week 3  → EC2 Storage: EBS types, attach/mount, snapshot, AMI, EFS
Week 4  → EC2 Advanced: Purchasing options, Spot, Auto Scaling, Cost Optimizer
Week 5  → S3: Buckets, policies, versioning, lifecycle, encryption, pre-signed URLs
Week 6  → VPC Foundation: Subnets, IGW, NAT, Route Tables, SGs, NACLs
Week 7  → VPC Advanced: Peering, TGW, Endpoints, Flow Logs, Reachability Analyzer
Week 8  → Route 53: All routing policies, Health Checks, Hybrid DNS, TTL strategy
Week 9  → Containers Part 1: Docker (terminal), ECR, ECS Fargate, ALB, secrets, CI/CD
Week 10 → Containers Part 2: EKS, kubectl, IRSA, HPA, Network Policies, upgrades
```

---

*Built for hands-on L3 DevOps engineers. Practice in the console first, automate with CLI second, codify with Terraform third.*
*Region: ap-south-1 (Mumbai) | All labs tested on AWS Free Tier + minimal paid resources*
*Always clean up resources after practice to avoid unexpected charges.*
