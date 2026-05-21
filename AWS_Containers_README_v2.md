# AWS Containers — Practical Learning Guide
### ECR · ECS · EKS | Console & CLI | Beginner → Advanced

> **How to use this guide:** Follow the phases in order. Each topic has a Console section and a CLI section — practice both. Complete the hands-on exercises before moving to the next phase. You will need an AWS account with admin or power-user IAM access.

---

## Prerequisites & Setup

### Tools to Install

```bash
# AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws --version

# Docker
sudo apt-get install docker.io -y   # Ubuntu/Debian
# OR: https://docs.docker.com/get-docker/

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client

# eksctl (for EKS)
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/
eksctl version

# Helm (for EKS add-ons)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Configure AWS CLI

```bash
aws configure
# AWS Access Key ID:     <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name:   ap-south-1          # Mumbai — change to your region
# Default output format: json

# Verify
aws sts get-caller-identity
```

### Key Concepts to Understand First

| Term | Meaning |
|------|---------|
| **Image** | Packaged snapshot of your app + dependencies (like a ZIP file) |
| **Container** | Running instance of an image |
| **Registry** | Storage for images (ECR = AWS's registry) |
| **Orchestrator** | System that runs and manages containers at scale (ECS, EKS) |
| **Task / Pod** | The unit that runs one or more containers |
| **Cluster** | Group of compute resources that run your tasks/pods |
| **IAM Role** | Permissions attached to a service or task |

---

---

# PART 1 — Amazon ECR (Elastic Container Registry)

> **What is ECR?** AWS's private Docker image registry. Think of it as a private Docker Hub hosted inside your AWS account.

---

## Phase 1 — ECR Basics

### Topic 1.1 — Create a Repository

#### Console

1. Open **AWS Console** → search **ECR** → click **Elastic Container Registry**
2. Click **Create repository**
3. Set:
   - **Visibility:** Private
   - **Repository name:** `my-app`
   - **Tag immutability:** Disabled (enable later in advanced)
   - **Image scan on push:** Disabled for now
4. Click **Create repository**
5. Note the **URI** shown: `<account-id>.dkr.ecr.<region>.amazonaws.com/my-app`

> **Tip:** The repository URI is what you use in `docker push` and in ECS/EKS task definitions.

#### CLI

```bash
# Create a private repository
aws ecr create-repository \
  --repository-name my-app \
  --region ap-south-1

# List all repositories
aws ecr describe-repositories

# Get repository URI (save this — you'll use it often)
aws ecr describe-repositories \
  --query "repositories[?repositoryName=='my-app'].repositoryUri" \
  --output text
```

---

### Topic 1.2 — Authenticate Docker to ECR

#### Console

> Authentication is CLI-only — the console does not manage Docker credentials. Follow the CLI section.

#### CLI

```bash
# Get an auth token and pipe it to Docker login
# This token is valid for 12 hours
aws ecr get-login-password --region ap-south-1 \
  | docker login \
    --username AWS \
    --password-stdin \
    <account-id>.dkr.ecr.ap-south-1.amazonaws.com

# Expected output: Login Succeeded
```

> **Important:** You must re-authenticate every 12 hours. In CI/CD pipelines this is automated. On EC2 instances with the right IAM role, this works without storing credentials.

---

### Topic 1.3 — Build, Tag, and Push an Image

#### Console

> Build/push is CLI/Docker — you view the result in the console.

1. After pushing (see CLI steps), open ECR → your repo
2. Click **Images** tab — you will see the pushed image with its tag, digest, size, and push date

#### CLI

```bash
# Step 1: Create a simple Dockerfile
mkdir my-app && cd my-app
cat > Dockerfile <<EOF
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EOF

echo "<h1>Hello from ECR + ECS!</h1>" > index.html

# Step 2: Build the image
docker build -t my-app .

# Step 3: Tag for ECR
docker tag my-app:latest \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest

# Step 4: Push to ECR
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest

# Step 5: Verify
aws ecr list-images --repository-name my-app
```

---

### Hands-on Exercise 1 — ECR Basics

> Complete all steps before moving on.

- [ ] Create a repo named `workshop-app` in ECR
- [ ] Write a Dockerfile that serves a custom HTML page via nginx
- [ ] Build, tag, and push the image to ECR
- [ ] Verify the image appears in the ECR console with correct tag and size
- [ ] Pull the same image back: `docker pull <your-ecr-uri>:latest`

---

## Phase 2 — ECR Image Management

### Topic 2.1 — Lifecycle Policies (Auto-delete old images)

#### Console

1. ECR → select your repository → **Lifecycle policies** tab
2. Click **Edit lifecycle policy** → **Add rule**
3. Configure:
   - **Priority:** 1
   - **Rule description:** Delete untagged images older than 7 days
   - **Image status:** Untagged
   - **Match criteria:** Since image pushed — 7 days
4. Add a second rule:
   - **Priority:** 2
   - **Image status:** Tagged
   - **Count more than:** 10 images
5. Click **Save**

#### CLI

```bash
# Create lifecycle policy JSON
cat > lifecycle-policy.json <<EOF
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
      "action": { "type": "expire" }
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
      "action": { "type": "expire" }
    }
  ]
}
EOF

# Apply the policy
aws ecr put-lifecycle-policy \
  --repository-name my-app \
  --lifecycle-policy-text file://lifecycle-policy.json

# Preview what would be deleted (dry run)
aws ecr get-lifecycle-policy-preview \
  --repository-name my-app
```

---

### Topic 2.2 — Image Scanning (CVE / Vulnerability Detection)

#### Console

1. ECR → repository → **Edit** → enable **Scan on push**
2. Push a new image — ECR automatically scans it
3. Click on the image → **Vulnerabilities** tab to see findings
4. Findings are rated: CRITICAL / HIGH / MEDIUM / LOW / INFORMATIONAL

#### CLI

```bash
# Enable scan on push for a repo
aws ecr put-image-scanning-configuration \
  --repository-name my-app \
  --image-scanning-configuration scanOnPush=true

# Manually trigger a scan
aws ecr start-image-scan \
  --repository-name my-app \
  --image-id imageTag=latest

# Get scan results
aws ecr describe-image-scan-findings \
  --repository-name my-app \
  --image-id imageTag=latest \
  --query "imageScanFindings.findings[?severity=='CRITICAL']"
```

> **Tip:** Enable **Amazon Inspector** (enhanced scanning) for deeper scanning including OS and language-level packages. Go to Inspector → enable ECR integration.

---

### Topic 2.3 — IAM Permissions for ECR

#### Console

1. IAM → Roles → Create role → AWS service → EC2 (or ECS)
2. Attach **AmazonEC2ContainerRegistryReadOnly** (for pull-only)
3. Or **AmazonEC2ContainerRegistryFullAccess** (for push + pull)
4. For cross-account access: ECR repo → **Permissions** → Edit JSON policy

#### CLI

```bash
# Allow another account (111122223333) to pull from your repo
cat > repo-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountPull",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": [
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:BatchCheckLayerAvailability"
      ]
    }
  ]
}
EOF

aws ecr set-repository-policy \
  --repository-name my-app \
  --policy-text file://repo-policy.json
```

---

### Hands-on Exercise 2 — ECR Image Management

- [ ] Push 5 images with different version tags (v1.0 through v5.0)
- [ ] Apply a lifecycle policy that keeps only the 3 most recent tagged images
- [ ] Enable scan on push and view vulnerabilities of a public base image (e.g. `python:3.9`)
- [ ] Create a read-only IAM policy for ECR and attach it to a test IAM user

---

---

# PART 2 — Amazon ECS (Elastic Container Service)

> **What is ECS?** AWS's container orchestrator. You define what containers to run and how — ECS handles scheduling, placement, restarts, and scaling.

---

## Phase 3 — ECS Core Concepts

### Topic 3.1 — Understand ECS Architecture

```
Cluster
 └── Service (keeps N tasks running, handles load balancing)
      └── Task (one running group of containers)
           └── Container (the actual Docker container)
```

| Component | Role |
|-----------|------|
| **Cluster** | Logical grouping of compute resources |
| **Task Definition** | Blueprint — which image, CPU, memory, ports, env vars |
| **Task** | One running instance of a task definition |
| **Service** | Keeps a desired number of tasks running; integrates with ALB |
| **Fargate** | Serverless — no EC2 to manage |
| **EC2 launch type** | You manage the EC2 instances in the cluster |

---

### Topic 3.2 — Create a Task Definition

#### Console

1. ECS → **Task definitions** → **Create new task definition**
2. Choose **Fargate** as launch type
3. Configure:
   - **Task definition family:** `my-app-task`
   - **CPU:** 0.5 vCPU, **Memory:** 1 GB
   - **Task role:** (leave empty for now)
   - **Task execution role:** `ecsTaskExecutionRole` (create if not exists)
4. Under **Container** section:
   - **Name:** `my-app`
   - **Image URI:** `<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest`
   - **Port mappings:** 80 (TCP)
   - **Log configuration:** awslogs, log group `/ecs/my-app`
5. Click **Create**

> **Tip — Two IAM Roles in ECS:**
> - **Task Execution Role** — used by ECS agent to pull image and send logs (always `ecsTaskExecutionRole`)
> - **Task Role** — used by your app code inside the container to call AWS APIs (S3, DynamoDB, etc.)

#### CLI

```bash
# Create the task execution role (one-time setup)
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Create a CloudWatch log group
aws logs create-log-group --log-group-name /ecs/my-app

# Register task definition
cat > task-def.json <<EOF
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest",
      "portMappings": [
        { "containerPort": 80, "protocol": "tcp" }
      ],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF

aws ecs register-task-definition \
  --cli-input-json file://task-def.json
```

---

### Topic 3.3 — Create a Cluster and Run a Task

#### Console

1. ECS → **Clusters** → **Create cluster**
2. Name: `my-cluster`, Infrastructure: **AWS Fargate** (serverless)
3. Click **Create**
4. Open cluster → **Tasks** tab → **Run new task**
5. Select:
   - Launch type: Fargate
   - Task definition: `my-app-task`
   - VPC, Subnet, Security group (allow port 80 inbound)
   - Auto-assign public IP: **Enabled**
6. Click **Run task**
7. Wait for status → **RUNNING**, then click on task → note the public IP → open in browser

#### CLI

```bash
# Create cluster
aws ecs create-cluster --cluster-name my-cluster

# Get default VPC and subnet
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text)

# Create a security group
SG_ID=$(aws ec2 create-security-group \
  --group-name ecs-sg \
  --description "ECS security group" \
  --vpc-id $VPC_ID \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# Run a task
aws ecs run-task \
  --cluster my-cluster \
  --task-definition my-app-task \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[$SUBNET_ID],
    securityGroups=[$SG_ID],
    assignPublicIp=ENABLED
  }"

# List running tasks
aws ecs list-tasks --cluster my-cluster

# Describe a task (get public IP)
TASK_ARN=$(aws ecs list-tasks --cluster my-cluster --query "taskArns[0]" --output text)
aws ecs describe-tasks --cluster my-cluster --tasks $TASK_ARN \
  --query "tasks[0].attachments[0].details"
```

---

## Phase 4 — ECS Services & Networking

### Topic 4.1 — ECS Service with Load Balancer

#### Console

1. Create an **Application Load Balancer** (EC2 → Load Balancers → Create)
   - Scheme: Internet-facing, IP type: IPv4
   - VPC + at least 2 subnets in different AZs
   - Listener: HTTP port 80
   - Target group: IP type, port 80, protocol HTTP
2. ECS → cluster → **Services** tab → **Create**
3. Configure:
   - Launch type: Fargate
   - Task definition: `my-app-task`
   - Service name: `my-app-service`
   - Desired count: 2
4. Networking: select VPC, subnets, security group (allow 80)
5. Load balancing: select the ALB and target group created above
6. Click **Create service**
7. Wait for 2 tasks to be RUNNING, then access via ALB DNS name

#### CLI

```bash
# Create target group
TG_ARN=$(aws elbv2 create-target-group \
  --name my-app-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-path "/" \
  --query "TargetGroups[0].TargetGroupArn" \
  --output text)

# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name my-app-alb \
  --subnets $SUBNET_ID $(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "Subnets[1].SubnetId" --output text) \
  --security-groups $SG_ID \
  --query "LoadBalancers[0].LoadBalancerArn" \
  --output text)

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Create ECS service
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-app-service \
  --task-definition my-app-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[$SUBNET_ID],
    securityGroups=[$SG_ID],
    assignPublicIp=ENABLED
  }" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=my-app,containerPort=80"
```

---

### Topic 4.2 — ECS Networking Modes

| Mode | Use with | How it works |
|------|----------|--------------|
| **awsvpc** | Fargate (required), EC2 | Each task gets its own ENI and private IP. Security groups per task. |
| **bridge** | EC2 only | Docker NAT — container port maps to random host port. Default for EC2. |
| **host** | EC2 only | Container shares host network stack. No port mapping needed but no isolation. |

> **Real-world advice:** Always use `awsvpc` for new workloads. It gives you per-task security groups and better VPC-native networking.

---

### Topic 4.3 — Auto Scaling for ECS Services

#### Console

1. ECS → cluster → service → **Update service**
2. Scroll to **Service auto scaling** → enable
3. Set:
   - Minimum tasks: 1
   - Maximum tasks: 10
4. Add scaling policy:
   - Policy type: Target tracking
   - Metric: ECSServiceAverageCPUUtilization
   - Target value: 50%

#### CLI

```bash
# Register the ECS service as a scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-app-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 \
  --max-capacity 10

# Create a target tracking scaling policy
cat > scaling-policy.json <<EOF
{
  "TargetValue": 50.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
  },
  "ScaleInCooldown": 60,
  "ScaleOutCooldown": 60
}
EOF

aws application-autoscaling put-scaling-policy \
  --policy-name cpu-tracking \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-app-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration file://scaling-policy.json
```

---

### Topic 4.4 — ECS Deployment Strategies

| Strategy | How it works | Use case |
|----------|-------------|----------|
| **Rolling update** | Replaces tasks gradually. Default. | Most workloads |
| **Blue/Green (CodeDeploy)** | Runs new version alongside old, shifts traffic when healthy | Zero-downtime deploys |
| **Circuit breaker** | Auto-rollback if new tasks fail to start | Safety net |

#### CLI — Force redeploy with circuit breaker

```bash
# Update service with circuit breaker enabled
aws ecs update-service \
  --cluster my-cluster \
  --service my-app-service \
  --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true}" \
  --force-new-deployment

# Monitor deployment
aws ecs describe-services \
  --cluster my-cluster \
  --services my-app-service \
  --query "services[0].deployments"
```

---

## Phase 5 — ECS Operations & Observability

### Topic 5.1 — Logs with CloudWatch

#### Console

1. CloudWatch → **Log groups** → `/ecs/my-app`
2. Click a log stream (named `ecs/my-app/<task-id>`)
3. Use **Log Insights** for queries:
   ```
   fields @timestamp, @message
   | filter @message like /ERROR/
   | sort @timestamp desc
   | limit 50
   ```

#### CLI

```bash
# List log streams for a task
aws logs describe-log-streams \
  --log-group-name /ecs/my-app \
  --order-by LastEventTime \
  --descending

