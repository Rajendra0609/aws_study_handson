# 🔐 AWS IAM Lab — Part 2: Real-World Production Additions
### Everything Missing from a Basic Lab That Actually Happens in Startups

---

## ⚠️ What Was Missing in Part 1

| # | Missing Topic | Why It Matters in Real Life |
|---|--------------|----------------------------|
| 1 | S3 Bucket Policies | Users alone aren't enough — buckets need their own policy |
| 2 | Instance Profiles | How EC2 actually uses an IAM Role (not automatic) |
| 3 | CI/CD Role (GitHub Actions) | Deployments need AWS access without hardcoded keys |
| 4 | Permission Boundaries | Prevent even admins from going rogue |
| 5 | CloudTrail Audit Logging | Required by law in many industries — track every action |
| 6 | IAM Access Analyzer | Find who has access they shouldn't |
| 7 | Credential Report | See all users, keys, MFA status in one report |
| 8 | Access Advisor | Find unused permissions and shrink them |
| 9 | AWS Config Rules | Automated compliance checks |
| 10 | Break-Glass Emergency Access | What if the admin is unavailable? |
| 11 | Tag-Based Access Control (ABAC) | Scale beyond group policies |
| 12 | Secrets Manager | No more hardcoded DB passwords anywhere |
| 13 | Multi-Environment Setup | Dev / Staging / Prod must be separate accounts |
| 14 | IAM Identity Center (SSO) | Real companies don't use IAM users — they use SSO |
| 15 | CloudWatch Alarms on IAM Events | Alert when someone deletes a policy or disables MFA |
| 16 | Key Rotation Automation | Access keys must rotate every 90 days |

---

## LAB 11 — S3 Bucket Policies (Resource-Based Policies)

> IAM user policies alone are NOT enough. S3 buckets need their own policy too.
> Both must allow for access to work. Either one denying = access denied.

### The Frontend Bucket — Only Frontend Devs Can Upload

```bash
# First create the bucket
aws s3api create-bucket \
  --bucket technova-frontend \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create bucket policy — only grp-frontend-devs users + CloudFront can access
cat > s3-frontend-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFrontendDevsOnly",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::${ACCOUNT_ID}:user/sneha-frontend",
          "arn:aws:iam::${ACCOUNT_ID}:user/karthik-frontend"
        ]
      },
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::technova-frontend",
        "arn:aws:s3:::technova-frontend/*"
      ]
    },
    {
      "Sid": "DenyEveryoneElse",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::technova-frontend",
        "arn:aws:s3:::technova-frontend/*"
      ],
      "Condition": {
        "StringNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::${ACCOUNT_ID}:user/sneha-frontend",
            "arn:aws:iam::${ACCOUNT_ID}:user/karthik-frontend",
            "arn:aws:iam::${ACCOUNT_ID}:user/arjun-admin"
          ]
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket technova-frontend \
  --policy file://s3-frontend-bucket-policy.json

echo "✅ S3 bucket policy applied — only frontend devs can upload"
```

### Block Public Access (Always Do This)

```bash
aws s3api put-public-access-block \
  --bucket technova-frontend \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

echo "✅ Public access blocked on technova-frontend"
```

---

## LAB 12 — Instance Profiles (How EC2 Actually Uses a Role)

> Creating a role is NOT enough. You must create an Instance Profile and attach it to EC2.
> Without this, EC2 cannot use the role — this is a very common mistake.

```bash
# Step 1: Create Instance Profile (wrapper around the role for EC2)
aws iam create-instance-profile \
  --instance-profile-name profile-ec2-s3-access

# Step 2: Add the role INTO the instance profile
aws iam add-role-to-instance-profile \
  --instance-profile-name profile-ec2-s3-access \
  --role-name role-ec2-s3-access

# Step 3: Attach the instance profile to a running EC2 instance
# Replace i-0abc123 with your actual instance ID
aws ec2 associate-iam-instance-profile \
  --instance-id i-0abc123def456789 \
  --iam-instance-profile Name=profile-ec2-s3-access

# Step 4: Verify the EC2 has the role
aws ec2 describe-instances \
  --instance-ids i-0abc123def456789 \
  --query 'Reservations[0].Instances[0].IamInstanceProfile'

# Step 5: SSH into the EC2 and verify it can access S3
# (Run this from INSIDE the EC2 — no keys needed, role handles it)
# aws s3 ls s3://technova-data-lake  → Should work!
# aws iam get-user              → Should FAIL (EC2 role, not a user)

echo "✅ Instance profile attached — EC2 now has S3 access via role"
```

