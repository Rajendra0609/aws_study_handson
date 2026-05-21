# 🪣 AWS S3 — Complete Hands-On Learning Guide
> A structured, practical guide to mastering Amazon S3 from beginner to advanced level.

---

## 📚 Table of Contents

| Level | Topics |
|-------|--------|
| 🟢 **Beginner** | S3 Overview, Buckets & Objects, Security & Bucket Policies, Static Website Hosting |
| 🟡 **Intermediate** | Versioning, Replication, Storage Classes, Lifecycle Rules, Performance, Event Notifications |
| 🔴 **Advanced** | Encryption, CORS, MFA Delete, Access Logs, Pre-signed URLs, Object Lock, Access Points, Object Lambda |

---

## 🟢 BEGINNER LEVEL

---

### 1. S3 Overview

**What it is:** Amazon S3 (Simple Storage Service) is an object storage service offering industry-leading scalability, availability, and durability.

**Key Concepts:**
- **Buckets** — Containers for storing objects (files). Bucket names must be globally unique.
- **Objects** — Files stored in buckets. Each object has a key (full path), value (data), metadata, and version ID.
- **Region** — Buckets are created in a specific AWS region.
- Max object size: **5 TB**. For uploads > 5 GB, use **multi-part upload**.
- S3 is a **key-value store** — the "path" is just part of the key name (no real folder structure).

**Important Tips:**
- Think of the bucket as the top-level namespace and objects as everything inside.
- S3 is not a file system — it's flat, but the UI shows it as folders using `/` in key names.
- S3 URLs follow the format: `https://<bucket-name>.s3.<region>.amazonaws.com/<object-key>`

---

### 2. S3 Hands On

**Goal:** Create your first S3 bucket and upload an object.

**Steps:**
1. Go to **AWS Console → S3 → Create Bucket**.
2. Enter a globally unique bucket name (e.g., `my-learning-bucket-2024`).
3. Choose a region closest to you.
4. Leave **Block all public access** enabled (default) — we'll change this later.
5. Click **Create Bucket**.
6. Open the bucket → click **Upload** → drag & drop any file (e.g., `hello.txt`).
7. Click **Upload** to confirm.
8. Click on the uploaded object → view its **Properties**, **URL**, and **Metadata**.

