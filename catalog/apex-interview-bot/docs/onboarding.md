# Onboarding — APEX Interview Automation Bot

---

## Prerequisites

### AWS

- AWS account with permissions for: EC2, ALB, S3, Secrets Manager, ECR, IAM, CloudWatch, SSM, Route 53
- Existing VPC with public and private subnets (APEX reuses these via Terraform data sources — it does not create them)
- ACM certificate for the APEX domain (e.g., `apex-bot.yourcompany.com`)
- Route 53 hosted zone

### External Services

| Service | What you need |
|---------|--------------|
| Zoom | Server-to-Server OAuth app: Account ID, Client ID, Client Secret, Webhook Secret Token |
| Microsoft Azure AD | App registration: Tenant ID, Client ID, Client Secret |
| Slack | Bot app: Bot Token (`xoxb-...`), target channel ID |
| Confluence | Service account API token, Space Key, Parent Page ID |
| Anthropic | API key from [console.anthropic.com](https://console.anthropic.com) |

### Local Tooling

```bash
aws --version       # >= 2.x
terraform --version # >= 1.9
docker --version    # >= 24.x
git --version
```

---

## Step 1 — Create ECR Repository

```bash
REGION="us-east-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws ecr create-repository \
  --repository-name apex-interview-bot \
  --region "$REGION" \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256

echo "ECR URI: $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/apex-interview-bot"
```

---

## Step 2 — Bootstrap Terraform State Bucket

```bash
STATE_BUCKET="apex-tf-state-$ACCOUNT_ID"

aws s3api create-bucket \
  --bucket "$STATE_BUCKET" \
  --region "$REGION" \
  --create-bucket-configuration LocationConstraint="$REGION"

aws s3api put-bucket-versioning \
  --bucket "$STATE_BUCKET" \
  --versioning-configuration Status=Enabled

aws s3api put-bucket-encryption \
  --bucket "$STATE_BUCKET" \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'
```

---

## Step 3 — Terraform Init, Plan, Apply

```bash
cd iac/

terraform init \
  -backend-config="bucket=$STATE_BUCKET" \
  -backend-config="key=apex-interview-bot/terraform.tfstate" \
  -backend-config="region=$REGION" \
  -backend-config="encrypt=true"

# Create terraform.tfvars with your VPC/subnet/domain values
terraform plan -var-file=terraform.tfvars -out=tfplan
terraform apply tfplan

# Save the EC2 instance ID from outputs
terraform output ec2_instance_id
```

---

## Step 4 — Populate Secrets Manager

```bash
aws secretsmanager put-secret-value \
  --secret-id "apex-interview-bot/secrets" \
  --region "$REGION" \
  --secret-string '{
    "anthropic_api_key":             "sk-ant-YOUR_KEY",
    "zoom_account_id":               "YOUR_ZOOM_ACCOUNT_ID",
    "zoom_client_id":                "YOUR_ZOOM_CLIENT_ID",
    "zoom_client_secret":            "YOUR_ZOOM_CLIENT_SECRET",
    "zoom_webhook_secret_token":     "YOUR_ZOOM_WEBHOOK_TOKEN",
    "slack_bot_token":               "xoxb-YOUR_SLACK_TOKEN",
    "slack_channel_interviewers":    "C0XXXXXXXXX",
    "graph_tenant_id":               "YOUR_TENANT_ID",
    "graph_client_id":               "YOUR_CLIENT_ID",
    "graph_client_secret":           "YOUR_CLIENT_SECRET",
    "graph_webhook_client_state":    "YOUR_RANDOM_SECRET_STRING",
    "confluence_base_url":           "https://yourcompany.atlassian.net",
    "confluence_username":           "serviceaccount@yourcompany.com",
    "confluence_api_token":          "YOUR_CONFLUENCE_TOKEN",
    "confluence_space_key":          "APEX",
    "confluence_parent_page_id":     "123456789"
  }'
```

---

## Step 5 — Build and Push Docker Image

```bash
IMAGE_URI="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/apex-interview-bot"

aws ecr get-login-password --region "$REGION" \
  | docker login --username AWS --password-stdin \
    "$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com"

docker build -t "$IMAGE_URI:latest" -t "$IMAGE_URI:$(git rev-parse --short HEAD)" .
docker push "$IMAGE_URI:latest"
docker push "$IMAGE_URI:$(git rev-parse --short HEAD)"
```

---

## Step 6 — Verify Health Endpoint

```bash
APP_DOMAIN="apex-bot.yourcompany.com"

curl -s "https://$APP_DOMAIN/health" | jq .
# Expected: {"status": "healthy", "version": "..."}
```

---

## Step 7 — Register Webhooks

### Zoom

1. Go to **Zoom Marketplace → Your App → Event Subscriptions**
2. Set endpoint URL: `https://apex-bot.yourcompany.com/webhooks/zoom`
3. Subscribe to: `recording.completed`, `meeting.ended`
4. Zoom will send a challenge — APEX responds automatically

### Microsoft Graph

APEX registers the Graph subscription per interview during scheduling. Confirm via CloudWatch:

```bash
aws logs tail /apex-interview-bot/system --follow --filter-pattern "graph_subscription_id"
```

---

## Step 8 — GitHub Actions Setup

Add these secrets to **GitHub → Settings → Secrets and variables → Actions**:

| Secret | Value |
|--------|-------|
| `AWS_ACCOUNT_ID` | 12-digit AWS account ID |
| `AWS_REGION` | e.g. `us-east-1` |
| `ECR_REPOSITORY` | `apex-interview-bot` |
| `SECRETS_MANAGER_SECRET_NAME` | `apex-interview-bot/secrets` |
| `S3_BUCKET_NAME` | `apex-interview-bot-state` |
| `EC2_INSTANCE_ID` | From `terraform output` in Step 3 |
| `APP_DOMAIN` | e.g. `apex-bot.yourcompany.com` |

Also create a GitHub OIDC provider in AWS and an IAM role `github-actions-apex-deploy`
with permissions to push to ECR and send SSM Run Commands.

---

## Step 9 — End-to-End Test

```bash
curl -s -X POST "https://$APP_DOMAIN/interviews" \
  -H "Content-Type: application/json" \
  -d '{
    "candidate_name": "Test Candidate",
    "role": "Software Engineer",
    "interviewer_emails": ["interviewer@yourcompany.com"],
    "scheduled_time": "2026-06-15T14:00:00Z",
    "duration_minutes": 60
  }' | jq .

# Check interview state in S3
aws s3 ls s3://apex-interview-bot-state/interviews/ --recursive
```