---

## LAB 13 — CI/CD Role for GitHub Actions (No Hardcoded Keys)

> Real deployments use GitHub Actions / CodePipeline.
> NEVER store AWS access keys in GitHub secrets — use OIDC roles instead.

### Step 1: Create the OIDC Identity Provider for GitHub

```bash
# Register GitHub as a trusted identity provider in AWS
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

echo "✅ GitHub OIDC provider registered"
```

### Step 2: Create the Deployment Role

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
GITHUB_ORG="your-org-name"       # Replace with your GitHub org/username
GITHUB_REPO="your-repo-name"     # Replace with your repo name

cat > trust-github-actions.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:${GITHUB_ORG}/${GITHUB_REPO}:*"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name role-github-deploy \
  --assume-role-policy-document file://trust-github-actions.json

# Attach only what deployment needs (S3 deploy + CloudFront invalidation)
cat > policy-github-deploy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Deploy",
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::technova-frontend",
        "arn:aws:s3:::technova-frontend/*"
      ]
    },
    {
      "Sid": "CloudFrontInvalidate",
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name policy-github-deploy \
  --policy-document file://policy-github-deploy.json

aws iam attach-role-policy \
  --role-name role-github-deploy \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-github-deploy

echo "✅ GitHub Actions OIDC role created — no access keys needed"
```

### Step 3: Use the Role in GitHub Actions Workflow

```yaml
# .github/workflows/deploy.yml
name: Deploy Frontend to S3

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/role-github-deploy
          aws-region: ap-south-1

      - name: Deploy to S3
        run: |
          aws s3 sync ./dist s3://technova-frontend --delete

      - name: Invalidate CloudFront
        run: |
          aws cloudfront create-invalidation \
            --distribution-id YOUR_DISTRIBUTION_ID \
            --paths "/*"
```

---

## LAB 14 — Permission Boundaries (Stop Privilege Escalation)

> Without this, Arjun (admin) could create a new user with full admin access
> and bypass all controls. Permission Boundaries prevent this.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# This boundary limits what any dev user can ever do — even if admin grants more
cat > permission-boundary-dev.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServices",
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "lambda:*",
        "rds:*",
        "cloudwatch:*",
        "logs:*",
        "glue:*",
        "athena:*",
        "ecr:*",
        "ecs:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyIAMWriteAccess",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:DeleteUser",
        "iam:AttachUserPolicy",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:CreatePolicy",
        "iam:DeletePolicy",
        "iam:PutUserPolicy",
        "organizations:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyBillingChanges",
      "Effect": "Deny",
      "Action": [
        "aws-portal:ModifyBilling",
        "aws-portal:ModifyAccount"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name permission-boundary-dev \
  --policy-document file://permission-boundary-dev.json

# Apply this boundary when creating any dev/devops user
# This caps their max possible permissions — even if admin grants AdministratorAccess,
# the boundary STILL blocks IAM write and billing

aws iam put-user-permissions-boundary \
  --user-name priya-backend \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/permission-boundary-dev

aws iam put-user-permissions-boundary \
  --user-name rahul-backend \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/permission-boundary-dev

aws iam put-user-permissions-boundary \
  --user-name anand-devops \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/permission-boundary-dev

echo "✅ Permission boundaries applied — devs can never touch IAM or billing"
```

---

## LAB 15 — Enable CloudTrail (Audit Every Action)

> CloudTrail records EVERY API call — who did what, when, from which IP.
> This is mandatory for security audits and required in most compliance standards.

```bash
# Step 1: Create an S3 bucket for CloudTrail logs
aws s3api create-bucket \
  --bucket technova-cloudtrail-logs-${ACCOUNT_ID} \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Step 2: Attach bucket policy that allows CloudTrail to write
cat > cloudtrail-bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSCloudTrailAclCheck",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::technova-cloudtrail-logs-${ACCOUNT_ID}"
    },
    {
      "Sid": "AWSCloudTrailWrite",
      "Effect": "Allow",
      "Principal": { "Service": "cloudtrail.amazonaws.com" },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::technova-cloudtrail-logs-${ACCOUNT_ID}/AWSLogs/${ACCOUNT_ID}/*",
      "Condition": {
        "StringEquals": {
          "s3:x-amz-acl": "bucket-owner-full-control"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket technova-cloudtrail-logs-${ACCOUNT_ID} \
  --policy file://cloudtrail-bucket-policy.json

# Step 3: Create the trail
aws cloudtrail create-trail \
  --name technova-audit-trail \
  --s3-bucket-name technova-cloudtrail-logs-${ACCOUNT_ID} \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation

# Step 4: Start logging
aws cloudtrail start-logging \
  --name technova-audit-trail

# Step 5: Verify it's active
aws cloudtrail get-trail-status --name technova-audit-trail \
  --query '{Logging: IsLogging, LatestDelivery: LatestDeliveryTime}'

echo "✅ CloudTrail enabled — every AWS action is now being logged"
```

### How to Read CloudTrail Logs

```bash
# Search CloudTrail logs for who deleted a resource (last 90 days via console)
# Console: CloudTrail → Event History → filter by Event Name or Username

# Via CLI — find all IAM actions by a specific user in last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=priya-backend \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[].{Time:EventTime, Action:EventName, Source:EventSource}' \
  --output table

# Find all DeletePolicy events across all users
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeletePolicy \
  --query 'Events[].{Time:EventTime, User:Username, Action:EventName}' \
  --output table
```

---

## LAB 16 — IAM Access Analyzer

> Finds resources that are accessible from outside your account — over-permissive policies.

```bash
# Step 1: Enable Access Analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name technova-analyzer \
  --type ACCOUNT

# Step 2: List findings (things that might be over-exposed)
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:ap-south-1:${ACCOUNT_ID}:analyzer/technova-analyzer \
  --query 'findings[].{Resource:resource, ResourceType:resourceType, Status:status}' \
  --output table

# Step 3: Validate a custom policy before applying it
aws accessanalyzer validate-policy \
  --policy-document file://policy-backend-dev.json \
  --policy-type IDENTITY_POLICY \
  --query 'findings[].{Type:findingType, Message:findingDetails, Issue:issueCode}' \
  --output table

echo "✅ Access Analyzer active — scans for unintended external access"
```

---

## LAB 17 — Credential Report (Full Audit of All Users)

> One command to see all users, their password age, last login, MFA status, key age.

```bash
# Step 1: Generate the report
aws iam generate-credential-report

# Step 2: Wait a moment, then download
sleep 5

# Step 3: Decode and view the report
aws iam get-credential-report \
  --query 'Content' \
  --output text | base64 --decode > credential-report.csv

# Step 4: View it
cat credential-report.csv | column -t -s','

# What to look for in the report:
echo "
Check these columns:
  password_last_used       → Anyone not logged in for 90+ days? Deactivate them.
  mfa_active               → Anyone with 'false'? Enforce MFA immediately.
  access_key_1_last_used   → Keys not used in 90 days? Rotate or delete.
  access_key_1_last_rotated → Keys older than 90 days? Rotate now.
  password_last_changed    → Accounts with old passwords? Force reset.
"
```

---

## LAB 18 — Access Advisor (Find & Remove Unused Permissions)

> Access Advisor shows the last time each service was actually used.
> If a dev hasn't used EC2 in 6 months, remove EC2 from their policy.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Check what services Priya actually used recently
JOB_ID=$(aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::${ACCOUNT_ID}:user/priya-backend \
  --query 'JobId' --output text)

echo "Waiting for report to generate..."
sleep 8

# Get the results
aws iam get-service-last-accessed-details \
  --job-id $JOB_ID \
  --query 'ServicesLastAccessed[?TotalAuthenticatedEntities>`0`].{Service:ServiceName, LastUsed:LastAuthenticated}' \
  --output table

# Show services NEVER used by Priya (remove these from her policy!)
aws iam get-service-last-accessed-details \
  --job-id $JOB_ID \
  --query 'ServicesLastAccessed[?TotalAuthenticatedEntities==`0`].ServiceName' \
  --output table

echo "Services with 0 usage should be REMOVED from the policy — apply least-privilege"
```

---

## LAB 19 — AWS Config Rules (Automated Compliance Monitoring)

> Config Rules automatically check if your IAM setup follows security rules.
> Triggers alerts if someone creates a user without MFA or leaves a key unrotated.

```bash
# Enable AWS Config recorder first (if not already on)
aws configservice put-configuration-recorder \
  --configuration-recorder name=default,roleARN=arn:aws:iam::${ACCOUNT_ID}:role/aws-service-role/config.amazonaws.com/AWSServiceRoleForConfig

aws configservice put-delivery-channel \
  --delivery-channel name=default,s3BucketName=technova-cloudtrail-logs-${ACCOUNT_ID}

aws configservice start-configuration-recorder --configuration-recorder-name default

# Now add managed compliance rules

# Rule 1: Root account must have MFA
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "root-account-mfa-enabled",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "ROOT_ACCOUNT_MFA_ENABLED"
  }
}'

