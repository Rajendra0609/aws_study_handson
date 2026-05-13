# ☁️ AWS EC2 & Storage — Hands-On Learning Guide

> A structured, practical guide to mastering AWS EC2, Networking, and Storage — organized from **Beginner → Advanced**.

---

## 📋 Table of Contents

1. [How to Use This Guide](#how-to-use-this-guide)
2. [Phase 1 — Foundations (Beginner)](#phase-1--foundations-beginner)
3. [Phase 2 — Networking & Security (Intermediate)](#phase-2--networking--security-intermediate)
4. [Phase 3 — Storage Deep Dive (Intermediate–Advanced)](#phase-3--storage-deep-dive-intermediateadvanced)
5. [Phase 4 — Advanced Compute & Cost Optimization](#phase-4--advanced-compute--cost-optimization)
   - 4.1 EC2 Purchasing Options
   - 4.2 Spot Instances & Spot Fleet
   - **4.3 Cost Optimization Strategies** *(newly added)*
6. [Quick Reference Cheat Sheet](#quick-reference-cheat-sheet)
7. [Common Gotchas & Pro Tips](#common-gotchas--pro-tips)

---

## How to Use This Guide

- Follow the **phases in order** — each phase builds on the previous.
- Every topic has a **Concept Summary**, **Console Steps**, and **Hands-On Exercise**.
- Topics marked 🖥️ have an official AWS Console hands-on component.
- Complete the **Checkpoint** at the end of each phase before moving on.

> **Prerequisites:** An AWS account with billing alerts set up (covered in Phase 1).

---

## Phase 1 — Foundations (Beginner)

### 1.1 AWS Budget Setup

**Why it matters:** Prevents surprise bills while you experiment.

**Console Steps:**
1. Go to **AWS Console → Billing & Cost Management → Budgets**
2. Click **Create Budget** → Choose **Cost Budget**
3. Set a monthly amount (e.g., $10)
4. Add your email under **Alert thresholds** at 80% and 100%
5. Click **Create Budget**

> 💡 **Tip:** Also enable **Free Tier usage alerts** under Billing Preferences.

---

### 1.2 EC2 Basics

**Concept Summary:**
EC2 (Elastic Compute Cloud) provides resizable virtual servers (instances) in the cloud. Key attributes:
- **AMI** — The OS image used to launch an instance
- **Instance Type** — Defines CPU, RAM, and network capacity
- **Key Pair** — SSH credentials to access your instance
- **Security Group** — Virtual firewall controlling inbound/outbound traffic

**EC2 Instance Type Families:**

| Family | Use Case | Example |
|--------|----------|---------|
| `t3`, `t4g` | General purpose / burstable | Web servers, dev/test |
| `m6i`, `m7g` | Balanced compute & memory | App servers |
| `c6i`, `c7g` | Compute optimized | Batch, HPC, gaming |
| `r6i`, `r7g` | Memory optimized | Databases, caches |
| `i3`, `i4i` | Storage optimized | NoSQL, data warehousing |
| `p3`, `g4dn` | GPU / Accelerated | ML, video rendering |

---

### 1.3 🖥️ Create an EC2 Instance with User Data (Hands-On)

**Goal:** Launch a web server automatically on boot using EC2 User Data.

**Console Steps:**
1. Go to **EC2 → Instances → Launch Instances**
2. **Name:** `my-web-server`
3. **AMI:** Amazon Linux 2023 (Free Tier eligible)
4. **Instance Type:** `t2.micro` (Free Tier)
5. **Key Pair:** Create new → name it `my-ec2-key` → Download `.pem` file
6. **Network Settings:** Create a Security Group:
   - Allow **SSH (port 22)** from My IP
   - Allow **HTTP (port 80)** from Anywhere (0.0.0.0/0)
7. Expand **Advanced Details → User Data** and paste:

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2 — $(hostname -f)</h1>" > /var/www/html/index.html
```

8. Click **Launch Instance**
9. Copy the **Public IPv4 address** and open it in your browser

✅ **Expected Result:** A webpage displaying your instance hostname.

**Exercise:** Modify the User Data to display today's date dynamically using `$(date)`.

---

### 1.4 Security Groups & Classic Ports

**Concept Summary:**
Security Groups are **stateful** firewalls attached to EC2 instances.

| Port | Protocol | Use Case |
|------|----------|----------|
| 22 | SSH | Linux remote access |
| 3389 | RDP | Windows remote access |
| 80 | HTTP | Web traffic |
| 443 | HTTPS | Secure web traffic |
| 3306 | MySQL/Aurora | Database |
| 5432 | PostgreSQL | Database |

**Key Rules:**
- Security Groups only have **Allow** rules (no explicit deny)
- They are **stateful** — if you allow inbound, the response is automatically allowed outbound
- One instance can have **multiple** Security Groups
- Security Groups can **reference other Security Groups** (great for inter-service communication)

**🖥️ Hands-On — Security Groups:**
1. Go to **EC2 → Security Groups → Create Security Group**
2. Add inbound rules for ports 22 and 80
3. Attach it to your running EC2 instance via **Actions → Security → Change Security Groups**
4. Try removing port 80 and confirm the website becomes unreachable

---

### 1.5 SSH Access

**Concept Summary:** SSH lets you securely connect to your Linux EC2 instance from your terminal.

**Linux/Mac:**
```bash
chmod 400 my-ec2-key.pem
ssh -i my-ec2-key.pem ec2-user@<PUBLIC-IP>
```

**Windows (PuTTY):**
1. Convert `.pem` to `.ppk` using **PuTTYgen**
2. Open PuTTY → enter `ec2-user@<PUBLIC-IP>`
3. Under **SSH → Auth**, browse to your `.ppk` file
4. Click **Open**

**Windows 10+ (Native SSH):**
```powershell
icacls my-ec2-key.pem /inheritance:r /grant:r "%username%:R"
ssh -i my-ec2-key.pem ec2-user@<PUBLIC-IP>
```

**EC2 Instance Connect (Browser-based):**
1. Select your instance → Click **Connect**
2. Choose **EC2 Instance Connect** tab
3. Click **Connect** — no `.pem` file needed!

> 💡 **Tip:** EC2 Instance Connect works only if port 22 is open and you're using Amazon Linux or Ubuntu.

**SSH Troubleshooting Checklist:**
- [ ] Port 22 open in Security Group?
- [ ] Using the correct username? (`ec2-user` for Amazon Linux, `ubuntu` for Ubuntu)
- [ ] `.pem` file has correct permissions (`chmod 400`)?
- [ ] Connecting to the **Public** IP, not Private?
- [ ] Instance is in **running** state?

---

### 1.6 EC2 Instance Roles Demo

**Concept Summary:**
Never hardcode AWS credentials on EC2. Use **IAM Roles** instead.

**🖥️ Hands-On:**
1. Go to **IAM → Roles → Create Role**
2. Trusted entity: **EC2**
3. Attach policy: `AmazonS3ReadOnlyAccess`
4. Name it: `EC2-S3-ReadOnly-Role`
5. Go to **EC2 → Actions → Security → Modify IAM Role**
6. Attach `EC2-S3-ReadOnly-Role`
7. SSH into your instance and run:

```bash
aws s3 ls   # Works without any credentials configured!
```

> 💡 **Pro Tip:** The instance fetches temporary credentials automatically from the **Instance Metadata Service (IMDS)** at `http://169.254.169.254`.

---

### Phase 1 Checkpoint ✅

Before moving on, confirm you can:
- [ ] Launch an EC2 instance with a User Data script
- [ ] Connect via SSH and EC2 Instance Connect
- [ ] Create and modify Security Groups
- [ ] Attach an IAM Role to an EC2 instance
- [ ] Set up a billing alert

---

## Phase 2 — Networking & Security (Intermediate)

### 2.1 Private vs Public vs Elastic IP

**Concept Summary:**

| Type | Scope | Persists on stop/start? | Cost |
|------|-------|--------------------------|------|
| **Private IP** | VPC-internal only | ✅ Yes | Free |
| **Public IP** | Internet-facing | ❌ Changes on restart | Free (while running) |
| **Elastic IP** | Internet-facing | ✅ Yes (static) | Free when attached; ~$0.005/hr when idle |

**Key Rules:**
- Every EC2 instance always gets a **Private IP**
- **Public IPs** are lost when you stop an instance
- **Elastic IPs** are static public IPs you allocate and attach
- You can only have **5 Elastic IPs** per region by default (soft limit)
- Best practice: Avoid Elastic IPs — use **DNS names** or **Load Balancers** instead

**🖥️ Hands-On:**
1. Note your instance's current Public IP
2. Stop and restart the instance — the Public IP changes!
3. Go to **EC2 → Elastic IPs → Allocate Elastic IP address**
4. Click **Associate Elastic IP** → select your instance
5. Stop/start again — the IP stays the same
6. **Clean up:** Disassociate and release the Elastic IP to avoid charges

---

### 2.2 EC2 Placement Groups

**Concept Summary:**
Placement Groups control **where** AWS physically places your EC2 instances.

| Strategy | Layout | Best For | Trade-off |
|----------|--------|----------|-----------|
| **Cluster** | Same rack, same AZ | Low-latency HPC, Big Data | High failure risk |
| **Spread** | Different racks, up to 7 per AZ | Critical instances (max isolation) | Max 7 instances/AZ |
| **Partition** | Groups on separate partitions | Hadoop, Kafka, Cassandra | Up to 7 partitions/AZ |

**🖥️ Hands-On:**
1. Go to **EC2 → Placement Groups → Create Placement Group**
2. Create one of each type and observe the options
3. When launching an EC2 instance, under **Advanced Details → Placement Group**, select your group
4. Try launching multiple instances into a **Cluster** group and ping between them

> 💡 **Tip:** You cannot merge placement groups or move a running instance into one — plan ahead!

---

### 2.3 Elastic Network Interfaces (ENI)

**Concept Summary:**
An ENI is a **virtual network card** for your EC2 instance. Each instance has at least one (eth0).

**ENI Attributes:**
- Primary private IPv4 address
- One or more secondary private IPv4 addresses
- One Elastic IP per private IPv4
- One public IPv4
- One or more Security Groups
- A MAC address

**Use Cases:**
- **Failover:** Detach ENI from a failing instance and attach to a standby instance — the private IP and MAC address move with it
- **Dual-homed instances:** Attach a second ENI in a different subnet for management traffic
- **License management:** Software licensed by MAC address

**🖥️ Hands-On:**
1. Go to **EC2 → Network Interfaces → Create Network Interface**
2. Select a subnet and Security Group
3. Attach it to a running instance via **Actions → Attach**
4. SSH in and run `ip addr` — you'll see `eth1`
5. Detach and re-attach to a different instance — the private IP follows!

> 💡 **Tip:** ENIs are AZ-specific. They can only be attached to instances in the same AZ.

---

### 2.3.1 ENI — Extra Reading

**How ENIs Work Under the Hood:**
When you launch an EC2 instance, AWS automatically creates a **primary ENI** (`eth0`) and attaches it. This ENI holds:
- The **primary private IPv4** (from the subnet CIDR)
- Optionally, **secondary private IPs** (useful for hosting multiple websites/SSL certs on one instance)
- The instance's **MAC address** — this is permanent and moves with the ENI

**Secondary ENI Use Cases in Detail:**

| Scenario | How ENI Helps |
|----------|--------------|
| **High Availability Failover** | Pre-create a "failover ENI" with a fixed private IP. Detach from a failed instance, attach to standby — clients reconnect to the same IP instantly |
| **Management Network** | Attach a second ENI in a private management subnet; keep app traffic on `eth0`, admin/monitoring on `eth1` |
| **MAC-Licensed Software** | Software tied to a MAC address? Move the ENI → move the license |
| **Pod Networking (EKS)** | AWS CNI plugin allocates secondary IPs on ENIs for Kubernetes pods |

**ENI Limits Per Instance:**
The number of ENIs you can attach depends on the instance type. Examples:

| Instance Type | Max ENIs | Max Private IPs per ENI |
|---------------|----------|------------------------|
| `t3.micro` | 2 | 2 |
| `m5.large` | 3 | 10 |
| `c5.4xlarge` | 8 | 30 |

> 📖 Full limits: [AWS ENI limits per instance type](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI)

**ENI vs Elastic IP vs Secondary Private IP:**
- **Secondary Private IP** — stays within the VPC, no extra cost, useful for internal failover
- **Elastic IP** — public static IP, attached to an ENI's private IP, costs money when unattached
- **ENI** — the container that holds all of the above; you move the whole ENI for full failover

**Best Practices:**
- Keep ENIs in the same AZ as your instance — cross-AZ attachment is not supported
- Use **Elastic Fabric Adapter (EFA)** instead of standard ENI for HPC/ML workloads needing ultra-low latency
- Tag your ENIs clearly — they persist independently of instances and can become orphaned

---

### 2.4 EC2 Hibernate

**Concept Summary:**
Hibernate saves the **in-memory (RAM) state** to the EBS root volume, so when you start again, the OS resumes instantly — processes, connections, and all.

**vs Stop vs Terminate:**
| Action | RAM | EBS Root | Startup Time |
|--------|-----|----------|--------------|
| Stop | Lost | Preserved | Full boot |
| Hibernate | Saved to EBS | Preserved | Near-instant |
| Terminate | Lost | Deleted (default) | N/A |

**Requirements:**
- Instance must be EBS-backed
- Root EBS volume must be **encrypted**
- RAM must be < 150 GB
- Supported families: C, M, R, and more
- Max hibernate duration: **60 days**

**🖥️ Hands-On:**
1. Launch an instance with an **encrypted** root EBS volume
2. Enable **Stop - Hibernate behavior** under Advanced Details
3. SSH in, start a long-running process (e.g., `sleep 1000 &`)
4. Hibernate the instance via **Instance State → Hibernate**
5. Start it again — your process is still running! Verify with `ps aux | grep sleep`

---

### Phase 2 Checkpoint ✅

- [ ] Can you explain when to use Elastic IP vs DNS?
- [ ] Can you create all 3 types of Placement Groups?
- [ ] Can you detach an ENI and attach it to another instance?
- [ ] Can you launch and resume a Hibernated instance?

---

## Phase 3 — Storage Deep Dive (Intermediate–Advanced)

### 3.1 EBS Overview

**Concept Summary:**
EBS (Elastic Block Store) is a **network-attached block storage** for EC2. Think of it as a USB drive you can plug into your server.

**Key Properties:**
- Locked to a **single AZ** (must snapshot to move)
- Can be detached and re-attached to instances
- Billed for **provisioned** capacity, even if unused
- **Delete on Termination** is enabled for root volumes by default

---

### 3.2 EBS Volume Types

| Type | Category | Use Case | Max IOPS | Max Throughput |
|------|----------|----------|----------|----------------|
| `gp3` | SSD | General purpose (default) | 16,000 | 1,000 MB/s |
| `gp2` | SSD | General purpose (legacy) | 16,000 | 250 MB/s |
| `io1/io2` | SSD | High-performance DBs | 64,000 / 256,000 | 1,000 MB/s |
| `st1` | HDD | Big Data, logs, streaming | 500 | 500 MB/s |
| `sc1` | HDD | Cold data, archives | 250 | 250 MB/s |

> 💡 **Rules:** Only `gp2`, `gp3`, `io1`, `io2` can be used as **boot volumes**. HDD types cannot.

**🖥️ Hands-On:**
1. Go to **EC2 → Volumes → Create Volume**
2. Create a `gp3` volume (e.g., 10 GiB) in the same AZ as your instance
3. Attach it via **Actions → Attach Volume**
4. SSH in and run:

```bash
lsblk                         # See the new disk
sudo mkfs -t ext4 /dev/xvdf   # Format it
sudo mkdir /data
sudo mount /dev/xvdf /data    # Mount it
df -h                         # Confirm
```

5. Write a test file and detach/reattach to confirm data persists

---

### 3.3 EBS Snapshots

**Concept Summary:**
Snapshots are **incremental backups** of EBS volumes stored in S3. Only changed blocks are saved after the first full backup.

**Key Features:**
- **EBS Snapshot Archive:** 75% cheaper, but 24–72 hrs to restore
- **Recycle Bin:** Protect snapshots from accidental deletion (retention 1 day–1 year)
- **Fast Snapshot Restore (FSR):** No latency on first use (costs extra)
- Snapshots can be **copied across regions**

**🖥️ Hands-On:**
1. Select a volume → **Actions → Create Snapshot**
2. Add a description tag
3. Go to **Snapshots → Actions → Copy Snapshot** → change region to copy cross-region
4. Create a new volume from a snapshot:
   - **Snapshots → Actions → Create Volume from Snapshot**
   - Change AZ to migrate data across availability zones!
5. Enable the **Recycle Bin:** Go to **EC2 → Recycle Bin → Create retention rule**

---

### 3.4 AMI Overview & Hands-On

**Concept Summary:**
AMI (Amazon Machine Image) is a **blueprint** for launching EC2 instances. It includes: OS, application server, applications, and permissions.

**Types:**
- **AWS-provided:** Amazon Linux, Ubuntu, Windows
- **AWS Marketplace:** Pre-configured third-party AMIs
- **Custom AMIs:** You build and own them

**Why create a Custom AMI?**
- Pre-install software, agents, config → faster boot time
- Consistency across environments
- Bake in compliance/security settings

**🖥️ Hands-On — Create a Custom AMI:**
1. Launch an instance and customize it (install Nginx, configure settings)
2. **Instance → Actions → Image and Templates → Create Image**
3. Name: `my-custom-nginx-ami`
4. Enable **No reboot** if you don't want downtime
5. Go to **AMIs** and wait for status: `available`
6. Launch a **new instance** from your custom AMI — it comes pre-configured!
7. Share it across accounts: **Actions → Edit AMI Permissions**

---

### 3.5 EC2 Instance Store

**Concept Summary:**
Instance Store is **physically attached** disk storage — not a network drive. It's the fastest possible storage for EC2.

| | EBS | Instance Store |
|---|-----|----------------|
| Persistence | ✅ Survives stop/start | ❌ Lost on stop/terminate |
| Speed | Up to 256,000 IOPS | Millions of IOPS |
| Availability | All instance types | Selected types only (e.g., `i3`) |
| Use case | General storage | Temp buffers, caches, scratch data |

> ⚠️ **Warning:** Instance Store is **ephemeral**. Never use it for data you can't afford to lose. Replicate it if persistence matters.

**Instance types with Instance Store:** `i3`, `i3en`, `d2`, `h1`, `c5d`, `m5d`

---

### 3.6 EBS Multi-Attach

**Concept Summary:**
`io1`/`io2` volumes can be attached to **up to 16 EC2 instances** in the **same AZ** simultaneously.

**Requirements:**
- Only for `io1`/`io2` volume types
- Must use a **cluster-aware file system** (e.g., GFS2, OCFS2) — NOT ext4 or XFS
- All instances must be in the **same AZ**

**Use Cases:** High-availability clustered applications (e.g., Teradata, SAP HANA)

---

### 3.7 EBS Encryption

**Concept Summary:**
EBS encryption uses **AWS KMS (AES-256)** to encrypt data at rest and in transit between EBS and EC2.

**What gets encrypted:**
- Data at rest on the volume
- All snapshots of the volume
- All volumes created from encrypted snapshots
- Data in-flight between EC2 and EBS

**Encrypt an Existing Unencrypted Volume:**
1. Create a **Snapshot** of the unencrypted volume
2. **Copy the Snapshot** → enable encryption during copy
3. Create a new volume from the **encrypted snapshot**
4. Attach the new encrypted volume to your instance

> 💡 **Tip:** Enable **default EBS encryption** account-wide under EC2 Settings to encrypt all future volumes automatically.

---

### 3.8 Amazon EFS (Elastic File System)

**Concept Summary:**
EFS is a **managed NFS (Network File System)** that can be mounted by **many EC2 instances at once** — even across AZs.

**EFS vs EBS:**

| Feature | EBS | EFS |
|---------|-----|-----|
| Type | Block storage | File storage (NFS) |
| Access | 1 instance (Multi-Attach: up to 16) | Thousands of instances |
| AZ scope | Single AZ | Multi-AZ |
| OS | Any | **Linux only** |
| Pricing | Provisioned capacity | Pay per GB used |
| Size | Fixed provisioned | Automatically scales |

**EFS Storage Classes:**
- **Standard** — Frequently accessed files
- **EFS-IA (Infrequent Access)** — Up to 92% cheaper, small retrieval fee

**🖥️ Hands-On:**
1. Go to **EFS → Create File System**
2. Choose your VPC and enable **Multi-AZ**
3. Launch **2 EC2 instances** in different AZs
4. On both instances, install the EFS mount helper:

```bash
sudo yum install -y amazon-efs-utils
sudo mkdir /efs
sudo mount -t efs -o tls <EFS-DNS-NAME>:/ /efs
```

5. Create a file on instance 1:

```bash
echo "Hello from Instance 1" | sudo tee /efs/test.txt
```

6. Read it from instance 2:

```bash
cat /efs/test.txt   # Shared filesystem in action!
```

---

### 3.9 EFS vs EBS — Decision Guide

```
Need shared storage across instances?
    └── YES → EFS
    └── NO → Do you need max IOPS?
                 └── YES + temporary → Instance Store
                 └── YES + persistent → io1/io2 EBS
                 └── NO → gp3 EBS (default choice)
```

---

### 3.10 🖥️ Section Cleanup

> ⚠️ Always clean up to avoid charges!

**Checklist:**
- [ ] Terminate EC2 instances
- [ ] Delete unattached EBS volumes (`Volume State: available`)
- [ ] Delete EBS snapshots
- [ ] Deregister custom AMIs (EC2 → AMIs → Deregister)
- [ ] Delete EFS file systems
- [ ] Release unattached Elastic IPs
- [ ] Delete custom Security Groups (default cannot be deleted)
- [ ] Delete custom Placement Groups

---

### Phase 3 Checkpoint ✅

- [ ] Can you create, attach, format, and mount an EBS volume?
- [ ] Can you create a snapshot and use it to move data across AZs?
- [ ] Can you build a custom AMI and launch from it?
- [ ] Can you mount an EFS share on two instances simultaneously?
- [ ] Can you encrypt an existing unencrypted EBS volume?

---

## Phase 4 — Advanced Compute & Cost Optimization

### 4.1 EC2 Purchasing Options

| Option | Best For | Savings vs On-Demand | Commitment |
|--------|----------|----------------------|------------|
| **On-Demand** | Short-term, unpredictable workloads | Baseline | None |
| **Reserved (1yr/3yr)** | Steady-state apps (DBs, web servers) | Up to 72% | 1 or 3 years |
| **Savings Plans** | Flexible reserved pricing | Up to 72% | 1 or 3 years |
| **Spot Instances** | Fault-tolerant, flexible workloads | Up to 90% | None (can be interrupted) |
| **Dedicated Hosts** | Compliance, BYOL licensing | — | On-demand or reserved |
| **Dedicated Instances** | Isolation at hardware level | — | None |
| **Capacity Reservations** | Ensure capacity in specific AZ | None | None |

**Reserved Instance Types:**
- **Standard RI:** Most savings, cannot change instance family
- **Convertible RI:** Less savings, can change instance type/OS/tenancy

---

### 4.2 Spot Instances & Spot Fleet

**Concept Summary:**
Spot Instances use **spare AWS capacity** at up to 90% discount. AWS can **reclaim** them with a 2-minute warning.

**Spot Request Types:**
- **One-time:** Instance terminates after interruption
- **Persistent:** Request is re-fulfilled after interruption

**Spot Fleet = Spot Instances + (optional) On-Demand**
A Spot Fleet tries to maintain a target capacity using a mix of instance types and AZs.

**Fleet Allocation Strategies:**
| Strategy | Description |
|----------|-------------|
| `lowestPrice` | Launch from cheapest pool |
| `diversified` | Spread across all pools |
| `capacityOptimized` | Launch from most available pool |
| `priceCapacityOptimized` | Best of price + availability (recommended) |

**🖥️ Hands-On — Spot Instance:**
1. Go to **EC2 → Launch Instance → Advanced Details**
2. Set **Purchasing option** to **Spot Instance**
3. Set a **max price** (or leave as on-demand price)
4. Launch and monitor under **Spot Requests**

**Spot Interruption Handler:**
```bash
# Poll metadata to detect 2-minute warning
TOKEN=$(curl -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" \
  http://169.254.169.254/latest/api/token)

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/spot/termination-time
# Returns a timestamp if interruption is imminent
```

> 💡 **Best Practice:** Use Spot for stateless, containerized, or batch workloads. Pair with Auto Scaling Groups for resilience.

---

### 4.3 Cost Optimization Strategies

**Concept Summary:**
AWS cost optimization is about matching the right resource to the right workload at the right price. The main levers are: **purchase model**, **right-sizing**, **auto-scaling**, and **storage tiering**.

---

**🔑 The 5 Pillars of EC2 Cost Optimization:**

#### 1. Choose the Right Purchase Model
| Workload Pattern | Best Option | Typical Savings |
|-----------------|-------------|-----------------|
| Always-on, predictable | Reserved Instance (1yr/3yr) or Savings Plans | Up to 72% |
| Flexible compute type | Compute Savings Plan | Up to 66% |
| Fault-tolerant, batch | Spot Instances | Up to 90% |
| Short-term / unpredictable | On-Demand | Baseline |
| Regulatory / BYOL | Dedicated Host (Reserved) | Varies |

> 💡 **Tip:** Mix purchase models in an Auto Scaling Group — use Reserved or Savings Plans for the base capacity, Spot for burst.

---

#### 2. Right-Sizing
Paying for more CPU/RAM than you use is waste. Use **AWS Compute Optimizer** to get ML-powered recommendations.

**🖥️ Hands-On — Compute Optimizer:**
1. Go to **AWS Console → Compute Optimizer** (enable it — free service)
2. View **EC2 instance recommendations**
3. It shows: current type, recommended type, estimated monthly savings, and utilization graphs
4. Downsize over-provisioned instances with a single AMI swap

**Common Right-Sizing Patterns:**
- `m5.2xlarge` running at 10% CPU → downsize to `m5.large` (saves ~75%)
- Burstable (`t3`) vs fixed (`m5`) — use `t3` for spiky workloads, `m5` for sustained

---

#### 3. Auto Scaling — Pay Only for What You Use
Never run fixed fleets 24/7 if demand fluctuates.

**Auto Scaling Group (ASG) Cost Tips:**
- Set **minimum capacity** as low as safe (e.g., 1 for non-prod)
- Use **Scheduled Scaling** to pre-emptively scale down nights/weekends
- Use **Target Tracking** (e.g., 60% CPU) rather than step scaling for smoother cost curves
- Combine with Spot Instances for 60–90% fleet cost reduction

---

#### 4. Storage Cost Optimization

| Resource | Cost Leak | Fix |
|----------|-----------|-----|
| EBS volumes | Unattached volumes (`available` state) | Automate detection with AWS Config |
| EBS snapshots | Accumulating old snapshots | Set lifecycle policies in **Data Lifecycle Manager** |
| EBS type | Using `gp2` instead of `gp3` | Migrate — `gp3` is ~20% cheaper with more performance |
| EFS | All files in Standard tier | Enable **Lifecycle Management** → auto-move to EFS-IA after 30 days (92% cheaper) |

**🖥️ Hands-On — EBS Lifecycle Manager:**
1. Go to **EC2 → Lifecycle Manager → Create Lifecycle Policy**
2. Target: EBS snapshots
3. Set retention: keep last 7 daily snapshots, delete older ones
4. This automates cleanup and controls snapshot sprawl

---

#### 5. Monitor & Set Guardrails

**Cost Visibility Tools:**

| Tool | Purpose |
|------|---------|
| **AWS Cost Explorer** | Visualize spending by service, region, tag |
| **AWS Budgets** | Alert when spend exceeds thresholds |
| **AWS Compute Optimizer** | Right-sizing recommendations |
| **Trusted Advisor** | Flags idle/underutilized resources |
| **Cost Anomaly Detection** | ML-based alerts for unexpected spend spikes |

**🖥️ Hands-On — Cost Explorer:**
1. Go to **Billing → Cost Explorer → Enable**
2. Filter by **Service: EC2** and group by **Instance Type**
3. Identify your top 3 cost drivers
4. Switch to **Savings Plans recommendations** tab — AWS shows exactly how much you'd save

---

**Quick Cost Optimization Checklist:**
- [ ] Idle EC2 instances stopped or terminated?
- [ ] Unattached EBS volumes deleted?
- [ ] Old snapshots cleaned up with a lifecycle policy?
- [ ] Over-provisioned instances right-sized via Compute Optimizer?
- [ ] Reserved Instances or Savings Plans purchased for steady workloads?
- [ ] Spot Instances used for batch/dev/test workloads?
- [ ] EFS Lifecycle Management enabled?
- [ ] Budget alert set for unexpected spend?

---

### Phase 4 Checkpoint ✅

- [ ] Can you explain when to use each purchasing option?
- [ ] Can you launch a Spot Instance and understand its interruption model?
- [ ] Can you configure a Spot Fleet with `capacityOptimized` strategy?
- [ ] Can you estimate savings between On-Demand, Reserved, and Spot?
- [ ] Can you use Compute Optimizer to right-size an instance?
- [ ] Can you set up an EBS Lifecycle Policy to automate snapshot cleanup?
- [ ] Can you identify and eliminate idle EC2/EBS resources using Cost Explorer?

---

## Quick Reference Cheat Sheet

```
EC2 USER DATA LOG:     /var/log/cloud-init-output.log
METADATA URL:          http://169.254.169.254/latest/meta-data/
INSTANCE ID:           curl http://169.254.169.254/latest/meta-data/instance-id
PUBLIC IP:             curl http://169.254.169.254/latest/meta-data/public-ipv4
EFS MOUNT:             sudo mount -t efs -o tls <fs-id>:/ /efs
LIST BLOCK DEVICES:    lsblk
FORMAT VOLUME:         sudo mkfs -t ext4 /dev/xvdf
MOUNT VOLUME:          sudo mount /dev/xvdf /mnt/data
CHECK DISK USAGE:      df -h
```

---

## Common Gotchas & Pro Tips

| # | Gotcha / Tip |
|---|-------------|
| 1 | 🔴 **Public IPs change** on stop/start. Use Elastic IP or Route 53 if you need a fixed address. |
| 2 | 🔴 **EBS volumes are AZ-locked.** Use snapshots to move data across AZs. |
| 3 | 🔴 **Instance Store data is lost** when an instance stops or terminates. Never use for persistent data alone. |
| 4 | 🟡 **Security Groups are stateful** — inbound rules automatically allow the response. NACLs are stateless. |
| 5 | 🟡 **EFS is Linux-only.** For Windows shared storage, use FSx for Windows File Server. |
| 6 | 🟡 **Elastic IPs cost money when idle.** Always release unused EIPs. |
| 7 | 🟡 **HDD volume types (st1, sc1) cannot be boot volumes.** |
| 8 | 🟢 **gp3 is almost always better than gp2** — more IOPS, more throughput, lower cost. |
| 9 | 🟢 **Use IAM Roles, never access keys** on EC2 instances. |
| 10 | 🟢 **Tag everything** — instances, volumes, snapshots — makes cost tracking and cleanup much easier. |

---

## Recommended Practice Order

```
Week 1: Phase 1 (Basics + SSH + Security Groups + IAM Roles)
Week 2: Phase 2 (IPs + ENI + Placement Groups + Hibernate)
Week 3: Phase 3 (EBS types + Snapshots + AMI + EFS)
Week 4: Phase 4 (Purchasing options + Spot Fleet + Cost optimization)
```

---

*Happy Cloud Learning! ☁️ — Always clean up your resources after practice.*
