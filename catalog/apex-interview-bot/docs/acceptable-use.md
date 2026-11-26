# Acceptable Use — APEX Interview Automation Bot

## Intended Users

- **Recruiting coordinators** who initiate interview scheduling requests via the APEX API
- **HR program managers** who review published interview summaries in Confluence
- **Engineering interviewers** who receive Slack notifications and Microsoft 365 calendar invitations

---

## Permitted Use Cases

- Scheduling structured interviews for open engineering, product, and operations roles
- Automating Zoom meeting creation and Microsoft 365 calendar event distribution
- Processing post-interview transcripts through AI cleanup and summarization
- Publishing standardized interview summaries to the APEX Confluence space

---

## Prohibited Use Cases

- General meeting scheduling unrelated to structured interviews
- Recording or processing non-interview Zoom sessions
- Using APEX to schedule meetings for any purpose other than formal candidate interviews
- Storing personally identifiable information beyond what is required for the interview record

---

## Data Handling

| Data | Storage | Retention |
|------|---------|-----------|
| Raw transcript (VTT) | S3: `interviews/{request_id}/transcript.txt` | Per S3 lifecycle policy [OPEN] |
| Cleaned transcript | S3: `interviews/{request_id}/transcript-clean.txt` | Same as above |
| Interview summary (JSON) | S3: `interviews/{request_id}/summary.json` | Same as above |
| API keys and tokens | AWS Secrets Manager | Rotated per secret owner policy |
| Application logs | CloudWatch Logs | 90 days [INTENDED] |

> No data is stored in a relational database. S3 is the sole persistence layer.
> Secrets are fetched at container startup and cached in-process — never written to logs.

---

## Compliance Notes

> The structured data classification, retention policy, and regulatory scope for this
> service are tracked in `entry.yaml` under `architectureNotes.dataClassification` —
> that is the canonical location tooling checks for resolution. The summary below
> mirrors those open items for human readability.

- **[OPEN]** Retention policy for interview transcripts in S3 has not been formally defined.
- **[INTENDED]** S3 bucket lifecycle rules will enforce a 90-day retention window.
- **[OPEN]** Legal review of AI-generated summaries used in hiring decisions is pending.
- **[OPEN]** GDPR / CCPA applicability to candidate transcript data has not been assessed.
- **[OPEN]** Anthropic API data retention tier (zero-retention vs. standard) not yet confirmed.
