# Cold Outreach Agent (Cold Outreach Agent with Prompt Optimization)

A multi-agent cold outreach system built in n8n that drafts personalized cold emails, tracks conversation history, and automatically switches messaging angles when a lead doesn't reply. Built for Binox Limited's outbound sales workflow targeting the Greater Bay Area market.

---

## Overview

This system runs four coordinated n8n workflows:

| Workflow | Trigger | Purpose |
|---|---|---|
| `Personal Message Workflow` | Daily schedule (10:30 AM) | Sends first-touch cold email to new, unsent leads |
| `Follow Up (3 days) Work Flow` | Daily schedule (11:00 AM) | Sends 2nd touch if no reply after 3+ days |
| `Follow Up (7 days) Work Flow` | Daily schedule (11:00 AM) | Sends 3rd and final touch if no reply after 4+ more days |
| `Response Tracker` | IMAP email trigger | Detects replies and marks `reply_received = true` in the sheet |

All lead data, conversation state, and message history are stored in a **Google Sheet** (used as a lightweight JSON memory store).

---

## Architecture

### Multi-Agent Design

Each sending workflow uses two separate Gemini model nodes, intentionally split by task complexity:

```
┌─────────────────────────────────────────────────────┐
│                  Per-lead execution                 │
│                                                     │
│  ┌─────────────────────┐   ┌─────────────────────┐  │
│  │   Strategy Agent    │──▶│  Copywriter Agent   │  │
│  │                     │   │                     │  │
│  │ gemini-3.1-flash-   │   │ gemini-2.5-flash-   │  │
│  │ lite-preview        │   │ lite                │  │
│  │                     │   │                     │  │
│  │ Input: lead profile │   │ Input: lead profile │  │
│  │ + chat_history JSON │   │ + angle + reason    │  │
│  │                     │   │                     │  │
│  │ Output: JSON        │   │ Output: email body  │  │
│  │ { angle, reason }   │   │ only (no subject    │  │
│  │                     │   │ on follow-ups)      │  │
│  └─────────────────────┘   └─────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**Strategy agent** (`Message a model1`): Uses a lighter model to make a low-token JSON decision — which angle to use next, based on conversation history. Outputs `{ "angle": "...", "reason": "..." }` only.

**Copywriter agent** (`Message a model`): Receives the angle decision and full lead context, then drafts the actual email body. Uses a more capable model for quality generation.

### Cost-Aware Model Selection

| Agent | Model | Role |
|---|---|---|
| Strategy | `gemini-3.1-flash-lite-preview` | Short JSON output, angle selection only |
| Copywriter | `gemini-2.5-flash-lite` | Full email body generation |

> **Testing note:** During development and testing, both agents were run on `gemini-3.1-flash-lite-preview` due to Google's free tier having a higher requests-per-minute quota for that model compared to `gemini-2.5-flash-lite`. In actual production usage, `gemini-2.5-flash-lite` is the cheaper option at scale and is what the copywriter nodes are configured to use in the committed workflows.

---

## Prompt Evolution Logic

The core intelligence of the system is the angle rotation mechanism — the strategy agent reads the full conversation history for each lead and picks the next unused messaging angle.

### Available Angles

| Angle | Description |
|---|---|
| `cost_savings` | Lead with 90-day ROI and $5M cost reduction stats |
| `social_proof` | Lead with a similar company success story |
| `curiosity_hook` | Open with a provocative question about their industry |
| `case_study` | Reference a specific logistics/retail/manufacturing win |

### How Angles Evolve Across Touches

```
Touch 1 (Personal Message Workflow)
  └── Strategy agent reads: angle_used, reply_received, follow_up_angle
  └── Rule: if angle_used is empty → default to cost_savings
  └── Rule: never reuse an angle where replied = false
  └── Copywriter drafts full email with subject line
  └── chat_history updated: [{ touch: 1, angle, sent_at, replied: false }]

Touch 2 (Follow Up — 3 days, no reply)
  └── Strategy agent reads: full chat_history JSON array + angle_used + follow_up_angle
  └── Picks next unused angle from pool (e.g. cost_savings used → try social_proof)
  └── Copywriter drafts follow-up body only, different angle, no mention of previous email
  └── chat_history updated: [..., { touch: 2, angle, sent_at, replied: false }]

Touch 3 (Follow Up — 7 days, still no reply)
  └── Strategy agent reads: full chat_history, angle_used, follow_up_angle
  └── Picks next unused angle (e.g. → curiosity_hook or case_study)
  └── Copywriter drafts a short 2-3 sentence final check-in, framed as last touch
  └── chat_history updated: [..., { touch: 3, angle, sent_at, replied: false }]