# Get log events
aws logs get-log-events \
  --log-group-name /ecs/my-app \
  --log-stream-name ecs/my-app/<task-id>

# Tail logs live (requires AWS CLI v2)
aws logs tail /ecs/my-app --follow
```

---

### Topic 5.2 — ECS Exec (Shell into Running Container)

#### Console

> ECS Exec must be enabled via CLI or at service creation. Console cannot directly open a terminal.

#### CLI

```bash
# Enable ECS Exec on service (update service)
aws ecs update-service \
  --cluster my-cluster \
  --service my-app-service \
  --enable-execute-command

# Force new deployment so tasks have exec enabled
aws ecs update-service \
  --cluster my-cluster \
  --service my-app-service \
  --force-new-deployment

# Get a task ARN
TASK_ARN=$(aws ecs list-tasks \
  --cluster my-cluster \
  --service-name my-app-service \
  --query "taskArns[0]" --output text)

# Open a shell
aws ecs execute-command \
  --cluster my-cluster \
  --task $TASK_ARN \
  --container my-app \
  --interactive \
  --command "/bin/sh"
```

> **Requirement:** Task execution role needs `ssmmessages:*` permissions + SSM Agent must be in the container (it's pre-installed in AWS base images).

---

### Topic 5.3 — Secrets Management in ECS

#### Console

1. AWS Secrets Manager → **Store a new secret** → Other type of secret
2. Key: `DB_PASSWORD`, Value: `mypassword123`
3. Name: `/myapp/prod/db-password`
4. In ECS task definition → container → **Environment variables** section:
   - Type: **ValueFrom**
   - Key: `DB_PASSWORD`
   - Value: ARN of your secret

#### CLI

```bash
# Store a secret
aws secretsmanager create-secret \
  --name /myapp/prod/db-password \
  --secret-string "mypassword123"

# Get the ARN
SECRET_ARN=$(aws secretsmanager describe-secret \
  --secret-id /myapp/prod/db-password \
  --query ARN --output text)

# Add to task definition containerDefinitions:
# "secrets": [
#   {
#     "name": "DB_PASSWORD",
#     "valueFrom": "<secret-arn>"
#   }
# ]

# Grant task execution role access to the secret
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite
```

---

### Hands-on Exercise 3 — Full ECS Deployment

- [ ] Push your `my-app` image to ECR with tag `v1.0`
- [ ] Create a task definition with environment variable `APP_ENV=production`
- [ ] Create an ECS service with 2 tasks behind an ALB
- [ ] Access your app via the ALB DNS name in a browser
- [ ] Update your HTML, push as `v2.0`, update task definition, and trigger a rolling deploy
- [ ] Use ECS Exec to shell into a running container and inspect env vars with `env`
- [ ] Enable auto scaling and stress the service to trigger a scale-out

---

---

# PART 3 — Amazon EKS (Elastic Kubernetes Service)

> **What is EKS?** AWS's managed Kubernetes service. More powerful and flexible than ECS but requires Kubernetes knowledge. Use EKS when you need Kubernetes-native features, multi-cloud portability, or a rich ecosystem of tools.

> **Before starting EKS:** Make sure you understand these Kubernetes concepts: Pod, Deployment, Service, Namespace, ConfigMap, Secret, Ingress, ServiceAccount, RBAC.

---

## Phase 6 — EKS Fundamentals

### Topic 6.1 — Kubernetes Core Objects (Quick Reference)

| Object | Purpose | Example |
|--------|---------|---------|
| **Pod** | Smallest deployable unit — runs 1+ containers | `kubectl get pods` |
| **Deployment** | Manages replica pods, rolling updates | `kubectl apply -f deploy.yaml` |
| **Service** | Stable network endpoint for pods | ClusterIP / LoadBalancer |
| **Namespace** | Virtual cluster for isolation | `kubectl get ns` |
| **ConfigMap** | Non-sensitive configuration | App config, flags |
| **Secret** | Sensitive data (base64 encoded) | Passwords, tokens |
| **Ingress** | HTTP routing rules (path/host-based) | Routes / to app-a, /api to app-b |
| **ServiceAccount** | Identity for pods to call AWS APIs | Used with IRSA |

---

### Topic 6.2 — Create an EKS Cluster

#### Console

1. EKS → **Add cluster** → **Create**
2. Configure cluster:
   - Name: `my-eks-cluster`
   - Kubernetes version: latest stable (e.g. 1.30)
   - Cluster service role: create `eksClusterRole` with `AmazonEKSClusterPolicy`
3. Networking:
   - VPC: default VPC
   - Subnets: select all
   - Security group: leave default
   - Cluster endpoint access: **Public**
4. Logging: enable API server and Audit (recommended)
5. Click **Create** — takes 10–15 minutes
6. After creation: **Compute** tab → **Add node group**
   - Name: `workers`
   - Node IAM role: create with `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`
   - Instance type: `t3.medium`
   - Desired / Min / Max: 2 / 1 / 4

#### CLI (using eksctl — recommended)

```bash
# Create cluster + node group in one command
eksctl create cluster \
  --name my-eks-cluster \
  --region ap-south-1 \
  --version 1.30 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# This takes ~15-20 minutes. eksctl creates:
# - VPC, subnets, security groups
# - EKS control plane
# - Managed node group
# - Updates your ~/.kube/config automatically

# Verify cluster is up
kubectl get nodes
kubectl get pods --all-namespaces
```

---

### Topic 6.3 — Configure kubectl to Connect to EKS

#### Console

> After cluster creation, the console shows a **Connect** button with the exact CLI command to run.

#### CLI

```bash
# Update kubeconfig (adds/merges cluster into ~/.kube/config)
aws eks update-kubeconfig \
  --name my-eks-cluster \
  --region ap-south-1

# Verify connectivity
kubectl cluster-info
kubectl get nodes -o wide    # Should show nodes in Ready state

# Check current context
kubectl config current-context
kubectl config get-contexts   # List all clusters in kubeconfig
```

---

### Topic 6.4 — Node Types in EKS

| Node Type | Description | Best for |
|-----------|-------------|----------|
| **Managed node groups** | AWS manages OS patching, draining, and termination. | Most workloads |
| **Self-managed nodes** | You manage the EC2 ASG and lifecycle. | Custom AMIs, special hardware |
| **Fargate profiles** | Serverless — no nodes. Pod gets dedicated compute. | Bursty, stateless workloads |

#### CLI — Add a Fargate profile

```bash
# Create a Fargate profile for a specific namespace
eksctl create fargateprofile \
  --cluster my-eks-cluster \
  --region ap-south-1 \
  --name fp-default \
  --namespace fargate-ns

# Create the namespace
kubectl create namespace fargate-ns

# Deploy a pod — it will run on Fargate automatically
kubectl run nginx --image=nginx -n fargate-ns
kubectl get pods -n fargate-ns -o wide   # Node name will start with fargate-
```

---

## Phase 7 — Workloads & Networking

### Topic 7.1 — Deploy an Application

#### Console

1. EKS → cluster → **Resources** tab → **Workloads** → **Create**
2. Paste YAML and click **Create**

#### CLI

```bash
# Create a Deployment
cat > deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
EOF

kubectl apply -f deployment.yaml

# Expose with a LoadBalancer Service (creates an NLB in AWS)
cat > service.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
EOF

kubectl apply -f service.yaml

# Watch deployment rollout
kubectl rollout status deployment/my-app

# Get the external DNS of the NLB
kubectl get service my-app-service

# Update image and roll out new version
kubectl set image deployment/my-app my-app=<ecr-uri>:v2.0
kubectl rollout status deployment/my-app

# Rollback if something goes wrong
kubectl rollout undo deployment/my-app
kubectl rollout history deployment/my-app
```

---

### Topic 7.2 — AWS Load Balancer Controller & Ingress

> The AWS Load Balancer Controller creates ALBs from Kubernetes Ingress resources. Install it once per cluster.

#### CLI — Install AWS Load Balancer Controller

```bash
# Step 1: Create OIDC provider for the cluster
eksctl utils associate-iam-oidc-provider \
  --region ap-south-1 \
  --cluster my-eks-cluster \
  --approve

# Step 2: Create IAM policy for the controller
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Step 3: Create service account with IAM role
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Step 4: Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify
kubectl get deployment -n kube-system aws-load-balancer-controller
```

#### CLI — Create Ingress resource

```bash
cat > ingress.yaml <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
EOF

kubectl apply -f ingress.yaml
kubectl get ingress my-app-ingress   # Wait for ADDRESS to populate (ALB DNS)
```

---

### Topic 7.3 — IRSA: IAM Roles for Service Accounts

> IRSA lets pods assume an IAM role to access AWS services (S3, DynamoDB, etc.) — without storing credentials.

#### Console

1. EKS → cluster → **Configuration** → verify OIDC provider is associated
2. IAM → **Roles** → **Create role** → Web identity
3. Identity provider: your cluster's OIDC URL
4. Audience: `sts.amazonaws.com`
5. Attach required policies (e.g. `AmazonS3ReadOnlyAccess`)
6. In **Trust policy**, scope to your service account:
   ```json
   "StringEquals": {
     "oidc.eks.ap-south-1.amazonaws.com/id/<OIDC_ID>:sub":
       "system:serviceaccount:<namespace>:<service-account-name>"
   }
   ```

#### CLI

```bash
# Step 1: Get OIDC issuer
OIDC_URL=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --query "cluster.identity.oidc.issuer" \
  --output text)

# Step 2: Create service account with IAM role using eksctl (easiest way)
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace default \
  --name s3-reader \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# Step 3: Use the service account in a pod
cat > pod-with-irsa.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: s3-test
spec:
  serviceAccountName: s3-reader
  containers:
  - name: aws-cli
    image: amazon/aws-cli:latest
    command: ["aws", "s3", "ls"]
EOF

kubectl apply -f pod-with-irsa.yaml
kubectl logs s3-test   # Should list your S3 buckets
```

---

### Topic 7.4 — Horizontal Pod Autoscaler & Cluster Autoscaler

#### CLI — HPA (scale pods)

```bash
# Install Metrics Server (required for HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify
kubectl top nodes
kubectl top pods

# Create HPA targeting 50% CPU
kubectl autoscale deployment my-app \
  --cpu-percent=50 \
  --min=2 \
  --max=10

kubectl get hpa

# Simulate load to trigger scaling
kubectl run load-test --image=busybox --restart=Never -- \
  sh -c "while true; do wget -q -O- http://my-app-service; done"
```

#### CLI — Cluster Autoscaler (scale nodes)

```bash
# Apply Cluster Autoscaler deployment
# Download and edit the manifest to set your cluster name
curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Edit: replace <YOUR CLUSTER NAME> with my-eks-cluster
sed -i 's/<YOUR CLUSTER NAME>/my-eks-cluster/g' cluster-autoscaler-autodiscover.yaml

kubectl apply -f cluster-autoscaler-autodiscover.yaml

# Monitor cluster autoscaler logs
kubectl logs -n kube-system -l app=cluster-autoscaler -f
```

---

## Phase 8 — EKS Security & Operations

### Topic 8.1 — EKS Authentication & RBAC

#### Console

1. EKS → cluster → **Access** tab → **IAM access entries** → **Create access entry**
2. Enter IAM user/role ARN
3. Select Kubernetes groups (e.g. `system:masters` for admin)

#### CLI

```bash
# Old method: edit aws-auth ConfigMap
kubectl edit configmap aws-auth -n kube-system
# Add under mapRoles:
# - rolearn: arn:aws:iam::<account-id>:role/MyRole
#   username: my-user
#   groups:
#     - system:masters

# New method (EKS access entries)
aws eks create-access-entry \
  --cluster-name my-eks-cluster \
  --principal-arn arn:aws:iam::<account-id>:user/dev-user \
  --kubernetes-groups developers

# Create RBAC role for developers namespace
cat > rbac.yaml <<EOF
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

kubectl apply -f rbac.yaml
```

---

### Topic 8.2 — EKS Logging & Monitoring

#### Console

1. EKS → cluster → **Logging** → **Manage logging** → enable:
   - API server, Audit, Authenticator, Controller manager, Scheduler
2. CloudWatch → Log groups → `/aws/eks/my-eks-cluster/cluster`

#### CLI

```bash
# Enable control plane logging
aws eks update-cluster-config \
  --name my-eks-cluster \
  --logging '{"clusterLogging":[{"types":["api","audit","authenticator","controllerManager","scheduler"],"enabled":true}]}'

# Install CloudWatch Container Insights
ClusterName=my-eks-cluster
RegionName=ap-south-1

curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml \
  | sed "s/{{cluster_name}}/$ClusterName/;s/{{region_name}}/$RegionName/" \
  | kubectl apply -f -

# Node and pod metrics will appear in CloudWatch under ContainerInsights namespace
```

---

### Topic 8.3 — EKS Managed Add-ons

#### Console

1. EKS → cluster → **Add-ons** → **Get more add-ons**
2. Common add-ons to install: vpc-cni, CoreDNS, kube-proxy, Amazon EBS CSI driver, Pod Identity Agent

#### CLI

```bash
# List available add-ons
aws eks describe-addon-versions --kubernetes-version 1.30 \
  --query "addons[].addonName" --output table

# Install EBS CSI driver (enables PersistentVolumes)
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --resolve-conflicts OVERWRITE

# Check add-on status
aws eks describe-addon \
  --cluster-name my-eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --query "addon.status"

# List installed add-ons
aws eks list-addons --cluster-name my-eks-cluster
```

---

### Topic 8.4 — EKS Version Upgrades

> Upgrade one minor version at a time. Example: 1.28 → 1.29 → 1.30.

#### Console

1. EKS → cluster → **Configuration** → **Update now** (if upgrade available)
2. After control plane upgrade: **Compute** → node group → **Update now**
3. After node group: **Add-ons** → update each add-on

#### CLI

```bash
# Step 1: Check for upgrade availability
aws eks describe-update --cluster-name my-eks-cluster

# Step 2: Upgrade control plane
aws eks update-cluster-version \
  --name my-eks-cluster \
  --kubernetes-version 1.30

# Wait for upgrade to complete
aws eks describe-cluster \
  --name my-eks-cluster \
  --query "cluster.status"

# Step 3: Upgrade managed node group
aws eks update-nodegroup-version \
  --cluster-name my-eks-cluster \
  --nodegroup-name workers

# Step 4: Upgrade add-ons
aws eks update-addon \
  --cluster-name my-eks-cluster \
  --addon-name vpc-cni \
  --resolve-conflicts OVERWRITE