# Rule 2: All IAM users must have MFA
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "iam-user-mfa-enabled",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "IAM_USER_MFA_ENABLED"
  }
}'

# Rule 3: No root access keys
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "iam-root-access-key-check",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "IAM_ROOT_ACCESS_KEY_CHECK"
  }
}'

# Rule 4: Access keys must be rotated within 90 days
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "access-keys-rotated",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "ACCESS_KEYS_ROTATED"
  },
  "InputParameters": "{\"maxAccessKeyAge\": \"90\"}"
}'

# Rule 5: Unused credentials should be disabled
aws configservice put-config-rule --config-rule '{
  "ConfigRuleName": "iam-user-unused-credentials-check",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "IAM_USER_UNUSED_CREDENTIALS_CHECK"
  },
  "InputParameters": "{\"maxCredentialUsageAge\": \"90\"}"
}'

# Check compliance status
aws configservice describe-compliance-by-config-rule \
  --query 'ComplianceByConfigRules[].{Rule:ConfigRuleName, Status:Compliance.ComplianceType}' \
  --output table

echo "✅ Config rules active — auto compliance monitoring running"
```

---

## LAB 20 — Break-Glass Emergency Access

> What if Arjun is hit by a bus? You need emergency access that is:
> locked away normally, but usable in a real emergency.

```bash
# Step 1: Create a break-glass user (locked, no console login normally)
aws iam create-user --user-name breakglass-emergency

