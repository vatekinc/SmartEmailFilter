# Smart Email Cleanup Service

AI-powered Gmail cleanup system that automatically classifies and trashes unnecessary emails, sends daily summaries, and learns from user feedback.

---

## Architecture Overview

```
+------------------+     +------------------+     +------------------+
| Workflow 1       |     | Workflow 2       |     | Workflow 3       |
| Smart Cleanup    |     | Daily Summary    |     | Rescue & Feedback|
| (Every 4 hrs)    |     | (10 PM EST)      |     | (Webhook)        |
+--------+---------+     +--------+---------+     +--------+---------+
         |                         |                        |
         v                         v                        v
+--------+---------+     +--------+---------+     +--------+---------+
| Gmail API        |     | Supabase Query   |     | Supabase Update  |
| Fetch + Trash    |     | Today's Actions  |     | + Gmail Untrash  |
+--------+---------+     +--------+---------+     +--------+---------+
         |                         |                        |
         v                         v                        v
+--------+-------------------------+------------------------+---------+
|                         Supabase (PostgreSQL)                       |
|  email_accounts | email_rules | email_actions | email_feedback      |
+--------+------------------------------------------------------------------------+
         |
         v
+--------+---------+
| Gemini 2.0 Flash |
| (AI Classifier)  |
+------------------+
```

---

## Components

### Infrastructure

| Component | Details |
|-----------|---------|
| **n8n Instance** | abhishek15.n8n-wsk.com |
| **Supabase** | falgykkspbtrwdcchayi.supabase.co (PropLedger DB) |
| **AI Model** | Google Gemini 2.0 Flash (free tier) |
| **Gmail (Personal)** | abhishek15@gmail.com (OAuth via n8n) |
| **Gmail (Galaxy)** | galaxyestateswv@gmail.com (OAuth via n8n) |
| **Google Cloud Project** | N8N-Access (project ID: n8n-access-487117) |

### Workflow 1: Smart Email Cleanup (`fZbUXzCfwwEJtDwT`)

**Trigger:** Every 4 hours
**File:** `workflow_1_smart_email_cleanup.json`

**Flow:**
1. **Config** - Hardcoded credentials (Supabase URL/key, Gemini API key, email address)
2. **Setup Account & Rules** - Gets/creates account in Supabase, fetches rules, gets already-processed message IDs for dedup
3. **Fetch Today's Emails** - Gmail node fetches all emails from today (excluding trash/spam/sent/draft)
4. **Classify & Log** - Two-pass classification:
   - **Pass 1 (Rules):** Matches against `email_rules` table. KEEP rules have priority over TRASH rules. Free and instant.
   - **Pass 2 (AI):** Unknown emails sent to Gemini 2.0 Flash in batches of 20. Conservative — defaults to "keep" when unsure.
5. **Is Trash?** - IF node routes trash vs keep
6. **Trash Email** - Gmail delete operation moves to trash
7. **Cleanup Complete** - NoOp terminal node

**Key Design Decisions:**
- Dedup via `email_actions` table — same email never processed twice in a day
- KEEP rules always win over TRASH rules (conservative approach)
- AI errors default to "keep" (safe fallback)
- Batch processing (20 emails per Gemini call) to stay within free tier limits

### Workflow 2: Daily Summary (`A6dnerk25g3xXnnW`)

**Trigger:** Daily at 10:00 PM EST (cron: `0 22 * * *`)
**File:** `workflow_2_daily_summary.json`

**Flow:**
1. **Config** - Credentials + summary email recipient
2. **Build Summary** - Queries Supabase for today's trashed/kept counts, builds styled HTML email with:
   - Stats cards (Processed / Trashed / Kept)
   - Table of all trashed emails with sender, subject, AI reason
   - **Rescue buttons** — one-click links to restore incorrectly trashed emails
3. **Has Activity?** - IF node skips sending if no emails were processed
4. **Send Summary Email** - Gmail sends HTML summary to abhishek15@gmail.com