```

---

### Hands-on Exercise 4 — Full EKS Deployment

- [ ] Create an EKS cluster with a managed node group (2 nodes)
- [ ] Deploy your ECR image as a Kubernetes Deployment with 3 replicas
- [ ] Expose it via an ALB using Ingress (requires AWS Load Balancer Controller)
- [ ] Create a Kubernetes Secret for a DB password and mount it as env var
- [ ] Configure HPA to scale on CPU > 50%, trigger it with a load generator
- [ ] Create a service account with IRSA permissions to list S3 buckets and verify from inside a pod
- [ ] Enable CloudWatch Container Insights and view pod metrics

---

---

# PART 4 — Advanced Topics & Real-world Patterns

## Topic 9.1 — CI/CD Pipeline with ECR + ECS

```
Developer push → GitHub → GitHub Actions → Build image → Push to ECR → Update ECS service
```

#### GitHub Actions Workflow

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
  CONTAINER_NAME: my-app

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Configure AWS credentials (OIDC — no keys needed)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::<account-id>:role/github-actions-role
        aws-region: ${{ env.AWS_REGION }}

    - name: Login to ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: Build, tag, push image to ECR
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ github.sha }}
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

    - name: Download task definition
      run: |
        aws ecs describe-task-definition --task-definition my-app-task \
          --query taskDefinition > task-definition.json

    - name: Update ECS task definition with new image
      id: task-def
      uses: aws-actions/amazon-ecs-render-task-definition@v1
      with:
        task-definition: task-definition.json
        container-name: ${{ env.CONTAINER_NAME }}
        image: ${{ steps.build-image.outputs.image }}

    - name: Deploy to ECS
      uses: aws-actions/amazon-ecs-deploy-task-definition@v1
      with:
        task-definition: ${{ steps.task-def.outputs.task-definition }}
        service: ${{ env.ECS_SERVICE }}
        cluster: ${{ env.ECS_CLUSTER }}
        wait-for-service-stability: true
```

---

## Topic 9.2 — ECS Capacity Providers & Spot Instances

#### CLI

```bash
# Create a capacity provider using Fargate Spot (60-90% cheaper)
aws ecs put-cluster-capacity-providers \
  --cluster my-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE_SPOT,weight=4 \
    capacityProvider=FARGATE,weight=1,base=1

# The above runs 80% on Spot, 20% on regular Fargate
# base=1 guarantees at least 1 regular Fargate task always runs

# Handle Spot interruption: add SIGTERM handler in your app
# ECS sends SIGTERM 120 seconds before termination on Spot
```

> **Cost tip:** Use FARGATE_SPOT for non-critical/stateless workloads. Keep your code stateless and handle SIGTERM gracefully.

---

## Topic 9.3 — Karpenter (Modern Node Autoscaler for EKS)

> Karpenter replaces the Cluster Autoscaler. It provisions right-sized nodes in seconds.

```bash
# Install Karpenter (simplified)
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --name karpenter \
  --namespace karpenter \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/KarpenterControllerPolicy \
  --approve

helm install karpenter oci://public.ecr.aws/karpenter/karpenter \
  --namespace karpenter \
  --create-namespace \
  --set settings.clusterName=my-eks-cluster

# Create a NodePool (replaces Cluster Autoscaler's node group config)
cat > nodepool.yaml <<EOF
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
EOF

kubectl apply -f nodepool.yaml
```

---

## Topic 9.4 — Multi-container Task Patterns in ECS

```json
// Sidecar pattern in ECS task definition
{
  "containerDefinitions": [
    {
      "name": "app",
      "image": "<ecr-uri>/my-app:latest",
      "portMappings": [{"containerPort": 8080}],
      "essential": true
    },
    {
      "name": "log-router",
      "image": "amazon/aws-for-fluent-bit:latest",
      "essential": false,
      "dependsOn": [{"containerName": "app", "condition": "START"}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/firelens",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

---
---

# PART 5 — ECR Advanced Topics

---

## Topic 10.1 — ECR Pull Through Cache

> Pull Through Cache lets ECR automatically cache images from public registries (Docker Hub, Quay, GitHub Container Registry) into your private ECR. Your workloads always pull from ECR — no direct internet dependency.

#### Console

1. ECR → **Pull through cache** → **Add rule**
2. Select upstream registry: **Docker Hub**
3. Prefix: `dockerhub`
4. ECR creates a repo automatically on first pull

#### CLI

```bash
# Create pull-through cache rule for Docker Hub
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix dockerhub \
  --upstream-registry-url registry-1.docker.io \
  --region ap-south-1

# Create pull-through cache rule for Public ECR
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix ecr-public \
  --upstream-registry-url public.ecr.aws

# Now pull via ECR instead of Docker Hub:
# Instead of: docker pull nginx:alpine
# Use:        docker pull <account-id>.dkr.ecr.ap-south-1.amazonaws.com/dockerhub/nginx:alpine
# ECR fetches from Docker Hub on first pull, caches it in your account

# List existing pull-through cache rules
aws ecr describe-pull-through-cache-rules
```

> **Real-world use case:** Docker Hub rate-limits unauthenticated pulls (100/6h). Using ECR pull-through cache avoids rate limiting and speeds up builds by pulling from your local AWS region.

---

## Topic 10.2 — Tag Immutability

> Tag immutability prevents overwriting an existing image tag. Once `v1.0` is pushed, no one can push a different image with the same `v1.0` tag. Critical for production safety.

#### Console

1. ECR → repository → **Edit**
2. **Tag immutability:** Enable
3. Save

#### CLI

```bash
# Enable tag immutability on an existing repo
aws ecr put-image-tag-mutability \
  --repository-name my-app \
  --image-tag-mutability IMMUTABLE

# Create a new repo with immutability enabled from the start
aws ecr create-repository \
  --repository-name prod-app \
  --image-tag-mutability IMMUTABLE

# Attempting to push a duplicate tag will now fail with:
# ImageTagAlreadyExistsException
```

> **Best practice:** Use `IMMUTABLE` for production repositories. Use semantic versioning (`v1.2.3`) or git SHA as tags. Never use `latest` as the only tag in production — it makes rollbacks impossible.

---

## Topic 10.3 — KMS Encryption for ECR

> By default, ECR encrypts images at rest using AES-256 (AWS-managed key). You can bring your own KMS key for compliance requirements.

#### Console

1. ECR → **Create repository**
2. Encryption settings → **Customer-managed key**
3. Select your KMS key (must be in same region)

#### CLI

```bash
# Create a KMS key
KMS_KEY_ARN=$(aws kms create-key \
  --description "ECR encryption key" \
  --query KeyMetadata.Arn --output text)

# Create ECR repo with KMS encryption
aws ecr create-repository \
  --repository-name secure-app \
  --encryption-configuration encryptionType=KMS,kmsKey=$KMS_KEY_ARN

# Verify encryption config
aws ecr describe-repositories \
  --repository-names secure-app \
  --query "repositories[0].encryptionConfiguration"
```

---

## Topic 10.4 — VPC Endpoints for ECR (PrivateLink)

> Without VPC endpoints, your ECS/EKS nodes must go to the internet to pull images from ECR. VPC endpoints keep all traffic inside AWS private network — required for private clusters.

#### Console

1. VPC → **Endpoints** → **Create endpoint**
2. Create three endpoints (all required for ECR):
   - `com.amazonaws.ap-south-1.ecr.api` — ECR API calls
   - `com.amazonaws.ap-south-1.ecr.dkr` — Docker image layer pulls
   - `com.amazonaws.ap-south-1.s3` — ECR stores layers in S3 (Gateway endpoint)
3. Select your VPC and private subnets
4. Attach security group allowing HTTPS (443) from your containers

#### CLI

```bash
# Get VPC and subnet info
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text)

# Create security group for endpoints
EP_SG=$(aws ec2 create-security-group \
  --group-name vpce-sg \
  --description "VPC Endpoint SG" \
  --vpc-id $VPC_ID \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $EP_SG --protocol tcp --port 443 --cidr 10.0.0.0/8

# Create ECR API endpoint (Interface type)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.ecr.api \
  --vpc-endpoint-type Interface \
  --subnet-ids $(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "Subnets[0].SubnetId" --output text) \
  --security-group-ids $EP_SG \
  --private-dns-enabled

# Create ECR Docker endpoint (Interface type)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids $(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "Subnets[0].SubnetId" --output text) \
  --security-group-ids $EP_SG \
  --private-dns-enabled

# Create S3 Gateway endpoint (free — always create this)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.ap-south-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids $(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "RouteTables[0].RouteTableId" --output text)
```

---

## Topic 10.5 — Docker Multi-Stage Builds (Smaller, Safer Images)

> Multi-stage builds produce small production images by separating build-time tools from runtime. A Go binary built in a 1GB image can be shipped in a 10MB image.

#### CLI

```bash
# Multi-stage Dockerfile example (Node.js app)
cat > Dockerfile <<'EOF'
# ---- Stage 1: Build ----
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# ---- Stage 2: Production image ----
FROM node:20-alpine AS production
WORKDIR /app

# Security: run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

COPY --from=builder /app/node_modules ./node_modules
COPY --chown=appuser:appgroup src/ ./src/

EXPOSE 3000
CMD ["node", "src/index.js"]
EOF

# Build — only the production stage is in the final image
docker build -t my-node-app:latest .
docker image inspect my-node-app:latest --format='{{.Size}}' | numfmt --to=iec

# Compare sizes: without multi-stage vs with
# Without: ~1.2 GB   With: ~150 MB
```

> **Security wins from this Dockerfile:**
> - Non-root user (`appuser`) — limits damage if app is compromised
> - Alpine base — minimal attack surface
> - `npm ci` instead of `npm install` — deterministic, reproducible builds
> - Build tools (compilers, dev deps) never reach production

---
---

# PART 6 — ECS Advanced Topics

---

## Topic 11.1 — ECS with EC2 Launch Type (Container Instances)

> When you need GPU support, custom AMIs, specific instance types, or spot EC2 savings — use EC2 launch type instead of Fargate.

#### Console

1. ECS → cluster → **Infrastructure** tab → **Add EC2 instances**
2. Create an Auto Scaling Group with ECS-optimized AMI
3. The ECS agent on the instance automatically registers it with the cluster

#### CLI

```bash
# Step 1: Create an ECS-optimized AMI launch template
# Get the latest ECS-optimized AMI for your region
ECS_AMI=$(aws ssm get-parameters \
  --names /aws/service/ecs/optimized-ami/amazon-linux-2/recommended/image_id \
  --query "Parameters[0].Value" --output text)

echo "ECS AMI: $ECS_AMI"

# Step 2: Create an IAM instance profile for EC2 container instances
aws iam create-role \
  --role-name ecsInstanceRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name ecsInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role

aws iam create-instance-profile --instance-profile-name ecsInstanceProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name ecsInstanceProfile \
  --role-name ecsInstanceRole

# Step 3: Launch EC2 instance that joins the cluster
# The user-data script tells the ECS agent which cluster to join
aws ec2 run-instances \
  --image-id $ECS_AMI \
  --instance-type t3.medium \
  --iam-instance-profile Name=ecsInstanceProfile \
  --user-data "#!/bin/bash
echo ECS_CLUSTER=my-cluster >> /etc/ecs/ecs.config" \
  --count 2

# Step 4: Register task definition for EC2 launch type
# Note: networkMode "bridge" is default for EC2
cat > ec2-task-def.json <<EOF
{
  "family": "ec2-app-task",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest",
      "memory": 512,
      "cpu": 256,
      "portMappings": [{"containerPort": 80, "hostPort": 0}],
      "essential": true
    }
  ]
}
EOF

aws ecs register-task-definition --cli-input-json file://ec2-task-def.json

# List EC2 container instances in a cluster
aws ecs list-container-instances --cluster my-cluster
aws ecs describe-container-instances \
  --cluster my-cluster \
  --container-instances $(aws ecs list-container-instances \
    --cluster my-cluster --query "containerInstanceArns[0]" --output text)
```

---

## Topic 11.2 — Task Placement Strategies & Constraints

> Control where ECS places tasks across your EC2 container instances.

#### Console

1. ECS → service → **Edit** → **Task placement** section
2. Choose strategy: Spread / Binpack / Random

#### CLI

```bash
# Strategy 1: SPREAD — distribute tasks evenly across AZs (high availability)
aws ecs create-service \
  --cluster my-cluster \
  --service-name ha-service \
  --task-definition my-task \
  --desired-count 4 \
  --launch-type EC2 \
  --placement-strategy '[
    {"type": "spread", "field": "attribute:ecs.availability-zone"},
    {"type": "spread", "field": "instanceId"}
  ]'

# Strategy 2: BINPACK — pack tasks tightly to minimize EC2 instances used (cost savings)
aws ecs create-service \
  --cluster my-cluster \
  --service-name binpack-service \
  --task-definition my-task \
  --desired-count 4 \
  --launch-type EC2 \
  --placement-strategy '[
    {"type": "binpack", "field": "cpu"}
  ]'

# Placement CONSTRAINT — only place on specific instance types
aws ecs create-service \
  --cluster my-cluster \
  --service-name constrained-service \
  --task-definition my-task \
  --desired-count 2 \
  --launch-type EC2 \
  --placement-constraints '[
    {"type": "memberOf", "expression": "attribute:ecs.instance-type =~ t3.*"}
  ]' \
  --placement-strategy '[
    {"type": "spread", "field": "instanceId"}
  ]'

# Constraint: one task per instance (distinctInstance)
aws ecs create-service \
  --cluster my-cluster \
  --service-name one-per-instance \
  --task-definition my-task \
  --desired-count 2 \
  --launch-type EC2 \
  --placement-constraints '[{"type": "distinctInstance"}]'
```

---

## Topic 11.3 — ECS Service Connect

> ECS Service Connect is the modern (2022+) replacement for Cloud Map service discovery. It gives services a stable DNS name and mutual TLS within a namespace — without managing a sidecar yourself.

#### Console

1. ECS → cluster → **Namespaces** tab → **Create namespace**
2. When creating/updating a service → enable **Service Connect**
3. Set port alias (e.g. `http` on port 80)
4. Other services discover this one via `http://<service-name>:80`

#### CLI

```bash
# Step 1: Create a namespace (cluster-level)
aws ecs create-cluster \
  --cluster-name connected-cluster \
  --service-connect-defaults namespace=myapp

# Step 2: Create a backend service with Service Connect
aws ecs create-service \
  --cluster connected-cluster \
  --service-name backend \
  --task-definition backend-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID]}" \
  --service-connect-configuration '{
    "enabled": true,
    "namespace": "myapp",
    "services": [
      {
        "portName": "http",
        "clientAliases": [{"port": 80, "dnsName": "backend"}]
      }
    ]
  }'

# Step 3: Frontend service can now reach backend at http://backend:80
# No more managing Cloud Map namespaces or DNS records manually
```

---

## Topic 11.4 — EFS Volumes in ECS (Persistent Storage)

> ECS tasks are stateless by default. For persistent data (uploads, databases, shared config), mount an EFS (Elastic File System) volume.

