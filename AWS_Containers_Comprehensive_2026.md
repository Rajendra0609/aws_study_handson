# AWS Containers — Comprehensive Learning Resource 2026
### ECR · ECS · EKS · Security · Observability · GitOps · CI/CD
#### Beginner → Expert | Theory + Hands-On | Updated for 2026 Best Practices

> **How to use this guide:** Work through each part in order. Every topic provides a **Theoretical Explanation** followed by a **Hands-On Exercise**. Complete all exercises before moving to the next section. Finish the Capstone Projects at the end to validate end-to-end mastery.
>
> **Prerequisites:** AWS account with admin or power-user IAM access, basic Linux command-line familiarity.

---

## Prerequisites & Environment Setup

### Tools to Install

```bash
# ── AWS CLI v2 ──────────────────────────────────────────────────────────
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install
aws --version

# ── Docker Engine + Buildx ──────────────────────────────────────────────
sudo apt-get update && sudo apt-get install -y docker.io
sudo usermod -aG docker $USER && newgrp docker
docker version

# ── kubectl ─────────────────────────────────────────────────────────────
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client

# ── eksctl ──────────────────────────────────────────────────────────────
curl --silent --location \
  "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
  | tar xz -C /tmp && sudo mv /tmp/eksctl /usr/local/bin/
eksctl version

# ── Helm ────────────────────────────────────────────────────────────────
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version

# ── AWS CDK ─────────────────────────────────────────────────────────────
npm install -g aws-cdk && cdk --version

# ── Trivy (vulnerability scanner) ───────────────────────────────────────
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh \
  | sh -s -- -b /usr/local/bin
trivy --version

# ── Cosign (image signing) ───────────────────────────────────────────────
curl -Lo cosign https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
chmod +x cosign && sudo mv cosign /usr/local/bin/
cosign version

# ── Terraform ───────────────────────────────────────────────────────────
curl -fsSL https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
terraform --version
```

### Configure AWS CLI

```bash
aws configure
# AWS Access Key ID:     <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name:   ap-south-1   # Change to your nearest region
# Default output format: json

aws sts get-caller-identity   # Verify identity

# Convenience variables used throughout this guide
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-south-1
ECR_URI="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com"
```

### Core Concepts Glossary

| Term | Meaning |
|------|---------|
| **Image** | Immutable packaged snapshot of app + dependencies |
| **Container** | Running instance of an image — isolated Linux process |
| **Registry** | Storage for images (ECR = AWS managed OCI registry) |
| **Orchestrator** | Manages containers at scale (ECS, EKS) |
| **Task / Pod** | Unit running one or more containers |
| **Cluster** | Compute resources that run tasks/pods |
| **IAM Role** | Permissions attached to a service, task, or pod |
| **Namespace** | Kubernetes virtual cluster for isolation |
| **Helm Chart** | Kubernetes application package (templates + values) |
| **GitOps** | Infrastructure managed via Git commits as source of truth |
| **IRSA / Pod Identity** | Pods assume IAM roles without stored credentials |
| **SBOM** | Software Bill of Materials — inventory of all components |
| **Service Mesh** | Infrastructure for service-to-service communication (mTLS, tracing) |

---

# PART 1 — Docker Fundamentals

---

## Topic 1.1 — Container Concepts & Docker Basics

### Theoretical Explanation

A **container** is a lightweight, isolated runtime environment that packages an application with all its dependencies into a single portable unit. Containers share the host OS kernel but are isolated using Linux **namespaces** (pid, net, mnt, uts, ipc) and **cgroups** (resource limits). This makes them far more efficient than VMs while still providing strong isolation.

**Why containers matter in 2026:**
- **Consistency:** Identical images run from developer laptop → CI → staging → production.
- **Density:** One EC2 host runs dozens of containers vs. a handful of VMs.
- **Speed:** Containers start in milliseconds; VMs take minutes.
- **OCI Standard:** Docker, containerd, Podman, Finch (AWS), and all cloud platforms are interoperable through the Open Container Initiative spec.

**2026 tooling note:** Docker Desktop has largely been replaced in enterprise environments by open-source alternatives — **Finch** (AWS native), **nerdctl + containerd**, and **Podman**. The Dockerfile format and CLI commands remain compatible across all of them.

**Key Dockerfile instructions:**

| Instruction | Purpose |
|------------|---------|
| `FROM` | Base image — starting point |
| `WORKDIR` | Working directory inside the image |
| `COPY` / `ADD` | Copy files from build context into image |
| `RUN` | Execute commands during build (creates a new layer) |
| `ENV` | Set environment variables |
| `EXPOSE` | Document which port the app listens on |
| `USER` | Set the user to run the process (always non-root in production) |
| `CMD` / `ENTRYPOINT` | Default command when container starts |

### Hands-On Exercise

```bash
# ── Step 1: Create a simple Python app ───────────────────────────────
mkdir -p ~/lab-docker && cd ~/lab-docker

cat > app.py <<'EOF'
from http.server import HTTPServer, BaseHTTPRequestHandler
import os

class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.end_headers()
        env = os.environ.get("APP_ENV", "development")
        self.wfile.write(f"Hello from {env}!\n".encode())

if __name__ == "__main__":
    HTTPServer(("0.0.0.0", 8080), Handler).serve_forever()
EOF

# ── Step 2: Write a production-grade Dockerfile ───────────────────────
cat > Dockerfile <<'EOF'
FROM python:3.12-alpine

# Security: run as non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app
COPY app.py .

USER appuser
EXPOSE 8080
CMD ["python", "app.py"]
EOF

# ── Step 3: Build, run, and verify ───────────────────────────────────
docker build -t hello-app:v1.0 .
docker run -d -p 8080:8080 -e APP_ENV=production --name hello hello-app:v1.0
curl http://localhost:8080     # Expected: Hello from production!

# Verify non-root user
docker exec hello whoami       # Should print: appuser

# Inspect logs and stats
docker logs hello
docker stats hello --no-stream

# ── Cleanup ────────────────────────────────────────────────────────────
docker stop hello && docker rm hello
```

**Checklist:**
- [ ] Dockerfile uses non-root USER
- [ ] `curl` returns expected response with injected env var
- [ ] `docker exec whoami` confirms non-root user

---

## Topic 1.2 — Multi-Stage Builds

### Theoretical Explanation

A **multi-stage build** uses multiple `FROM` statements in one Dockerfile. Each stage can use a different base image. You selectively `COPY` artifacts from one stage to the next — build tools, compilers, test frameworks and source code are left behind. Only the minimal runtime artifact reaches the final image.

**Impact:**
- Node.js app with build tools: ~1.2 GB → after multi-stage: ~150 MB
- Go binary: 800 MB build image → ~10 MB scratch image
- Fewer packages = fewer CVEs = smaller ECR bill

**Three-stage pattern:**
```
Stage 1 (builder) — compiler, dev deps, source code
Stage 2 (tester)  — runs unit tests in CI; never ships to production
Stage 3 (prod)    — only the compiled artifact + minimal runtime
```

### Hands-On Exercise

```bash
cd ~/lab-docker

cat > package.json <<'EOF'
{"name":"demo","version":"1.0.0","scripts":{"start":"node server.js","test":"echo Tests passed"},"dependencies":{"express":"^4.18.0"}}
EOF

cat > server.js <<'EOF'
const express = require("express");
const app = express();
app.get("/", (req, res) => res.send("Multi-stage app running!\n"));
app.listen(3000, () => console.log("Server on :3000"));
EOF

cat > Dockerfile.multistage <<'EOF'
# ── Stage 1: Build ────────────────────────────────────────────────────
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production   # Deterministic, reproducible

# ── Stage 2: Test (run in CI, skip in prod builds) ────────────────────
FROM builder AS tester
RUN npm test

# ── Stage 3: Production ───────────────────────────────────────────────
FROM node:20-alpine AS production
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
USER appuser
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --chown=appuser:appgroup server.js .
EXPOSE 3000
CMD ["node", "server.js"]
EOF

# Build only the production stage
docker build -f Dockerfile.multistage --target production -t node-app:v1.0 .

# Compare sizes
echo "node:20-alpine size:"
docker image inspect node:20-alpine --format '{{.Size}}' | numfmt --to=iec
echo "Our production image size:"
docker image inspect node-app:v1.0 --format '{{.Size}}' | numfmt --to=iec

docker run -d -p 3000:3000 --name node-demo node-app:v1.0
curl http://localhost:3000
docker stop node-demo && docker rm node-demo
```

**Checklist:**
- [ ] Multi-stage Dockerfile with 3 stages created
- [ ] Production image significantly smaller than the base node:20 image
- [ ] App runs correctly from the production stage

---

## Topic 1.3 — Multi-Architecture Builds (AMD64 + ARM64)

### Theoretical Explanation

AWS Graviton (ARM64) instances offer up to **20% better price-performance** vs. equivalent x86 instances. **Multi-architecture images** are OCI manifest lists that bundle platform-specific variants under one tag. Docker automatically pulls the correct variant for the running platform.

**How it works:**
1. `docker buildx` creates a builder using QEMU emulation (or native cross-compile).
2. Each platform produces separate image layers.
3. A manifest list pushed to ECR references each platform digest.
4. ECS/EKS pull the matching variant automatically.

**2026 standard:** Building multi-arch images is now expected for any image intended for production. AWS CodeBuild supports native ARM64 build agents, eliminating QEMU overhead for ARM builds.

### Hands-On Exercise

```bash
# Create multi-platform buildx builder
docker buildx create --name multiarch --driver docker-container --use
docker buildx inspect --bootstrap

# Authenticate to ECR
aws ecr create-repository --repository-name multi-arch-app --region $REGION 2>/dev/null || true
aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin $ECR_URI

# Build and push for both platforms simultaneously
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --tag $ECR_URI/multi-arch-app:v1.0 \
  --push \
  -f ~/lab-docker/Dockerfile \
  ~/lab-docker/

# Verify the manifest list contains both platforms
docker buildx imagetools inspect $ECR_URI/multi-arch-app:v1.0
```

**Checklist:**
- [ ] Buildx builder created and bootstrapped
- [ ] Image pushed with both AMD64 and ARM64 variants
- [ ] `imagetools inspect` shows manifest list with two platform digests

---

## Topic 1.4 — Layer Caching for Fast CI Builds

### Theoretical Explanation

Docker caches layers and reuses them if neither the instruction nor any preceding layer has changed. **Layer cache efficiency** is the biggest single optimization in CI/CD pipelines.

**Rules for maximum cache reuse:**
1. **Least-changing layers first** — copy `requirements.txt` before `COPY . .`
2. **Install dependencies before copying code** — deps change far less often than code
3. **Use `--mount=type=cache`** in BuildKit for package manager caches (pip, npm)
4. **Remote cache with ECR** — share cache between CI agents via `--cache-from`/`--cache-to`

**2026 update:** BuildKit (default since Docker 23.x) supports ECR as a remote cache backend natively.

### Hands-On Exercise

```bash
cat > ~/lab-docker/Dockerfile.cached <<'EOF'
FROM python:3.12-alpine

WORKDIR /app

# Layer 1: system deps (rarely changes)
RUN apk add --no-cache gcc musl-dev

# Layer 2: Python deps (changes only when requirements.txt changes)
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Layer 3: app code (changes on every commit)
COPY . .
RUN addgroup -S app && adduser -S appuser -G app
USER appuser
CMD ["python", "app.py"]
EOF

echo "flask==3.0.0\ngunicorn==21.2.0" > ~/lab-docker/requirements.txt

# Cold build
time docker build -f ~/lab-docker/Dockerfile.cached -t cached-app:v1 ~/lab-docker/

# Simulate code-only change
echo "# updated" >> ~/lab-docker/app.py

# Warm build — only Layer 3 rebuilds
time docker build -f ~/lab-docker/Dockerfile.cached -t cached-app:v2 ~/lab-docker/

# Remote cache via ECR (share across CI agents)
docker buildx build \
  --cache-from type=registry,ref=$ECR_URI/multi-arch-app:cache \
  --cache-to   type=registry,ref=$ECR_URI/multi-arch-app:cache,mode=max \
  --tag $ECR_URI/multi-arch-app:latest \
  --push ~/lab-docker/
```

**Checklist:**
- [ ] Dockerfile layers ordered least-changing → most-changing
- [ ] Second build (code-only change) skips dependency installation
- [ ] Remote cache push to ECR succeeds

---

## Topic 1.5 — Dockerfile Security Best Practices

### Theoretical Explanation

Container security starts at the Dockerfile. A misconfigured Dockerfile can expose secrets, run as root, or bundle tools that expand the attack surface.

**2026 Security Checklist:**

| Practice | Why It Matters |
|---------|----------------|
| Non-root `USER` | Root in container ≈ root on host if container escapes |
| Minimal base (`-alpine`, `distroless`) | Fewer packages = fewer CVEs |
| Pin base image digest (`FROM nginx@sha256:...`) | Prevents upstream tag mutation |
| Multi-stage builds | Build tools never reach production |
| `.dockerignore` file | Prevents `.git`, secrets, large files from entering the build context |
| No secrets in `ENV` / `RUN` | `docker history` reveals all ENV vars |
| `COPY` not `ADD` | `ADD` has implicit tar-extraction/URL-fetching behaviour |
| Read-only rootfs | Pair with `readOnlyRootFilesystem: true` in pod spec |

**Trivy** is the industry-standard open-source vulnerability scanner for images, filesystems, SBOMs, and IaC files.

### Hands-On Exercise

```bash
# Secure Dockerfile
cat > ~/lab-docker/.dockerignore <<'EOF'
.git
.github
*.md
__pycache__
*.pyc
.env
node_modules
EOF

cat > ~/lab-docker/Dockerfile.secure <<'EOF'
FROM python:3.12-alpine

RUN apk add --no-cache curl \
    && addgroup -S appgroup \
    && adduser -S appuser -G appgroup

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY --chown=appuser:appgroup app.py .
USER appuser
EXPOSE 8080
CMD ["python", "app.py"]
EOF

docker build -f ~/lab-docker/Dockerfile.secure -t secure-app ~/lab-docker/

# Scan with Trivy
trivy image --severity HIGH,CRITICAL secure-app

# Confirm no secrets in image history
docker history secure-app   # Should show no secret values
```

**Checklist:**
- [ ] `.dockerignore` excludes `.git` and sensitive files
- [ ] Secure image uses non-root user and Alpine base
- [ ] Trivy shows fewer (or zero) CRITICAL CVEs vs. an ubuntu:latest base
- [ ] `docker history` contains no secret values

---

# PART 2 — Amazon ECR (Elastic Container Registry)