**Rescue Link Format:**
```
https://abhishek15.n8n-wsk.com/webhook/email-rescue?action_id=UUID&message_id=GMAIL_ID&account_id=UUID
```

### Workflow 3: Rescue & Feedback (`QMiSz8oKXK9mY5iY`)

**Trigger:** GET webhook at `/webhook/email-rescue`
**File:** `workflow_3_rescue_feedback.json`

**Flow:**
1. **Rescue Webhook** - Receives GET request with `action_id`, `message_id`, `account_id`
2. **Process Rescue** - Does 4 things:
   - Looks up the original action in `email_actions`
   - Logs feedback to `email_feedback` table
   - Creates a **KEEP rule** for the sender (confidence: 1.0, source: `user_feedback`) so they're never trashed again
   - Deletes any AI-created TRASH rules for that sender
3. **Untrash Email** - Gmail removes TRASH label
4. **Rescue Confirmation** - Returns plain text confirmation to the browser

### Workflow 4: Flag as Trash (`N45it8TfhLnpGMxY`)

**Trigger:** GET webhook at `/webhook/email-flag-trash`
**File:** `workflow_4_flag_as_trash.json`

**Flow:**
1. **Flag Webhook** - Receives GET request with `action_id`, `message_id`, `account_id`
2. **Process Flag** - Does 4 things:
   - Looks up the original action in `email_actions` to get sender info
   - Creates a **TRASH rule** for the sender (confidence: 1.0, source: `user_feedback`)
   - Deletes any user-created KEEP rules for that sender
   - Logs feedback to `email_feedback` table with `feedback_type: flag_trash`
3. **Trash Email** - Gmail adds TRASH label
4. **Flag Confirmation** - Returns plain text confirmation to the browser

**Flag Link Format:**
```
https://abhishek15.n8n-wsk.com/webhook/email-flag-trash?action_id=UUID&message_id=GMAIL_ID&account_id=UUID
```

---

## Database Schema

**Database:** Supabase (falgykkspbtrwdcchayi.supabase.co)
**SQL File:** `001_create_tables.sql`

### Tables

#### `email_accounts`
Registry of connected email accounts. Supports multi-account.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID (PK) | Auto-generated |
| email_address | TEXT (UNIQUE) | Gmail address |
| provider | TEXT | `gmail` |
| label | TEXT | Display name |
| is_active | BOOLEAN | Enable/disable scanning |
| scan_frequency_hrs | INT | Hours between scans (default: 4) |
| aggressiveness | TEXT | `conservative`, `moderate`, `aggressive` |
| last_scanned_at | TIMESTAMPTZ | Last successful scan |

#### `email_rules`
Learned rules for classification. Supports account-specific and global rules.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID (PK) | Auto-generated |
| account_id | UUID (FK, nullable) | NULL = global rule for all accounts |
| pattern | TEXT | Match pattern (email, domain, keyword) |
| pattern_type | TEXT | `sender`, `domain`, `keyword`, `subject` |
| action | TEXT | `trash` or `keep` |
| confidence | FLOAT | 0.0 to 1.0 |
| source | TEXT | `ai`, `user_feedback`, `manual` |
| hit_count | INT | Times this rule has matched |

#### `email_actions`
Log of every cleanup action taken. Used for dedup and daily summary.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID (PK) | Auto-generated |
| account_id | UUID (FK) | Which account |
| message_id | TEXT | Gmail message ID |
| thread_id | TEXT | Gmail thread ID |
| subject | TEXT | Email subject |
| sender | TEXT | Full "From" field |
| snippet | TEXT | First 200 chars of body |
| action_taken | TEXT | `trashed`, `kept`, `skipped` |
| classified_by | TEXT | `rule`, `ai`, `manual` |
| rule_id | UUID (FK, nullable) | Which rule matched (if any) |
| ai_reasoning | TEXT | AI's explanation |

#### `email_feedback`
User rescue/approve feedback for learning.

