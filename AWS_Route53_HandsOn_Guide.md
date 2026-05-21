# 🌐 AWS Route 53 — Complete Hands-On Lab Guide

> **Goal:** Learn AWS Route 53 from scratch to advanced, using real AWS Console steps, hands-on exercises, and real-world startup-style scenarios.
> **Approach:** Each topic builds on the previous one. Follow in order for best results.

---

## 📋 Table of Contents

| # | Topic | Level |
|---|-------|-------|
| 1 | [What is DNS?](#1-what-is-dns) | 🟢 Beginner |
| 2 | [Route 53 Overview](#2-route-53-overview) | 🟢 Beginner |
| 3 | [Registering a Domain](#3-route-53---registering-a-domain) | 🟢 Beginner |
| 4 | [Creating First Records](#4-route-53---creating-our-first-records) | 🟢 Beginner |
| 5 | [EC2 Setup for DNS Testing](#5-route-53---ec2-setup) | 🟡 Intermediate |
| 6 | [TTL (Time To Live)](#6-route-53---ttl) | 🟡 Intermediate |
| 7 | [CNAME vs Alias](#7-route-53-cname-vs-alias) | 🟡 Intermediate |
| 8 | [Routing Policy – Simple](#8-routing-policy---simple) | 🟡 Intermediate |
| 9 | [Routing Policy – Weighted](#9-routing-policy---weighted) | 🟡 Intermediate |
| 10 | [Routing Policy – Latency](#10-routing-policy---latency) | 🟡 Intermediate |
| 11 | [Health Checks](#11-route-53---health-checks) | 🟡 Intermediate |
| 12 | [Health Checks Hands-On](#12-route-53---health-checks-hands-on) | 🟡 Intermediate |
| 13 | [Routing Policy – Failover](#13-routing-policy---failover) | 🔴 Advanced |
| 14 | [Routing Policy – Geolocation](#14-routing-policy---geolocation) | 🔴 Advanced |
| 15 | [Routing Policy – Geoproximity](#15-routing-policy---geoproximity) | 🔴 Advanced |
| 16 | [Routing Policy – IP-Based](#16-routing-policy---ip-based) | 🔴 Advanced |
| 17 | [Routing Policy – Multi-Value](#17-routing-policy---multi-value) | 🔴 Advanced |
| 18 | [3rd Party Domains & Route 53](#18-3rd-party-domains--route-53) | 🔴 Advanced |
| 19 | [Route 53 Resolvers & Hybrid DNS](#19-route-53-resolvers--hybrid-dns) | 🔴 Advanced |
| 20 | [Section Cleanup](#20-route-53---section-cleanup) | 🟢 All Levels |

---

## 🏗️ Scenario Setup (Use Throughout This Guide)

> **Startup Context:** You're the DevOps engineer at **TechNova Pvt. Ltd.**, a SaaS startup with users across India, Europe, and the US. Your job is to set up production-grade DNS using AWS Route 53.

**What you'll build:**
- A registered domain (or use a free `.com` subdomain trick)
- DNS records pointing to EC2 instances in multiple regions
- Health checks + failover routing
- Geolocation-based traffic routing for global users

---

## 1. What is DNS?

### 📖 Concept

DNS (Domain Name System) is the phonebook of the internet. It translates human-readable domain names into IP addresses that computers use to communicate.

```
User types: www.technova.com
DNS resolves: 13.234.56.78  ← EC2 Public IP
```

### DNS Resolution Flow

```
Browser → Recursive Resolver (ISP)
         → Root Name Server (.)
         → TLD Name Server (.com)
         → Authoritative Name Server (Route 53)
         → Returns IP → Browser connects
```

### Key DNS Terms

| Term | Meaning | Example |
|------|---------|---------|
| **Domain** | Human-readable name | `technova.com` |
| **FQDN** | Fully Qualified Domain Name | `www.technova.com.` |
| **TLD** | Top-Level Domain | `.com`, `.in`, `.io` |
| **Subdomain** | Prefix to domain | `api.technova.com` |
| **Record** | DNS mapping entry | A, CNAME, MX, TXT |
| **Resolver** | Translates DNS queries | Your ISP or `8.8.8.8` |
| **Zone** | DNS namespace for a domain | Hosted Zone in Route 53 |
| **TTL** | Cache duration in seconds | `300` = 5 minutes |

### Common DNS Record Types

| Record | Purpose | Example |
|--------|---------|---------|
| **A** | Maps domain → IPv4 | `app.technova.com → 13.234.56.78` |
| **AAAA** | Maps domain → IPv6 | `app.technova.com → 2001:db8::1` |
| **CNAME** | Alias to another domain | `www → app.technova.com` |
| **MX** | Mail exchange | Routes email |
| **TXT** | Text data (SPF, DKIM, verification) | `"v=spf1 include:..."` |
| **NS** | Name servers for the domain | Route 53's NS servers |
| **SOA** | Start of Authority | Zone metadata |
| **SRV** | Service locator | Kubernetes, SIP |
| **PTR** | Reverse DNS (IP → name) | `78.56.234.13.in-addr.arpa` |

### 💡 Tips
- DNS is **hierarchical** — changes propagate from authoritative servers outward
- Always check propagation using: `nslookup`, `dig`, or https://dnschecker.org
- Low TTL = faster propagation but more DNS queries (more cost)

---

## 2. Route 53 Overview

### 📖 Concept

AWS Route 53 is a **highly available, scalable, fully managed DNS and domain registrar** service. It's the only AWS service with a **100% SLA**.

### Why "Route 53"?
Port 53 is the standard DNS port. AWS named it "Route 53" as a play on that.

### Route 53 — What It Does

```
┌─────────────────────────────────────────┐
│              AWS Route 53               │
├─────────────┬──────────────┬────────────┤
│  Domain     │  DNS         │  Health    │
│  Registrar  │  Service     │  Checks    │
│  (buy .com) │  (A, CNAME)  │  (monitor) │
└─────────────┴──────────────┴────────────┘
```

### Route 53 — Core Components

| Component | Description |
|-----------|-------------|
| **Hosted Zone** | Container for DNS records of a domain |
| **Public Hosted Zone** | DNS for internet-facing domains |
| **Private Hosted Zone** | DNS inside a VPC (internal services) |
| **Record Set** | Individual DNS mapping |
| **Routing Policy** | Controls how traffic is routed |
| **Health Check** | Monitors endpoints; used in routing decisions |
| **Traffic Policy** | Advanced routing with visual flow editor |

### Pricing (Important for SAA-C03 Exam)

| Item | Cost |
|------|------|
| Hosted Zone | $0.50/month per zone |
| DNS Queries (Standard) | $0.40 per million |
| Health Checks (Basic) | $0.50/month per endpoint |
| Domain Registration | Varies by TLD (`.com` = ~$13/year) |

### 🧪 Console Exploration Exercise
1. Go to **AWS Console → Route 53**
2. Explore the left sidebar: **Hosted Zones**, **Health Checks**, **Traffic Policies**, **Resolver**
3. Note the **Dashboard** shows your domain count, hosted zones, and health checks

---

## 3. Route 53 — Registering a Domain

### 📖 Concept

Route 53 can act as your **domain registrar** — you can buy and manage domains directly from AWS.

> ⚠️ **Cost Note:** Domain registration costs money (~$13/year for `.com`). For practice, use an existing domain you own OR use the **free subdomain trick** described below.

### Option A: Register a New Domain (Paid)

**Steps in AWS Console:**

1. Go to **Route 53 → Registered Domains → Register Domain**
2. Search for a domain (e.g., `technova-labs.com`)
3. Add to cart → Fill contact information
4. Enable/disable **Auto-Renew**
5. Check the **Privacy Protection** box (hides WHOIS info)
6. Complete purchase → **Wait 10–15 minutes** for activation
7. Route 53 automatically creates a **Hosted Zone** with NS and SOA records

### Option B: Free Practice (Recommended for Learning)

> Use a **free subdomain from freenom.com** (.tk, .ml, .ga domains) OR use an existing owned domain and delegate a subdomain to Route 53.

**Subdomain delegation approach (if you own a domain):**
```
Parent domain: yourdomain.com  (at GoDaddy/Namecheap)
Practice zone:  aws.yourdomain.com  (Route 53)
```

### 🧪 Hands-On Exercise

**Goal:** See how Route 53 auto-creates NS/SOA records after domain registration.

1. Go to **Route 53 → Hosted Zones**
2. Click your domain's hosted zone
3. Observe the **2 default records**:
   - `NS` — 4 Route 53 name servers assigned to your domain
   - `SOA` — Authoritative record for the zone

**Important:** The 4 NS records shown are what you'd paste into a registrar to point a domain at Route 53.

```bash
# Verify NS records from your terminal
nslookup -type=NS technova.com
# OR
dig NS technova.com
```

### 💡 Tips
- NS propagation after registration: up to **48 hours** (usually faster)
- Never delete the **NS records** from your hosted zone — you'll break DNS
- Keep **Privacy Protection** ON to avoid spam from WHOIS scraping

---

## 4. Route 53 — Creating Our First Records

### 📖 Concept

A **Record Set** inside a Hosted Zone is a DNS mapping. Let's create A records and test resolution.

### 🧪 Hands-On Exercise — Create an A Record

**Pre-req:** Have an EC2 instance running with a public IP (we'll set this up in Topic 5 properly, but you can use any public IP for now).

**Steps:**

1. Go to **Route 53 → Hosted Zones → [Your Domain]**
2. Click **Create Record**
3. Fill in:
   ```
   Record name:  www
   Record type:  A
   Value:        <EC2 Public IP e.g., 13.234.56.78>
   TTL:          300
   Routing:      Simple
   ```
4. Click **Create Records**

**Expected Result:**
```
www.technova.com → 13.234.56.78
```

**Test the record:**
```bash
# From your terminal:
nslookup www.technova.com
dig www.technova.com
dig +short www.technova.com

# Windows:
Resolve-DnsName www.technova.com
```

### 🧪 Create Additional Records

| Record | Type | Value | Purpose |
|--------|------|-------|---------|
| `@` or blank | A | EC2 IP | Root domain (technova.com) |
| `www` | A | EC2 IP | www subdomain |
| `api` | A | EC2 IP | API endpoint |
| `mail` | MX | `10 mail.technova.com` | Email routing |

### 💡 Tips
- **Record name blank = root domain** (`technova.com`)
- `@` in some tools also means root domain
- Route 53 has **no concept of wildcard A records** for routing policies (only simple)
- TXT records are commonly used for **domain verification** (Google Workspace, SSL certs)

---

## 5. Route 53 — EC2 Setup

### 📖 Concept

For DNS testing, we need running EC2 instances. We'll launch instances in **multiple regions** to demonstrate latency-based and geolocation routing later.

### 🧪 Hands-On — Launch EC2 Instances in Multiple Regions

**We'll use 3 regions:**
- `ap-south-1` (Mumbai) — for Indian users
- `eu-west-1` (Ireland) — for European users
- `us-east-1` (N. Virginia) — for US users

**Steps for each region:**

1. Switch to the target region using the top-right dropdown
2. Go to **EC2 → Launch Instance**
3. Configure:
   ```
   AMI:           Amazon Linux 2023 (free tier)
   Instance Type: t2.micro (free tier)
   Key Pair:      Create or select existing
   Security Group: Allow HTTP (80), HTTPS (443), SSH (22)
   ```
4. **User Data script** (auto-starts a web server showing region):
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/region)
   PUBLIC_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
   echo "<h1>TechNova App - Region: $REGION</h1><p>IP: $PUBLIC_IP</p>" > /var/www/html/index.html
   ```
5. Launch and note the **Public IPv4 address** of each instance

**Record your IPs:**
```
Mumbai  (ap-south-1):  ________________
Ireland (eu-west-1):   ________________
Virginia (us-east-1):  ________________
```

### 🧪 Verify Web Servers
```bash
curl http://<mumbai-ip>
curl http://<ireland-ip>
curl http://<virginia-ip>
```
Each should return its region name.

### 💡 Tips
- **IMDSv2** may be required in newer Amazon Linux — if curl fails, use:
  ```bash
  TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  REGION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/region)
  ```
- Make sure **Security Groups** allow port 80 from `0.0.0.0/0`
- Tag instances: `Name = technova-mumbai`, `Name = technova-ireland`, etc.

---

## 6. Route 53 — TTL (Time To Live)

### 📖 Concept

**TTL** tells DNS resolvers **how long to cache** a record before re-querying. It's set in seconds.

```
TTL = 300  → cache for 5 minutes
TTL = 3600 → cache for 1 hour
TTL = 86400 → cache for 24 hours
```

### TTL Trade-offs

| TTL | Pros | Cons |
|-----|------|------|
| **Low (60s)** | Faster propagation of changes | More DNS queries → higher cost |
| **High (86400s)** | Fewer queries → cheaper | Slow to propagate IP changes |

### TTL in Production — Strategy

```
Normal operation:   TTL = 3600 (1 hour)  ← balance of cost and flexibility
Before IP change:   TTL = 60  (1 min)    ← reduce before planned change
After IP change:    TTL = 3600 again     ← restore after propagation
```

### 🧪 Hands-On Exercise — Observe TTL in Action

1. Create an A record for `test.yourdomain.com` with TTL = **60** pointing to Mumbai IP
2. Query it:
   ```bash
   dig test.yourdomain.com
   # Note the "ANSWER SECTION" shows TTL countdown
   ```
3. Change the IP to the Ireland EC2
4. Run `dig` again — notice how long before the new IP resolves (up to 60s)
5. Change TTL to `300` and repeat

**Reading TTL in dig output:**
```
;; ANSWER SECTION:
test.yourdomain.com.  287  IN  A  13.234.56.78
                      ^^^
                      This number decreases — it's the remaining TTL
```

### 💡 Tips
- **Alias records have no TTL** — Route 53 manages it internally
- When migrating servers, **lower TTL 24 hours before** the change
- Route 53 minimum TTL is **0** seconds (not recommended)

---

## 7. Route 53 CNAME vs Alias

### 📖 Concept

Both CNAME and Alias map one name to another, but they behave differently. This is a **critical concept** for the AWS SAA-C03 exam.

### Comparison Table

| Feature | CNAME | Alias |
|---------|-------|-------|
| Maps to | Any domain name | AWS resource or zone apex |
| Works at root domain | ❌ No | ✅ Yes |
| Works with AWS resources | ❌ No (use Alias) | ✅ Yes (ELB, CloudFront, S3, etc.) |
| DNS query charge | ✅ Charged per query | ❌ Free for AWS resources |
| TTL | Set by you | Managed by Route 53 |
| Standard RFC | ✅ Yes (RFC 1035) | ❌ Route 53 extension |

### CNAME — When to Use

```
www.technova.com  →  app.technova.com   ✅ Valid CNAME
api.technova.com  →  prod.elb.amazonaws.com  ❌ Use Alias instead
technova.com      →  www.technova.com   ❌ CANNOT use CNAME at root
```

### Alias — When to Use

```
technova.com      → my-lb.ap-south-1.elb.amazonaws.com  ✅ Alias (root domain!)
www.technova.com  → my-cf-dist.cloudfront.net           ✅ Alias
api.technova.com  → my-api.execute-api.amazonaws.com    ✅ Alias
```

### AWS Resources That Support Alias

- Elastic Load Balancer (ALB, NLB, CLB)
- CloudFront distributions
- API Gateway
- S3 static website endpoints
- Elastic Beanstalk
- VPC Interface Endpoints
- **Another Route 53 record in the same hosted zone**

### 🧪 Hands-On Exercise

**Create a CNAME:**
1. Route 53 → Hosted Zone → Create Record
2. Record: `www`, Type: `CNAME`, Value: `app.technova.com`, TTL: 300

**Create an Alias:**
1. Route 53 → Hosted Zone → Create Record
2. Record: `app`, Type: `A`
3. Toggle **Alias = ON**
4. Choose: **Alias to Application and Classic Load Balancer**
5. Region: `ap-south-1`, choose your ALB

**Verify:**
```bash
dig www.technova.com
dig app.technova.com
# CNAME shows intermediate step; Alias resolves directly to IP
```

### 💡 Tips
- **Root domain (`@`) MUST use Alias** if pointing to an AWS resource
- CNAME cannot point to an IP address — that's what A records are for
- Alias is a Route 53-only concept — not portable to other DNS providers

---

## 8. Routing Policy — Simple

### 📖 Concept

**Simple routing** is the default policy — one record, one or multiple IPs. If multiple IPs are listed, Route 53 returns them all and the **client randomly picks one** (no health checks).

```
Request → Route 53 → Returns all IPs → Client picks one
```

### 🧪 Hands-On Exercise

1. Go to **Route 53 → Hosted Zone → Create Record**
2. Record name: `simple`
3. Type: `A`
4. Values (add multiple):
   ```
   13.234.56.78   (Mumbai)
   52.30.12.34    (Ireland)
   ```
5. TTL: `60`
6. Routing Policy: **Simple**

**Test multiple times:**
```bash
for i in {1..5}; do dig +short simple.yourdomain.com; done
# You'll see both IPs returned each time (client load balancing)
```

### When to Use Simple

| ✅ Use When | ❌ Don't Use When |
|------------|-----------------|
| Single endpoint | Need failover |
| Testing/development | Need geographic routing |
| No health check needed | Need load distribution by weight |

### 💡 Tips
- Simple routing with multiple values has **no health checks** — if one IP is down, clients may still hit it
- For health-aware multi-value routing → use **Multi-Value routing** instead

---

## 9. Routing Policy — Weighted

### 📖 Concept

Weighted routing lets you send **different percentages of traffic** to different resources. Useful for **blue-green deployments** and **A/B testing**.

```
Weight 70 → Mumbai EC2 (70% of traffic)
Weight 30 → Ireland EC2 (30% of traffic)
Weight 0  → Excluded from rotation
```

**Formula:**
```
Traffic % = (Record Weight) / (Sum of All Weights) × 100
```

### 🧪 Hands-On Exercise — Canary Deployment (80/20 Split)

> Scenario: You're deploying v2 of the TechNova app. Route 80% to stable v1, 20% to new v2.

**Create Record 1 (v1 — Mumbai):**
1. Create Record → `weighted.yourdomain.com`
2. Type: A, Value: `<Mumbai IP>`
3. Routing Policy: **Weighted**
4. Weight: **80**
5. Record ID: `v1-mumbai`

**Create Record 2 (v2 — Ireland):**
1. Create Record → `weighted.yourdomain.com` (same name!)
2. Type: A, Value: `<Ireland IP>`
3. Routing Policy: **Weighted**
4. Weight: **20**
5. Record ID: `v2-ireland`

**Test:**
```bash
for i in {1..10}; do dig +short weighted.yourdomain.com; sleep 1; done
# ~8 times you should get Mumbai IP, ~2 times Ireland IP
```

### Use Cases

| Use Case | Weight Config |
|----------|---------------|
| Blue-Green deployment | 100/0 → slowly shift to 0/100 |
| A/B Testing | 50/50 |
| Gradual canary rollout | 95/5 → 80/20 → 50/50 → 0/100 |
| Disable a region | Set weight to 0 |

### 💡 Tips
- Weight `0` = record is **excluded** (not deleted)
- If ALL weights are 0, Route 53 returns them equally
- Can combine with **Health Checks** — if unhealthy, traffic shifts to others
- Record IDs must be **unique** per weighted record group

---

## 10. Routing Policy — Latency

### 📖 Concept

Latency-based routing directs users to the **AWS region with the lowest latency** for that user. Route 53 uses a latency database (not real-time pings) to make this decision.

```
User in Mumbai    → ap-south-1 EC2  (lowest latency for India)
User in London    → eu-west-1 EC2   (lowest latency for Europe)
User in New York  → us-east-1 EC2  (lowest latency for US)
```

### 🧪 Hands-On Exercise

**Create 3 records for the same hostname across regions:**

**Record 1 — Mumbai:**
1. Create Record → `app.yourdomain.com`
2. Type: A, Value: `<Mumbai IP>`
3. Routing Policy: **Latency**
4. Region: `ap-south-1`
5. Record ID: `latency-mumbai`

**Record 2 — Ireland:**
1. Same hostname `app.yourdomain.com`
2. Value: `<Ireland IP>`
3. Routing Policy: **Latency**
4. Region: `eu-west-1`
5. Record ID: `latency-ireland`

**Record 3 — Virginia:**
1. Same hostname `app.yourdomain.com`
2. Value: `<Virginia IP>`
3. Routing Policy: **Latency**
4. Region: `us-east-1`
5. Record ID: `latency-virginia`

**Test:**
```bash
dig app.yourdomain.com
# From India → should return Mumbai IP
# Use VPN to test from other regions
```

### 💡 Tips
- Route 53 uses **historical latency data**, not real-time measurement
- Best for **globally distributed applications**
- Can combine with health checks for **automatic failover**
- Doesn't consider user's actual location — only **network latency**

---

## 11. Route 53 — Health Checks

### 📖 Concept

Route 53 Health Checks **monitor the health of your endpoints**. When used with routing policies, Route 53 automatically stops routing to unhealthy endpoints.

### Health Check Types

| Type | Description |
|------|-------------|
| **Endpoint** | Monitor an IP or domain (HTTP, HTTPS, TCP) |
| **Calculated** | Combine multiple health checks (AND/OR logic) |
| **CloudWatch Alarm** | Based on CloudWatch metric threshold |

### Health Check — Endpoint Config

```
Protocol:         HTTP / HTTPS / TCP
IP or Domain:     13.234.56.78 or app.technova.com
Port:             80 / 443 / custom
Path:             /health  (recommended endpoint to check)
Interval:         30s (standard) or 10s (fast — costs more)
Failure Threshold: 3 (marks unhealthy after 3 consecutive failures)
```

### Health Checker Locations

Route 53 uses **15+ global health checker locations**. Your endpoint is healthy only if the majority of checkers agree it's up.

> ⚠️ Make sure your **Security Groups allow Route 53 health checker IPs** or use a public endpoint.

### Calculated Health Checks

```
Parent Check = (DB check AND App check AND Cache check)
```
If any child check fails, the parent check fails — useful for composite service health.

### 🧪 Concept Exercise

Think about your TechNova app:
```
/health endpoint returns:
  200 OK  → "healthy"
  500     → "unhealthy"
  timeout → "unhealthy"
```

Design health checks for:
- App server (HTTP /health)
- Database port (TCP 5432)
- Combined check (app + db must both pass)

---

## 12. Route 53 — Health Checks Hands-On

### 🧪 Step 1: Add a /health endpoint to EC2

SSH into your Mumbai EC2 and add a health endpoint:
```bash
# Add health check page
echo "OK - TechNova Mumbai Healthy" > /var/www/html/health
# Verify
curl http://localhost/health
```

### 🧪 Step 2: Create Health Check in Route 53

1. Go to **Route 53 → Health Checks → Create Health Check**
2. Configure:
   ```
   Name:            technova-mumbai-health
   What to monitor: Endpoint
   Specify by:      IP address
   Protocol:        HTTP
   IP Address:      <Mumbai EC2 Public IP>
   Port:            80
   Path:            /health
   ```
3. Advanced:
   ```
   Interval:             30 seconds
   Failure threshold:    3
   String matching:      Enable → type "OK" (Route 53 checks response body)
   ```
4. Create health check
5. Wait 60–90 seconds → Status should show **Healthy** ✅

### 🧪 Step 3: Simulate Failure

```bash
# On Mumbai EC2 — stop the web server
sudo systemctl stop httpd

# Watch Route 53 health check status change to Unhealthy
# (may take 30–90s)
```

Go to **Route 53 → Health Checks** and watch the status change from ✅ Healthy to ❌ Unhealthy.

```bash
# Restore
sudo systemctl start httpd
```

### 🧪 Step 4: Link Health Check to DNS Record

1. Edit your Mumbai A record
2. Set **Health Check ID** = `technova-mumbai-health`
3. Save

Now if the health check fails, Route 53 **stops routing to that record**.

### 💡 Tips
- Health checkers come from specific AWS IP ranges — whitelist them or use a **public endpoint**
- CloudWatch alarms can trigger health check state changes for **private resources**
- Health checks cost **$0.50/month per endpoint** (basic), **$1.00/month** for HTTPS

---

## 13. Routing Policy — Failover

### 📖 Concept

Failover routing requires **exactly 2 records**: one PRIMARY, one SECONDARY. Traffic goes to primary unless it fails its health check, then Route 53 automatically routes to secondary.

```
Normal:   User → PRIMARY (Mumbai EC2)
Failure:  User → SECONDARY (S3 static site or another region)
```

### 🧪 Hands-On Exercise — Active-Passive Failover

**Pre-req:** Health Check must exist for the primary endpoint (from Topic 12).

**Step 1: Create Primary Record**
1. Create Record → `failover.yourdomain.com`
2. Type: A, Value: `<Mumbai IP>`
3. Routing Policy: **Failover**
4. Failover record type: **Primary**
5. Health Check: select `technova-mumbai-health`
6. Record ID: `failover-primary`

**Step 2: Create Secondary Record (S3 static failover page)**

First, create an S3 bucket with a static "Maintenance Mode" page:
```bash
# Create S3 bucket (same name as subdomain)
aws s3 mb s3://failover.yourdomain.com --region ap-south-1

# Enable static website hosting
aws s3 website s3://failover.yourdomain.com --index-document index.html

# Upload maintenance page
echo "<h1>TechNova - Maintenance Mode</h1><p>We'll be back shortly!</p>" > index.html
aws s3 cp index.html s3://failover.yourdomain.com/ --acl public-read
```

Back in Route 53:
1. Create Record → `failover.yourdomain.com` (same name)
2. Type: A → **Alias → S3 website endpoint**
3. Routing Policy: **Failover**
4. Failover record type: **Secondary**
5. Record ID: `failover-secondary`
6. **No health check required on secondary**

**Step 3: Test Failover**
```bash
# Normal
dig failover.yourdomain.com  # Returns Mumbai IP

# Stop the web server on Mumbai EC2
sudo systemctl stop httpd

# After 30-90s, query again
dig failover.yourdomain.com  # Now returns S3 endpoint
```

### 💡 Tips
- Primary MUST have a health check; secondary is optional
- Secondary can be an S3 bucket, another region, or any valid endpoint
- Failover works with **private hosted zones** too (for VPC internal failover)

---

## 14. Routing Policy — Geolocation

### 📖 Concept

Geolocation routing routes based on the **user's geographic location** (continent, country, or US state). You define which records serve which locations.

> ⚠️ Different from Latency routing — Geolocation is based on **where the user IS**, not which region is **fastest**.

### 🧪 Hands-On Exercise — Region-Specific App Versions

> Scenario: TechNova serves different content for Indian users (Hindi UI), European users (GDPR compliance), and everyone else (global default).

**Record 1 — India:**
1. Create Record → `geo.yourdomain.com`
2. Type: A, Value: `<Mumbai IP>`
3. Routing Policy: **Geolocation**
4. Location: **Asia** → **India**
5. Record ID: `geo-india`

**Record 2 — Europe:**
1. Same hostname `geo.yourdomain.com`
2. Value: `<Ireland IP>`
3. Routing Policy: **Geolocation**
4. Location: **Europe**
5. Record ID: `geo-europe`

**Record 3 — Default (MUST create this):**
1. Same hostname `geo.yourdomain.com`
2. Value: `<Virginia IP>`
3. Routing Policy: **Geolocation**
4. Location: **Default**
5. Record ID: `geo-default`

> ⚠️ **Always create a Default record.** Without it, users from unlisted locations get no response.

**Test:**
```bash
# From India (or via VPN)
dig geo.yourdomain.com  # → Mumbai IP

# From Europe VPN
dig geo.yourdomain.com  # → Ireland IP
```

### Geolocation vs Geoproximity

| | Geolocation | Geoproximity |
|--|-------------|--------------|
| Basis | Country/Continent | Physical distance + bias |
| Flexibility | Fixed geographic zones | Adjustable bias (+/- up to 99) |
| Use case | Legal compliance, localization | Optimization of traffic split |

### 💡 Tips
- Geolocation is great for **data sovereignty** and **regulatory compliance** (GDPR)
- Works well for **multi-language apps** where content differs by country
- Users outside defined regions go to **Default** — always define it

---

## 15. Routing Policy — Geoproximity

### 📖 Concept

Geoproximity routing routes traffic based on **physical distance** between users and AWS resources. You can adjust this with a **bias** value (+99 to -99) to expand or shrink the region's coverage.

```
Bias +50 → Expand region coverage (attract more users)
Bias -50 → Shrink region coverage (push users to other regions)
Bias 0   → Pure geographic distance
```

### Geoproximity — Bias Explained

```
      Without Bias              With Bias (+50 on Mumbai)
   EU ←→ Mumbai: 50%           EU ←→ Mumbai: 70%
   EU ←→ Virginia: 50%         EU ←→ Virginia: 30%
```

### 🧪 Hands-On Exercise — Traffic Policy (Required for Geoproximity)

Geoproximity requires **Route 53 Traffic Policies** (not available in simple record creation).

1. Go to **Route 53 → Traffic Policies → Create Traffic Policy**
2. Name: `technova-geoproximity`
3. DNS Type: `A`
4. In the visual editor:
   - Add **Geoproximity Rule**
   - Add endpoint: `ap-south-1` → Mumbai EC2 IP, Bias: `+20`
   - Add endpoint: `eu-west-1` → Ireland EC2 IP, Bias: `0`
   - Add endpoint: `us-east-1` → Virginia EC2 IP, Bias: `0`
5. Connect rule to endpoints
6. Create policy → Create Policy Record
7. DNS Name: `geopx.yourdomain.com`, TTL: 60

**Observe the Coverage Map** — AWS shows a visual map of which region covers which geographic area.

### 💡 Tips
- **Traffic Flow editor** gives a visual canvas for complex routing logic
- Useful when you want to **gradually shift traffic** between regions
- Combine with **health checks** for automatic failover

---

## 16. Routing Policy — IP-Based

### 📖 Concept

IP-Based routing routes traffic based on the **CIDR block (IP range)** of the user's network. This is the most granular routing option — you control routing based on specific IP ranges.

```
Users from IP range 203.0.113.0/24  → Mumbai EC2
Users from IP range 198.51.100.0/24 → Virginia EC2
All others                          → Default
```

### Use Cases

- Route **corporate office users** (known IP range) to internal/private resources
- Route **ISP-specific traffic** to the closest datacenter
- Separate **test users** (VPN IP) from production users

### 🧪 Hands-On Exercise

**Step 1: Create a CIDR Collection**
1. Go to **Route 53 → CIDR Collections → Create CIDR Collection**
2. Name: `technova-corp-ips`
3. Add CIDR location:
   ```
   Location name: technova-office-mumbai
   CIDRs:         103.21.58.0/24  (example corporate IP range)
   ```

**Step 2: Create IP-Based Records**
1. Create Record → `ipbased.yourdomain.com`
2. Type: A, Value: `<Mumbai IP>`
3. Routing Policy: **IP-based**
4. CIDR Collection: `technova-corp-ips`
5. CIDR Location: `technova-office-mumbai`
6. Record ID: `ipbased-corp`

**Step 3: Create Default Record**
1. Same hostname `ipbased.yourdomain.com`
2. Value: `<Virginia IP>`
3. Routing Policy: **IP-based**
4. CIDR Location: **Default**
5. Record ID: `ipbased-default`

### 💡 Tips
- IP-based routing overrides geolocation (more specific)
- Max **1,000 CIDRs per collection**, **5 collections per hosted zone**
- Great for **ISP/carrier-level traffic optimization** (used by large enterprises)

---

## 17. Routing Policy — Multi-Value

### 📖 Concept

Multi-Value routing returns **up to 8 healthy A records** per query. Unlike Simple routing (which returns all IPs including unhealthy ones), Multi-Value only returns **healthy endpoints**.

```
Route 53 has: Mumbai (✅), Ireland (✅), Virginia (❌ unhealthy)
Multi-Value returns: Mumbai IP, Ireland IP  (skips Virginia)
```

> Multi-Value is **NOT** a replacement for a load balancer — it's client-side load balancing with basic health awareness.

### Simple vs Multi-Value Comparison

| Feature | Simple (Multi-IP) | Multi-Value |
|---------|-----------------|-------------|
| Health checks | ❌ No | ✅ Yes |
| Returns unhealthy IPs | ✅ Yes (bad!) | ❌ No |
| Max IPs returned | All | 8 |
| Use case | Testing | Basic HA |

### 🧪 Hands-On Exercise

**Create 3 Multi-Value records:**

**Record 1:**
1. Create Record → `multi.yourdomain.com`
2. Type: A, Value: `<Mumbai IP>`
3. Routing Policy: **Multivalue answer**
4. Health Check: `technova-mumbai-health`
5. Record ID: `multi-mumbai`

**Record 2:**
1. Same hostname `multi.yourdomain.com`
2. Value: `<Ireland IP>`
3. Health Check: (create health check for Ireland too)
4. Record ID: `multi-ireland`

**Record 3:**
1. Same hostname `multi.yourdomain.com`
2. Value: `<Virginia IP>`
3. Health Check: (create health check for Virginia)
4. Record ID: `multi-virginia`

**Test:**
```bash
# Query multiple times
dig multi.yourdomain.com
# Should return all 3 IPs (all healthy)

# Stop one EC2 web server
sudo systemctl stop httpd  # on Virginia EC2

# After health check marks it unhealthy (~60s):
dig multi.yourdomain.com
# Now returns only 2 IPs
```

### 💡 Tips
- Each record in Multi-Value must have its **own health check**
- Records **without health checks** are always returned (treat as always healthy)
- Maximum **8 records** per Multi-Value group

---

## 18. 3rd Party Domains & Route 53

### 📖 Concept

You can use Route 53 as your DNS provider even if your domain was registered at **GoDaddy, Namecheap, Google Domains**, etc. You just update the **NS records** at your registrar to point to Route 53 name servers.

```
Domain registered at: GoDaddy
DNS managed by:       Route 53

GoDaddy NS records → route53-ns1.awsdns.com
                      route53-ns2.awsdns.net
                      route53-ns3.awsdns.org
                      route53-ns4.awsdns.info
```

### 🧪 Hands-On Exercise — Delegate External Domain to Route 53

**Step 1: Create Hosted Zone in Route 53**
1. Route 53 → Hosted Zones → Create Hosted Zone
2. Domain name: `yourdomain.com` (domain you own elsewhere)
3. Type: Public Hosted Zone
4. **Note the 4 NS server values** shown in the NS record

**Step 2: Update NS at Your Registrar**

At GoDaddy/Namecheap:
1. Go to Domain Settings → Name Servers
2. Choose "Custom Name Servers"
3. Paste the 4 Route 53 NS values
4. Save

**Step 3: Wait for Propagation**
```bash
# Check NS propagation
dig NS yourdomain.com
# Should show Route 53 NS values after propagation (up to 48 hrs)

# Faster check:
nslookup -type=NS yourdomain.com 8.8.8.8
```

**Step 4: Add Records in Route 53**

Once NS is delegated, all DNS records must be managed in Route 53. Add:
- A record for `www`
- MX record for email
- TXT record for domain verification

### Subdomain Delegation

You can also delegate only a **subdomain** to Route 53:

```
technova.com     → Namecheap DNS (keep here)
aws.technova.com → Route 53 (delegate only this subdomain)
```

Steps:
1. Create Route 53 hosted zone for `aws.technova.com`
2. Note the NS records
3. At Namecheap, add NS records for `aws` subdomain pointing to Route 53
4. Route 53 manages `aws.technova.com` and all sub-subdomains

### 💡 Tips
- **Never delete the SOA/NS records** in Route 53 hosted zone
- Verify delegation with: `dig NS yourdomain.com @8.8.8.8`
- Some registrars take up to **48 hours** to propagate NS changes

---

## 19. Route 53 Resolvers & Hybrid DNS

### 📖 Concept

Route 53 Resolver enables **DNS resolution across hybrid environments** — between your AWS VPC and on-premises networks.

### Problem It Solves

```
Scenario: TechNova has on-premises servers at its Hyderabad office.
EC2 in VPC: cannot resolve internal.technova.local (on-prem DNS)
On-prem:    cannot resolve rds.ap-south-1.amazonaws.com (private AWS DNS)
```

### Route 53 Resolver Components

| Component | Direction | Description |
|-----------|-----------|-------------|
| **Inbound Endpoint** | On-prem → VPC | Allows on-prem DNS to query Route 53 |
| **Outbound Endpoint** | VPC → On-prem | Allows VPC to query on-prem DNS |
| **Resolver Rules** | VPC → Specific DNS | Forward specific domains to target DNS |

### Architecture Diagram

```
On-Premises Network (10.0.0.0/8)
  └── DNS Server (192.168.1.53)
          │
          │ VPN / Direct Connect
          │
AWS VPC (172.16.0.0/16)
  ├── Route 53 Inbound Endpoint (ENI: 172.16.1.10)
  ├── Route 53 Outbound Endpoint (ENI: 172.16.1.20)
  └── EC2 Instances

Forwarding Rules:
  *.technova.local  → Forward to 192.168.1.53 (on-prem DNS)
  *.amazonaws.com   → Route 53 Resolver (default)
```

### 🧪 Hands-On Exercise — Create Resolver Endpoints

**Step 1: Create Inbound Endpoint**
1. Route 53 → Resolver → Inbound Endpoints → Create
2. Name: `technova-inbound`
3. VPC: select your VPC
4. Security Group: allow UDP/TCP 53 from on-prem CIDR
5. Add 2 IP addresses (one per AZ for HA)
6. Note the endpoint IP addresses → configure on-prem DNS to forward AWS queries here

**Step 2: Create Outbound Endpoint**
1. Route 53 → Resolver → Outbound Endpoints → Create
2. Name: `technova-outbound`
3. VPC + Security Group (same as inbound)
4. Add IPs in 2 AZs

**Step 3: Create Forwarding Rule**
1. Route 53 → Resolver → Rules → Create Rule
2. Name: `forward-onprem`
3. Rule type: **Forward**
4. Domain: `technova.local`
5. Outbound endpoint: `technova-outbound`
6. Target IP: `192.168.1.53:53` (your on-prem DNS server)
7. Associate with VPC

**Test from EC2:**
```bash
# Should now resolve on-prem hostname
nslookup db-server.technova.local
# → Returns on-prem IP
```

### Private Hosted Zone — VPC Association

For internal-only DNS records (e.g., `db.internal`):
1. Create **Private Hosted Zone** (check "Private" when creating)
2. Associate it with your VPC(s)
3. Add records — these only resolve inside the VPC

```bash
# From EC2 in associated VPC:
nslookup db.internal  # Works!

# From internet:
nslookup db.internal  # Fails — private only
```

### 💡 Tips
- Always deploy endpoints in **2+ AZs** for high availability
- Resolver endpoints have **ENIs** — they count against your ENI limits
- Use **RAM (Resource Access Manager)** to share resolver rules across accounts

---

## 20. Route 53 — Section Cleanup

### ⚠️ Important — Clean Up to Avoid Charges

Follow this checklist **in order** to avoid unexpected AWS bills.

### Cleanup Checklist

**1. Delete Route 53 Records**
```
Route 53 → Hosted Zones → [Your Zone] → Select all records EXCEPT NS and SOA → Delete
```

**2. Delete Health Checks**
```
Route 53 → Health Checks → Select all → Delete
```

**3. Delete Traffic Policies**
```
Route 53 → Traffic Policies → Delete policy records first → Then delete policies
```

**4. Delete CIDR Collections**
```
Route 53 → CIDR Collections → Delete
```

**5. Delete Resolver Endpoints and Rules**
```
Route 53 → Resolver → Rules → Disassociate from VPCs → Delete rules
Route 53 → Resolver → Inbound/Outbound Endpoints → Delete
```

**6. Delete Private Hosted Zones**
```
Route 53 → Hosted Zones → [Private Zone] → Delete all records → Delete zone
```

**7. Delete EC2 Instances**
```
EC2 → Instances → Select Mumbai/Ireland/Virginia → Terminate Instance
```

*Switch regions and repeat for each EC2!*

**8. Release Elastic IPs (if used)**
```
EC2 → Elastic IPs → Select → Release Elastic IP Address
```

**9. Delete NAT Gateways (if used)**
```
VPC → NAT Gateways → Delete → Wait → Delete
```

**10. Delete S3 Buckets (failover page)**
```bash
aws s3 rb s3://failover.yourdomain.com --force
```

**11. Delete Hosted Zone (if you don't need it)**
```
Route 53 → Hosted Zones → Select → Delete Hosted Zone
```
> Note: You cannot delete a hosted zone with records other than NS/SOA. Delete all records first.

**12. Cancel Domain Registration (if you registered one)**
```
Route 53 → Registered Domains → [Domain] → Delete Domain
```
> Note: Domain refunds are only available within **5 days** of registration.

### Cost Summary — This Lab

| Resource | Monthly Cost |
|----------|-------------|
| Hosted Zone | $0.50 |
| Health Checks (3) | $1.50 |
| EC2 t2.micro × 3 | Free (Free Tier) or ~$7.50 |
| DNS Queries (testing) | < $0.01 |
| **If cleaned up after lab** | **~$0** |

---

## 📚 Quick Reference — All Routing Policies

| Policy | Use Case | Health Checks | Max Records |
|--------|----------|---------------|-------------|
| **Simple** | Single or random multi-value | ❌ | 1 (with multiple IPs) |
| **Weighted** | A/B testing, gradual rollout | ✅ Optional | Unlimited |
| **Latency** | Global app, lowest latency | ✅ Optional | One per region |
| **Failover** | Active-passive HA | ✅ Required on Primary | 2 (Primary + Secondary) |
| **Geolocation** | Legal compliance, localization | ✅ Optional | One per location |
| **Geoproximity** | Distance + bias control | ✅ Optional | Unlimited |
| **IP-based** | CIDR-based routing | ✅ Optional | Unlimited |
| **Multi-Value** | Client-side LB with HA | ✅ Recommended | Up to 8 |

---

## 🔑 Key Exam Tips (AWS SAA-C03)

1. **Alias vs CNAME:** Alias is free, works at root domain, points to AWS resources. CNAME has a charge, can't be at root.
2. **Health Check requirement:** Failover Primary MUST have a health check. Secondary doesn't.
3. **TTL:** Alias records have no TTL (managed by Route 53). Standard records have configurable TTL.
4. **Route 53 SLA:** 100% availability SLA — only AWS service with this guarantee.
5. **Private Hosted Zone:** Requires VPC association. Won't resolve from internet.
6. **Geolocation Default:** Always create a default record. Without it, unlisted regions get NXDOMAIN.
7. **Weighted 0:** Doesn't delete a record — just removes it from rotation.
8. **Multi-Value ≠ ELB:** Multi-Value is client-side and DNS-based; ELB operates at layer 4/7.
9. **Resolver Inbound:** Used so on-prem can query AWS private DNS.
10. **Resolver Outbound:** Used so VPC EC2 can query on-prem DNS.

---

## 🛠️ Useful Commands Reference

```bash
# Basic DNS lookup
nslookup www.technova.com
dig www.technova.com
dig +short www.technova.com

# Check NS records (who manages the domain)
dig NS technova.com

# Check specific record type
dig A www.technova.com
dig CNAME www.technova.com
dig MX technova.com
dig TXT technova.com

# Query a specific DNS server
dig @8.8.8.8 www.technova.com       # Google DNS
dig @1.1.1.1 www.technova.com       # Cloudflare DNS

# Check TTL countdown
watch -n 1 "dig +short www.technova.com"

# Trace DNS resolution path
dig +trace www.technova.com

# Reverse DNS lookup
dig -x 13.234.56.78

# AWS CLI — List hosted zones
aws route53 list-hosted-zones

# AWS CLI — List records in a zone
aws route53 list-resource-record-sets --hosted-zone-id Z1234ABCDEF

# AWS CLI — Create a health check
aws route53 create-health-check \
  --caller-reference $(date +%s) \
  --health-check-config '{
    "IPAddress": "13.234.56.78",
    "Port": 80,
    "Type": "HTTP",
    "ResourcePath": "/health",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }'
```

---

*Guide created for TechNova DevOps Lab | AWS Route 53 — Beginner to Advanced*
*Covers: DNS Fundamentals, Route 53 Console, All Routing Policies, Health Checks, Hybrid DNS*
