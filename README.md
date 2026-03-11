# Kumo Capital — n8n Workflows

Three n8n automation workflows for Kumo Capital's commercial real estate outreach system. All DB nodes use **Supabase (Postgres)** credentials.

---

## Workflows

### 1. `fil-chatbot-workflow.json` — Inbound iMessage Chatbot
Handles all inbound iMessages via Blooio webhook. For each message it:
- Looks up the contact and active thread in Supabase
- Fetches the KB doc from SharePoint + calendar from Outlook
- Passes full context to Claude (Haiku 4.5) via the n8n Agent node
- Parses the AI reply, adds a human-like typing delay, then sends via Blooio
- Logs both inbound and outbound messages to `app.messages`, updates the thread
- Escalates to Outlook email if the model flags a handover (pricing, LOI, etc.)

**Trigger:** Blooio webhook POST → `https://your-n8n-instance/webhook/blooio-inbound`

---

### 2. `fil-outreach-trigger.json` — Single-Thread Cold Outreach
Manual trigger workflow. Picks one untouched thread (touchpoint_count = 0), generates a personalised opening message via Claude, adds a random human-like delay, and sends via Blooio. Logs the message and updates the thread.

**Trigger:** Manual (run from n8n UI)

---

### 3. `fil-batch-outreach.json` — Batch Cold Outreach
Manual trigger workflow. Queries all active threads with no touchpoints, loops through each one, sends a cold opening message via Blooio, logs it, and waits 5 seconds between sends to avoid rate limits.

**Trigger:** Manual (run from n8n UI)

---

## Setup

### 1. Supabase Credential
All Postgres nodes reference a credential named **"Supabase account"** with the placeholder ID `REPLACE_WITH_SUPABASE_CREDENTIAL_ID`.

In n8n:
1. Go to **Credentials → Add credential → Postgres**
2. Fill in your Supabase connection details (host: `db.<project-ref>.supabase.co`, port: `5432`, SSL required)
3. Note the credential ID assigned by n8n
4. Either import the workflows and reconnect credentials via the UI, or do a global find-replace of `REPLACE_WITH_SUPABASE_CREDENTIAL_ID` in the JSON files before importing

### 2. Other Credentials
| Credential name | Used in |
|---|---|
| `Anthropic account` | All three workflows — Claude API key |
| `Header Auth account 3` | Blooio API key (Authorization header) |
| `Microsoft Outlook account` | Chatbot — KB SharePoint + Calendar + Escalation email |

### 3. Import to n8n
1. Open n8n → **Workflows → Import from file**
2. Select each JSON file
3. Reconnect credentials (n8n will highlight any missing ones in red)
4. Activate the chatbot webhook workflow — all others are manual-only

---

## Database Schema
All queries target tables in the `app` schema on Supabase:

- `app.contacts` + `app.contact_phones`
- `app.properties` + `app.listings`
- `app.message_threads`
- `app.messages`
- `app.arbitrage_events`
- `app.handover_events`

Enums used: `message_direction`, `message_sender`, `conversation_stage`, `handover_reason`, `arbitrage_type`

---

## Known Bugs Fixed in This Repo
- `DB: Lookup Contact + Thread2` — added `queryParams` for `$1` phone bind
- `DB: Log Messages + Update Thread2` and `DB: Log Arbitrage + Handover2` — fixed missing `=` prefix on n8n expression (`={{ $json.query }}`)
- `DB: Get Pending Outreach` — fixed `l.cap_rate` → `l.listed_cap_rate`, added `AND cp.is_primary = TRUE` filter
- `DB: Log + Update Thread` (batch) — fixed positional params (`$1`/`$2`) with proper `queryParams` binding and enum casts (`::message_direction`, `::message_sender`); changed `touchpoint_count = 1` to `touchpoint_count + 1`