#### Console

1. Create EFS filesystem: EFS → **Create file system** → select VPC
2. ECS task definition → **Storage** section → **Add volume**
3. Volume type: EFS, select your filesystem
4. Mount point: `/data`

#### CLI

```bash
# Step 1: Create EFS filesystem
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose \
  --throughput-mode bursting \
  --query FileSystemId --output text)

# Step 2: Create mount target in your subnet
aws efs create-mount-target \
  --file-system-id $EFS_ID \
  --subnet-id $SUBNET_ID \
  --security-groups $SG_ID

# Wait for mount target to become available
aws efs describe-mount-targets --file-system-id $EFS_ID

# Step 3: Task definition with EFS volume
cat > efs-task-def.json <<EOF
{
  "family": "efs-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
  "volumes": [
    {
      "name": "efs-storage",
      "efsVolumeConfiguration": {
        "fileSystemId": "$EFS_ID",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED",
        "authorizationConfig": {"iam": "ENABLED"}
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest",
      "portMappings": [{"containerPort": 80}],
      "essential": true,
      "mountPoints": [
        {
          "sourceVolume": "efs-storage",
          "containerPath": "/data",
          "readOnly": false
        }
      ]
    }
  ]
}
EOF

aws ecs register-task-definition --cli-input-json file://efs-task-def.json
```

---

## Topic 11.5 — ECS Scheduled Tasks (Cron Jobs)

> Run ECS tasks on a schedule using EventBridge (formerly CloudWatch Events). Use for batch jobs, report generation, database backups.

#### Console

1. EventBridge → **Rules** → **Create rule**
2. Event source: **Schedule** → cron or rate expression
3. Target: **ECS task**
4. Select cluster, task definition, VPC/subnet/security group

#### CLI

```bash
# Step 1: Create an IAM role for EventBridge to run ECS tasks
aws iam create-role \
  --role-name ecsEventsRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "events.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name ecsEventsRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceEventsRole

EVENTS_ROLE_ARN=$(aws iam get-role --role-name ecsEventsRole \
  --query Role.Arn --output text)

# Step 2: Create scheduled rule (runs every day at 2 AM UTC)
aws events put-rule \
  --name daily-batch-job \
  --schedule-expression "cron(0 2 * * ? *)" \
  --state ENABLED \
  --description "Daily batch job at 2 AM UTC"

# Step 3: Add ECS task as the target
aws events put-targets \
  --rule daily-batch-job \
  --targets '[
    {
      "Id": "ecs-batch-target",
      "Arn": "arn:aws:ecs:ap-south-1:<account-id>:cluster/my-cluster",
      "RoleArn": "'$EVENTS_ROLE_ARN'",
      "EcsParameters": {
        "TaskDefinitionArn": "arn:aws:ecs:ap-south-1:<account-id>:task-definition/my-task:1",
        "LaunchType": "FARGATE",
        "NetworkConfiguration": {
          "awsvpcConfiguration": {
            "Subnets": ["'$SUBNET_ID'"],
            "SecurityGroups": ["'$SG_ID'"],
            "AssignPublicIp": "ENABLED"
          }
        }
      }
    }
  ]'

# Useful cron expressions
# "cron(0 2 * * ? *)"         — Every day at 2 AM UTC
# "cron(0 9 ? * MON-FRI *)"   — Every weekday at 9 AM UTC
# "cron(0 0 1 * ? *)"         — First day of every month
# "rate(5 minutes)"            — Every 5 minutes
```

---

## Topic 11.6 — Container Health Checks in ECS

> Health checks determine if a container is healthy. ECS will replace unhealthy tasks automatically.

#### Console

1. Task definition → container → **Health check** section
2. Command: `CMD-SHELL, curl -f http://localhost/health || exit 1`
3. Interval: 30s, Timeout: 5s, Retries: 3, Start period: 60s

#### CLI

```bash
# Health check defined in task definition containerDefinitions
# Add this to your container definition JSON:
cat > healthcheck-task.json <<'EOF'
{
  "family": "healthcheck-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest",
      "portMappings": [{"containerPort": 80}],
      "essential": true,
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
EOF

aws ecs register-task-definition --cli-input-json file://healthcheck-task.json

# Check task health status
TASK_ARN=$(aws ecs list-tasks --cluster my-cluster --query "taskArns[0]" --output text)
aws ecs describe-tasks --cluster my-cluster --tasks $TASK_ARN \
  --query "tasks[0].containers[0].healthStatus"
```

| Field | Meaning |
|-------|---------|
| `interval` | How often to run health check (seconds) |
| `timeout` | How long to wait for response before marking as failed |
| `retries` | How many consecutive failures before marking UNHEALTHY |
| `startPeriod` | Grace period after container starts before health checks count |

---

## Topic 11.7 — Blue/Green Deployment with CodeDeploy + ECS

> Blue/Green deployment runs the new version alongside the old one. Traffic shifts only after the new version is healthy. Supports instant rollback.

#### Console

1. CodeDeploy → **Applications** → **Create application** → compute platform: **Amazon ECS**
2. Create a deployment group:
   - ECS cluster + service
   - Two target groups (blue + green)
   - ALB listener
3. Deploy: CodeDeploy → deployment → **Create deployment**

#### CLI

```bash
# Step 1: Create two target groups (blue and green)
BLUE_TG=$(aws elbv2 create-target-group \
  --name blue-tg --protocol HTTP --port 80 \
  --vpc-id $VPC_ID --target-type ip \
  --query "TargetGroups[0].TargetGroupArn" --output text)

GREEN_TG=$(aws elbv2 create-target-group \
  --name green-tg --protocol HTTP --port 80 \
  --vpc-id $VPC_ID --target-type ip \
  --query "TargetGroups[0].TargetGroupArn" --output text)

# Step 2: Create service with CODE_DEPLOY controller
aws ecs create-service \
  --cluster my-cluster \
  --service-name bg-service \
  --task-definition my-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --deployment-controller '{"type": "CODE_DEPLOY"}' \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID]}" \
  --load-balancers "targetGroupArn=$BLUE_TG,containerName=my-app,containerPort=80"

# Step 3: Create CodeDeploy application and deployment group
aws deploy create-application \
  --application-name my-ecs-app \
  --compute-platform ECS

aws deploy create-deployment-group \
  --application-name my-ecs-app \
  --deployment-group-name my-dg \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --service-role-arn arn:aws:iam::<account-id>:role/CodeDeployRole \
  --ecs-services clusterName=my-cluster,serviceName=bg-service \
  --load-balancer-info "targetGroupPairInfoList=[{
    targetGroups: [{name: blue-tg},{name: green-tg}],
    prodTrafficRoute: {listenerArns: [<alb-listener-arn>]}
  }]"

# Step 4: Trigger a deployment (provide appspec.yaml in S3 or inline)
cat > appspec.yaml <<EOF
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: "arn:aws:ecs:ap-south-1:<account-id>:task-definition/my-task:2"
        LoadBalancerInfo:
          ContainerName: "my-app"
          ContainerPort: 80
Hooks:
  - BeforeAllowTraffic: "arn:aws:lambda:ap-south-1:<account-id>:function:health-check"
EOF
```

---
---

# PART 7 — EKS Advanced Topics

---

## Topic 12.1 — ConfigMaps & Secrets in Kubernetes

> ConfigMaps store non-sensitive config. Secrets store sensitive data (base64-encoded). Both can be injected as environment variables or mounted as files.

#### CLI

```bash
# ---- ConfigMaps ----

# Create from literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info

# Create from a file
echo "server.port=8080" > app.properties
kubectl create configmap app-config-file --from-file=app.properties

# View
kubectl describe configmap app-config
kubectl get configmap app-config -o yaml

# Use ConfigMap in a pod as env vars
cat > pod-configmap.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
  - name: app
    image: nginx:alpine
    envFrom:
    - configMapRef:
        name: app-config
    # Or individual keys:
    env:
    - name: MY_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    # Mount as file:
    volumeMounts:
    - name: config-vol
      mountPath: /etc/config
  volumes:
  - name: config-vol
    configMap:
      name: app-config-file
EOF

kubectl apply -f pod-configmap.yaml

# ---- Secrets ----

# Create a secret (base64-encoded automatically)
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=mypassword123

# Create from files (e.g. TLS certs)
kubectl create secret tls my-tls \
  --cert=tls.crt \
  --key=tls.key

# Use secret in a pod
cat > pod-secret.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  containers:
  - name: app
    image: nginx:alpine
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
    volumeMounts:
    - name: secret-vol
      mountPath: /etc/secrets
      readOnly: true
  volumes:
  - name: secret-vol
    secret:
      secretName: db-credentials
EOF

kubectl apply -f pod-secret.yaml
kubectl exec -it secret-demo -- cat /etc/secrets/password
```

> **Security tip:** Kubernetes secrets are base64-encoded, NOT encrypted by default in etcd. Enable **Envelope Encryption** with KMS for EKS to encrypt secrets at rest:
> ```bash
> aws eks create-cluster --name my-cluster \
>   --encryption-config '[{"provider":{"keyArn":"<kms-key-arn>"},"resources":["secrets"]}]' ...
> ```

---

## Topic 12.2 — Persistent Volumes: EBS & EFS on EKS

> Pods are ephemeral — when they restart, data is lost. Persistent Volumes (PV) keep data alive across pod restarts.

#### CLI — EBS CSI (block storage, single pod)

```bash
# Install EBS CSI driver add-on (if not done)
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name aws-ebs-csi-driver

# Create a StorageClass for gp3 EBS volumes
cat > ebs-storageclass.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
parameters:
  type: gp3
  encrypted: "true"
EOF

kubectl apply -f ebs-storageclass.yaml

# Create a PersistentVolumeClaim
cat > ebs-pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-ebs-pvc
spec:
  accessModes:
  - ReadWriteOnce        # EBS = single node only
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 10Gi
EOF

kubectl apply -f ebs-pvc.yaml

# Mount in a pod
cat > ebs-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: ebs-test
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: ebs-vol
      mountPath: /data
  volumes:
  - name: ebs-vol
    persistentVolumeClaim:
      claimName: my-ebs-pvc
EOF

kubectl apply -f ebs-pod.yaml
kubectl exec -it ebs-test -- sh -c "echo 'persistent!' > /data/test.txt"
```

#### CLI — EFS CSI (shared file storage, multiple pods)

```bash
# EFS = ReadWriteMany — multiple pods can mount simultaneously

# Install EFS CSI driver
helm repo add aws-efs-csi-driver https://kubernetes-sigs.github.io/aws-efs-csi-driver/
helm install aws-efs-csi-driver aws-efs-csi-driver/aws-efs-csi-driver \
  -n kube-system

# Create EFS filesystem (see Topic 11.4 for CLI)
# Create StorageClass pointing to your EFS
cat > efs-storageclass.yaml <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-sc
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: $EFS_ID
  directoryPerms: "700"
EOF

kubectl apply -f efs-storageclass.yaml

# PVC with ReadWriteMany
cat > efs-pvc.yaml <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-pvc
spec:
  accessModes:
  - ReadWriteMany         # EFS = multiple nodes/pods can mount
  storageClassName: efs-sc
  resources:
    requests:
      storage: 5Gi
EOF

kubectl apply -f efs-pvc.yaml
```

| Feature | EBS | EFS |
|---------|-----|-----|
| Access mode | ReadWriteOnce (1 node) | ReadWriteMany (many nodes) |
| Type | Block storage | Network file system |
| Performance | Very fast, low latency | Slower, higher latency |
| Use case | Databases, single-pod storage | Shared content, ML datasets |

---

## Topic 12.3 — StatefulSets (for Databases and Stateful Apps)

> StatefulSets manage pods that need stable identity and persistent storage — databases, Kafka, Zookeeper, Redis clusters.

#### CLI

```bash
cat > statefulset.yaml <<'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:          # Each pod gets its own PVC automatically
  - metadata:
      name: postgres-data
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: ebs-sc
      resources:
        requests:
          storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None               # Headless service — pods get DNS: postgres-0.postgres
  selector:
    app: postgres
  ports:
  - port: 5432
EOF

kubectl apply -f statefulset.yaml
kubectl get statefulset
kubectl get pods -l app=postgres   # Pods are named postgres-0, postgres-1...

# StatefulSet pods start/stop in order
# Delete postgres-0 — EKS recreates it with the same PVC and same name
kubectl delete pod postgres-0
```

---

## Topic 12.4 — DaemonSets

> DaemonSets ensure one pod runs on every node. Used for log collectors, monitoring agents, security scanners, network plugins.

#### CLI

```bash
# Example: deploy a log collector (Fluent Bit) on every node
cat > daemonset.yaml <<'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentbit
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: fluentbit
  template:
    metadata:
      labels:
        app: fluentbit
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentbit
        image: amazon/aws-for-fluent-bit:latest
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 100Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
EOF

kubectl apply -f daemonset.yaml

# Verify one pod per node
kubectl get daemonset -n kube-system fluentbit
kubectl get pods -n kube-system -l app=fluentbit -o wide
```

---

## Topic 12.5 — Jobs and CronJobs

> Jobs run a task to completion (not forever). CronJobs run jobs on a schedule.

#### CLI

```bash
# ---- One-time Job ----
cat > job.yaml <<'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
spec:
  backoffLimit: 3               # Retry up to 3 times on failure
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: migrator
        image: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
        command: ["python", "manage.py", "migrate"]
        env:
        - name: DB_HOST
          value: postgres.default.svc.cluster.local
EOF

kubectl apply -f job.yaml
kubectl get jobs
kubectl logs -l job-name=db-migration

# ---- CronJob ----
cat > cronjob.yaml <<'EOF'
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 2 * * *"        # Every day at 2 AM UTC
  concurrencyPolicy: Forbid    # Don't run if previous is still running
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: reporter
            image: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
            command: ["python", "generate_report.py"]
EOF

kubectl apply -f cronjob.yaml
kubectl get cronjobs

# Manually trigger a CronJob now (for testing)
kubectl create job --from=cronjob/daily-report manual-run-$(date +%s)
```

---

## Topic 12.6 — Liveness, Readiness & Startup Probes

> Probes let Kubernetes know if your app is alive and ready to serve traffic.

| Probe | What it does | Failure action |
|-------|-------------|----------------|
| **Liveness** | Is the app still running? (deadlock, crash) | Restart the container |
| **Readiness** | Is the app ready to receive traffic? | Remove from Service endpoints |
| **Startup** | Is the app still starting up? (slow apps) | Don't check liveness/readiness yet |

#### CLI

```bash
cat > probes-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probed-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: probed-app
  template:
    metadata:
      labels:
        app: probed-app
    spec:
      containers:
      - name: app
        image: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest
        ports:
        - containerPort: 80
        # Startup probe: app has 60s to start before liveness kicks in
        startupProbe:
          httpGet:
            path: /health
            port: 80
          failureThreshold: 12
          periodSeconds: 5
        # Liveness probe: restart if app stops responding
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
        # Readiness probe: remove from load balancer until ready
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 3
EOF

kubectl apply -f probes-deployment.yaml

# Watch probe status
kubectl describe pod $(kubectl get pods -l app=probed-app -o name | head -1)
```

