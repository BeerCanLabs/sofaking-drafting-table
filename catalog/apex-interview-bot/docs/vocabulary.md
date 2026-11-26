# Vocabulary — APEX Interview Automation Bot

| Term | Definition |
|------|------------|
| **APEX** | Automated Processing and Exchange for interviews. Internal name for the interview automation bot built by the aiops team at Sofa King. |
| **Interview State** | The current lifecycle stage of an interview, tracked in `state.json` on S3. One of: `created`, `scheduled`, `accepted`, `transcript_ready`, `processing`, `completed`, or `failed`. |
| **VTT** | WebVTT (Web Video Text Tracks). The transcript format produced by Zoom Recordings. Contains timestamped speaker turns in a structured text format. |
| **Raw Transcript** | The verbatim Zoom VTT transcript as downloaded from the Zoom API. Not suitable for publication — contains filler words and raw timestamps. |
| **Cleaned Transcript** | The AI-processed version of the raw transcript, produced by Anthropic Claude. Filler words removed, acronyms expanded, speaker names normalized. |
| **Summary** | The structured JSON document produced by Claude from the cleaned transcript. Contains overall assessment, key points, action items, and risks. Published to Confluence as the permanent interview record. |
| **Graph Subscription** | A Microsoft Graph webhook subscription that fires when a calendar event changes state (e.g., an invitee accepts). APEX creates one subscription per interview. |
| **clientState** | A secret string APEX sets when creating a Microsoft Graph webhook subscription. Graph includes this value in every change notification. APEX verifies it matches before processing the event, preventing spoofed webhooks. |
| **HMAC-SHA256** | Hash-based message authentication code using SHA-256. Used by Zoom to sign webhook payloads. APEX recomputes the signature on every inbound Zoom webhook and rejects requests where signatures do not match or the timestamp exceeds a 30-second window. |
| **BackgroundTasks** | FastAPI's built-in mechanism for dispatching long-running work off the HTTP request-response cycle. APEX uses this to run interview workflows asynchronously — the HTTP response returns 202 Accepted immediately. |
| **S3 State Model** | APEX's persistence strategy. Each interview is a S3 key prefix (`interviews/{meeting-id}/`). JSON files within that prefix serve as both state indicators and data payloads, replacing a relational database entirely. |
| **Candidate** | The person being interviewed. Their name and role are captured at scheduling time and included in all Slack notifications and the Confluence summary page. |
| **Coordinator** | The recruiting coordinator or HR staff member who initiates the interview scheduling request by calling `POST /interviews`. |
| **Server-to-Server OAuth** | Zoom's authentication method for backend integrations. Uses Account ID + Client ID + Client Secret to obtain a token — no user login required. |
| **[DOCUMENTED]** | Observed in source code or AWS. This behaviour currently exists and is implemented. |
| **[INTENDED]** | Target state that is planned but not yet built. Do not treat as implemented. |
| **[OPEN]** | An architectural or policy decision that has not yet been resolved. |
