# AWS Messaging & Observability Mentor Guide
## SNS · SQS · CloudWatch · CloudWatch API · CloudWatch Logs — Console + CLI + Real-World Patterns

> **How to use this guide:** Read it top to bottom once for concepts, then use it as a living reference during your project. Every subtopic has three parts: **Concept**, **Console Steps**, **CLI Steps**, followed by **Real-Time Use Cases**. Replace placeholder values like `<ACCOUNT_ID>`, `<REGION>`, `<QUEUE_NAME>` with your own.

---

## Table of Contents

0. [Prerequisites & Environment Setup](#0-prerequisites--environment-setup)
1. [The Big Picture — How These Services Fit Together](#1-the-big-picture--how-these-services-fit-together)
2. [Amazon SNS (Simple Notification Service)](#2-amazon-sns-simple-notification-service)
3. [Amazon SQS (Simple Queue Service)](#3-amazon-sqs-simple-queue-service)
4. [Amazon CloudWatch — Metrics, Alarms & Dashboards](#4-amazon-cloudwatch--metrics-alarms--dashboards)
5. [CloudWatch API / CLI / SDK Access Patterns](#5-cloudwatch-api--cli--sdk-access-patterns)
6. [Amazon CloudWatch Logs](#6-amazon-cloudwatch-logs)
7. [End-to-End Real-Time Project Architecture](#7-end-to-end-real-time-project-architecture)
8. [IAM Permission Reference](#8-iam-permission-reference)
9. [Troubleshooting & FAQ](#9-troubleshooting--faq)
10. [Cost Optimization Tips](#10-cost-optimization-tips)
11. [CLI Cheat Sheet (All Commands in One Place)](#11-cli-cheat-sheet-all-commands-in-one-place)
12. [Suggested Learning Path (Day-wise)](#12-suggested-learning-path-day-wise)

---

## 0. Prerequisites & Environment Setup

### 0.1 AWS Account & IAM User
Never use the root account for daily work.

**Console steps:**
1. Sign in to the AWS Console → go to **IAM** → **Users** → **Create user**.
2. Name it (e.g., `devops-trainee`), enable **Console access** if you need UI login.
3. Attach permissions: for learning, you can attach `AmazonSNSFullAccess`, `AmazonSQSFullAccess`, `CloudWatchFullAccess`, `CloudWatchLogsFullAccess`. In a real project, scope this down (see [Section 8](#8-iam-permission-reference)).
4. Go to **Security credentials** → **Create access key** → choose "Command Line Interface (CLI)" → download/save the Access Key ID and Secret Access Key.

### 0.2 Install & Configure AWS CLI
```bash
# Check if installed
aws --version

# Install (Linux)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure credentials
aws configure
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region name: ap-south-1   (or your region)
# Default output format: json
```

You can also keep multiple named profiles:
```bash
aws configure --profile devops-trainee
aws sns list-topics --profile devops-trainee
```

### 0.3 Sanity Check
```bash
aws sts get-caller-identity
```
This confirms which account/user the CLI is authenticated as — run this whenever something "doesn't work," it's the #1 debugging step.

---

## 1. The Big Picture — How These Services Fit Together

| Service | Category | One-line purpose |
|---|---|---|
| **SNS** | Pub/Sub messaging | Broadcast one message to many subscribers (fan-out) |
| **SQS** | Queueing | Decouple producers and consumers; buffer work reliably |
| **CloudWatch (Metrics/Alarms/Dashboards)** | Monitoring | Track numeric health/performance data and alert on thresholds |
| **CloudWatch API/CLI/SDK** | Access layer | Programmatic way to push metrics, manage alarms, query logs |
| **CloudWatch Logs** | Logging | Centralize, search, and alert on text log data |

**Mental model:**
- **SQS** = a mailbox / to-do list. One message, one consumer (per message) at a time, that consumer can choose when to read it.
- **SNS** = a megaphone. One message, broadcast to every subscriber simultaneously (email, SMS, Lambda, SQS queues, HTTP endpoints).
- **SNS + SQS combo ("fan-out pattern")** = megaphone shouting into multiple mailboxes — extremely common in real systems.
- **CloudWatch** = the dashboard in your car (speed, fuel, temperature) + the warning lights (alarms).
- **CloudWatch Logs** = the black-box flight recorder — the actual text/event trail of what happened.
- Almost every production alerting pipeline looks like: **App emits logs/metrics → CloudWatch detects a problem → CloudWatch Alarm fires → SNS notifies a human or triggers automation.**

```
 Producers (EC2 / Lambda / ECS / on-prem app)
        |
        |--(logs)----------> CloudWatch Logs --> Metric Filter --> CloudWatch Alarm --+
        |                                                                              |
        |--(custom metrics)-> CloudWatch Metrics ------------------------------------> +--> SNS Topic --> Email / SMS / Slack(via Lambda) / PagerDuty
        |
        |--(events/jobs)----> SQS Queue <---(fan-out)--- SNS Topic <--- Publisher
                                  |
                                  v
                          Worker Lambda/EC2 consumes & processes
```

---

## 2. Amazon SNS (Simple Notification Service)

### 2.1 Core Concepts

- **Topic**: A named communication channel. Publishers send messages to a topic; subscribers receive them.
- **Publisher**: Anything that sends a message to a topic (app code, CloudWatch Alarm, S3 event, another AWS service).
- **Subscriber/Subscription**: An endpoint that receives messages from a topic. Supported protocols:
  - **Email / Email-JSON** — human notifications
  - **SMS** — text message alerts
  - **HTTP/HTTPS** — webhook to your own server
  - **SQS** — fan-out into a queue (most common in pipelines)
  - **Lambda** — trigger a function directly
  - **Application (mobile push)** — APNs/FCM push notifications
  - **Firehose** — stream into Kinesis Data Firehose
- **Standard Topic vs FIFO Topic**:
  - *Standard*: at-least-once delivery, best-effort ordering, highest throughput.
  - *FIFO* (`.fifo` suffix required): strict ordering + exactly-once publish, used when message sequence matters (e.g., financial transactions). Lower throughput than Standard.
- **Message Attributes & Filter Policies**: Subscribers can filter which messages they receive based on attributes, instead of getting every message published to the topic.
- **Access Policy**: A resource-based JSON policy controlling who can publish/subscribe to the topic (cross-account access, specific services like S3 or CloudWatch).
- **Delivery Retry Policy**: Configurable retry behavior for HTTP/S endpoints (backoff, max retries).
- **Dead-Letter Queue (DLQ) for SNS subscriptions**: If a subscriber repeatedly fails to receive a message, SNS can redirect it to an SQS DLQ instead of dropping it.
- **Encryption**: Server-side encryption (SSE) using KMS for messages at rest.

### 2.2 Console Steps

**Create a topic:**
1. Console → **SNS** → **Topics** → **Create topic**.
2. Choose **Standard** or **FIFO**.
3. Name it (e.g., `order-events` or `order-events.fifo`).
4. (Optional) Enable encryption, set access policy → **Create topic**.

**Create a subscription:**
1. Open the topic → **Create subscription**.
2. Choose protocol (e.g., Email).
3. Enter endpoint (e.g., your email address) → **Create subscription**.
4. For Email/SMS: check your inbox/phone and **confirm the subscription** — it stays "Pending confirmation" until you click the confirm link.

**Publish a message (console, for testing):**
1. Open the topic → **Publish message**.
2. Enter subject + message body → (optional) add message attributes → **Publish message**.

**Set a filter policy:**
1. Open the **subscription** (not the topic) → **Edit**.
2. Under **Subscription filter policy**, add JSON like:
```json
{ "event_type": ["ORDER_FAILED", "PAYMENT_FAILED"] }
```
3. Save — this subscriber now only receives messages whose attributes match.

**Configure a DLQ for a subscription:**
1. Create an SQS queue first (see Section 3).
2. On the subscription → **Edit** → **Subscription dead-letter queue** → select the SQS queue → **Save**.

### 2.3 CLI Steps

```bash
# Create a standard topic
aws sns create-topic --name order-events

# Create a FIFO topic
aws sns create-topic --name order-events.fifo --attributes FifoTopic=true

# List topics
aws sns list-topics

# Subscribe an email endpoint
aws sns subscribe \
  --topic-arn arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events \
  --protocol email \
  --notification-endpoint you@example.com

# Subscribe an SQS queue (fan-out)
aws sns subscribe \
  --topic-arn arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:<REGION>:<ACCOUNT_ID>:order-queue

# Subscribe a Lambda function
aws sns subscribe \
  --topic-arn arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:<REGION>:<ACCOUNT_ID>:function:processOrder

# Publish a message
aws sns publish \
  --topic-arn arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events \
  --message "Order #1234 has failed" \
  --subject "Order Failure Alert" \
  --message-attributes '{"event_type":{"DataType":"String","StringValue":"ORDER_FAILED"}}'

# Set a filter policy on a subscription
aws sns set-subscription-attributes \
  --subscription-arn <SUBSCRIPTION_ARN> \
  --attribute-name FilterPolicy \
  --attribute-value '{"event_type":["ORDER_FAILED","PAYMENT_FAILED"]}'

# Set a redrive policy (DLQ) on a subscription
aws sns set-subscription-attributes \
  --subscription-arn <SUBSCRIPTION_ARN> \
  --attribute-name RedrivePolicy \
  --attribute-value '{"deadLetterTargetArn":"arn:aws:sqs:<REGION>:<ACCOUNT_ID>:order-dlq"}'

# List subscriptions for a topic
aws sns list-subscriptions-by-topic --topic-arn arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events

# Delete a topic
aws sns delete-topic --topic-arn arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events
```

### 2.4 Real-Time Use Cases & Patterns

1. **Alarm notifications** — CloudWatch Alarm → SNS Topic → Email/SMS/PagerDuty. This is the #1 use of SNS in DevOps.
2. **Fan-out architecture** — One SNS topic → multiple SQS queues, each owned by a different microservice (e.g., `OrderPlaced` event consumed independently by Billing, Inventory, and Shipping services).
3. **Decoupling microservices** — Service A publishes events without knowing who's listening; new consumers can subscribe later without changing Service A.
4. **Mobile push notifications** — App backend publishes to SNS, which fans out to APNs (iOS) / FCM (Android).
5. **Automated incident response** — SNS → Lambda subscriber that posts directly into Slack/Teams via webhook.

### 2.5 Best Practices
- Always use **message attributes + filter policies** instead of creating dozens of near-duplicate topics.
- Prefer **SNS → SQS fan-out** over **SNS → HTTP endpoint** for anything that needs reliability/retry — SQS persists the message; an HTTP endpoint that's down can lose it (unless you configure a DLQ).
- Use **FIFO topics** only when ordering truly matters; they cost more in complexity and throughput limits.
- Restrict the topic's **access policy** so random AWS accounts can't publish to or subscribe to it.

---

## 3. Amazon SQS (Simple Queue Service)

### 3.1 Core Concepts

- **Queue**: A buffer that holds messages until a consumer processes and deletes them.
- **Standard Queue**: Nearly unlimited throughput, at-least-once delivery, **best-effort ordering** (messages can arrive out of order, occasional duplicates).
- **FIFO Queue** (`.fifo` suffix): Strict ordering, exactly-once processing, throughput capped (~300 msg/sec without batching, ~3000/sec with batching).
- **Visibility Timeout**: When a consumer reads a message, it becomes "invisible" to other consumers for a set duration (default 30s) so it isn't processed twice. If the consumer doesn't delete it within this window, it reappears in the queue.
- **Message Retention Period**: How long an unconsumed message stays in the queue (default 4 days, max 14 days).
- **Long Polling vs Short Polling**: Long polling (`WaitTimeSeconds` up to 20s) waits for messages to arrive instead of returning empty immediately — reduces API calls/cost and latency. Always prefer long polling.
- **Dead-Letter Queue (DLQ) + Redrive Policy**: After a message fails processing `maxReceiveCount` times, SQS automatically moves it to a separate DLQ for investigation instead of retrying forever.
- **Delay Queues**: Messages are invisible for a set time after being sent (e.g., "process this 5 minutes from now").
- **Message Attributes**: Key-value metadata attached to a message, separate from the body (useful for routing/filtering).
- **Encryption**: SSE using SQS-managed keys or KMS.
- **Visibility into queue depth**: `ApproximateNumberOfMessages` — critical for autoscaling workers based on backlog.

### 3.2 Console Steps

**Create a queue:**
1. Console → **SQS** → **Create queue**.
2. Choose **Standard** or **FIFO**.
3. Name it (e.g., `order-queue`).
4. Configure: Visibility timeout, Message retention period, Delivery delay → **Create queue**.

**Configure a Dead-Letter Queue:**
1. First create a second queue to act as the DLQ (e.g., `order-dlq`).
2. Open the main queue → **Edit** → scroll to **Dead-letter queue** → **Enable**.
3. Select the DLQ ARN and set **Maximum receives** (e.g., 5) → **Save**.

**Send/receive/delete a message (console, for testing):**
1. Open the queue → **Send and receive messages**.
2. Under "Send message" tab, type a body → **Send message**.
3. Under "Receive messages" tab → **Poll for messages** → select the message → **Delete**.

**Set access policy:**
1. Open queue → **Access policy** tab → **Edit** → paste a JSON policy (e.g., allow only a specific SNS topic to send) → **Save**.

**Attach an SQS queue as a Lambda trigger:**
1. Open the Lambda function → **Add trigger** → select **SQS** → choose the queue → set batch size → **Add**.

### 3.3 CLI Steps

```bash
# Create a standard queue
aws sqs create-queue --queue-name order-queue

# Create a FIFO queue
aws sqs create-queue --queue-name order-queue.fifo --attributes FifoQueue=true,ContentBasedDeduplication=true

# Get queue URL (needed for most other commands)
aws sqs get-queue-url --queue-name order-queue

# Send a message
aws sqs send-message \
  --queue-url https://sqs.<REGION>.amazonaws.com/<ACCOUNT_ID>/order-queue \
  --message-body '{"orderId":"1234","status":"FAILED"}'

# Receive messages (long polling, wait up to 20s)
aws sqs receive-message \
  --queue-url https://sqs.<REGION>.amazonaws.com/<ACCOUNT_ID>/order-queue \
  --wait-time-seconds 20 \
  --max-number-of-messages 5

# Delete a message after processing (ReceiptHandle comes from receive-message output)
aws sqs delete-message \
  --queue-url https://sqs.<REGION>.amazonaws.com/<ACCOUNT_ID>/order-queue \
  --receipt-handle "<RECEIPT_HANDLE_FROM_RECEIVE>"

# Get queue attributes (e.g., approximate message count)
aws sqs get-queue-attributes \
  --queue-url https://sqs.<REGION>.amazonaws.com/<ACCOUNT_ID>/order-queue \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible

# Set a redrive policy (DLQ) on the main queue
aws sqs set-queue-attributes \
  --queue-url https://sqs.<REGION>.amazonaws.com/<ACCOUNT_ID>/order-queue \
  --attributes '{"RedrivePolicy":"{\"deadLetterTargetArn\":\"arn:aws:sqs:<REGION>:<ACCOUNT_ID>:order-dlq\",\"maxReceiveCount\":\"5\"}"}'

# Purge all messages from a queue (use carefully!)
aws sqs purge-queue --queue-url https://sqs.<REGION>.amazonaws.com/<ACCOUNT_ID>/order-queue

# Delete a queue
aws sqs delete-queue --queue-url https://sqs.<REGION>.amazonaws.com/<ACCOUNT_ID>/order-queue
```

### 3.4 Real-Time Use Cases & Patterns

1. **Worker/job queue** — Web app pushes long-running tasks (image processing, report generation) into SQS; a fleet of worker EC2/Lambda instances pulls and processes them, scaling independently of the web tier.
2. **SNS → SQS fan-out subscriber** — Each microservice owns its own queue subscribed to a shared event topic.
3. **Lambda event source mapping** — Lambda automatically polls SQS and invokes your function per batch of messages — no polling code needed.
4. **Buffering traffic spikes** — Protects downstream databases/services from being overwhelmed during traffic bursts (load leveling).
5. **Order processing pipelines** — Orders flow through Validate → Payment → Fulfillment queues, each stage decoupled.

### 3.5 Best Practices
- Always configure a **DLQ** in real projects — silent message loss/looping is a common production bug otherwise.
- Use **long polling** (`WaitTimeSeconds=20`) to cut empty-receive API costs significantly.
- Set **visibility timeout** to be longer than your typical processing time (a good rule: 6x your average processing duration) to avoid duplicate processing.
- For idempotency, design consumers so processing the same message twice has no harmful side effect (at-least-once delivery means duplicates *will* happen with Standard queues).
- Use **CloudWatch alarms on `ApproximateNumberOfMessagesVisible`** to detect backlog buildup and trigger autoscaling of consumers.

---

## 4. Amazon CloudWatch — Metrics, Alarms & Dashboards

### 4.1 Core Concepts

- **Metric**: A time-ordered set of data points (e.g., CPUUtilization, Errors, custom business KPI).
- **Namespace**: A container that organizes metrics (e.g., `AWS/EC2`, `AWS/Lambda`, or your own `MyApp/Orders`).
- **Dimension**: A name/value pair that uniquely identifies a metric within a namespace (e.g., `InstanceId=i-0123`).
- **Statistic**: How datapoints are aggregated over a period — `Average`, `Sum`, `Minimum`, `Maximum`, `SampleCount`, percentiles (`p90`, `p99`).
- **Period**: The granularity of aggregation in seconds (e.g., 60s, 300s).
- **Standard vs Detailed Monitoring**: EC2 default metrics come every 5 minutes (standard) unless you enable detailed monitoring (1 minute, additional cost).
- **Custom Metrics**: Any metric you push yourself via `put-metric-data` (e.g., "items in cart," "queue lag," "active sessions").
- **Alarm**: Watches one metric (or a math expression of metrics) and changes state — `OK`, `ALARM`, `INSUFFICIENT_DATA` — based on a threshold over a number of evaluation periods. Alarms trigger **Actions**: notify an SNS topic, trigger Auto Scaling, trigger EC2 actions (reboot/stop).
- **Composite Alarms**: Combine multiple alarms with AND/OR logic (e.g., alert only if both high CPU AND high error rate are true — reduces noise).
- **Anomaly Detection**: CloudWatch learns a metric's normal pattern (band) and alarms on statistical deviation instead of a fixed threshold.
- **Dashboards**: Visual, customizable widgets (graphs, numbers, text, logs) combining metrics from multiple services into one view.
- **CloudWatch Events / EventBridge** (related but distinct service): Reacts to *state changes* (e.g., "EC2 instance terminated," "S3 object created," scheduled cron-like rules) and routes them to targets like Lambda or SNS. Often used alongside CloudWatch Alarms in real architectures.

### 4.2 Console Steps

**View built-in metrics:**
1. Console → **CloudWatch** → **Metrics** → **All metrics**.
2. Browse by namespace (e.g., `EC2` → `Per-Instance Metrics`) → select a metric → it plots on a graph.

**Create an alarm:**
1. **CloudWatch** → **Alarms** → **All alarms** → **Create alarm**.
2. **Select metric** (e.g., EC2 `CPUUtilization` for a specific instance).
3. Set **Statistic** = Average, **Period** = 5 minutes.
4. Set **Condition**: Threshold type = Static, "Greater than" 80.
5. Set **Additional configuration**: "Datapoints to alarm" (e.g., 3 out of 3) to avoid false positives from a single spike.
6. **Configure actions** → **In alarm** → select **SNS topic** (existing or create new) → choose your `order-events`-style alert topic.
7. Name and create the alarm.

**Create a composite alarm:**
1. **Alarms** → **Create composite alarm**.
2. Build a rule like `ALARM("HighCPU") AND ALARM("HighErrorRate")`.
3. Attach an SNS action → **Create**.

**Create a dashboard:**
1. **CloudWatch** → **Dashboards** → **Create dashboard** → name it.
2. **Add widget** → choose type (Line, Number, Alarm status, Logs table, etc.).
3. Select the metric(s)/alarm(s)/log query to display → **Create widget** → repeat → **Save dashboard**.

**Enable anomaly detection:**
1. On a metric graph → **Actions** → **Create anomaly detection alarm**.
2. Configure sensitivity (standard deviation band) → set SNS action → **Create**.

### 4.3 CLI Steps

```bash
# Push a custom metric data point
aws cloudwatch put-metric-data \
  --namespace "MyApp/Orders" \
  --metric-name "OrdersFailed" \
  --dimensions Service=Checkout,Env=Production \
  --value 1 \
  --unit Count

# List metrics in a namespace
aws cloudwatch list-metrics --namespace "AWS/EC2"

# Get statistics for a metric over time
aws cloudwatch get-metric-statistics \
  --namespace "AWS/EC2" \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time 2026-06-23T00:00:00Z \
  --end-time 2026-06-24T00:00:00Z \
  --period 300 \
  --statistics Average Maximum

# Create a standard alarm with an SNS action
aws cloudwatch put-metric-alarm \
  --alarm-name "High-CPU-WebServer" \
  --namespace "AWS/EC2" \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events \
  --ok-actions arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events

# Create a composite alarm
aws cloudwatch put-composite-alarm \
  --alarm-name "Checkout-Service-Unhealthy" \
  --alarm-rule '(ALARM("High-CPU-WebServer") AND ALARM("High-Error-Rate"))' \
  --alarm-actions arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events

# Describe alarms and their current state
aws cloudwatch describe-alarms --alarm-names "High-CPU-WebServer"

# List alarm history
aws cloudwatch describe-alarm-history --alarm-name "High-CPU-WebServer"

# Create/update a dashboard from a JSON definition file
aws cloudwatch put-dashboard \
  --dashboard-name "OrdersServiceDashboard" \
  --dashboard-body file://dashboard.json

# Delete an alarm
aws cloudwatch delete-alarms --alarm-names "High-CPU-WebServer"
```

Example minimal `dashboard.json`:
```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "metrics": [["AWS/EC2", "CPUUtilization", "InstanceId", "i-0123456789abcdef0"]],
        "period": 300,
        "stat": "Average",
        "region": "<REGION>",
        "title": "Web Server CPU"
      }
    }
  ]
}
```

### 4.4 Real-Time Use Cases

1. **Auto Scaling triggers** — CPU/RequestCount alarms scale EC2/ECS fleets up or down automatically.
2. **SLA/SLO monitoring** — Alarm on p99 latency or error rate exceeding agreed thresholds.
3. **Business KPIs** — Custom metric `OrdersFailed`, `CartAbandonment`, `PaymentDeclines` pushed from app code, visualized on an executive dashboard.
4. **Queue backlog alarms** — Alarm on SQS `ApproximateNumberOfMessagesVisible` to detect stuck consumers.
5. **Cost-saving idle detection** — Alarm on low CPU over long periods to flag/stop unused instances.
6. **Composite alarms for noise reduction** — Only page on-call engineers when multiple signals agree something is actually wrong.

### 4.5 Best Practices
- Always set **"Datapoints to alarm" > 1** for noisy metrics (e.g., 3 of 3) to avoid alert fatigue from transient spikes.
- Route **all alarms through SNS**, never leave an alarm with no action — it becomes invisible.
- Tag custom metrics with consistent **dimensions** (Service, Environment) so they're filterable later.
- Use **dashboards per service/team**, not one giant dashboard nobody reads.
- Prefer **anomaly detection** for metrics with daily/weekly seasonality (e.g., traffic) instead of static thresholds.

---

## 5. CloudWatch API / CLI / SDK Access Patterns

This isn't a separate AWS service — it's *how* you interact with CloudWatch programmatically, which matters a lot in real DevOps work (automation, custom monitoring agents, CI/CD pipelines).

### 5.1 Core Concepts
- All console actions map 1:1 to API calls (`PutMetricAlarm`, `PutMetricData`, `GetMetricData`, `DescribeAlarms`, etc.) — the CLI is a thin wrapper over this API.
- **boto3** (Python SDK) is the most common way apps push custom metrics or query data programmatically.
- **GetMetricData** (newer, supports math expressions) is generally preferred over the older **GetMetricStatistics** for anything beyond a single simple query.
- Authentication for API/CLI calls uses the same IAM credentials/roles as everything else — on EC2/Lambda, prefer an **IAM Role** over hardcoded keys.
- Rate limits exist (API throttling) — for high-volume custom metrics, batch your `put-metric-data` calls (up to 1000 metrics per call) instead of one call per data point.

### 5.2 Example: Pushing a Custom Metric via boto3 (Python)
```python
import boto3

cloudwatch = boto3.client("cloudwatch", region_name="<REGION>")

cloudwatch.put_metric_data(
    Namespace="MyApp/Orders",
    MetricData=[
        {
            "MetricName": "OrdersFailed",
            "Dimensions": [
                {"Name": "Service", "Value": "Checkout"},
                {"Name": "Env", "Value": "Production"},
            ],
            "Value": 1,
            "Unit": "Count",
        }
    ],
)
```

### 5.3 Example: Querying Metrics with GetMetricData (CLI)
```bash
aws cloudwatch get-metric-data \
  --metric-data-queries '[
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/Lambda",
          "MetricName": "Errors",
          "Dimensions": [{"Name":"FunctionName","Value":"processOrder"}]
        },
        "Period": 300,
        "Stat": "Sum"
      }
    }
  ]' \
  --start-time 2026-06-23T00:00:00Z \
  --end-time 2026-06-24T00:00:00Z
```

### 5.4 Real-Time Use Cases
- A **deployment script** (CI/CD pipeline) calls `put-metric-data` to record deployment frequency/success, then queries it later for DORA metrics dashboards.
- A **custom health-check Lambda** runs every minute, calls `GetMetricData` for downstream service latency, and triggers a custom alert path beyond what static alarms can express.
- **Cross-account monitoring** tools call the CloudWatch API with assumed IAM roles to pull metrics from multiple AWS accounts into one central observability platform.

### 5.5 Best Practices
- Prefer **IAM roles** (EC2 instance profile, Lambda execution role, ECS task role) over static access keys for anything running in AWS.
- Batch metric pushes to avoid throttling and reduce cost.
- Use **`--profile`** with named CLI profiles when managing multiple AWS accounts from one machine.

---

## 6. Amazon CloudWatch Logs

### 6.1 Core Concepts

- **Log Group**: A container for log streams from one source (e.g., `/aws/lambda/processOrder`, `/var/log/app/access.log`).
- **Log Stream**: A sequence of log events from a single source within a group (e.g., one Lambda execution environment, one EC2 instance).
- **Retention Policy**: How long logs are kept (default: *never expire* — costs add up; set this explicitly, e.g., 30/90/365 days).
- **CloudWatch Agent (Unified Agent)**: Software installed on EC2/on-prem servers to ship log files and system-level metrics (memory, disk — which aren't collected by default) to CloudWatch.
- **Metric Filters**: Scan incoming log lines for a pattern (e.g., the word `ERROR`) and increment a custom CloudWatch metric — this is the bridge connecting Logs → Metrics → Alarms → SNS.
- **Subscription Filters**: Stream matching log events in real time to a Lambda function, Kinesis stream, or Kinesis Firehose for further processing (e.g., SIEM ingestion).
- **CloudWatch Logs Insights**: A purpose-built query language to search/aggregate log data interactively (similar to SQL-ish syntax).
- **Log Export to S3**: Batch export of historical logs (not real-time) for long-term archival/compliance.
- **Lambda Auto-Logging**: Every Lambda function automatically logs to a `/aws/lambda/<function-name>` group with zero setup.

### 6.2 Console Steps

**View log groups & streams:**
1. Console → **CloudWatch** → **Log groups**.
2. Click a group (e.g., `/aws/lambda/processOrder`) → click a log stream → view individual log lines.

**Set retention policy:**
1. Open the log group → **Actions** → **Edit retention setting** → choose e.g., 30 days → **Save**.

**Create a metric filter (Logs → Metric):**
1. Open the log group → **Metric filters** tab → **Create metric filter**.
2. Define **filter pattern**, e.g., `"ERROR"` or `{ $.level = "ERROR" }` for JSON logs.
3. Test against a sample log to confirm it matches.
4. Assign **Metric namespace/name** (e.g., `MyApp/Logs`, `ErrorCount`) → **Create**.
5. Now go create a **CloudWatch Alarm** on this new metric (see Section 4.2) → attach an SNS action. This completes Logs → Metric → Alarm → SNS.

**Create a subscription filter (Logs → Lambda real-time streaming):**
1. Open log group → **Subscription filters** tab → **Create** → **Create Lambda subscription filter**.
2. Choose the destination Lambda function and define the filter pattern → **Start streaming**.

**Run a Logs Insights query:**
1. Console → **CloudWatch** → **Logs Insights**.
2. Select one or more log groups.
3. Write a query, e.g.:
```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 50
```
4. **Run query**.

**Export logs to S3:**
1. Open log group → **Actions** → **Export data to Amazon S3**.
2. Choose time range and destination bucket → **Export**.

### 6.3 CLI Steps

```bash
# Create a log group
aws logs create-log-group --log-group-name /myapp/checkout-service

# Set retention policy (in days)
aws logs put-retention-policy \
  --log-group-name /myapp/checkout-service \
  --retention-in-days 30

# Create a log stream (manual logging use case)
aws logs create-log-stream \
  --log-group-name /myapp/checkout-service \
  --log-stream-name instance-i-0123456789abcdef0

# Push log events manually (rare — normally an agent/SDK does this)
aws logs put-log-events \
  --log-group-name /myapp/checkout-service \
  --log-stream-name instance-i-0123456789abcdef0 \
  --log-events timestamp=$(date +%s000),message="Service started"

# Search/filter log events
aws logs filter-log-events \
  --log-group-name /myapp/checkout-service \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s000)

# Create a metric filter
aws logs put-metric-filter \
  --log-group-name /myapp/checkout-service \
  --filter-name ErrorCountFilter \
  --filter-pattern "ERROR" \
  --metric-transformations \
      metricName=ErrorCount,metricNamespace=MyApp/Logs,metricValue=1,defaultValue=0

# Create a subscription filter to stream into a Lambda
aws logs put-subscription-filter \
  --log-group-name /myapp/checkout-service \
  --filter-name StreamErrorsToLambda \
  --filter-pattern "ERROR" \
  --destination-arn arn:aws:lambda:<REGION>:<ACCOUNT_ID>:function:logProcessor

# Start a Logs Insights query
aws logs start-query \
  --log-group-name /myapp/checkout-service \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'

# Fetch results of that query (use the queryId returned above)
aws logs get-query-results --query-id <QUERY_ID>

# Export logs to S3 (async task)
aws logs create-export-task \
  --log-group-name /myapp/checkout-service \
  --from $(date -d '7 days ago' +%s000) \
  --to $(date +%s000) \
  --destination my-log-archive-bucket \
  --destination-prefix checkout-service-logs

# Delete a log group
aws logs delete-log-group --log-group-name /myapp/checkout-service
```

### 6.4 Installing the CloudWatch Agent on EC2 (Real Steps)

1. Attach an IAM role with the `CloudWatchAgentServerPolicy` to the EC2 instance.
2. SSH into the instance and install the agent:
```bash
sudo yum install -y amazon-cloudwatch-agent   # Amazon Linux/RHEL
# or: sudo apt-get install -y amazon-cloudwatch-agent   # Ubuntu/Debian
```
3. Create a config file (or use the wizard):
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```
4. Or write `/opt/aws/amazon-cloudwatch-agent/etc/config.json` manually, e.g.:
```json
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/app/access.log",
            "log_group_name": "/myapp/checkout-service",
            "log_stream_name": "{instance_id}"
          }
        ]
      }
    }
  },
  "metrics": {
    "metrics_collected": {
      "mem": { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["used_percent"], "resources": ["/"] }
    }
  }
}
```
5. Start the agent:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json -s
```
6. Verify in the console under **CloudWatch → Log groups** that `/myapp/checkout-service` is receiving data, and under **Metrics → CWAgent** namespace for memory/disk metrics (which EC2 does **not** report by default).

### 6.5 Real-Time Use Cases

1. **Centralized logging** — All EC2/ECS/Lambda logs flow into CloudWatch Logs instead of scattering across individual servers, critical once you have more than one instance.
2. **Error-rate alerting pipeline** — Metric filter counts `"ERROR"` occurrences → CloudWatch alarm on that metric → SNS notifies the on-call engineer the moment errors spike, without anyone watching logs manually.
3. **Lambda debugging** — Every Lambda invocation's `print()`/`console.log()` output lands automatically in `/aws/lambda/<function-name>`; Logs Insights lets you trace a specific request by `requestId`.
4. **Security/SIEM streaming** — Subscription filters stream specific log patterns (e.g., failed logins) into a security pipeline in near real time.
5. **Compliance archival** — Export task ships logs older than X days to S3 for cheap long-term storage, then CloudWatch retention can be shortened to control cost.

### 6.6 Best Practices
- **Always set a retention policy** — "Never expire" silently accumulates storage cost forever.
- Structure application logs as **JSON** so metric filters and Logs Insights queries can target specific fields (`{ $.level = "ERROR" }`) instead of fragile string matching.
- Use **one log group per service/component**, not one giant shared group — easier permissions, retention, and cost control.
- Keep **hot/recent logs in CloudWatch** and **archive older logs to S3** (cheaper) using export tasks or S3 lifecycle-integrated logging.

---

## 7. End-to-End Real-Time Project Architecture

**Scenario: E-commerce Order Processing with Full Observability**

```
[API Gateway] --> [Lambda: createOrder]
                        |
                        |-- writes structured JSON logs --> CloudWatch Logs (/aws/lambda/createOrder)
                        |                                         |
                        |                                   metric filter "ERROR"
                        |                                         |
                        |                                   CloudWatch Alarm (ErrorCount > 5 in 5 min)
                        |                                         |
                        |                                         v
                        |                                  SNS Topic: ops-alerts --> Email/SMS/Slack(via Lambda)
                        |
                        v
                 SNS Topic: order-events
                   /        \
                  v          v
         SQS: billing-queue   SQS: inventory-queue
                  |                  |
                  v                  v
         Lambda: chargeCard   Lambda: reserveStock
                  |                  |
            (on failure -->)   (on failure -->)
                  v                  v
              billing-dlq       inventory-dlq
                  |                  |
                  +--------+---------+
                           v
              CloudWatch Alarm: DLQ depth > 0
                           |
                           v
                  SNS Topic: ops-alerts
```

**What's happening:**
1. `createOrder` Lambda logs everything as JSON; errors are auto-counted via a metric filter and alarmed on.
2. The successful order event is published once to `order-events` (SNS) and fanned out to two independent SQS queues — Billing and Inventory teams evolve their consumers independently.
3. Each consumer Lambda has its own DLQ — if charge/reservation fails repeatedly, the message lands in the DLQ instead of vanishing.
4. A CloudWatch alarm watches DLQ depth (`ApproximateNumberOfMessagesVisible > 0`) on both DLQs and pages the on-call SNS topic.
5. Everything (Lambda errors, DLQ depth, queue backlog, custom business metrics like `OrdersPlaced`) is visualized together on one CloudWatch dashboard.

### Setup Order (Recommended Sequence for a Real Project)
1. Create SQS queues + their DLQs first (`billing-queue`/`billing-dlq`, `inventory-queue`/`inventory-dlq`).
2. Create the SNS topic `order-events`, subscribe both SQS queues to it, set redrive policies.
3. Deploy Lambdas; wire SQS as event source for consumer Lambdas.
4. Set log retention on all Lambda log groups.
5. Create metric filters for `ERROR` patterns on each log group.
6. Create CloudWatch alarms: Lambda errors, DLQ depth, queue backlog — all pointing to an `ops-alerts` SNS topic.
7. Subscribe your team's email/Slack-Lambda/PagerDuty to `ops-alerts`.
8. Build a CloudWatch dashboard combining all of the above into one view.
9. (Optional, for IaC) Replicate all of the above in Terraform or AWS CloudFormation so the whole stack is reproducible — strongly recommended once you're past the learning phase.

---

## 8. IAM Permission Reference

Minimal example policy for an application that publishes to SNS, sends/receives SQS messages, and writes custom metrics/logs (scope ARNs down in production, this example is illustrative):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SNSPublish",
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "arn:aws:sns:<REGION>:<ACCOUNT_ID>:order-events"
    },
    {
      "Sid": "SQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": [
        "arn:aws:sqs:<REGION>:<ACCOUNT_ID>:billing-queue",
        "arn:aws:sqs:<REGION>:<ACCOUNT_ID>:inventory-queue"
      ]
    },
    {
      "Sid": "CloudWatchMetrics",
      "Effect": "Allow",
      "Action": ["cloudwatch:PutMetricData"],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:<REGION>:<ACCOUNT_ID>:log-group:/myapp/*"
    }
  ]
}
```

> Note: `cloudwatch:PutMetricData` does not support resource-level restriction (must be `"*"`); control it by restricting *who* can assume the role instead.

---

## 9. Troubleshooting & FAQ

| Symptom | Likely Cause | Fix |
|---|---|---|
| SNS subscription stuck on "Pending confirmation" | Confirmation link not clicked | Resend / check spam folder / click confirm link |
| Messages disappear without being processed | No DLQ configured + consumer crashing silently | Configure DLQ + redrive policy, inspect DLQ messages |
| Consumer processes same SQS message twice | Visibility timeout too short for processing time | Increase visibility timeout; make processing idempotent |
| CloudWatch Alarm stuck in `INSUFFICIENT_DATA` | No data points published yet, or wrong dimensions/namespace | Verify metric exists via `list-metrics`; check dimension names match exactly |
| Metric filter not matching | Pattern syntax wrong (text vs JSON pattern) | Use "Test pattern" in console against a real sample log line |
| Lambda logs not appearing | Execution role missing `logs:CreateLogGroup`/`PutLogEvents` | Attach `AWSLambdaBasicExecutionRole` or equivalent |
| `AccessDenied` on any CLI call | IAM permissions missing, or wrong profile/region | Run `aws sts get-caller-identity`; check attached policies |
| High CloudWatch Logs bill | No retention policy set ("Never expire") | Set retention policy on every log group |
| SNS → HTTP endpoint not receiving messages | Endpoint down/unreachable, no DLQ to catch failures | Add a DLQ to the subscription; check endpoint health |

---

## 10. Cost Optimization Tips

- **CloudWatch Logs**: Set retention policies everywhere (default is forever). Export older logs to S3 + Glacier for cheap long-term storage.
- **CloudWatch Metrics**: Custom metrics cost per metric/month — avoid creating high-cardinality dimensions (e.g., per-user-id dimensions explode cost).
- **CloudWatch Alarms**: Each alarm has a small monthly cost — consolidate with composite alarms where it makes sense rather than dozens of near-identical alarms.
- **SQS**: Long polling reduces the number of billable API requests dramatically compared to short polling/tight loops.
- **SNS**: SMS is the most expensive protocol — prefer email/Slack/push for non-critical alerts, reserve SMS for true pages.
- **Detailed (1-minute) EC2 monitoring**: Only enable where you genuinely need faster reaction time; standard 5-minute monitoring is free and often sufficient.

---

## 11. CLI Cheat Sheet (All Commands in One Place)

```bash
### SNS ###
aws sns create-topic --name <TOPIC_NAME>
aws sns list-topics
aws sns subscribe --topic-arn <ARN> --protocol email --notification-endpoint <EMAIL>
aws sns publish --topic-arn <ARN> --message "<TEXT>"
aws sns set-subscription-attributes --subscription-arn <ARN> --attribute-name FilterPolicy --attribute-value '<JSON>'
aws sns list-subscriptions-by-topic --topic-arn <ARN>
aws sns delete-topic --topic-arn <ARN>

### SQS ###
aws sqs create-queue --queue-name <NAME>
aws sqs get-queue-url --queue-name <NAME>
aws sqs send-message --queue-url <URL> --message-body "<TEXT>"
aws sqs receive-message --queue-url <URL> --wait-time-seconds 20
aws sqs delete-message --queue-url <URL> --receipt-handle <HANDLE>
aws sqs get-queue-attributes --queue-url <URL> --attribute-names All
aws sqs set-queue-attributes --queue-url <URL> --attributes '<JSON>'
aws sqs purge-queue --queue-url <URL>
aws sqs delete-queue --queue-url <URL>

### CloudWatch (Metrics/Alarms/Dashboards) ###
aws cloudwatch put-metric-data --namespace <NS> --metric-name <NAME> --value <N>
aws cloudwatch list-metrics --namespace <NS>
aws cloudwatch get-metric-data --metric-data-queries '<JSON>' --start-time <ISO> --end-time <ISO>
aws cloudwatch put-metric-alarm --alarm-name <NAME> --namespace <NS> --metric-name <METRIC> --statistic Average --period 300 --evaluation-periods 3 --threshold <N> --comparison-operator GreaterThanThreshold --alarm-actions <SNS_ARN>
aws cloudwatch describe-alarms
aws cloudwatch put-dashboard --dashboard-name <NAME> --dashboard-body file://dashboard.json
aws cloudwatch delete-alarms --alarm-names <NAME>

### CloudWatch Logs ###
aws logs create-log-group --log-group-name <NAME>
aws logs put-retention-policy --log-group-name <NAME> --retention-in-days 30
aws logs filter-log-events --log-group-name <NAME> --filter-pattern "ERROR"
aws logs put-metric-filter --log-group-name <NAME> --filter-name <NAME> --filter-pattern "ERROR" --metric-transformations metricName=<M>,metricNamespace=<NS>,metricValue=1,defaultValue=0
aws logs put-subscription-filter --log-group-name <NAME> --filter-name <NAME> --filter-pattern "ERROR" --destination-arn <LAMBDA_ARN>
aws logs start-query --log-group-name <NAME> --start-time <EPOCH> --end-time <EPOCH> --query-string '<QUERY>'
aws logs get-query-results --query-id <ID>
aws logs create-export-task --log-group-name <NAME> --from <EPOCH_MS> --to <EPOCH_MS> --destination <S3_BUCKET>
aws logs delete-log-group --log-group-name <NAME>

### Identity / Debug ###
aws sts get-caller-identity
aws configure list
```

---

## 12. Suggested Learning Path (Day-wise)

| Day | Focus | Hands-on Goal |
|---|---|---|
| 1 | SQS basics | Create a standard queue, send/receive/delete a message via console + CLI |
| 2 | SQS DLQ | Configure a DLQ + redrive policy; force a failure and watch the message land in DLQ |
| 3 | SNS basics | Create a topic, subscribe email + SQS, publish a test message, observe fan-out |
| 4 | SNS filtering | Add message attributes + a filter policy; confirm selective delivery |
| 5 | CloudWatch Metrics | Push a custom metric via CLI/boto3; view it on a graph |
| 6 | CloudWatch Alarms | Create an alarm on your custom metric; wire it to an SNS topic; trigger it manually |
| 7 | CloudWatch Dashboards | Build a dashboard combining EC2/Lambda metrics + your custom metric + alarm status |
| 8 | CloudWatch Logs basics | Deploy a small Lambda, view its auto-created log group, run a Logs Insights query |
| 9 | Metric Filters | Create a metric filter on `"ERROR"`, alarm on it, route to SNS — full pipeline |
| 10 | End-to-End project | Rebuild the architecture in [Section 7](#7-end-to-end-real-time-project-architecture) from scratch in your own account |
| 11+ | IaC (stretch goal) | Recreate the entire setup using Terraform or CloudFormation for repeatability |

---

*End of guide. Keep this file in your project repo (e.g., `docs/aws-messaging-observability.md`) and update it as your architecture evolves — it doubles as onboarding documentation for new team members.*
