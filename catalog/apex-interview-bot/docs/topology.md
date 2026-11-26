# Topology — APEX Interview Automation Bot

## Overview

Single-region AWS deployment. One EC2 t3.medium instance running Amazon Linux 2023 hosts
the APEX Docker container. An Application Load Balancer provides HTTPS termination and
serves as the public ingress point for both coordinator API calls and inbound webhooks
from Zoom and Microsoft Graph. All outbound traffic to external APIs exits through a NAT
Gateway. No ECS, Kubernetes, Lambda, Step Functions, or managed database is used.

---

## Network Zones

| Zone | Components | Traffic |
|------|-----------|---------|
| **Public** | Route 53, ALB | Inbound HTTPS :443 from internet |
| **Private** | EC2 t3.medium, Docker container | Inbound HTTP :8000 from ALB only |

---

## Compute

| Resource | Detail | Status |
|----------|--------|--------|
| EC2 t3.medium | Amazon Linux 2023, private subnet | [DOCUMENTED] |
| Docker container | `apex-interview-bot` image from ECR, port 8000 | [DOCUMENTED] |
| Instance type | t3.medium (2 vCPU, 4 GiB RAM) | [DOCUMENTED] |

---

## Networking

| Resource | Detail | Status |
|----------|--------|--------|
| Application Load Balancer | Internet-facing, HTTPS :443, ACM certificate | [DOCUMENTED] |
| Route 53 | Alias A record → ALB DNS name | [DOCUMENTED] |
| NAT Gateway | Outbound internet access for EC2 in private subnet | [DOCUMENTED] |
| VPC / Subnets | Reused via Terraform data sources — not created by APEX IaC | [DOCUMENTED] |
| Security Group (ALB) | Allow 443 inbound from 0.0.0.0/0 | [DOCUMENTED] |
| Security Group (EC2) | Allow 8000 from ALB SG only | [DOCUMENTED] |

---

## Storage

| Resource | Purpose | Access | Status |
|----------|---------|--------|--------|
| S3 bucket `apex-interview-bot-state` | Sole persistence layer — interview state, transcripts, summaries | EC2 IAM role (scoped read/write) | [DOCUMENTED] |

---

## Secrets

| Resource | Purpose | Access | Status |
|----------|---------|--------|--------|
| Secrets Manager `apex-interview-bot/secrets` | All credentials — Zoom, Graph, Slack, Confluence, Anthropic | EC2 IAM role (read-only on specific ARN) | [DOCUMENTED] |

> Secrets are fetched once at container startup and cached in-process for the container lifetime.

---

## Observability

| Resource | Detail | Status |
|----------|--------|--------|
| CloudWatch Logs | Log group `/apex-interview-bot/system` via Docker `awslogs` driver | [DOCUMENTED] |

---

## CI/CD

| Resource | Detail | Status |
|----------|--------|--------|
| GitHub Actions | Triggers on push to `main`: pytest → docker build → ECR push → SSM deploy | [DOCUMENTED] |
| Amazon ECR | Repository `apex-interview-bot`, SHA-pinned tags + `:latest` | [DOCUMENTED] |
| SSM Run Command | Pulls new image and restarts container on EC2 — no SSH required | [DOCUMENTED] |

---

## IAM Permissions (EC2 Instance Role)

| Permission | Scope |
|-----------|-------|
| `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` | `apex-interview-bot-state` bucket only |
| `secretsmanager:GetSecretValue` | Specific secret ARN only |
| `logs:CreateLogStream`, `logs:PutLogEvents` | `/apex-interview-bot/system` log group |
| `ecr:GetAuthorizationToken` | Cannot be resource-scoped — applies to all ECR (AWS constraint) |
| `ecr:BatchGetImage`, `ecr:GetDownloadUrlForLayer` | Scoped to `apex-interview-bot` repo ARN only |
| bounded SSM + EC2Messages permissions | Via `AmazonSSMManagedInstanceCore` managed policy (not `ssm:*` — the policy grants a fixed set: DescribeAssociation, GetDocument, GetParameter/s, ListAssociations, UpdateInstanceInformation, ssmmessages:*, ec2messages:*) |

---

## Infrastructure Diagram

```
Internet
    │ HTTPS :443
    ▼
Route 53 (alias)
    │
    ▼
Application Load Balancer  ─── ACM Certificate
    │ HTTP :8000
    ▼
EC2 t3.medium (private subnet)
    └── Docker: apex-interview-bot (:8000)
            │
            ├──► S3 (state store)
            ├──► Secrets Manager
            ├──► CloudWatch Logs
            ├──► ECR (image pull)
            │
            └── via NAT Gateway ──►
                    ├── Zoom API
                    ├── Microsoft Graph API
                    ├── Slack API
                    ├── Anthropic API
                    └── Confluence API

Zoom API ──────────► ALB /webhooks/zoom    (inbound)
Microsoft Graph ───► ALB /webhooks/graph   (inbound)
```