aws iam attach-user-policy \
  --user-name breakglass-emergency \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# Step 2: Create access keys (store OFFLINE in a password manager vault)
aws iam create-access-key --user-name breakglass-emergency
# ⚠️  Print these keys. Store in physical safe OR sealed envelope.
# NEVER store digitally unless in a secure vault (like 1Password, Bitwarden Teams).

# Step 3: Disable the keys NOW (only enable in real emergency)
aws iam update-access-key \
  --user-name breakglass-emergency \
  --access-key-id AKIAIOSFODNN7EXAMPLE \
  --status Inactive

# Step 4: Set a CloudWatch alarm for when breakglass is used
aws cloudwatch put-metric-alarm \
  --alarm-name "EMERGENCY-BreakGlass-Used" \
  --alarm-description "Break-glass account was accessed — INVESTIGATE IMMEDIATELY" \
  --metric-name "ErrorCount" \
  --namespace "CloudTrailMetrics" \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

echo "✅ Break-glass account created, locked, and monitored"
```

---

## LAB 21 — CloudWatch Alarms for Suspicious IAM Activity

> Get instant alerts when something dangerous happens.

```bash
# Step 1: Create SNS topic for security alerts (sends email)
aws sns create-topic --name security-alerts

aws sns subscribe \
  --topic-arn arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts \
  --protocol email \
  --notification-endpoint arjun@technova.com
# Arjun must confirm the email subscription

# Step 2: Create CloudTrail metric filters (these watch the log stream)

LOG_GROUP_NAME="aws-cloudtrail-logs-technova"

# First create a log group for CloudTrail if not exists
aws logs create-log-group --log-group-name $LOG_GROUP_NAME

# Alarm 1: Someone disabled CloudTrail (very suspicious)
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name "CloudTrailStopped" \
  --filter-pattern '{ ($.eventName = StopLogging) }' \
  --metric-transformations metricName=CloudTrailStopped,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "ALERT-CloudTrail-Stopped" \
  --metric-name CloudTrailStopped \
  --namespace SecurityAlerts \
  --statistic Sum --period 300 \
  --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