> **What is ECR?** AWS's managed OCI-compliant container registry — a private Docker Hub inside your AWS account, with IAM integration, VPC PrivateLink, image scanning, lifecycle management, and image signing.

---

## Topic 2.1 — Creating and Managing Repositories

### Theoretical Explanation

An ECR **repository** stores all versions of a single container image. Key design decisions:

- **Visibility:** Private (IAM auth required) or Public (`public.ecr.aws` — for open-source).
- **Tag immutability:** `IMMUTABLE` prevents overwriting an existing tag. Critical for production.
- **Encryption:** AES-256 by default (AWS-managed key). Use KMS CMK for compliance (HIPAA, PCI-DSS).
- **Replication:** Cross-region and cross-account at the registry level.

**2026 naming convention:** Hierarchical names like `payments/api`, `payments/worker` — simplifies lifecycle policies and IAM.

### Hands-On Exercise

```bash
# Create private repos
aws ecr create-repository \
  --repository-name workshop-app \
  --image-tag-mutability MUTABLE \
  --image-scanning-configuration scanOnPush=false \
  --region $REGION

aws ecr create-repository \
  --repository-name prod-app \
  --image-tag-mutability IMMUTABLE \
  --image-scanning-configuration scanOnPush=true \
  --region $REGION

# List repos
aws ecr describe-repositories --region $REGION \
  --query "repositories[*].{Name:repositoryName,URI:repositoryUri,Immutable:imageTagMutability}"

WORKSHOP_URI=$(aws ecr describe-repositories \
  --repository-names workshop-app \
  --query "repositories[0].repositoryUri" --output text --region $REGION)
echo "Workshop URI: $WORKSHOP_URI"
```

**Checklist:**
- [ ] `workshop-app` (MUTABLE) and `prod-app` (IMMUTABLE) repos created
- [ ] Repository URIs captured in shell variables

---

## Topic 2.2 — Authenticating Docker and Pushing Images

### Theoretical Explanation

ECR uses short-lived **12-hour tokens** retrieved with `aws ecr get-login-password`. Authentication patterns:

- **Local dev:** Run `get-login-password` manually before pushing.
- **EC2/ECS/Lambda:** Attach an IAM role — SDK handles credential rotation automatically. No stored keys.
- **CI/CD (GitHub Actions, CodeBuild):** Use OIDC federation — no long-lived IAM keys needed.
- **Cross-account:** ECR repository needs a resource-based policy granting access to the other account.

**2026 best practice:** Replace all long-lived IAM user access keys in CI/CD with OIDC/web identity federation.

### Hands-On Exercise

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region $REGION \
  | docker login --username AWS --password-stdin $ECR_URI
# Expected: Login Succeeded

# Build, tag, push
docker build -t hello-app:v1.0 ~/lab-docker/
docker tag hello-app:v1.0 $WORKSHOP_URI:v1.0
docker tag hello-app:v1.0 $WORKSHOP_URI:latest
docker push $WORKSHOP_URI:v1.0
docker push $WORKSHOP_URI:latest

# Verify
aws ecr list-images --repository-name workshop-app --region $REGION

# Pull back
docker rmi $WORKSHOP_URI:v1.0
docker pull $WORKSHOP_URI:v1.0
docker run --rm $WORKSHOP_URI:v1.0 echo "Pulled from ECR!"

# Cross-account read policy
cat > /tmp/ecr-policy.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[{"Sid":"AllowCrossAccountPull","Effect":"Allow",
    "Principal":{"AWS":"arn:aws:iam::111122223333:root"},
    "Action":["ecr:GetDownloadUrlForLayer","ecr:BatchGetImage","ecr:BatchCheckLayerAvailability"]}]
}
EOF
aws ecr set-repository-policy \
  --repository-name workshop-app \
  --policy-text file:///tmp/ecr-policy.json --region $REGION
```

**Checklist:**
- [ ] Docker authenticated to ECR
- [ ] Image pushed with `v1.0` and `latest` tags
- [ ] Image pulled back and run from ECR

---

## Topic 2.3 — Lifecycle Policies

### Theoretical Explanation

ECR charges ~$0.10/GB/month. Without lifecycle policies, repositories accumulate thousands of untagged images silently growing your bill. Rules are evaluated in **priority order** (lower number = higher priority) and specify whether to expire based on image count or age.

**2026 best practice:** Apply lifecycle policies to every repository at creation time, ideally via Terraform/CDK so it's never forgotten.

### Hands-On Exercise

```bash
# Push several test images
for TAG in v1.0 v2.0 v3.0 v4.0 v5.0; do
  docker tag hello-app:v1.0 $WORKSHOP_URI:$TAG
  docker push $WORKSHOP_URI:$TAG
done

cat > /tmp/lifecycle-policy.json <<'EOF'
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Remove untagged images after 1 day",
      "selection": {"tagStatus":"untagged","countType":"sinceImagePushed","countUnit":"days","countNumber":1},
      "action": {"type":"expire"}
    },
    {
      "rulePriority": 2,
      "description": "Keep only 3 most recent tagged images",
      "selection": {"tagStatus":"tagged","tagPrefixList":["v"],"countType":"imageCountMoreThan","countNumber":3},
      "action": {"type":"expire"}
    }
  ]
}
EOF

aws ecr put-lifecycle-policy \
  --repository-name workshop-app \
  --lifecycle-policy-text file:///tmp/lifecycle-policy.json \
  --region $REGION

# Dry-run preview
aws ecr get-lifecycle-policy-preview \
  --repository-name workshop-app --region $REGION
```

**Checklist:**
- [ ] Multiple images pushed; lifecycle policy applied
- [ ] Preview shows v1.0 and v2.0 would be expired (keeping v3–v5)

---

## Topic 2.4 — Image Scanning with Amazon Inspector

### Theoretical Explanation

**Basic scanning** uses Clair (free, OS packages only, on-push or on-demand).

**Enhanced scanning** uses Amazon Inspector and adds:
- Language package scanning (npm, pip, Maven, Go modules, Ruby gems)
- **Continuous re-scanning** when new CVEs are published — not just at push time
- Security Hub integration for centralized findings

**2026 shift-left pattern:** Run **Trivy** locally and in CI before pushing to ECR, catching vulnerabilities earlier. Inspector provides the continuous monitoring layer in ECR.

**Severity levels:** CRITICAL → HIGH → MEDIUM → LOW → INFORMATIONAL

### Hands-On Exercise

```bash
# Enable Amazon Inspector at registry level
aws inspector2 enable --resource-types ECR --region $REGION 2>/dev/null || true

aws ecr put-registry-scanning-configuration \
  --scan-type ENHANCED \
  --rules '[{"repositoryFilters":[{"filter":"*","filterType":"WILDCARD"}],"scanFrequency":"CONTINUOUS_SCAN"}]' \
  --region $REGION

# On-demand basic scan
aws ecr start-image-scan \
  --repository-name workshop-app \
  --image-id imageTag=v5.0 --region $REGION

aws ecr describe-image-scan-findings \
  --repository-name workshop-app \
  --image-id imageTag=v5.0 --region $REGION \
  --query "imageScanFindings.findingSeverityCounts"

# Local Trivy scan before push (shift-left)
trivy image --severity HIGH,CRITICAL --exit-code 1 hello-app:v1.0
```

**Checklist:**
- [ ] Enhanced scanning (Inspector) enabled at registry level
- [ ] On-demand scan triggered and severity counts reviewed
- [ ] Trivy local scan run; exit-code 1 on CRITICAL findings

---

## Topic 2.5 — Tag Immutability & KMS Encryption

### Theoretical Explanation

**Tag immutability** prevents overwriting an existing tag. Attempting to push a different image to an IMMUTABLE repo with an existing tag raises `ImageTagAlreadyExistsException`. Reference images by digest or semantic version in production manifests — never `latest`.

**KMS encryption** must be set at repository creation (cannot be changed later). A CMK gives you rotation control, access revocation, and CloudTrail audit of every cryptographic operation — required for HIPAA, PCI-DSS, and SOC 2.

### Hands-On Exercise

```bash
# Test tag immutability
PROD_URI="$ECR_URI/prod-app"
docker tag hello-app:v1.0 $PROD_URI:v1.0
docker push $PROD_URI:v1.0
docker tag node-app:v1.0 $PROD_URI:v1.0 2>/dev/null || true
docker push $PROD_URI:v1.0   # Expected: ImageTagAlreadyExistsException

# Get digest for production reference
IMAGE_DIGEST=$(aws ecr describe-images \
  --repository-name prod-app \
  --image-ids imageTag=v1.0 \
  --query "imageDetails[0].imageDigest" --output text --region $REGION)
echo "Production image digest: $IMAGE_DIGEST"

# KMS-encrypted repository
KMS_KEY_ARN=$(aws kms create-key \
  --description "ECR encryption key" \
  --query "KeyMetadata.Arn" --output text --region $REGION)

aws kms create-alias \
  --alias-name alias/ecr-encryption \
  --target-key-id $KMS_KEY_ARN --region $REGION

aws ecr create-repository \
  --repository-name secure-app \
  --image-tag-mutability IMMUTABLE \
  --encryption-configuration "encryptionType=KMS,kmsKey=${KMS_KEY_ARN}" \
  --region $REGION

aws ecr describe-repositories --repository-names secure-app --region $REGION \
  --query "repositories[0].encryptionConfiguration"
```

**Checklist:**
- [ ] Duplicate `v1.0` push to IMMUTABLE repo fails with expected error
- [ ] Image digest captured for production-safe reference
- [ ] `secure-app` repo created with KMS encryption verified

---

## Topic 2.6 — VPC Endpoints for ECR (PrivateLink)

### Theoretical Explanation

Without VPC endpoints, containers pull images through the public internet (or NAT Gateway). VPC endpoints keep all traffic inside the AWS network — required for private EKS clusters and eliminates NAT Gateway costs (~$0.045/GB) for image pulls.

**Required endpoints for ECR:**

| Service | Name | Type |
|---------|------|------|
| ECR API | `com.amazonaws.<region>.ecr.api` | Interface |
| ECR Docker | `com.amazonaws.<region>.ecr.dkr` | Interface |
| S3 | `com.amazonaws.<region>.s3` | Gateway (free) |
| CloudWatch Logs | `com.amazonaws.<region>.logs` | Interface |
| SSM | `com.amazonaws.<region>.ssm` | Interface |

### Hands-On Exercise

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text --region $REGION)
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text --region $REGION)
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "RouteTables[0].RouteTableId" --output text --region $REGION)

EP_SG=$(aws ec2 create-security-group \
  --group-name ecr-endpoint-sg \
  --description "ECR VPC Endpoint SG" \
  --vpc-id $VPC_ID --query "GroupId" --output text --region $REGION)

aws ec2 authorize-security-group-ingress \
  --group-id $EP_SG --protocol tcp --port 443 --cidr 10.0.0.0/8 --region $REGION

# S3 Gateway endpoint (free — always create)
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name "com.amazonaws.${REGION}.s3" \
  --vpc-endpoint-type Gateway \
  --route-table-ids $ROUTE_TABLE_ID --region $REGION

# ECR API Interface endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name "com.amazonaws.${REGION}.ecr.api" \
  --vpc-endpoint-type Interface \
  --subnet-ids $SUBNET_ID \
  --security-group-ids $EP_SG \
  --private-dns-enabled --region $REGION

# ECR Docker Interface endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name "com.amazonaws.${REGION}.ecr.dkr" \
  --vpc-endpoint-type Interface \
  --subnet-ids $SUBNET_ID \
  --security-group-ids $EP_SG \
  --private-dns-enabled --region $REGION

aws ec2 describe-vpc-endpoints \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "VpcEndpoints[*].{Service:ServiceName,State:State}" --region $REGION
```

**Checklist:**
- [ ] S3 Gateway endpoint created
- [ ] ECR API and Docker Interface endpoints created with private DNS
- [ ] All endpoints show `available` state

---

## Topic 2.7 — Pull Through Cache & Image Signing

### Theoretical Explanation

**Pull Through Cache** automatically caches images from Docker Hub, ECR Public, Quay, GitHub CR, and `registry.k8s.io` into your private ECR. Docker Hub rate-limits unauthenticated pulls to 100/6h per IP — this causes failures in busy EKS clusters and CI.

**Image signing** (Cosign / Notation + AWS Signer) proves an image was built by your trusted pipeline and hasn't been tampered with. Kyverno admission policies can enforce that only signed images are admitted to the cluster — blocking supply chain attacks.

### Hands-On Exercise

```bash
# Pull Through Cache — Docker Hub
aws ecr create-pull-through-cache-rule \
  --ecr-repository-prefix dockerhub \
  --upstream-registry-url registry-1.docker.io --region $REGION

# Pull via ECR (fetches from Docker Hub on first pull)
docker pull "$ECR_URI/dockerhub/nginx:alpine"

# Image signing with Cosign + KMS
SIGN_KEY_ARN=$(aws kms create-key \
  --description "Cosign signing key" \
  --key-usage SIGN_VERIFY \
  --key-spec ECC_NIST_P256 \
  --query "KeyMetadata.Arn" --output text --region $REGION)

aws kms create-alias \
  --alias-name alias/cosign-signing \
  --target-key-id $SIGN_KEY_ARN --region $REGION

cosign sign \
  --key "awskms:///alias/cosign-signing" \
  --tlog-upload=false \
  "$PROD_URI:v1.0"

cosign verify \
  --key "awskms:///alias/cosign-signing" \
  --insecure-ignore-tlog=true \
  "$PROD_URI:v1.0"
```

**Checklist:**
- [ ] Pull-through cache rule created for Docker Hub
- [ ] `nginx:alpine` pulled via ECR pull-through URI
- [ ] Image signed with Cosign + KMS key and signature verified

---

## Topic 2.8 — SBOM Generation (2026 Compliance)

### Theoretical Explanation

A **Software Bill of Materials (SBOM)** is a machine-readable inventory of all components in a software artifact. SBOMs are mandated by US Executive Order 14028 for federal software suppliers and are becoming standard in enterprise procurement. When a new CVE is published, you query your SBOM database to immediately identify affected images — rather than re-scanning everything.

**2026 formats:** CycloneDX (most widely supported) and SPDX. **Syft** generates SBOMs; **Grype** scans them for vulnerabilities.

### Hands-On Exercise

```bash
# Install Syft and Grype
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

# Generate SBOM
syft hello-app:v1.0 -o cyclonedx-json > /tmp/hello-app-sbom.json

# Count components
cat /tmp/hello-app-sbom.json | python3 -c \
  "import json,sys; sbom=json.load(sys.stdin); print(f'Components: {len(sbom.get(\"components\",[]))}')"

# Scan SBOM for vulnerabilities
grype sbom:/tmp/hello-app-sbom.json --severity high
```