**Exercise:**
- Try accessing the object URL directly in a browser — observe the `AccessDenied` error (it's private by default).
- Upload a file with a "folder-like" key: `images/photo.jpg` and observe how S3 simulates folders.

---

### 3. S3 Security: Bucket Policy

**What it is:** JSON-based resource policies attached to S3 buckets to control access.

**Key Concepts:**
- **Bucket Policies** — Grant/deny access at bucket or object level to AWS accounts, IAM users, or the public.
- **ACLs (Access Control Lists)** — Legacy method; prefer bucket policies.
- **IAM Policies** — Attached to users/roles; combined with bucket policies.
- **Block Public Access** — Account/bucket-level setting that overrides all public-granting policies.

**Policy Structure:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket-name/*"
    }
  ]
}
```

**Access Decision Logic:** IAM Policy + Bucket Policy must both allow (no explicit deny anywhere).

---

### 4. S3 Security: Bucket Policy Hands On

**Goal:** Make your bucket publicly readable using a bucket policy.

**Steps:**
1. Open your bucket → **Permissions** tab.
2. Under **Block public access** → click **Edit** → uncheck all boxes → **Save** (type `confirm`).
3. Scroll to **Bucket Policy** → click **Edit**.
4. Paste the policy above (replace `my-bucket-name` with your bucket name).
5. Click **Save changes**.
6. Now try accessing your object URL in the browser — it should be publicly accessible!

**Exercise:**
- Add a **Condition** to restrict access by IP:
  ```json
  "Condition": { "IpAddress": { "aws:SourceIp": "YOUR_IP/32" } }
  ```
- Try the **AWS Policy Generator** (link in the console) to build policies visually.
- Test **Deny** statements — add a Deny for a specific object key and verify it blocks access.

---

### 5. S3 Website Overview

**What it is:** S3 can host **static websites** (HTML, CSS, JS) directly — no servers needed.

**Key Concepts:**
- The website endpoint format: `http://<bucket-name>.s3-website-<region>.amazonaws.com`
- You must set an **Index Document** (e.g., `index.html`) and optionally an **Error Document** (e.g., `error.html`).
- The bucket **must be publicly readable** for website hosting to work.
- S3 websites support **HTTP only** — for HTTPS, use CloudFront in front of S3.
- **CORS** must be configured if your website calls APIs from another domain.

---

### 6. S3 Website Hands On

**Goal:** Host a static website on S3.

**Steps:**
1. Create `index.html` locally:
   ```html
   <!DOCTYPE html>
   <html><body><h1>Hello from S3 Static Website!</h1></body></html>
   ```
2. Upload `index.html` to your bucket.
3. Go to bucket → **Properties** → scroll to **Static website hosting** → **Edit**.
4. Enable it → set **Index document** to `index.html` → **Save**.
5. Apply the public read bucket policy (from Step 4 above).
6. Copy the **Website endpoint URL** from Properties and open it in your browser.

**Exercise:**
- Create an `error.html` and set it as the Error Document — then try accessing a non-existent page.
- Add a CSS file and link it in your HTML — verify it loads correctly.
- Try hosting a single-page React/Vue build (just the `dist/` folder contents).

---

## 🟡 INTERMEDIATE LEVEL

---

### 7. S3 Versioning

**What it is:** S3 Versioning keeps multiple versions of an object, protecting against accidental deletions and overwrites.

**Key Concepts:**
- Enabled at the **bucket level**.
- Each version gets a unique **Version ID**.
- Deleting an object adds a **Delete Marker** (doesn't permanently delete unless you delete the version itself).
- Once enabled, versioning **cannot be fully disabled** — only suspended.
- **MFA Delete** adds extra protection (covered later).

**States:** Unversioned → Versioning-enabled → Versioning-suspended

---

### 8. S3 Versioning Hands On

**Goal:** Enable versioning and observe version history.

**Steps:**
1. Open your bucket → **Properties** → **Bucket Versioning** → **Edit** → **Enable**.
2. Re-upload the same `index.html` with modified content.
3. In the bucket, toggle **Show versions** — you'll see both versions with different Version IDs.
4. Click on older versions to access/download them.
5. Delete the object — notice a **Delete Marker** is created, not actual deletion.
6. Delete the Delete Marker to "restore" the object.

**Exercise:**
- Permanently delete a specific version and verify it's gone.
- Calculate cost implications — each version is stored and billed separately.
- Try using the AWS CLI: `aws s3api list-object-versions --bucket your-bucket-name`

---

### 9. S3 Replication

**What it is:** Automatically replicate objects from a source bucket to a destination bucket.

**Key Concepts:**
- **CRR (Cross-Region Replication)** — Source and destination in different AWS regions. Use cases: compliance, lower latency, disaster recovery.
- **SRR (Same-Region Replication)** — Same region. Use cases: log aggregation, dev/prod sync.
- Requires **Versioning enabled** on both source and destination buckets.
- Replication is **asynchronous** (near real-time).
- Only **new objects** are replicated after enabling — existing objects require **S3 Batch Replication**.
- Delete markers are **not replicated** by default (can be enabled).

---

### 10. S3 Replication Notes

**Important Nuances:**
- There is **no chaining** — if Bucket A replicates to B, and B replicates to C, objects in A are NOT replicated to C automatically.
- You can replicate objects **across AWS accounts** with the right IAM role setup.
- Replicated objects retain the **same storage class** unless you specify otherwise.
- **Replication Time Control (RTC)** provides an SLA of 99.99% of objects replicated within 15 minutes (additional cost).
- Objects encrypted with **SSE-KMS** can be replicated but require additional KMS configuration.

---

### 11. S3 Replication Hands On

**Goal:** Set up Cross-Region Replication between two buckets.

**Steps:**
1. Create a **source bucket** in `us-east-1` with versioning enabled.
2. Create a **destination bucket** in `ap-south-1` (Mumbai) with versioning enabled.
3. In the source bucket → **Management** → **Replication rules** → **Create replication rule**.
4. Name it `replicate-all`, set status to **Enabled**.
5. Source: **Apply to all objects**.
6. Destination: choose your destination bucket.
7. IAM Role: Let AWS **create a new role** automatically.
8. Click **Save** — AWS asks if you want to replicate existing objects → choose **No** for now.
9. Upload a new object to the source — wait ~1 minute → check the destination bucket.

**Exercise:**
- Enable **Delete marker replication** and test what happens when you delete an object.
- Use **S3 Batch Replication** to replicate existing objects.

---

### 12. S3 Storage Classes Overview

**What it is:** S3 offers multiple storage classes optimized for different access patterns and costs.

| Storage Class | Use Case | Retrieval Time | Min Duration |
|---|---|---|---|
| **S3 Standard** | Frequently accessed data | Milliseconds | None |
| **S3 Standard-IA** | Infrequent access, rapid retrieval | Milliseconds | 30 days |
| **S3 One Zone-IA** | Infrequent, non-critical, single AZ | Milliseconds | 30 days |
| **S3 Intelligent-Tiering** | Unknown/changing access patterns | Milliseconds | None |
| **S3 Glacier Instant Retrieval** | Archives, quarterly access | Milliseconds | 90 days |
| **S3 Glacier Flexible Retrieval** | Archives, minutes to hours | 1 min–12 hrs | 90 days |
| **S3 Glacier Deep Archive** | Long-term archive, rarely accessed | 12–48 hours | 180 days |

**Key Rule:** You can transition objects **down** the tier chain but not back up automatically.

---

### 13. S3 Storage Classes Hands On

**Goal:** Manually set and observe storage classes.

**Steps:**
1. Upload an object → under **Properties** → choose a **Storage Class** (e.g., Standard-IA).
2. Go to an existing object → **Properties** → click **Storage Class** → change it.
3. Observe how the class is reflected in the bucket view.
4. Check **pricing differences** in the [S3 pricing page](https://aws.amazon.com/s3/pricing/).

**Exercise:**
- Upload the same 1 MB file to Standard and Standard-IA — compare costs over 30 days.
- Use the **AWS Cost Calculator** to estimate monthly S3 costs for your use case.

---

### 14. S3 Express One Zone

**What it is:** A high-performance, single-AZ storage class for latency-sensitive applications.

**Key Concepts:**
- Offers **single-digit millisecond** latency (10x faster than S3 Standard).
- Uses **Directory Buckets** (different from general-purpose buckets).
- Costs less per request but does not have multi-AZ redundancy.
- Ideal for: ML training jobs, real-time analytics, HPC workloads.
- Naming convention for directory buckets: `bucket-name--az-id--x-s3`

**Tip:** Use Express One Zone only when latency is critical and data loss of a single AZ is acceptable.

---

### 15. S3 Lifecycle Rules (with S3 Analytics)

**What it is:** Automate transitioning objects between storage classes or expiring them.

**Key Concepts:**
- **Transition Actions** — Move objects to a cheaper class after N days (e.g., move to Glacier after 90 days).
- **Expiration Actions** — Delete objects after N days (e.g., delete old log files after 365 days).
- Rules can be scoped by **prefix** or **object tags**.
- **S3 Analytics** — Analyzes access patterns and recommends when to transition to Standard-IA. Report takes 24–48 hours to generate initially.

**Common Lifecycle Pattern:**
```
Day 0:   Upload → S3 Standard
Day 30:  Transition → Standard-IA
Day 90:  Transition → Glacier Flexible Retrieval
Day 365: Expire (delete)
```

---

### 16. S3 Lifecycle Rules Hands On

**Goal:** Create a lifecycle rule to automatically transition and expire objects.

**Steps:**
1. Open your bucket → **Management** → **Lifecycle rules** → **Create lifecycle rule**.
2. Name: `archive-old-files`, apply to **all objects**.
3. **Lifecycle rule actions** → check:
   - Transition current versions between storage classes
   - Expire current versions of objects
4. Add transition: after **30 days** → Standard-IA.
5. Add transition: after **90 days** → Glacier Flexible Retrieval.
6. Add expiration: after **365 days**.
7. Review and **Create rule**.

**Exercise:**
- Create a rule scoped to a prefix like `logs/` only.
- Enable **S3 Analytics** on your bucket and check back in 48 hours for recommendations.
- Create a rule to **delete incomplete multipart uploads** after 7 days (best practice!).

---

### 17. S3 Requester Pays

**What it is:** Normally the bucket owner pays for storage and data transfer. With Requester Pays, the **requester** (downloader) pays for data transfer and request costs.

**Key Concepts:**
- Useful when sharing large datasets publicly (e.g., open datasets, research data).
- The requester **must be authenticated** — anonymous access is not allowed.
- Requester must include `x-amz-request-payer: requester` in their request header.
- Common in public AWS datasets (e.g., satellite imagery, genomics data).

**Tip:** Enable via bucket → **Properties** → **Requester pays**.

---

### 18. S3 Event Notifications

**What it is:** Trigger automated actions when events occur in your S3 bucket.

**Key Concepts:**
- Events: `s3:ObjectCreated:*`, `s3:ObjectRemoved:*`, `s3:ObjectRestore:*`, `s3:Replication:*`, etc.
- Destinations:
  - **SNS** (Simple Notification Service) — fan-out notifications
  - **SQS** (Simple Queue Service) — queue for processing
  - **Lambda** — run code on object upload/delete
  - **EventBridge** — advanced routing, filtering, and replay (recommended)
- Notifications are delivered in **seconds** (occasionally a minute or more).

**Architecture Pattern:**
```
S3 Upload → Event Notification → Lambda → Process image/file → Store result
```

---

### 19. S3 Event Notifications Hands On

**Goal:** Trigger a Lambda function when an object is uploaded to S3.

**Steps:**
1. Create a Lambda function (e.g., `s3-event-handler`) with the **S3 trigger** blueprint.
2. In Lambda → **Add trigger** → S3 → choose your bucket → event type `PUT`.
3. Or do it from S3: bucket → **Properties** → **Event notifications** → **Create notification**.
4. Name it, select event type `s3:ObjectCreated:*`, destination: your Lambda.
5. Upload a file to S3 → go to **Lambda → Monitor → CloudWatch Logs** to see the trigger.

**Exercise:**
- Log the object key and size in your Lambda function.
- Use **EventBridge** instead — enable it in bucket properties and create an EventBridge rule.
- Build a pipeline: upload a CSV → Lambda parses it → writes to DynamoDB.

---

### 20. S3 Performance

**What it is:** S3 is built for high throughput, but understanding its performance model helps you optimize.

**Key Concepts:**
- **Baseline performance:** 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD requests per second **per prefix**.
- **Multi-part Upload:** Recommended for files > 100 MB, required for > 5 GB. Uploads parts in parallel.
- **S3 Transfer Acceleration:** Routes uploads through AWS Edge Locations (CloudFront network) for faster global uploads. Uses a special endpoint: `bucket.s3-accelerate.amazonaws.com`.
- **Byte-Range Fetches:** Parallelize downloads by requesting specific byte ranges. Also useful for partial reads (e.g., reading just the header of a large file).
- **Prefix Strategy:** Spread requests across multiple prefixes to maximize throughput (e.g., `2024-01/`, `2024-02/` instead of all under one prefix).

**Tip:** Never use sequential prefixes (e.g., `0001`, `0002`) — use hashes or random prefixes for high-throughput workloads.

---

### 21. S3 Batch Operations

**What it is:** Run bulk operations on billions of existing S3 objects with a single request.

**Key Concepts:**
- Supported operations: Copy, Invoke Lambda, Restore from Glacier, Replace ACL, Replace tags, Delete, Set Object Lock.
- You provide an **object inventory** (S3 Inventory report or CSV manifest).
- S3 manages retries, tracks progress, sends completion reports.
- Useful for: backups, tagging, encryption migration, cross-account copies.

**Steps to use:**
1. Generate an S3 Inventory report for your bucket (or create a CSV manifest).
2. Go to **S3 → Batch Operations → Create job**.
3. Choose your manifest, operation, IAM role, and report destination.
4. Review and **Run job**.

**Exercise:**
- Use Batch Operations to copy all objects from one bucket to another with a new storage class.
- Run a Lambda function across all objects to add a custom metadata tag.

---

### 22. S3 Storage Lens

**What it is:** Organization-wide analytics dashboard for S3 usage and activity metrics.

**Key Concepts:**
- Provides metrics across **all buckets, regions, and accounts** in your AWS Organization.
- **Free Metrics** — 28 usage metrics with 14-day retention.
- **Advanced Metrics** — Activity metrics, CloudWatch publishing, prefix-level aggregation (paid).
- Metrics include: total storage, object count, requests, replication, encryption, versioning coverage.
- Export data to an S3 bucket (CSV or Parquet) for custom analysis.

**Steps:**
1. Go to **S3 → Storage Lens → Dashboards**.
2. View the default `default-account-dashboard` created automatically.
3. Create a custom dashboard scoped to specific buckets or regions.
4. Analyze recommendations (e.g., "X GB not accessed in 90 days").

---

## 🔴 ADVANCED LEVEL

---

### 23. S3 Encryption

**What it is:** S3 supports multiple encryption mechanisms for data at rest.

**Encryption Types:**

| Method | Key Management | Who manages it? |
|---|---|---|
| **SSE-S3** | AES-256, S3-managed keys | AWS |
| **SSE-KMS** | AWS KMS keys | You (via KMS) |
| **DSSE-KMS** | Dual-layer KMS | You (via KMS) |
| **SSE-C** | Customer-provided keys | You entirely |
| **Client-Side** | Encrypted before upload | You entirely |

- **SSE-S3** is the **default** encryption since Jan 2023.
- **SSE-KMS** gives you audit trails via CloudTrail and key rotation control.
- **SSE-C** — you provide the key with every request; AWS does NOT store the key.

---

### 24. About DSSE-KMS

**What it is:** Dual-layer Server-Side Encryption with AWS KMS keys.

**Key Concepts:**
- Applies **two independent layers** of encryption to each object.
- Compliant with **CNSSI SP 800-57** and designed for workloads requiring dual-layer encryption (e.g., US government/defense).
- Both layers use KMS keys — you can use the same or different keys per layer.
- Slight performance overhead vs. SSE-KMS due to double encryption.
- Only supported on **general-purpose buckets** (not Express One Zone).

**Tip:** Use DSSE-KMS only when required by compliance mandates. For most use cases, SSE-KMS provides sufficient security with auditability.

---

### 25. S3 Encryption Hands On

**Goal:** Upload an object with SSE-KMS encryption.

**Steps:**
1. Go to **KMS → Create a key** (symmetric, type: KMS). Name it `s3-demo-key`.
2. In your bucket → upload a file → expand **Properties** → under Server-side encryption:
   - Choose **AWS Key Management Service key (SSE-KMS)**
   - Select your `s3-demo-key`
3. Click **Upload**.
4. Open the object → **Properties** → see encryption details.

**Exercise:**
- Try `SSE-C`: use AWS CLI:
  ```bash
  aws s3 cp file.txt s3://my-bucket/ \
    --sse-c AES256 \
    --sse-c-key fileb://encryption-key.bin
  ```
- Enable **automatic key rotation** on your KMS key (annually by default).
- Check **CloudTrail** to see KMS `Decrypt` events when S3 serves encrypted objects.

---

### 26. S3 Default Encryption

**What it is:** Set the default encryption behavior for all objects uploaded to a bucket.

**Key Concepts:**
- Enforced at the bucket level — applies to all new objects without requiring per-upload specification.
- Go to bucket → **Properties** → **Default encryption** → Edit.
- Options: **SSE-S3** (default) or **SSE-KMS** (choose your KMS key).
- A **bucket policy** can enforce encryption by denying uploads without the correct header:
  ```json
  {
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringNotEquals": {
        "s3:x-amz-server-side-encryption": "aws:kms"
      }
    }
  }
  ```

**Tip:** Use default encryption + a bucket policy deny to enforce encryption compliance automatically.

---

### 27. S3 CORS

**What it is:** Cross-Origin Resource Sharing — controls how browsers access S3 resources from different domains.

**Key Concepts:**
- Required when your **web app on one domain** (e.g., `myapp.com`) fetches resources from an **S3 bucket on another domain**.
- CORS is enforced by the **browser** — server-to-server calls don't need CORS.
- Configured using a JSON CORS configuration on the bucket.

**CORS Configuration Example:**
```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET", "PUT", "POST"],
    "AllowedOrigins": ["https://www.myapp.com"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

---

### 28. S3 CORS Hands On

**Goal:** Enable cross-origin access between two S3-hosted websites.

**Steps:**
1. Create two buckets: `bucket-main` and `bucket-assets` — both with static website hosting.
2. In `bucket-main/index.html`, fetch a resource from `bucket-assets`:
   ```html
   <script>
     fetch('http://bucket-assets.s3-website-us-east-1.amazonaws.com/data.json')
       .then(r => r.json()).then(console.log);
   </script>
   ```
3. Open browser console → see CORS error.
4. In `bucket-assets` → **Permissions** → **CORS** → add the CORS JSON above (set origin to `bucket-main`'s URL).
5. Refresh — the error should be gone.

**Exercise:**
- Use browser DevTools → Network tab → look for the `OPTIONS` preflight request.
- Try `AllowedOrigins: ["*"]` — understand when this is acceptable vs. a security risk.

---

### 29. S3 MFA Delete

**What it is:** Adds a second layer of protection requiring MFA to permanently delete objects or suspend versioning.

**Key Concepts:**
- Requires **versioning to be enabled**.
- Only the **root account** can enable/disable MFA Delete (not IAM users, even admins).
- MFA is required to: permanently delete a versioned object, suspend versioning.
- Does NOT protect against regular delete (adding a delete marker).
- Protects against accidental or malicious permanent deletion.

---

### 30. S3 MFA Delete Hands On

**Goal:** Enable MFA Delete using AWS CLI (console doesn't support it).

**Steps:**
1. Generate MFA token from your MFA device/app.
2. Enable MFA Delete using AWS CLI (must be root credentials):
   ```bash
   aws s3api put-bucket-versioning \
     --bucket your-bucket-name \
     --versioning-configuration Status=Enabled,MFADelete=Enabled \
     --mfa "arn:aws:iam::ACCOUNT_ID:mfa/root-account-mfa-device TOKEN_CODE"
   ```
3. Try permanently deleting a versioned object via console — it will fail.
4. Verify you need MFA to permanently delete via CLI.

**Tip:** Enable MFA Delete on production buckets containing critical data to prevent ransomware or accidental loss.

---

### 31. S3 Access Logs

**What it is:** Detailed logs of all requests made to your S3 bucket, stored in another S3 bucket.

**Key Concepts:**
- Logs include: requester, bucket, request time, action, response status, error codes.
- Logs are delivered **on a best-effort basis** with a delay of a few hours.
- **Never log to the same bucket** — causes a logging loop that grows storage infinitely.
- Common use: security auditing, compliance, access pattern analysis.
- Can be analyzed with **Amazon Athena** using SQL queries.

---

### 32. S3 Access Logs Hands On

**Goal:** Enable server access logging.

**Steps:**
1. Create a **separate logging bucket** (e.g., `my-bucket-logs-2024`).
2. In your main bucket → **Properties** → **Server access logging** → **Edit**.
3. Enable → Target bucket: select `my-bucket-logs-2024` → Target prefix: `logs/`.
4. Save. Now make some requests (upload, download, delete) to your main bucket.
5. Wait 1–2 hours → check the logging bucket for log files.
6. Open a log file and read the space-delimited entries.

**Exercise:**
- Use **Amazon Athena** to query your logs:
  ```sql
  SELECT requester, operation, key, httpstatus, bytessent
  FROM s3_access_logs
  WHERE httpstatus = '403'
  LIMIT 100;
  ```

---

### 33. S3 Pre-signed URLs

**What it is:** Generate a temporary URL that grants time-limited access to a private S3 object.

**Key Concepts:**
- Inherits the permissions of the **IAM user/role that generates it**.
- Can be for **GET** (download) or **PUT** (upload) operations.
- Default expiration: 3,600 seconds (1 hour). Maximum: 7 days (for IAM roles: 1 hour via console, 7 days via CLI with STS).
- Use cases: share private files temporarily, let users upload directly to S3 without AWS credentials.

**Common Architecture:**
```
User → App Backend → Generate Pre-signed URL → Return URL to User → User uploads/downloads directly to S3
```

---

### 34. S3 Pre-signed URLs Hands On

**Goal:** Generate a pre-signed URL and use it for secure temporary access.

**Steps — via Console:**
1. Open your private bucket → click on any private object.
2. Click **Object actions** → **Share with a pre-signed URL**.
3. Set duration (e.g., 10 minutes) → **Create presigned URL**.
4. Copy the URL and paste into a browser — it works!
5. Wait for the URL to expire and try again — observe `Request has expired` error.

**Steps — via AWS CLI:**
```bash
# Generate a pre-signed URL valid for 300 seconds
aws s3 presign s3://my-bucket/private-file.txt --expires-in 300

# Generate a PUT pre-signed URL (for uploads)
aws s3 presign s3://my-bucket/upload-target.txt \
  --expires-in 3600 \
  --http-method PUT
```

**Exercise:**
- Build a simple HTML form that uploads directly to S3 using a PUT pre-signed URL.
- Generate a pre-signed URL for a Glacier object (must restore it first).

---

### 35. Glacier Vault Lock & S3 Object Lock

**What it is:** WORM (Write Once Read Many) protection — once written, objects cannot be modified or deleted.

#### S3 Object Lock (on S3 buckets):
- **Retention Modes:**
  - **Compliance Mode** — Not even the root user can delete/modify. Cannot shorten retention period.
  - **Governance Mode** — Users with special IAM permissions CAN override. More flexible.
- **Retention Period** — Fixed time period during which the object is locked.
- **Legal Hold** — Independent of retention period; can be placed/removed by users with `s3:PutObjectLegalHold` permission.
- Must be enabled at bucket creation time.

#### Glacier Vault Lock:
- Similar WORM policy for **Glacier Vaults**.
- Policy is **locked** — once locked, it cannot be changed even by root/admins.
- Two-step process: initiate lock (24-hour window to validate) → complete lock.

**Use Case:** Compliance regulations like SEC 17a-4, HIPAA, PCI-DSS that require immutable records.

**Exercise:**
1. Create a new bucket with **Object Lock enabled** (must be enabled at creation).
2. Upload an object and apply a **Governance mode** lock for 1 day.
3. Try deleting the object — observe the denial.
4. Try deleting with a user that has `s3:BypassGovernanceRetention` — it should succeed.

---

### 36. S3 Access Points

**What it is:** Named network endpoints with their own access policies — simplify security management for shared buckets.

**Key Concepts:**
- Each Access Point has its own **DNS name** and **access point policy** (like a mini bucket policy).
- You can create different access points for different teams/applications, each with scoped permissions.
- Access Points can be restricted to a **VPC** (private access only — no internet).
- The bucket policy must delegate control to access points (or trust all access points from the account).
- Simplifies management: instead of one complex bucket policy, you have many focused access point policies.

**Example:**
```
Finance Team → Access Point (finance-ap) → Bucket (company-data)
               └─ Policy: Allow read on /finance/*

Engineering  → Access Point (eng-ap)     → Bucket (company-data)
               └─ Policy: Allow read/write on /code/*
```

**Steps:**
1. Open your bucket → **Access Points** → **Create access point**.
2. Name it (e.g., `finance-access-point`).
3. Network: **Internet** or **VPC** (choose VPC for private access).
4. Add an access point policy scoped to a prefix.
5. Use the access point ARN/alias in your application instead of the bucket name.

---

### 37. S3 Object Lambda

**What it is:** Use AWS Lambda to transform S3 objects **on the fly** as they are retrieved, without storing modified copies.

**Key Concepts:**
- When an application fetches an object, Lambda intercepts the request, transforms the data, and returns the modified result.
- The original object in S3 remains unchanged.
- Use cases:
  - **PII Redaction** — Remove sensitive data before returning to the app.
  - **Image Resizing** — Dynamically resize images for different devices.
  - **Data Format Conversion** — Convert XML to JSON on retrieval.
  - **Watermarking** — Add watermarks to documents/images.
- Requires: an **S3 Access Point** + an **S3 Object Lambda Access Point**.

**Architecture:**
```
App → S3 Object Lambda Access Point → Lambda (transforms data) → S3 Access Point → S3 Bucket
```

**Steps:**
1. Create an S3 Access Point for your bucket (from Step 36).
2. Create a Lambda function that receives the `GetObject` event and returns transformed data.
3. Go to **S3 → Object Lambda Access Points → Create**.
4. Link it to your Access Point and your Lambda function.
5. Use the Object Lambda Access Point ARN in your app instead of the bucket URL.
6. All GET requests are now intercepted by Lambda.

**Exercise:**
- Build a Lambda that converts CSV to JSON on retrieval.
- Build a Lambda that redacts any field named `email` or `ssn` from JSON objects.
- Test performance — measure latency added by Lambda transformation.

---

## 🗺️ Suggested Learning Path

```
Week 1 (Beginner)
├── Day 1: S3 Overview + First bucket + Object upload
├── Day 2: Bucket Policies + Public access
├── Day 3: Static Website Hosting
└── Day 4: Versioning

Week 2 (Intermediate)
├── Day 1: Replication (CRR + SRR)
├── Day 2: Storage Classes + Lifecycle Rules
├── Day 3: Event Notifications + Lambda integration
└── Day 4: Performance + Batch Operations

Week 3 (Advanced)
├── Day 1: Encryption (SSE-S3, SSE-KMS, SSE-C)
├── Day 2: CORS + Pre-signed URLs
├── Day 3: MFA Delete + Object Lock + Access Logs
└── Day 4: Access Points + Object Lambda + Storage Lens
```

---

## 💡 General Best Practices

| Practice | Why |
|---|---|
| Always enable **versioning** on production buckets | Protect against accidental deletion |
| Use **S3 Lifecycle Rules** | Reduce costs automatically |
| Enable **default encryption** (SSE-S3 or SSE-KMS) | Data protection at rest |
| Never log to the **same bucket** | Avoid logging loops |
| Use **Pre-signed URLs** for temporary access | Avoid making buckets public |
| Enable **MFA Delete** on critical buckets | Prevent ransomware/accidental deletion |
| Use **Access Points** for multi-team buckets | Simplify access management |
| Enable **S3 Storage Lens** | Monitor usage and reduce waste |
| Delete **incomplete multipart uploads** via Lifecycle | Avoid hidden storage costs |
| Use **Block Public Access** at the account level | Defense in depth |

---

## 🔗 Useful Resources

- [AWS S3 Documentation](https://docs.aws.amazon.com/s3/)
- [S3 Pricing](https://aws.amazon.com/s3/pricing/)
- [AWS Policy Generator](https://awspolicygen.s3.amazonaws.com/policygen.html)
- [S3 Storage Classes Comparison](https://aws.amazon.com/s3/storage-classes/)
- [AWS CLI S3 Reference](https://docs.aws.amazon.com/cli/latest/reference/s3/)
- [S3 Transfer Acceleration Speed Comparison](https://s3-accelerate-speedtest.s3-accelerate.amazonaws.com/en/accelerate-speed-comparsion.html)

---

*Guide covers AWS SAA-C03 / SAP-C02 exam objectives. All steps based on AWS Console as of 2024.*
