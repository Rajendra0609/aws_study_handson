# ☁️ AWS Route 53 + S3 — Complete DevOps Master Guide (Console + CLI)

> **Who this is for:** Absolute beginners with **zero** AWS knowledge who want to become job-ready for **real-time DevOps projects**.
> **What you get:** Every Route 53 and S3 topic explained simply, with **both AWS Console steps AND AWS CLI commands** for each task, plus real-world project patterns, Infrastructure-as-Code (Terraform), and automation tips.
> **How to use:** Read top to bottom the first time. Later, jump to any topic as a reference. Every command is copy-paste ready.

---

## 📑 Master Table of Contents

### Part 0 — Foundations & Setup
- [0.1 How AWS Works (Mental Model)](#01-how-aws-works-mental-model)
- [0.2 Create an IAM User (never use root)](#02-create-an-iam-user-never-use-root)
- [0.3 Install & Configure the AWS CLI](#03-install--configure-the-aws-cli)
- [0.4 The Real-Time Project We Will Build](#04-the-real-time-project-we-will-build)
- [0.5 Launch Test EC2 Instances (Multi-Region)](#05-launch-test-ec2-instances-multi-region)

### Part 1 — AWS Route 53 (DNS)
- [1.1 What is DNS?](#11-what-is-dns)
- [1.2 Route 53 Overview & Components](#12-route-53-overview--components)
- [1.3 Registering / Bringing a Domain](#13-registering--bringing-a-domain)
- [1.4 Hosted Zones (Public & Private)](#14-hosted-zones-public--private)
- [1.5 Creating DNS Records](#15-creating-dns-records)
- [1.6 TTL (Time To Live)](#16-ttl-time-to-live)
- [1.7 CNAME vs Alias](#17-cname-vs-alias)
- [1.8 Routing Policy — Simple](#18-routing-policy--simple)
- [1.9 Routing Policy — Weighted](#19-routing-policy--weighted)
- [1.10 Routing Policy — Latency](#110-routing-policy--latency)
- [1.11 Health Checks](#111-health-checks)
- [1.12 Routing Policy — Failover](#112-routing-policy--failover)
- [1.13 Routing Policy — Geolocation](#113-routing-policy--geolocation)
- [1.14 Routing Policy — Geoproximity](#114-routing-policy--geoproximity)
- [1.15 Routing Policy — IP-Based](#115-routing-policy--ip-based)
- [1.16 Routing Policy — Multi-Value](#116-routing-policy--multi-value)
- [1.17 Third-Party Domains & Delegation](#117-third-party-domains--delegation)
- [1.18 Route 53 Resolver & Hybrid DNS](#118-route-53-resolver--hybrid-dns)
- [1.19 DNSSEC (Security)](#119-dnssec-security)
- [1.20 Query Logging & Monitoring](#120-query-logging--monitoring)

### Part 2 — AWS S3 (Storage)
- [2.1 S3 Overview](#21-s3-overview)
- [2.2 Buckets & Objects](#22-buckets--objects)
- [2.3 Security: Bucket Policies, IAM, Block Public Access](#23-security-bucket-policies-iam-block-public-access)
- [2.4 Static Website Hosting](#24-static-website-hosting)
- [2.5 Versioning](#25-versioning)
- [2.6 Replication (CRR & SRR)](#26-replication-crr--srr)
- [2.7 Storage Classes](#27-storage-classes)
- [2.7.1 S3 Express One Zone](#271-s3-express-one-zone)
- [2.8 Lifecycle Rules & S3 Analytics](#28-lifecycle-rules--s3-analytics)
- [2.8.1 Requester Pays](#281-requester-pays)
- [2.9 Event Notifications](#29-event-notifications)
- [2.10 Performance & Multipart Upload](#210-performance--multipart-upload)
- [2.11 Batch Operations & Inventory](#211-batch-operations--inventory)
- [2.12 Encryption (SSE-S3, SSE-KMS, SSE-C, DSSE-KMS)](#212-encryption-sse-s3-sse-kms-sse-c-dsse-kms)
- [2.13 CORS](#213-cors)
- [2.14 MFA Delete, Object Lock & Glacier Vault Lock](#214-mfa-delete-object-lock--glacier-vault-lock)
- [2.15 Access Logs](#215-access-logs)
- [2.16 Pre-signed URLs](#216-pre-signed-urls)
- [2.17 Access Points & Object Lambda](#217-access-points--object-lambda)
- [2.18 Storage Lens & Monitoring](#218-storage-lens--monitoring)

### Part 3 — Real-Time Integration Project
- [3.1 Secure HTTPS Static Site: S3 + CloudFront + ACM + Route 53](#31-secure-https-static-site-s3--cloudfront--acm--route-53)
- [3.2 Active-Passive Disaster Recovery](#32-active-passive-disaster-recovery)

### Part 4 — DevOps Automation (IaC & CI/CD)
- [4.1 Terraform for Route 53 + S3](#41-terraform-for-route-53--s3)
- [4.2 CI/CD Deploy Pipeline](#42-cicd-deploy-pipeline)
- [4.3 Cleanup (Avoid Charges)](#43-cleanup-avoid-charges)

---

# Part 0 — Foundations & Setup

## 0.1 How AWS Works (Mental Model)

Think of AWS as a giant rental warehouse of computing services. You don't buy servers — you rent them by the second/GB and return them when done.

| Term | Plain English |
|------|---------------|
| **Region** | A physical geographic area (e.g., Mumbai `ap-south-1`, Virginia `us-east-1`). |
| **Availability Zone (AZ)** | An isolated datacenter inside a region. Use 2+ for high availability. |
| **Service** | A product like S3 (storage) or Route 53 (DNS). |
| **IAM** | Identity & Access Management — controls *who* can do *what*. |
| **ARN** | Amazon Resource Name — the unique ID of any resource, e.g., `arn:aws:s3:::my-bucket`. |
| **Console** | The web UI (point-and-click). |
| **CLI** | The terminal tool (`aws ...`) for automation & scripting. |

> 🧠 **DevOps mindset:** The Console is great for *learning* and one-off tasks. In real projects you use the **CLI and Infrastructure-as-Code (Terraform)** so everything is repeatable, reviewable, and version-controlled.

---

## 0.2 Create an IAM User (never use root)

The **root account** (the email you signed up with) is all-powerful. Never use it for daily work. Create an IAM user with admin rights for learning.

### 🖱️ Console
1. Sign in as root → go to **IAM → Users → Create user**.
2. Username: `devops-admin`. Click **Next**.
3. **Permissions** → *Attach policies directly* → check `AdministratorAccess`.
4. Create user → open the user → **Security credentials** tab.
5. **Create access key** → choose *Command Line Interface (CLI)* → confirm → **Download .csv**.
   - This gives you an **Access Key ID** and **Secret Access Key** (the CLI password). Keep it secret.
6. Also enable **Console access** with a password if you want to log in to the web UI as this user.
7. **Enable MFA** on this user: Security credentials → *Assign MFA device* → use an authenticator app.

### 💻 CLI
You can't bootstrap your *first* user via CLI (no credentials yet), but once you have one admin you can create more:
```bash
# Create a user
aws iam create-user --user-name ci-deployer

# Create programmatic access keys for that user
aws iam create-access-key --user-name ci-deployer

# Attach a managed policy
aws iam attach-user-policy \
  --user-name ci-deployer \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

> 🔐 **Real-time tip:** For automation (CI/CD), don't hand out admin keys. Create **scoped IAM policies** giving only the S3/Route 53 permissions a pipeline needs. Better still, use **IAM Roles** (no long-lived keys at all).

---

## 0.3 Install & Configure the AWS CLI

### Install (Windows / macOS / Linux)
```powershell
# Windows (PowerShell) — official MSI installer
msiexec.exe /i https://awscli.amazonaws.com/AWSCLIV2.msi
```
```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
```

### Verify & Configure
```bash
aws --version           # should print aws-cli/2.x

aws configure
# AWS Access Key ID:      <paste from the .csv>
# AWS Secret Access Key:  <paste from the .csv>
# Default region name:    ap-south-1
# Default output format:  json

# Confirm it works (shows your account + user ARN)
aws sts get-caller-identity
```

> 💡 Use **named profiles** when you juggle multiple accounts:
> ```bash
> aws configure --profile prod
> aws s3 ls --profile prod
> ```

---

## 0.4 The Real-Time Project We Will Build

**Scenario:** You are the DevOps engineer at **TechNova Pvt. Ltd.**, a SaaS startup with users in India, Europe, and the US. Throughout this guide you'll build a production-style setup:

```
                         Route 53 (DNS + routing + health checks + failover)
                                          │
                ┌─────────────────────────┼─────────────────────────┐
                │                          │                         │
        CloudFront (HTTPS/CDN)     ALB → EC2 (Mumbai)        S3 static failover site
                │                  ALB → EC2 (Ireland)
         S3 origin (website        ALB → EC2 (Virginia)
         assets, encrypted,
         versioned, lifecycle)
```

You'll learn each Route 53 and S3 feature on its own, then wire them together in **Part 3**.

---

## 0.5 Launch Test EC2 Instances (Multi-Region)

Several routing policies (latency, geolocation, failover, multi-value) need **real endpoints in different regions**. Launch a tiny web server in each region that prints its own region — so you can *see* routing working.

**Regions we'll use:** `ap-south-1` (Mumbai), `eu-west-1` (Ireland), `us-east-1` (Virginia).

### 🖱️ Console (repeat per region — switch region top-right)
1. **EC2 → Launch instance**.
2. AMI **Amazon Linux 2023** · Type **t2.micro** (free tier) · create/select a **key pair**.
3. **Security group:** allow HTTP (80), HTTPS (443), SSH (22).
4. Expand **Advanced details → User data** and paste the script below.
5. **Launch** → note the **Public IPv4** address.

### User data (auto-installs a web server + `/health` page)
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl enable --now httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
REGION=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/region)
IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4)
echo "<h1>TechNova App - Region: $REGION</h1><p>IP: $IP</p>" > /var/www/html/index.html
echo "OK - $REGION healthy" > /var/www/html/health
```

### 💻 CLI (launch in one region; repeat with `--region` for others)
```bash
# Save the user-data above to userdata.sh, then:
aws ec2 run-instances \
  --region ap-south-1 \
  --image-id resolve:ssm:/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --instance-type t2.micro \
  --key-name my-key \
  --security-group-ids sg-0abc123 \
  --user-data file://userdata.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=technova-mumbai}]'

# Get the public IP once running
aws ec2 describe-instances --region ap-south-1 \
  --filters "Name=tag:Name,Values=technova-mumbai" \
  --query 'Reservations[].Instances[].PublicIpAddress' --output text
```

**Record your IPs** (used throughout Part 1):
```
Mumbai  (ap-south-1):  ________________
Ireland (eu-west-1):   ________________
Virginia (us-east-1):  ________________
```

### Verify each server
```bash
curl http://<mumbai-ip>          # → "TechNova App - Region: ap-south-1"
curl http://<mumbai-ip>/health   # → "OK - ap-south-1 healthy"
```
> 💡 Make sure the security group allows **port 80 from 0.0.0.0/0** or curl/health checks will fail.

---

# Part 1 — AWS Route 53 (DNS)

## 1.1 What is DNS?

DNS (Domain Name System) is the **phonebook of the internet**. It translates human names into IP addresses.

```
You type:   www.technova.com
DNS gives:  13.234.56.78   ← a server's IP
Browser connects to that IP.
```

### Resolution flow
```
Browser → Recursive Resolver (your ISP / 8.8.8.8)
        → Root servers (.)
        → TLD servers (.com)
        → Authoritative server (Route 53)
        → returns IP → browser connects
```

### Common record types
| Record | Maps | Example |
|--------|------|---------|
| **A** | name → IPv4 | `app.technova.com → 13.234.56.78` |
| **AAAA** | name → IPv6 | `app → 2001:db8::1` |
| **CNAME** | name → another name | `www → app.technova.com` |
| **MX** | mail routing | `10 mail.technova.com` |
| **TXT** | text (SPF, verification) | `"v=spf1 ..."` |
| **NS** | name servers for the zone | Route 53's 4 NS |
| **SOA** | zone metadata | auto-created |

### Test DNS from your machine
```bash
nslookup www.technova.com          # Windows + all OS
dig www.technova.com               # macOS/Linux (detailed)
dig +short www.technova.com        # just the IP
Resolve-DnsName www.technova.com   # Windows PowerShell
```

> 💡 Lower TTL = changes propagate faster but cost more queries. Check global propagation at https://dnschecker.org.

---

## 1.2 Route 53 Overview & Components

Route 53 is AWS's **managed DNS + domain registrar + health-checking** service. It is the only AWS service with a **100% availability SLA**. (Named after **port 53**, the DNS port.)

| Component | What it is |
|-----------|-----------|
| **Hosted Zone** | A container for all DNS records of a domain. |
| **Public Hosted Zone** | Resolves on the public internet. |
| **Private Hosted Zone** | Resolves only inside your VPC(s). |
| **Record** | A single mapping (A, CNAME, MX...). |
| **Routing Policy** | How Route 53 chooses which answer to return. |
| **Health Check** | Monitors an endpoint; feeds routing decisions. |
| **Traffic Policy** | Visual builder for complex routing (geoproximity, etc.). |

### Pricing essentials
| Item | Cost |
|------|------|
| Hosted Zone | $0.50/month each |
| DNS queries | $0.40 per million (standard) |
| Health check | $0.50/month (basic) |
| `.com` domain | ~$13/year |

### 💻 Explore via CLI
```bash
aws route53 list-hosted-zones
aws route53 get-hosted-zone-count
aws route53 list-health-checks
```

---

## 1.3 Registering / Bringing a Domain

You have three options. For learning, **Option B or C** avoids cost.

### Option A — Register a new domain (paid)

**🖱️ Console**
1. **Route 53 → Domains → Registered domains → Register domains**.
2. Search a name (e.g., `technova-labs.com`) → **Select** → **Proceed to checkout**.
3. Fill contact info, enable **Privacy protection** and **Auto-renew** → **Submit**.
4. Wait 10–15 min. Route 53 auto-creates a **public hosted zone** with NS + SOA.

**💻 CLI**
```bash
# Check availability
aws route53domains check-domain-availability \
  --region us-east-1 --domain-name technova-labs.com

# Register (contact JSON omitted for brevity)
aws route53domains register-domain \
  --region us-east-1 \
  --domain-name technova-labs.com \
  --duration-in-years 1 \
  --auto-renew \
  --admin-contact file://contact.json \
  --registrant-contact file://contact.json \
  --tech-contact file://contact.json
```
> ⚠️ Route 53 Domains API only runs in **us-east-1**.

### Option B — Use a domain you already own elsewhere
Delegate it to Route 53 (covered fully in [1.17](#117-third-party-domains--delegation)).

### Option C — Practice with only a hosted zone (no real domain)
You can create a hosted zone and test records with `dig @<route53-ns>` directly, without owning the domain — perfect for free practice.

---

## 1.4 Hosted Zones (Public & Private)

A **hosted zone** holds the records for one domain.

### 🖱️ Console — Public hosted zone
1. **Route 53 → Hosted zones → Create hosted zone**.
2. Domain name: `technova.com`. Type: **Public hosted zone** → **Create**.
3. Note the **4 NS records** + **SOA** that appear automatically.

### 🖱️ Console — Private hosted zone (VPC-internal DNS)
1. **Create hosted zone** → name `internal.technova.com` → Type: **Private hosted zone**.
2. Choose the **Region** and **VPC** to associate → **Create**.
3. Records here resolve only inside that VPC.

### 💻 CLI — Public zone
```bash
aws route53 create-hosted-zone \
  --name technova.com \
  --caller-reference $(date +%s)

# Get the NS records (the 4 name servers to give your registrar)
aws route53 get-hosted-zone --id /hostedzone/Z123456ABCDEF
```

### 💻 CLI — Private zone
```bash
aws route53 create-hosted-zone \
  --name internal.technova.com \
  --caller-reference $(date +%s) \
  --vpc VPCRegion=ap-south-1,VPCId=vpc-0abc123 \
  --hosted-zone-config PrivateZone=true
```

> 💡 **Real-time tip:** Use a **private hosted zone** for internal service discovery (e.g., `db.internal.technova.com`) so app servers never hardcode IPs.

---

## 1.5 Creating DNS Records

A record maps a name to a value. Let's create an **A record** for `www`.

### 🖱️ Console
1. **Route 53 → Hosted zones → technova.com → Create record**.
2. Record name: `www` · Type: `A` · Value: `13.234.56.78` · TTL: `300` · Routing: **Simple** → **Create records**.

### 💻 CLI (uses a "change batch" JSON)
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "www.technova.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{ "Value": "13.234.56.78" }]
      }
    }]
  }'
```
- `UPSERT` = create or update. Other actions: `CREATE`, `DELETE`.

### Verify
```bash
dig +short www.technova.com
aws route53 list-resource-record-sets --hosted-zone-id Z123456ABCDEF
```

> 💡 Leave **Record name blank** to create the **root/apex** record (`technova.com`).

---

## 1.6 TTL (Time To Live)

TTL = how long resolvers **cache** a record (in seconds) before asking again.

```
TTL 60    → 1 min  (changes propagate fast, more queries/cost)
TTL 3600  → 1 hour (balanced — common default)
TTL 86400 → 24 hr  (cheap, slow to change)
```

### Production strategy
```
Steady state:          TTL 3600
24h BEFORE a migration: lower TTL to 60
AFTER cutover & stable: raise back to 3600
```

### 🖱️ Console
Edit the record → change **TTL** field → save.

### 💻 CLI
Re-`UPSERT` the same record with a new `"TTL"` value (see 1.5).

### Observe TTL counting down
```bash
dig www.technova.com
# ;; ANSWER SECTION:
# www.technova.com.  287  IN  A  13.234.56.78   ← 287 = seconds left in cache
```

> 💡 **Alias records have no TTL** — Route 53 manages it for you.

---

## 1.7 CNAME vs Alias

Both point one name at another, but differ critically (a top interview/exam topic).

| Feature | CNAME | Alias |
|---------|-------|-------|
| Points to | any domain name | AWS resource or same-zone record |
| Works at root/apex (`technova.com`) | ❌ No | ✅ Yes |
| Points to AWS resource (ALB, CloudFront, S3) | ❌ | ✅ |
| Query charge | charged | free for AWS resources |
| TTL | you set | Route 53 manages |
| Standard DNS | ✅ RFC | ❌ Route 53 extension |

```
www  → app.technova.com                       ✅ CNAME ok
technova.com → my-alb.elb.amazonaws.com        ✅ MUST be Alias (apex)
www  → d123.cloudfront.net                     ✅ Alias preferred
```

### 🖱️ Console — Alias to a Load Balancer
1. Create record → name `app` · Type `A` → toggle **Alias** ON.
2. *Route traffic to* → **Alias to Application/Classic Load Balancer** → region → pick ALB → **Create**.

### 💻 CLI — Alias record
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "app.technova.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "ZP97RAFLXTNZK",
          "DNSName": "my-alb-1234.ap-south-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```
> The `HostedZoneId` inside `AliasTarget` is the **target's** zone ID (each ELB/CloudFront/S3 endpoint has a fixed one — look it up in AWS docs or via `describe-load-balancers`).

---

## 1.8 Routing Policy — Simple

Default policy: one record, one or many IPs. If multiple, Route 53 returns them all and the **client picks randomly**. **No health checks.**

### 🖱️ Console
Create record → name `simple` · Type `A` · add multiple values (one per line) · Routing: **Simple**.

### 💻 CLI
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "simple.technova.com", "Type": "A", "TTL": 60,
        "ResourceRecords": [{ "Value": "13.234.56.78" }, { "Value": "52.30.12.34" }]
      }
    }]
  }'
```
✅ Use for: single endpoints, dev/test. ❌ Avoid when you need failover or health awareness.

---

## 1.9 Routing Policy — Weighted

Split traffic by **percentage**. Great for **blue-green / canary** rollouts and A/B testing.

```
Traffic % = this weight / sum of all weights × 100
Weight 0 = excluded (but not deleted)
```

### 🖱️ Console (create two records, same name)
- Record A: name `weighted` · value Mumbai IP · Routing **Weighted** · Weight `80` · Record ID `v1-mumbai`.
- Record B: name `weighted` · value Ireland IP · Routing **Weighted** · Weight `20` · Record ID `v2-ireland`.

### 💻 CLI
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "weighted.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "v1-mumbai", "Weight": 80,
          "ResourceRecords": [{ "Value": "13.234.56.78" }] }},
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "weighted.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "v2-ireland", "Weight": 20,
          "ResourceRecords": [{ "Value": "52.30.12.34" }] }}
    ]
  }'
```
### Test
```bash
for i in $(seq 1 10); do dig +short weighted.technova.com; done   # ~8 Mumbai, ~2 Ireland
```
> 💡 **Real-time:** Canary a new release: 95/5 → 80/20 → 50/50 → 0/100, watching error rates between steps.

---

## 1.10 Routing Policy — Latency

Send users to the **AWS region with the lowest latency** for them. One record per region, same name.

### 🖱️ Console
Create a record per region: name `app` · value region IP · Routing **Latency** · Region `ap-south-1` / `eu-west-1` / `us-east-1` · unique Record ID each.

### 💻 CLI
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "app.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "mumbai", "Region": "ap-south-1",
          "ResourceRecords": [{ "Value": "13.234.56.78" }] }},
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "app.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "ireland", "Region": "eu-west-1",
          "ResourceRecords": [{ "Value": "52.30.12.34" }] }}
    ]
  }'
```
> 💡 Latency routing uses AWS's **latency database**, not live pings. Combine with health checks for auto-failover.

---

## 1.11 Health Checks

Health checks monitor endpoints so Route 53 stops sending traffic to dead ones.

| Type | Use |
|------|-----|
| **Endpoint** | Ping an IP/domain over HTTP/HTTPS/TCP. |
| **Calculated** | Combine child checks with AND/OR. |
| **CloudWatch Alarm** | Health from a metric (good for private resources). |

### 🖱️ Console
1. **Route 53 → Health checks → Create health check**.
2. Name `technova-mumbai-health` · Monitor **Endpoint** · by **IP address** · Protocol `HTTP` · IP `<Mumbai IP>` · Port `80` · Path `/health`.
3. Advanced: Interval `30s` · Failure threshold `3` · (optional) **String matching** = `OK`.
4. Create → wait ~60–90s → status **Healthy**.

### 💻 CLI
```bash
# Create an HTTP health check
aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "IPAddress": "13.234.56.78",
    "Port": 80, "Type": "HTTP", "ResourcePath": "/health",
    "RequestInterval": 30, "FailureThreshold": 3
  }'

# List + check status
aws route53 list-health-checks
aws route53 get-health-check-status --health-check-id <id>
```
> ⚠️ Allow Route 53 **health-checker IP ranges** in your security group, or expose a public `/health` path.
> 💡 Add the health-check ID onto routing records (next section) so failover actually triggers.

---

## 1.12 Routing Policy — Failover

Active-passive HA: exactly **one PRIMARY + one SECONDARY**. Traffic goes to primary until its health check fails, then to secondary.

### 🖱️ Console
- Primary: record `failover` · value Mumbai IP · Routing **Failover** · type **Primary** · attach health check.
- Secondary: record `failover` · **Alias → S3 website endpoint** (a "maintenance" page) · type **Secondary**.

### Build the S3 failover page (CLI)
```bash
aws s3 mb s3://failover.technova.com --region ap-south-1
aws s3 website s3://failover.technova.com --index-document index.html
echo "<h1>TechNova — Maintenance Mode</h1>" > index.html
aws s3 cp index.html s3://failover.technova.com/
```

### 💻 CLI — Failover records
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "failover.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "primary", "Failover": "PRIMARY",
          "HealthCheckId": "<health-check-id>",
          "ResourceRecords": [{ "Value": "13.234.56.78" }] }},
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "failover.technova.com", "Type": "A",
          "SetIdentifier": "secondary", "Failover": "SECONDARY",
          "AliasTarget": {
            "HostedZoneId": "Z11RGJOFQNVNQO",
            "DNSName": "s3-website.ap-south-1.amazonaws.com",
            "EvaluateTargetHealth": false } }}
    ]
  }'
```
> 💡 Primary **must** have a health check; secondary need not. Works in private zones too.

---

## 1.13 Routing Policy — Geolocation

Route by **where the user is** (continent / country / US state). **Always create a `Default` record** for unlisted locations.

### 🖱️ Console
- `geo` · Mumbai IP · Geolocation **Asia → India** · ID `geo-india`.
- `geo` · Ireland IP · Geolocation **Europe** · ID `geo-europe`.
- `geo` · Virginia IP · Geolocation **Default** · ID `geo-default`.

### 💻 CLI
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "geo.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "india", "GeoLocation": { "CountryCode": "IN" },
          "ResourceRecords": [{ "Value": "13.234.56.78" }] }},
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "geo.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "europe", "GeoLocation": { "ContinentCode": "EU" },
          "ResourceRecords": [{ "Value": "52.30.12.34" }] }},
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "geo.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "default", "GeoLocation": { "CountryCode": "*" },
          "ResourceRecords": [{ "Value": "3.91.10.20" }] }}
    ]
  }'
```
> 💡 Best for **data sovereignty (GDPR)** and **localized content**. Geolocation = where user IS; Latency = what's fastest.

---

## 1.14 Routing Policy — Geoproximity

Route by **physical distance**, with a **bias** (−99…+99) to grow/shrink a region's coverage. Requires **Traffic Flow** (Traffic Policies).

### 🖱️ Console
1. **Route 53 → Traffic policies → Create traffic policy** → name `technova-geoprox` · DNS type `A`.
2. In the visual editor add a **Geoproximity rule**; add each region endpoint with a Bias (e.g., Mumbai `+20`, others `0`).
3. **Create traffic policy** → **Create policy record** → DNS name `geopx.technova.com` · TTL 60.

### 💻 CLI
```bash
# Traffic policy document defined in policy.json (visual editor exports this)
aws route53 create-traffic-policy \
  --name technova-geoprox --document file://policy.json

aws route53 create-traffic-policy-instance \
  --hosted-zone-id Z123456ABCDEF \
  --name geopx.technova.com --ttl 60 \
  --traffic-policy-id <id> --traffic-policy-version 1
```
> 💡 Positive bias **attracts** more users to a region; negative **pushes** them away. The console shows a live **coverage map**.

---

## 1.15 Routing Policy — IP-Based

Route by the **user's CIDR (IP range)** — the most granular option. You define a **CIDR collection** first.

### 🖱️ Console
1. **Route 53 → CIDR collections → Create** → name `technova-corp-ips` → add location `office-mumbai` with CIDR `103.21.58.0/24`.
2. Create record `ipbased` · Routing **IP-based** · pick the collection + location · value Mumbai IP.
3. Create a second `ipbased` record with CIDR location **Default** · value Virginia IP.

### 💻 CLI
```bash
# 1) Create the CIDR collection
aws route53 create-cidr-collection --name technova-corp-ips --caller-reference $(date +%s)

# 2) Add CIDR blocks to a location
aws route53 change-cidr-collection \
  --id <collection-id> \
  --changes '[{ "LocationName": "office-mumbai", "Action": "PUT", "CidrList": ["103.21.58.0/24"] }]'

# 3) Create the IP-based record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [{ "Action": "UPSERT", "ResourceRecordSet": {
      "Name": "ipbased.technova.com", "Type": "A", "TTL": 60,
      "SetIdentifier": "corp",
      "CidrRoutingConfig": { "CollectionId": "<collection-id>", "LocationName": "office-mumbai" },
      "ResourceRecords": [{ "Value": "13.234.56.78" }] }}]
  }'
```
> 💡 Limits: 1,000 CIDRs/collection, 5 collections/zone. Used by enterprises for ISP/carrier-level steering.

---

## 1.16 Routing Policy — Multi-Value

Returns up to **8 healthy** records per query (client-side load balancing with health awareness). Not a replacement for a load balancer.

### 🖱️ Console
Create several `multi` records (same name), each with its **own health check**, Routing **Multivalue answer**, unique Record IDs.

### 💻 CLI
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "multi.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "mumbai", "MultiValueAnswer": true,
          "HealthCheckId": "<hc-mumbai>",
          "ResourceRecords": [{ "Value": "13.234.56.78" }] }},
      { "Action": "UPSERT", "ResourceRecordSet": {
          "Name": "multi.technova.com", "Type": "A", "TTL": 60,
          "SetIdentifier": "ireland", "MultiValueAnswer": true,
          "HealthCheckId": "<hc-ireland>",
          "ResourceRecords": [{ "Value": "52.30.12.34" }] }}
    ]
  }'
```
> 💡 Records *without* a health check are always returned (assumed healthy).

---

## 1.17 Third-Party Domains & Delegation

Use Route 53 for DNS even if the domain is registered at GoDaddy/Namecheap — just point the registrar's **NS records** at Route 53.

### Full-domain delegation
1. **🖱️ Create a public hosted zone** for `yourdomain.com` in Route 53; copy the **4 NS values**.
2. At your registrar → *Nameservers* → **Custom** → paste the 4 Route 53 NS → save.
3. Verify propagation:
```bash
dig NS yourdomain.com @8.8.8.8
nslookup -type=NS yourdomain.com 8.8.8.8
```

### Subdomain delegation (keep parent elsewhere)
```
technova.com      → stays at Namecheap
aws.technova.com  → delegated to Route 53
```
1. Create hosted zone `aws.technova.com` in Route 53 → copy its NS.
2. At Namecheap, add an **NS record** for host `aws` pointing to those 4 servers.

### 💻 CLI — get a new zone's NS quickly
```bash
aws route53 get-hosted-zone --id /hostedzone/Z123456ABCDEF \
  --query 'DelegationSet.NameServers'
```
> ⚠️ Never delete the **NS/SOA** records — it breaks the zone.

---

## 1.18 Route 53 Resolver & Hybrid DNS

Resolver enables DNS resolution **between AWS VPC and on-premises** networks.

| Component | Direction | Purpose |
|-----------|-----------|---------|
| **Inbound endpoint** | on-prem → VPC | on-prem can resolve AWS private DNS |
| **Outbound endpoint** | VPC → on-prem | EC2 can resolve on-prem DNS |
| **Resolver rule** | forward | send specific domains to a target DNS |

### 🖱️ Console
1. **Route 53 → Resolver → Inbound endpoints → Create**; pick VPC, security group (allow TCP/UDP 53), 2 IPs in 2 AZs.
2. **Outbound endpoints → Create** similarly.
3. **Rules → Create rule** → type **Forward** · domain `technova.local` · outbound endpoint · target on-prem DNS `192.168.1.53` · associate VPC.

### 💻 CLI
```bash
aws route53resolver create-resolver-endpoint \
  --name technova-outbound --direction OUTBOUND \
  --security-group-ids sg-0abc \
  --ip-addresses SubnetId=subnet-1,Ip=172.16.1.20 SubnetId=subnet-2,Ip=172.16.2.20

aws route53resolver create-resolver-rule \
  --name forward-onprem --rule-type FORWARD \
  --domain-name technova.local \
  --resolver-endpoint-id rslvr-out-xxxx \
  --target-ips Ip=192.168.1.53,Port=53

aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-xxxx --vpc-id vpc-0abc123
```
> 💡 Deploy endpoints in **2+ AZs**. Share rules across accounts with **AWS RAM**.

---

## 1.19 DNSSEC (Security)

DNSSEC cryptographically signs your zone so resolvers can verify answers weren't tampered with (prevents DNS spoofing/cache poisoning).

### 🖱️ Console
1. **Route 53 → Hosted zones → [zone] → DNSSEC signing → Enable DNSSEC signing**.
2. Create/choose a **KMS asymmetric key (ECC_NIST_P256)** in `us-east-1` for the KSK.
3. Route 53 generates a **DS record** → give it to your **registrar / parent zone** to complete the chain of trust.

### 💻 CLI
```bash
# Enable signing (KSK must reference a us-east-1 asymmetric KMS key)
aws route53 create-key-signing-key \
  --caller-reference $(date +%s) \
  --hosted-zone-id Z123456ABCDEF \
  --key-management-service-arn arn:aws:kms:us-east-1:111122223333:key/abc \
  --name technova-ksk --status ACTIVE

aws route53 enable-hosted-zone-dnssec --hosted-zone-id Z123456ABCDEF

# Get the DS record to hand to your registrar
aws route53 get-dnssec --hosted-zone-id Z123456ABCDEF
```
> ⚠️ Enable DNSSEC at the **registrar** too (add the DS record), or validation won't work end-to-end.

---

## 1.20 Query Logging & Monitoring

Log every public DNS query for auditing/troubleshooting; ship to **CloudWatch Logs**.

### 🖱️ Console
1. **Route 53 → Hosted zones → [zone] → Configure query logging**.
2. Choose/create a CloudWatch Logs log group (must be in **us-east-1**) → **Create**.

### 💻 CLI
```bash
aws route53 create-query-logging-config \
  --hosted-zone-id Z123456ABCDEF \
  --cloud-watch-logs-log-group-arn \
    arn:aws:logs:us-east-1:111122223333:log-group:/aws/route53/technova.com
```
Monitor health checks in **CloudWatch** (`HealthCheckStatus`, `HealthCheckPercentageHealthy`) and alarm on failures.

> 💡 **Real-time:** Query logs help debug "why is traffic going to the wrong region?" and detect unusual query spikes (possible attacks).

---

# Part 2 — AWS S3 (Storage)

## 2.1 S3 Overview

**Amazon S3 (Simple Storage Service)** is object storage: you put files ("objects") into "buckets". It's massively scalable, 11-nines (99.999999999%) durable, and pay-as-you-go.

| Concept | Meaning |
|---------|---------|
| **Bucket** | Top-level container. Name is **globally unique** across all AWS. |
| **Object** | A file + its metadata + version ID. Max **5 TB**. |
| **Key** | The object's full "path" name (e.g., `images/logo.png`). |
| **Region** | Where the bucket physically lives. |
| **Prefix** | The part of a key before `/` — S3 *simulates* folders this way. |

> 🧠 S3 is a **flat key-value store** — there are no real folders, just keys with `/` in them. The console *shows* folders for convenience.
> URL format: `https://<bucket>.s3.<region>.amazonaws.com/<key>`

```bash
# See your account's buckets right now
aws s3 ls
```

---

## 2.2 Buckets & Objects

### Create a bucket + upload a file

**🖱️ Console**
1. **S3 → Create bucket** → name `technova-learning-2024` (globally unique) → choose region → keep **Block all public access** ON → **Create bucket**.
2. Open bucket → **Upload** → add files → **Upload**.
3. Click an object → see **Properties**, **Object URL**, **Metadata**.

**💻 CLI**
```bash
# Create bucket (note: us-east-1 needs no LocationConstraint)
aws s3 mb s3://technova-learning-2024 --region ap-south-1

# Upload a single file
echo "hello s3" > hello.txt
aws s3 cp hello.txt s3://technova-learning-2024/

# Upload a whole folder (recursive) — simulates "folders" with key prefixes
aws s3 cp ./website s3://technova-learning-2024/website/ --recursive

# List objects
aws s3 ls s3://technova-learning-2024/ --recursive --human-readable --summarize

# Download / sync / remove
aws s3 cp s3://technova-learning-2024/hello.txt ./
aws s3 sync ./dist s3://technova-learning-2024/   # only changed files
aws s3 rm s3://technova-learning-2024/hello.txt
```
> 💡 `aws s3` = high-level (cp/sync/mb/rb). `aws s3api` = low-level, one-to-one with the REST API (more control).
> Try opening the Object URL in a browser → you'll get **AccessDenied** (private by default). Good.

---

## 2.3 Security: Bucket Policies, IAM, Block Public Access

Access is granted only if **IAM policy AND bucket policy allow** it, with **no explicit Deny** anywhere, and **Block Public Access** not overriding it.

| Control | Scope |
|---------|-------|
| **IAM policy** | Attached to users/roles ("what can this identity do"). |
| **Bucket policy** | Attached to the bucket ("who can touch this bucket"). |
| **ACL** | Legacy per-object/bucket grants — avoid; prefer policies. |
| **Block Public Access (BPA)** | Master safety switch that overrides public grants. |

### Example bucket policy (public read of objects)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::technova-learning-2024/*"
  }]
}
```

**🖱️ Console**
1. Bucket → **Permissions** → **Block public access** → **Edit** → uncheck → **Save** (type `confirm`).
2. **Bucket policy** → **Edit** → paste JSON (fix the bucket name) → **Save**.

**💻 CLI**
```bash
# Turn off Block Public Access for this bucket (needed for public policy)
aws s3api put-public-access-block \
  --bucket technova-learning-2024 \
  --public-access-block-configuration \
    BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false

# Apply the bucket policy
aws s3api put-bucket-policy \
  --bucket technova-learning-2024 \
  --policy file://policy.json

# Read it back
aws s3api get-bucket-policy --bucket technova-learning-2024
```
> 🔐 **Real-time:** Keep BPA **ON** for almost every bucket. Serve "public" content through **CloudFront with Origin Access Control**, not a public bucket. Restrict by IP with a `Condition`:
> ```json
> "Condition": { "IpAddress": { "aws:SourceIp": "203.0.113.0/24" } }
> ```

---

## 2.4 Static Website Hosting

S3 can serve static sites (HTML/CSS/JS) directly — **HTTP only**. For HTTPS use CloudFront (Part 3).

**🖱️ Console**
1. Upload `index.html` (and `error.html`).
2. Bucket → **Properties → Static website hosting → Edit → Enable** → index `index.html`, error `error.html` → **Save**.
3. Apply the public-read bucket policy (2.3).
4. Open the **Website endpoint**: `http://<bucket>.s3-website-<region>.amazonaws.com`.

**💻 CLI**
```bash
echo "<h1>Hello from S3 Static Website</h1>" > index.html
echo "<h1>Oops - Not Found</h1>" > error.html
aws s3 cp index.html s3://technova-learning-2024/
aws s3 cp error.html s3://technova-learning-2024/

aws s3 website s3://technova-learning-2024/ \
  --index-document index.html --error-document error.html

# website endpoint:
# http://technova-learning-2024.s3-website-ap-south-1.amazonaws.com
```
> 💡 Website endpoint ≠ REST endpoint. Use the **s3-website** URL for hosting. CORS needed if it calls other-domain APIs (2.13).

---

## 2.5 Versioning

Keeps multiple versions of each object — protects against overwrites and deletes. Enabled at the **bucket level**; can be suspended but never fully removed once enabled.

**🖱️ Console**
Bucket → **Properties → Bucket Versioning → Edit → Enable**. Toggle **Show versions** to view history.

**💻 CLI**
```bash
aws s3api put-bucket-versioning \
  --bucket technova-learning-2024 \
  --versioning-configuration Status=Enabled

# List all versions and delete markers
aws s3api list-object-versions --bucket technova-learning-2024

# Get a specific old version
aws s3api get-object \
  --bucket technova-learning-2024 --key index.html \
  --version-id <VERSION_ID> index-old.html
```
> 💡 Deleting a versioned object adds a **delete marker** (object still recoverable). Each version is billed separately — pair with **lifecycle rules** (2.8) to expire old versions.

---

## 2.6 Replication (CRR & SRR)

Automatically copy objects to another bucket. **CRR** = cross-region (DR, latency, compliance). **SRR** = same-region (log aggregation, dev/prod sync). **Versioning must be ON** in both buckets. Async; only **new** objects replicate (use Batch Replication for existing).

**🖱️ Console**
Source bucket → **Management → Replication rules → Create** → apply to all objects → choose destination bucket → let AWS **create the IAM role** → save.

**💻 CLI**
```bash
# Both buckets must have versioning enabled first.
aws s3api put-bucket-replication \
  --bucket technova-source \
  --replication-configuration '{
    "Role": "arn:aws:iam::111122223333:role/s3-replication-role",
    "Rules": [{
      "ID": "replicate-all", "Status": "Enabled", "Priority": 1,
      "Filter": {}, "DeleteMarkerReplication": { "Status": "Disabled" },
      "Destination": {
        "Bucket": "arn:aws:s3:::technova-destination-mumbai",
        "StorageClass": "STANDARD"
      }
    }]
  }'
```
> 💡 No **chaining** (A→B→C doesn't reach C from A). Cross-account replication is supported with the right role/policy. **RTC** gives a 15-min SLA (extra cost).

---

## 2.7 Storage Classes

Pick a class based on access frequency vs cost.

| Class | Use case | Min duration |
|-------|----------|--------------|
| **Standard** | Hot, frequent | none |
| **Standard-IA** | Infrequent, fast retrieval | 30 days |
| **One Zone-IA** | Infrequent, non-critical (1 AZ) | 30 days |
| **Intelligent-Tiering** | Unknown/changing patterns | none |
| **Glacier Instant Retrieval** | Archive, ms access | 90 days |
| **Glacier Flexible Retrieval** | Archive, min–hrs | 90 days |
| **Glacier Deep Archive** | Cold, 12–48 hr | 180 days |

**🖱️ Console** — On upload (or object → **Properties → Storage class → Edit**) pick a class.

**💻 CLI**
```bash
# Upload directly into a class
aws s3 cp report.csv s3://technova-learning-2024/ --storage-class STANDARD_IA

# Change an existing object's class (copy onto itself)
aws s3 cp s3://technova-learning-2024/report.csv s3://technova-learning-2024/report.csv \
  --storage-class GLACIER --metadata-directive COPY
```
> 💡 Don't hand-pick classes for thousands of objects — automate with **lifecycle rules** (2.8) or use **Intelligent-Tiering**.

---

## 2.7.1 S3 Express One Zone

A **single-AZ, ultra-low-latency** storage class (single-digit-ms, up to 10× faster than Standard) for latency-critical workloads — ML training, analytics, HPC. It uses special **Directory Buckets** (not general-purpose buckets) and trades multi-AZ redundancy for speed/cost-per-request.

- Directory bucket naming: `name--<az-id>--x-s3` (e.g., `technova--apse1-az1--x-s3`).
- Cheaper per request, but data is in **one AZ** — acceptable only if losing that AZ's copy is OK.

**🖱️ Console** — **S3 → Directory buckets → Create directory bucket** → choose **Availability Zone** + name → create. Upload as usual.

**💻 CLI**
```bash
# Create a directory bucket (S3 Express One Zone)
aws s3api create-bucket \
  --bucket technova--apse1-az1--x-s3 \
  --create-bucket-configuration \
    'Location={Type=AvailabilityZone,Name=apse1-az1},Bucket={DataRedundancy=SingleAvailabilityZone,Type=Directory}' \
  --region ap-southeast-1

# Use it like any bucket
aws s3 cp model.bin s3://technova--apse1-az1--x-s3/
```
> 💡 Use Express One Zone only when **latency is critical** and single-AZ durability is acceptable. Pair it with compute in the **same AZ** for max speed.

---

## 2.8 Lifecycle Rules & S3 Analytics

Automate transitions to cheaper classes and expiration of old objects/versions.

```
Day 0   Standard
Day 30  → Standard-IA
Day 90  → Glacier Flexible
Day 365 expire (delete)
```

**🖱️ Console**
Bucket → **Management → Lifecycle rules → Create** → name → scope (all or prefix/tags) → add transitions + expiration → create.

**💻 CLI**
```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket technova-learning-2024 \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "archive-old-files", "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "Transitions": [
        { "Days": 30, "StorageClass": "STANDARD_IA" },
        { "Days": 90, "StorageClass": "GLACIER" }
      ],
      "Expiration": { "Days": 365 },
      "NoncurrentVersionExpiration": { "NoncurrentDays": 90 },
      "AbortIncompleteMultipartUpload": { "DaysAfterInitiation": 7 }
    }]
  }'
```
> 💡 **Always** add `AbortIncompleteMultipartUpload` (7 days) — it cleans hidden, billable junk from failed uploads.

### S3 Analytics — Storage Class Analysis
Before guessing transition timings, let S3 **observe access patterns** and recommend when to move data to Standard-IA. The first report takes **24–48 hours**.

**🖱️ Console** — Bucket → **Metrics → Storage Class Analysis → Create analytics configuration** → scope (whole bucket or prefix) → optionally export the daily report to a bucket.

**💻 CLI**
```bash
aws s3api put-bucket-analytics-configuration \
  --bucket technova-learning-2024 --id whole-bucket \
  --analytics-configuration '{
    "Id": "whole-bucket",
    "StorageClassAnalysis": {
      "DataExport": {
        "OutputSchemaVersion": "V_1",
        "Destination": { "S3BucketDestination": {
          "Format": "CSV",
          "Bucket": "arn:aws:s3:::technova-analytics",
          "Prefix": "sca/" } }
      }
    }
  }'
```
> 💡 Workflow: **Analytics observes → recommends → you encode it into a lifecycle rule.**

---

## 2.8.1 Requester Pays

Normally the **bucket owner** pays for storage + transfer. With **Requester Pays**, the **downloader** pays for data transfer and request costs — ideal for sharing large public datasets (satellite imagery, genomics) without footing the bandwidth bill. Requesters must be **authenticated** (no anonymous access) and send `x-amz-request-payer: requester`.

**🖱️ Console** — Bucket → **Properties → Requester pays → Edit → Enable**.

**💻 CLI**
```bash
# Enable Requester Pays on a bucket
aws s3api put-bucket-request-payment \
  --bucket technova-public-datasets \
  --request-payment-configuration Payer=Requester

# A requester must opt in to pay when accessing it
aws s3api get-object \
  --bucket technova-public-datasets --key big-dataset.csv out.csv \
  --request-payer requester
```
> 💡 Common for AWS Open Data. Without `--request-payer requester`, requests to such buckets are denied.

---

## 2.9 Event Notifications

Trigger actions when objects are created/removed. Destinations: **SNS**, **SQS**, **Lambda**, or **EventBridge** (recommended for rich filtering/replay).

```
S3 upload → event → Lambda → process file → store result
```

**🖱️ Console**
Bucket → **Properties → Event notifications → Create** → name → event `s3:ObjectCreated:*` → destination (Lambda/SQS/SNS). Or enable **EventBridge** integration and build a rule there.

**💻 CLI**
```bash
# Allow S3 to invoke the Lambda first
aws lambda add-permission \
  --function-name s3-event-handler \
  --statement-id s3invoke --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn arn:aws:s3:::technova-learning-2024

# Wire the notification
aws s3api put-bucket-notification-configuration \
  --bucket technova-learning-2024 \
  --notification-configuration '{
    "LambdaFunctionConfigurations": [{
      "LambdaFunctionArn": "arn:aws:lambda:ap-south-1:111122223333:function:s3-event-handler",
      "Events": ["s3:ObjectCreated:*"]
    }]
  }'

# Or just route everything to EventBridge
aws s3api put-bucket-notification-configuration \
  --bucket technova-learning-2024 \
  --notification-configuration '{ "EventBridgeConfiguration": {} }'
```

---

## 2.10 Performance & Multipart Upload

- **Baseline:** 3,500 writes & 5,500 reads **per second per prefix** — spread keys across prefixes for scale.
- **Multipart upload:** recommended >100 MB, required >5 GB; uploads parts in parallel and resumes.
- **Transfer Acceleration:** uploads via CloudFront edge for faster global transfers.
- **Byte-range fetches:** download parts in parallel / read just a file header.

**🖱️ Console** — Enable **Transfer Acceleration** under bucket **Properties**. The console auto-uses multipart for big files.

**💻 CLI** (the high-level `cp`/`sync` already do multipart automatically)
```bash
# Tune multipart thresholds
aws configure set default.s3.multipart_threshold 100MB
aws configure set default.s3.multipart_chunksize 50MB
aws s3 cp bigfile.zip s3://technova-learning-2024/    # auto multipart + parallel

# Enable & use Transfer Acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket technova-learning-2024 --accelerate-configuration Status=Enabled
aws s3 cp bigfile.zip s3://technova-learning-2024/ \
  --endpoint-url https://s3-accelerate.amazonaws.com
```
> 💡 Avoid sequential prefixes (`0001`, `0002`) for hot workloads — use hashed/random prefixes to spread load.

---

## 2.11 Batch Operations & Inventory

Run one operation (copy, tag, ACL, Lambda, restore, Object Lock) across **billions** of objects, driven by an **S3 Inventory** report or a CSV manifest.

**🖱️ Console**
1. (Optional) Bucket → **Management → Inventory configurations → Create** (daily/weekly CSV/Parquet report).
2. **S3 → Batch Operations → Create job** → choose manifest → operation → IAM role → report destination → **Run**.

**💻 CLI**
```bash
# Set up a daily inventory report
aws s3api put-bucket-inventory-configuration \
  --bucket technova-learning-2024 --id daily-inv \
  --inventory-configuration '{
    "Id": "daily-inv", "IsEnabled": true,
    "IncludedObjectVersions": "Current",
    "Schedule": { "Frequency": "Daily" },
    "Destination": { "S3BucketDestination": {
      "Bucket": "arn:aws:s3:::technova-inventory", "Format": "CSV" } }
  }'

# Create a batch job (manifest + operation in JSON)
aws s3control create-job \
  --account-id 111122223333 \
  --operation file://operation.json \
  --manifest file://manifest.json \
  --report file://report.json \
  --priority 10 --role-arn arn:aws:iam::111122223333:role/s3-batch-role
```

---

## 2.12 Encryption (SSE-S3, SSE-KMS, SSE-C, DSSE-KMS)

All objects can be encrypted at rest. SSE-S3 is the **default** since 2023.

| Method | Keys managed by | Notes |
|--------|-----------------|-------|
| **SSE-S3** | AWS (AES-256) | Free, automatic. |
| **SSE-KMS** | You (via KMS) | Audit trail in CloudTrail, key rotation. |
| **DSSE-KMS** | You (dual layer) | Two independent KMS layers; compliance/defense. |
| **SSE-C** | You provide key per request | AWS never stores your key. |
| **Client-side** | You, before upload | AWS sees only ciphertext. |

### Set bucket default encryption (recommended)

**🖱️ Console** — Bucket → **Properties → Default encryption → Edit** → SSE-S3 or SSE-KMS (pick key).

**💻 CLI**
```bash
# Default SSE-KMS on the bucket
aws s3api put-bucket-encryption \
  --bucket technova-learning-2024 \
  --server-side-encryption-configuration '{
    "Rules": [{ "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "arn:aws:kms:ap-south-1:111122223333:key/abc" },
      "BucketKeyEnabled": true }]
  }'

# Per-object SSE-KMS on upload
aws s3 cp secret.txt s3://technova-learning-2024/ \
  --sse aws:kms --sse-kms-key-id alias/s3-demo-key

# SSE-C (customer-provided key)
aws s3 cp file.txt s3://technova-learning-2024/ \
  --sse-c AES256 --sse-c-key fileb://encryption-key.bin
```

### Enforce encryption with a bucket policy (deny unencrypted PUTs)
```json
{
  "Effect": "Deny", "Principal": "*",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::technova-learning-2024/*",
  "Condition": { "StringNotEquals": { "s3:x-amz-server-side-encryption": "aws:kms" } }
}
```

### DSSE-KMS — Dual-layer Server-Side Encryption
Applies **two independent layers** of KMS encryption to every object. Designed for workloads with strict mandates (e.g., US government / CNSSI compliance). Slight performance overhead vs SSE-KMS; supported only on **general-purpose buckets**.

```bash
# Upload with dual-layer KMS encryption
aws s3api put-object \
  --bucket technova-learning-2024 --key topsecret.dat --body topsecret.dat \
  --server-side-encryption aws:kms:dsse \
  --ssekms-key-id alias/s3-demo-key

# Or set it as the bucket default
aws s3api put-bucket-encryption \
  --bucket technova-learning-2024 \
  --server-side-encryption-configuration '{
    "Rules": [{ "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms:dsse",
      "KMSMasterKeyID": "alias/s3-demo-key" } }]
  }'
```
> 💡 Use DSSE-KMS **only** when a compliance rule demands dual-layer encryption — otherwise SSE-KMS is enough and cheaper/faster.

---

## 2.13 CORS

CORS lets a browser on **one domain** call your S3 bucket on **another domain**. Enforced by the browser; server-to-server calls don't need it.

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": ["https://www.technova.com"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

**🖱️ Console** — Bucket → **Permissions → Cross-origin resource sharing (CORS) → Edit** → paste JSON.

**💻 CLI**
```bash
aws s3api put-bucket-cors \
  --bucket technova-learning-2024 \
  --cors-configuration file://cors.json

aws s3api get-bucket-cors --bucket technova-learning-2024
```
> 💡 In DevTools → Network you'll see an `OPTIONS` **preflight** request. Use `"AllowedOrigins": ["*"]` only when truly public.

---

## 2.14 MFA Delete, Object Lock & Glacier Vault Lock

Both protect against deletion. **MFA Delete** needs an MFA token to permanently delete versions. **Object Lock** is WORM (Write Once Read Many) for compliance.

### MFA Delete (root + CLI only; versioning required)
```bash
aws s3api put-bucket-versioning \
  --bucket technova-learning-2024 \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::111122223333:mfa/root-account-mfa-device 123456"
```

### Object Lock (enable at bucket creation)
- **Governance mode:** users with `s3:BypassGovernanceRetention` can override.
- **Compliance mode:** nobody (not even root) can delete until retention ends.
- **Legal hold:** on/off, independent of retention.

**🖱️ Console** — Create bucket with **Object Lock enabled** → upload object → set retention mode + period or legal hold.

**💻 CLI**
```bash
# Bucket must be created with object lock enabled
aws s3api create-bucket --bucket technova-vault --region us-east-1 \
  --object-lock-enabled-for-bucket

# Apply retention to an object
aws s3api put-object-retention \
  --bucket technova-vault --key records.dat \
  --retention '{ "Mode": "GOVERNANCE", "RetainUntilDate": "2027-01-01T00:00:00Z" }'

# Legal hold
aws s3api put-object-legal-hold \
  --bucket technova-vault --key records.dat \
  --legal-hold Status=ON
```
> 💡 Use for SEC 17a-4 / HIPAA / PCI-DSS immutability requirements.

### Glacier Vault Lock (WORM for Glacier vaults)
Glacier **Vault Lock** enforces a WORM policy on an **S3 Glacier vault** that, once locked, **cannot be changed — even by the root user**. It's a two-step process with a 24-hour validation window: **initiate lock** (attach policy, get a lock ID) → test → **complete lock**.

```bash
# 1) Initiate the lock with a vault lock policy
aws glacier initiate-vault-lock \
  --account-id - --vault-name technova-archive \
  --policy file://vault-lock-policy.json

# 2) (Within 24h) complete the lock using the returned lockId — IRREVERSIBLE
aws glacier complete-vault-lock \
  --account-id - --vault-name technova-archive \
  --lock-id <lockId>

# Check status
aws glacier get-vault-lock --account-id - --vault-name technova-archive
```
> ⚠️ Once **completed**, the policy is permanent. Test thoroughly during the 24-hour window before completing. Use for regulatory retention (e.g., "keep for 7 years, no deletes").

---

## 2.15 Access Logs

Server access logs record every request to a bucket, delivered (best-effort, delayed) into **another** bucket. **Never log to the same bucket** (infinite loop).

**🖱️ Console** — Source bucket → **Properties → Server access logging → Edit → Enable** → target bucket + prefix `logs/`.

**💻 CLI**
```bash
# Target bucket needs a policy allowing the S3 logging service to write.
aws s3api put-bucket-logging \
  --bucket technova-learning-2024 \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "technova-logs-2024",
      "TargetPrefix": "logs/"
    }
  }'
```
> 💡 Analyze logs with **Amazon Athena** (SQL over the log files). For richer, faster auditing use **CloudTrail data events** instead.

---

## 2.16 Pre-signed URLs

A pre-signed URL grants **temporary** access to a private object — no AWS credentials needed by the recipient. Inherits the permissions of whoever generated it.

**🖱️ Console** — Object → **Object actions → Share with a presigned URL** → set duration → copy URL.

**💻 CLI**
```bash
# Download (GET) URL valid 5 minutes
aws s3 presign s3://technova-learning-2024/private.pdf --expires-in 300
```
```python
# Upload (PUT) URL via SDK (boto3) — common in real apps
import boto3
s3 = boto3.client("s3")
url = s3.generate_presigned_url(
    "put_object",
    Params={"Bucket": "technova-learning-2024", "Key": "uploads/file.png"},
    ExpiresIn=3600,
)
print(url)
```
> 💡 **Real-time pattern:** Backend generates a pre-signed **PUT** URL; the browser uploads **directly to S3**, skipping your server entirely. Avoids making buckets public.

---

## 2.17 Access Points & Object Lambda

**Access Points** = named endpoints, each with its own policy, simplifying access for shared buckets (per-team/app, optionally VPC-only). **Object Lambda** transforms objects **on the fly** during GET (redact PII, resize images, reformat) without storing copies.

### Access Point

**🖱️ Console** — Bucket → **Access Points → Create access point** → name → network (Internet/VPC) → policy.

**💻 CLI**
```bash
aws s3control create-access-point \
  --account-id 111122223333 \
  --name finance-ap \
  --bucket technova-company-data \
  --vpc-configuration VpcId=vpc-0abc123   # omit for internet access
```

### Object Lambda (high level)
1. Create a normal **Access Point** on the bucket.
2. Create a **Lambda** that receives the GetObject event and returns transformed bytes.
3. Create an **Object Lambda Access Point** linking the two.

```bash
aws s3control create-access-point-for-object-lambda \
  --account-id 111122223333 \
  --name redact-pii-olap \
  --configuration '{
    "SupportingAccessPoint": "arn:aws:s3:ap-south-1:111122223333:accesspoint/finance-ap",
    "TransformationConfigurations": [{
      "Actions": ["GetObject"],
      "ContentTransformation": { "AwsLambda": {
        "FunctionArn": "arn:aws:lambda:ap-south-1:111122223333:function:redact-pii" } }
    }]
  }'
```
> 💡 Use cases: redact `email`/`ssn` fields, dynamic image resizing, CSV→JSON on read, watermarking.

---

## 2.18 Storage Lens & Monitoring

**Storage Lens** = org-wide analytics dashboard (usage, activity, recommendations) across all buckets/accounts. Free tier = 28 metrics, 14-day retention.

**🖱️ Console** — **S3 → Storage Lens → Dashboards** → view the default dashboard or create a scoped one.

**💻 CLI**
```bash
aws s3control list-storage-lens-configurations --account-id 111122223333
aws s3control get-storage-lens-configuration \
  --account-id 111122223333 --config-id default-account-dashboard
```
Also watch **CloudWatch** S3 metrics (`BucketSizeBytes`, `NumberOfObjects`, request metrics) and set alarms/budgets.

> 💡 **Real-time:** Storage Lens surfaces "X TB not accessed in 90 days" → feed that into lifecycle rules to cut cost.

---

# Part 3 — Real-Time Integration Project

This is where Route 53 and S3 come together the way they're used **on the job**.

## 3.1 Secure HTTPS Static Site: S3 + CloudFront + ACM + Route 53

**Goal:** Serve `https://www.technova.com` from a **private** S3 bucket through CloudFront with a free TLS certificate and Route 53 DNS. This is the standard pattern for marketing sites, SPAs (React/Vue/Angular), and docs.

```
User → https://www.technova.com
     → Route 53 (Alias A record)
     → CloudFront (HTTPS via ACM cert, global CDN cache)
     → (Origin Access Control)
     → Private S3 bucket (website assets, encrypted, versioned)
```

### Why this design (vs plain S3 website)
| Plain S3 website | S3 + CloudFront |
|------------------|-----------------|
| HTTP only | **HTTPS** (ACM cert) |
| Bucket must be public | Bucket stays **private** (OAC) |
| No CDN | Global edge caching, lower latency |
| No WAF | Can attach **AWS WAF** |

### Step 1 — Create & populate a private bucket
```bash
aws s3 mb s3://technova-web-assets --region us-east-1
aws s3api put-bucket-versioning --bucket technova-web-assets \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket technova-web-assets \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
aws s3 sync ./dist s3://technova-web-assets/        # your built site
# Keep Block Public Access ON — CloudFront will read it via OAC.
```

### Step 2 — Request a TLS certificate (ACM)
> ⚠️ For CloudFront the certificate **must** be in **us-east-1**.

**🖱️ Console:** **ACM (us-east-1) → Request certificate** → public → add `technova.com` and `www.technova.com` → **DNS validation** → after issuing, click **Create records in Route 53** (auto-adds the CNAME validation records).

**💻 CLI:**
```bash
aws acm request-certificate --region us-east-1 \
  --domain-name technova.com \
  --subject-alternative-names www.technova.com \
  --validation-method DNS

# Describe to get the CNAME records you must add to Route 53, then UPSERT them
aws acm describe-certificate --region us-east-1 --certificate-arn <cert-arn>
```

### Step 3 — Create the CloudFront distribution with OAC
**🖱️ Console:** **CloudFront → Create distribution** → Origin = your S3 bucket → **Origin access: Origin access control (OAC)** → create OAC → set **Viewer protocol policy: Redirect HTTP to HTTPS** → **Alternate domain names (CNAMEs):** `technova.com`, `www.technova.com` → **Custom SSL certificate:** select the ACM cert → Default root object `index.html` → **Create**. Then click **Copy policy** and add it to the bucket so CloudFront can read it.

**💻 CLI (high level):**
```bash
# 1) Create an Origin Access Control
aws cloudfront create-origin-access-control \
  --origin-access-control-config \
  Name=technova-oac,SigningProtocol=sigv4,SigningBehavior=always,OriginAccessControlOriginType=s3

# 2) Create the distribution (full config in distribution.json)
aws cloudfront create-distribution --distribution-config file://distribution.json
```

### Step 4 — Let CloudFront read the private bucket (OAC policy)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontRead",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::technova-web-assets/*",
    "Condition": { "StringEquals": {
      "AWS:SourceArn": "arn:aws:cloudfront::111122223333:distribution/E123ABC" } }
  }]
}
```
```bash
aws s3api put-bucket-policy --bucket technova-web-assets --policy file://oac-policy.json
```

### Step 5 — Point Route 53 at CloudFront (Alias)
**🖱️ Console:** Hosted zone → **Create record** → name `www` · type `A` · **Alias** ON → *Alias to CloudFront distribution* → pick it → create. Repeat for the apex (blank name).

**💻 CLI:**
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456ABCDEF \
  --change-batch '{
    "Changes": [{ "Action": "UPSERT", "ResourceRecordSet": {
      "Name": "www.technova.com", "Type": "A",
      "AliasTarget": {
        "HostedZoneId": "Z2FDTNDATAQYW2",      
        "DNSName": "d123abc.cloudfront.net",
        "EvaluateTargetHealth": false } }}]
  }'
```
> `Z2FDTNDATAQYW2` is CloudFront's **fixed** alias hosted-zone ID (same for every CloudFront distribution).

### Step 6 — Verify
```bash
dig +short www.technova.com
curl -I https://www.technova.com          # expect HTTP/2 200, valid TLS

# After deploying new site content, invalidate the CDN cache:
aws cloudfront create-invalidation --distribution-id E123ABC --paths "/*"
```

> 💡 **Deploy update flow (real-time):** `aws s3 sync ./dist s3://technova-web-assets/ --delete` → `aws cloudfront create-invalidation --paths "/*"`.

---

## 3.2 Active-Passive Disaster Recovery

Combine **Route 53 failover** + **S3** so the site stays up even if the primary app dies.

```
Primary: www.technova.com → ALB/EC2 (health-checked)
Failure: www.technova.com → S3 static "we'll be back" site (or DR region)
```

1. Health check on the primary ALB/EC2 (1.11).
2. Failover **Primary** record → ALB (Alias), health check attached.
3. Failover **Secondary** record → S3 website endpoint (or a CloudFront site in another region).
4. Stop the primary → within ~60–90s DNS resolves to the S3 failover site.

```bash
# Quick test
aws route53 get-health-check-status --health-check-id <id>
dig +short www.technova.com   # flips to the secondary when primary is unhealthy
```
> 💡 For a **multi-region active-active** setup, swap failover for **latency** or **geolocation** routing, each region health-checked, with cross-region S3 replication keeping assets in sync.

---

# Part 4 — DevOps Automation (IaC & CI/CD)

In real projects you **don't** click the console for production — you codify everything. Here's the same work as **Terraform** plus a deploy pipeline.

## 4.1 Terraform for Route 53 + S3

```hcl
# providers.tf
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" {
  region = "ap-south-1"
}
# CloudFront/ACM must use us-east-1
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}
```

```hcl
# s3.tf — private, versioned, encrypted website-assets bucket
resource "aws_s3_bucket" "web" {
  bucket = "technova-web-assets"
}

resource "aws_s3_bucket_versioning" "web" {
  bucket = aws_s3_bucket.web.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "web" {
  bucket = aws_s3_bucket.web.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_s3_bucket_public_access_block" "web" {
  bucket                  = aws_s3_bucket.web.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "web" {
  bucket = aws_s3_bucket.web.id
  rule {
    id     = "expire-old-versions"
    status = "Enabled"
    noncurrent_version_expiration { noncurrent_days = 90 }
    abort_incomplete_multipart_upload { days_after_initiation = 7 }
  }
}
```

```hcl
# route53.tf — hosted zone + records + health check + failover
resource "aws_route53_zone" "main" {
  name = "technova.com"
}

# Simple A record
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.technova.com"
  type    = "A"
  ttl     = 300
  records = ["13.234.56.78"]
}

# Health check for the primary
resource "aws_route53_health_check" "primary" {
  ip_address        = "13.234.56.78"
  port              = 80
  type              = "HTTP"
  resource_path     = "/health"
  request_interval  = 30
  failure_threshold = 3
}

# Failover primary + secondary
resource "aws_route53_record" "failover_primary" {
  zone_id         = aws_route53_zone.main.zone_id
  name            = "app.technova.com"
  type            = "A"
  ttl             = 60
  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id
  records         = ["13.234.56.78"]
  failover_routing_policy { type = "PRIMARY" }
}

resource "aws_route53_record" "failover_secondary" {
  zone_id        = aws_route53_zone.main.zone_id
  name           = "app.technova.com"
  type           = "A"
  ttl            = 60
  set_identifier = "secondary"
  records        = ["3.91.10.20"]
  failover_routing_policy { type = "SECONDARY" }
}
```

```bash
# Standard Terraform workflow
terraform init       # download providers
terraform plan       # preview changes (review like a PR)
terraform apply      # create/update infra
terraform destroy    # tear everything down (clean lab)
```
> 💡 Store Terraform **state** in an S3 bucket with a DynamoDB lock table — the canonical remote-state pattern. Run `plan` in CI on every PR.

---

## 4.2 CI/CD Deploy Pipeline

Example **GitHub Actions** workflow that builds a site, syncs to S3, and invalidates CloudFront on every push to `main`.

```yaml
# .github/workflows/deploy.yml
name: Deploy Static Site
on:
  push:
    branches: [ main ]

permissions:
  id-token: write     # for OIDC (no long-lived keys!)
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC role)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::111122223333:role/github-deploy
          aws-region: us-east-1

      - name: Build
        run: |
          npm ci
          npm run build        # outputs ./dist

      - name: Sync to S3
        run: aws s3 sync ./dist s3://technova-web-assets/ --delete

      - name: Invalidate CloudFront cache
        run: aws cloudfront create-invalidation \
               --distribution-id E123ABC --paths "/*"
```
> 🔐 **Best practice:** Authenticate CI to AWS via **OIDC + an IAM role** (no stored access keys). Scope the role's policy to only the S3 bucket and CloudFront distribution it needs.

### Minimal scoped IAM policy for the pipeline
```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow",
      "Action": ["s3:PutObject","s3:DeleteObject","s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::technova-web-assets",
        "arn:aws:s3:::technova-web-assets/*"
      ] },
    { "Effect": "Allow",
      "Action": ["cloudfront:CreateInvalidation"],
      "Resource": "arn:aws:cloudfront::111122223333:distribution/E123ABC" }
  ]
}
```

---

## 4.3 Cleanup (Avoid Charges)

Run this after labs so you're not billed.

```bash
# --- S3 ---
aws s3 rm s3://technova-learning-2024 --recursive
aws s3 rb s3://technova-learning-2024            # remove bucket
aws s3 rb s3://technova-web-assets --force
aws s3 rb s3://failover.technova.com --force

# --- Route 53 ---
# Delete all records EXCEPT NS and SOA first, then:
aws route53 list-health-checks
aws route53 delete-health-check --health-check-id <id>
aws route53 delete-hosted-zone --id /hostedzone/Z123456ABCDEF

# --- CloudFront ---  (must disable, wait for "Deployed", then delete)
aws cloudfront get-distribution-config --id E123ABC   # grab ETag
# set Enabled=false, update-distribution, then:
aws cloudfront delete-distribution --id E123ABC --if-match <ETag>

# --- EC2 (each region) ---
aws ec2 terminate-instances --instance-ids i-0abc123
```

### Console cleanup checklist
1. Delete S3 objects → empty buckets → delete buckets.
2. Route 53: delete records (keep NS/SOA), health checks, traffic policies, CIDR collections, resolver rules/endpoints.
3. Disable + delete CloudFront distribution; delete ACM cert if unused.
4. Terminate EC2 in **every** region; release Elastic IPs; delete NAT gateways.
5. Delete hosted zone (only after non-NS/SOA records are gone).
6. Cancel domain registration only if you don't want the domain (refund within 5 days).

> 💰 With everything cleaned up, ongoing cost returns to **~$0** (a kept hosted zone is $0.50/month; a registered domain is ~$13/year).

---

## 📋 Cheat Sheets

### Route 53 routing policies at a glance
| Policy | Use case | Health checks | Records |
|--------|----------|---------------|---------|
| Simple | one endpoint / random multi-IP | ❌ | 1 |
| Weighted | A/B, canary, blue-green | optional | many |
| Latency | global, lowest latency | optional | 1/region |
| Failover | active-passive HA | required on primary | 2 |
| Geolocation | compliance, localization | optional | 1/location (+default) |
| Geoproximity | distance + bias | optional | many (Traffic Flow) |
| IP-based | CIDR steering | optional | many |
| Multi-Value | client-side LB + HA | recommended | up to 8 |

### S3 best practices
| Do | Why |
|----|-----|
| Keep **Block Public Access ON** | Avoid accidental data leaks |
| Enable **versioning** on prod | Recover from deletes/overwrites |
| Enable **default encryption** | Data protected at rest |
| Use **lifecycle rules** | Auto-tier + delete to cut cost |
| Abort incomplete multipart (7d) | Remove hidden billable junk |
| Serve via **CloudFront + OAC** | HTTPS + private bucket + CDN |
| **Pre-signed URLs** for sharing | No public buckets needed |
| Log via **CloudTrail data events** | Audit access |

### Most-used commands
```bash
# S3
aws s3 ls                                   # list buckets
aws s3 sync ./dist s3://bucket/ --delete    # deploy a site
aws s3 presign s3://bucket/file --expires-in 300

# Route 53
aws route53 list-hosted-zones
aws route53 list-resource-record-sets --hosted-zone-id <id>
aws route53 change-resource-record-sets --hosted-zone-id <id> --change-batch file://change.json

# DNS testing
dig +short www.technova.com
nslookup www.technova.com
curl -I https://www.technova.com
```

---

## 🗺️ Suggested Learning Path

```
Week 1 — Foundations + Route 53 basics
  Setup (IAM, CLI) → DNS → Hosted zones → Records → TTL → CNAME/Alias
Week 2 — Route 53 routing + health
  Simple/Weighted/Latency → Health checks → Failover → Geo/IP/Multi-value → Resolver/DNSSEC
Week 3 — S3 basics + security
  Buckets/Objects → Policies/BPA → Static hosting → Versioning → Encryption
Week 4 — S3 advanced + integration + DevOps
  Replication/Lifecycle/Events → Pre-signed/Access Points → S3+CloudFront+ACM+Route53 → Terraform + CI/CD
```

---

## 🔗 Useful Resources
- Route 53 docs: https://docs.aws.amazon.com/route53/
- S3 docs: https://docs.aws.amazon.com/s3/
- AWS CLI reference: https://docs.aws.amazon.com/cli/latest/reference/
- Terraform AWS provider: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- DNS propagation checker: https://dnschecker.org
- AWS Policy Generator: https://awspolicygen.s3.amazonaws.com/policygen.html

---

*Master guide — AWS Route 53 + S3 for real-time DevOps. Console + CLI + Terraform + CI/CD. Beginner → Advanced.*