**Checklist:**
- [ ] Syft and Grype installed
- [ ] SBOM generated in CycloneDX format
- [ ] Grype vulnerability scan run against SBOM
- [ ] Component count inspected

---

## Topic 2.9 — ECR Cross-Region Replication

### Theoretical Explanation

ECR replication copies images automatically to other regions after a push. Use cases: disaster recovery, multi-region ECS/EKS deployments (lower pull latency), multi-account architectures.

Replication is **asynchronous** and registry-wide. Optional prefix filters scope it to specific repositories (e.g., `prod-` prefix only).

### Hands-On Exercise

```bash
aws ecr put-replication-configuration \
  --replication-configuration '{
    "rules":[{"destinations":[
      {"region":"us-east-1","registryId":"'"$ACCOUNT_ID"'"},
      {"region":"eu-west-1","registryId":"'"$ACCOUNT_ID"'"}
    ],"repositoryFilters":[{"filter":"prod-","filterType":"PREFIX_MATCH"}]}]
  }' --region $REGION

# Verify
aws ecr describe-registry --region $REGION --query "replicationConfiguration"

# Push a prod image — check us-east-1 after ~2-5 minutes
docker tag hello-app:v1.0 $PROD_URI:v2.0
docker push $PROD_URI:v2.0
sleep 120
aws ecr describe-images --repository-name prod-app --region us-east-1 2>/dev/null \
  --query "imageDetails[*].imageTags" || echo "Check after a few minutes"
```

**Checklist:**
- [ ] Cross-region replication configured for `prod-` prefix
- [ ] `prod-app:v2.0` replicated to `us-east-1`


---

# PART 3 — Amazon ECS (Elastic Container Service)

> **What is ECS?** AWS's managed container orchestrator. You define what containers to run (task definitions) and how many (services). ECS handles scheduling, placement, health checking, auto scaling, and deep AWS-service integration. No Kubernetes knowledge required.

---

## Topic 3.1 — ECS Architecture & IAM Roles

### Theoretical Explanation

```
Cluster
 └── Service  (maintains N tasks, integrates with ALB)
      └── Task Definition  (blueprint: image, CPU, memory, ports, env)
           └── Task  (one running instance → one or more containers)
```

**Launch types:**

| Type | Description | Best For |
|------|-------------|----------|
| **Fargate** | Serverless — AWS manages underlying EC2 | Most workloads, variable traffic |
| **EC2** | You manage EC2 instances | GPU, custom AMIs, high-density |
| **External (ECS Anywhere)** | On-premises servers | Hybrid cloud, edge |

**Two IAM roles (often confused):**
- **Task Execution Role** — Used by the ECS agent to pull ECR images and send logs. Always attach `AmazonECSTaskExecutionRolePolicy`.
- **Task Role** — Used by code *inside* the container to call AWS APIs (S3, DynamoDB, SQS). Assign the minimum permissions your app needs.

### Hands-On Exercise

```bash
# Task execution role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  2>/dev/null || true

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite

# App task role
aws iam create-role \
  --role-name my-app-task-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  2>/dev/null || true

aws iam attach-role-policy \
  --role-name my-app-task-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

EXEC_ROLE_ARN=$(aws iam get-role --role-name ecsTaskExecutionRole --query "Role.Arn" --output text)
TASK_ROLE_ARN=$(aws iam get-role --role-name my-app-task-role --query "Role.Arn" --output text)
```

**Checklist:**
- [ ] `ecsTaskExecutionRole` created with execution policy attached
- [ ] `my-app-task-role` created with S3 read access
- [ ] Understand the difference between the two roles

---

## Topic 3.2 — Task Definitions

### Theoretical Explanation

A **task definition** is the immutable, versioned blueprint for running containers. Every change creates a new revision (`:1`, `:2`, ...). Key fields:

- `family` — name shared across revisions
- `cpu` / `memory` — 1024 cpu units = 1 vCPU; memory in MB
- `networkMode: awsvpc` — required for Fargate; each task gets its own ENI
- `executionRoleArn` — task execution role
- `taskRoleArn` — app's AWS identity
- `containerDefinitions` — image, ports, env, secrets, log config, health check

**2026 best practice:** Define task definitions as code (Terraform, CDK, JSON in Git). Never edit manually in the console.

**Sidecar patterns:**
- **Log router** (FireLens/Fluent Bit) — multi-destination log routing
- **X-Ray daemon** — distributed tracing
- **Init container** — DB migrations via `dependsOn` + `COMPLETE` condition

### Hands-On Exercise

```bash
aws logs create-log-group --log-group-name /ecs/my-app --region $REGION 2>/dev/null || true

cat > /tmp/task-def.json <<EOF
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "${EXEC_ROLE_ARN}",
  "taskRoleArn": "${TASK_ROLE_ARN}",
  "containerDefinitions": [{
    "name": "my-app",
    "image": "${WORKSHOP_URI}:v1.0",
    "portMappings": [{"containerPort": 8080,"protocol":"tcp","name":"http"}],
    "essential": true,
    "environment": [{"name":"APP_ENV","value":"production"}],
    "healthCheck": {
      "command": ["CMD-SHELL","curl -f http://localhost:8080/ || exit 1"],
      "interval": 30,"timeout": 5,"retries": 3,"startPeriod": 60
    },
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {"awslogs-group":"/ecs/my-app","awslogs-region":"${REGION}","awslogs-stream-prefix":"ecs"}
    }
  }]
}
EOF

aws ecs register-task-definition --cli-input-json file:///tmp/task-def.json --region $REGION

aws ecs describe-task-definition --task-definition my-app-task --region $REGION \
  --query "taskDefinition.{Family:family,Revision:revision,CPU:cpu,Memory:memory}"
```

**Checklist:**
- [ ] Task definition registered with health check and log config
- [ ] Revision number shows `:1`
- [ ] Both IAM role ARNs present in task definition

---

## Topic 3.3 — ECS Clusters, Services, and ALB

### Theoretical Explanation

An ECS **cluster** is a namespace for services and tasks. In Fargate mode it has no underlying infrastructure to manage. Combining a Service with an Application Load Balancer (ALB) is the standard production pattern:

```
Internet → ALB (port 80/443) → Target Group → ECS Tasks (multi-AZ)
```

Key service settings:
- `desiredCount` — target number of task replicas
- `minimumHealthyPercent: 100` — no tasks removed until replacements are healthy
- `maximumPercent: 200` — allows running 2x tasks during deployment
- `deploymentCircuitBreaker` — auto-rollback on failed deployments

### Hands-On Exercise

```bash
aws ecs create-cluster \
  --cluster-name my-cluster \
  --settings name=containerInsights,value=enabled \
  --region $REGION

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query "Vpcs[0].VpcId" --output text --region $REGION)
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[0].SubnetId" --output text --region $REGION)
SG_ID=$(aws ec2 create-security-group \
  --group-name ecs-task-sg \
  --description "ECS Task SG" \
  --vpc-id $VPC_ID --query "GroupId" --output text --region $REGION)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 8080 --cidr 0.0.0.0/0 --region $REGION

# Create ALB, Target Group, Listener
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name my-app-alb \
  --subnets $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
    --query "Subnets[*].SubnetId" --output text --region $REGION) \
  --security-groups $SG_ID --scheme internet-facing --type application \
  --query "LoadBalancers[0].LoadBalancerArn" --output text --region $REGION)

TG_ARN=$(aws elbv2 create-target-group \
  --name my-app-tg --protocol HTTP --port 8080 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-path "/" --health-check-interval-seconds 15 --healthy-threshold-count 2 \
  --query "TargetGroups[0].TargetGroupArn" --output text --region $REGION)

aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 \
  --default-actions "Type=forward,TargetGroupArn=$TG_ARN" --region $REGION

ALB_DNS=$(aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --query "LoadBalancers[0].DNSName" --output text --region $REGION)

# Create ECS service
aws ecs create-service \
  --cluster my-cluster \
  --service-name my-app-service \
  --task-definition my-app-task \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=my-app,containerPort=8080" \
  --health-check-grace-period-seconds 30 \
  --deployment-configuration "minimumHealthyPercent=100,maximumPercent=200,deploymentCircuitBreaker={enable=true,rollback=true}" \
  --region $REGION

aws ecs wait services-stable --cluster my-cluster --services my-app-service --region $REGION
sleep 30 && curl "http://${ALB_DNS}/"
```

**Checklist:**
- [ ] Cluster created with Container Insights enabled
- [ ] ALB + target group + listener created
- [ ] Service with 2 tasks stable and accessible via ALB DNS

---

## Topic 3.4 — Auto Scaling, Deployment Strategies & Secrets

### Theoretical Explanation

**Auto scaling** adjusts `desiredCount` automatically:
- **Target Tracking** — maintain a metric (e.g., 70% CPU). AWS calculates replicas.
- **Step Scaling** — different scale steps for different alarm thresholds.
- **Scheduled Scaling** — pre-warm capacity before predictable peak hours.

**Deployment strategies:**
- **Rolling Update** (default) — gradual task replacement; controlled by min/max healthy %.
- **Blue/Green (CodeDeploy)** — parallel green environment, configurable traffic shift, automatic rollback on health failure.
- **Circuit Breaker** — auto-rollback if new tasks fail to start. Enable on every service.

**Secrets** — use `secrets` array in task definition (not `environment`) to inject values from Secrets Manager or SSM Parameter Store at launch time. Values are fetched by the execution role and injected securely — never visible in the task definition JSON.

### Hands-On Exercise

```bash
# Auto Scaling: register target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id "service/my-cluster/my-app-service" \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 --max-capacity 10 --region $REGION

aws application-autoscaling put-scaling-policy \
  --policy-name cpu-tracking --service-namespace ecs \
  --resource-id "service/my-cluster/my-app-service" \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration \
    '{"TargetValue":70.0,"PredefinedMetricSpecification":{"PredefinedMetricType":"ECSServiceAverageCPUUtilization"},"ScaleInCooldown":300,"ScaleOutCooldown":60}' \
  --region $REGION

# Secrets: store in Secrets Manager, reference by ARN in task definition
aws secretsmanager create-secret \
  --name /myapp/prod/db-password \
  --secret-string "SuperSecretPassword123!" --region $REGION

DB_PASS_ARN=$(aws secretsmanager describe-secret \
  --secret-id /myapp/prod/db-password --query ARN --output text --region $REGION)

# In task definition containerDefinitions, use "secrets" not "environment":
# "secrets": [{"name":"DB_PASSWORD","valueFrom":"<arn>"}]
echo "Secret ARN for task definition: $DB_PASS_ARN"
```

**Checklist:**
- [ ] Scalable target registered (min=2, max=10)
- [ ] CPU target tracking policy applied at 70%
- [ ] Secret stored in Secrets Manager; ARN captured for task definition

---

## Topic 3.5 — ECS Exec, FireLens Logging & EFS Storage

### Theoretical Explanation

**ECS Exec** opens an interactive shell into a Fargate container via AWS Systems Manager — no SSH, no bastion host, no inbound port 22. Sessions are audited in CloudTrail and can be logged to S3/CloudWatch.

**FireLens** is ECS's Fluent Bit-powered log routing sidecar. It can route logs to CloudWatch, S3, Kinesis, Elasticsearch, Datadog, Splunk, or Loki — with enrichment and field redaction — without changing application code.

**EFS volumes** provide persistent, shared (`ReadWriteMany`) NFS storage for stateful ECS tasks. When a task stops and restarts, data on the EFS mount survives.

### Hands-On Exercise

```bash
# ECS Exec: enable on service
aws ecs update-service \
  --cluster my-cluster --service my-app-service \
  --enable-execute-command --force-new-deployment --region $REGION

aws ecs wait services-stable --cluster my-cluster --services my-app-service --region $REGION

TASK_ARN=$(aws ecs list-tasks --cluster my-cluster \
  --service-name my-app-service --query "taskArns[0]" --output text --region $REGION)

# Install Session Manager plugin first:
# curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb" -o /tmp/smp.deb && sudo dpkg -i /tmp/smp.deb

aws ecs execute-command \
  --cluster my-cluster --task $TASK_ARN --container my-app \
  --interactive --command "/bin/sh" --region $REGION
# Inside: whoami → appuser; env → see injected secrets

# EFS persistent volume
EFS_ID=$(aws efs create-file-system \
  --performance-mode generalPurpose --throughput-mode elastic \
  --encrypted --query FileSystemId --output text --region $REGION)

EFS_SG=$(aws ec2 create-security-group \
  --group-name efs-sg --description "EFS SG" \
  --vpc-id $VPC_ID --query GroupId --output text --region $REGION)
aws ec2 authorize-security-group-ingress \
  --group-id $EFS_SG --protocol tcp --port 2049 \
  --source-group $SG_ID --region $REGION
aws efs create-mount-target \
  --file-system-id $EFS_ID --subnet-id $SUBNET_ID \
  --security-groups $EFS_SG --region $REGION

echo "EFS ID: $EFS_ID — mount in task definition as efsVolumeConfiguration"
```

**Checklist:**
- [ ] ECS Exec enabled; shell opened into running container
- [ ] Verified container runs as non-root user
- [ ] EFS filesystem and mount target created

---

## Topic 3.6 — ECS Anywhere, Service Connect & Scheduled Tasks

### Theoretical Explanation

**ECS Anywhere** runs ECS tasks on your own servers (on-premises, edge, bare-metal) while managing them from the AWS console. Same task definitions and APIs — no AWS compute charges for the on-prem host. Useful for hybrid cloud, data-residency constraints, and edge computing.

**ECS Service Connect** (2022+, recommended over Cloud Map) provides:
- Stable DNS names for services (`http://backend:8080`)
- Automatic mutual TLS between services
- Per-connection metrics (latency, errors) in CloudWatch
- Built-in load balancing — no client-side retry configuration

**Scheduled Tasks** via EventBridge run task definitions on cron schedules. For complex multi-step workflows with dependencies, use AWS Step Functions with ECS integrations instead.

### Hands-On Exercise

```bash
# Service Connect: cluster with default namespace
aws ecs create-cluster \
  --cluster-name connected-cluster \
  --service-connect-defaults namespace=myapp-ns --region $REGION

# Backend service registers itself on the namespace
aws ecs create-service \
  --cluster connected-cluster --service-name backend \
  --task-definition my-app-task --desired-count 2 --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
  --service-connect-configuration '{
    "enabled":true,"namespace":"myapp-ns",
    "services":[{"portName":"http","clientAliases":[{"port":8080,"dnsName":"backend"}]}]
  }' --region $REGION

# EventBridge scheduled task
aws iam create-role \
  --role-name ecsEventsRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"events.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  2>/dev/null || true
aws iam attach-role-policy \
  --role-name ecsEventsRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceEventsRole

EVENTS_ROLE_ARN=$(aws iam get-role --role-name ecsEventsRole --query "Role.Arn" --output text)

aws events put-rule \
  --name daily-batch-job \
  --schedule-expression "cron(0 2 * * ? *)" \
  --state ENABLED --region $REGION
```