| Column | Type | Description |
|--------|------|-------------|
| id | UUID (PK) | Auto-generated |
| account_id | UUID (FK) | Which account |
| action_id | UUID (FK) | Which action is being corrected |
| message_id | TEXT | Gmail message ID |
| feedback_type | TEXT | `rescue`, `approve`, or `flag_trash` |
| apply_rule_to | TEXT | `this_account` or `all_accounts` |

### Indexes
- `idx_email_rules_account` - Rules by account
- `idx_email_rules_pattern_type` - Rules by type + action
- `idx_email_rules_global` - Global rules (account_id IS NULL)
- `idx_email_actions_account_date` - Actions by account + date (for daily summary)
- `idx_email_actions_message` - Actions by message ID (for dedup)
- `idx_email_feedback_action` - Feedback by action

### RLS
Row Level Security is enabled on all tables. Service role has full access (n8n uses service key).

---

## Configuration

All credentials are hardcoded in n8n Code nodes (n8n Cloud doesn't support OS-level env vars).

### Workflow 1 - Config Node
```javascript
{
  EMAIL_ADDRESS: 'abhishek15@gmail.com',
  SUPABASE_URL: 'https://falgykkspbtrwdcchayi.supabase.co',
  SUPABASE_SERVICE_KEY: '<service_role_key>',
  GEMINI_API_KEY: '<google_ai_api_key>'
}
```

### Workflow 2 - Config Node
```javascript
{
  SUPABASE_URL: 'https://falgykkspbtrwdcchayi.supabase.co',
  SUPABASE_SERVICE_KEY: '<service_role_key>',
  RESCUE_WEBHOOK_URL: 'https://abhishek15.n8n-wsk.com/webhook/email-rescue',
  FLAG_WEBHOOK_URL: 'https://abhishek15.n8n-wsk.com/webhook/email-flag-trash',
  WEBHOOK_SECRET: '<hmac_secret>',
  SUMMARY_TO_EMAIL: 'abhishek15@gmail.com'
}
```

### Security — HMAC Signed URLs

All Rescue and Trash buttons in the summary email include an HMAC-SHA256 signature (`&sig=...`). The Rescue and Flag webhooks verify this signature before processing. This prevents URL tampering — if someone modifies the action_id, message_id, or account_id, the signature check fails and the request is rejected.

### Google Cloud Setup
- **Project:** N8N-Access (n8n-access-487117)
- **API Enabled:** Gemini API (Generative Language API)
- **API Key:** Unrestricted, created from Cloud Console > Credentials

---

## How the Learning Loop Works

```
1. Workflow 1 runs → AI classifies email as "trash" → email moved to trash
2. Workflow 2 runs → sends summary email with "Rescue" buttons
3. User clicks "Rescue" → Workflow 3 triggers:
   a. Email restored to inbox
   b. KEEP rule created for that sender (confidence=1.0)
   c. Any AI TRASH rules for that sender deleted
4. Next time Workflow 1 runs → rule matches FIRST (before AI) → email kept
```

The system gets smarter over time as users provide feedback. Rules from user feedback have confidence=1.0 and always override AI decisions.

### Flag as Trash (Inverse Learning)

```
1. Workflow 1 runs → AI keeps an email (wrong decision)
2. Workflow 2 runs → sends summary email with "Flag Trash" buttons next to kept emails
3. User clicks "Flag Trash" → Workflow 4 triggers:
   a. Email moved to trash
   b. TRASH rule created for that sender (confidence=1.0)
   c. Any user-created KEEP rules for that sender deleted
   d. Feedback logged
4. Next time Workflow 1 runs → rule matches FIRST → email auto-trashed
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| All emails show "AI_ERROR: Request failed with status code 403" | Gemini API not enabled | Enable "Gemini API" in Google Cloud Console > APIs & Services > Library |
| "All emails already processed" | Dedup preventing reprocessing | `DELETE FROM email_actions WHERE created_at >= CURRENT_DATE;` |
| Emails not being fetched | Gmail OAuth expired | Re-authenticate Gmail credential in n8n |
| Wrong node references | Import adds suffix (e.g., `Config1`) | Update `$('NodeName')` references in Code nodes |

### Clearing Test Data
```sql
-- Clear today's actions (for re-testing)
DELETE FROM email_actions WHERE created_at >= CURRENT_DATE;

-- Clear ALL data (full reset)
DELETE FROM email_feedback;
DELETE FROM email_actions;
DELETE FROM email_rules;
DELETE FROM email_accounts;
```

---

## Cost Analysis

| Component | Cost |
|-----------|------|
| Gemini 2.0 Flash | Free (15 RPM, 1M tokens/day) |
| Supabase | Free tier (shared with PropLedger) |
| n8n Cloud | Existing subscription |
| **Total** | **$0/month** |

---

## Future Enhancements

- ~~Add support for multiple Gmail accounts~~ ✅ Done (Galaxy Estates added)
- ~~Implement "approve" feedback (confirm AI was correct)~~ ✅ Done (Workflow 4: Flag as Trash)
- Auto-create TRASH rules when AI consistently trashes same sender (3+ times)
- Add weekly/monthly analytics dashboard
- Support for label-based organization (not just trash/keep)

---

## Files

| File | Description |
|------|-------------|
| `001_create_tables.sql` | Supabase database schema (4 tables, indexes, RLS) |
| `workflow_1_smart_email_cleanup.json` | Main cleanup workflow (n8n import) |
| `workflow_2_daily_summary.json` | Daily summary email workflow (n8n import) |
| `workflow_3_rescue_feedback.json` | Rescue webhook workflow (n8n import) |
| `workflow_4_flag_as_trash.json` | Flag as Trash webhook workflow (n8n import) |
| `galaxy_workflow_1_smart_cleanup.json` | Galaxy Estates cleanup workflow (n8n import) |
| `galaxy_workflow_2_daily_summary.json` | Galaxy Estates daily summary (n8n import) |
| `galaxy_workflow_3_rescue.json` | Galaxy Estates rescue webhook (n8n import) |
| `galaxy_workflow_4_flag_as_trash.json` | Galaxy Estates flag as trash webhook (n8n import) |
| `README.md` | This documentation |

---

## Multi-Account Setup

The system supports multiple Gmail accounts. Each account has its own set of 4 workflows on its own n8n instance. All accounts share the same Supabase database — data is isolated by `account_id`.

### Accounts

| Account | Label | n8n Instance | Webhook Paths |
|---------|-------|--------------|---------------|
| abhishek15@gmail.com | Personal | abhishek15.n8n-wsk.com | `/email-rescue`, `/email-flag-trash` |
| galaxyestateswv@gmail.com | Galaxy Estates | galaxyestateswv.n8n-wsk.com | `/email-rescue`, `/email-flag-trash` |

### Adding a New Account

1. **Google Cloud** — Add the email as a test user in Google Cloud Console (OAuth consent screen, project: N8N-Access).
2. **Gmail OAuth** — Create a Gmail OAuth2 credential in the n8n instance using the same Client ID/Secret from the N8N-Access project.
3. **Import 4 workflows** — Copy the galaxy workflow files, update:
   - `EMAIL_ADDRESS` in Config nodes
   - `RESCUE_WEBHOOK_URL` and `FLAG_WEBHOOK_URL` to point to the correct n8n instance
   - `SUMMARY_TO_EMAIL` in Daily Summary Config
   - Account label in Setup Account & Rules
4. **Connect Gmail credential** on all Gmail nodes (Fetch, Trash, Untrash, Send)
5. **Activate** all 4 workflows

### Galaxy Estates Workflow IDs

| Workflow | ID | Webhook Path |
|----------|----|--------------|
| Smart Cleanup - Galaxy | *import to get ID* | N/A (schedule trigger) |
| Daily Summary - Galaxy | *import to get ID* | N/A (schedule trigger) |
| Rescue - Galaxy | *import to get ID* | `/webhook/email-rescue` |
| Flag as Trash - Galaxy | *import to get ID* | `/webhook/email-flag-trash` |

---

*Built: February 17, 2026 | Updated: February 19, 2026*
*Stack: n8n + Gmail API + Google Gemini 2.0 Flash + Supabase*
