# User Manual — APEX Interview Automation Bot

---

## Scheduling an Interview

Recruiting coordinators trigger interviews by calling the APEX scheduling endpoint. APEX
handles everything from that point forward — Zoom, calendar, Slack, transcript, summary,
and Confluence.

**Endpoint:** `POST https://apex-bot.yourcompany.com/interviews`

**Request body:**

```json
{
  "candidate_name": "Jane Smith",
  "role": "Senior Software Engineer",
  "interviewer_emails": ["alice@company.com", "bob@company.com"],
  "scheduled_time": "2026-06-15T14:00:00Z",
  "duration_minutes": 60
}
```

**Response (202 Accepted):**

```json
{
  "request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "status": "scheduling"
}
```

> The 202 is immediate — APEX dispatches a FastAPI BackgroundTask and returns before
> Zoom, Graph, or Slack are called. `meeting_id` and `zoom_join_url` are not available
> at this point. Poll `GET /interviews/{request_id}` until `status` is `scheduled` to
> retrieve them from the state record.

**What happens next (in the background task):**
1. APEX creates a Zoom meeting and stores the `meeting_id` and `join_url` in S3
2. A Microsoft 365 calendar event is created and sent to all interviewers
3. A Slack DM is sent to each interviewer with the Zoom link
4. A channel message is posted to the interviewers Slack channel
5. A Microsoft Graph webhook is registered to track invitation acceptance

---

## Checking Interview Status

**Via API:**

```bash
curl -s "https://apex-bot.yourcompany.com/interviews/{request_id}" | jq .
```

**Via S3 (direct):**

```bash
aws s3 cp s3://apex-interview-bot-state/interviews/{meeting-id}/state.json - | jq .
```

**State progression:**

```
created → scheduled → accepted → transcript_ready → processing → completed
                                                                     ↓
                                                  (error at any step) → failed
```

---

## What Happens After the Interview

Once Zoom sends a `recording.completed` webhook:

1. APEX verifies the HMAC-SHA256 signature
2. Downloads the VTT transcript from Zoom
3. Sends the transcript to Claude for cleanup (grammar, acronyms, normalization)
4. Sends the cleaned transcript to Claude for structured summarization
5. Creates a Confluence child page in the APEX space with the transcript and summary
6. Posts the Confluence page URL to the Slack interviewers channel
7. Sends an email notification via Microsoft Graph

The full post-meeting processing typically completes within 3–5 minutes of the webhook arriving.

---

## Reading the Confluence Page

Each interview gets its own Confluence child page under the configured parent page.
The page includes:

- Candidate name and role
- Interview date and participants
- AI-generated overall assessment
- Key discussion points (bulleted list)
- Action items
- Risk flags
- Full cleaned transcript (collapsible section)

---

## Troubleshooting

### Interview stuck in `scheduled` state

The Graph webhook may not have fired. Check:

```bash
# Verify the subscription was created
aws s3 cp s3://apex-interview-bot-state/interviews/{request_id}/state.json - | jq .graph_subscription_id

# Check CloudWatch for webhook errors
aws logs tail /apex-interview-bot/system --follow --filter-pattern "graph"
```

### Transcript not processed after meeting ended

Check whether the Zoom webhook was received and the transcript downloaded:

```bash
# Check current state
aws s3 cp s3://apex-interview-bot-state/interviews/{id}/state.json - | jq .state

# Check for error record
aws s3 cp s3://apex-interview-bot-state/interviews/{id}/error.json - 2>/dev/null | jq .

# Tail logs for Zoom webhook activity
aws logs tail /apex-interview-bot/system --filter-pattern "zoom"
```

### Confluence page not created

If `summary.json` exists in S3 but no Confluence page was created:

```bash
# Confirm summary exists
aws s3 ls s3://apex-interview-bot-state/interviews/{id}/

# Check for Confluence API errors in logs
aws logs tail /apex-interview-bot/system --filter-pattern "confluence"
```

### Container is down or unresponsive

```bash
# Open an SSM session to the instance (no SSH needed)
aws ssm start-session --target {EC2_INSTANCE_ID} --region us-east-2

# Inside the session:
docker ps
docker logs apex-bot --tail 100
curl -s http://localhost:8000/health | jq .

# Restart if needed:
docker restart apex-bot
```

---

## Rolling Back a Deployment

Use this when a new deployment introduces a regression:

```bash
REGION="us-east-2"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ECR_IMAGE="$ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/apex-interview-bot"

# Find the previous image tag
PREVIOUS_TAG=$(aws ecr describe-images \
  --repository-name apex-interview-bot \
  --region "$REGION" \
  --query 'sort_by(imageDetails, &imagePushedAt)[-2].imageTags[0]' \
  --output text)

echo "Rolling back to: $PREVIOUS_TAG"

# Deploy via SSM
aws ssm send-command \
  --instance-ids "{EC2_INSTANCE_ID}" \
  --document-name "AWS-RunShellScript" \
  --parameters "commands=[
    'aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com',
    'docker pull $ECR_IMAGE:$PREVIOUS_TAG',
    'docker stop apex-bot && docker rm apex-bot',
    'docker run -d --name apex-bot --restart=unless-stopped -p 8000:8000 -e AWS_REGION=$REGION -e SECRETS_MANAGER_SECRET_NAME=apex-interview-bot/secrets -e APP_ENV=production $ECR_IMAGE:$PREVIOUS_TAG',
    'sleep 10 && curl -sf http://localhost:8000/health || exit 1'
  ]"

# Verify
curl -s "https://apex-bot.yourcompany.com/health" | jq .
```

---

## Tailing Logs

```bash
# Live log stream
aws logs tail /apex-interview-bot/system --follow --format short

# Filter for errors only
aws logs filter-log-events \
  --log-group-name /apex-interview-bot/system \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s)000 \
  --query 'events[*].message' \
  --output text
```