**Checklist:**
- [ ] `connected-cluster` created with Service Connect namespace
- [ ] Backend service registered; frontend can reach it at `http://backend:8080`
- [ ] EventBridge daily batch rule created

---

# PART 4 — Amazon EKS (Elastic Kubernetes Service)

> **What is EKS?** AWS's managed Kubernetes control plane. AWS manages etcd, kube-apiserver, scheduler, and controller-manager. You manage worker nodes (managed node groups, Fargate, or Auto Mode) and workloads.

---

## Topic 4.1 — Kubernetes Core Objects Reference

### Theoretical Explanation

| Object | Purpose |
|--------|---------|
| **Pod** | Smallest unit — 1+ containers sharing network/storage |
| **Deployment** | Manages replica Pods, rolling updates |
| **Service** | Stable DNS + IP for pods (ClusterIP / LoadBalancer / NodePort) |
| **Ingress** | HTTP routing rules → Services (requires Ingress Controller) |
| **ConfigMap** | Non-sensitive config (key-value or files) |
| **Secret** | Sensitive data (base64, optionally KMS-encrypted in etcd) |
| **Namespace** | Virtual cluster — isolation boundary |
| **ServiceAccount** | Pod identity for RBAC and IAM |
| **PersistentVolumeClaim** | Pod's request for persistent storage |
| **StatefulSet** | Stable pod identity + ordered ops (databases, Kafka) |
| **DaemonSet** | One pod per node (log collectors, monitoring agents) |
| **Job / CronJob** | Run-to-completion / scheduled batch processing |
| **HorizontalPodAutoscaler** | Scale pods based on metrics |
| **PodDisruptionBudget** | Minimum pods available during voluntary disruptions |
| **NetworkPolicy** | Pod-level ingress/egress firewall |

---

## Topic 4.2 — Creating and Connecting to an EKS Cluster

### Theoretical Explanation

EKS charges $0.10/hour per cluster for the control plane. You provide worker nodes via:
- **Managed Node Groups** — AWS manages OS patching and graceful draining. Recommended.
- **Fargate Profiles** — Serverless pods; no nodes to manage.
- **EKS Auto Mode** (2024+) — AWS manages node provisioning, scaling, and lifecycle. Lowest operational overhead.

**kubectl → EKS auth flow:** `aws eks update-kubeconfig` writes credentials to `~/.kube/config`. kubectl sends the IAM identity token to EKS. EKS maps the IAM identity to Kubernetes RBAC groups.

### Hands-On Exercise

```bash
# Create EKS cluster (15-20 min)
eksctl create cluster \
  --name my-eks-cluster \
  --region $REGION \
  --version 1.31 \
  --nodegroup-name workers \
  --node-type t3.medium \
  --nodes 2 --nodes-min 1 --nodes-max 4 \
  --managed --with-oidc --ssh-access=false

# Verify
kubectl get nodes -o wide
kubectl get pods --all-namespaces
kubectl cluster-info

# Add Spot node group for cost optimization
eksctl create nodegroup \
  --cluster my-eks-cluster --region $REGION \
  --name spot-workers \
  --instance-types t3.medium,t3.large \
  --spot --nodes-min 0 --nodes-max 6
```

**Checklist:**
- [ ] EKS cluster created; all nodes in `Ready` state
- [ ] `kubectl get pods -A` shows kube-system pods running
- [ ] Spot node group added

---

## Topic 4.3 — Deploying Applications, Services & Ingress

### Theoretical Explanation

Core deployment pattern:
```
Deployment → ReplicaSet → Pods ← Service (stable endpoint)
                                       ↑
                                  Ingress (HTTP routing)
```

**Always set resource `requests`** — without them, pods are scheduled arbitrarily and are first to be evicted under node memory pressure.

**Zero-downtime rolling update settings:**
- `maxUnavailable: 0` — no pods removed until replacements are ready
- `maxSurge: 1` — one extra pod created during rollout

### Hands-On Exercise

```bash
kubectl create namespace workshop

cat > /tmp/deployment.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: workshop
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate: {maxUnavailable: 0, maxSurge: 1}
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: ${WORKSHOP_URI}:v1.0
        ports: [{containerPort: 8080}]
        env: [{name: APP_ENV, value: kubernetes}]
        resources:
          requests: {cpu: 100m, memory: 128Mi}
          limits:   {cpu: 500m, memory: 256Mi}
        readinessProbe:
          httpGet: {path: /, port: 8080}
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet: {path: /, port: 8080}
          periodSeconds: 10
EOF

kubectl apply -f /tmp/deployment.yaml
kubectl rollout status deployment/my-app -n workshop

cat > /tmp/svc.yaml <<'EOF'
apiVersion: v1
kind: Service
metadata: {name: my-app, namespace: workshop}
spec:
  selector: {app: my-app}
  ports: [{port: 80, targetPort: 8080}]
  type: ClusterIP
EOF
kubectl apply -f /tmp/svc.yaml

# Test
kubectl run test --image=curlimages/curl --restart=Never -n workshop \
  -- curl http://my-app/
kubectl logs test -n workshop
kubectl delete pod test -n workshop
```

**Checklist:**
- [ ] Deployment created with 3 replicas and resource requests
- [ ] Service created; pod-to-service connectivity verified
- [ ] Rolling update and rollback tested (`kubectl rollout undo`)

---

## Topic 4.4 — AWS Load Balancer Controller & Ingress

### Theoretical Explanation

The **AWS Load Balancer Controller** watches Kubernetes Ingress resources and creates/manages AWS ALBs automatically. Install it once per cluster via Helm. Use `target-type: ip` for direct pod IP routing (works with Fargate and reduces network hops).

**2026:** The controller also supports the Kubernetes **Gateway API** (`HTTPRoute`, `GRPCRoute`) — the successor to Ingress, with more expressive routing semantics. Use Gateway API for new deployments.

### Hands-On Exercise

```bash
eksctl utils associate-iam-oidc-provider \
  --region $REGION --cluster my-eks-cluster --approve

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.8.0/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json --region $REGION 2>/dev/null || true

LBC_POLICY=$(aws iam list-policies \
  --query "Policies[?PolicyName=='AWSLoadBalancerControllerIAMPolicy'].Arn" --output text)

eksctl create iamserviceaccount \
  --cluster my-eks-cluster --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn $LBC_POLICY \
  --override-existing-serviceaccounts --approve --region $REGION

helm repo add eks https://aws.github.io/eks-charts && helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$REGION --set vpcId=$VPC_ID

cat > /tmp/ingress.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: workshop
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
            name: my-app
            port: {number: 80}
EOF

kubectl apply -f /tmp/ingress.yaml
sleep 90
ALB_DNS=$(kubectl get ingress my-app-ingress -n workshop \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
echo "ALB: http://$ALB_DNS"
curl "http://${ALB_DNS}/"
```

**Checklist:**
- [ ] OIDC provider associated
- [ ] AWS LBC IAM policy, service account, and Helm release created
- [ ] Ingress resource provisioned an ALB
- [ ] App accessible via ALB DNS

---

## Topic 4.5 — IRSA and EKS Pod Identity

### Theoretical Explanation

**IRSA** (IAM Roles for Service Accounts) uses the cluster OIDC provider to let pods assume IAM roles without stored credentials. The trust policy scopes the role to a specific namespace + service account.

**EKS Pod Identity** (GA 2023, preferred in 2026) is simpler:
- No OIDC trust policy needed in IAM — association stored in EKS
- Supports up to 24-hour sessions
- Credentials refresh every 15 minutes
- Works with EKS Auto Mode

| | IRSA | Pod Identity |
|--|------|-------------|
| Trust policy | Required (OIDC conditions) | Not needed |
| Session | Up to 1 hour | Up to 24 hours |
| Multiple roles/SA | No | Yes |
| Preferred for new workloads | No | Yes |

### Hands-On Exercise

```bash
# ── IRSA approach ─────────────────────────────────────────────────────
OIDC_ID=$(aws eks describe-cluster --name my-eks-cluster \
  --query "cluster.identity.oidc.issuer" --output text --region $REGION | sed 's|https://||')

cat > /tmp/irsa-trust.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Allow",
    "Principal":{"Federated":"arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_ID}"},
    "Action":"sts:AssumeRoleWithWebIdentity",
    "Condition":{"StringEquals":{
      "${OIDC_ID}:sub":"system:serviceaccount:workshop:s3-reader",
      "${OIDC_ID}:aud":"sts.amazonaws.com"
    }}
  }]
}
EOF

aws iam create-role --role-name eks-s3-reader \
  --assume-role-policy-document file:///tmp/irsa-trust.json 2>/dev/null || true
aws iam attach-role-policy --role-name eks-s3-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

kubectl create serviceaccount s3-reader -n workshop 2>/dev/null || true
kubectl annotate serviceaccount s3-reader -n workshop \
  eks.amazonaws.com/role-arn="arn:aws:iam::${ACCOUNT_ID}:role/eks-s3-reader"

# ── Pod Identity approach (preferred) ─────────────────────────────────
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name eks-pod-identity-agent --region $REGION

aws iam create-role --role-name eks-pod-identity-s3 \
  --assume-role-policy-document \
    '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}' \
  2>/dev/null || true
aws iam attach-role-policy --role-name eks-pod-identity-s3 \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

kubectl create serviceaccount pod-identity-sa -n workshop 2>/dev/null || true

aws eks create-pod-identity-association \
  --cluster-name my-eks-cluster --namespace workshop \
  --service-account pod-identity-sa \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/eks-pod-identity-s3 \
  --region $REGION
```

**Checklist:**
- [ ] IRSA trust policy created scoped to `s3-reader` service account
- [ ] Pod Identity agent add-on installed
- [ ] Pod Identity association created (no OIDC trust policy needed)

---

## Topic 4.6 — HPA, Karpenter & Cluster Autoscaling

### Theoretical Explanation

**HPA** (Horizontal Pod Autoscaler) scales pod replicas based on metrics. **Karpenter** provisions/removes EC2 nodes to match pod scheduling needs.

**Karpenter vs Cluster Autoscaler (2026):**

| | Cluster Autoscaler | Karpenter |
|--|-------------------|-----------|
| Speed | 2-5 min | 30-90 sec |
| Instance selection | Fixed node group types | Any EC2 type |
| Consolidation | Limited | Aggressive (replaces underused nodes) |
| Spot handling | Separate node groups | Native |

**Karpenter NodePool** defines constraints (instance family, AZ, capacity type). `EC2NodeClass` provides AWS-specific config (AMI, security groups, subnets).

**KEDA** (Kubernetes Event-Driven Autoscaling) extends HPA to scale based on external events — SQS queue depth, Kafka lag, HTTP request rate — and supports **scale-to-zero**.

### Hands-On Exercise

```bash
# Install Metrics Server (required for HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
sleep 30 && kubectl top nodes

# HPA — declarative YAML (preferred for GitOps)
cat > /tmp/hpa.yaml <<'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: workshop
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: my-app}
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target: {type: Utilization, averageUtilization: 50}
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
EOF

kubectl apply -f /tmp/hpa.yaml
kubectl get hpa -n workshop

# Generate load
kubectl run load --image=busybox --restart=Never -n workshop \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://my-app/; done"
kubectl get hpa -n workshop --watch &
sleep 90; kill %1
kubectl delete pod load -n workshop
```

**Checklist:**
- [ ] Metrics Server installed; `kubectl top nodes` works
- [ ] HPA with CPU and scale behavior configured
- [ ] Load generator triggered scale-out

---

## Topic 4.7 — ConfigMaps, Secrets & Persistent Storage

### Theoretical Explanation

**ConfigMaps** store non-sensitive config; **Secrets** store sensitive data. Both can be consumed as environment variables or volume mounts. Kubernetes Secrets are base64-encoded (not encrypted) in etcd by default — enable **KMS envelope encryption** at cluster creation for production.

**Persistent storage on EKS:**
- **EBS CSI** — `ReadWriteOnce` (one node), low latency, for databases
- **EFS CSI** — `ReadWriteMany` (multiple pods/nodes), for shared content

**External Secrets Operator** syncs secrets from Secrets Manager/SSM into Kubernetes Secrets automatically — the recommended pattern for 2026.

### Hands-On Exercise

```bash
# ConfigMap and Secret
kubectl create configmap app-config \
  --from-literal=APP_ENV=production --from-literal=LOG_LEVEL=info -n workshop

kubectl create secret generic db-credentials \
  --from-literal=username=appuser --from-literal=password=SuperSecret -n workshop

# EBS StorageClass (gp3)
cat > /tmp/ebs-sc.yaml <<'EOF'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata: {name: ebs-gp3}
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters: {type: gp3, encrypted: "true"}
EOF
kubectl apply -f /tmp/ebs-sc.yaml

# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace --set installCRDs=true

# Create ExternalSecret that syncs from Secrets Manager
cat > /tmp/external-secret.yaml <<EOF
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata: {name: aws-secrets-manager, namespace: workshop}
spec:
  provider:
    aws:
      service: SecretsManager
      region: ${REGION}
      auth:
        jwt:
          serviceAccountRef: {name: external-secrets-sa}
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata: {name: app-db-creds, namespace: workshop}
spec:
  refreshInterval: 1h
  secretStoreRef: {name: aws-secrets-manager, kind: SecretStore}
  target: {name: db-creds-synced, creationPolicy: Owner}
  data:
  - secretKey: password
    remoteRef: {key: /myapp/prod/db-password}
EOF
kubectl apply -f /tmp/external-secret.yaml 2>/dev/null || echo "Requires ESO IAM setup"
```

**Checklist:**
- [ ] ConfigMap and Secret created; consumed in a pod
- [ ] EBS gp3 StorageClass created
- [ ] External Secrets Operator installed
- [ ] ExternalSecret applied (syncs Secrets Manager → Kubernetes Secret)

---

## Topic 4.8 — StatefulSets, DaemonSets, Jobs & Probes

### Theoretical Explanation

**StatefulSet** pods have stable names (`postgres-0`, `postgres-1`), stable DNS (`postgres-0.postgres.default.svc`), and per-pod PVCs (via `volumeClaimTemplates`). Used for databases, Kafka, Redis clusters.

**DaemonSet** ensures one pod per node — for log collectors (Fluent Bit), monitoring agents (node-exporter), security scanners (Falco).

