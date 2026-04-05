# 🔒 Secure WhatsApp AI Assistant

> A fully automated, **zero-cost**, self-hosted WhatsApp assistant powered by Groq + n8n — with a contact whitelist, dynamic AI personality, 10-turn memory, and rate-limit resilience. No third-party data leaks. No per-message charges.

---

## ✨ Features

| Feature | Details |
|---|---|
| 🤖 **AI Engine** | Groq API — `llama-3.3-70b` for complex queries, `llama-4-scout` for quick replies |
| 🔐 **Gatekeeper** | Whitelist-only access via Google Sheets — unknown senders are silently logged |
| 🎭 **Dynamic Personality** | Formal/professional for Clients, casual/brief for Personal contacts |
| 🧠 **10-Turn Memory** | Per-contact conversation history stored in Supabase (auto-pruned) |
| 🛡️ **PII Sanitizer** | Strips emails, phone numbers & card numbers from logs before any AI call |
| ⚡ **Rate Limit Resilience** | Reads Groq response headers, waits exact reset time, auto-retries once |
| ➕ **Whitelist Manager** | HTML button panel + dedicated webhook to add new authorized contacts |
| 💸 **Zero Cost** | Groq free tier + Supabase free tier + self-hosted n8n + Evolution API Docker |

---

## 🏗️ Architecture

```
WhatsApp (User)
      │
      ▼
Evolution API (Docker) ──webhook──▶ n8n Workflow
                                         │
                              ┌──────────┴──────────┐
                              │                     │
                        [Gatekeeper]         [NOT Whitelisted]
                        Google Sheets              │
                              │              Log to Leads Sheet
                        [Authorized]               │
                              │                   STOP
                     Build System Prompt
                     (Client / Personal)
                              │
                     Fetch Memory (Supabase)
                              │
                     Groq API Inference ◀─── Retry if 429
                              │
                     Send Reply (Evolution API)
                              │
                     Save Turn to Supabase
```

> **Zero-leaking guarantee:** The data pipeline is strictly `WhatsApp → n8n → Groq → WhatsApp`. No OpenAI wrappers, no Zapier, no third-party relay services.

---

## 📦 What's Included

```
📁 project/
├── whatsapp_assistant_workflow.json   # Main n8n workflow (21 nodes)
├── whitelist_manager_workflow.json    # Add-contact workflow (6 nodes)
└── whitelist_panel.html               # Admin button UI (open in browser)
```

---

## ⚙️ Tech Stack