# Alarm 2: Root account used
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name "RootAccountUsed" \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations metricName=RootAccountUsed,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "ALERT-Root-Account-Used" \
  --metric-name RootAccountUsed \
  --namespace SecurityAlerts \
  --statistic Sum --period 60 \
  --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

# Alarm 3: IAM Policy deleted
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name "IAMPolicyDeleted" \
  --filter-pattern '{ ($.eventName = DeleteGroupPolicy) || ($.eventName = DeleteRolePolicy) || ($.eventName = DeleteUserPolicy) || ($.eventName = DetachGroupPolicy) || ($.eventName = DetachRolePolicy) || ($.eventName = DetachUserPolicy) }' \
  --metric-transformations metricName=IAMPolicyDeleted,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "ALERT-IAM-Policy-Deleted" \
  --metric-name IAMPolicyDeleted \
  --namespace SecurityAlerts \
  --statistic Sum --period 300 \
  --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

# Alarm 4: MFA disabled on any user
aws logs put-metric-filter \
  --log-group-name $LOG_GROUP_NAME \
  --filter-name "MFADisabled" \
  --filter-pattern '{ ($.eventName = DeleteVirtualMFADevice) || ($.eventName = DeactivateMFADevice) }' \
  --metric-transformations metricName=MFADisabled,metricNamespace=SecurityAlerts,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name "ALERT-MFA-Disabled" \
  --metric-name MFADisabled \
  --namespace SecurityAlerts \
  --statistic Sum --period 300 \
  --evaluation-periods 1 --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions arn:aws:sns:ap-south-1:${ACCOUNT_ID}:security-alerts

echo "✅ 4 security alarms active — Arjun will get email alerts instantly"
```

---

## LAB 22 — Access Key Rotation (90-Day Rule)

> Old keys = biggest security risk. Real teams rotate every 90 days.

```bash
# Step 1: Check age of all access keys
aws iam generate-credential-report
sleep 5
aws iam get-credential-report --query 'Content' --output text | \
  base64 --decode | \
  awk -F',' 'NR==1 || $10 != "N/A" {print $1, $10}' | \
  column -t
# Shows: username and access_key_1_last_rotated date

# Step 2: Rotate a key (create new → update app/CLI → delete old)

# Create new key
aws iam create-access-key --user-name priya-backend
# Output: New AccessKeyId + SecretAccessKey
# → Give to Priya to update her CLI: aws configure

# After Priya confirms new key works, delete the old one
aws iam list-access-keys --user-name priya-backend
# Find the OLD key ID (the one created earlier)

aws iam update-access-key \
  --user-name priya-backend \
  --access-key-id AKIAOLDKEYID12345 \
  --status Inactive
# Test for 24 hours that nothing broke, then delete

aws iam delete-access-key \
  --user-name priya-backend \
  --access-key-id AKIAOLDKEYID12345

echo "✅ Key rotated — Priya now has a fresh access key"
```

### Key Rotation Reminder Script (Run Monthly via Cron)

```bash
#!/bin/bash
# save as: check-key-age.sh
# cron: 0 9 1 * * /path/to/check-key-age.sh

echo "=== IAM Access Key Age Report — $(date) ==="
aws iam generate-credential-report > /dev/null
sleep 5

aws iam get-credential-report --query 'Content' --output text | \
  base64 --decode | \
  python3 -c "
import sys, csv
from datetime import datetime, timezone

reader = csv.DictReader(sys.stdin)
print(f'{'User':<25} {'Key Created':<15} {'Days Old':<10} {'Status':<10}')
print('-' * 65)
for row in reader:
    for key_num in ['1', '2']:
        created = row.get(f'access_key_{key_num}_last_rotated', 'N/A')
        if created != 'N/A' and created != 'not_supported':
            created_date = datetime.fromisoformat(created.replace('Z', '+00:00'))
            age = (datetime.now(timezone.utc) - created_date).days
            status = '⚠️  ROTATE NOW' if age > 90 else '✅ OK'
            print(f'{row[\"user\"]: <25} {created[:10]:<15} {str(age):<10} {status}')
