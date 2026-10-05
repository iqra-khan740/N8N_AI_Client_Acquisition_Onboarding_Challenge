# N8N AI Client Acquisition & Onboarding System

A client lifecycle system with **no frontend**: every request is simulated in Postman. It takes a lead from first inquiry through proposal, reply handling, scheduling, follow-ups and weekly reporting.

**Stack:** n8n · Airtable · Google Drive · Gmail · Google Calendar · Groq (`openai/gpt-oss-120b`) · Postman

## Workflows

| # | File | Trigger | What it does |
|---|------|---------|--------------|
| 1 | `01_new_lead_intake.json` | Webhook `POST /new-lead` | AI summarises the project and drafts the proposal, creates a Drive folder, sends the proposal email, stores the lead in Airtable as **Proposal Sent** |
| 2 | `02_client_reply_handler.json` | Webhook `POST /client-reply` | AI classifies the reply and routes it. *Ready to proceed* → calendar event for the next weekday at 11:00 PKT, confirmation email, **Call Scheduled**. *Negotiating* → **Negotiation**. *Rejected* → **Lost**. *Interested* → status unchanged, follow-up clock reset |
| 3 | `03_automated_follow_up.json` | Schedule, daily 22:01 | Finds **Proposal Sent / Negotiation / Follow Up 1 / Follow Up 2** leads with 3+ days since the last email. Sends follow-up #1, #2 or #3 (status **Follow Up 1** then **Follow Up 2**, count incremented, date updated). After 3 follow-ups and 3 more silent days, the lead becomes **Cold Lead** and gets no more emails |
| 4 | `04_weekly_management_report.json` | Schedule, Mondays 09:01 | Counts leads by status from Airtable, has AI write a management email, and sends it |

## Airtable

One table (`Clients`) with: Client ID, Name, Email, Company, Project Details, AI Project Summary, Budget, Status, Last Email Sent, Follow-up Count, Drive Folder Link, Meeting Date, Created At.

Status values: New Lead → Proposal Sent → Negotiation → Follow Up 1 → Follow Up 2 → Call Scheduled → Won / Lost / Cold Lead.

## Setup

1. n8n: **Workflows → Import from File** for each JSON in `n8n-workflows/`.
2. Create credentials for Airtable (personal access token), Gmail, Google Drive, Google Calendar and Groq, then select them on each node.
3. In the Airtable nodes, pick your own base and table (the IDs in the files are placeholders).
4. In workflow 2, pick your own calendar. In workflow 4, set the management recipient email.
5. Import `00_global_error_handler.json`, set your admin email in its Config node, activate it, and in workflows 1-4 open *Settings → Error Workflow* and select it. Failed executions then email you.
6. Activate the workflows.

## Testing with Postman

Webhook paths are `/webhook/new-lead` and `/webhook/client-reply` on your n8n URL (`/webhook-test/...` while testing in the editor).

New lead:
```json
{ "name": "John Doe", "email": "john@test.com", "company": "ABC Inc",
  "project": "Need AI chatbot for customer support", "budget": "3000" }
```

Client reply (field names as read by workflow 2):
```json
{ "Client_id": "123", "Email": "john@test.com", "message": "Looks good but pricing is high" }
```

Workflow 2 needs `Client_id` to match a lead's Client_id in Airtable, and `Email` is used for the confirmation email.

Suggested order: send a lead → send a reply for each outcome (interested, negotiating, rejected, ready to proceed) → set a lead's *Last Email Sent* to 4+ days ago and wait for / manually run workflow 3 → manually run workflow 4.

## Screenshots

### Workflow 1: New Lead Intake
![Workflow 1: New Lead Intake](https://github.com/iqra-khan740/N8N_AI_Client_Acquisition_Onboarding_Challenge/blob/main/subworkflow1.jpeg)

### Workflow 2: Client Reply Handler
![Workflow 2: Client Reply Handler](https://github.com/iqra-khan740/N8N_AI_Client_Acquisition_Onboarding_Challenge/blob/main/subworkflow2.jpeg)

### Workflow 3: Automated Follow-Up
![Workflow 3: Automated Follow-Up](https://github.com/iqra-khan740/N8N_AI_Client_Acquisition_Onboarding_Challenge/blob/main/subworkflow3.jpeg)

### Workflow 4: Weekly Management Report
![Workflow 4: Weekly Management Report](https://github.com/iqra-khan740/N8N_AI_Client_Acquisition_Onboarding_Challenge/blob/main/subworkflow4.jpeg)

## Error handling

Each AI, Airtable, Gmail, Drive and Calendar node retries on failure, and any execution that still fails triggers the global error workflow (`00_global_error_handler.json`), which emails the admin the workflow name, failed node and error message.

See `CHANGELOG.md` for the fixes made to the original exports.