---

## Topic 12.7 — Taints, Tolerations & Node Affinity

> Control which pods run on which nodes. Useful for dedicated GPU nodes, spot instances, or compliance isolation.

#### CLI

```bash
# ---- Taints ----
# Add a taint to a node: only pods that tolerate it will run here
kubectl taint nodes node-1 dedicated=gpu:NoSchedule

# Pod with matching toleration can run on this node
cat > gpu-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  tolerations:
  - key: dedicated
    value: gpu
    operator: Equal
    effect: NoSchedule
  containers:
  - name: trainer
    image: my-gpu-image:latest
EOF

# Remove a taint
kubectl taint nodes node-1 dedicated=gpu:NoSchedule-

# ---- Node Affinity ----
# Schedule pods only on specific node types (soft preference or hard requirement)
cat > affinity-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: affinity-demo
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:     # HARD rule
        nodeSelectorTerms:
        - matchExpressions:
          - key: karpenter.sh/capacity-type
            operator: In
            values: [on-demand]
      preferredDuringSchedulingIgnoredDuringExecution:    # SOFT preference
      - weight: 80
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: [ap-south-1a]
  containers:
  - name: app
    image: nginx:alpine
EOF

# Label a node (so affinity rules can match it)
kubectl label node <node-name> environment=production

# ---- Pod Anti-Affinity (spread pods across AZs) ----
cat > spread-deploy.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spread-app
  template:
    metadata:
      labels:
        app: spread-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: spread-app
            topologyKey: topology.kubernetes.io/zone
      containers:
      - name: app
        image: nginx:alpine
EOF
```

---

## Topic 12.8 — Network Policies (Pod-Level Firewall)

> By default all pods can talk to each other. Network Policies are like security groups for pods.

#### CLI

```bash
# Deny all ingress by default (then allow selectively)
cat > deny-all.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}         # Applies to all pods in namespace
  policyTypes:
  - Ingress
EOF

kubectl apply -f deny-all.yaml

# Allow only frontend pods to reach backend on port 8080
cat > allow-frontend.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
EOF

kubectl apply -f allow-frontend.yaml

# Allow traffic from a specific namespace
cat > allow-namespace.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-namespace
spec:
  podSelector:
    matchLabels:
      app: my-app
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
EOF
```

> **Note:** Network Policies require a CNI that supports them. EKS with VPC CNI supports Network Policies from version 1.14+. Enable it:
> ```bash
> kubectl set env daemonset aws-node -n kube-system ENABLE_NETWORK_POLICY_CONTROLLER=true
> ```

---

## Topic 12.9 — Pod Disruption Budgets (PDB)

> PDBs protect your app during voluntary disruptions (node upgrades, cluster autoscaler scale-down). They guarantee a minimum number of healthy pods.

#### CLI

```bash
# Guarantee at least 2 pods are always available
cat > pdb.yaml <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 2               # OR use maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
EOF

kubectl apply -f pdb.yaml
kubectl get pdb

# PDB in action during node drain:
# kubectl drain node-1 --ignore-daemonsets
# Kubernetes will wait for the PDB to allow eviction
# Will evict one pod, wait for replacement, then evict next
```

---

## Topic 12.10 — Namespaces & Resource Quotas

> Namespaces provide isolation between teams or environments. Resource Quotas prevent one team from consuming all cluster resources.

#### CLI

```bash
# Create namespaces for different environments/teams
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod

# Label namespaces for network policies and affinity
kubectl label namespace prod environment=production

# Set resource quotas per namespace
cat > resource-quota.yaml <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
    persistentvolumeclaims: "5"
EOF

kubectl apply -f resource-quota.yaml
kubectl describe resourcequota dev-quota -n dev

# Set default resource limits for all pods in a namespace (LimitRange)
cat > limitrange.yaml <<'EOF'
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:          # Default limit if not specified
      cpu: 500m
      memory: 256Mi
    defaultRequest:   # Default request if not specified
      cpu: 100m
      memory: 128Mi
    max:              # Maximum allowed
      cpu: "2"
      memory: 1Gi
EOF

kubectl apply -f limitrange.yaml

# Work in a specific namespace
kubectl get pods -n dev
kubectl run nginx --image=nginx -n dev
kubectl config set-context --current --namespace=dev   # Switch default namespace
```

---

## Topic 12.11 — EKS Pod Identity (New IRSA Alternative)

> EKS Pod Identity (2023) is the simpler replacement for IRSA. No need to configure OIDC trust policies manually — just associate a role to a service account.

#### Console

1. EKS → cluster → **Access** tab → **Pod Identity associations** → **Create**
2. Select namespace, service account name, and IAM role
3. Install Pod Identity Agent add-on if not present

#### CLI

```bash
# Step 1: Install Pod Identity Agent add-on
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name eks-pod-identity-agent

# Step 2: Create IAM role with pod identity trust policy
cat > pod-identity-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "pods.eks.amazonaws.com"
    },
    "Action": ["sts:AssumeRole", "sts:TagSession"]
  }]
}
EOF

aws iam create-role \
  --role-name my-pod-role \
  --assume-role-policy-document file://pod-identity-trust.json

aws iam attach-role-policy \
  --role-name my-pod-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Step 3: Create Pod Identity Association (no OIDC setup needed!)
aws eks create-pod-identity-association \
  --cluster-name my-eks-cluster \
  --namespace default \
  --service-account my-sa \
  --role-arn arn:aws:iam::<account-id>:role/my-pod-role

# Step 4: Create service account and pod
kubectl create serviceaccount my-sa
kubectl run test-pod \
  --image=amazon/aws-cli:latest \
  --serviceaccount=my-sa \
  -- aws s3 ls

kubectl logs test-pod
```

---
---

# PART 8 — Security, Observability & Cost

---

## Topic 13.1 — Container Security Best Practices

#### CLI

```bash
# Secure Dockerfile
cat > Dockerfile.secure <<'EOF'
FROM python:3.12-alpine AS builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM python:3.12-alpine
WORKDIR /app

# Run as non-root
RUN adduser -D -u 1001 appuser
USER appuser

# Copy only what's needed
COPY --from=builder /usr/local/lib/python3.12/site-packages /usr/local/lib/python3.12/site-packages
COPY --chown=appuser:appuser src/ ./src/

# Read-only root filesystem
VOLUME ["/tmp", "/var/log"]

EXPOSE 8080
CMD ["python", "src/main.py"]
EOF

# Kubernetes security context
cat > secure-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1001
    fsGroup: 1001
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: <ecr-uri>/my-app:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: [ALL]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
EOF
```

#### Security Checklist

| Control | ECS | EKS |
|---------|-----|-----|
| Non-root container | `user` in Dockerfile | `securityContext.runAsNonRoot: true` |
| Read-only filesystem | `readonlyRootFilesystem: true` in task def | `readOnlyRootFilesystem: true` |
| Drop capabilities | Not applicable | `capabilities.drop: [ALL]` |
| No privilege escalation | Not applicable | `allowPrivilegeEscalation: false` |
| Limit network access | Security groups on tasks | Network Policies |
| Secrets not in env | Use Secrets Manager with `valueFrom` | Use k8s Secrets or External Secrets Operator |
| Image scanning | ECR scan on push | ECR scan + Amazon Inspector |

---

## Topic 13.2 — AWS X-Ray Distributed Tracing

> X-Ray traces requests across multiple containers/services — invaluable for debugging latency in microservices.

#### ECS with X-Ray Sidecar

```json
// Add X-Ray daemon as a sidecar in your task definition
{
  "containerDefinitions": [
    {
      "name": "my-app",
      "image": "<ecr-uri>/my-app:latest",
      "essential": true,
      "environment": [
        {"name": "AWS_XRAY_DAEMON_ADDRESS", "value": "127.0.0.1:2000"}
      ]
    },
    {
      "name": "xray-daemon",
      "image": "amazon/aws-xray-daemon:latest",
      "essential": false,
      "portMappings": [{"containerPort": 2000, "protocol": "udp"}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/xray",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "xray"
        }
      }
    }
  ]
}
```

#### EKS with X-Ray (as DaemonSet)

```bash
# Deploy X-Ray daemon as DaemonSet
kubectl apply -f https://eksworkshop.com/x-ray/daemonset.files/xray-k8s-daemonset.yaml

# App code sends traces to UDP 2000 on the node IP:
# XRAY_DAEMON_ADDRESS = $(HOST_IP):2000
# Or using ADOT (AWS Distro for OpenTelemetry) - preferred for new workloads

# Install ADOT add-on
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name adot
```

---

## Topic 13.3 — Prometheus & Grafana on EKS

> The standard observability stack for Kubernetes. Prometheus collects metrics; Grafana visualizes them.

#### CLI

```bash
# Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.service.type=LoadBalancer \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=ebs-sc \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi

# Get Grafana admin password
kubectl get secret kube-prometheus-grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode

# Get Grafana URL
kubectl get svc kube-prometheus-grafana -n monitoring

# Useful Grafana dashboard IDs to import:
# 315  — Kubernetes cluster monitoring
# 6417 — Kubernetes pods
# 3119 — Node exporter
# 13770 — AWS EKS

# Check Prometheus targets
kubectl port-forward svc/kube-prometheus-kube-prome-prometheus -n monitoring 9090:9090
# Open http://localhost:9090/targets
```

---

## Topic 13.4 — ECS vs EKS — When to Choose What

| Factor | Choose ECS | Choose EKS |
|--------|-----------|-----------|
| **Team Kubernetes experience** | Low / none | Medium to high |
| **Workload complexity** | Simple to moderate microservices | Complex microservices, stateful apps |
| **Ecosystem & tooling** | AWS-native tools only | Huge CNCF ecosystem (Helm, Argo, Istio...) |
| **Setup time** | Minutes (Fargate) | 20–60 minutes + add-ons |
| **Operations overhead** | Very low (especially Fargate) | Higher (upgrades, add-ons, CNI) |
| **Multi-cloud / portability** | AWS-only | Portable (same YAML runs on GKE, AKS) |
| **Windows containers** | Yes | Yes (Windows node groups) |
| **GPU workloads** | EC2 launch type | GPU node groups |
| **Service mesh** | ECS Service Connect | Istio, Linkerd, AWS App Mesh |
| **Cost (small scale)** | Fargate is cost-effective | EC2 spot nodes can be cheaper at scale |
| **Custom schedulers** | No | Yes (full k8s scheduler extensibility) |

> **Rule of thumb:** Start with ECS Fargate. Migrate to EKS when you need Kubernetes-native features, have teams with k8s experience, or need cross-cloud portability.

---

## Topic 13.5 — Cost Optimization Strategies

#### ECR Costs

```bash
# ECR charges for storage (~$0.10/GB/month) — keep repos clean
# Apply lifecycle policies to all repos (see Topic 2.1)

# List repos with no lifecycle policy (should be fixed)
aws ecr describe-repositories --query "repositories[].repositoryName" --output text | \
  xargs -I {} sh -c 'policy=$(aws ecr get-lifecycle-policy --repository-name {} 2>/dev/null); \
    [ -z "$policy" ] && echo "No lifecycle policy: {}"'
```

#### ECS Costs

```bash
# 1. Use FARGATE_SPOT for non-critical workloads (60-90% savings)
# See Topic 9.2 for Capacity Provider setup

# 2. Right-size tasks — find over-provisioned tasks
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=my-app-service Name=ClusterName,Value=my-cluster \
  --start-time $(date -u -d '7 days ago' '+%Y-%m-%dT%H:%M:%S') \
  --end-time $(date -u '+%Y-%m-%dT%H:%M:%S') \
  --period 86400 --statistics Average

# If average CPU < 20% consistently — halve your CPU allocation in task definition
```

#### EKS Costs

```bash
# 1. Use Spot instances for node groups (60-90% savings)
eksctl create nodegroup \
  --cluster my-eks-cluster \
  --name spot-workers \
  --instance-types t3.medium,t3.large,t3a.medium \
  --spot \
  --nodes-min 1 --nodes-max 10

# 2. Use Karpenter consolidation to remove underused nodes
# See Topic 9.3 — consolidationPolicy: WhenUnderutilized

# 3. Set resource requests accurately (not too high — wastes node capacity)
# Use VPA (Vertical Pod Autoscaler) for recommendations
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install vpa fairwinds-stable/vpa --namespace vpa --create-namespace

# VPA shows recommended requests/limits
kubectl get vpa

# 4. Delete unused EKS clusters (they charge ~$0.10/hour for control plane)
eksctl delete cluster --name my-eks-cluster
```

---

## Topic 13.6 — AWS Copilot (High-level ECS Tool)

> AWS Copilot abstracts ECS complexity. One command deploys a full environment with ECS, ALB, ECR, VPC, IAM — everything.

#### CLI

```bash
# Install Copilot
curl -Lo copilot https://github.com/aws/copilot-cli/releases/latest/download/copilot-linux
chmod +x copilot && sudo mv copilot /usr/local/bin/

# Initialize a new app
copilot app init my-webapp

# Deploy a Load Balanced Web Service (creates ECS + ALB automatically)
copilot init \
  --app my-webapp \
  --name frontend \
  --type "Load Balanced Web Service" \
  --dockerfile ./Dockerfile \
  --port 80

# Deploy to a new environment
copilot env init --name prod --profile default --app my-webapp
copilot deploy --name frontend --env prod

# Tail logs
copilot svc logs --name frontend --env prod --follow

# Scale
copilot svc deploy --name frontend --env prod

# Delete everything (ECS service, ALB, ECR, VPC)
copilot app delete my-webapp
```

> **Best for:** Teams new to ECS who want a working production setup in minutes without writing CloudFormation or Terraform.

---

---
---

# PART 9 — ECR: Multi-Arch, Caching, Signing & Replication

---

## Topic 14.1 — Multi-Architecture Image Builds (ARM64 + AMD64)

> AWS Graviton (ARM) instances cost ~20% less than x86. Building multi-arch images lets the same ECR image run on both Fargate ARM and standard x86 — chosen automatically by the platform at pull time.

#### CLI

```bash
# Step 1: Create and use a multi-platform buildx builder
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# Step 2: Build and push for both AMD64 (standard) and ARM64 (Graviton)
# This creates a single manifest list in ECR covering both platforms
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0 \
  --push .

# Step 3: Verify the manifest list
docker buildx imagetools inspect \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0
# Shows: linux/amd64 digest + linux/arm64 digest

# Step 4: Use ARM64 on Fargate — add to task definition JSON:
# "runtimePlatform": {
#   "operatingSystemFamily": "LINUX",
#   "cpuArchitecture": "ARM64"
# }
```

