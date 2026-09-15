# AWS DevOps Interview Prep — Study Order & Timeline

A dependency-based reading order for the AWS guides in the repo, with a rough day estimate per topic (assuming ~1–2 focused hours/day alongside full-time work). Adjust up or down based on how much time you can actually give it each day.

| # | File | Focus | Est. Days |
|---|------|-------|-----------|
| 1 | `AWS_IAM_Hands_On_Guide.md` | Identity & access — the foundation everything else depends on | 1 |
| 2 | `aws_vpc_handson_guide.md` | Networking foundation — subnets, route tables, IGW/NAT | 2 |
| 3 | `AWS_EC2_Hands_On_Guide.md` | Core compute, built directly on VPC + IAM | 1.5 |
| 4 | `ALB_NLB_AutoScaling_Mastery.md` | Load balancing & auto scaling on top of EC2 | 2 |
| 5 | `AWS_Route53_S3_DevOps_MasterGuide.md` | DNS + object storage, usually paired | 2 |
| 6 | `AWS-RDS-DisasterRecovery-DevOps-Guide.md` | Managed databases & DR strategy | 2 |
| 7 | `AWS_Secrets_Manager_Mastery.md` | Secrets management, ties back to IAM | 1 |
| 8 | `AWS_Containers_Master_Guide.md` | ECS/EKS — pulls together VPC, IAM, ALB, Secrets | 2.5 *(faster for you given existing EKS background)* |
| 9 | `AWS-SNS-SQS-CloudWatch-Logs-Mentor-Guide.md` | Messaging (SNS/SQS) + CloudWatch logging basics | 2 |
| 10 | `AWS_SNS_SQS_CloudWatch_ELK_Logging_Mastery.md` *(name truncated in repo view — confirm exact filename)* | Advanced logging, integrates ELK | 1.5 *(faster for you given existing ELK background)* |

**Total: ~17.5 days (~3.5 weeks)** at the pace above.

## How to use this
- Work through in order — each guide leans on concepts from the ones before it.
- Treat these as hands-on, not just reading: actually spin up the resources in a free-tier/sandbox account as you go. That's what sticks for interviews.
- This order also roughly tracks the AWS Solutions Architect Associate exam flow, so it does double duty toward that cert.
- If an interview comes up before you finish all 10, prioritize **IAM → VPC → EC2 → Containers** — these get asked about most in DevOps/Platform Engineer interviews.
- Track your actual progress by checking off each row and noting your real completion date vs. the estimate — that'll tell you fast whether the pacing needs adjusting.
