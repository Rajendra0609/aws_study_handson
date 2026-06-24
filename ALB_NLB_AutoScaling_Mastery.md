# AWS ALB / NLB & Auto Scaling — Complete Mastery Guide

> Your personal **AWS Load Balancing + Auto Scaling Trainer + AWS DevOps Mentor** in one document.
> Designed for **zero prior knowledge** → **real-time, production-grade projects**.
> Every topic includes: **plain-English concept** + **why it matters** + **🖱️ AWS Console steps** + **💻 AWS CLI commands (line-by-line)** + **real-world approaches** + **🐞 troubleshooting** + **✅ best practices** + **📝 assignments** + **interview questions**.

---

## Table of Contents

1. [How to Use This Guide](#1-how-to-use-this-guide)
2. [The Big Picture — Why Load Balancers & Auto Scaling Exist](#2-the-big-picture--why-load-balancers--auto-scaling-exist)
3. [Core Vocabulary (Read This First)](#3-core-vocabulary-read-this-first)
4. [Prerequisites & Environment Setup](#4-prerequisites--environment-setup)
5. [Elastic Load Balancing (ELB) Family Overview](#5-elastic-load-balancing-elb-family-overview)
6. [Building Blocks: Target Groups, Listeners, Health Checks](#6-building-blocks-target-groups-listeners-health-checks)
7. [Application Load Balancer (ALB) — Deep Dive](#7-application-load-balancer-alb--deep-dive)
8. [Network Load Balancer (NLB) — Deep Dive](#8-network-load-balancer-nlb--deep-dive)
9. [Gateway Load Balancer (GWLB) — Awareness](#9-gateway-load-balancer-gwlb--awareness)
10. [ALB vs NLB vs GWLB vs Classic — Decision Guide](#10-alb-vs-nlb-vs-gwlb-vs-classic--decision-guide)
11. [TLS/SSL, ACM Certificates & HTTPS Termination](#11-tlsssl-acm-certificates--https-termination)
12. [Advanced ALB Routing (Path, Host, Header, Weighted)](#12-advanced-alb-routing-path-host-header-weighted)
13. [Launch Templates (Foundation for Auto Scaling)](#13-launch-templates-foundation-for-auto-scaling)
14. [Auto Scaling Groups (ASG) — Deep Dive](#14-auto-scaling-groups-asg--deep-dive)
15. [Scaling Policies (Target Tracking, Step, Simple, Scheduled, Predictive)](#15-scaling-policies-target-tracking-step-simple-scheduled-predictive)
16. [Connecting ASG to ALB/NLB (The Full Loop)](#16-connecting-asg-to-albnlb-the-full-loop)
17. [Lifecycle Hooks, Health Checks & Instance Refresh](#17-lifecycle-hooks-health-checks--instance-refresh)
18. [Real-Time Architecture Approaches & Patterns](#18-real-time-architecture-approaches--patterns)
19. [End-to-End Capstone Project (Console + CLI)](#19-end-to-end-capstone-project-console--cli)
20. [Infrastructure as Code (Approaches)](#20-infrastructure-as-code-approaches)
21. [Monitoring, Logging & Observability](#21-monitoring-logging--observability)
22. [Security Best Practices](#22-security-best-practices)
23. [Cost Optimization](#23-cost-optimization)
24. [Troubleshooting Playbook](#24-troubleshooting-playbook)
25. [Best Practices Cheat Sheet](#25-best-practices-cheat-sheet)
26. [CLI Command Quick Reference](#26-cli-command-quick-reference)
27. [Interview Questions](#27-interview-questions)
28. [Glossary](#28-glossary)

---

## 1. How to Use This Guide

- **Read top to bottom the first time.** Concepts build on each other: *Vocabulary → Target Groups → Listeners → ALB/NLB → Launch Templates → Auto Scaling → Wiring it all together → Capstone.*
- Each section follows the same repeatable pattern so it becomes muscle memory.
- Symbols used throughout:
  - `🧠 Concept` — the theory in plain English
  - `🖱️ Console` — click-by-click GUI steps
  - `💻 CLI` — terminal commands with line-by-line explanation
  - `🏗️ Real-World` — how it's actually done on projects
  - `🐞 Troubleshooting` — common failures & fixes
  - `✅ Best Practice`
  - `📝 Assignment` — hands-on practice

> **Golden rules before every action:**
> 1. **Which Region am I in?** (e.g. `us-east-1`) — resources are region-scoped.
> 2. **Which IAM identity am I using?** — least privilege always.
> 3. **What does it cost?** — load balancers and EC2 run 24/7 and bill hourly. **Delete lab resources when done.**

---

## 2. The Big Picture — Why Load Balancers & Auto Scaling Exist

### 🧠 The problem

Imagine you run a website on **one** EC2 server.

- **Problem 1 — Single point of failure:** if that one server crashes, your site is *down*.
- **Problem 2 — Can't handle traffic spikes:** on a sale day, 1 server is overwhelmed → slow/crash.
- **Problem 3 — Wasted money at night:** at 3 AM almost nobody visits, but you still pay for a big server.

### 🧠 The solution (two tools working together)

1. **Load Balancer (ALB/NLB):** A traffic cop that sits in front of *many* servers and spreads incoming requests across all of them. If one server dies, it stops sending traffic there. Users only ever talk to the load balancer — they never know how many servers are behind it.

2. **Auto Scaling (ASG):** A robot that **adds servers when busy** and **removes servers when quiet** — automatically. You set rules like "keep CPU around 50%" and it does the rest.

### 🧠 How they combine (the classic pattern)

```
                Internet
                   │
                   ▼
          ┌─────────────────┐
          │  Load Balancer  │   (ALB or NLB) — public, fixed DNS name
          └────────┬────────┘
                   │ distributes traffic
        ┌──────────┼──────────┐
        ▼          ▼          ▼
    ┌───────┐  ┌───────┐  ┌───────┐
    │ EC2 #1│  │ EC2 #2│  │ EC2 #3│   ← managed by Auto Scaling Group
    └───────┘  └───────┘  └───────┘
        ▲ ASG adds/removes these based on demand & health
```

**Result:** Highly available (no single point of failure), elastic (scales with demand), and cost-efficient (shrinks when idle). This is the backbone of nearly every production AWS workload.

### 🏗️ Real-world value
- **Availability:** survive instance and Availability Zone (AZ) failures.
- **Elasticity:** handle Black Friday spikes without manual intervention.
- **Zero-downtime deployments:** roll out new app versions while old ones still serve traffic.
- **Decoupling:** clients use one stable DNS name; backend fleet changes freely.

---

## 3. Core Vocabulary (Read This First)

| Term | Plain-English meaning |
|------|----------------------|
| **Region** | A geographic area (e.g. `us-east-1` = N. Virginia). Everything lives in a region. |
| **Availability Zone (AZ)** | An isolated datacenter within a region (e.g. `us-east-1a`). Spread across 2+ AZs for HA. |
| **VPC** | Your private network in AWS. |
| **Subnet** | A slice of the VPC inside one AZ. **Public** = internet-reachable, **Private** = internal only. |
| **Security Group (SG)** | A virtual firewall attached to instances/load balancers. Allows specific ports/sources. |
| **ELB** | Elastic Load Balancing — the umbrella service. ALB, NLB, GWLB, and Classic are its types. |
| **ALB** | Application Load Balancer — Layer 7 (HTTP/HTTPS), smart routing. |
| **NLB** | Network Load Balancer — Layer 4 (TCP/UDP/TLS), ultra-fast, static IP. |
| **Listener** | A rule on the LB: "listen on port X using protocol Y, then do Z." |
| **Target Group** | A logical bucket of backends (EC2/IP/Lambda) the LB forwards to. Owns health checks. |
| **Target** | An actual backend registered in a target group (an instance, IP, or Lambda). |
| **Health Check** | The LB periodically pings targets; unhealthy ones stop receiving traffic. |
| **Launch Template** | A blueprint for new EC2 instances (AMI, type, key, SG, user data). |
| **Auto Scaling Group (ASG)** | Manages a fleet of EC2: min/desired/max count, scaling, self-healing. |
| **Scaling Policy** | The rule that tells the ASG when to add/remove instances. |
| **Desired / Min / Max Capacity** | ASG keeps `desired` instances; never below `min`, never above `max`. |
| **Cooldown / Warm-up** | A pause so metrics stabilize before scaling again. |
| **Target Tracking** | "Keep metric X at value Y" — the easiest, most common scaling policy. |
| **AMI** | Amazon Machine Image — a snapshot used to launch instances. |
| **ACM** | AWS Certificate Manager — free TLS/SSL certificates for HTTPS. |
| **Sticky Session** | Pin a user to the same backend for their session. |
| **Cross-Zone Load Balancing** | Spread traffic evenly across all AZs, not just per-AZ. |
| **Connection Draining (Deregistration Delay)** | Let in-flight requests finish before removing a target. |

---

## 4. Prerequisites & Environment Setup

### 🧠 What you need
- An AWS account (free tier is enough for labs; some LB hours/data may bill).
- A VPC with **at least 2 subnets in 2 different AZs** (the default VPC already has this).
- AWS CLI v2 installed and configured.
- (For labs) an EC2 key pair if you want SSH access.

### 💻 Install & verify AWS CLI v2

```bash
aws --version
# → aws-cli/2.x.x ...   (must be v2; v1 lacks newer flags)
```

```bash
aws configure
# AWS Access Key ID     : <paste>
# AWS Secret Access Key : <paste>
# Default region name   : us-east-1     ← pick your region
# Default output format  : json
```

```bash
aws sts get-caller-identity
# Confirms WHO you are (Account, UserId, ARN). Always run this first.
```

### 💻 Capture reusable variables (used throughout this guide)

```bash
# Region
export AWS_REGION=us-east-1

# Get the default VPC id
export VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query 'Vpcs[0].VpcId' --output text)
echo "VPC_ID=$VPC_ID"

# Get two subnets in two different AZs (needed for HA load balancing)
export SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID \
  --query 'Subnets[0:2].SubnetId' --output text)
echo "SUBNETS=$SUBNETS"   # e.g. subnet-aaa  subnet-bbb
```

> **Windows PowerShell users:** replace `export VAR=value` with `$VAR = "value"` and reference with `$VAR`. The `aws` commands themselves are identical.

> **✅ Best Practice:** Never hard-code IDs. Capture them into variables (Bash) or use parameters (IaC). It makes scripts portable and repeatable.

---

## 5. Elastic Load Balancing (ELB) Family Overview

### 🧠 Concept
**Elastic Load Balancing (ELB)** is the AWS service that automatically distributes incoming traffic across multiple targets. It is **managed** (AWS runs it, patches it, scales it) and **highly available** (it spans multiple AZs by design). It comes in four flavors:

| Type | OSI Layer | Protocols | Killer Feature | Typical Use |
|------|-----------|-----------|----------------|-------------|
| **Application LB (ALB)** | 7 (App) | HTTP, HTTPS, gRPC, WebSocket | Content-based routing (path/host/header) | Web apps, microservices, containers |
| **Network LB (NLB)** | 4 (Transport) | TCP, UDP, TLS | Millions of req/s, ultra-low latency, **static IP** | High-performance, gaming, IoT, financial |
| **Gateway LB (GWLB)** | 3 (Network) | IP | Insert virtual appliances (firewalls/IDS) | Security inspection pipelines |
| **Classic LB (CLB)** | 4 & 7 | HTTP/HTTPS/TCP | Legacy only | **Avoid** — old EC2-Classic; deprecated |

### 🧠 Why "Layer 4 vs Layer 7" matters (simple analogy)
- **Layer 4 (NLB)** is like a postal sorting machine that reads only the *envelope* (IP + port). It's blazing fast but can't read the *letter inside*.
- **Layer 7 (ALB)** opens the *letter* (the HTTP request), reads the URL/headers/cookies, and makes smart decisions ("this `/api` request → API servers; this `/images` → image servers").

> **Rule of thumb:** Default to **ALB** for web/HTTP workloads. Choose **NLB** when you need extreme performance, non-HTTP protocols, or a static/Elastic IP. We cover both fully below.

---

## 6. Building Blocks: Target Groups, Listeners, Health Checks

Before creating any load balancer, understand its three internal parts. **Every ALB/NLB is just: Listeners → Rules → Target Groups → Targets, with Health Checks watching the targets.**

### 6.1 Target Group

#### 🧠 Concept
A **target group** is a labeled bucket of backends. The load balancer forwards traffic *to a target group*, not directly to instances. Target types:
- **instance** — register EC2 by instance ID (most common, used with ASG).
- **ip** — register raw IP addresses (containers, on-prem via Direct Connect/VPN, Fargate).
- **lambda** — invoke a Lambda function (ALB only).
- **alb** — an ALB as a target of an NLB (advanced chaining).

#### 🖱️ Console
1. **EC2 Console → Target Groups → Create target group.**
2. Choose target type (e.g. **Instances**).
3. Name: `tg-web`. Protocol/Port: `HTTP : 80`. VPC: select yours.
4. Protocol version: `HTTP1` (default).
5. Configure **Health checks** (next section).
6. **Next → Register targets** (or leave empty if an ASG will register them).
7. **Create target group.**

#### 💻 CLI
```bash
# Create a target group for HTTP on port 80
aws elbv2 create-target-group \
  --name tg-web \                # name of the target group
  --protocol HTTP \              # protocol the LB uses to reach targets
  --port 80 \                    # port on the targets
  --vpc-id $VPC_ID \             # which VPC the targets live in
  --target-type instance \       # instance | ip | lambda | alb
  --health-check-protocol HTTP \ # how to health-check
  --health-check-path /health \  # URL path that returns 200 when healthy
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --query 'TargetGroups[0].TargetGroupArn' --output text
# Save the printed ARN:
export TG_ARN=<paste-arn>
```

### 6.2 Listener

#### 🧠 Concept
A **listener** checks for connection requests on a **protocol + port** you specify, then applies **rules** to decide where traffic goes. Example: "Listen on port 443 (HTTPS); forward to `tg-web`." An LB can have many listeners (e.g. 80 and 443).

#### 💻 CLI (created together with/after the LB — full example in §7)
```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN
```

### 6.3 Health Checks

#### 🧠 Concept
The LB periodically sends a probe to each target. If a target passes `HealthyThresholdCount` checks → **healthy** (gets traffic). If it fails `UnhealthyThresholdCount` checks → **unhealthy** (traffic stops, but the target stays registered and is rechecked).

| Setting | Meaning | Typical |
|---------|---------|---------|
| Protocol/Port | How to probe | HTTP:80 (traffic port) |
| Path | URL that must return success (ALB) | `/health` |
| Healthy threshold | Consecutive passes to mark healthy | 2–5 |
| Unhealthy threshold | Consecutive fails to mark unhealthy | 2–3 |
| Interval | Seconds between checks | 15–30 |
| Timeout | Seconds to wait for a response | 5 |
| Success codes | HTTP codes meaning "OK" (ALB) | `200` or `200-299` |

> **✅ Best Practice:** Build a dedicated lightweight `/health` endpoint in your app that checks *real* dependencies (DB, cache) and returns `200` only when the app can truly serve traffic. Don't health-check `/` (the heavy homepage).

#### 🐞 Common health-check gotcha
If targets show **unhealthy** even though the app works:
1. The instance **Security Group must allow the LB's SG** on the health-check port.
2. The health-check **path must exist and return 2xx**.
3. The app must actually be **listening on the configured port**.

---

## 7. Application Load Balancer (ALB) — Deep Dive

### 🧠 Concept
The **ALB** operates at **Layer 7**. It understands HTTP/HTTPS, so it can route based on **URL path, hostname, HTTP headers, query strings, HTTP method, and source IP**. It supports HTTP/2, gRPC, WebSockets, sticky sessions, redirects, fixed responses, authentication (Cognito/OIDC), and integrates natively with WAF. It scales automatically and is **never given a fixed IP** — you always use its **DNS name**.

### 🧠 When to use ALB
- Web applications and REST APIs.
- **Microservices**: one ALB routes `/orders` → orders service, `/users` → users service.
- Containers (ECS/EKS) — ALB is the standard ingress.
- You need HTTPS termination, redirects, host-based multi-tenant routing, or WAF.

### 7.1 🖱️ Console — Create an Internet-facing ALB

1. **EC2 Console → Load Balancers → Create load balancer.**
2. Choose **Application Load Balancer → Create.**
3. **Name:** `web-alb`.
4. **Scheme:** `Internet-facing` (use `Internal` for private/internal apps).
5. **IP address type:** `IPv4`.
6. **Network mapping:** select your **VPC**, then tick **at least 2 AZs** and pick a **public subnet** in each. *(Two AZs is the minimum for HA.)*
7. **Security groups:** create/select an SG that allows inbound `80` and `443` from `0.0.0.0/0` (for a public web app).
8. **Listeners and routing:** Listener `HTTP : 80` → forward to a target group (create `tg-web` if needed).
9. (Optional) Add `HTTPS : 443` listener and attach an ACM certificate (see §11).
10. **Create load balancer.** Wait until **State: Active**.
11. Copy the **DNS name** (e.g. `web-alb-123.us-east-1.elb.amazonaws.com`) — this is your app's address.

### 7.2 💻 CLI — Create an ALB end-to-end

```bash
# 1) Create a security group for the ALB (allow HTTP/HTTPS from the internet)
export ALB_SG=$(aws ec2 create-security-group \
  --group-name web-alb-sg --description "ALB SG" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --protocol tcp --port 80 --cidr 0.0.0.0/0      # allow HTTP
aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --protocol tcp --port 443 --cidr 0.0.0.0/0     # allow HTTPS

# 2) Create the ALB across 2 subnets/AZs
export ALB_ARN=$(aws elbv2 create-load-balancer \
  --name web-alb \
  --type application \
  --scheme internet-facing \
  --subnets $SUBNETS \              # two subnets in two AZs
  --security-groups $ALB_SG \
  --ip-address-type ipv4 \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
echo "ALB_ARN=$ALB_ARN"

# 3) Create a listener on port 80 that forwards to the target group
export LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --query 'Listeners[0].ListenerArn' --output text)

# 4) Get the public DNS name to test in a browser
aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text
```

### 7.3 Registering targets manually (when not using ASG)

```bash
# Register one or more EC2 instances into the target group
aws elbv2 register-targets --target-group-arn $TG_ARN \
  --targets Id=i-0123456789abcdef0 Id=i-0fedcba9876543210

# Watch health status until 'healthy'
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].{Id:Target.Id,State:TargetHealth.State}'
```

### 7.4 Key ALB attributes worth knowing
| Attribute | What it does |
|-----------|--------------|
| **Idle timeout** (default 60s) | Closes connections idle this long. Raise for slow/long polling. |
| **Deletion protection** | Prevents accidental deletion of the LB. Enable in prod. |
| **HTTP/2 & gRPC** | Supported natively. |
| **Sticky sessions** | Cookie-based; pin a user to one target (set on the target group). |
| **Access logs** | Write request logs to S3 (off by default). |
| **WAF integration** | Attach AWS WAF for L7 protection (SQLi, XSS, rate limiting). |
| **Desync mitigation** | Protects against HTTP request smuggling. |

```bash
# Example: turn ON access logs and deletion protection
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=my-alb-logs-bucket \
    Key=deletion_protection.enabled,Value=true \
    Key=idle_timeout.timeout_seconds,Value=60
```

### 🐞 ALB Troubleshooting
- **502 Bad Gateway:** target returned malformed response, crashed, or closed the connection — check the app and its port.
- **503 Service Unavailable:** no **healthy** targets in the target group — check health checks & registration.
- **504 Gateway Timeout:** target took longer than the idle timeout — optimize the app or raise idle timeout.
- **Targets unhealthy:** instance SG must allow the **ALB SG** on the health-check port; verify path returns 2xx.

### 7.5 Sticky Sessions (Session Affinity)

#### 🧠 Concept
By default the ALB spreads each request across targets. **Sticky sessions** pin a given user to the **same target** for the duration of their session — needed when the app stores session state locally (e.g. an in-memory shopping cart). Two cookie types:
- **Application-based** (`app_cookie`): your app's own cookie governs stickiness.
- **Duration-based** (`lb_cookie`): the ALB issues an `AWSALB` cookie with a fixed lifetime.

> **✅ Best Practice:** Prefer **stateless** apps (store sessions in Redis/ElastiCache/DynamoDB). Stickiness defeats even load distribution and complicates scale-in. Use it only when you must.

#### 💻 CLI — enable duration-based stickiness (1 hour)
```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes \
    Key=stickiness.enabled,Value=true \
    Key=stickiness.type,Value=lb_cookie \
    Key=stickiness.lb_cookie.duration_seconds,Value=3600
```
#### 🖱️ Console
Target group → **Attributes → Edit → Turn on stickiness** → choose type & duration.

### 7.6 Target Group Attributes (Algorithm, Slow Start)

#### 🧠 Load-balancing algorithm
- **`round_robin`** (default) — each healthy target in turn. Good when requests are uniform.
- **`least_outstanding_requests` (LOR)** — send the next request to the target with the fewest in-flight requests. Better when request durations vary widely.
- **`weighted_random`** — random with optional anomaly mitigation.

```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=load_balancing.algorithm.type,Value=least_outstanding_requests
```

#### 🧠 Slow start
**Slow start** ramps traffic to a *newly registered* target gradually over N seconds, so a freshly booted instance (cold caches/JIT) isn't hit with full load instantly.
```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=slow_start.duration_seconds,Value=30   # 30s ramp (0 = off)
```

### 7.7 Cross-Zone Load Balancing (ALB)
#### 🧠 Concept
With **cross-zone** ON, every LB node can send to targets in **all** AZs (truly even distribution). It is **always ON for ALB** (and free). On a target group you can override it; on NLB it's OFF by default (see §8.4).
```bash
# Override at the target-group level if ever needed
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

### 7.8 ALB Authentication (Cognito / OIDC)
#### 🧠 Concept
The ALB can **require login before forwarding** a request — offloading authentication from your app. Use **`authenticate-cognito`** (Amazon Cognito user pools) or **`authenticate-oidc`** (any OpenID Connect IdP: Google, Okta, Azure AD, Auth0). Unauthenticated users are redirected to the IdP; the ALB injects identity headers to the backend.
```bash
# Require OIDC login, then forward to the app (rule on the HTTPS listener)
aws elbv2 create-rule --listener-arn $HTTPS_LISTENER_ARN --priority 1 \
  --conditions Field=path-pattern,Values='/secure/*' \
  --actions '[
    {"Type":"authenticate-oidc","Order":1,"AuthenticateOidcConfig":{
      "Issuer":"https://idp.example.com",
      "AuthorizationEndpoint":"https://idp.example.com/authorize",
      "TokenEndpoint":"https://idp.example.com/token",
      "UserInfoEndpoint":"https://idp.example.com/userinfo",
      "ClientId":"<client-id>","ClientSecret":"<client-secret>"}},
    {"Type":"forward","Order":2,"TargetGroupArn":"'"$TG_ARN"'"}
  ]'
```
> **Note:** Authentication actions require an **HTTPS** listener.

### 7.9 AWS WAF Integration (L7 protection)
#### 🧠 Concept
**AWS WAF** attaches to a public ALB to block common web attacks (SQL injection, XSS), enforce **rate limiting**, and apply **geo/IP** rules — before traffic reaches your app.
```bash
# Associate an existing WAFv2 Web ACL with the ALB
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:111122223333:regional/webacl/my-acl/abc \
  --resource-arn $ALB_ARN
```
#### 🖱️ Console
WAF & Shield → **Web ACLs** → create/select → **Associated AWS resources → Add → your ALB**.

### 7.10 Lambda & IP Targets (serverless / mixed backends)
#### 🧠 Concept
An ALB can invoke a **Lambda function** as a target (target type `lambda`) — great for lightweight/serverless endpoints behind the same DNS as your EC2 app. The **`ip`** target type registers raw IPs (containers, on-prem via VPN/Direct Connect, Fargate tasks).
```bash
# Lambda target group + register the function
export LAMBDA_TG=$(aws elbv2 create-target-group --name tg-lambda \
  --target-type lambda --query 'TargetGroups[0].TargetGroupArn' --output text)

# Allow ELB to invoke the function (resource-based permission)
aws lambda add-permission --function-name my-func \
  --statement-id elb --action lambda:InvokeFunction \
  --principal elasticloadbalancing.amazonaws.com

aws elbv2 register-targets --target-group-arn $LAMBDA_TG \
  --targets Id=arn:aws:lambda:us-east-1:111122223333:function:my-func
```

### 7.11 Client IP & Proxy Headers (X-Forwarded-*)
#### 🧠 Concept
Because the **ALB terminates the connection**, your backend sees the **ALB's IP** as the source, not the real user. The ALB therefore injects HTTP headers so your app can recover the original details:
- **`X-Forwarded-For`** — the real client IP (append-only chain).
- **`X-Forwarded-Proto`** — original scheme (`http`/`https`) — essential to detect HTTPS after TLS termination.
- **`X-Forwarded-Port`** — original port.

> **🏗️ Real-World:** Configure your framework to **trust these headers** (e.g. `ProxyPass`/`trust proxy` in Express, `ForwardedHeaders` in .NET, `X-Forwarded-*` in Nginx) so logging, redirects, and rate-limiting use the true client IP/scheme. Without this, every request appears to come from the load balancer.

```bash
# Control how the header is handled: append (default) | preserve | remove
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes Key=routing.http.xff_header_processing.mode,Value=append

# Preserve the original client port too
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes Key=routing.http.xff_client_port.enabled,Value=true
```

### 7.12 Mutual TLS (mTLS) for Client Authentication
#### 🧠 Concept
**mTLS** makes the ALB verify the *client's* certificate (not just the client verifying the server) — used for secure machine-to-machine APIs, IoT, and partner integrations. You upload a **trust store** (CA bundle) to S3 and reference it on the HTTPS listener. Modes: **`verify`** (reject untrusted) or **`passthrough`** (forward the cert to the app to validate).
```bash
# 1) Create a trust store from a CA bundle in S3
export TS_ARN=$(aws elbv2 create-trust-store --name partner-ca \
  --ca-certificates-bundle-s3-bucket my-certs --ca-certificates-bundle-s3-key ca-bundle.pem \
  --query 'TrustStores[0].TrustStoreArn' --output text)

# 2) Attach mTLS to the HTTPS listener in verify mode
aws elbv2 modify-listener --listener-arn $HTTPS_LISTENER_ARN \
  --mutual-authentication Mode=verify,TrustStoreArn=$TS_ARN
```

---

## 8. Network Load Balancer (NLB) — Deep Dive

### 🧠 Concept
The **NLB** operates at **Layer 4** (TCP/UDP/TLS). It does **not** read HTTP content — it forwards packets extremely fast with **ultra-low latency** and can handle **millions of requests per second**. Its standout features:
- **Static IP per AZ** (and you can assign an **Elastic IP**) — great for firewall allow-lists and clients that need a fixed IP.
- **Preserves the client source IP** by default (the backend sees the real client IP).
- Handles **TCP, UDP, and TLS** (not just HTTP).
- Can do **TLS termination** at Layer 4.

### 🧠 When to use NLB
- Extreme performance / low latency (trading, gaming, real-time).
- **Non-HTTP** protocols (TCP/UDP) — databases, MQTT/IoT, SMTP, DNS, syslog.
- You need a **static IP / Elastic IP** or a single PrivateLink endpoint.
- Long-lived TCP connections (e.g. millions of IoT devices).

### 8.1 🖱️ Console — Create an NLB
1. **EC2 Console → Load Balancers → Create load balancer.**
2. Choose **Network Load Balancer → Create.**
3. **Name:** `tcp-nlb`. **Scheme:** Internet-facing or Internal.
4. **Network mapping:** pick AZs/subnets. *(Optional: assign an Elastic IP per AZ for a fixed IP.)*
5. **Listeners:** `TCP : 80` (or `TLS : 443`, `UDP : 53`, etc.) → forward to a **TCP** target group.
6. Create the target group with protocol **TCP** and a TCP/HTTP health check.
7. **Create load balancer.**

### 8.2 💻 CLI — Create an NLB end-to-end
```bash
# 1) TCP target group (note: protocol TCP, not HTTP)
export NLB_TG=$(aws elbv2 create-target-group \
  --name tg-tcp --protocol TCP --port 80 \
  --vpc-id $VPC_ID --target-type instance \
  --health-check-protocol TCP \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# 2) Create the NLB (type=network). No security group needed historically,
#    but NLBs now SUPPORT security groups — attach one if you want SG control.
export NLB_ARN=$(aws elbv2 create-load-balancer \
  --name tcp-nlb --type network --scheme internet-facing \
  --subnets $SUBNETS \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

# 3) TCP listener forwarding to the TCP target group
aws elbv2 create-listener --load-balancer-arn $NLB_ARN \
  --protocol TCP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$NLB_TG

# 4) Get the NLB DNS name
aws elbv2 describe-load-balancers --load-balancer-arns $NLB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text
```

### 8.3 NLB specifics to remember
- **Source IP preservation:** backends see the client's real IP — your app/SGs must account for this. (With `instance` target type and IP preservation, the **target SG must allow client CIDRs**, not the NLB.)
- **Cross-zone load balancing is OFF by default** on NLB (and may incur inter-AZ data charges when ON). It's ON by default on ALB.
- **Static/Elastic IP:** assign EIPs at creation for fixed addresses.
- **Health checks:** TCP (connect only), HTTP, or HTTPS.

### 🐞 NLB Troubleshooting
- **Client can't connect, targets healthy:** with source-IP preservation, the **target instance SG must allow the client IP range**, not the NLB.
- **Uneven AZ load:** enable **cross-zone load balancing** (accept possible inter-AZ cost).
- **TLS issues:** verify the ACM cert and security policy on the TLS listener.

### 8.4 Cross-Zone Load Balancing on NLB
#### 🧠 Concept
Unlike ALB, NLB cross-zone is **OFF by default**. With it OFF, a client connecting to the AZ-A node only reaches AZ-A targets (can cause imbalance if AZs have unequal target counts). Turning it ON evens distribution but **may incur inter-AZ data charges**.
```bash
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $NLB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

### 8.5 PrivateLink / VPC Endpoint Service (NLB)
#### 🧠 Concept
**AWS PrivateLink** lets you expose a service privately to *other VPCs/accounts* without going over the internet. You front your service with an **NLB**, create an **endpoint service** from it, and consumers create an **interface VPC endpoint** to reach it privately. This is the standard SaaS / cross-account private connectivity pattern.
```bash
# Publish your NLB as a PrivateLink endpoint service
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns $NLB_ARN \
  --acceptance-required \
  --query 'ServiceConfiguration.ServiceName' --output text
# Share the returned service name with consumers; approve their connections:
# aws ec2 accept-vpc-endpoint-connections --service-id <id> --vpc-endpoint-ids <vpce-id>
```
> Consumers then run `aws ec2 create-vpc-endpoint --vpc-endpoint-type Interface --service-name <name> ...`.

### 8.6 Proxy Protocol v2 (preserve client IP with IP targets)
#### 🧠 Concept
NLB preserves the client IP automatically for **instance** targets, but for **IP** targets (or when traffic crosses a PrivateLink/peering boundary) the backend may see the NLB's IP. **Proxy Protocol v2** prepends a small header carrying the original source IP/port so the backend can recover it. Your application/server must be configured to **parse Proxy Protocol** (e.g. Nginx `proxy_protocol`, HAProxy).
```bash
aws elbv2 modify-target-group-attributes --target-group-arn $NLB_TG \
  --attributes Key=proxy_protocol_v2.enabled,Value=true
```

---

## 9. Gateway Load Balancer (GWLB) — Awareness

### 🧠 Concept
The **GWLB** lets you deploy, scale, and manage **third-party virtual appliances** (firewalls, IDS/IPS, deep packet inspection) transparently. It operates at **Layer 3** and uses the **GENEVE protocol (port 6081)**. Traffic is routed *through* a fleet of appliances for inspection, then back to its destination. You combine it with **GWLB Endpoints** in your route tables.

> **For most app developers this is awareness-level.** Use it when a security/network team needs to insert inline inspection appliances. It is not part of typical app load balancing or auto scaling, so we won't lab it here — just know it exists and what it's for.

---

## 10. ALB vs NLB vs GWLB vs Classic — Decision Guide

| Question | Choose |
|----------|--------|
| Web app / HTTP(S) / REST API? | **ALB** |
| Need path/host/header routing or microservices? | **ALB** |
| Need WebSocket / gRPC / HTTP/2? | **ALB** |
| Need Lambda as a target? | **ALB** |
| Need extreme performance / millions req/s / low latency? | **NLB** |
| Need TCP/UDP (non-HTTP) protocols? | **NLB** |
| Need a **static IP / Elastic IP** or PrivateLink? | **NLB** |
| Need to preserve client source IP simply? | **NLB** |
| Need to insert firewall/IDS appliances inline? | **GWLB** |
| Building something new and it's old EC2-Classic? | **Migrate off Classic** |

> **90% of projects:** ALB for the web tier, optionally NLB in front of it for static IP or for TCP services. Auto Scaling sits behind whichever LB you choose.

---

## 11. TLS/SSL, ACM Certificates & HTTPS Termination

### 🧠 Concept
Production sites must use **HTTPS**. The LB performs **TLS termination**: it holds the certificate, decrypts HTTPS from clients, and (typically) talks plain HTTP to backends inside your private VPC. AWS **Certificate Manager (ACM)** issues **free** public certificates and auto-renews them.

### 🧠 The standard secure pattern
1. **HTTP (80) listener → redirect to HTTPS (443).**
2. **HTTPS (443) listener → ACM certificate → forward to target group.**

### 11.1 💻 Request an ACM certificate (DNS validation)
```bash
# Request a cert for your domain (DNS validation auto-renews)
export CERT_ARN=$(aws acm request-certificate \
  --domain-name app.example.com \
  --validation-method DNS \
  --query CertificateArn --output text)

# Get the CNAME record you must add in Route 53/your DNS to validate
aws acm describe-certificate --certificate-arn $CERT_ARN \
  --query 'Certificate.DomainValidationOptions'
# Add the CNAME, then wait until status = ISSUED:
aws acm describe-certificate --certificate-arn $CERT_ARN \
  --query 'Certificate.Status'
```

### 11.2 💻 Add an HTTPS listener + redirect HTTP→HTTPS
```bash
# HTTPS (443) listener using the ACM cert and a modern TLS policy
aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=$CERT_ARN \
  --ssl-policy ELBSecurityPolicy-TLS13-1-2-2021-06 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Change the port-80 listener to REDIRECT everything to 443
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions '[{"Type":"redirect","RedirectConfig":{"Protocol":"HTTPS","Port":"443","StatusCode":"HTTP_301"}}]'
```

### 🖱️ Console (equivalent)
- On the ALB → **Listeners → Add listener → HTTPS:443 →** choose **ACM certificate** + **security policy** → forward to target group.
- Edit the **HTTP:80** listener → default action **Redirect → HTTPS 443, 301**.

> **✅ Best Practice:** Use a TLS 1.2+/1.3 security policy. Terminate TLS at the LB; keep backend traffic inside the VPC. Use ACM (free, auto-renew) instead of manual certs.

### 11.3 Pointing a Domain at the Load Balancer (Route 53)
#### 🧠 Concept
Users shouldn't type the ugly `web-alb-123.elb.amazonaws.com` name. In **Route 53** you create an **Alias A record** (and **AAAA** for IPv6) that maps your domain (`app.example.com`) to the load balancer. **Alias records are free, resolve to the LB's changing IPs automatically, and work at the zone apex** (unlike a CNAME).
```bash
# Find your hosted zone id
export ZONE_ID=$(aws route53 list-hosted-zones-by-name --dns-name example.com \
  --query 'HostedZones[0].Id' --output text)

# Get the ALB's hosted zone id + DNS name (needed for the alias target)
read ALB_DNS ALB_ZONE < <(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].[DNSName,CanonicalHostedZoneId]' --output text)

# Create the alias record app.example.com → ALB
aws route53 change-resource-record-sets --hosted-zone-id $ZONE_ID --change-batch '{
  "Changes":[{"Action":"UPSERT","ResourceRecordSet":{
    "Name":"app.example.com","Type":"A",
    "AliasTarget":{"HostedZoneId":"'"$ALB_ZONE"'","DNSName":"'"$ALB_DNS"'","EvaluateTargetHealth":true}
  }}]
}'
```
#### 🖱️ Console
Route 53 → **Hosted zones → your zone → Create record →** Name `app`, Type `A`, **Alias = Yes**, route traffic to **Alias to Application/Network Load Balancer → region → your LB**.

> **✅ Best Practice:** Use **Alias** (not CNAME) for LBs. Set `EvaluateTargetHealth` and consider **latency/failover/weighted** routing policies for multi-region HA.

---

## 12. Advanced ALB Routing (Path, Host, Header, Weighted)

### 🧠 Concept
ALB **listener rules** evaluate in **priority order** (lowest number first). Each rule has **conditions** (if…) and **actions** (then…). This is how one ALB serves many microservices or multiple domains.

### 🧠 Rule condition types
- **Path** — `/api/*`, `/images/*`
- **Host header** — `api.example.com` vs `shop.example.com`
- **HTTP header** — e.g. `X-Env: beta`
- **HTTP method** — `GET`, `POST`
- **Query string** — `?version=2`
- **Source IP** — restrict by CIDR

### 🧠 Action types
- **forward** (to one or more target groups, optionally **weighted** for canary/blue-green)
- **redirect** (e.g. HTTP→HTTPS, or old path → new path)
- **fixed-response** (return a static page/JSON, e.g. maintenance mode)
- **authenticate-cognito / authenticate-oidc** (require login before forwarding)

### 12.1 💻 Path-based routing example
```bash
# Send /api/* to the API target group with priority 10
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 10 \
  --conditions Field=path-pattern,Values='/api/*' \
  --actions Type=forward,TargetGroupArn=$API_TG_ARN
```

### 12.2 💻 Host-based routing example
```bash
# Route shop.example.com to the shop target group
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 20 \
  --conditions Field=host-header,Values='shop.example.com' \
  --actions Type=forward,TargetGroupArn=$SHOP_TG_ARN
```

### 12.3 💻 Weighted routing (canary / blue-green)
```bash
# Send 90% to v1, 10% to v2 (gradually shift traffic to test a new version)
aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
  --default-actions '[{
    "Type":"forward",
    "ForwardConfig":{
      "TargetGroups":[
        {"TargetGroupArn":"'"$TG_V1"'","Weight":90},
        {"TargetGroupArn":"'"$TG_V2"'","Weight":10}
      ]
    }
  }]'
```

### 12.4 💻 Fixed response (maintenance mode)
```bash
aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 5 \
  --conditions Field=path-pattern,Values='/maintenance' \
  --actions '[{"Type":"fixed-response","FixedResponseConfig":{"StatusCode":"503","ContentType":"text/plain","MessageBody":"Under maintenance"}}]'
```

> **🏗️ Real-World:** Microservice teams give each service its own **target group** and a **path or host rule** on a shared ALB. Blue/green and canary deploys use **weighted target groups** to shift traffic safely, then roll back instantly by changing weights.

---

## 13. Launch Templates (Foundation for Auto Scaling)

### 🧠 Concept
Auto Scaling launches new EC2 instances on demand — but it needs a **blueprint** describing *how* to launch them: which AMI, instance type, key pair, security groups, IAM role, and **user data** (startup script). That blueprint is a **Launch Template**.

> **Launch Template vs Launch Configuration:** Launch Configurations are **legacy** (no versioning, fewer features). **Always use Launch Templates** — they support versioning, multiple instance types, Spot, and newer features.

### 🧠 User data = the startup script
**User data** runs once when an instance first boots — perfect for installing your app/agent so every new instance is ready to serve traffic automatically.

### 13.1 🖱️ Console — Create a Launch Template
1. **EC2 Console → Launch Templates → Create launch template.**
2. **Name:** `web-lt`.
3. **AMI:** Amazon Linux 2023 (or your golden AMI).
4. **Instance type:** `t3.micro` (lab) / right-size for prod.
5. **Key pair:** optional (omit if using SSM Session Manager).
6. **Security groups:** the **instance** SG (allow ALB SG on app port, e.g. 80).
7. **Advanced → IAM instance profile:** attach a role (e.g. for SSM/CloudWatch).
8. **Advanced → User data:** paste a bootstrap script (below).
9. **Create launch template.**

### 13.2 💻 CLI — Create a Launch Template

First create a user-data script and base64-encode it:
```bash
cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo "Hello from $(hostname -f)" > /var/www/html/index.html
echo "OK" > /var/www/html/health    # health-check endpoint returns 200
EOF

# Base64 encode (Linux/macOS)
export USERDATA=$(base64 -w0 userdata.sh)   # Windows PowerShell: see note below
```

```bash
# Create an instance SG that allows the ALB SG on port 80 + SSH for admin
export INST_SG=$(aws ec2 create-security-group \
  --group-name web-inst-sg --description "Instance SG" --vpc-id $VPC_ID \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $INST_SG \
  --protocol tcp --port 80 --source-group $ALB_SG   # only the ALB can reach app port

# Find the latest Amazon Linux 2023 AMI id
export AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameters[0].Value' --output text)

# Create the launch template
aws ec2 create-launch-template \
  --launch-template-name web-lt \
  --version-description v1 \
  --launch-template-data '{
    "ImageId":"'"$AMI_ID"'",
    "InstanceType":"t3.micro",
    "SecurityGroupIds":["'"$INST_SG"'"],
    "UserData":"'"$USERDATA"'",
    "TagSpecifications":[{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"web-asg"}]}]
  }'
```

> **Windows PowerShell base64:**
> ```powershell
> $USERDATA = [Convert]::ToBase64String([IO.File]::ReadAllBytes("userdata.sh"))
> ```

### 13.3 Versioning launch templates
```bash
# Create a new version (e.g. new AMI), then ASG can use $Latest or a fixed version
aws ec2 create-launch-template-version \
  --launch-template-name web-lt --version-description v2 \
  --source-version 1 \
  --launch-template-data '{"ImageId":"ami-NEW"}'

# Set the default version
aws ec2 modify-launch-template --launch-template-name web-lt --default-version 2
```

> **✅ Best Practice:** Bake as much as possible into a **golden AMI** (faster boot, fewer failure points) and keep user data minimal. Use **Launch Template versions** for controlled rollouts via **Instance Refresh** (§17).

---

## 14. Auto Scaling Groups (ASG) — Deep Dive

### 🧠 Concept
An **Auto Scaling Group (ASG)** manages a fleet of EC2 instances as one unit. It:
- **Maintains capacity:** keeps `DesiredCapacity` instances running; never below `MinSize`, never above `MaxSize`.
- **Self-heals:** if an instance fails its health check or terminates, the ASG launches a replacement automatically.
- **Spreads across AZs:** balances instances across the subnets/AZs you give it.
- **Scales:** adds/removes instances based on **scaling policies** (§15).
- **Registers with load balancers:** automatically attaches new instances to your target group(s).

### 🧠 The three capacity numbers (most important concept)
| Setting | Meaning | Example |
|---------|---------|---------|
| **Min** | Floor — never go below | 2 (always at least 2 for HA) |
| **Desired** | Target the ASG tries to maintain right now | 2 |
| **Max** | Ceiling — never exceed | 6 (cost guardrail) |

ASG keeps **Desired** running. Scaling policies change **Desired** up/down between **Min** and **Max**.

### 14.1 🖱️ Console — Create an ASG
1. **EC2 Console → Auto Scaling Groups → Create Auto Scaling group.**
2. **Name:** `web-asg`. **Launch template:** `web-lt` (version `$Latest`).
3. **Network:** select your **VPC** and **2+ subnets in 2+ AZs**.
4. **Load balancing:** *Attach to an existing load balancer → choose target group* `tg-web`.
5. **Health checks:** enable **ELB health checks** (so ASG replaces instances the LB says are unhealthy). Health check grace period: ~120–300s (time for boot + app start).
6. **Group size:** Desired `2`, Min `2`, Max `6`.
7. **Scaling policy:** add a **Target tracking** policy (e.g. average CPU 50%).
8. (Optional) Notifications (SNS), tags.
9. **Create Auto Scaling group.**

### 14.2 💻 CLI — Create an ASG
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template "LaunchTemplateName=web-lt,Version=\$Latest" \
  --min-size 2 \
  --max-size 6 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$(echo $SUBNETS | tr ' ' ',')" \  # comma-separated subnets
  --target-group-arns $TG_ARN \                            # auto-register with ALB TG
  --health-check-type ELB \                                # use LB health, not just EC2
  --health-check-grace-period 300 \                        # grace time after launch
  --tags "Key=Name,Value=web-asg,PropagateAtLaunch=true"
```

### 14.3 💻 Inspect & adjust
```bash
# See current instances and their lifecycle/health state
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names web-asg \
  --query 'AutoScalingGroups[0].Instances[].{Id:InstanceId,AZ:AvailabilityZone,State:LifecycleState,Health:HealthStatus}'

# Manually change desired capacity (e.g. force to 3)
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name web-asg --desired-capacity 3 --honor-cooldown
```

### 14.4 Termination policies & AZ balance
When scaling **in**, ASG decides *which* instance to kill using **termination policies** (default: balance AZs, then oldest launch template/config, then closest to next billing hour). You can customize (e.g. `OldestInstance`). ASG always tries to keep AZs **balanced**.

> **✅ Best Practice:** Always set **Min ≥ 2** across **≥ 2 AZs** for high availability. Use **ELB health check type** so the ASG replaces instances that fail the load balancer's checks, not just instances that fail the basic EC2 status check.

---

## 15. Scaling Policies (Target Tracking, Step, Simple, Scheduled, Predictive)

### 🧠 Concept
A **scaling policy** is the rule that changes **Desired Capacity** automatically. There are five approaches — pick based on how predictable your load is.

| Policy | How it works | Best for |
|--------|--------------|----------|
| **Target Tracking** | "Keep metric at a target value" (e.g. CPU 50%). AWS does the math. | **Default choice** — simple & effective |
| **Step Scaling** | Add/remove N instances based on alarm *magnitude* (bigger breach → bigger step). | Fine-grained control |
| **Simple Scaling** | One adjustment per alarm, then cooldown. Legacy. | Rarely — prefer the above |
| **Scheduled** | Scale at specific **times** (e.g. up at 8 AM, down at 8 PM). | Predictable daily/weekly patterns |
| **Predictive** | ML forecasts load and pre-scales ahead of demand. | Recurring, cyclical traffic |

### 15.1 💻 Target Tracking (the one you'll use most)
```bash
# Keep average CPU across the ASG at 50%. AWS creates the CloudWatch alarms for you.
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name cpu-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},
    "TargetValue":50.0
  }'
```
Other predefined metrics: `ASGAverageNetworkIn/Out`, and the very useful **`ALBRequestCountPerTarget`** (scale by requests per instance — great for web apps):
```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name req-per-target-1000 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification":{
      "PredefinedMetricType":"ALBRequestCountPerTarget",
      "ResourceLabel":"app/web-alb/abc123/targetgroup/tg-web/def456"
    },
    "TargetValue":1000.0
  }'
```
> `ResourceLabel` format: `app/<alb-name>/<alb-id>/targetgroup/<tg-name>/<tg-id>` — find the IDs in the ALB/TG ARNs.

### 15.2 💻 Step Scaling (with a CloudWatch alarm)
```bash
# 1) Create a step-scaling policy: +1 if breach is small, +2 if large
export STEP_POLICY=$(aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name cpu-step-out \
  --policy-type StepScaling \
  --adjustment-type ChangeInCapacity \
  --metric-aggregation-type Average \
  --step-adjustments \
    MetricIntervalLowerBound=0,MetricIntervalUpperBound=20,ScalingAdjustment=1 \
    MetricIntervalLowerBound=20,ScalingAdjustment=2 \
  --query PolicyARN --output text)

# 2) Alarm that triggers it when CPU > 70% for 2 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name web-asg-cpu-high \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 60 --evaluation-periods 2 \
  --threshold 70 --comparison-operator GreaterThanThreshold \
  --dimensions Name=AutoScalingGroupName,Value=web-asg \
  --alarm-actions $STEP_POLICY
```

### 15.3 💻 Scheduled Scaling (predictable patterns)
```bash
# Scale UP every weekday at 08:00 UTC
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name web-asg \
  --scheduled-action-name scale-up-morning \
  --recurrence "0 8 * * MON-FRI" \
  --min-size 4 --desired-capacity 4 --max-size 8

# Scale DOWN every day at 20:00 UTC
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name web-asg \
  --scheduled-action-name scale-down-night \
  --recurrence "0 20 * * *" \
  --min-size 2 --desired-capacity 2 --max-size 6
```

### 15.4 💻 Predictive Scaling
```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name predictive-cpu \
  --policy-type PredictiveScaling \
  --predictive-scaling-configuration '{
    "MetricSpecifications":[{
      "TargetValue":50,
      "PredefinedMetricPairSpecification":{"PredefinedMetricType":"ASGCPUUtilization"}
    }],
    "Mode":"ForecastAndScale"
  }'
```

### 🧠 Cooldown vs Warm-up
- **Cooldown** (simple scaling): wait period after a scaling action before another, so metrics settle.
- **Instance warm-up** (target tracking/step): how long a *new* instance is excluded from metrics while it boots and warms caches — prevents premature further scaling.

### 15.5 💻 Scaling on Custom Metrics & SQS Queue Depth
#### 🧠 Concept
CPU isn't always the right signal. **Worker / batch fleets** scale best on **backlog** — e.g. the number of messages waiting in an **SQS** queue. The recommended metric is **backlog per instance** = `ApproximateNumberOfMessagesVisible / RunningInstances`, used in a target-tracking policy with a **customized metric**.
```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name worker-asg --policy-name sqs-backlog \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "CustomizedMetricSpecification":{
      "MetricName":"ApproximateNumberOfMessagesVisible",
      "Namespace":"AWS/SQS",
      "Dimensions":[{"Name":"QueueName","Value":"jobs-queue"}],
      "Statistic":"Average"
    },
    "TargetValue":100.0
  }'
```
> **🏗️ Real-World:** Image/video processing, ETL, and email workers scale on queue depth so capacity tracks the actual job backlog — scale to zero when the queue is empty (set Min=0 for non-HA workers), surge when it fills.

### 15.6 🧠 Application Auto Scaling (beyond EC2)
The **Auto Scaling Group** scales **EC2**. A separate service, **Application Auto Scaling**, scales other resources with the *same* target-tracking/step concepts:
- **ECS services** (task count), **EKS** via Karpenter/HPA, **DynamoDB** (read/write capacity), **Aurora** replicas, **Lambda** provisioned concurrency, **SageMaker** endpoints, **Spot Fleet**.
```bash
# Example: scale an ECS service between 2 and 20 tasks on CPU
aws application-autoscaling register-scalable-target \
  --service-namespace ecs --resource-id service/my-cluster/my-svc \
  --scalable-dimension ecs:service:DesiredCount --min-capacity 2 --max-capacity 20
```
> Know the distinction in interviews: **EC2 Auto Scaling (ASG)** vs **Application Auto Scaling** (everything else).

> **✅ Best Practice:** Start with **one Target Tracking policy** (CPU or RequestCountPerTarget). Add **Scheduled** scaling for known patterns. Reserve Step/Predictive for advanced needs. Combine **scale-out fast, scale-in slow** to avoid flapping.

---

## 16. Connecting ASG to ALB/NLB (The Full Loop)

### 🧠 Concept — how the pieces click together
```
Launch Template  ──►  Auto Scaling Group  ──registers──►  Target Group  ◄──forward── Listener ◄── Load Balancer ◄── Internet
   (blueprint)          (manages fleet)                   (health checks)              (rules)         (ALB/NLB)
```
1. The **ASG** uses the **Launch Template** to start instances.
2. The ASG **automatically registers** each new instance into the **Target Group**.
3. The **Load Balancer's Listener** forwards traffic to that Target Group.
4. **Health checks** decide which instances receive traffic; the ASG **replaces** unhealthy ones.
5. When scaling **in**, the ASG **deregisters** instances (honoring connection draining) before terminating.

### 💻 Attach/detach an ASG to a target group (if not done at creation)
```bash
# Attach
aws autoscaling attach-load-balancer-target-groups \
  --auto-scaling-group-name web-asg --target-group-arns $TG_ARN

# Verify the ASG knows about the target group
aws autoscaling describe-load-balancer-target-groups \
  --auto-scaling-group-name web-asg

# Detach (e.g. during blue/green cutover)
aws autoscaling detach-load-balancer-target-groups \
  --auto-scaling-group-name web-asg --target-group-arns $TG_ARN
```

### 🧠 Connection draining (deregistration delay)
When an instance is removed, the LB stops *new* connections but lets *in-flight* requests finish for up to the **deregistration delay** (default 300s). Tune it on the target group:
```bash
aws elbv2 modify-target-group-attributes --target-group-arn $TG_ARN \
  --attributes Key=deregistration_delay.timeout_seconds,Value=60
```

> **🏗️ Real-World:** This is the standard 3-tier web pattern. The ALB gives a stable DNS name and HTTPS; the ASG gives elasticity and self-healing; the target group glues them with health checks. Everything new instance the ASG launches is automatically wired in — no manual registration.

---

## 17. Lifecycle Hooks, Health Checks & Instance Refresh

### 17.1 Health check types
- **EC2 health check:** only checks the hypervisor/instance status (is the VM alive?).
- **ELB health check:** checks your **app** via the target group's health check. **Use ELB** so the ASG replaces app-level failures.
```bash
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --health-check-type ELB --health-check-grace-period 300
```

### 17.2 Lifecycle Hooks
#### 🧠 Concept
A **lifecycle hook** pauses an instance during **launch** (`Pending:Wait`) or **terminate** (`Terminating:Wait`) so you can run custom logic — register with a config system, warm caches, drain connections, copy logs off the box — *before* it goes into service or is destroyed.
```bash
# Pause on launch for up to 300s so bootstrap/registration can complete
aws autoscaling put-lifecycle-hook \
  --auto-scaling-group-name web-asg \
  --lifecycle-hook-name on-launch \
  --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING \
  --heartbeat-timeout 300 \
  --default-result ABANDON

# Your automation signals completion when ready:
aws autoscaling complete-lifecycle-action \
  --auto-scaling-group-name web-asg \
  --lifecycle-hook-name on-launch \
  --lifecycle-action-result CONTINUE \
  --instance-id i-0123456789abcdef0
```

### 17.3 Instance Refresh (rolling deployments)
#### 🧠 Concept
**Instance Refresh** rolls out a new Launch Template version (e.g. new AMI/app) by **gradually replacing** instances while keeping the service available — the built-in way to do zero-downtime deploys with ASGs.
```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name web-asg \
  --preferences '{"MinHealthyPercentage":90,"InstanceWarmup":120}'

# Check progress
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name web-asg \
  --query 'InstanceRefreshes[0].{Status:Status,Pct:PercentageComplete}'

# Roll back if something is wrong
aws autoscaling rollback-instance-refresh --auto-scaling-group-name web-asg
```

### 17.4 Warm Pools (faster scale-out)
A **warm pool** keeps pre-initialized (stopped/hibernated) instances ready so scale-out is near-instant for slow-booting apps.
```bash
aws autoscaling put-warm-pool --auto-scaling-group-name web-asg \
  --min-size 2 --pool-state Stopped
```

> **✅ Best Practice:** Use **ELB health checks + a real `/health` endpoint**, **Instance Refresh** for deployments (with `MinHealthyPercentage` ≥ 90), and **lifecycle hooks** when instances need setup/cleanup beyond user data.

### 17.5 Suspend & Resume Scaling Processes
#### 🧠 Concept
You can temporarily **suspend** individual ASG processes (e.g. during maintenance or debugging) without deleting the ASG. Common processes: `Launch`, `Terminate`, `HealthCheck`, `ReplaceUnhealthy`, `AZRebalance`, `AlarmNotification`, `ScheduledActions`, `AddToLoadBalancer`.
```bash
# Pause scaling/healing while you investigate (e.g. stop replacing instances)
aws autoscaling suspend-processes --auto-scaling-group-name web-asg \
  --scaling-processes HealthCheck ReplaceUnhealthy AZRebalance

# Resume everything when done
aws autoscaling resume-processes --auto-scaling-group-name web-asg
```
> **🏗️ Real-World:** Suspend `AZRebalance` and `Terminate` during a delicate manual debug so the ASG doesn't kill the instance you're inspecting.

### 17.6 Standby State (pull an instance out without losing it)
#### 🧠 Concept
Move an instance to **Standby** to take it out of service (deregistered from the LB) for patching/forensics while keeping it in the ASG. It still counts toward desired capacity unless you decrement.
```bash
# Put into standby (and reduce desired so a replacement isn't launched)
aws autoscaling enter-standby --auto-scaling-group-name web-asg \
  --instance-ids i-0123456789abcdef0 --should-decrement-desired-capacity

# Return it to service when finished
aws autoscaling exit-standby --auto-scaling-group-name web-asg \
  --instance-ids i-0123456789abcdef0
```

### 17.7 Instance Scale-In Protection
#### 🧠 Concept
Protect specific instances from being **terminated during scale-in** (e.g. a node running a long batch job). Scaling-out still works; only scale-in skips protected instances.
```bash
aws autoscaling set-instance-protection --auto-scaling-group-name web-asg \
  --instance-ids i-0123456789abcdef0 --protected-from-scale-in
```

### 17.8 Max Instance Lifetime (forced rotation)
#### 🧠 Concept
Force the ASG to **replace every instance after N seconds** (min 86,400s = 1 day). Great for compliance/hygiene — guarantees fresh, patched instances and prevents long-lived configuration drift.
```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name web-asg \
  --max-instance-lifetime 604800   # rotate every 7 days
```

### 17.9 Capacity Rebalancing (Spot resilience)
#### 🧠 Concept
With Spot instances, **capacity rebalancing** proactively launches a replacement when AWS signals an instance is at **elevated risk of interruption** — *before* the 2-minute termination notice — so you keep capacity during Spot reclaims.
```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name cost-asg \
  --capacity-rebalance
```

### 17.10 Attach / Detach Individual Instances
#### 🧠 Concept
You can bring an **existing standalone EC2 instance** under ASG management, or remove one from the ASG without terminating it (e.g. to debug it independently).
```bash
# Attach an existing instance to the ASG
aws autoscaling attach-instances --auto-scaling-group-name web-asg \
  --instance-ids i-0123456789abcdef0

# Detach (optionally keep desired capacity by launching a replacement)
aws autoscaling detach-instances --auto-scaling-group-name web-asg \
  --instance-ids i-0123456789abcdef0 --should-decrement-desired-capacity
```

### 17.11 Notifications (SNS) & EventBridge
#### 🧠 Concept
Get notified on launch/terminate/failure events for auditing or automation.
```bash
aws autoscaling put-notification-configuration --auto-scaling-group-name web-asg \
  --topic-arn arn:aws:sns:us-east-1:111122223333:asg-events \
  --notification-types \
    autoscaling:EC2_INSTANCE_LAUNCH autoscaling:EC2_INSTANCE_TERMINATE \
    autoscaling:EC2_INSTANCE_LAUNCH_ERROR autoscaling:EC2_INSTANCE_TERMINATE_ERROR
```
> EventBridge can also match ASG events to trigger Lambda/automation pipelines.

---

## 18. Real-Time Architecture Approaches & Patterns

### 🏗️ Pattern A — Classic Public Web App (most common)
```
Route 53 ──► ALB (public, HTTPS) ──► Target Group ──► ASG (EC2 in private subnets, 2+ AZs) ──► RDS (Multi-AZ)
```
- ALB in **public** subnets; EC2 in **private** subnets (NAT for outbound).
- Target tracking on CPU or `ALBRequestCountPerTarget`.
- HTTPS via ACM; HTTP→HTTPS redirect; WAF on the ALB.

### 🏗️ Pattern B — Microservices on one ALB
- One ALB, **path/host rules** → many target groups → many ASGs (one per service).
- Independent scaling per service. Blue/green via weighted target groups.

### 🏗️ Pattern C — NLB in front of ALB (static IP + L7)
```
Clients (need fixed IP / allow-list) ──► NLB (Elastic IP) ──► ALB ──► Target Groups ──► ASG
```
- Get **static/Elastic IPs** from the NLB *and* keep ALB's smart L7 routing.

### 🏗️ Pattern D — TCP/UDP service
```
Clients ──► NLB (TCP/UDP) ──► Target Group ──► ASG (e.g. game servers, MQTT brokers)
```

### 🏗️ Pattern E — Containers
- **ECS/EKS** services register tasks/pods as **IP targets** in target groups; the ALB is the ingress; scaling handled by ECS Service Auto Scaling / Kubernetes HPA + Cluster Autoscaler/Karpenter (ASG underneath for EC2 nodes).

### 🏗️ Pattern F — Internal/private app
- **Internal** ALB/NLB (scheme `internal`) for service-to-service traffic inside the VPC; no internet exposure.

> **Decision flow:** *HTTP app?* → ALB. *Need static IP / TCP-UDP?* → NLB (optionally → ALB). *Elastic backend?* → ASG behind the target group. *Predictable spikes?* → add Scheduled/Predictive scaling.

---

## 19. End-to-End Capstone Project (Console + CLI)

> **Goal:** Build a highly available web app: **Internet → ALB (HTTPS) → Target Group → Auto Scaling Group (2–6 EC2 across 2 AZs)** with target-tracking scaling and self-healing. Then test failover and scaling.

### Step 0 — Variables (from §4)
Ensure `$AWS_REGION`, `$VPC_ID`, `$SUBNETS` are set.

### Step 1 — Security Groups
```bash
# ALB SG: allow 80/443 from internet
export ALB_SG=$(aws ec2 create-security-group --group-name cap-alb-sg \
  --description "ALB" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0

# Instance SG: allow app port 80 ONLY from the ALB SG
export INST_SG=$(aws ec2 create-security-group --group-name cap-inst-sg \
  --description "Instances" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $INST_SG --protocol tcp --port 80 --source-group $ALB_SG
```

### Step 2 — Target Group
```bash
export TG_ARN=$(aws elbv2 create-target-group --name cap-tg \
  --protocol HTTP --port 80 --vpc-id $VPC_ID --target-type instance \
  --health-check-path /health --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 2 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
```

### Step 3 — ALB + Listener
```bash
export ALB_ARN=$(aws elbv2 create-load-balancer --name cap-alb \
  --type application --scheme internet-facing \
  --subnets $SUBNETS --security-groups $ALB_SG \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

export LISTENER_ARN=$(aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --query 'Listeners[0].ListenerArn' --output text)

export ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query 'LoadBalancers[0].DNSName' --output text)
echo "Open: http://$ALB_DNS"
```

### Step 4 — Launch Template (web server + /health)
```bash
cat > userdata.sh <<'EOF'
#!/bin/bash
dnf install -y httpd
systemctl enable --now httpd
echo "Served by $(hostname -f)" > /var/www/html/index.html
echo "OK" > /var/www/html/health
EOF
export USERDATA=$(base64 -w0 userdata.sh)

export AMI_ID=$(aws ssm get-parameters \
  --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --query 'Parameters[0].Value' --output text)

aws ec2 create-launch-template --launch-template-name cap-lt \
  --launch-template-data '{
    "ImageId":"'"$AMI_ID"'","InstanceType":"t3.micro",
    "SecurityGroupIds":["'"$INST_SG"'"],"UserData":"'"$USERDATA"'",
    "TagSpecifications":[{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"cap-web"}]}]
  }'
```

### Step 5 — Auto Scaling Group (wired to the ALB)
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name cap-asg \
  --launch-template "LaunchTemplateName=cap-lt,Version=\$Latest" \
  --min-size 2 --max-size 6 --desired-capacity 2 \
  --vpc-zone-identifier "$(echo $SUBNETS | tr ' ' ',')" \
  --target-group-arns $TG_ARN \
  --health-check-type ELB --health-check-grace-period 300 \
  --tags "Key=Name,Value=cap-web,PropagateAtLaunch=true"
```

### Step 6 — Target Tracking Scaling Policy
```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name cap-asg --policy-name cpu-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},
    "TargetValue":50.0}'
```

### Step 7 — Verify
```bash
# Wait ~2-3 min, then targets should be healthy:
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].TargetHealth.State'

# Hit the app repeatedly — note the hostname rotates across instances:
for i in $(seq 1 6); do curl -s http://$ALB_DNS; echo; done
```

### Step 8 — Test self-healing (failover)
```bash
# Terminate one instance; the ASG must launch a replacement automatically
INST=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names cap-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)
aws ec2 terminate-instances --instance-ids $INST
# Watch the ASG bring capacity back to 2:
watch -n5 "aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names cap-asg \
  --query 'AutoScalingGroups[0].Instances[].LifecycleState'"
```

### Step 9 — Test scale-out (load)
```bash
# Generate CPU load on the instances (SSM or SSH), or use a load tool against the ALB:
# e.g. from a separate box:  ab -n 200000 -c 200 http://$ALB_DNS/
# Watch desired capacity rise toward Max as CPU passes 50%.
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names cap-asg \
  --query 'AutoScalingGroups[0].DesiredCapacity'
```

### Step 10 — 🔥 Clean up (avoid charges!)
```bash
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name cap-asg --force-delete
aws ec2 delete-launch-template --launch-template-name cap-lt
aws elbv2 delete-listener --listener-arn $LISTENER_ARN
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
sleep 30   # let the LB detach before deleting the target group/SGs
aws elbv2 delete-target-group --target-group-arn $TG_ARN
aws ec2 delete-security-group --group-id $INST_SG
aws ec2 delete-security-group --group-id $ALB_SG
```

> **📝 Assignment:** Repeat the capstone with **(a)** an HTTPS listener + ACM cert, **(b)** a second target group + path rule `/api/*`, and **(c)** a scheduled scale-up at a chosen time. Tear everything down afterward.

---

## 20. Infrastructure as Code (Approaches)

> In real projects you rarely click or run ad-hoc CLI for production — you codify infrastructure so it's repeatable, reviewable, and versioned.

### 🏗️ Approach 1 — Terraform (most common multi-cloud IaC)
```hcl
resource "aws_lb" "web" {
  name               = "web-alb"
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnets
}

resource "aws_lb_target_group" "web" {
  name        = "tg-web"
  port        = 80
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "instance"
  health_check { path = "/health" }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.web.arn
  }
}

resource "aws_launch_template" "web" {
  name_prefix   = "web-lt-"
  image_id      = var.ami_id
  instance_type = "t3.micro"
  user_data     = base64encode(file("userdata.sh"))
  vpc_security_group_ids = [aws_security_group.instance.id]
}

resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  min_size            = 2
  max_size            = 6
  desired_capacity    = 2
  vpc_zone_identifier = var.private_subnets
  target_group_arns   = [aws_lb_target_group.web.arn]
  health_check_type   = "ELB"
  launch_template { id = aws_launch_template.web.id, version = "$Latest" }
}

resource "aws_autoscaling_policy" "cpu" {
  name                   = "cpu-50"
  autoscaling_group_name = aws_autoscaling_group.web.name
  policy_type            = "TargetTrackingScaling"
  target_tracking_configuration {
    predefined_metric_specification { predefined_metric_type = "ASGAverageCPUUtilization" }
    target_value = 50
  }
}
```

### 🏗️ Approach 2 — AWS CloudFormation (native AWS IaC)
Use resource types `AWS::ElasticLoadBalancingV2::LoadBalancer`, `::TargetGroup`, `::Listener`, `AWS::EC2::LaunchTemplate`, `AWS::AutoScaling::AutoScalingGroup`, and `AWS::AutoScaling::ScalingPolicy`.

### 🏗️ Approach 3 — AWS CDK
Define the same constructs in TypeScript/Python with high-level abstractions (`elbv2.ApplicationLoadBalancer`, `autoscaling.AutoScalingGroup`).

> **✅ Best Practice:** Use **CLI/Console to learn and prototype**, then **Terraform/CloudFormation/CDK** for anything that lives longer than a demo. Store state remotely, review changes via pull requests, and deploy through CI/CD.

---

## 21. Monitoring, Logging & Observability

### Key CloudWatch metrics
| Source | Metric | Why it matters |
|--------|--------|----------------|
| ALB | `RequestCount` | Total traffic volume |
| ALB | `TargetResponseTime` | App latency |
| ALB | `HTTPCode_Target_5XX_Count` | App errors |
| ALB | `HTTPCode_ELB_5XX_Count` | LB-side errors (e.g. no healthy targets) |
| ALB | `HealthyHostCount` / `UnHealthyHostCount` | Fleet health |
| ALB | `RequestCountPerTarget` | Drives request-based scaling |
| NLB | `ActiveFlowCount`, `ProcessedBytes`, `TCP_*_Reset` | Connection health |
| ASG | `GroupDesiredCapacity`, `GroupInServiceInstances` | Scaling behavior |
| EC2 | `CPUUtilization` | Common scaling signal |

### 💻 Enable ALB access logs (to S3)
```bash
aws elbv2 modify-load-balancer-attributes --load-balancer-arn $ALB_ARN \
  --attributes Key=access_logs.s3.enabled,Value=true \
               Key=access_logs.s3.bucket,Value=my-alb-logs
```

### 💻 Useful alarms
```bash
# Alert when there are unhealthy hosts
aws cloudwatch put-metric-alarm --alarm-name alb-unhealthy \
  --namespace AWS/ApplicationELB --metric-name UnHealthyHostCount \
  --dimensions Name=LoadBalancer,Value=app/web-alb/abc Name=TargetGroup,Value=targetgroup/tg-web/def \
  --statistic Average --period 60 --evaluation-periods 2 \
  --threshold 0 --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:...:alerts
```

> **✅ Best Practice:** Watch **5XX**, **TargetResponseTime**, and **UnHealthyHostCount**. Enable **access logs** for forensic analysis. Add an **SNS** topic so alarms page you. Consider **CloudWatch ServiceLens / X-Ray** for distributed tracing.

---

## 22. Security Best Practices
- **Least-privilege Security Groups:** ALB SG allows 80/443 from internet; **instance SG allows only the ALB SG** on the app port. Never expose instances directly.
- **Always HTTPS:** ACM cert on the LB, redirect HTTP→HTTPS, TLS 1.2/1.3 policy.
- **WAF on public ALBs:** block SQLi/XSS, add rate limiting and geo rules.
- **Private subnets for instances:** outbound via NAT; no public IPs on backends.
- **IAM roles, not keys:** attach instance profiles; use SSM Session Manager instead of SSH/key pairs.
- **Enable deletion protection** on production load balancers.
- **Enable access logs** and retain them.
- **Encrypt** EBS volumes (launch template) and S3 log buckets.
- **NLB + source IP preservation:** remember backends see the real client IP — size SGs accordingly.

---

## 23. Cost Optimization
- **Delete idle load balancers** — they bill hourly + per **LCU** even with no traffic.
- **Scale in aggressively at night** with scheduled scaling; set a sensible **Max** as a guardrail.
- **Use Spot Instances** in the ASG (mixed instances policy) for fault-tolerant fleets — big savings.
- **Right-size** instance types; prefer Graviton (`t4g`, `c7g`) for price/performance.
- **Mind cross-zone charges** on NLB (data transfer between AZs when enabled).
- **Consolidate** microservices onto **one ALB** with host/path rules instead of many LBs.
- **Clean up unused target groups, launch template versions, and EIPs.**

### 💻 Mixed instances + Spot (cost-optimized ASG)
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name cost-asg \
  --min-size 2 --max-size 10 --desired-capacity 2 \
  --vpc-zone-identifier "$(echo $SUBNETS | tr ' ' ',')" \
  --target-group-arns $TG_ARN --health-check-type ELB \
  --mixed-instances-policy '{
    "LaunchTemplate":{
      "LaunchTemplateSpecification":{"LaunchTemplateName":"web-lt","Version":"$Latest"},
      "Overrides":[{"InstanceType":"t3.micro"},{"InstanceType":"t3a.micro"},{"InstanceType":"t4g.micro"}]
    },
    "InstancesDistribution":{
      "OnDemandBaseCapacity":1,
      "OnDemandPercentageAboveBaseCapacity":25,
      "SpotAllocationStrategy":"price-capacity-optimized"
    }
  }'
```

---

## 24. Troubleshooting Playbook

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| **503 from ALB** | No healthy targets | Check target health, SG rules, health-check path returns 2xx |
| **502 from ALB** | App crashed / bad response / wrong port | Check app logs, listening port, keep-alive |
| **504 from ALB** | Backend slower than idle timeout | Optimize app or raise idle timeout |
| **Targets stuck unhealthy** | Instance SG doesn't allow ALB SG; wrong path/port | Allow ALB SG on health port; fix path |
| **ASG launches then terminates instances repeatedly** | Failing ELB health check during boot | Increase **health-check grace period**; fix `/health` |
| **ASG not scaling out** | Policy/alarm misconfigured or Max reached | Check alarm state, Max size, metric data |
| **NLB client can't connect, targets healthy** | Source-IP preservation; target SG blocks client | Allow client CIDR on target SG |
| **Uneven AZ traffic** | Cross-zone LB off (NLB) | Enable cross-zone (mind cost) |
| **New deploy didn't roll out** | ASG still on old LT version | Update default version + run **Instance Refresh** |
| **Sticky sessions not working** | Stickiness off / cookie mismatch | Enable stickiness on the target group |
| **HTTPS cert error** | Cert not ISSUED / wrong domain | Validate ACM cert; match domain/SAN |

### 💻 Fast diagnostics
```bash
# Why is a target unhealthy? (Reason + Description are gold)
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query 'TargetHealthDescriptions[].{Id:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason,Desc:TargetHealth.Description}'

# Recent ASG activity (launches, terminations, failures + causes)
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name web-asg --max-items 10 \
  --query 'Activities[].{Status:StatusCode,Cause:Cause,Time:StartTime}'
```

---

## 25. Best Practices Cheat Sheet
- ✅ **Min ≥ 2 instances across ≥ 2 AZs** — always, for HA.
- ✅ Use **ALB** for HTTP, **NLB** for TCP/UDP/static-IP/extreme performance.
- ✅ **ELB health checks** on the ASG + a real lightweight **`/health`** endpoint.
- ✅ **Launch Templates** (not Launch Configurations) with **versioning**.
- ✅ **Target Tracking** scaling first; add Scheduled/Predictive as needed.
- ✅ **HTTPS everywhere** via ACM; redirect HTTP→HTTPS; modern TLS policy.
- ✅ Instance SG allows **only the LB SG** on the app port.
- ✅ **Instance Refresh** with `MinHealthyPercentage ≥ 90` for zero-downtime deploys.
- ✅ Tune **deregistration delay** (connection draining) for graceful scale-in.
- ✅ **Enable access logs**, alarms on 5XX/latency/unhealthy hosts.
- ✅ **Spot + mixed instances** for cost; **Scheduled** scale-down at night.
- ✅ **Delete lab resources** to avoid charges.
- ✅ Codify with **Terraform/CloudFormation/CDK** for anything beyond a demo.

---

## 26. CLI Command Quick Reference

```bash
# ---------- Load Balancers (elbv2) ----------
aws elbv2 create-load-balancer ...                 # create ALB/NLB
aws elbv2 describe-load-balancers                  # list LBs
aws elbv2 create-target-group ...                  # create TG
aws elbv2 register-targets / deregister-targets    # add/remove targets
aws elbv2 describe-target-health --target-group-arn $TG  # health status
aws elbv2 create-listener / modify-listener        # listeners
aws elbv2 create-rule / modify-rule                # routing rules
aws elbv2 modify-target-group-attributes           # stickiness, draining
aws elbv2 modify-load-balancer-attributes          # logs, idle timeout, deletion protection
aws elbv2 delete-listener / delete-target-group / delete-load-balancer

# ---------- Launch Templates (ec2) ----------
aws ec2 create-launch-template ...
aws ec2 create-launch-template-version ...
aws ec2 modify-launch-template --default-version N
aws ec2 describe-launch-templates

# ---------- Auto Scaling ----------
aws autoscaling create-auto-scaling-group ...
aws autoscaling update-auto-scaling-group ...
aws autoscaling set-desired-capacity ...
aws autoscaling put-scaling-policy ...             # target/step/predictive
aws autoscaling put-scheduled-update-group-action  # scheduled
aws autoscaling attach/detach-load-balancer-target-groups
aws autoscaling put-lifecycle-hook ...
aws autoscaling start-instance-refresh ...
aws autoscaling describe-scaling-activities ...
aws autoscaling delete-auto-scaling-group --force-delete

# ---------- TLS / Certs ----------
aws acm request-certificate ...
aws acm describe-certificate ...
```

---

## 27. Interview Questions

**Load Balancing**
1. Difference between ALB, NLB, GWLB, and Classic LB? When would you pick each?
2. Layer 4 vs Layer 7 load balancing — what can ALB do that NLB cannot?
3. What is a target group and a listener? How do rules work?
4. How do health checks work? What happens to an unhealthy target?
5. How do you achieve HTTPS? What is TLS termination and how does ACM help?
6. What is cross-zone load balancing? Default behavior on ALB vs NLB?
7. How do sticky sessions work and when are they a bad idea?
8. How would you do path-based / host-based routing for microservices?
9. How do you give a load balancer a static IP?
10. How does NLB source IP preservation affect security groups?
11. What causes 502 vs 503 vs 504 from an ALB?
12. How do you do blue/green or canary with an ALB?

**Auto Scaling**
13. Explain Min, Desired, and Max capacity.
14. Launch Template vs Launch Configuration — why prefer templates?
15. Compare target tracking, step, simple, scheduled, and predictive scaling.
16. How does an ASG self-heal? EC2 vs ELB health check type?
17. What is a lifecycle hook and when do you use one?
18. What is Instance Refresh and how does it enable zero-downtime deploys?
19. How does an ASG integrate with an ALB target group?
20. What are cooldown and instance warm-up? Why do they matter?
21. How would you use Spot instances safely in an ASG?
22. How do you scale based on requests per instance rather than CPU?
23. What is a warm pool and what problem does it solve?
24. Design a highly available, cost-optimized web tier end to end.

---

## 28. Glossary
- **ELB** — Elastic Load Balancing (the service family).
- **ALB / NLB / GWLB / CLB** — Application / Network / Gateway / Classic load balancers.
- **Listener** — protocol+port the LB listens on, with rules.
- **Target Group** — bucket of backends with health checks.
- **Target** — a registered backend (instance/IP/Lambda).
- **Health Check** — periodic probe deciding target health.
- **Sticky Session** — pin a client to one target via cookie.
- **Cross-Zone LB** — even traffic distribution across all AZs.
- **Deregistration Delay / Connection Draining** — grace period for in-flight requests on removal.
- **ACM** — AWS Certificate Manager (free TLS certs).
- **Launch Template** — versioned EC2 launch blueprint.
- **User Data** — startup script run on first boot.
- **ASG** — Auto Scaling Group.
- **Min / Desired / Max** — ASG capacity bounds.
- **Scaling Policy** — rule that changes desired capacity.
- **Target Tracking** — keep a metric at a target value.
- **Step / Simple / Scheduled / Predictive Scaling** — other scaling strategies.
- **Cooldown / Warm-up** — stabilization pauses.
- **Lifecycle Hook** — pause on launch/terminate for custom logic.
- **Instance Refresh** — rolling replacement of instances.
- **Warm Pool** — pre-initialized standby instances.
- **LCU** — Load Balancer Capacity Unit (billing metric).
- **AZ** — Availability Zone.
- **AMI** — Amazon Machine Image.

---

> **You now have a complete, project-ready reference.** Work through the **Capstone (§19)** hands-on, then rebuild it with **Terraform (§20)**. Master the **decision guide (§10)**, **scaling policies (§15)**, and **troubleshooting (§24)** — those three carry you through real projects and interviews. **Always delete lab resources to avoid charges.**