"
```

---

## LAB 23 — Secrets Manager (No Hardcoded Passwords)

> Developers must NEVER hardcode DB passwords in code or .env files.
> Secrets Manager stores and auto-rotates them.

```bash
# Step 1: Store the RDS database password in Secrets Manager
aws secretsmanager create-secret \
  --name "technova/prod/db-password" \
  --description "TechNova production RDS password" \
  --secret-string '{"username":"admin","password":"Sup3rS3cureP@ss!","host":"prod-db.cluster-xyz.ap-south-1.rds.amazonaws.com","port":"5432","dbname":"technova_prod"}'

# Step 2: Create policy so backend EC2 can READ the secret (not write)
cat > policy-read-db-secret.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "secretsmanager:GetSecretValue",
      "secretsmanager:DescribeSecret"
    ],
    "Resource": "arn:aws:secretsmanager:ap-south-1:*:secret:technova/prod/*"
  }]
}
EOF

aws iam create-policy \
  --policy-name policy-read-db-secret \
  --policy-document file://policy-read-db-secret.json

# Attach to the EC2 role (so backend servers can get DB creds automatically)
aws iam attach-role-policy \
  --role-name role-ec2-s3-access \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/policy-read-db-secret

echo "✅ DB password stored in Secrets Manager — no more .env files with passwords"
```

### How Backend Code Retrieves the Secret (No Hardcoding)

```python
# backend app — Python example
import boto3
import json

def get_db_credentials():
    client = boto3.client('secretsmanager', region_name='ap-south-1')
    response = client.get_secret_value(SecretId='technova/prod/db-password')
    secret = json.loads(response['SecretString'])
    return secret['host'], secret['username'], secret['password']

host, user, password = get_db_credentials()
# Connect to DB using these — never stored in code!
```

---

## LAB 24 — Tag-Based Access Control (ABAC)

> Instead of creating new policies for every team, use TAGS to control access.
> This scales much better as the startup grows.

```bash
# Tag EC2 instances by environment
aws ec2 create-tags \
  --resources i-0abc123 \
  --tags Key=Environment,Value=prod Key=Team,Value=backend

aws ec2 create-tags \
  --resources i-0def456 \
  --tags Key=Environment,Value=dev Key=Team,Value=backend

# Policy: Backend devs can ONLY stop/start EC2 tagged with Team=backend
cat > policy-abac-backend.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2ActionsOnTeamResources",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Team": "backend"
        }
      }
    },
    {
      "Sid": "DenyProdWithoutMFA",
      "Effect": "Deny",
      "Action": [
        "ec2:StopInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Environment": "prod"
        },
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
EOF

# With this policy:
# Priya (backend dev) CAN stop dev backend EC2 instances
# Priya CANNOT stop prod instances without MFA
# Priya CANNOT touch frontend or data instances at all

echo "✅ ABAC policy ready — tag your resources, not your users"
```

---

## LAB 25 — Multi-Environment Account Setup (Dev / Staging / Prod)

> Real startups NEVER mix dev and prod in the same AWS account.
> Use AWS Organizations to manage 3 separate accounts.

```
AWS Organization Structure:
│
├── Management Account (billing only — no workloads)
│
├── Organizational Unit: Environments
│   ├── technova-dev     Account (account ID: 111111111111)
│   ├── technova-staging Account (account ID: 222222222222)
│   └── technova-prod    Account (account ID: 333333333333)
│
└── SCP (Service Control Policies) applied at OU level
    └── Deny: ec2:TerminateInstances in prod without MFA
    └── Deny: deleting CloudTrail in any account
```

### Create Cross-Account Role (Dev team accesses Prod for emergencies)

```bash
# In the PROD account — create role that DEV account admin can assume
PROD_ACCOUNT_ID="333333333333"
DEV_ACCOUNT_ID="111111111111"

cat > trust-cross-account.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::${DEV_ACCOUNT_ID}:user/arjun-admin"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "Bool": { "aws:MultiFactorAuthPresent": "true" }
    }
  }]
}
EOF

# Run this in PROD account
aws iam create-role \
  --role-name role-prod-readonly-from-dev \
  --assume-role-policy-document file://trust-cross-account.json

aws iam attach-role-policy \
  --role-name role-prod-readonly-from-dev \
  --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