```

### chat_history Schema

The `chat_history` column in the Google Sheet stores a JSON array serialized as a string. Each entry represents one sent touch:

```json
[
  { "touch": 1, "angle": "cost_savings", "sent_at": "2025-04-01T10:30:00.000Z", "replied": false },
  { "touch": 2, "angle": "social_proof", "sent_at": "2025-04-04T11:00:00.000Z", "replied": false },
  { "touch": 3, "angle": "curiosity_hook", "sent_at": "2025-04-09T11:00:00.000Z", "replied": false }
]
```

When the Response Tracker detects a reply, it sets `reply_received = true` on the lead's row. Future strategy agent calls would see `replied: true` in the history and would know not to rotate away from that angle (it worked).

### Why This Qualifies as Prompt Evolution

The prompt sent to the strategy agent changes on every subsequent touch because it includes the live `chat_history` array. The agent is not just following hardcoded rules — it is instructed to read the history and reason about which angle is unused and most likely to work given what has already been tried. As the history grows, the effective strategy prompt evolves with it.

---

## Memory Store

All state is stored in a **Google Sheet** (`Binox Leads Sheet`, sheet: `leads-100`). This acts as the JSON memory store.

### Sheet Columns

| Column | Type | Description |
|---|---|---|
| `first_name` | string | Used as the row match key |
| `last_name` | string | Lead surname |
| `role` | string | Job title |
| `company` | string | Company name |
| `industry` | string | Industry category |
| `email` | string | Recipient email |
| `linkedin_url` | string | LinkedIn profile (optional) |
| `notes` | string | Freeform context about the lead |
| `subject` | string | Subject line from touch 1 |
| `outreach` | string | Full body of touch 1 email |
| `date_sent` | ISO string | Timestamp of touch 1 |
| `sent` | boolean | Whether touch 1 was sent |
| `angle_used` | string | Angle used in touch 1 |
| `reply_received` | boolean | Whether any reply was detected |
| `reply_date` | ISO string | Timestamp of reply (if any) |
| `follow_up_angle` | string | Angle used in touch 2 |
| `follow_up_outreach` | string | Full body of touch 2 email |
| `follow_up_date` | ISO string | Timestamp of touch 2 |
| `follow_up_2_angle` | string | Angle used in touch 3 |
| `follow_up_outreach_2` | string | Full body of touch 3 email |
| `follow_up_2_date` | ISO string | Timestamp of touch 3 |
| `chat_history` | JSON string | Serialized array of all touch records |

---

## Workflow Details

### Personal Message Workflow

**Trigger:** Daily at 10:30 AM

**Filter conditions (all must be true):**
- `outreach` is empty (not yet contacted)
- `subject` is empty
- `sent` is false

**Flow:**
1. Read all rows from sheet
2. Filter to unsent leads
3. Strategy agent selects angle → JSON parsed → Copywriter drafts email with subject
4. Subject/body split via JS code node
5. Touch 1 entry pushed to `chat_history`
6. Row updated in sheet (outreach, subject, date_sent, angle_used, chat_history)
7. Email sent via SMTP

### Follow Up (3 days) Workflow

**Trigger:** Daily at 11:00 AM

**Filter conditions (all must be true):**
- `outreach` is not empty (touch 1 was sent)
- `reply_received` is false
- `date_sent` is more than 3 days ago
- `follow_up_outreach` is empty (touch 2 not yet sent)

**Flow:** Same agent pattern as above. Touch 2 entry pushed to history. Updates `follow_up_outreach`, `follow_up_angle`, `follow_up_date`, `chat_history`.

### Follow Up (7 days) Workflow

**Trigger:** Daily at 11:00 AM

**Filter conditions (all must be true):**
- `follow_up_outreach` is not empty (touch 2 was sent)
- `reply_received` is false
- `follow_up_date` is more than 4 days ago (7+ days total since touch 1)
- `follow_up_outreach_2` is empty (touch 3 not yet sent)

**Flow:** Same pattern. Copywriter is instructed to keep it to 2-3 sentences as a final check-in. Updates `follow_up_2_angle`, `follow_up_outreach_2`, `follow_up_2_date`, `chat_history`.

### Response Tracker

**Trigger:** IMAP — new email received in inbox

**Flow:**
1. Extract sender email from incoming message
2. Look up matching row in sheet by email
3. Set `reply_received = true` and `reply_date = now`


---
## Setup

### Prerequisites

- n8n (self-hosted) — see [n8n self-hosting docs](https://docs.n8n.io/hosting/)
- Google Sheets OAuth2 credential configured in n8n
- Google Gemini (PaLM) API credential configured in n8n
- SMTP credential configured in n8n
- IMAP credential configured in n8n (for Response Tracker)

### Self-Hosting n8n

This project was built and tested on a **self-hosted n8n instance** running locally. To replicate:

1. Install n8n via npm or Docker:
   ```bash
   # npm
   npm install -g n8n
   n8n start

   # or Docker
   docker run -it --rm --name n8n -p 5678:5678 n8nio/n8n
   ```
2. n8n will be available at `http://localhost:5678` by default.

### Exposing n8n with ngrok

For the **IMAP Response Tracker** and any webhook-based triggers to function properly, n8n needs to be reachable from the internet. This was achieved during development using **[ngrok](https://ngrok.com/)**, which creates a secure public tunnel to the local n8n instance.

```bash
# After starting n8n locally, run:
ngrok http 5678
```

ngrok will output a public URL (e.g. `https://abc123.ngrok-free.app`). Set this as your n8n webhook base URL:

- In n8n, go to **Settings → General**
- Set **Webhook URL** to your ngrok URL (e.g. `https://abc123.ngrok-free.app/`)

> **Note:** The free tier of ngrok assigns a new URL each time it is restarted. For persistent deployments, use a paid ngrok plan with a fixed domain, or deploy n8n to a cloud server with a stable URL.

### Google Sheet Structure

Create a sheet with the column headers listed in the Memory Store section above. The sheet ID and GID in the workflow JSONs will need to be updated to point to your own sheet.

### Importing Workflows

1. In n8n, go to **Workflows → Import from file**
2. Import each of the four JSON files
3. Update the credential references in each workflow to match your own n8n credentials
4. Update the Google Sheet document ID and sheet GID in all Google Sheets nodes
5. Activate the workflows

---

## Tech Stack

| Component | Tool |
|---|---|
| Workflow orchestration | n8n (self-hosted) |
| Local tunnel / public URL | ngrok |
| AI models | Google Gemini (via PaLM API) |
| Memory / data store | Google Sheets |
| Email sending | SMTP |
| Reply detection | IMAP |