**Probes:**

| Probe | Failure Action |
|-------|---------------|
| **Startup** | Delay liveness/readiness checks while app starts |
| **Liveness** | Restart the container |
| **Readiness** | Remove from Service endpoints |

**2026:** Always define all three probes. Use separate `/health` (liveness) and `/ready` (readiness) endpoints in your app. Readiness should check external dependencies (DB connection).

### Hands-On Exercise

```bash
# StatefulSet with EBS
kubectl create secret generic postgres-secret \
  --from-literal=password=PGPass123 -n workshop

cat > /tmp/statefulset.yaml <<'EOF'
apiVersion: apps/v1
kind: StatefulSet
metadata: {name: postgres, namespace: workshop}
spec:
  serviceName: postgres
  replicas: 1
  selector: {matchLabels: {app: postgres}}
  template:
    metadata: {labels: {app: postgres}}
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports: [{containerPort: 5432}]
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef: {name: postgres-secret, key: password}
        - {name: PGDATA, value: /var/lib/postgresql/data/pgdata}
        volumeMounts:
        - {name: data, mountPath: /var/lib/postgresql/data}
  volumeClaimTemplates:
  - metadata: {name: data}
    spec:
      accessModes: [ReadWriteOnce]
      storageClassName: ebs-gp3
      resources: {requests: {storage: 10Gi}}
---
apiVersion: v1
kind: Service
metadata: {name: postgres, namespace: workshop}
spec:
  clusterIP: None
  selector: {app: postgres}
  ports: [{port: 5432}]
EOF
kubectl apply -f /tmp/statefulset.yaml
kubectl get statefulset -n workshop

# DaemonSet
cat > /tmp/daemonset.yaml <<'EOF'
apiVersion: apps/v1
kind: DaemonSet
metadata: {name: fluent-bit, namespace: kube-system}
spec:
  selector: {matchLabels: {app: fluent-bit}}
  template:
    metadata: {labels: {app: fluent-bit}}
    spec:
      containers:
      - name: fluent-bit
        image: amazon/aws-for-fluent-bit:stable
        resources: {limits: {memory: 200Mi}, requests: {cpu: 100m, memory: 100Mi}}
        volumeMounts: [{name: varlog, mountPath: /var/log, readOnly: true}]
      volumes: [{name: varlog, hostPath: {path: /var/log}}]
EOF
kubectl apply -f /tmp/daemonset.yaml
kubectl get daemonset -n kube-system fluent-bit
```

**Checklist:**
- [ ] PostgreSQL StatefulSet deployed; pod named `postgres-0`
- [ ] Fluent Bit DaemonSet shows one pod per node
- [ ] All three probe types understood and configured

---

## Topic 4.9 — Network Policies, PDBs & RBAC

### Theoretical Explanation

**NetworkPolicies** are pod-level firewall rules. By default all pods communicate freely — apply a deny-all then explicitly allow required paths (zero-trust networking).

**Pod Disruption Budgets (PDB)** protect availability during voluntary disruptions (node upgrades, `kubectl drain`, Karpenter consolidation). `minAvailable: 2` ensures at least 2 pods remain up.

**RBAC** controls which Kubernetes identities (users, ServiceAccounts) can perform which actions on which resources. Follow least-privilege: grant only the specific verbs and resources needed.

### Hands-On Exercise

```bash
# Deny all ingress in workshop namespace
cat > /tmp/deny-all.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: deny-all-ingress, namespace: workshop}
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
kubectl apply -f /tmp/deny-all.yaml

# Allow frontend → my-app only
cat > /tmp/allow-app.yaml <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata: {name: allow-frontend, namespace: workshop}
spec:
  podSelector: {matchLabels: {app: my-app}}
  policyTypes: [Ingress]
  ingress:
  - from:
    - podSelector: {matchLabels: {role: frontend}}
    ports: [{protocol: TCP, port: 8080}]
EOF
kubectl apply -f /tmp/allow-app.yaml

# PDB
cat > /tmp/pdb.yaml <<'EOF'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: my-app-pdb, namespace: workshop}
spec:
  minAvailable: 2
  selector: {matchLabels: {app: my-app}}
EOF
kubectl apply -f /tmp/pdb.yaml
kubectl get pdb -n workshop

# RBAC — read-only role for a team
cat > /tmp/rbac.yaml <<'EOF'
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: {name: pod-reader, namespace: workshop}
rules:
- apiGroups: [""]
  resources: [pods, pods/log]
  verbs: [get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: {name: pod-reader-binding, namespace: workshop}
subjects:
- kind: ServiceAccount
  name: pod-identity-sa
  namespace: workshop
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF
kubectl apply -f /tmp/rbac.yaml
```

**Checklist:**
- [ ] Deny-all NetworkPolicy applied to `workshop` namespace
- [ ] Explicit allow policy for frontend→my-app created
- [ ] PDB requiring `minAvailable: 2` applied
- [ ] RBAC Role and RoleBinding created

---

# PART 5 — Security

---

## Topic 5.1 — Kyverno Policy Engine

### Theoretical Explanation

**Kyverno** is a Kubernetes-native policy engine that validates, mutates, and generates resources at admission time. Policies are YAML — no Rego language needed.

**Three modes:**
- **Validate:** Block non-compliant resources (e.g., pods without resource limits)
- **Mutate:** Auto-modify resources (e.g., add default labels)
- **Generate:** Create related resources (e.g., auto-create NetworkPolicy per namespace)

**2026 standard Kyverno policies:**
- Require non-root containers
- Disallow privilege escalation
- Require resource limits
- Require image signatures (supply chain security)
- Disallow `latest` tag in production namespaces

### Hands-On Exercise

```bash
helm repo add kyverno https://kyverno.github.io/kyverno/ && helm repo update
helm install kyverno kyverno/kyverno -n kyverno --create-namespace \
  --set admissionController.replicas=3

# Policy: require non-root
cat > /tmp/kyverno-non-root.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: require-non-root}
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-runAsNonRoot
    match:
      any:
      - resources: {kinds: [Pod], namespaces: [workshop]}
    validate:
      message: "Containers must set securityContext.runAsNonRoot=true"
      pattern:
        spec:
          containers:
          - (name): "?*"
            securityContext: {runAsNonRoot: true}
EOF
kubectl apply -f /tmp/kyverno-non-root.yaml

# Policy: disallow latest tag
cat > /tmp/kyverno-no-latest.yaml <<'EOF'
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: disallow-latest-tag}
spec:
  validationFailureAction: Enforce
  rules:
  - name: require-image-tag
    match:
      any:
      - resources: {kinds: [Pod], namespaces: [workshop]}
    validate:
      message: "Image tag 'latest' is not allowed in production."
      deny:
        conditions:
          any:
          - key: "{{request.object.spec.containers[].image | [?contains(@,':latest')] | length(@)}}"
            operator: GreaterThan
            value: "0"
EOF
kubectl apply -f /tmp/kyverno-no-latest.yaml

# Test: root container (should be blocked)
kubectl run root-test --image=nginx:alpine -n workshop \
  --overrides='{"spec":{"securityContext":{"runAsUser":0}}}' 2>&1 | grep "admission webhook"

kubectl get policyreport --all-namespaces 2>/dev/null | head -10
```

**Checklist:**
- [ ] Kyverno installed with 3 admission controller replicas
- [ ] `require-non-root` policy applied to `workshop` namespace
- [ ] `disallow-latest-tag` policy applied
- [ ] Root container pod rejected by admission webhook

---

## Topic 5.2 — GuardDuty, VPC Endpoints & Container Hardening

### Theoretical Explanation

**GuardDuty EKS Runtime Monitoring** uses an eBPF agent on every node to detect:
- Unexpected process execution inside containers
- Crypto-mining activity
- Container escape attempts
- Privilege escalation, sensitive file access

**Container hardening (2026 pod security standards):**

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
  seccompProfile:
    type: RuntimeDefault
```

**kube-bench** validates CIS Kubernetes Benchmark compliance on your cluster nodes.

### Hands-On Exercise

```bash
# Enable GuardDuty with EKS Runtime Monitoring
DETECTOR_ID=$(aws guardduty create-detector --enable \
  --features '[{"Name":"EKS_RUNTIME_MONITORING","Status":"ENABLED"}]' \
  --query DetectorId --output text --region $REGION 2>/dev/null || \
  aws guardduty list-detectors --query "DetectorIds[0]" --output text --region $REGION)
echo "GuardDuty Detector: $DETECTOR_ID"

# Hardened pod spec example
cat > /tmp/hardened-pod.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata: {name: hardened-demo, namespace: workshop}
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10000
    seccompProfile: {type: RuntimeDefault}
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      readOnlyRootFilesystem: true
      allowPrivilegeEscalation: false
      capabilities: {drop: [ALL]}
    resources:
      requests: {cpu: 50m, memory: 64Mi}
      limits: {cpu: 200m, memory: 128Mi}
    volumeMounts:
    - {name: tmp, mountPath: /tmp}
    - {name: cache, mountPath: /var/cache/nginx}
    - {name: run, mountPath: /var/run}
  volumes:
  - {name: tmp, emptyDir: {}}
  - {name: cache, emptyDir: {}}
  - {name: run, emptyDir: {}}
EOF
kubectl apply -f /tmp/hardened-pod.yaml
kubectl get pod hardened-demo -n workshop

# kube-bench CIS compliance
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-eks.yaml
sleep 30
kubectl logs job/kube-bench | head -50
```

**Checklist:**
- [ ] GuardDuty EKS Runtime Monitoring enabled
- [ ] Hardened pod spec with all security fields applied and running
- [ ] kube-bench CIS benchmark job completed

---

# PART 6 — Observability

---

## Topic 6.1 — Prometheus, Grafana & CloudWatch Container Insights

### Theoretical Explanation

**kube-prometheus-stack** (via Helm) installs Prometheus, Grafana, AlertManager, node-exporter, and kube-state-metrics in one chart. This is the 2026 standard observability stack for EKS.

**CloudWatch Container Insights** is the AWS-native alternative — collects metrics and logs from ECS and EKS and presents them in CloudWatch dashboards. Lower setup overhead; less flexibility than self-managed Prometheus.

**Key metrics to alert on:**
- Pod restart rate > 3 in 15 minutes (crashlooping)
- CPU utilization > 85% sustained (right-sizing needed)
- HPA `maxReplicas` reached (capacity planning)
- Node disk pressure
- HTTP error rate > 1% (application health)

### Hands-On Exercise

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace monitoring 2>/dev/null || true

helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n monitoring \
  --set grafana.adminPassword=SecureGrafanaPass123 \
  --set grafana.service.type=LoadBalancer \
  --set prometheus.prometheusSpec.retention=7d

kubectl get pods -n monitoring

GRAFANA_LB=$(kubectl get svc -n monitoring kube-prometheus-stack-grafana \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
echo "Grafana: http://$GRAFANA_LB  (admin / SecureGrafanaPass123)"
echo "Import dashboards: 315 (cluster), 6417 (pods), 1860 (nodes)"

# AlertManager rule
cat > /tmp/alert-rule.yaml <<'EOF'
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: workshop-alerts
  namespace: workshop
  labels: {release: kube-prometheus-stack}
spec:
  groups:
  - name: workshop
    rules:
    - alert: HighPodRestartRate
      expr: increase(kube_pod_container_status_restarts_total{namespace="workshop"}[15m]) > 3
      for: 5m
      labels: {severity: warning}
      annotations:
        summary: "Pod {{ $labels.pod }} is restarting frequently"
EOF
kubectl apply -f /tmp/alert-rule.yaml
```

**Checklist:**
- [ ] kube-prometheus-stack installed in `monitoring` namespace
- [ ] Grafana accessible via LoadBalancer
- [ ] PrometheusRule with restart alert applied
- [ ] Dashboard IDs 315 and 6417 imported

---

## Topic 6.2 — AWS X-Ray & OpenTelemetry Distributed Tracing

### Theoretical Explanation

**AWS Distro for OpenTelemetry (ADOT)** is the recommended approach in 2026 — it sends traces to X-Ray while remaining vendor-neutral. Your app uses the OpenTelemetry SDK (auto-instrumented for most languages); the ADOT Collector sidecar or DaemonSet handles exporting.

**When to use tracing:**
- Debugging slow requests across multiple microservices
- Identifying bottlenecks (which service adds the most latency)
- Correlating errors across service boundaries

### Hands-On Exercise

```bash
# Install ADOT Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml

# ADOT Collector pointing to X-Ray
cat > /tmp/otel-collector.yaml <<EOF
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata: {name: adot, namespace: workshop}
spec:
  mode: DaemonSet
  config: |
    receivers:
      otlp:
        protocols:
          grpc: {endpoint: "0.0.0.0:4317"}
          http: {endpoint: "0.0.0.0:4318"}
    processors:
      batch: {timeout: 1s}
    exporters:
      awsxray:
        region: ${REGION}
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [awsxray]
EOF
kubectl apply -f /tmp/otel-collector.yaml 2>/dev/null || echo "Requires ADOT operator fully installed"

# View X-Ray traces
aws xray get-trace-summaries \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) --region $REGION 2>/dev/null | head -20
```

**Checklist:**
- [ ] ADOT Operator installed
- [ ] OpenTelemetry Collector DaemonSet deployed
- [ ] X-Ray traces queryable via CLI

---

# PART 7 — CI/CD & GitOps

---

## Topic 7.1 — GitHub Actions CI/CD Pipeline

### Theoretical Explanation

**GitHub Actions** with OIDC federation to AWS eliminates long-lived IAM access keys from CI. The pipeline authenticates via a short-lived token, builds a multi-arch image, scans it with Trivy, and deploys to ECS or EKS.

**Complete pipeline flow:**
```
git push → Actions triggered → configure-aws-credentials (OIDC)
→ docker buildx (multi-arch, ECR layer cache)
→ trivy scan (fail on CRITICAL)
→ push to ECR (tagged with git SHA)
→ update ECS service / commit updated K8s manifest (GitOps)
```

### Hands-On Exercise