echo "✅ Cross-account role created — Arjun can read prod from dev account"
```

### Assuming the Cross-Account Role

```bash
# Arjun runs this from DEV account to get PROD read access temporarily
aws sts assume-role \
  --role-arn arn:aws:iam::333333333333:role/role-prod-readonly-from-dev \
  --role-session-name arjun-prod-readonly-session \
  --serial-number arn:aws:iam::111111111111:mfa/arjun-admin \
  --token-code 123456

# Use the temporary credentials returned (expire in 1 hour automatically)
export AWS_ACCESS_KEY_ID=ASIA...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

aws s3 ls  # Now in prod account, read-only
```

---

## 🗺️ Complete Real-World Architecture (What You've Built)

```
TechNova AWS Setup (Production-Ready)
│
├── 🔒 Security Foundation
│   ├── Root MFA enabled + locked away
│   ├── CloudTrail logging every action
│   ├── Config Rules auto-checking compliance
│   ├── Access Analyzer finding over-exposure
│   └── CloudWatch alarms for 4 critical events
│
├── 👤 Identity
│   ├── 9 IAM users in correct groups
│   ├── MFA enforced on all users
│   ├── Permission Boundaries on devs (no IAM/billing)
│   ├── Break-glass emergency account (locked + monitored)
│   └── Password policy (90-day expiry, 12+ chars)
│
├── 🔑 Access Control
│   ├── 6 Groups with custom least-privilege policies
│   ├── S3 bucket policies (dual-layer protection)
│   ├── ABAC tag-based access for EC2
│   └── Cross-account role for prod access (MFA required)
│
├── 🤖 Service Roles
│   ├── EC2 → S3 (via instance profile)
│   ├── Lambda → DynamoDB
│   ├── EC2 → CloudWatch logs
│   └── GitHub Actions → S3 deploy (OIDC, no keys)
│
├── 🗄️ Secrets
│   └── RDS passwords in Secrets Manager (never hardcoded)
│
└── 🌍 Multi-Account
    ├── Dev account (111111111111)
    ├── Staging account (222222222222)
    └── Prod account (333333333333) — cross-account role access only
```

---

## 📋 Production IAM Security Checklist (Full)

```
Foundation:
 ✅ Root MFA enabled, no root access keys, root never used
 ✅ CloudTrail enabled in ALL regions (multi-region trail)
 ✅ CloudTrail logs protected in S3 with MFA delete
 ✅ Config Rules active for IAM compliance
 ✅ Access Analyzer enabled

Users & Groups:
 ✅ No shared user accounts — one user per person
 ✅ All users in groups — no direct policy attachments (except billing)
 ✅ MFA enforced via policy on all groups
 ✅ Credential report reviewed monthly
 ✅ Unused credentials (90+ days) deactivated

Permissions:
 ✅ Least-privilege applied — no Action:* or Resource:*
 ✅ Permission boundaries on all non-admin users
 ✅ Access Advisor reviewed quarterly — unused services removed
 ✅ Resource-based policies (S3 bucket policies) in place

Keys & Secrets:
 ✅ Access keys rotated every 90 days
 ✅ No access keys for services — use IAM Roles
 ✅ No keys in GitHub repos, .env files, or source code
 ✅ DB passwords and API keys in Secrets Manager

Roles:
 ✅ EC2 uses instance profiles (not access keys)
 ✅ Lambda uses execution roles (not access keys)
 ✅ GitHub Actions uses OIDC (not access keys)
 ✅ Cross-account access uses roles with MFA condition

Monitoring & Response:
 ✅ CloudWatch alarm: Root account used
 ✅ CloudWatch alarm: CloudTrail stopped
 ✅ CloudWatch alarm: IAM policy deleted
 ✅ CloudWatch alarm: MFA disabled
 ✅ Break-glass account exists, locked, and monitored
 ✅ Security team email subscribed to SNS alerts

Multi-Account:
 ✅ Dev / Staging / Prod in separate accounts
 ✅ AWS Organizations with SCPs in place
 ✅ No human access to prod — only cross-account roles with MFA
```

---

*This is the full real-world IAM setup that production startups actually run.
Part 1 = structure. Part 2 = what keeps you safe at 3am. 🔐*
