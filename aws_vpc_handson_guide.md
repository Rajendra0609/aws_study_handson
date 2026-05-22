# AWS VPC — Practical Hands-On Learning Guide

> A structured, beginner-to-advanced guide for mastering AWS Virtual Private Cloud (VPC) through real console practice.

---

## Table of Contents

1. [Prerequisites & Setup](#prerequisites--setup)
2. [BEGINNER — Foundations](#beginner--foundations)
   - [Networking Basics: CIDR, Private vs Public IP](#1-networking-basics-cidr-private-vs-public-ip)
   - [Default VPC Overview](#2-default-vpc-overview)
   - [VPC Overview & Creation](#3-vpc-overview--creation)
   - [Subnets](#4-subnets)
   - [Internet Gateways & Route Tables](#5-internet-gateways--route-tables)
3. [INTERMEDIATE — Connectivity & Security](#intermediate--connectivity--security)
   - [Bastion Hosts](#6-bastion-hosts)
   - [NAT Instances](#7-nat-instances)
   - [NAT Gateways](#8-nat-gateways)
   - [Regional NAT Gateway](#9-regional-nat-gateway)
   - [NACLs & Security Groups](#10-nacls--security-groups)
4. [ADVANCED — Networking Deep Dive](#advanced--networking-deep-dive)
   - [VPC Peering](#11-vpc-peering)
   - [VPC Endpoints](#12-vpc-endpoints)
   - [VPC Flow Logs + Athena](#13-vpc-flow-logs--athena)
   - [Site-to-Site VPN, Virtual Private Gateway & Customer Gateway](#14-site-to-site-vpn-virtual-private-gateway--customer-gateway)
   - [Direct Connect & Direct Connect Gateway](#15-direct-connect--direct-connect-gateway)
   - [Direct Connect + Site-to-Site VPN](#16-direct-connect--site-to-site-vpn)
   - [Transit Gateway](#17-transit-gateway)
   - [VPC Traffic Mirroring](#18-vpc-traffic-mirroring)
5. [ADVANCED — IPv6, Egress & Firewall](#advanced--ipv6-egress--firewall)
   - [IPv6 for VPC](#19-ipv6-for-vpc)
   - [Egress-Only Internet Gateway](#20-egress-only-internet-gateway)
   - [AWS Network Firewall](#21-aws-network-firewall)
6. [Networking Costs in AWS](#22-networking-costs-in-aws)
7. [Section Cleanup Checklist](#section-cleanup-checklist)
8. [VPC Section Summary Cheat Sheet](#vpc-section-summary-cheat-sheet)

---

## Prerequisites & Setup

Before you begin, make sure you have:

- An **AWS Account** (Free Tier is sufficient for most exercises)
- **IAM User** with Administrator or VPC Full Access permissions
- **AWS CLI** installed and configured (`aws configure`)
- Basic understanding of networking (IP addresses, TCP/UDP, DNS)
- A terminal/SSH client (e.g., PuTTY on Windows, Terminal on Mac/Linux)

> 💡 **Tip:** Always work in a **non-default region** (e.g., `eu-west-1`) during practice so you don't accidentally interfere with anything in your default region.

---

## BEGINNER — Foundations

---

### 1. Networking Basics: CIDR, Private vs Public IP

#### Concepts to Understand

| Concept | Description |
|---|---|
| **CIDR** | Classless Inter-Domain Routing — a method for allocating IP addresses (e.g., `10.0.0.0/16`) |
| **Private IP Ranges** | `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16` — not routable on the internet |
| **Public IP** | Globally routable IP assigned by ISPs or AWS |
| **Subnet Mask** | Defines the network and host portions (e.g., `/24` = 256 addresses, `/16` = 65,536) |

#### Key Formulas

```
Total IPs in a CIDR block = 2^(32 - prefix)
Usable IPs = Total - 5  (AWS reserves 5 IPs per subnet)

Example: 10.0.0.0/24
  Total IPs  = 2^(32-24) = 256
  Usable IPs = 256 - 5   = 251
```

#### AWS Reserved IPs (per subnet)

| Address | Reserved For |
|---|---|
| x.x.x.0 | Network address |
| x.x.x.1 | VPC Router |
| x.x.x.2 | AWS DNS |
| x.x.x.3 | Future use |
| x.x.x.255 | Broadcast address |

#### Practice Exercise

1. Use [CIDR.xyz](https://cidr.xyz) or [Visual Subnet Calculator](https://www.davidc.net/sites/default/subnets/subnets.html) to visualise a `/16` block.
2. Plan a VPC CIDR of `10.0.0.0/16` and divide it into four `/24` subnets (public + private across 2 AZs).
3. Write down which IPs AWS will reserve in each subnet.

> 💡 **Tip:** Always plan your CIDR ranges before creating a VPC — they **cannot be changed** after creation.

---

### 2. Default VPC Overview

#### Concepts to Understand

- AWS creates a **Default VPC** in every region automatically.
- CIDR: `172.31.0.0/16`
- Contains one **default public subnet per AZ**, each with a `/20` CIDR.
- Instances launched here get both a **public and private IP** by default.
- An **Internet Gateway** is already attached.
- Never delete your default VPC unless you are sure — it can be recreated, but it's tedious.

#### Console Walkthrough

1. Go to **VPC Console → Your VPCs**.
2. Identify the Default VPC (tagged "Yes" under Default VPC).
3. Go to **Subnets** — note one subnet per AZ with `Auto-assign public IPv4` enabled.
4. Go to **Internet Gateways** — observe the attached IGW.
5. Go to **Route Tables** — check the main route table with `0.0.0.0/0 → IGW`.

> 💡 **Tip:** For production, **never use the default VPC**. Always create a custom VPC with private subnets.

---

### 3. VPC Overview & Creation

#### Concepts to Understand

- A VPC is a **logically isolated virtual network** within AWS.
- Spans **all AZs in a region**.
- You define the IPv4 CIDR block (e.g., `10.0.0.0/16`).
- Supports optional IPv6 CIDR block.
- Components: Subnets, Route Tables, IGW, NAT, Security Groups, NACLs.

#### Hands-On: Create a Custom VPC

**Step 1 — Create the VPC**
```
VPC Console → Your VPCs → Create VPC
  Name: my-custom-vpc
  IPv4 CIDR: 10.0.0.0/16
  Tenancy: Default
→ Create
```

**Step 2 — Enable DNS Settings**
```
Select VPC → Actions → Edit VPC Settings
  ✅ Enable DNS resolution
  ✅ Enable DNS hostnames
→ Save
```

> 💡 **Tip:** DNS hostnames must be enabled if you want EC2 instances to get public DNS names.

**Step 3 — Verify**
- Confirm the VPC appears with state `Available`.
- Note the VPC ID — you will use it throughout all exercises.

---

### 4. Subnets

#### Concepts to Understand

| Type | Description |
|---|---|
| **Public Subnet** | Has a route to an Internet Gateway; instances can be directly reachable |
| **Private Subnet** | No direct route to the internet; uses NAT for outbound access |
| **AZ Placement** | Each subnet lives in exactly **one Availability Zone** |

#### Hands-On: Create Public & Private Subnets

**Create Public Subnet (AZ-a)**
```
VPC Console → Subnets → Create Subnet
  VPC: my-custom-vpc
  Name: public-subnet-1a
  AZ: <region>a
  IPv4 CIDR: 10.0.1.0/24
→ Create
```

**Enable Auto-Assign Public IP on Public Subnet**
```
Select public-subnet-1a → Actions → Edit Subnet Settings
  ✅ Enable auto-assign public IPv4 address
→ Save
```

**Create Private Subnet (AZ-a)**
```
  Name: private-subnet-1a
  AZ: <region>a
  IPv4 CIDR: 10.0.2.0/24
```

**Create Subnets in AZ-b** (repeat with `10.0.3.0/24` public, `10.0.4.0/24` private)

> 💡 **Tip:** Use a consistent naming scheme like `<env>-<type>-subnet-<az>` (e.g., `prod-private-subnet-1a`) to avoid confusion.

---

### 5. Internet Gateways & Route Tables

#### Concepts to Understand

| Component | Role |
|---|---|
| **Internet Gateway (IGW)** | Allows VPC resources to communicate with the internet |
| **Route Table** | A set of rules (routes) that determine where network traffic is directed |
| **Main Route Table** | Default table associated with all subnets unless explicitly overridden |
| **Custom Route Table** | Explicitly associated with specific subnets |

#### Hands-On: Attach an IGW & Configure Routes

**Step 1 — Create & Attach an Internet Gateway**
```
VPC Console → Internet Gateways → Create Internet Gateway
  Name: my-igw
→ Create
→ Actions → Attach to VPC → select my-custom-vpc
```

**Step 2 — Create a Public Route Table**
```
VPC Console → Route Tables → Create Route Table
  Name: public-rt
  VPC: my-custom-vpc
→ Create
```

**Step 3 — Add Internet Route**
```
Select public-rt → Routes tab → Edit Routes → Add Route
  Destination: 0.0.0.0/0
  Target: Internet Gateway → my-igw
→ Save
```

**Step 4 — Associate Public Subnets**
```
public-rt → Subnet Associations → Edit → Select public-subnet-1a, public-subnet-1b
→ Save
```

**Step 5 — Verify Connectivity**
- Launch an EC2 instance in `public-subnet-1a`.
- Assign a Security Group allowing SSH (port 22) from your IP.
- SSH into the instance and run `curl https://checkip.amazonaws.com` — you should see a public IP returned.

> 💡 **Tip:** A subnet is only "public" when it has a route to an IGW **and** instances have public IPs. Both conditions must be true.

---

## INTERMEDIATE — Connectivity & Security

---

### 6. Bastion Hosts

#### Concepts to Understand

- A **Bastion Host** (Jump Box) is an EC2 instance in a **public subnet** used to SSH into instances in **private subnets**.
- Acts as a single, hardened entry point — minimises the attack surface.
- Private instances have **no public IP** and are not directly reachable from the internet.

#### Architecture

```
Internet → IGW → Public Subnet [Bastion Host] → Private Subnet [App Server]
```

#### Hands-On: Set Up a Bastion Host

**Step 1 — Launch Bastion in Public Subnet**
```
EC2 → Launch Instance
  Name: bastion-host
  AMI: Amazon Linux 2023
  Subnet: public-subnet-1a
  Auto-assign Public IP: Enable
  Security Group: bastion-sg
    Inbound: SSH (22) from YOUR IP only
```

**Step 2 — Launch Private EC2**
```
EC2 → Launch Instance
  Name: private-app-server
  Subnet: private-subnet-1a
  Auto-assign Public IP: Disable
  Security Group: private-sg
    Inbound: SSH (22) from bastion-sg
```

**Step 3 — SSH via Bastion (Agent Forwarding)**
```bash
# On your local machine
ssh-add ~/.ssh/my-key.pem
ssh -A -i ~/.ssh/my-key.pem ec2-user@<bastion-public-ip>


cat > mykey.pem << 'EOF'
key----
-------
EOF
chmod 400 mykey.pem

# From inside bastion
ssh ec2-user@<private-instance-private-ip>
```

> 💡 **Tip:** Use **AWS Systems Manager Session Manager** as a more secure, keyless alternative to Bastion Hosts — no open port 22 required.

---

### 7. NAT Instances

#### Concepts to Understand

- A **NAT Instance** is an EC2 instance that routes outbound traffic from private subnets to the internet.
- **Legacy approach** — replaced by NAT Gateway, but still tested in AWS exams.
- Requires disabling **source/destination check** on the EC2 instance.
- Single point of failure unless you add HA manually.

#### Hands-On: Configure a NAT Instance

**Step 1 — Launch NAT Instance**
```
EC2 → Launch Instance
  AMI: Search "amzn-ami-vpc-nat" in Community AMIs
  Subnet: public-subnet-1a
  Auto-assign Public IP: Enable
  Security Group:
    Inbound: SSH from bastion-sg; All Traffic from private-subnet CIDR (10.0.2.0/24, 10.0.4.0/24)
```

**Step 2 — Disable Source/Destination Check**
```
EC2 → Select NAT Instance → Actions → Networking → Change Source/Destination Check
  → Disable
```

**Step 3 — Update Private Route Table**
```
VPC → Route Tables → Create Route Table
  Name: private-rt
→ Routes → Add Route
  Destination: 0.0.0.0/0
  Target: Instance → <NAT Instance ID>
→ Subnet Associations → Add private subnets
```

**Step 4 — Test from Private Instance**
```bash
# SSH to private instance via bastion, then:
curl https://checkip.amazonaws.com
# Should return the NAT instance's public IP
```

> ⚠️ **Warning:** NAT Instances are NOT recommended for production. They have limited bandwidth tied to instance size and no built-in HA.

---

### 8. NAT Gateways

#### Concepts to Understand

| Feature | NAT Instance | NAT Gateway |
|---|---|---|
| Managed by | You | AWS |
| Availability | Manual HA needed | Automatic within AZ |
| Bandwidth | Limited by EC2 type | Up to 100 Gbps |
| Cost | EC2 hourly + data | Per hour + data |
| Security Groups | Required | Not applicable |

#### Hands-On: Create a NAT Gateway

**Step 1 — Allocate Elastic IP**
```
VPC Console → Elastic IPs → Allocate Elastic IP Address → Allocate
```

**Step 2 — Create NAT Gateway**
```
VPC Console → NAT Gateways → Create NAT Gateway
  Name: my-nat-gateway
  Subnet: public-subnet-1a   ← must be PUBLIC
  Connectivity: Public
  Elastic IP: <allocated EIP>
→ Create NAT Gateway (takes ~2 minutes)
```

**Step 3 — Update Private Route Table**
```
private-rt → Routes → Edit → Add Route
  Destination: 0.0.0.0/0
  Target: NAT Gateway → my-nat-gateway
→ Save
```

**Step 4 — Test**
- SSH into private instance via Bastion → `curl https://checkip.amazonaws.com`

> 💡 **Tip:** For high availability, create **one NAT Gateway per AZ** and update each AZ's private route table to use its local NAT Gateway. This avoids cross-AZ data transfer costs.

---

### 9. Regional NAT Gateway

#### Concepts to Understand

- By default, NAT Gateways are **AZ-scoped** — traffic stays within the AZ.
- A **Regional NAT Gateway** setup means placing NAT Gateways in multiple AZs (one per AZ) so each AZ's private subnets have local NAT egress.
- Reduces **cross-AZ data transfer costs** and improves resilience.

#### Best Practice Architecture

```
AZ-a: private-subnet-1a → private-rt-1a (0.0.0.0/0 → nat-gw-1a in public-subnet-1a)
AZ-b: private-subnet-1b → private-rt-1b (0.0.0.0/0 → nat-gw-1b in public-subnet-1b)
```

#### Hands-On

- Repeat the NAT Gateway creation for `public-subnet-1b`.
- Create a separate private route table `private-rt-1b`.
- Associate it with `private-subnet-1b`.
- Add route `0.0.0.0/0 → nat-gw-1b`.

---

### 10. NACLs & Security Groups

#### Concepts to Understand

| Feature | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow & Deny |
| Rule Order | All rules evaluated | Rules evaluated in order (lowest number first) |
| Default | Deny all inbound | Allow all |

#### Stateful vs Stateless

- **Security Group (Stateful):** If you allow inbound SSH, the return traffic is automatically allowed.
- **NACL (Stateless):** You must explicitly allow **both** inbound AND outbound for each connection.

#### Hands-On: Configure a NACL

**Step 1 — Create a NACL**
```
VPC Console → Network ACLs → Create Network ACL
  Name: public-nacl
  VPC: my-custom-vpc
→ Create
```

**Step 2 — Add Inbound Rules**
```
Inbound Rules → Edit:
  Rule 100: HTTP  (80)  — Allow — 0.0.0.0/0
  Rule 110: HTTPS (443) — Allow — 0.0.0.0/0
  Rule 120: SSH   (22)  — Allow — YOUR-IP/32
  Rule 130: Custom TCP  1024-65535 — Allow — 0.0.0.0/0  ← Ephemeral ports for return traffic
  Rule *  : All Traffic — Deny (default)
```

**Step 3 — Add Outbound Rules**
```
Outbound Rules → Edit:
  Rule 100: HTTP  (80)  — Allow — 0.0.0.0/0
  Rule 110: HTTPS (443) — Allow — 0.0.0.0/0
  Rule 120: Custom TCP  1024-65535 — Allow — 0.0.0.0/0  ← Ephemeral ports
  Rule *  : All Traffic — Deny
```

**Step 4 — Associate with Subnet**
```
public-nacl → Subnet Associations → Edit → Add public-subnet-1a, public-subnet-1b
```

> 💡 **Tip:** Ephemeral ports (1024–65535) must be allowed outbound in NACLs because clients connect on a random port for the response. Forgetting these is a common mistake.

---

## ADVANCED — Networking Deep Dive

---

### 11. VPC Peering

#### Concepts to Understand

- Allows **private routing between two VPCs** (same or different accounts/regions).
- Traffic stays within the AWS backbone — never traverses the internet.
- **Non-transitive:** VPC A ↔ VPC B and VPC B ↔ VPC C does NOT mean A can reach C.
- CIDR ranges must **not overlap**.

#### Hands-On: Peer Two VPCs

**Step 1 — Create a Second VPC**
```
  Name: peer-vpc
  CIDR: 10.1.0.0/16
  Subnet: 10.1.1.0/24 (public)
```

**Step 2 — Create Peering Connection**
```
VPC Console → Peering Connections → Create Peering Connection
  Name: vpc-peer-1
  Requester: my-custom-vpc
  Accepter: peer-vpc (same account/region)
→ Create → Actions → Accept Request
```

**Step 3 — Update Route Tables in BOTH VPCs**
```
my-custom-vpc (public-rt):
  Destination: 10.1.0.0/16
  Target: Peering Connection → vpc-peer-1

peer-vpc (its route table):
  Destination: 10.0.0.0/16
  Target: Peering Connection → vpc-peer-1
```

**Step 4 — Update Security Groups**
- Allow traffic from the peer VPC's CIDR in the relevant security groups.

**Step 5 — Test**
```bash
# From an instance in my-custom-vpc, ping instance in peer-vpc using private IP
ping 10.1.1.x
```

> 💡 **Tip:** For cross-account peering, the Accepter must log into their account and accept the request manually.

---

### 12. VPC Endpoints

#### Concepts to Understand

| Type | Description | Use Case |
|---|---|---|
| **Interface Endpoint** | ENI with private IP using PrivateLink | Most AWS services (SSM, SQS, KMS) |
| **Gateway Endpoint** | Route table target (free) | **S3** and **DynamoDB** only |

- Allows private subnets to access AWS services **without internet, NAT, or IGW**.
- Traffic never leaves the AWS network.

#### Hands-On: Create an S3 Gateway Endpoint

```
VPC Console → Endpoints → Create Endpoint
  Service category: AWS Services
  Service: com.amazonaws.<region>.s3 (Type: Gateway)
  VPC: my-custom-vpc
  Route Tables: select private-rt
→ Create Endpoint
```

**Verify**
```bash
# From a private EC2 instance (no NAT, no IGW):
aws s3 ls  # Should work via endpoint
```

#### Hands-On: Create an SSM Interface Endpoint

```
Endpoint → Create:
  Service: com.amazonaws.<region>.ssm (Type: Interface)
  VPC: my-custom-vpc
  Subnets: private-subnet-1a
  Security Group: allow HTTPS (443) from your VPC CIDR
  Enable Private DNS: ✅
```

> 💡 **Tip:** Gateway Endpoints are **free**. Interface Endpoints cost per hour + data. Use Gateway Endpoints for S3/DynamoDB always.

---

### 13. VPC Flow Logs + Athena

#### Concepts to Understand

- **VPC Flow Logs** capture metadata about IP traffic to/from network interfaces.
- Can be enabled at: VPC, Subnet, or ENI level.
- Logs are sent to **CloudWatch Logs** or **S3**.
- Useful for: security analysis, troubleshooting connectivity, compliance.

#### Flow Log Fields

```
version  account-id  interface-id  srcaddr  dstaddr  srcport  dstport  protocol  packets  bytes  start  end  action  log-status
```

#### Hands-On: Enable Flow Logs to S3

**Step 1 — Create S3 Bucket**
```
S3 → Create Bucket
  Name: my-flow-logs-<accountid>
  Region: same as VPC
  Block all public access: ✅
```

**Step 2 — Enable Flow Logs**
```
VPC → Select my-custom-vpc → Flow Logs tab → Create Flow Log
  Filter: All
  Max aggregation interval: 1 minute
  Destination: Send to S3 bucket → arn:aws:s3:::my-flow-logs-<accountid>
  Format: AWS default
→ Create
```

**Step 3 — Generate Traffic & Query with Athena**

```sql
-- In Athena, create a table pointing to your flow log S3 path:
CREATE EXTERNAL TABLE IF NOT EXISTS vpc_flow_logs (
  version INT, account STRING, interfaceid STRING,
  sourceaddress STRING, destinationaddress STRING,
  sourceport INT, destinationport INT, protocol INT,
  numpackets BIGINT, numbytes BIGINT,
  starttime INT, endtime INT,
  action STRING, logstatus STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ' '
LOCATION 's3://my-flow-logs-<accountid>/AWSLogs/<accountid>/vpcflowlogs/<region>/';

-- Query: Find all REJECTED traffic
SELECT sourceaddress, destinationaddress, destinationport, action
FROM vpc_flow_logs
WHERE action = 'REJECT'
LIMIT 50;
```

> 💡 **Tip:** Use Athena Partition Projection to query large flow log datasets efficiently without manually defining partitions.

---

### 14. Site-to-Site VPN, Virtual Private Gateway & Customer Gateway

#### Concepts to Understand

| Component | Description |
|---|---|
| **Virtual Private Gateway (VGW)** | AWS-side VPN endpoint, attached to your VPC |
| **Customer Gateway (CGW)** | Represents your on-premises VPN device |
| **Site-to-Site VPN** | Encrypted IPsec tunnel between VGW and CGW over the internet |
| **Two Tunnels** | AWS always creates 2 tunnels for redundancy |

#### Architecture

```
On-Premises Network ←→ [Customer Gateway Device] ←IPsec VPN→ [Virtual Private Gateway] ←→ AWS VPC
```

#### Hands-On: Configure Site-to-Site VPN

**Step 1 — Create a Customer Gateway**
```
VPC → Customer Gateways → Create
  Name: my-cgw
  Routing: Static
  IP Address: <your on-premises public IP or simulated IP>
```

**Step 2 — Create Virtual Private Gateway**
```
VPC → Virtual Private Gateways → Create
  Name: my-vgw
→ Actions → Attach to VPC → my-custom-vpc
```

**Step 3 — Create VPN Connection**
```
VPC → Site-to-Site VPN Connections → Create
  VGW: my-vgw
  CGW: my-cgw
  Routing: Static
  Static Routes: <on-premises CIDR, e.g., 192.168.1.0/24>
→ Download Configuration → select your device type
```

**Step 4 — Enable Route Propagation**
```
VPC → Route Tables → Select private-rt → Route Propagation tab
  → Edit → Enable for my-vgw
```

> 💡 **Tip:** For testing without real on-premises hardware, you can simulate using a **Strongswan** EC2 instance acting as the on-prem VPN device.

---

### 15. Direct Connect & Direct Connect Gateway

#### Concepts to Understand

- **AWS Direct Connect:** A dedicated **physical network connection** between your data centre and AWS (1 Gbps, 10 Gbps, 100 Gbps).
- **NOT encrypted by default** — use with VPN for encryption.
- **Lower latency, consistent bandwidth, no internet variability.**
- **Direct Connect Gateway:** Enables a single DX connection to access VPCs in **multiple regions**.

#### Connection Types

| Type | Speed | Setup Time |
|---|---|---|
| Dedicated Connection | 1/10/100 Gbps | Weeks (physical cabling) |
| Hosted Connection | 50 Mbps – 10 Gbps | Faster (via DX partner) |

#### Console Walkthrough

```
Direct Connect Console → Connections → Create Connection
  Name: my-dx-connection
  Location: <DX location nearest to you>
  Port Speed: 1Gbps
→ You will receive a Letter of Authorization (LOA)
→ Provide LOA to colocation provider for physical cross-connect
→ Once connected: Create Virtual Interface (VIF)
  → Private VIF for VPC access
  → Public VIF for AWS public services
```

> 💡 **Tip:** For the exam and real-world: use **Direct Connect + VPN** as a backup. DX is primary (fast), VPN is failover (internet-based).

---

### 16. Direct Connect + Site-to-Site VPN

#### Architecture

```
On-Premises ──DX (primary, fast)──► AWS VPC
On-Premises ──VPN (backup, encrypted)──► AWS VPC (via VGW)
```

- This combination gives you **high-throughput primary** and **encrypted failover**.
- Configure BGP routing to prefer DX path, failover to VPN automatically.
- When DX goes down, traffic reroutes over VPN within seconds (with proper BGP config).

> 💡 **Tip:** You can also run **VPN over Direct Connect** (the IPsec tunnel runs inside the DX connection) for an encrypted, private, high-bandwidth link.

---

### 17. Transit Gateway

#### Concepts to Understand

- A **Transit Gateway (TGW)** is a regional network transit hub.
- Replaces complex, non-transitive VPC peering meshes with a **hub-and-spoke model**.
- Supports: VPCs, VPNs, Direct Connect Gateways, other TGWs (peering).
- **Transitive routing:** Traffic can flow between all attachments.

#### VPC Peering vs Transit Gateway

```
VPC Peering (non-transitive, complex):       Transit Gateway (transitive, simple):
A ↔ B                                         A ─┐
A ↔ C                                         B ─┤─ TGW ─ all can talk
B ↔ C                                         C ─┘
(needs 3 peering connections)                 (1 TGW with 3 attachments)
```

#### Hands-On: Create a Transit Gateway

**Step 1 — Create TGW**
```
VPC → Transit Gateways → Create Transit Gateway
  Name: my-tgw
  ASN: 64512 (default)
  DNS Support: ✅
  VPN ECMP: ✅
→ Create
```

**Step 2 — Attach VPCs**
```
Transit Gateway Attachments → Create Attachment
  TGW: my-tgw
  Attachment Type: VPC
  VPC: my-custom-vpc
  Subnets: private-subnet-1a, private-subnet-1b
→ Repeat for peer-vpc
```

**Step 3 — Update Route Tables**
```
In each VPC's route table:
  Destination: <other VPC CIDR>
  Target: Transit Gateway → my-tgw
```

> 💡 **Tip:** TGW supports **multicast** and **inter-region peering** — very powerful for large enterprise architectures.

---

### 18. VPC Traffic Mirroring

#### Concepts to Understand

- Copies **network traffic from ENIs** to a target (another ENI or NLB) for inspection.
- Use cases: intrusion detection, network forensics, packet capture.
- Does NOT impact the performance of the source instance.

#### Components

| Component | Description |
|---|---|
| **Mirror Source** | ENI of the EC2 instance to monitor |
| **Mirror Target** | ENI or NLB where traffic is sent |
| **Mirror Filter** | Rules to select which traffic to mirror |
| **Mirror Session** | Connects source → filter → target |

#### Hands-On: Set Up Traffic Mirroring

```
VPC → Traffic Mirroring → Mirror Filters → Create
  Name: capture-http
  Add Inbound Rule: Accept, All protocols, All sources

VPC → Traffic Mirroring → Mirror Targets → Create
  Name: ids-target
  Target type: Network Interface
  Network Interface: <ENI of your IDS/monitoring instance>

VPC → Traffic Mirroring → Mirror Sessions → Create
  Session number: 1
  Mirror source: <ENI of monitored instance>
  Mirror target: ids-target
  Filter: capture-http
```

> 💡 **Tip:** On the monitoring instance, use `tcpdump -i eth0 -w capture.pcap` to capture mirrored traffic.

---

## ADVANCED — IPv6, Egress & Firewall

---

### 19. IPv6 for VPC

#### Concepts to Understand

- IPv6 addresses are **globally unique** — there are no private IPv6 ranges.
- AWS assigns a `/56` IPv6 CIDR to the VPC and `/64` to each subnet.
- IPv6 is **dual-stack** (both IPv4 and IPv6 simultaneously).
- All IPv6 addresses are public — use **Egress-Only IGW** to control outbound access from private resources.

#### Hands-On: Enable IPv6 on VPC

**Step 1 — Add IPv6 CIDR to VPC**
```
VPC → select my-custom-vpc → Actions → Edit CIDRs
  IPv6 CIDRs → Add IPv6 CIDR → Amazon-provided → Add
```

**Step 2 — Add IPv6 CIDR to Subnets**
```
Each subnet → Actions → Edit IPv6 CIDRs
  → Add subnet CIDR (e.g., ::/64 — AWS auto-assigns)
```

**Step 3 — Update Route Tables**
```
public-rt → Add Route:
  Destination: ::/0
  Target: my-igw
```

**Step 4 — Enable Auto-assign IPv6 on Public Subnets**
```
Subnet → Actions → Edit Subnet Settings → Auto-assign IPv6 ✅
```

---

### 20. Egress-Only Internet Gateway

#### Concepts to Understand

- An **Egress-Only Internet Gateway** is for **IPv6 traffic only**.
- It allows outbound IPv6 traffic from private instances **but blocks inbound** connections.
- The IPv4 equivalent is a NAT Gateway.

```
IPv4 Private → NAT Gateway → Internet (outbound only)
IPv6 Private → Egress-Only IGW → Internet (outbound only)
```

#### Hands-On

**Step 1 — Create Egress-Only IGW**
```
VPC → Egress-Only Internet Gateways → Create
  VPC: my-custom-vpc
→ Create
```

**Step 2 — Update Private Route Table**
```
private-rt → Add Route:
  Destination: ::/0
  Target: Egress-Only IGW
```

**Step 3 — Test**
```bash
# From private instance with IPv6:
curl -6 https://ipv6.google.com  # Should succeed (outbound)
```

---

### 21. AWS Network Firewall

#### Concepts to Understand

- A **managed, stateful Layer 3–7 network firewall** for your VPC.
- Sits between your IGW and your subnets.
- Supports: stateful/stateless rules, domain filtering, IDS/IPS, TLS inspection.
- Deployed in a dedicated **firewall subnet** per AZ.

#### Architecture

```
Internet → IGW → [Firewall Subnet: Network Firewall] → Public/Private Subnets
```

#### Hands-On: Deploy Network Firewall

**Step 1 — Create Firewall Subnet**
```
Create new subnet:
  Name: firewall-subnet-1a
  CIDR: 10.0.10.0/28
  AZ: <region>a
```

**Step 2 — Create Firewall Policy**
```
Network Firewall → Firewall Policies → Create
  Name: my-fw-policy
  Stateless default: Forward to stateful
  Stateful rules:
    Add Rule Group → Managed: AWSManagedRulesThreatSignaturesDoS
```

**Step 3 — Create Firewall**
```
Network Firewall → Firewalls → Create
  Name: my-network-firewall
  VPC: my-custom-vpc
  Subnet: firewall-subnet-1a (one per AZ)
  Policy: my-fw-policy
```

**Step 4 — Redirect Traffic Through Firewall**
```
Update route tables:
  IGW Ingress Route Table:
    Destination: 10.0.1.0/24 (public subnet)
    Target: VPC Endpoint (firewall)
  
  Public Subnet Route Table:
    Destination: 0.0.0.0/0
    Target: VPC Endpoint (firewall)
  
  Firewall Subnet Route Table:
    Destination: 0.0.0.0/0
    Target: IGW
```

> 💡 **Tip:** Network Firewall uses **Gateway Load Balancer endpoints** behind the scenes to distribute traffic. It supports horizontal scaling automatically.

---

## 22. Networking Costs in AWS

#### Key Cost Drivers

| Traffic Type | Cost |
|---|---|
| Data IN to AWS | Free |
| Data OUT to internet | ~$0.09/GB (varies by region) |
| Data between AZs (same region) | ~$0.01/GB each way |
| Data between regions | ~$0.02–$0.08/GB |
| VPC Peering (same region) | Same as inter-AZ cost |
| NAT Gateway processing | ~$0.045/GB |
| PrivateLink/Interface Endpoint | Hourly + ~$0.01/GB |
| Gateway Endpoint (S3/DynamoDB) | **Free** |
| Direct Connect | Port hour + data |

#### Cost Optimisation Tips

1. **Place resources in the same AZ** when possible to avoid inter-AZ costs.
2. Use **Gateway Endpoints** for S3/DynamoDB instead of NAT Gateway (saves NAT processing cost).
3. Use **one NAT Gateway per AZ** to avoid cross-AZ data charges.
4. Use **VPC Peering instead of Transit Gateway** when you have <5 VPCs (TGW charges per attachment/hour).
5. Enable **S3 Transfer Acceleration** only when genuinely needed (extra cost).
6. Use **CloudFront** to reduce data-out costs (CloudFront-to-origin is cheaper than direct internet egress).

---

## Section Cleanup Checklist

> ⚠️ **Always clean up resources to avoid unexpected AWS charges!**

Run through this list in order (dependencies must be removed first):

```
☐ Terminate all EC2 instances (Bastion, NAT Instance, Private, Monitoring)
☐ Delete NAT Gateways (wait for "deleted" state before releasing EIPs)
☐ Release Elastic IP Addresses
☐ Detach & delete Internet Gateway
☐ Delete Egress-Only Internet Gateway
☐ Delete VPC Endpoints
☐ Delete Network Firewall (then policy, then rule groups)
☐ Delete Transit Gateway Attachments → then Transit Gateway
☐ Delete VPC Peering Connections
☐ Delete Site-to-Site VPN Connection
☐ Delete Virtual Private Gateway (detach from VPC first)
☐ Delete Customer Gateway
☐ Delete Subnets
☐ Delete Route Tables (all custom ones)
☐ Delete Security Groups (all custom ones)
☐ Delete Network ACLs (custom ones)
☐ Delete VPC Flow Logs → delete S3 bucket (if created)
☐ Delete VPCs (my-custom-vpc, peer-vpc)
☐ Verify Elastic IPs are all released
```

---

## VPC Section Summary Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        AWS VPC QUICK REFERENCE                          │
├────────────────────┬────────────────────────────────────────────────────┤
│ Component          │ Key Points                                          │
├────────────────────┼────────────────────────────────────────────────────┤
│ VPC                │ Regional; CIDR can't change; DNS hostname needed    │
│ Subnet             │ AZ-scoped; 5 IPs reserved by AWS                   │
│ IGW                │ One per VPC; enables internet access                │
│ Route Table        │ Most specific route wins; propagate for VPN        │
│ Security Group     │ Stateful; instance level; Allow only               │
│ NACL               │ Stateless; subnet level; Allow + Deny              │
│ NAT Gateway        │ Public subnet; 1 per AZ for HA; managed by AWS     │
│ Bastion Host       │ Jump box in public subnet; use SSM as alternative   │
│ VPC Peering        │ Non-transitive; no overlapping CIDRs               │
│ Transit Gateway    │ Transitive hub-spoke; replaces peering mesh        │
│ VPC Endpoint       │ Gateway (S3/DDB, free) | Interface (PrivateLink)   │
│ Flow Logs          │ Metadata only (no payload); CW or S3               │
│ Site-to-Site VPN   │ IPsec; 2 tunnels; over internet                    │
│ Direct Connect     │ Physical; fast; not encrypted by default            │
│ Egress-Only IGW    │ IPv6 outbound only (like NAT for IPv6)             │
│ Network Firewall   │ L3-L7; stateful; domain filtering; IPS/IDS         │
│ Traffic Mirroring  │ Copy ENI traffic; forensics/IDS                    │
└────────────────────┴────────────────────────────────────────────────────┘
```

---

*Guide version 1.0 — Covers AWS VPC topics from foundational to advanced. Always refer to the [AWS Official Documentation](https://docs.aws.amazon.com/vpc/) for the latest updates.*