```bash
# Configure GitHub OIDC provider in AWS
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1 2>/dev/null || true

GITHUB_OIDC_ARN=$(aws iam list-open-id-connect-providers \
  --query "OpenIDConnectProviderList[?contains(Arn,'token.actions.githubusercontent.com')].Arn" \
  --output text)

cat > /tmp/github-role-trust.json <<EOF
{
  "Version":"2012-10-17",
  "Statement":[{"Effect":"Allow",
    "Principal":{"Federated":"${GITHUB_OIDC_ARN}"},
    "Action":"sts:AssumeRoleWithWebIdentity",
    "Condition":{"StringLike":{
      "token.actions.githubusercontent.com:sub":"repo:YOUR-ORG/YOUR-REPO:*"
    },"StringEquals":{
      "token.actions.githubusercontent.com:aud":"sts.amazonaws.com"
    }}
  }]
}
EOF

aws iam create-role --role-name github-actions-ci \
  --assume-role-policy-document file:///tmp/github-role-trust.json 2>/dev/null || true
aws iam attach-role-policy --role-name github-actions-ci \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPowerUser

# GitHub Actions workflow (save as .github/workflows/deploy.yml)
mkdir -p /tmp/gha
cat > /tmp/gha/deploy.yml <<'YAML'
name: Build and Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/github-actions-ci
        aws-region: ap-south-1

    - name: Login to ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - uses: docker/setup-buildx-action@v3

    - name: Build, cache & push multi-arch image
      env:
        REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        REPO: workshop-app
        SHA: ${{ github.sha }}
      run: |
        docker buildx build \
          --platform linux/amd64,linux/arm64 \
          --cache-from type=registry,ref=$REGISTRY/$REPO:cache \
          --cache-to   type=registry,ref=$REGISTRY/$REPO:cache,mode=max \
          --tag $REGISTRY/$REPO:$SHA \
          --push .

    - name: Trivy security scan
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: "${{ steps.login-ecr.outputs.registry }}/workshop-app:${{ github.sha }}"
        severity: CRITICAL,HIGH
        exit-code: 1

    - name: Deploy to ECS
      run: |
        aws ecs update-service \
          --cluster my-cluster --service my-app-service \
          --force-new-deployment --region ap-south-1
YAML
echo "GitHub Actions workflow saved to /tmp/gha/deploy.yml"
```

**Checklist:**
- [ ] GitHub OIDC provider configured in AWS IAM
- [ ] IAM role with GitHub trust policy created
- [ ] Workflow file created with OIDC auth, multi-arch build, Trivy scan, ECS deploy

---

## Topic 7.2 — Blue/Green Deployments with CodeDeploy

### Theoretical Explanation

Blue/Green deploys eliminate deployment risk by running old (blue) and new (green) versions simultaneously, shifting traffic only after health checks pass:

```
Blue tasks (live) →
CodeDeploy creates Green tasks →
ALB routes 10% to Green (bake) →
If healthy: 100% to Green, Blue terminated →
If unhealthy: auto-rollback to Blue in < 60 seconds
```

**Deployment configurations:**
- `CodeDeployDefault.ECSCanary10Percent5Minutes` — 10% for 5 min then 90%
- `CodeDeployDefault.ECSLinear10PercentEvery1Minutes` — gradual shift
- Custom — define your own percentages and intervals

### Hands-On Exercise

```bash
# Create second target group (green)
TG_GREEN_ARN=$(aws elbv2 create-target-group \
  --name my-app-tg-green --protocol HTTP --port 8080 \
  --vpc-id $VPC_ID --target-type ip --health-check-path "/" \
  --query "TargetGroups[0].TargetGroupArn" --output text --region $REGION)

# CodeDeploy application
aws deploy create-application \
  --application-name my-ecs-app \
  --compute-platform ECS --region $REGION 2>/dev/null || true

# CodeDeploy IAM role
aws iam create-role --role-name AWSCodeDeployRoleForECS \
  --assume-role-policy-document \
    '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"codedeploy.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  2>/dev/null || true
aws iam attach-role-policy --role-name AWSCodeDeployRoleForECS \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS 2>/dev/null || true

echo "Blue/Green deployment group — create via console with:"
echo "  App: my-ecs-app"
echo "  ECS cluster: my-cluster, service: my-app-service"
echo "  Config: CodeDeployDefault.ECSCanary10Percent5Minutes"
echo "  Blue TG: my-app-tg | Green TG: my-app-tg-green"
```

**Checklist:**
- [ ] Second target group (green) created
- [ ] CodeDeploy application and IAM role created
- [ ] Blue/Green deployment strategy understood

---

## Topic 7.3 — GitOps with ArgoCD

### Theoretical Explanation

**GitOps:** Git is the single source of truth. Changes to the cluster happen via merged PRs — `kubectl apply` is banned in production. ArgoCD continuously reconciles cluster state with the Git-defined desired state.

**ArgoCD "App of Apps" pattern:** A root Application points to a directory of other Application manifests — each managing a workload or cluster component. One `kubectl apply` bootstraps the entire platform.

**ArgoCD key features:**
- `selfHeal: true` — restores deleted resources within minutes
- `prune: true` — removes resources not in Git
- Sync history — full audit trail of every change
- Rollback — one click to restore any previous state

### Hands-On Exercise

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl wait --for=condition=Available deployment/argocd-server \
  -n argocd --timeout=300s

kubectl patch svc argocd-server -n argocd -p '{"spec":{"type":"LoadBalancer"}}'
ARGOCD_LB=$(kubectl get svc argocd-server -n argocd \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null)
ARGOCD_PASS=$(kubectl get secret argocd-initial-admin-secret \
  -n argocd -o jsonpath='{.data.password}' | base64 -d)
echo "ArgoCD URL: https://$ARGOCD_LB"

# Install ArgoCD CLI
curl -sSL https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 \
  -o argocd && chmod +x argocd && sudo mv argocd /usr/local/bin/

argocd login $ARGOCD_LB --username admin --password $ARGOCD_PASS --insecure

# Deploy the ArgoCD example app from Git
argocd app create guestbook \
  --repo https://github.com/argoproj/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace workshop \
  --sync-policy automated \
  --self-heal --auto-prune

argocd app list
argocd app get guestbook

# Test self-healing
kubectl delete deployment guestbook-ui -n workshop 2>/dev/null || true
echo "Wait 3 minutes — ArgoCD will restore the deleted deployment"
sleep 180
kubectl get deployment guestbook-ui -n workshop 2>/dev/null && echo "✓ Self-healed!"

argocd app history guestbook
```

**Checklist:**
- [ ] ArgoCD installed and accessible
- [ ] ArgoCD CLI installed and logged in
- [ ] `guestbook` app synced from Git
- [ ] Self-healing verified (deleted deployment restored)
- [ ] Sync history shows audit trail

---

## Topic 7.4 — Helm Charts

### Theoretical Explanation

**Helm** is the Kubernetes package manager. Charts package all Kubernetes manifests with Go-template based configuration. Store charts in ECR (OCI format, supported since Helm 3.7).

**Helm + ArgoCD:** ArgoCD natively supports Helm charts as application sources — point ArgoCD at a chart + values file in Git for fully GitOps-managed Helm releases.

**Helmfile** manages multiple Helm releases declaratively in one YAML file — the equivalent of Docker Compose for Helm.

### Hands-On Exercise

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update

# Install Redis with custom values
cat > /tmp/redis-values.yaml <<'EOF'
auth: {enabled: true, password: "RedisPass123"}
architecture: standalone
master:
  persistence: {enabled: true, storageClass: ebs-gp3, size: 5Gi}
  resources: {requests: {cpu: 100m, memory: 128Mi}}
EOF

helm install redis bitnami/redis -f /tmp/redis-values.yaml -n workshop
helm status redis -n workshop
kubectl get pods -n workshop -l app.kubernetes.io/name=redis

# Scaffold your own chart
helm create workshop-app
cat > workshop-app/values.yaml <<EOF
replicaCount: 2
image:
  repository: ${WORKSHOP_URI}
  tag: v1.0
  pullPolicy: IfNotPresent
service: {type: ClusterIP, port: 80, targetPort: 8080}
resources:
  requests: {cpu: 100m, memory: 128Mi}
  limits:   {cpu: 500m, memory: 256Mi}
EOF

helm install workshop workshop-app -n workshop --dry-run | head -40
helm install workshop workshop-app -n workshop

# Push chart to ECR
helm package workshop-app
aws ecr create-repository --repository-name helm-charts --region $REGION 2>/dev/null || true
helm push workshop-app-0.1.0.tgz oci://$ECR_URI/helm-charts

# Install from ECR
helm install workshop-from-ecr \
  oci://$ECR_URI/helm-charts/workshop-app --version 0.1.0 -n workshop

helm uninstall redis workshop -n workshop
```

**Checklist:**
- [ ] Redis installed from Bitnami chart
- [ ] Custom chart scaffolded and installed
- [ ] Chart pushed to ECR in OCI format
- [ ] Chart installed from ECR URI

---

# PART 8 — Cost Optimization

---

## Topic 8.1 — Cost Optimization Strategies

### Theoretical Explanation

**ECR:** ~$0.10/GB/month. Apply lifecycle policies everywhere. Use Pull Through Cache.

**ECS:**
- **FARGATE_SPOT:** 60–90% savings for fault-tolerant, non-critical workloads.
- **Right-size:** If 7-day average CPU < 30%, halve the CPU allocation.
- **Compute Savings Plans:** 1-year commit on predictable Fargate baseline.

**EKS:**
- **Karpenter consolidation** (`consolidateAfter: 30s`) removes underused nodes.
- **EC2 Spot + multiple instance families** in NodePool (never single instance type).
- **Graviton ARM64** — 20% cheaper; use multi-arch images.
- **VPA recommendations** for right-sizing resource requests.
- **EKS Auto Mode** includes automatic cost optimization.

**Networking:** VPC endpoints eliminate NAT Gateway charges for AWS API traffic.

### Hands-On Exercise

```bash
# Find ECR repos without lifecycle policies
echo "=== ECR repos without lifecycle policies ==="
aws ecr describe-repositories --region $REGION \
  --query "repositories[].repositoryName" --output text | tr '\t' '\n' | while read repo; do
    aws ecr get-lifecycle-policy --repository-name "$repo" --region $REGION \
      &>/dev/null || echo "⚠️  No lifecycle policy: $repo"
  done

# ECS CPU utilization (7-day average)
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=my-app-service Name=ClusterName,Value=my-cluster \
  --start-time $(date -d '7 days ago' +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -v-7d +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date +%Y-%m-%dT%H:%M:%S) \
  --period 604800 --statistics Average --region $REGION \
  --query "Datapoints[0].Average" 2>/dev/null

# VPA for EKS right-sizing
helm repo add fairwinds-stable https://charts.fairwinds.com/stable && helm repo update
helm install vpa fairwinds-stable/vpa -n vpa --create-namespace \
  --set recommender.enabled=true \
  --set updater.enabled=false \
  --set admission-controller.enabled=false

cat > /tmp/vpa.yaml <<'EOF'
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata: {name: my-app-vpa, namespace: workshop}
spec:
  targetRef: {apiVersion: apps/v1, kind: Deployment, name: my-app}
  updatePolicy: {updateMode: "Off"}
EOF
kubectl apply -f /tmp/vpa.yaml 2>/dev/null || true

# Savings estimate
echo "=== Savings Potential ==="
echo "Fargate Spot vs On-Demand: ~70% savings"
echo "EC2 Spot vs On-Demand:     ~60-90% savings"
echo "Graviton ARM64 vs x86:     ~20% savings"
echo "Combined potential:        up to 85% savings"

# Cost Explorer for container services
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d 2>/dev/null || date -v-30d +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --metrics "UnblendedCost" \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Container Service","Amazon Elastic Kubernetes Service"]}}' \
  --query "ResultsByTime[0].Total" --region $REGION 2>/dev/null
```

**Checklist:**
- [ ] ECR repos without lifecycle policies identified
- [ ] ECS CPU utilization checked for right-sizing
- [ ] VPA installed for EKS resource recommendations
- [ ] Cost optimization levers understood (Spot, Graviton, Karpenter consolidation)


---

# APPENDIX — Quick Reference

## Full CLI Command Cheat Sheet

### ECR

| Action | Command |
|--------|---------|
| Login | `aws ecr get-login-password \| docker login --username AWS --password-stdin <uri>` |
| Create repo | `aws ecr create-repository --repository-name <name> --region <region>` |
| List repos | `aws ecr describe-repositories` |
| List images | `aws ecr list-images --repository-name <name>` |
| Delete image | `aws ecr batch-delete-image --repository-name <name> --image-ids imageTag=<tag>` |
| Enable immutability | `aws ecr put-image-tag-mutability --repository-name <name> --image-tag-mutability IMMUTABLE` |
| Apply lifecycle policy | `aws ecr put-lifecycle-policy --repository-name <name> --lifecycle-policy-text file://policy.json` |
| Lifecycle preview | `aws ecr get-lifecycle-policy-preview --repository-name <name>` |
| Start image scan | `aws ecr start-image-scan --repository-name <name> --image-id imageTag=latest` |
| Get scan findings | `aws ecr describe-image-scan-findings --repository-name <name> --image-id imageTag=latest` |
| Set repo policy | `aws ecr set-repository-policy --repository-name <name> --policy-text file://policy.json` |
| Pull-through cache | `aws ecr create-pull-through-cache-rule --ecr-repository-prefix dockerhub --upstream-registry-url registry-1.docker.io` |
| Replication config | `aws ecr put-replication-configuration --replication-configuration file://config.json` |

### ECS

| Action | Command |
|--------|---------|
| Create cluster | `aws ecs create-cluster --cluster-name <name>` |
| Register task def | `aws ecs register-task-definition --cli-input-json file://task.json` |
| List task defs | `aws ecs list-task-definitions` |
| Run task | `aws ecs run-task --cluster <name> --task-definition <name> --launch-type FARGATE ...` |
| List tasks | `aws ecs list-tasks --cluster <name>` |
| Describe task | `aws ecs describe-tasks --cluster <name> --tasks <arn>` |
| Create service | `aws ecs create-service --cluster <name> --service-name <name> ...` |
| Update service | `aws ecs update-service --cluster <name> --service <name> --desired-count <n>` |
| Force redeploy | `aws ecs update-service --cluster <name> --service <name> --force-new-deployment` |
| Shell into container | `aws ecs execute-command --cluster <name> --task <arn> --container <name> --interactive --command "/bin/sh"` |
| Tail logs | `aws logs tail /ecs/<log-group> --follow` |
| Create scheduled rule | `aws events put-rule --name <name> --schedule-expression "cron(...)"` |
| Wait service stable | `aws ecs wait services-stable --cluster <name> --services <name>` |

### EKS / kubectl