> **Cost tip:** Fargate Graviton (ARM64) is ~20% cheaper and often faster. If your Dockerfile has no architecture-specific instructions, multi-arch builds require zero app changes.

---

## Topic 14.2 — Docker Layer Caching (Faster CI Builds)

> Without caching, every `docker build` rebuilds all layers from scratch. Layer caching in ECR cuts CI build times from minutes to seconds.

#### CLI

```bash
# Build with ECR as remote cache backend
docker buildx build \
  --cache-from type=registry,ref=<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:cache \
  --cache-to   type=registry,ref=<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:cache,mode=max \
  --tag <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest \
  --push .

# In GitHub Actions workflow:
# - name: Build and push with cache
#   run: |
#     docker buildx build \
#       --cache-from type=registry,ref=$ECR_REGISTRY/$ECR_REPO:cache \
#       --cache-to   type=registry,ref=$ECR_REGISTRY/$ECR_REPO:cache,mode=max \
#       --tag $ECR_REGISTRY/$ECR_REPO:$GITHUB_SHA \
#       --push .

# Optimise Dockerfile layer order for maximum cache reuse:
# BAD  — cache busted on every code change:
#   COPY . .
#   RUN pip install -r requirements.txt

# GOOD — dependencies layer cached separately from code:
#   COPY requirements.txt .
#   RUN pip install -r requirements.txt   ← cached unless requirements.txt changes
#   COPY . .                              ← only this layer rebuilds on code changes
```

---

## Topic 14.3 — Image Signing & Supply Chain Security

> Image signing proves an image was built by your trusted pipeline — not tampered with. Required for compliance in regulated industries (HIPAA, PCI-DSS, FedRAMP).

#### CLI

```bash
# --- Method 1: AWS Signer + Notation (AWS-native) ---

# Install Notation CLI
curl -Lo notation.tar.gz \
  https://github.com/notaryproject/notation/releases/latest/download/notation_linux_amd64.tar.gz
tar xzf notation.tar.gz && sudo mv notation /usr/local/bin/

# Create a signing profile in AWS Signer
aws signer put-signing-profile \
  --profile-name ecr-signing-profile \
  --platform-id Notation-OCI-SHA384-ECDSA \
  --signature-validity-period value=12,type=MONTHS

PROFILE_ARN=$(aws signer describe-signing-profile \
  --profile-name ecr-signing-profile \
  --query signingProfileVersionArn --output text)

# Sign an image in ECR
notation sign \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0 \
  --plugin com.amazonaws.signer.notation.plugin \
  --id $PROFILE_ARN

# Verify a signature
notation verify \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0

# --- Method 2: Cosign + KMS (CNCF, popular in EKS) ---
curl -Lo cosign \
  https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign && sudo mv cosign /usr/local/bin/

cosign sign --key awskms:///alias/ecr-signing-key \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0

cosign verify --key awskms:///alias/ecr-signing-key \
  <account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:v1.0
```

---

## Topic 14.4 — ECR Replication (Cross-Region & Cross-Account)

> Replicate images to other regions for lower-latency pulls or DR, and to other accounts for multi-account setups.

#### Console

1. ECR → **Private registry** → **Replication** → **Edit** → **Add rule**
2. Set destination region(s) — e.g. `us-east-1`, `eu-west-1`
3. Optionally filter by repo name prefix
4. Save — new pushes replicate automatically

#### CLI

```bash
# Cross-region replication (same account, filtered by prefix)
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{
      "destinations": [
        {"region": "us-east-1", "registryId": "<account-id>"},
        {"region": "eu-west-1", "registryId": "<account-id>"}
      ],
      "repositoryFilters": [
        {"filter": "prod-", "filterType": "PREFIX_MATCH"}
      ]
    }]
  }'

# Cross-account replication — Step 1: destination account grants permission
aws ecr put-registry-policy \
  --policy-text '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::<SOURCE-ACCOUNT>:root"},
      "Action": ["ecr:CreateRepository", "ecr:ReplicateImage"],
      "Resource": "arn:aws:ecr:ap-south-1:<DEST-ACCOUNT>:repository/*"
    }]
  }'

# Step 2: source account adds cross-account destination
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules": [{"destinations": [
      {"region": "ap-south-1", "registryId": "<DEST-ACCOUNT>"}
    ]}]
  }'

# Verify
aws ecr describe-registry --query replicationConfiguration
```

---
---

# PART 10 — ECS: FireLens, Anywhere, CDK & SSM

---

## Topic 15.1 — FireLens (Advanced Log Routing)

> FireLens is ECS's log routing layer powered by Fluent Bit. Instead of only CloudWatch, route logs to S3, Kinesis, Elasticsearch, Datadog, or Splunk — from a single sidecar, no app changes.

#### CLI

```bash
cat > firelens-task.json <<'EOF'
{
  "family": "firelens-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512", "memory": "1024",
  "executionRoleArn": "arn:aws:iam::<account-id>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "log-router",
      "image": "amazon/aws-for-fluent-bit:latest",
      "essential": true,
      "firelensConfiguration": {
        "type": "fluentbit",
        "options": {"enable-ecs-log-metadata": "true"}
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/firelens-router",
          "awslogs-region": "ap-south-1",
          "awslogs-stream-prefix": "router"
        }
      }
    },
    {
      "name": "my-app",
      "image": "<account-id>.dkr.ecr.ap-south-1.amazonaws.com/my-app:latest",
      "essential": true,
      "portMappings": [{"containerPort": 80}],
      "dependsOn": [{"containerName": "log-router", "condition": "START"}],
      "logConfiguration": {
        "logDriver": "awsfirelens",
        "options": {
          "Name": "cloudwatch_logs",
          "region": "ap-south-1",
          "log_group_name": "/ecs/my-app",
          "log_stream_prefix": "app/",
          "auto_create_group": "true"
        }
      }
    }
  ]
}
EOF

aws ecs register-task-definition --cli-input-json file://firelens-task.json
```

> **When to use:** Multiple log destinations, metadata enrichment, sensitive field redaction, or log platforms other than CloudWatch.

---

## Topic 15.2 — ECS Anywhere (Run Containers On-Premises)

> Run ECS tasks on your own servers while managing them from the AWS console. Same task definitions and APIs — no AWS compute charges for the on-prem server.

#### Console

1. ECS → Create cluster → enable **External instances using ECS Anywhere**
2. Cluster → **Infrastructure** → **Register external instances**
3. Run the generated script on your on-prem server

#### CLI

```bash
# Generate SSM activation for on-prem server registration
aws ssm create-activation \
  --iam-role AmazonEC2RunCommandRoleForManagedInstances \
  --registration-limit 5

# On your on-premises Linux server, run:
# curl -o /tmp/ecs-anywhere-install.sh \
#   https://amazon-ecs-agent-packages.s3.amazonaws.com/ecs-anywhere-install-latest.sh
# sudo bash /tmp/ecs-anywhere-install.sh \
#   --region ap-south-1 \
#   --cluster hybrid-cluster \
#   --activation-id <ActivationId> \
#   --activation-code <ActivationCode>

# Verify the external instance registered
aws ecs list-container-instances \
  --cluster hybrid-cluster \
  --filter "attribute:ecs.os-type==linux"

# Run task on the on-prem instance
aws ecs run-task \
  --cluster hybrid-cluster \
  --task-definition my-app-task \
  --launch-type EXTERNAL
```

---

## Topic 15.3 — Infrastructure as Code for ECS (AWS CDK)

> Define your entire ECS + ALB + VPC stack in Python/TypeScript. CDK synthesises CloudFormation — reproducible, version-controlled, reviewable.

#### CLI

```bash
# Install CDK and bootstrap
npm install -g aws-cdk
cdk bootstrap aws://<account-id>/ap-south-1

# Create CDK project
mkdir ecs-cdk && cd ecs-cdk
cdk init app --language python
source .venv/bin/activate
pip install aws-cdk-lib constructs

# ecs_cdk/ecs_stack.py — full ECS Fargate service + ALB in ~30 lines
cat > ecs_cdk/ecs_stack.py <<'EOF'
from aws_cdk import Stack, aws_ec2 as ec2, aws_ecs as ecs, aws_ecs_patterns as ecs_patterns, aws_ecr as ecr
from constructs import Construct

class EcsStack(Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)
        vpc     = ec2.Vpc(self, "AppVpc", max_azs=2)
        cluster = ecs.Cluster(self, "AppCluster", vpc=vpc)
        repo    = ecr.Repository.from_repository_name(self, "AppRepo", repository_name="my-app")
        ecs_patterns.ApplicationLoadBalancedFargateService(
            self, "AppService",
            cluster=cluster, cpu=512, memory_limit_mib=1024, desired_count=2,
            task_image_options=ecs_patterns.ApplicationLoadBalancedTaskImageOptions(
                image=ecs.ContainerImage.from_ecr_repository(repo, tag="latest"),
                container_port=80,
            ),
            public_load_balancer=True,
        )
EOF

cdk synth    # Preview CloudFormation
cdk diff     # Show planned changes
cdk deploy   # Deploy everything
cdk destroy  # Tear it all down
```

---

## Topic 15.4 — SSM Parameter Store vs Secrets Manager

| Feature | SSM Parameter Store | Secrets Manager |
|---------|--------------------|-----------------|
| **Cost** | Free (standard) | $0.40/secret/month |
| **Auto Rotation** | No | Yes (Lambda-based) |
| **Encryption** | Optional KMS | Always KMS |
| **Best for** | Config, feature flags | DB passwords, API keys, certs |
| **Max size** | 4 KB (standard) | 65 KB |

#### CLI — SSM in ECS Task Definitions

```bash
# Store values
aws ssm put-parameter --name /myapp/prod/db-host \
  --value "mydb.rds.amazonaws.com" --type String
aws ssm put-parameter --name /myapp/prod/api-key \
  --value "sk-prod-abc123" --type SecureString --key-id alias/aws/ssm

# Reference in task definition "secrets" array (same as Secrets Manager)
# "secrets": [
#   {"name": "DB_HOST",  "valueFrom": "arn:aws:ssm:ap-south-1:<account>:parameter/myapp/prod/db-host"},
#   {"name": "API_KEY",  "valueFrom": "arn:aws:ssm:ap-south-1:<account>:parameter/myapp/prod/api-key"}
# ]

# Bulk-get all params for an app
aws ssm get-parameters-by-path \
  --path /myapp/prod/ \
  --with-decryption \
  --query "Parameters[*].{Name:Name,Value:Value}"
```

---
---

# PART 11 — EKS: Helm, GitOps, VPA, External Secrets, Istio & Terraform

---

## Topic 16.1 — Helm Charts (Kubernetes Package Manager)

> Helm is the standard way to install and manage Kubernetes applications. Think of it as `apt` for Kubernetes.

#### CLI

```bash
# Add popular repos
helm repo add bitnami        https://charts.bitnami.com/bitnami
helm repo add ingress-nginx  https://kubernetes.github.io/ingress-nginx
helm repo add cert-manager   https://charts.jetstack.io
helm repo update

# Search + install
helm search repo bitnami/postgresql
helm install my-postgres bitnami/postgresql \
  --set auth.postgresPassword=secret123 \
  --set primary.persistence.size=20Gi

# Use a values file (recommended for production)
cat > postgres-values.yaml <<'EOF'
auth:
  postgresPassword: secret123
  username: appuser
  password: apppassword
  database: appdb
primary:
  persistence:
    enabled: true
    size: 20Gi
    storageClass: ebs-sc
EOF

helm install my-postgres bitnami/postgresql -f postgres-values.yaml

# Upgrade / rollback / history
helm upgrade   my-postgres bitnami/postgresql -f postgres-values.yaml
helm rollback  my-postgres 1
helm history   my-postgres
helm uninstall my-postgres

# Create your own chart
helm create my-app           # Scaffold chart structure
helm package my-app          # Package as .tgz

# Push chart to ECR (OCI format)
aws ecr create-repository --repository-name helm-charts
helm push my-app-0.1.0.tgz \
  oci://<account-id>.dkr.ecr.ap-south-1.amazonaws.com/helm-charts

# Install from ECR
helm install my-app \
  oci://<account-id>.dkr.ecr.ap-south-1.amazonaws.com/helm-charts/my-app \
  --version 0.1.0
```

---

## Topic 16.2 — GitOps with ArgoCD

> ArgoCD watches a Git repo and automatically syncs changes to your EKS cluster. Git is the single source of truth — no manual `kubectl apply` needed.

#### CLI

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Expose UI
kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'

# Get admin password
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 --decode

# Install ArgoCD CLI and login
curl -sSL -o argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/
argocd login <IP> --username admin --password <password> --insecure

# Create an Application pointing at your Git repo
cat > argocd-app.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/k8s-manifests
    targetRevision: main
    path: apps/my-app
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true         # delete resources removed from Git
      selfHeal: true      # revert manual kubectl changes
    syncOptions:
    - CreateNamespace=true
EOF

kubectl apply -f argocd-app.yaml

# Manage releases
argocd app get    my-app
argocd app sync   my-app
argocd app history my-app
argocd app rollback my-app <id>
```

---

## Topic 16.3 — Vertical Pod Autoscaler (VPA)

> HPA adds more pods. VPA adjusts CPU/memory of existing pods — right-sizing without manual guesswork.

#### CLI

```bash
# Install VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler && ./hack/vpa-up.sh

# Create VPA for a deployment
cat > vpa.yaml <<'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"       # Off = recommend only (safe) | Auto = apply + restart
  resourcePolicy:
    containerPolicies:
    - containerName: my-app
      minAllowed: {cpu: 50m, memory: 64Mi}
      maxAllowed:  {cpu: "2", memory: 2Gi}
EOF

kubectl apply -f vpa.yaml

# View recommendations after 30+ minutes of traffic
kubectl describe vpa my-app-vpa
# Target: cpu: 200m, memory: 256Mi  ← use this in your Deployment
```

---

## Topic 16.4 — External Secrets Operator

> Syncs AWS Secrets Manager or SSM Parameter Store secrets into Kubernetes Secrets automatically — with rotation support. No plaintext secrets anywhere in Git.

#### CLI

```bash
# Install
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets --create-namespace --set installCRDs=true

# IRSA for the operator
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --name external-secrets-sa \
  --namespace external-secrets \
  --attach-policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --approve

# SecretStore — connects operator to AWS
cat > secretstore.yaml <<'EOF'
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-store
  namespace: default
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
EOF

kubectl apply -f secretstore.yaml

# ExternalSecret — maps AWS secret → k8s Secret
cat > external-secret.yaml <<'EOF'
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  target:
    name: db-credentials
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: /myapp/prod/db-password
EOF

kubectl apply -f external-secret.yaml
kubectl get externalsecret db-credentials   # Check sync status
kubectl get secret db-credentials           # Verify k8s Secret exists
```

---

## Topic 16.5 — Policy Enforcement with Kyverno

> Kyverno validates, mutates, and generates Kubernetes resources. Enforce org standards automatically — no custom webhook code needed.

#### CLI

```bash
# Install Kyverno
helm repo add kyverno https://kyverno.github.io/kyverno/
helm install kyverno kyverno/kyverno --namespace kyverno --create-namespace