- **Orchestration:** [n8n](https://n8n.io) v2.0+ (self-hosted)
- **AI Inference:** [Groq API](https://console.groq.com) (free tier)
- **WhatsApp Bridge:** [Evolution API](https://github.com/EvolutionAPI/evolution-api) (Docker)
- **Memory Store:** [Supabase](https://supabase.com) (free tier) PostgreSQL
- **Whitelist Source:** Google Sheets

---

## 🚀 Setup Guide

### Prerequisites

- Docker installed on your server
- n8n self-hosted (v2.0+)
- A Google account (for Sheets)
- A Supabase account (free)
- A Groq account (free)
- A WhatsApp number to connect

---

### Step 1 — Import Workflows into n8n

1. Open n8n → **Workflows** → **Add Workflow** → **Import from file**
2. Import `whatsapp_assistant_workflow.json`
3. Repeat for `whitelist_manager_workflow.json`

You'll have **two separate workflows** in your dashboard. Keep them separate — do not merge them.

---

### Step 2 — Set n8n Variables

Go to **Settings → Variables** and create the following:

| Variable | Description | Example |
|---|---|---|
| `GROQ_API_KEY` | From [console.groq.com](https://console.groq.com) | `gsk_...` |
| `GOOGLE_SHEET_ID` | From the Sheet URL | `1BxiMV...` |
| `SUPABASE_URL` | Project URL (no `https://`) | `xyz.supabase.co` |
| `SUPABASE_ANON_KEY` | Anon/public key from Supabase settings | `eyJ...` |
| `EVOLUTION_API_HOST` | Host:port of your Evolution container | `localhost:8080` |
| `EVOLUTION_INSTANCE` | Your Evolution instance name | `mybot` |
| `EVOLUTION_API_KEY` | From Evolution dashboard | `abc123` |
| `WHITELIST_SECRET` | Any secret string you choose | `mysecret123` |

---

### Step 3 — Google Sheets Setup

Create a new Google Sheet with **two tabs**:

**Tab 1: `Whitelist`**

| Phone | Name | Tag | Status |
|---|---|---|---|
| +923001234567 | Ali Hassan | Client | Active |
| +923451234567 | Sara | Personal | Active |

- `Tag` must be exactly `Client` or `Personal`
- `Status` must be exactly `Active` or `Blocked`

**Tab 2: `Leads`**

Leave this empty — the workflow fills it automatically for unauthorized senders.

Headers: `Phone | Name | Message_Sanitized | Timestamp | Status`

---

### Step 4 — Connect Google Sheets Credential in n8n

1. Go to **Credentials → Add → Google Sheets OAuth2**
2. Authenticate with your Google account
3. In the **main workflow**, assign this credential to:
   - `Fetch Whitelist from Google Sheets`
   - `Log to Leads Sheet`
4. In the **whitelist manager workflow**, assign it to:
   - `Add to Whitelist Sheet`

---

### Step 5 — Supabase Database Setup

In your Supabase project, go to **SQL Editor** and run:

```sql
CREATE TABLE conversation_memory (
  id        BIGSERIAL PRIMARY KEY,
  phone     TEXT NOT NULL,
  role      TEXT NOT NULL,       -- 'user' or 'assistant'
  content   TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Index for fast per-contact lookups
CREATE INDEX ON conversation_memory(phone, created_at DESC);
```

Copy your **Project URL** and **Anon Key** from Project Settings → API into your n8n variables.

---

### Step 6 — Evolution API (Docker)

Run Evolution API on your server:

```bash
docker run -d \
  --name evolution-api \
  -p 8080:8080 \
  -e AUTHENTICATION_API_KEY=your_api_key \
  atendai/evolution-api:latest
```

Then:
1. Open `http://YOUR_SERVER:8080` → create an instance
2. Scan the QR code with WhatsApp on your phone
3. Set the webhook URL to:
   ```
   http://YOUR_SERVER:5678/webhook/whatsapp-incoming
   ```

---

### Step 7 — Activate Both Workflows

Toggle both workflows to **Active** in n8n. The main workflow begins listening for incoming WhatsApp messages immediately.

---

### Step 8 — Whitelist Manager Panel

1. Open `whitelist_panel.html` in a text editor
2. Update this line at the top of the `<script>`:
   ```js
   const N8N_WEBHOOK_URL = 'http://YOUR_SERVER:5678/webhook/whitelist-add';
   ```
3. Open the file in your browser
4. Fill in phone, name, contact type, and your `WHITELIST_SECRET` → click **Add to Whitelist**

> You can host this HTML file anywhere — locally, on Netlify, or on your own server.

---

### Step 9 — Test

1. **Authorized test:** Add your number to the Whitelist sheet (Tag: `Personal`, Status: `Active`). Send a message to the connected WhatsApp number — you should get a casual AI reply.
2. **Unauthorized test:** Send from an unlisted number. No reply should come. Check the `Leads` tab in your Google Sheet — the attempt should be logged there.
3. **Rate limit test:** The workflow auto-handles 429 responses from Groq. No action needed.

---

## 🧠 How the AI Personality Works

The system prompt is dynamically generated per contact based on their `Tag` in the whitelist:

**Client (Formal)**
- Professional, solution-oriented tone
- Uses bold text and bullet points for clarity
- Always offers further assistance
- Matches language (English/Urdu)

**Personal (Casual)**
- Brief and conversational — like texting a smart friend
- Short answers for short questions
- No unnecessary formalities
- Matches language (English/Urdu/Roman Urdu)

---

## 🔄 Model Selection Logic

The workflow automatically picks the right Groq model based on message complexity:

```
Message > 20 words OR contains a question with >10 words
    → llama-3.3-70b-versatile   (deep reasoning, up to 800 tokens)

Short / simple message
    → llama-4-scout-17b         (fast, up to 300 tokens)
```

This keeps you well within Groq's free-tier TPM limits.

---

## 🛡️ Security & Privacy

- **No third-party wrappers.** Data flows directly: `WhatsApp → n8n → Groq → WhatsApp`
- **PII is stripped** from all logs before any data leaves your server. Emails, phone numbers, and card numbers are replaced with `[REDACTED]` tokens in the Leads log
- **Whitelist panel is secret-gated.** The `WHITELIST_SECRET` must be included in every add-contact request or it's rejected with a 400
- **Group messages and self-messages** are silently dropped at the extraction stage — the AI never sees them
- **Supabase memory** stores only conversation content, never raw message metadata

---

## 💰 Cost Breakdown

| Service | Plan | Monthly Cost |
|---|---|---|
| n8n | Self-hosted | $0 |
| Groq API | Free tier | $0 |
| Evolution API | Self-hosted Docker | $0 |
| Supabase | Free tier (500MB) | $0 |
| Google Sheets | Free | $0 |
| **Total** | | **$0.00** |

> The only potential cost is your server/VPS to host n8n + Evolution API. A $4–6/month VPS (e.g., Hetzner, DigitalOcean) is sufficient.

---

## 🗂️ Google Sheet Column Reference

### Whitelist Tab
| Column | Values | Required |
|---|---|---|
| A — Phone | `+923001234567` | ✅ |
| B — Name | Any string | ✅ |
| C — Tag | `Client` or `Personal` | ✅ |
| D — Status | `Active` or `Blocked` | ✅ |

### Leads Tab (auto-filled)
| Column | Description |
|---|---|
| Phone | Sender's number |
| Name | WhatsApp display name |
| Message_Sanitized | PII-stripped message content |
| Timestamp | ISO timestamp |
| Status | `Unreviewed` (change to `Added` once you whitelist them) |

---

## 🔧 Troubleshooting

**No reply to messages**
- Confirm the main workflow is set to Active
- Check that your Evolution API webhook URL points to the correct n8n host and port
- Verify `GROQ_API_KEY` is valid in n8n Variables

**Unauthorized contacts not showing in Leads**
- Confirm the Google Sheets credential is assigned to the `Log to Leads Sheet` node
- Check that `GOOGLE_SHEET_ID` is the ID from the Sheet URL, not the full URL

**Memory not persisting**
- Confirm the Supabase table was created with the SQL above
- Verify `SUPABASE_URL` has no `https://` prefix
- Check the anon key is from **Project Settings → API → anon public**

**Whitelist panel shows "Could not reach n8n"**
- Confirm the whitelist manager workflow is Active
- Check the `N8N_WEBHOOK_URL` in the HTML file matches your server address exactly

---

## 📄 License

MIT — free to use, modify, and deploy for personal or commercial projects.

---

*Built for WebCorps · Self-hosted · Zero cost · Zero data leaks*