| Action | Command |
|--------|---------|
| Create cluster | `eksctl create cluster --name <name> --region <region>` |
| Connect kubectl | `aws eks update-kubeconfig --name <name> --region <region>` |
| Get nodes | `kubectl get nodes -o wide` |
| Get all pods | `kubectl get pods --all-namespaces` |
| Apply manifest | `kubectl apply -f <file>.yaml` |
| Describe resource | `kubectl describe pod/deployment/service <name> -n <ns>` |
| View logs | `kubectl logs -f <pod-name> -c <container> -n <ns>` |
| Shell into pod | `kubectl exec -it <pod-name> -n <ns> -- /bin/sh` |
| Rollout status | `kubectl rollout status deployment/<name> -n <ns>` |
| Rollback | `kubectl rollout undo deployment/<name> -n <ns>` |
| Scale deployment | `kubectl scale deployment <name> --replicas=5 -n <ns>` |
| Set image | `kubectl set image deployment/<name> <container>=<image>:<tag> -n <ns>` |
| Get events | `kubectl get events --sort-by='.lastTimestamp' -n <ns>` |
| Resource usage | `kubectl top nodes && kubectl top pods -n <ns>` |
| Drain node | `kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data` |
| Add-on | `aws eks create-addon --cluster-name <name> --addon-name <addon>` |
| Add IAM identity | `eksctl create iamidentitymapping --cluster <name> --arn <role-arn> --group system:masters` |
| Delete cluster | `eksctl delete cluster --name <name>` |

---

## 10-Week Learning Path

```
Week 1  ── Docker basics → Multistage → Multi-arch → Layer caching → Dockerfile security → Trivy/SBOM
Week 2  ── ECR: repo creation → auth → push/pull → lifecycle policies → image scanning (Inspector)
Week 3  ── ECR advanced: tag immutability → KMS encryption → VPC endpoints → pull-through cache → signing
Week 4  ── ECS Fargate: task defs → IAM roles → clusters → services + ALB → health checks → scaling
Week 5  ── ECS advanced: secrets (SM/SSM) → ECS Exec → FireLens → EFS → Service Connect → scheduled tasks
Week 6  ── Kubernetes: EKS cluster → kubectl → Deployments → Services → Ingress (AWS LBC)
Week 7  ── EKS workloads: IRSA/Pod Identity → HPA → Karpenter → StatefulSets → DaemonSets → probes/PDB
Week 8  ── EKS ops: NetworkPolicies → RBAC → ConfigMaps/Secrets → EBS/EFS CSI → External Secrets Operator
Week 9  ── Security + Observability: Kyverno → GuardDuty → Prometheus/Grafana → X-Ray/ADOT
Week 10 ── CI/CD + GitOps: GitHub Actions (OIDC) → Blue/Green → ArgoCD → Helm → Cost optimization
```

---

## Common Mistakes & Fixes

| Mistake | Fix |
|---------|-----|
| ECS task fails to pull image | Task **execution role** missing `AmazonEC2ContainerRegistryReadOnly` |
| ECS task exits immediately | Check CloudWatch logs; add `CMD ["sleep","infinity"]` for debug |
| Using `latest` tag in production | Tag with git SHA or semver; enable ECR tag immutability |
| No lifecycle policy on ECR repos | Apply policies at creation time via Terraform/CDK |
| `kubectl` shows "Unauthorized" | Re-run `aws eks update-kubeconfig` with the correct IAM identity |
| EKS pods in `Pending` state | Check node capacity, resource requests/limits, security groups |
| ECR auth token expired | Token valid 12h only — re-run `get-login-password` |
| Cross-account ECR pull fails | Add resource-based policy on ECR repo granting the other account's role |
| Fargate task can't reach internet | Use NAT Gateway for private subnets, or assign public IP in public subnet |
| EKS upgrade fails | Scan for deprecated APIs with `pluto` before upgrading |
| ECS task CPU over-provisioned | Cut CPU if CloudWatch average < 20% for 7 days |
| EKS pods evicted during node upgrade | Set PodDisruptionBudgets before draining nodes |
| No resource requests on K8s pods | Pods without requests are evicted first under node pressure |
| Secrets in plaintext env vars | Use Secrets Manager (ECS `valueFrom`) or External Secrets Operator (EKS) |
| Single AZ ECS service | Spread tasks across ≥ 2 AZs using `spread` placement strategy |
| EKS Network Policies not enforced | Enable VPC CNI network policy: `ENABLE_NETWORK_POLICY_CONTROLLER=true` |
| Spot node pool — one instance type | Use multiple compatible families in Karpenter NodePool |
| Docker Hub rate limiting in CI | Set up ECR pull-through cache |
| Containers running as root | `USER nonroot` in Dockerfile + `runAsNonRoot: true` in pod securityContext |
| No image signing in production | Implement Cosign or Notation + Kyverno admission policy |
| Missing add-on updates post-upgrade | Run `aws eks update-addon` for every add-on after control plane upgrade |
| Long-lived IAM keys in CI/CD | Replace with OIDC federation (GitHub Actions / CodeBuild) |
| No SBOM generated | Run Syft in CI; attach SBOM to ECR image with Cosign |

---

# Capstone Projects

These projects integrate knowledge from the entire learning resource. Complete them in order — Projects 1–3 build ECS skills, Projects 4–6 build EKS skills. Project 3 bridges both.

---

## Capstone Project 1 — Full-Stack Web App on ECS Fargate

**Difficulty:** ⭐⭐ Beginner–Intermediate | **Estimated Time:** 4–6 hours

**Services:** ECR, ECS Fargate, ALB, RDS PostgreSQL, Secrets Manager, CloudWatch, GitHub Actions

### Description

Deploy a containerised full-stack web application (frontend + REST API) to ECS Fargate. The API reads from an RDS PostgreSQL database with credentials injected from Secrets Manager. A CI/CD pipeline in GitHub Actions automatically builds, scans, and deploys on every push to `main`.

### Architecture

```
Internet → ALB → ECS Frontend Service (Fargate) → ECS API Service (Fargate)
                                                         → RDS PostgreSQL
GitHub push → Actions (OIDC) → ECR push → ECS rolling update
```

### Tasks

```bash
# 1. ECR repos with lifecycle policies
aws ecr create-repository --repository-name capstone1/frontend --region $REGION
aws ecr create-repository --repository-name capstone1/api --region $REGION
aws ecr put-lifecycle-policy --repository-name capstone1/frontend \
  --lifecycle-policy-text file:///tmp/lifecycle-policy.json --region $REGION

# 2. RDS PostgreSQL (Free Tier)
aws rds create-db-instance \
  --db-instance-identifier capstone1-db \
  --db-instance-class db.t3.micro \
  --engine postgres --engine-version 16 \
  --master-username dbadmin \
  --master-user-password "$(openssl rand -base64 16)" \
  --allocated-storage 20 \
  --no-multi-az \
  --publicly-accessible false \
  --region $REGION

# 3. Store DB password in Secrets Manager
aws secretsmanager create-secret \
  --name /capstone1/prod/db-password \
  --secret-string "your-generated-password" --region $REGION

# 4. ECS cluster + 2 services (frontend + API)
#    Each service: Fargate, 2 tasks, ALB target group
#    API task definition: inject DB_HOST, DB_PASSWORD via secrets array

# 5. GitHub Actions workflow
#    - OIDC auth to AWS
#    - Multi-arch build + ECR layer cache
#    - Trivy scan (fail on CRITICAL)
#    - Push with git SHA tag
#    - ECS update-service force-new-deployment

# 6. Auto scaling: target 70% CPU, min=2, max=10 for both services

# 7. CloudWatch Dashboard with CPU, memory, task count, ALB 5xx rate
```

### Validation Checklist

- [ ] Frontend accessible via ALB DNS on port 80
- [ ] API connects to RDS; database credentials never appear in task definition JSON
- [ ] GitHub push triggers pipeline; app updates within 5 minutes
- [ ] Kill one task — ECS replaces it automatically (service maintains desired count)
- [ ] Inject a CRITICAL CVE in base image — Trivy fails the build before deployment
- [ ] Auto scaling: run a load test; service scales out to ≥ 4 tasks; scales back after load stops
- [ ] CloudWatch Dashboard shows live metrics for both services

**Learning Objectives:** ECR lifecycle management, ECS service architecture with ALB, Secrets Manager injection, GitHub Actions CI/CD with OIDC, ECS auto scaling.

---

## Capstone Project 2 — Production Microservices on EKS

**Difficulty:** ⭐⭐⭐ Intermediate | **Estimated Time:** 8–12 hours

**Services:** EKS, ECR, AWS LBC, ArgoCD, IRSA/Pod Identity, HPA, Karpenter, Prometheus, Grafana, External Secrets Operator

### Description

Deploy a three-service microservices application (frontend, API, worker) to EKS using GitOps. All application secrets come from Secrets Manager via External Secrets Operator. ArgoCD manages all deployments from Git. HPA and Karpenter handle scaling. Prometheus and Grafana provide observability.

### Architecture

```
Internet → ALB (Ingress) → frontend (/) and api (/api)
                                    ↓
                              Worker (SQS consumer via KEDA)
                                    ↓
                              PostgreSQL (StatefulSet + EBS)

Git commit → ArgoCD detects → applies to EKS → Kyverno validates
Secrets Manager → External Secrets Operator → K8s Secrets → Pods
```

### Tasks

```bash
# 1. EKS cluster with Karpenter NodePool (mixed Spot/On-Demand, ARM64+x86)
eksctl create cluster --name capstone2 --region $REGION --version 1.31 \
  --managed --with-oidc --node-type t3.medium --nodes 2

# 2. AWS Load Balancer Controller + Ingress with path routing
# / → frontend service, /api → api service

# 3. Pod Identity association for API (Secrets Manager access)
aws eks create-pod-identity-association \
  --cluster-name capstone2 --namespace production \
  --service-account api-sa \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/capstone2-api-role --region $REGION

# 4. External Secrets Operator: sync DB creds from Secrets Manager
# SecretStore (IRSA) + ExternalSecret → K8s Secret

# 5. ArgoCD: manage all workloads from Git repo
#    Ingress + Deployments (3) + HPA (2) + PDB (2) + NetworkPolicies + StatefulSet

# 6. HPA: CPU 50%, min=2, max=10 for frontend and API

# 7. Kyverno policies:
#    - require-non-root
#    - disallow-latest-tag
#    - require-resource-limits

# 8. Prometheus + Grafana (dashboards 315 and 6417)
#    AlertManager rule: PodRestartRate > 3 in 15 min

# 9. KEDA ScaledObject for worker (SQS queue depth, scale-to-zero)
```

### Validation Checklist

- [ ] All three services accessible via Ingress path routing
- [ ] Commit to Git → ArgoCD syncs within 3 minutes, no `kubectl apply` needed
- [ ] API reads credentials from Secrets Manager via Pod Identity — zero hardcoded values
- [ ] Delete the API Deployment → ArgoCD self-heals within 3 minutes
- [ ] CPU load test → HPA scales API from 2 to ≥ 5 replicas
- [ ] Drain a node → PDB keeps all services live during drain; Karpenter provisions replacement
- [ ] Push pod with `runAsUser: 0` → Kyverno blocks it at admission
- [ ] Grafana shows live CPU, memory, and request rate per pod
- [ ] Send 50 messages to SQS → KEDA scales worker from 0 to ≥ 5 replicas

**Learning Objectives:** EKS GitOps with ArgoCD, Pod Identity, External Secrets Operator, HPA + Karpenter, Kyverno admission policies, Prometheus/Grafana observability, KEDA event-driven scaling.

---

## Capstone Project 3 — Zero-Downtime CI/CD Pipeline

**Difficulty:** ⭐⭐⭐ Intermediate | **Estimated Time:** 5–7 hours

**Services:** ECR, ECS, CodePipeline, CodeBuild, CodeDeploy (Blue/Green), ALB, SNS, CloudWatch Alarms

### Description

Build an end-to-end CI/CD pipeline that achieves zero-downtime deployments using Blue/Green strategy. A code push triggers CodePipeline, which runs automated tests and a vulnerability scan in CodeBuild, sends an approval email, then CodeDeploy shifts traffic gradually from blue to green — rolling back automatically if health checks fail.

### Pipeline Flow

```
GitHub PR merged → CodePipeline triggered automatically
→ CodeBuild: lint + test + docker build + Trivy scan + ECR push (git SHA tag)
→ Manual Approval gate (SNS email notification to on-call)
→ CodeDeploy Blue/Green to ECS:
    → Green tasks start + register with ALB target group
    → ALB shifts 10% traffic to green (5-min bake)
    → Monitor CloudWatch alarm (HTTP 5xx rate > 1%)
    → If healthy: shift 100% to green; terminate blue tasks
    → If unhealthy: auto-rollback to blue in < 60 seconds
→ CloudWatch alarm + SNS notification on success/failure
```

### Tasks

```bash
# 1. ECS service with CODE_DEPLOY deployment controller
aws ecs create-service \
  --cluster my-cluster --service-name capstone3-svc \
  --task-definition my-app-task --desired-count 3 \
  --deployment-controller '{"type":"CODE_DEPLOY"}' \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=my-app,containerPort=8080" \
  --region $REGION

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
    - docker buildx build --platform linux/amd64,linux/arm64 -t $ECR_URI/workshop-app:$IMAGE_TAG --push .
  post_build:
    commands:
    - trivy image --severity CRITICAL --exit-code 1 $ECR_URI/workshop-app:$IMAGE_TAG
    - printf '[{"name":"my-app","imageUri":"%s"}]' $ECR_URI/workshop-app:$IMAGE_TAG > imagedefinitions.json
artifacts:
  files: [imagedefinitions.json, appspec.yaml, taskdef.json]
EOF

# 3. SNS topic + email subscription for approval gate
aws sns create-topic --name pipeline-approval --region $REGION
aws sns subscribe --topic-arn <arn> --protocol email \
  --notification-endpoint oncall@example.com --region $REGION

# 4. CodePipeline: Source → Build → ManualApproval → Deploy

# 5. Test auto-rollback: intentionally break /health endpoint in a commit
#    Deploy → watch CodeDeploy detect health check failure → rollback to blue

# 6. CloudWatch alarm: HTTP 5xx rate > 1% → trigger rollback
aws cloudwatch put-metric-alarm \
  --alarm-name capstone3-5xx-alarm \
  --metric-name HTTPCode_Target_5XX_Count \
  --namespace AWS/ApplicationELB \
  --statistic Sum \
  --period 60 --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions <rollback-action-arn> \
  --region $REGION
```

### Validation Checklist

- [ ] Code push to `main` triggers pipeline automatically — no manual steps
- [ ] Approval email received before production deployment proceeds
- [ ] Old (blue) version continues serving 100% traffic while green tasks warm up
- [ ] Traffic shifts: 10% → 5-minute bake → 100% (visible in ALB target group metrics)
- [ ] Break `/health` in a commit → pipeline detects failure → rolls back to blue in < 60 seconds
- [ ] Zero requests dropped during successful deployment (verify with load test during deploy)
- [ ] Full audit trail visible in CodeDeploy console (who approved, when, which version)