# Policy 1: Require CPU + memory limits on all containers
cat > require-limits.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-container-resources
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "All containers must have CPU and memory limits."
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
EOF

kubectl apply -f require-limits.yaml
kubectl run bad-pod --image=nginx    # Rejected — no resource limits

# Policy 2: Disallow root containers
cat > disallow-root.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-root-user
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-runAsNonRoot
    match:
      any:
      - resources:
          kinds: [Pod]
    validate:
      message: "Containers must not run as root."
      pattern:
        spec:
          securityContext:
            runAsNonRoot: true
EOF

kubectl apply -f disallow-root.yaml

# View policy violations
kubectl get policyreport --all-namespaces
```

---

## Topic 16.6 — Service Mesh with Istio

> Istio handles mTLS, traffic shifting, retries, circuit breaking, and distributed tracing — without touching app code.

#### CLI

```bash
# Install Istio
curl -L https://istio.io/downloadIstio | sh -
sudo mv istio-*/bin/istioctl /usr/local/bin/
istioctl install --set profile=demo -y

# Enable sidecar injection (Envoy proxy added to all pods automatically)
kubectl label namespace default istio-injection=enabled

# Install observability addons (Kiali, Grafana, Jaeger)
kubectl apply -f istio-*/samples/addons/
istioctl dashboard kiali

# Traffic shifting — 90% v1, 10% v2 (canary)
cat > virtual-service.yaml <<'EOF'
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts: [my-app]
  http:
  - route:
    - destination: {host: my-app, subset: v1}
      weight: 90
    - destination: {host: my-app, subset: v2}
      weight: 10
---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-app
spec:
  host: my-app
  subsets:
  - {name: v1, labels: {version: v1}}
  - {name: v2, labels: {version: v2}}
EOF

kubectl apply -f virtual-service.yaml

# Enforce mutual TLS on all pod-to-pod communication
cat > mtls.yaml <<'EOF'
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: default
spec:
  mtls:
    mode: STRICT
EOF

kubectl apply -f mtls.yaml
```

---

## Topic 16.7 — Multi-Cluster EKS Management

> Managing dev/staging/prod or multi-region clusters requires consistent tooling across multiple kubeconfigs.

#### CLI

```bash
# Add multiple clusters to kubeconfig
aws eks update-kubeconfig --name dev-cluster  --region ap-south-1 --alias dev
aws eks update-kubeconfig --name prod-cluster --region us-east-1  --alias prod

# List and switch contexts
kubectl config get-contexts
kubectl config use-context prod
kubectl --context=dev get pods

# Install kubectx + kubens for fast switching
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens  /usr/local/bin/kubens
kubectx prod && kubens monitoring

# Deploy to all clusters with ArgoCD ApplicationSet
cat > appset.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-all-clusters
  namespace: argocd