**Learning Objectives:** End-to-end CI/CD pipeline, Blue/Green deployment with CodeDeploy, automatic rollback, human approval gates, CloudWatch-driven rollback triggers.

---

## Capstone Project 4 — Secure Containerised Platform

**Difficulty:** ⭐⭐⭐⭐ Advanced | **Estimated Time:** 6–8 hours

**Services:** EKS, ECR (KMS + signing), Kyverno, GuardDuty, VPC Endpoints, External Secrets Operator, kube-bench

### Description

Build a security-hardened container platform where every component enforces security at a different layer — from image creation through to runtime. No unsigned images can be deployed. No container can run as root. No secrets exist in plaintext anywhere. All AWS API traffic stays within the VPC.

### Security Stack

```
ECR:      KMS-encrypted + tag-immutable + image signing (Cosign or Notation)
Build:    Multi-arch + Trivy scan + SBOM generated + signature attached
EKS:      Private endpoint + KMS secret encryption + CloudTrail API logging
Pods:     non-root + read-only rootfs + no privilege escalation + all capabilities dropped
Secrets:  External Secrets Operator → Secrets Manager (zero plaintext in Git or K8s YAML)
Network:  deny-all default + explicit allow + VPC Endpoints for all AWS APIs
Policy:   Kyverno enforces all security standards at admission time
Runtime:  GuardDuty EKS Runtime Monitoring (eBPF agent on every node)
Audit:    kube-bench CIS benchmark validation
```

### Tasks

```bash
# 1. KMS-encrypted ECR repos with IMMUTABLE tags
aws ecr create-repository \
  --repository-name capstone4/app \
  --image-tag-mutability IMMUTABLE \
  --encryption-configuration "encryptionType=KMS,kmsKey=${KMS_KEY_ARN}" \
  --region $REGION

# 2. Sign all images in CI with Cosign + KMS
cosign sign --key "awskms:///alias/cosign-signing" \
  --tlog-upload=false \
  "$ECR_URI/capstone4/app:$GIT_SHA"

# 3. EKS cluster: KMS envelope encryption for Secrets
eksctl create cluster --name capstone4 \
  --encryption-config '[{"provider":{"keyArn":"<kms-key-arn>"},"resources":["secrets"]}]' \
  --region $REGION

# 4. GuardDuty EKS Runtime Monitoring
aws guardduty create-detector --enable \
  --features '[{"Name":"EKS_RUNTIME_MONITORING","Status":"ENABLED"}]' --region $REGION

# 5. Kyverno policies (Enforce mode):
#    - require-image-signature (cosign verify before admit)
#    - require-non-root (runAsNonRoot: true)
#    - disallow-privilege-escalation
#    - require-read-only-rootfs
#    - require-resource-limits
#    - disallow-latest-tag
cat > /tmp/kyverno-signature.yaml <<EOF
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata: {name: require-image-signature}
spec:
  validationFailureAction: Enforce
  background: false
  rules:
  - name: check-image-signature
    match:
      any:
      - resources: {kinds: [Pod], namespaces: [production]}
    verifyImages:
    - imageReferences: ["${ECR_URI}/capstone4/*"]
      attestors:
      - entries:
        - keys:
            kms: "awskms:///alias/cosign-signing"
EOF
kubectl apply -f /tmp/kyverno-signature.yaml

# 6. External Secrets Operator — all secrets from Secrets Manager
#    Zero secrets in Git or Kubernetes manifests

# 7. VPC Endpoints: ECR API, ECR DKR, S3, CloudWatch, SSM, STS
#    (private cluster: no NAT Gateway, all AWS traffic via endpoints)

# 8. Network Policies: deny-all default + explicit allow per service

# 9. Run kube-bench CIS benchmark
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job-eks.yaml
sleep 60 && kubectl logs job/kube-bench | grep -E "PASS|FAIL|WARN" | head -30
```

### Validation Checklist

- [ ] Push unsigned image → Kyverno admission webhook rejects the pod
- [ ] Run pod as root → Kyverno rejects it
- [ ] Run pod with `allowPrivilegeEscalation: true` → Kyverno rejects it
- [ ] `kubectl exec` into a pod → session appears in CloudTrail audit log
- [ ] VPC flow logs show zero traffic from pod subnet to ECR via internet (all via VPC endpoint)
- [ ] GuardDuty raises finding when `nmap` or `netcat` is run inside a container
- [ ] Rotate secret in Secrets Manager → External Secrets syncs new value to K8s Secret within 1 hour
- [ ] kube-bench scores > 80% PASS on EKS CIS benchmark
- [ ] `kubectl get secret <name> -o yaml` shows `data` as base64 (KMS encryption is transparent)

**Learning Objectives:** Defense-in-depth security, image signing with policy enforcement, runtime threat detection, VPC isolation for AWS APIs, CIS compliance validation, zero-trust secrets management.

---

## Capstone Project 5 — Multi-Region Disaster Recovery

**Difficulty:** ⭐⭐⭐⭐ Advanced | **Estimated Time:** 8–10 hours

**Services:** ECS/EKS, ECR Replication, RDS Cross-Region Replica, Route 53 Failover, CloudWatch Alarms, SNS

### Description

Build a multi-region active-passive disaster recovery setup. The primary region (`ap-south-1`) runs the full workload. The secondary region (`ap-southeast-1`) has standby infrastructure at zero scale, pre-positioned images, and a read replica. Route 53 health checks detect primary failure and automatically shift traffic to the secondary — with a target RTO of under 10 minutes and RPO under 5 seconds.

### Architecture

```
Primary (ap-south-1)                Secondary (ap-southeast-1)
  ECS Service (3 tasks)   ←─ECR replication──→   ECS Service (0 tasks, standby)
  RDS PostgreSQL           ←─read replica──────→  RDS Read Replica
  ALB                                             ALB
    ↑                                               ↑
    └──── Route 53 Failover (health check) ─────────┘
                    ↑ (automatic on primary ALB 503)
```

### Tasks

```bash
# 1. ECR cross-region replication (all prod repos to ap-southeast-1)
aws ecr put-replication-configuration \
  --replication-configuration '{"rules":[{"destinations":[
    {"region":"ap-southeast-1","registryId":"'"$ACCOUNT_ID"'"}
  ]}]}' --region $REGION

# 2. RDS cross-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier capstone5-db-replica \
  --source-db-instance-identifier arn:aws:rds:${REGION}:${ACCOUNT_ID}:db:capstone1-db \
  --db-instance-class db.t3.micro \
  --region ap-southeast-1

# 3. Secondary ECS infrastructure (0 desired tasks — standby)
aws ecs create-cluster --cluster-name secondary-cluster --region ap-southeast-1
aws ecs create-service \
  --region ap-southeast-1 --cluster secondary-cluster \
  --service-name app-standby --task-definition my-app-task \
  --desired-count 0 --launch-type FARGATE \
  --load-balancers "targetGroupArn=<secondary-tg-arn>,containerName=my-app,containerPort=8080"

# 4. Route 53 health check + failover DNS
aws route53 create-health-check \
  --health-check-config '{
    "FullyQualifiedDomainName":"<primary-alb-dns>",
    "Port":80,"Type":"HTTP",
    "ResourcePath":"/health",
    "RequestInterval":10,"FailureThreshold":2
  }' --caller-reference "primary-$(date +%s)"

# 5. DR Runbook — failover procedure
# a. Promote read replica (takes 5-10 min)
aws rds promote-read-replica \
  --db-instance-identifier capstone5-db-replica --region ap-southeast-1

# b. Scale ECS service in secondary
aws ecs update-service \
  --region ap-southeast-1 --cluster secondary-cluster \
  --service app-standby --desired-count 3

# c. Route 53 automatically fails over on primary health check failure

# 6. Failover drill: simulate primary failure
#    Return HTTP 503 from primary /health endpoint
#    Measure: time from failure → Route 53 completion
#    Target: < 10 minutes RTO

# 7. CloudWatch alarm: ReplicaLag > 5 seconds → SNS alert
aws cloudwatch put-metric-alarm \
  --alarm-name capstone5-replica-lag \
  --namespace AWS/RDS --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=capstone5-db-replica \
  --statistic Maximum --period 60 --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --region ap-southeast-1
```

### Validation Checklist

- [ ] ECR image pushed to primary → visible in `ap-southeast-1` within 5 minutes
- [ ] RDS `ReplicaLag` CloudWatch metric stays below 5 seconds during normal operation
- [ ] Primary returns HTTP 503 → Route 53 fails over automatically (no manual DNS change)
- [ ] Full failover (promote replica + scale ECS + DNS propagation) completes in < 10 minutes
- [ ] App accessible in secondary region after failover — reads committed data (no data loss)
- [ ] Failback procedure documented: re-sync data, re-replicate, flip Route 53 back
- [ ] Measured RTO (recovery time objective) and RPO (recovery point objective) documented

**Learning Objectives:** Multi-region architecture, ECR replication, RDS read replicas and promotion, Route 53 health checks and failover routing, DR runbook design and testing.

---

## Capstone Project 6 — GitOps-Driven EKS Platform (End-to-End)

**Difficulty:** ⭐⭐⭐⭐⭐ Expert | **Estimated Time:** 10–14 hours

**Services:** EKS, ECR, ArgoCD, Helm, Karpenter, Prometheus, Grafana, External Secrets Operator, Kyverno, Terraform, GitHub Actions

### Description

Build a complete, production-grade EKS platform where **everything is in Git** — from the EKS cluster itself (Terraform) to every application and cluster add-on (Helm charts via ArgoCD). After the initial bootstrap, `kubectl apply` is banned in production. Every change flows through a Git PR with CI validation.

### Philosophy: Everything in Git

```
platform-gitops/
├── terraform/              ← EKS cluster, VPC, IAM via Terraform
├── bootstrap/              ← ArgoCD "App of Apps" root application
├── cluster-addons/         ← AWS LBC, External Secrets, Karpenter, Kyverno (Helm via ArgoCD)
├── monitoring/             ← kube-prometheus-stack, Grafana dashboards as ConfigMaps
├── security/               ← Kyverno policies, NetworkPolicies
└── apps/
    ├── dev/                ← Dev environment Helm values
    └── prod/               ← Production Helm values
```

### Tasks

```bash
# Phase A: Infrastructure as Code
# EKS cluster via Terraform (all future infra changes via PR → plan → apply)
cd ~/terraform-eks
terraform apply -auto-approve

aws eks update-kubeconfig --name tf-eks-cluster --region $REGION

# Phase B: GitOps bootstrap (one-time only)
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCD "App of Apps" — after this, all changes come from Git
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
    automated:
      prune: true
      selfHeal: true
EOF
kubectl apply -f bootstrap/root-app.yaml
# From this point: git commit = cluster change. kubectl apply = BANNED in prod.

# Phase C: Add cluster add-ons via Git commits (ArgoCD Application YAMLs for each)
# Commit Helm-based Applications pointing at:
# → AWS Load Balancer Controller
# → External Secrets Operator
# → Karpenter NodePool + EC2NodeClass
# → Kyverno + policy bundle
# → kube-prometheus-stack + Grafana dashboards

# Phase D: Deploy application via Helm chart in Git
# → Package Helm chart → push to ECR
# → Commit ArgoCD Application YAML pointing at chart + prod values
# → ArgoCD deploys + keeps in sync

# Phase E: CI/CD (GitHub Actions)
# Push code → build multi-arch image → Trivy scan → push to ECR
# → update image tag in Git (values.yaml) via PR
# → ArgoCD detects change → deploys

# Phase F: Validation
argocd app list                                          # All apps: Synced + Healthy
kubectl get nodes                                         # Karpenter manages lifecycle
kubectl get policyreport --all-namespaces                 # Kyverno compliance report
kubectl get externalsecrets --all-namespaces             # Secrets synced
kubectl top pods --all-namespaces                         # Resource usage
```

### Validation Checklist

- [ ] **Zero `kubectl apply` in production** — every change flows via Git PR → ArgoCD
- [ ] Delete a Deployment manually → ArgoCD restores it within 3 minutes (selfHeal)
- [ ] Merge a PR adding a new Service → ArgoCD deploys it automatically, no manual step
- [ ] Scale to 20 replicas → Karpenter provisions new nodes within 90 seconds
- [ ] Apply a non-compliant pod (root user) → Kyverno rejects it at admission
- [ ] ArgoCD sync history provides full audit trail of every change and who made it
- [ ] Drain any node → Karpenter replaces it, pods reschedule, zero downtime (PDBs protect)
- [ ] Rotate a secret in Secrets Manager → External Secrets syncs within configured interval
- [ ] Grafana dashboards show live metrics; AlertManager fires on test alert
- [ ] Terraform `plan` accurately reflects the running cluster state (no drift)

**Learning Objectives:** Platform engineering, GitOps end-to-end, Infrastructure as Code, ArgoCD App of Apps pattern, Karpenter node lifecycle, complete security stack, multi-layer observability.

---

## Capstone Summary Table

| # | Project | Key Skills | Difficulty | Est. Time |
|---|---------|-----------|------------|-----------|
| 1 | Full-Stack Web App on ECS Fargate | ECR, ECS, ALB, RDS, Secrets, CI/CD | ⭐⭐ | 4–6 hrs |
| 2 | Production Microservices on EKS | EKS, GitOps, Pod Identity, HPA, Karpenter, Observability | ⭐⭐⭐ | 8–12 hrs |
| 3 | Zero-Downtime CI/CD Pipeline | CodePipeline, Blue/Green, Auto-rollback, Approval Gate | ⭐⭐⭐ | 5–7 hrs |
| 4 | Secure Containerised Platform | Image signing, Kyverno, GuardDuty, VPC Endpoints, kube-bench | ⭐⭐⭐⭐ | 6–8 hrs |
| 5 | Multi-Region Disaster Recovery | ECR Replication, RDS Replica, Route 53 Failover, DR Runbook | ⭐⭐⭐⭐ | 8–10 hrs |
| 6 | GitOps-Driven EKS Platform | ArgoCD, Karpenter, Helm, Terraform, Kyverno, ADOT | ⭐⭐⭐⭐⭐ | 10–14 hrs |

> **Suggested order:** Projects 1 and 3 = ECS track. Projects 2, 4, 5, 6 = EKS track. Project 2 is the bridge — complete it before attempting 4, 5, or 6. Project 6 requires completing all others first.

---

*This guide reflects 2026 industry best practices. The container ecosystem evolves rapidly — always check the latest AWS documentation, EKS release notes, and CNCF project changelogs for updates beyond this guide.*

*Happy learning! Practice every phase completely before advancing. Real-world AWS skills come from doing, not just reading.*