spec:
  generators:
  - list:
      elements:
      - {cluster: dev,  url: https://dev-endpoint}
      - {cluster: prod, url: https://prod-endpoint}
  template:
    metadata:
      name: 'my-app-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/your-org/k8s-manifests
        targetRevision: main
        path: apps/{{cluster}}
      destination:
        server: '{{url}}'
        namespace: default
      syncPolicy:
        automated: {prune: true}
EOF

kubectl apply -f appset.yaml
```

---

## Topic 16.8 — Terraform for EKS (Infrastructure as Code)

> Terraform is the most widely used IaC tool for EKS — reproducible, version-controlled cluster definitions.

#### CLI

```bash
# Install Terraform
curl -LO https://releases.hashicorp.com/terraform/1.8.0/terraform_1.8.0_linux_amd64.zip
unzip terraform_1.8.0_linux_amd64.zip && sudo mv terraform /usr/local/bin/

mkdir terraform-eks && cd terraform-eks
cat > main.tf <<'EOF'
terraform {
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
}
provider "aws" { region = "ap-south-1" }

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name = "eks-vpc"
  cidr = "10.0.0.0/16"
  azs             = ["ap-south-1a", "ap-south-1b", "ap-south-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true
  public_subnet_tags  = { "kubernetes.io/role/elb" = 1 }
  private_subnet_tags = { "kubernetes.io/role/internal-elb" = 1 }
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
  cluster_name    = "my-tf-cluster"
  cluster_version = "1.30"
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets
  cluster_endpoint_public_access = true
  eks_managed_node_groups = {
    workers = {
      instance_types = ["t3.medium"]
      min_size = 1; max_size = 5; desired_size = 2
    }
    spot-workers = {
      instance_types = ["t3.medium", "t3.large", "t3a.medium"]
      capacity_type  = "SPOT"
      min_size = 0; max_size = 10; desired_size = 2
    }
  }
  cluster_addons = {
    coredns            = { most_recent = true }
    kube-proxy         = { most_recent = true }
    vpc-cni            = { most_recent = true }
    aws-ebs-csi-driver = { most_recent = true }
  }
}

output "cluster_name"     { value = module.eks.cluster_name }
output "cluster_endpoint" { value = module.eks.cluster_endpoint }
EOF

terraform init
terraform plan  -out=tfplan
terraform apply tfplan

# Connect kubectl
aws eks update-kubeconfig --name my-tf-cluster --region ap-south-1

# Destroy everything cleanly
terraform destroy
```

---
---

# PART 12 — Capstone Projects

> **How to use:** Each project integrates multiple services. Build from scratch — use the guide as reference, not a copy-paste source. Estimated times assume you have completed all relevant phases.

---

## Capstone Project 1 — Full-Stack Web App on ECS Fargate

**Difficulty:** ⭐⭐ Beginner–Intermediate | **Time:** 4–6 hours
**Services:** ECR, ECS Fargate, ALB, RDS, Secrets Manager, CloudWatch, GitHub Actions

### Architecture

```
Internet → ALB (public subnet)
             → ECS Fargate Tasks (private subnet, 2 replicas)
                 → RDS PostgreSQL (private subnet)
                 → Secrets Manager (DB credentials)
                 → CloudWatch Logs
GitHub push → GitHub Actions → ECR push → ECS rolling deploy
```

### What You Build

A containerised Node.js or Flask web app with a real PostgreSQL backend, deployed on ECS Fargate with auto scaling, structured logging, and a fully automated CI/CD pipeline.

### Step-by-Step Tasks

```bash
# 1. VPC: 2 public subnets + 2 private subnets + NAT Gateway
# 2. RDS PostgreSQL in private subnet (db.t3.micro, no public access)
aws rds create-db-instance \
  --db-instance-identifier capstone1-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username appuser \
  --master-user-password $(openssl rand -base64 16) \
  --allocated-storage 20 \
  --no-publicly-accessible

# 3. Store DB creds in Secrets Manager
aws secretsmanager create-secret \
  --name /capstone1/db \
  --secret-string '{"host":"capstone1-db.xxx.rds.amazonaws.com","user":"appuser","password":"<pw>"}'

# 4. ECR repo + push multi-stage image
aws ecr create-repository --repository-name capstone1-app

# 5. ECS task definition: reference secret via valueFrom, private subnet
# 6. ECS service: Fargate, 2 tasks, ALB in public subnet
# 7. Auto scaling: CPU target 60%, min 2, max 10
# 8. GitHub Actions CI/CD (see Topic 9.1 for workflow YAML)
# 9. Enable CloudWatch Container Insights on the cluster
# 10. Enable ECS circuit breaker with rollback
aws ecs update-service \
  --cluster capstone1 \
  --service webapp \
  --deployment-configuration "deploymentCircuitBreaker={enable=true,rollback=true}"
```

### Validation Checklist

- [ ] App accessible via ALB DNS name in a browser
- [ ] App reads and writes to RDS (prove with POST → GET round-trip)
- [ ] Push a code change → GitHub Actions builds and deploys automatically (< 5 min)
- [ ] Load test triggers auto scale-out: `ab -n 10000 -c 50 http://<alb-dns>/`
- [ ] Kill a task manually → ECS replaces it within 60 seconds
- [ ] DB password is NEVER in code, environment, or Git — only via Secrets Manager

---

## Capstone Project 2 — Production Microservices on EKS

**Difficulty:** ⭐⭐⭐ Intermediate–Advanced | **Time:** 8–12 hours
**Services:** EKS, ECR, ALB Ingress, RDS, Secrets Manager, ArgoCD, Prometheus, Grafana, External Secrets

### Architecture

```
Internet → ALB Ingress
              /         → Frontend Service (Deployment, 3 replicas, HPA)
              /api       → API Service     (Deployment, 3 replicas, HPA, IRSA)
              /metrics   → Prometheus      → Grafana dashboards
Git push → ArgoCD auto-sync → EKS
API pods → Secrets Manager (via IRSA, no keys stored anywhere)
```

### Kubernetes Manifests Repo Structure

```
k8s-manifests/
├── namespaces.yaml
├── apps/
│   ├── frontend/  (Deployment, Service, HPA, PDB)
│   └── api/       (Deployment, Service, HPA, PDB, ServiceAccount)
├── ingress.yaml
├── network-policies/
│   ├── deny-all-ingress.yaml
│   └── allow-frontend-to-api.yaml
└── monitoring/
    └── prometheus-values.yaml
```

### Step-by-Step Tasks

```bash
# 1. EKS cluster via Terraform (Topic 16.8)
# 2. Install: AWS LBC, EBS CSI, kube-prometheus-stack, External Secrets Operator
# 3. IRSA for API service account
eksctl create iamserviceaccount \
  --cluster capstone2 --name api-sa --namespace default \
  --attach-policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite --approve

# 4. ExternalSecret syncing DB creds (Topic 16.4)
# 5. ArgoCD pointing at k8s-manifests/apps/ (Topic 16.2)
# 6. HPA for both frontend and API (cpu-percent=50, min=2, max=10)
# 7. PodDisruptionBudgets: minAvailable=2 for both services
# 8. Network Policies: deny-all then allow frontend → api only
# 9. Ingress with path routing (/ → frontend, /api → api)
# 10. Grafana dashboard IDs 315 + 6417 for cluster + pod metrics
```

### Validation Checklist

- [ ] Three services accessible via Ingress path routing
- [ ] Push manifest to Git → ArgoCD syncs within 3 minutes
- [ ] API reads AWS secret via IRSA — zero hardcoded credentials anywhere
- [ ] CPU load triggers HPA scale-out: `kubectl run load --image=busybox -- sh -c "while true; do wget -q -O- http://api; done"`
- [ ] Drain a node → PDB keeps services live throughout drain
- [ ] Grafana shows live CPU, memory, request rate per pod
- [ ] Network policy blocks traffic from a test pod directly to API: `kubectl exec test -- curl http://api` → Connection refused

---

## Capstone Project 3 — Zero-Downtime CI/CD Pipeline

**Difficulty:** ⭐⭐⭐ Intermediate | **Time:** 5–7 hours
**Services:** ECR, ECS, CodePipeline, CodeBuild, CodeDeploy, ALB, SNS

### Pipeline Flow

```
GitHub PR merged
  → CodePipeline triggered automatically
  → CodeBuild: test → lint → docker build → ECR push
  → Manual Approval gate (SNS email notification)
  → CodeDeploy: Blue/Green to ECS
      → ALB shifts 10% traffic to green
      → 5-minute bake time
      → ALB shifts 100% to green → blue tasks terminated
  → Health check failure? → auto-rollback to blue in < 60 sec
```

### Step-by-Step Tasks

```bash
# 1. ECS service with CODE_DEPLOY controller (Topic 11.7)
aws ecs create-service \
  --deployment-controller '{"type":"CODE_DEPLOY"}' ...

# 2. CodeBuild buildspec.yml
cat > buildspec.yml <<'EOF'
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
      - IMAGE_TAG=$CODEBUILD_RESOLVED_SOURCE_VERSION
  build:
    commands:
      - docker build -t $ECR_URI:$IMAGE_TAG .
      - docker push $ECR_URI:$IMAGE_TAG
  post_build:
    commands:
      - printf '[{"name":"my-app","imageUri":"%s"}]' $ECR_URI:$IMAGE_TAG > imagedefinitions.json
artifacts:
  files: [imagedefinitions.json, appspec.yaml, taskdef.json]
EOF

# 3. SNS topic + email subscription for approval notifications
aws sns create-topic --name pipeline-approval
aws sns subscribe --topic-arn <arn> --protocol email \
  --notification-endpoint you@example.com

# 4. CodePipeline: Source → Build → ManualApproval → Deploy
# 5. CodeDeploy app + deployment group with Blue TG + Green TG + ALB listener
# 6. Test auto-rollback: break health check in new code → deploy → observe rollback
```

### Validation Checklist

- [ ] Code push triggers pipeline — no manual steps required
- [ ] Approval email received before production deployment
- [ ] Old (blue) version stays live while green tasks warm up
- [ ] Traffic shifts gradually: 10% → 100% visible in ALB target group metrics
- [ ] Break `/health` endpoint in a commit → pipeline auto-rolls back to blue
- [ ] Full deployment audit trail visible in CodeDeploy console

---

## Capstone Project 4 — Secure Containerised Platform

**Difficulty:** ⭐⭐⭐⭐ Advanced | **Time:** 6–8 hours
**Services:** EKS, ECR (KMS, signed), Kyverno, GuardDuty, VPC Endpoints, External Secrets, kube-bench

### Security Stack

```
ECR: KMS encrypted + tag immutable + image signing (Notation)
EKS: Private endpoint + KMS secret encryption + CloudTrail
Pods: non-root + read-only fs + no privilege escalation + dropped capabilities
Secrets: External Secrets Operator → Secrets Manager (no plaintext anywhere)
Network: deny-all + explicit allow + VPC Endpoints (no NAT for AWS APIs)
Policy: Kyverno enforces all security standards at admission time
Detection: GuardDuty EKS runtime monitoring
Audit: kube-bench CIS benchmark validation
```

### Step-by-Step Tasks

```bash
# 1. ECR repos with KMS + IMMUTABLE tags (Topics 10.2, 10.3)
aws ecr create-repository \
  --repository-name secure-app \
  --image-tag-mutability IMMUTABLE \
  --encryption-configuration encryptionType=KMS,kmsKey=$KMS_KEY_ARN

# 2. Sign all images with Notation (Topic 14.3)
# 3. EKS cluster: private endpoint + KMS for secrets
eksctl create cluster --name secure-cluster \
  --endpoint-private-access=true --endpoint-public-access=false

# 4. Enable GuardDuty + EKS protection
aws guardduty create-detector --enable

# 5. Install Kyverno + apply policies:
#    require-resource-limits, disallow-root, disallow-privilege-escalation,
#    require-read-only-rootfs, require-image-signature
# 6. External Secrets Operator — no secrets in YAML or Git
# 7. VPC Endpoints for ECR API, ECR DKR, S3, CloudWatch (Topic 10.4)
# 8. Network Policies: deny all by default, allow explicit paths only
# 9. Deploy test app with all controls active
# 10. Run kube-bench for CIS compliance
kubectl apply -f \
  https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-eks.yaml
kubectl logs -l app=kube-bench
```

### Validation Checklist

- [ ] Push unsigned image → Kyverno admission webhook rejects the pod
- [ ] Run pod as root (`runAsUser: 0`) → Kyverno rejects it
- [ ] `kubectl exec` session appears in CloudTrail audit log
- [ ] VPC flow logs show zero internet traffic from pod subnet to ECR (all via endpoint)
- [ ] GuardDuty raises finding on suspicious runtime activity
- [ ] kube-bench scores > 80% PASS on EKS CIS benchmark
- [ ] Rotate secret in Secrets Manager → External Secrets syncs new value within 1 hour

---

## Capstone Project 5 — Multi-Region Disaster Recovery

**Difficulty:** ⭐⭐⭐⭐ Advanced | **Time:** 8–10 hours
**Services:** ECS/EKS, ECR Replication, RDS Cross-Region Replica, Route 53, CloudWatch Alarms

### Architecture

```
Primary (ap-south-1)              Secondary (ap-southeast-1)
  ECS Service (3 tasks)   ←ECR replication→  ECS Service (0 tasks, standby)
  RDS PostgreSQL           ←read replica  →  RDS Read Replica
  ALB                                        ALB
    ↑                                          ↑
    └────── Route 53 Failover (health check) ──┘
```

### Step-by-Step Tasks

```bash
# 1. ECR cross-region replication (Topic 14.4)
# 2. RDS cross-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier myapp-db-replica \
  --source-db-instance-identifier arn:aws:rds:ap-south-1:<account>:db:myapp-db \
  --region ap-southeast-1 \
  --db-instance-class db.t3.micro

# 3. Secondary ECS infrastructure (same task defs, 0 desired tasks)
aws ecs create-service \
  --region ap-southeast-1 \
  --cluster secondary-cluster \
  --service-name my-app-standby \
  --desired-count 0 ...

# 4. Route 53 health check + failover DNS records
aws route53 create-health-check \
  --health-check-config '{
    "FullyQualifiedDomainName": "primary-alb.ap-south-1.elb.amazonaws.com",
    "Port": 80, "Type": "HTTP",
    "ResourcePath": "/health",
    "RequestInterval": 30, "FailureThreshold": 2
  }' \
  --caller-reference primary-$(date +%s)

# 5. DR runbook — failover procedure:
# a. Promote RDS read replica to standalone
aws rds promote-read-replica \
  --db-instance-identifier myapp-db-replica --region ap-southeast-1
# b. Scale ECS service in secondary to desired count
aws ecs update-service \
  --region ap-southeast-1 --cluster secondary-cluster \
  --service my-app-standby --desired-count 3
# c. Route 53 fails over automatically when health check fails

# 6. Failover drill: return HTTP 503 from primary /health
#    Measure time from failure to Route 53 completion
```

### Validation Checklist

- [ ] Image pushed to primary ECR appears in secondary within 5 minutes
- [ ] RDS ReplicaLag CloudWatch metric < 5 seconds
- [ ] Route 53 fails over automatically when primary returns 503 (no manual action)
- [ ] Full failover (promote replica + scale ECS + traffic shift) completes in < 10 minutes
- [ ] App accessible in secondary region after failover — no data loss for reads
- [ ] Document measured RTO and RPO for your DR plan

---

## Capstone Project 6 — GitOps-Driven EKS Platform (End-to-End)

**Difficulty:** ⭐⭐⭐⭐⭐ Expert | **Time:** 10–14 hours
**Services:** EKS, ECR, ArgoCD, Helm, Karpenter, Prometheus, Grafana, External Secrets, Kyverno, Terraform

### Philosophy: Everything in Git

```
platform-gitops/
├── bootstrap/          ← ArgoCD "App of Apps" — manages all other apps
├── cluster-addons/     ← AWS LBC, External Secrets, Karpenter, Kyverno (via Helm)
├── monitoring/         ← Prometheus stack, Grafana dashboards as ConfigMaps
├── security/           ← Kyverno policies, Network Policies
└── apps/
    ├── dev/            ← Dev environment manifests
    └── prod/           ← Prod environment manifests
```

### Step-by-Step Tasks

```bash
# Phase A: Infrastructure
# 1. EKS cluster via Terraform (Topic 16.8) — committed and reviewed as code
terraform apply    # All future changes go through terraform plan + apply via CI

# Phase B: GitOps bootstrap
# 2. Install ArgoCD (one-time — bootstrap only)
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Apply the "App of Apps" — ArgoCD now manages itself
cat > bootstrap/root-app.yaml <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/platform-gitops
    targetRevision: main
    path: bootstrap
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: {prune: true, selfHeal: true}
EOF
kubectl apply -f bootstrap/root-app.yaml
# From now on: git commit = cluster change. kubectl apply = banned in prod.

# Phase C: Add add-ons via Git (commit ArgoCD Application YAMLs pointing at Helm charts)
# → AWS Load Balancer Controller
# → External Secrets Operator
# → Karpenter NodePool
# → Kyverno + policies
# → kube-prometheus-stack

# Phase D: Deploy apps via Git
# Helm chart for your app → push to ECR
# ArgoCD Application YAML pointing at the chart → commit
# ArgoCD deploys + keeps in sync

# Phase E: Validation
argocd app list         # All apps should show Synced + Healthy
kubectl get nodes       # Karpenter manages node lifecycle
kubectl get policyreport --all-namespaces  # Kyverno compliance report
```

### Validation Checklist

- [ ] **Zero `kubectl apply` in production** — all changes via Git PRs
- [ ] Manually delete a Deployment → ArgoCD restores it within 3 minutes (selfHeal)
- [ ] Merge a PR adding a new Service → ArgoCD deploys it automatically, no manual step
- [ ] Scale replicas to 20 → Karpenter provisions new nodes within 90 seconds
- [ ] Try to apply a non-compliant pod (root user) → Kyverno rejects it
- [ ] ArgoCD sync history shows full audit trail of every change and who made it
- [ ] Drain any node → Karpenter replaces it, pods reschedule, zero downtime

---

## Capstone Summary Table

| # | Project | Key Skills Practised | Difficulty | Est. Time |
|---|---------|---------------------|------------|-----------|
| 1 | Full-Stack Web App on ECS Fargate | ECR, ECS, ALB, RDS, Secrets, CI/CD | ⭐⭐ | 4–6 hrs |
| 2 | Production Microservices on EKS | EKS, GitOps, IRSA, HPA, Observability | ⭐⭐⭐ | 8–12 hrs |
| 3 | Zero-Downtime CI/CD Pipeline | CodePipeline, Blue/Green, Auto-rollback | ⭐⭐⭐ | 5–7 hrs |
| 4 | Secure Containerised Platform | Image signing, Kyverno, GuardDuty, VPC Endpoints | ⭐⭐⭐⭐ | 6–8 hrs |
| 5 | Multi-Region Disaster Recovery | ECR Replication, RDS Replica, Route 53 Failover | ⭐⭐⭐⭐ | 8–10 hrs |
| 6 | GitOps-Driven EKS Platform | ArgoCD, Karpenter, Helm, Terraform, Kyverno | ⭐⭐⭐⭐⭐ | 10–14 hrs |

> **Do them in order.** Projects 1–3 = ECS track. Projects 2 + 4–6 = EKS track. Project 2 is the bridge — complete it before attempting 4, 5, or 6.

---
---

# APPENDIX — Quick Reference, Learning Path & Tips

---

## Quick Reference — Full Command Cheat Sheet

### ECR

| Action | CLI Command |
|--------|-------------|
| Login to ECR | `aws ecr get-login-password \| docker login --username AWS --password-stdin <uri>` |
| Create repo | `aws ecr create-repository --repository-name <name>` |
| List repos | `aws ecr describe-repositories` |
| List images | `aws ecr list-images --repository-name <name>` |
| Delete image | `aws ecr batch-delete-image --repository-name <name> --image-ids imageTag=<tag>` |
| Enable tag immutability | `aws ecr put-image-tag-mutability --repository-name <name> --image-tag-mutability IMMUTABLE` |
| Apply lifecycle policy | `aws ecr put-lifecycle-policy --repository-name <name> --lifecycle-policy-text file://policy.json` |
| Start image scan | `aws ecr start-image-scan --repository-name <name> --image-id imageTag=latest` |
| Get scan findings | `aws ecr describe-image-scan-findings --repository-name <name> --image-id imageTag=latest` |
| Set repo policy | `aws ecr set-repository-policy --repository-name <name> --policy-text file://policy.json` |
| Create pull-through rule | `aws ecr create-pull-through-cache-rule --ecr-repository-prefix dockerhub --upstream-registry-url registry-1.docker.io` |

### ECS

| Action | CLI Command |
|--------|-------------|
| Create cluster | `aws ecs create-cluster --cluster-name <name>` |
| Register task def | `aws ecs register-task-definition --cli-input-json file://task.json` |
| List task definitions | `aws ecs list-task-definitions` |
| Run task | `aws ecs run-task --cluster <name> --task-definition <name> --launch-type FARGATE ...` |
| List tasks | `aws ecs list-tasks --cluster <name>` |
| Describe task | `aws ecs describe-tasks --cluster <name> --tasks <arn>` |
| Create service | `aws ecs create-service --cluster <name> --service-name <name> --task-definition <name> ...` |
| Update service | `aws ecs update-service --cluster <name> --service <name> --desired-count <n>` |
| Force redeploy | `aws ecs update-service --cluster <name> --service <name> --force-new-deployment` |
| Shell into container | `aws ecs execute-command --cluster <name> --task <arn> --container <name> --interactive --command "/bin/sh"` |
| List container instances | `aws ecs list-container-instances --cluster <name>` |
| Tail logs | `aws logs tail /ecs/<log-group> --follow` |
| Create scheduled task | `aws events put-rule --name <name> --schedule-expression "cron(...)"` |

### EKS / kubectl

| Action | CLI Command |
|--------|-------------|
| Create cluster | `eksctl create cluster --name <name> --region <region>` |
| Connect kubectl | `aws eks update-kubeconfig --name <name> --region <region>` |
| Get nodes | `kubectl get nodes -o wide` |
| Get pods (all ns) | `kubectl get pods --all-namespaces` |
| Apply manifest | `kubectl apply -f <file>.yaml` |
| Describe resource | `kubectl describe pod/deployment/service <name>` |
| View logs | `kubectl logs -f <pod-name> -c <container>` |
| Shell into pod | `kubectl exec -it <pod-name> -- /bin/sh` |
| Rollout status | `kubectl rollout status deployment/<name>` |
| Rollback deployment | `kubectl rollout undo deployment/<name>` |
| Scale deployment | `kubectl scale deployment <name> --replicas=5` |
| Set image | `kubectl set image deployment/<name> <container>=<image>:<tag>` |
| Get events | `kubectl get events --sort-by='.lastTimestamp'` |
| Top nodes/pods | `kubectl top nodes && kubectl top pods` |
| Drain node | `kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data` |
| Create add-on | `aws eks create-addon --cluster-name <name> --addon-name <addon>` |
| Add IAM identity | `eksctl create iamidentitymapping --cluster <name> --arn <role-arn> --group system:masters` |
| Delete cluster | `eksctl delete cluster --name <name>` |

---

## 10-Week Learning Path

```
Week 1  ── Docker basics → Multi-stage builds → ECR setup → push/pull → lifecycle policies → scanning
Week 2  ── ECS Fargate: task definitions → IAM roles → run tasks → services + ALB → health checks
Week 3  ── ECS: auto scaling → secrets (SSM/Secrets Manager) → ECS Exec → deployment strategies
Week 4  ── ECS Advanced: EC2 launch type → placement strategies → Service Connect → EFS → scheduled tasks
Week 5  ── Kubernetes concepts → EKS cluster → kubectl → ConfigMaps → Secrets → PV/PVC (EBS/EFS)
Week 6  ── EKS workloads: Deployments → StatefulSets → DaemonSets → CronJobs → probes → PDB
Week 7  ── EKS networking: ALB Ingress → Network Policies → IRSA → Pod Identity → RBAC → namespaces
Week 8  ── EKS ops: HPA → Cluster Autoscaler → Karpenter → managed add-ons → cluster upgrades
Week 9  ── Security: container hardening → X-Ray tracing → Prometheus + Grafana → VPC endpoints
Week 10 ── Advanced: CI/CD pipeline (GitHub Actions) → Blue/Green (CodeDeploy) → cost optimization → AWS Copilot
```

---

## Common Mistakes & Tips — Full List

| Mistake | Fix |
|---------|-----|
| ECS task fails to pull image | Check task **execution role** has `AmazonEC2ContainerRegistryReadOnly` |
| ECS task exits immediately | Check CloudWatch logs; add `CMD ["sleep", "infinity"]` to debug |
| Using `latest` tag in production | Tag with git SHA or semantic version; enable ECR tag immutability |
| No lifecycle policy on ECR repos | ECR storage costs grow silently — apply policies to every repo |
| `kubectl` shows "Unauthorized" | Re-run `aws eks update-kubeconfig` with the correct IAM identity |
| EKS pods in Pending state | Check node capacity, resource requests, and security group rules |
| ECR auth token expired | Token valid 12h only — re-run `get-login-password` |
| Cross-account ECR pull fails | Add resource-based policy on ECR repo granting the other account's role |
| Fargate task can't reach internet | Use NAT Gateway for private subnets, or assign public IP with public subnet |
| EKS upgrade fails | Check for deprecated API versions with `kubectl api-versions` before upgrading |
| ECS task CPU/memory over-provisioned | Check CloudWatch metrics weekly; cut allocation if avg CPU < 20% |
| EKS pods evicted during node upgrade | Set Pod Disruption Budgets before draining nodes |
| No resource requests on k8s pods | Pods without requests are evicted first under node pressure — always set them |
| Secrets in plaintext env vars | Use Secrets Manager (ECS `valueFrom`) or k8s Secrets with KMS encryption |
| Single AZ ECS service | Always spread tasks across at least 2 AZs using `spread` placement strategy |
| EKS Network Policies not enforced | Enable VPC CNI network policy controller: `ENABLE_NETWORK_POLICY_CONTROLLER=true` |
| Forgetting add-on updates after upgrade | Run `aws eks update-addon` for every add-on after each control plane upgrade |
| EC2 Spot node pool only one instance type | Use multiple compatible instance types in Karpenter NodePool to avoid capacity failures |
| Docker Hub rate limiting in CI | Set up ECR pull-through cache to avoid the 100/6h unauthenticated pull limit |
| Containers running as root | Add `USER nonroot` in Dockerfile and `runAsNonRoot: true` in pod securityContext |

---

*Happy learning! Practice every phase completely before advancing. Real-world AWS skills come from doing, not just reading.*

