# CleanInbox SaaS — Architecture & Implementation Guide

> Transform the existing n8n-based email cleanup system into a multi-tenant, serverless SaaS product where users sign up, connect their Gmail, and get automatic AI-powered email cleanup.

---

## Table of Contents

1. [Current System Analysis](#1-current-system-analysis)
2. [SaaS Vision](#2-saas-vision)
3. [Recommended Tech Stack](#3-recommended-tech-stack)
4. [System Architecture](#4-system-architecture)
5. [Database Schema](#5-database-schema)
6. [Gmail OAuth Strategy](#6-gmail-oauth-strategy)
7. [Serverless Processing Pipeline](#7-serverless-processing-pipeline)
8. [Frontend Dashboard](#8-frontend-dashboard)
9. [Cost Analysis](#9-cost-analysis)
10. [Implementation Roadmap](#10-implementation-roadmap)
11. [Key Risks & Mitigations](#11-key-risks--mitigations)

---

## 1. Current System Analysis

### What Exists Today

The current system runs on **n8n Cloud** with **4 workflows** per email account, backed by **Supabase PostgreSQL** and **Google Gemini 2.0 Flash** for AI classification.

```
┌─────────────────────────────────────────────────────────────────────┐
│                     CURRENT ARCHITECTURE (n8n)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  WF1: Smart Cleanup (every 4h)                                     │
│  ├─ Fetch today's Gmail emails                                     │
│  ├─ Pass 1: Rule-based classification (instant, free)              │
│  ├─ Pass 2: AI classification via Gemini (unknowns only)           │
│  ├─ Log all actions to Supabase                                    │
│  └─ Trash marked emails (add TRASH label)                          │
│                                                                     │
│  WF2: Daily Summary (11:55 PM EST)                                 │
│  ├─ Query today's trashed/kept actions from Supabase               │
│  ├─ Build HTML email with Rescue/Flag buttons                      │
│  └─ Send summary email to user                                     │
│                                                                     │
│  WF3: Rescue Webhook (user-triggered)                              │
│  ├─ Create KEEP rule for sender                                    │
│  ├─ Delete conflicting AI TRASH rules                              │
│  └─ Remove TRASH label (restore to inbox)                          │
│                                                                     │
│  WF4: Flag as Trash Webhook (user-triggered)                       │
│  ├─ Create TRASH rule for sender                                   │
│  ├─ Delete conflicting KEEP rules                                  │
│  └─ Add TRASH label to email                                       │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│  Database: Supabase (falgykkspbtrwdcchayi.supabase.co)             │
│  Tables: email_accounts, email_rules, email_actions, email_feedback │
│  AI: Gemini 2.0 Flash (free tier, 1000 req/day)                   │
│  Gmail: OAuth via n8n credential                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Workflow Interaction & Learning Loop

```
WF1 classifies email → logs to email_actions
                          ↓
WF2 reads email_actions → sends summary with Rescue/Flag buttons
                          ↓
User clicks Rescue    → WF3 → creates KEEP rule   ─┐
User clicks Flag      → WF4 → creates TRASH rule  ─┤
                                                     ↓
WF1 (next run) → rule matches FIRST (before AI) → learned behavior
```

### What Works Well
- Two-pass classification (rules first, then AI) is cost-effective
- User feedback creates rules at confidence=1.0 that override AI
- KEEP rules always win over TRASH rules (safe by design)
- Dedup prevents reprocessing same email in same day
- Conservative defaults — "keep" on AI error or uncertainty

### What Doesn't Scale
- Hardcoded credentials in n8n Code nodes (per-account)
- Separate n8n instance per email account
- Manual workflow duplication for each new user
- No self-service onboarding
- No billing or usage tracking
- Unsigned webhook URLs (security gap)

---

## 2. SaaS Vision

### User Journey

```
1. User visits cleaninbox.app
2. Signs up (email/password or Google login)
3. Clicks "Connect Gmail" → Google OAuth consent
4. Configures preferences (aggressiveness, scan frequency)
5. System starts scanning automatically every 4-6 hours
6. User receives daily summary email with Rescue/Flag buttons
7. System learns from feedback → gets smarter over time
8. User manages rules from web dashboard
```

### Core Features

| Feature | Description | Priority |
|---------|-------------|----------|
| Gmail OAuth | Connect Gmail with one click | P0 |
| Auto-scan | Scheduled email scanning (4x/day) | P0 |
| AI Classification | Gemini-powered trash/keep decisions | P0 |
| Rule Engine | User-defined + AI-learned rules | P0 |
| Daily Summary | HTML email with Rescue/Flag actions | P0 |
| Web Dashboard | View scan history, manage rules | P1 |
| Custom Frequency | User picks scan frequency | P1 |
| Multi-account | Connect multiple Gmail accounts | P2 |
| Outlook Support | Microsoft 365 email support | P3 |

---

## 3. Recommended Tech Stack

### Primary Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Compute** | Cloudflare Workers | Zero cold starts, built-in cron, queues, $5/mo |
| **Database** | Supabase (PostgreSQL) | Auth + DB + RLS in one, free tier covers 50K users |
| **AI** | Gemini 2.0 Flash | Free tier (1000 req/day), cheapest paid ($0.10/M tokens) |
| **Frontend** | Next.js on Vercel | Largest ecosystem, Supabase integration, free hosting |
| **Email API** | Gmail API (direct) | Free, full control, sufficient quotas |
| **Scheduling** | Cloudflare Cron Triggers | Free, built into Workers, fan-out via Queues |
| **Token Storage** | Supabase (encrypted) + Cloudflare KV (cache) | Encrypted at rest, fast reads |
| **Payments** | Stripe | Industry standard for SaaS billing |
| **Monitoring** | Sentry + Cloudflare Analytics | Error tracking + performance |

### Why Serverless Over n8n?

| Factor | n8n (Current) | Serverless (Proposed) |
|--------|--------------|----------------------|
| Per-user setup | Manual workflow duplication | Automatic via code |
| Cost at 100 users | ~$100/mo (n8n Cloud plan) | ~$5/mo (Workers) |
| Cost at 1000 users | ~$500+/mo | ~$15/mo |
| Credential management | Hardcoded in Code nodes | Encrypted in database |
| Self-service onboarding | Not possible | OAuth flow, instant |
| Custom domain | Limited | Full control |
| Scaling | n8n execution limits | Auto-scales |

### Why Cloudflare Workers Over AWS Lambda?

| Factor | AWS Lambda | Cloudflare Workers |
|--------|-----------|-------------------|
| Cold starts | 100-500ms | ~0ms (V8 isolates) |
| Free tier | 1M requests/mo | 100K requests/day (3M/mo) |
| Scheduling | EventBridge ($) | Cron Triggers (free) |
| Queues | SQS ($) | Workers Queues (included) |
| KV Store | DynamoDB ($) | Workers KV (included) |
| Base cost | Pay-per-use | $5/mo flat |
| Complexity | IAM, VPC, CloudFormation | Wrangler CLI, simple |

---

## 4. System Architecture

### High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                         CLEANINBOX SaaS ARCHITECTURE                     │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐  │
│  │   Browser    │────▶│  Next.js (Vercel) │────▶│  Supabase Auth     │  │
│  │  Dashboard   │◀────│  - Landing page   │◀────│  - Email/Password  │  │
│  │             │     │  - Dashboard      │     │  - Google OAuth    │  │
│  │             │     │  - Rule editor    │     │  - JWT sessions    │  │
│  └─────────────┘     │  - Scan history   │     └─────────────────────┘  │
│                       └──────────────────┘                               │
│                              │                                           │
│                       ┌──────▼──────┐                                    │
│                       │  Supabase   │                                    │
│                       │  PostgreSQL │                                    │
│                       │             │                                    │
│                       │ - profiles  │                                    │
│                       │ - oauth_tokens (encrypted)                      │
│                       │ - cleanup_rules                                 │
│                       │ - scan_logs                                     │
│                       │ - email_actions                                 │
│                       │ - email_feedback                                │
│                       │             │                                    │
│                       │  (RLS enforced per user)                        │
│                       └──────┬──────┘                                    │
│                              │                                           │
│  ┌───────────────────────────┼───────────────────────────────────────┐  │
│  │             CLOUDFLARE WORKERS (Serverless Compute)               │  │
│  │                           │                                       │  │
│  │  ┌───────────────────┐   │   ┌───────────────────────────┐      │  │
│  │  │  Cron Trigger      │   │   │  Webhook Worker           │      │  │
│  │  │  (Every 6 hours)   │   │   │  - POST /rescue           │      │  │
│  │  │                    │   │   │  - POST /flag-trash        │      │  │
│  │  │  1. Fetch active   │   │   │  - POST /disconnect       │      │  │
│  │  │     users from DB  │   │   │                           │      │  │
│  │  │  2. Publish 1 msg  │   │   │  Validates HMAC signature │      │  │
│  │  │     per user to    │   │   │  Updates rules + Gmail    │      │  │
│  │  │     Queue          │   │   └───────────────────────────┘      │  │
│  │  └────────┬───────────┘   │                                       │  │
│  │           │               │                                       │  │
│  │  ┌────────▼───────────┐   │                                       │  │
│  │  │  Cloudflare Queue  │   │                                       │  │
│  │  │  (scan-jobs)       │   │                                       │  │
│  │  │                    │   │                                       │  │
│  │  │  1 message = 1     │   │                                       │  │
│  │  │  user's scan job   │   │                                       │  │
│  │  └────────┬───────────┘   │                                       │  │
│  │           │               │                                       │  │
│  │  ┌────────▼───────────┐   │                                       │  │
│  │  │  Scanner Worker    │   │                                       │  │
│  │  │                    │   │                                       │  │
│  │  │  Per user:         │   │                                       │  │
│  │  │  1. Decrypt token  │──▶│──▶ Gmail API: messages.list           │  │
│  │  │  2. Refresh OAuth  │──▶│──▶ Gmail API: messages.get            │  │
│  │  │  3. Fetch emails   │   │                                       │  │
│  │  │  4. Apply rules    │   │   (Pass 1: instant, free)             │  │
│  │  │  5. AI classify    │──▶│──▶ Gemini API: classify unknowns      │  │
│  │  │  6. Trash/keep     │──▶│──▶ Gmail API: messages.modify         │  │
│  │  │  7. Log results    │──▶│──▶ Supabase: insert email_actions     │  │
│  │  └────────────────────┘   │                                       │  │
│  │                           │                                       │  │
│  │  ┌────────────────────┐   │                                       │  │
│  │  │  Summary Worker    │   │                                       │  │
│  │  │  (Cron: 11 PM)     │   │                                       │  │
│  │  │                    │   │                                       │  │
│  │  │  1. Query actions  │──▶│──▶ Supabase: today's actions          │  │
│  │  │  2. Build HTML     │   │                                       │  │
│  │  │  3. Send email     │──▶│──▶ Gmail API: send (or Resend/SES)    │  │
│  │  └────────────────────┘   │                                       │  │
│  └───────────────────────────┴───────────────────────────────────────┘  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Request Flow: User Connects Gmail

```
Browser                 Next.js (Vercel)         Google OAuth         Supabase
  │                          │                       │                   │
  │  1. Click "Connect"      │                       │                   │
  │─────────────────────────▶│                       │                   │
  │                          │  2. Redirect to       │                   │
  │                          │     Google OAuth      │                   │
  │◀─────────────────────────│──────────────────────▶│                   │
  │  3. User consents        │                       │                   │
  │─────────────────────────▶│                       │                   │
  │                          │  4. Exchange code     │                   │
  │                          │     for tokens        │                   │
  │                          │──────────────────────▶│                   │
  │                          │◀──────────────────────│                   │
  │                          │                       │                   │
  │                          │  5. Encrypt refresh   │                   │
  │                          │     token, store      │                   │
  │                          │─────────────────────────────────────────▶│
  │                          │                       │                   │
  │  6. "Gmail Connected!"   │                       │                   │
  │◀─────────────────────────│                       │                   │
```

### Request Flow: Scheduled Email Scan

```
Cron Trigger          Queue              Scanner Worker        Gmail API        Gemini        Supabase
  │                    │                      │                   │               │              │
  │ 1. Fetch users     │                      │                   │               │              │
  │────────────────────────────────────────────────────────────────────────────────────────────▶│
  │◀───────────────────────────────────────────────────────────────────────────────────────────│
  │                    │                      │                   │               │              │
  │ 2. Publish 1 msg   │                      │                   │               │              │
  │    per user        │                      │                   │               │              │
  │───────────────────▶│                      │                   │               │              │
  │                    │  3. Consume msg       │                   │               │              │
  │                    │─────────────────────▶│                   │               │              │
  │                    │                      │  4. Decrypt token  │               │              │
  │                    │                      │──────────────────────────────────────────────▶│
  │                    │                      │◀─────────────────────────────────────────────│
  │                    │                      │                   │               │              │
  │                    │                      │  5. Fetch emails   │               │              │
  │                    │                      │──────────────────▶│               │              │
  │                    │                      │◀──────────────────│               │              │
  │                    │                      │                   │               │              │
  │                    │                      │  6. Apply rules (Pass 1)          │              │
  │                    │                      │  (local, instant)  │               │              │
  │                    │                      │                   │               │              │
  │                    │                      │  7. Classify       │               │              │
  │                    │                      │     unknowns       │               │              │
  │                    │                      │──────────────────────────────────▶│              │
  │                    │                      │◀─────────────────────────────────│              │
  │                    │                      │                   │               │              │
  │                    │                      │  8. Trash emails   │               │              │
  │                    │                      │──────────────────▶│               │              │
  │                    │                      │                   │               │              │
  │                    │                      │  9. Log actions    │               │              │
  │                    │                      │──────────────────────────────────────────────▶│
```

---

## 5. Database Schema

### Multi-Tenant Schema (Supabase PostgreSQL)

```sql
-- ============================================
-- CleanInbox SaaS — Multi-Tenant Schema
-- ============================================

-- 1. PROFILES — Extends Supabase Auth users
CREATE TABLE profiles (
  id UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
  email TEXT NOT NULL,
  display_name TEXT,
  plan TEXT NOT NULL DEFAULT 'free'
    CHECK (plan IN ('free', 'pro', 'team')),
  scan_frequency_hrs INT NOT NULL DEFAULT 6,
  aggressiveness TEXT NOT NULL DEFAULT 'conservative'
    CHECK (aggressiveness IN ('conservative', 'moderate', 'aggressive')),
  timezone TEXT NOT NULL DEFAULT 'America/New_York',
  summary_enabled BOOLEAN NOT NULL DEFAULT true,
  summary_time TEXT NOT NULL DEFAULT '21:00',  -- 9 PM in user's timezone
  onboarded_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 2. CONNECTED ACCOUNTS — OAuth tokens for Gmail (supports multi-account)
CREATE TABLE connected_accounts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  email_address TEXT NOT NULL,
  provider TEXT NOT NULL DEFAULT 'gmail'
    CHECK (provider IN ('gmail', 'outlook')),
  encrypted_refresh_token TEXT NOT NULL,
  access_token_expires_at TIMESTAMPTZ,
  is_active BOOLEAN NOT NULL DEFAULT true,
  last_scanned_at TIMESTAMPTZ,
  scan_error TEXT,            -- Last error message (NULL = healthy)
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(user_id, email_address)
);

-- 3. CLEANUP RULES — Per-account rules (user-created + AI-learned)
CREATE TABLE cleanup_rules (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES connected_accounts(id) ON DELETE CASCADE,
  pattern TEXT NOT NULL,
  pattern_type TEXT NOT NULL
    CHECK (pattern_type IN ('sender', 'domain', 'keyword', 'subject')),
  action TEXT NOT NULL
    CHECK (action IN ('trash', 'keep')),
  confidence FLOAT NOT NULL DEFAULT 0.5
    CHECK (confidence >= 0 AND confidence <= 1),
  source TEXT NOT NULL DEFAULT 'ai'
    CHECK (source IN ('ai', 'user_feedback', 'manual')),
  is_active BOOLEAN NOT NULL DEFAULT true,
  hit_count INT NOT NULL DEFAULT 0,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 4. EMAIL ACTIONS — Audit log of every classification
CREATE TABLE email_actions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES connected_accounts(id) ON DELETE CASCADE,
  message_id TEXT NOT NULL,
  thread_id TEXT,
  subject TEXT,
  sender TEXT,
  snippet TEXT,
  action_taken TEXT NOT NULL
    CHECK (action_taken IN ('trashed', 'kept', 'skipped')),
  classified_by TEXT NOT NULL
    CHECK (classified_by IN ('rule', 'ai', 'manual')),
  rule_id UUID REFERENCES cleanup_rules(id) ON DELETE SET NULL,
  ai_reasoning TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(account_id, message_id)
);

-- 5. EMAIL FEEDBACK — User corrections (rescue/flag)
CREATE TABLE email_feedback (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES connected_accounts(id) ON DELETE CASCADE,
  action_id UUID NOT NULL REFERENCES email_actions(id) ON DELETE CASCADE,
  message_id TEXT NOT NULL,
  feedback_type TEXT NOT NULL
    CHECK (feedback_type IN ('rescue', 'approve', 'flag_trash')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 6. SCAN LOGS — Per-scan execution tracking
CREATE TABLE scan_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  account_id UUID NOT NULL REFERENCES connected_accounts(id) ON DELETE CASCADE,
  started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  completed_at TIMESTAMPTZ,
  emails_scanned INT DEFAULT 0,
  emails_trashed INT DEFAULT 0,
  emails_kept INT DEFAULT 0,
  ai_calls_made INT DEFAULT 0,
  status TEXT NOT NULL DEFAULT 'running'
    CHECK (status IN ('running', 'completed', 'failed')),
  error_message TEXT
);

-- ============================================
-- INDEXES
-- ============================================
CREATE INDEX idx_connected_accounts_user ON connected_accounts(user_id);
CREATE INDEX idx_cleanup_rules_account ON cleanup_rules(account_id, is_active);
CREATE INDEX idx_email_actions_account_date ON email_actions(account_id, created_at DESC);
CREATE INDEX idx_email_actions_message ON email_actions(message_id);
CREATE INDEX idx_email_feedback_action ON email_feedback(action_id);
CREATE INDEX idx_scan_logs_account ON scan_logs(account_id, started_at DESC);

-- ============================================
-- ROW LEVEL SECURITY
-- ============================================
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
ALTER TABLE connected_accounts ENABLE ROW LEVEL SECURITY;
ALTER TABLE cleanup_rules ENABLE ROW LEVEL SECURITY;
ALTER TABLE email_actions ENABLE ROW LEVEL SECURITY;
ALTER TABLE email_feedback ENABLE ROW LEVEL SECURITY;
ALTER TABLE scan_logs ENABLE ROW LEVEL SECURITY;

-- Users can only access their own data
CREATE POLICY "Own profile" ON profiles
  FOR ALL USING (auth.uid() = id);

CREATE POLICY "Own connected accounts" ON connected_accounts
  FOR ALL USING (auth.uid() = user_id);

CREATE POLICY "Own rules" ON cleanup_rules
  FOR ALL USING (
    account_id IN (SELECT id FROM connected_accounts WHERE user_id = auth.uid())
  );

CREATE POLICY "Own actions" ON email_actions
  FOR ALL USING (
    account_id IN (SELECT id FROM connected_accounts WHERE user_id = auth.uid())
  );

CREATE POLICY "Own feedback" ON email_feedback
  FOR ALL USING (
    account_id IN (SELECT id FROM connected_accounts WHERE user_id = auth.uid())
  );

CREATE POLICY "Own scan logs" ON scan_logs
  FOR ALL USING (
    account_id IN (SELECT id FROM connected_accounts WHERE user_id = auth.uid())
  );

-- Service role bypass (for Cloudflare Workers)
-- Service role automatically bypasses RLS in Supabase
```

### Schema Comparison: Current vs SaaS

| Current (n8n) | SaaS (Proposed) | Change |
|---------------|-----------------|--------|
| `email_accounts` | `profiles` + `connected_accounts` | Split user identity from email connection |
| Hardcoded credentials | `connected_accounts.encrypted_refresh_token` | Encrypted OAuth tokens per user |
| No auth | Supabase Auth + RLS | Full multi-tenant isolation |
| No scan tracking | `scan_logs` | Per-scan execution metrics |
| `email_rules` | `cleanup_rules` | Same concept, linked to `connected_accounts` |
| `email_actions` | `email_actions` | Nearly identical |
| `email_feedback` | `email_feedback` | Simplified (no `apply_rule_to`) |

---

## 6. Gmail OAuth Strategy

### The Critical Challenge: Restricted Scopes

Gmail's `gmail.modify` scope (needed to read + trash emails) is classified as **restricted** by Google. This has major implications:

| Phase | Users | OAuth Status | Cost | Timeline |
|-------|-------|-------------|------|----------|
| **Beta** | < 100 | Unverified app | $0 | Immediate |
| **Launch** | 100+ | Verified (CASA assessment) | $5,000-$15,000 | 4-12 weeks |
| **Annual** | Any | Re-verification | $500-$5,000/yr | Ongoing |

### What "Unverified" Means for Beta Users

Users will see this warning when connecting Gmail:

```
┌─────────────────────────────────────────────┐
│  ⚠️  Google hasn't verified this app        │
│                                             │
│  The app is requesting access to sensitive  │
│  info in your Google Account.               │
│                                             │
│  [Back to safety]                           │
│                                             │
│  Advanced ▼                                 │
│  ─────────                                  │
│  Continue to cleaninbox.app (unsafe)        │
└─────────────────────────────────────────────┘
```

**This is acceptable for beta** — early adopters understand this. Plan to invest in CASA verification once you have 50+ paying users to fund it.

### OAuth Flow Implementation

```
┌─────────────────────────────────────────────────────────────────┐
│                    GMAIL OAUTH FLOW                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. USER CLICKS "CONNECT GMAIL"                                │
│     → Redirect to Google OAuth consent screen                   │
│     → Scopes requested: gmail.modify, gmail.send               │
│     → Include: access_type=offline, prompt=consent              │
│       (ensures refresh_token is always returned)                │
│                                                                 │
│  2. USER APPROVES                                              │
│     → Google redirects to /api/auth/gmail/callback              │
│     → Callback receives authorization code                     │
│                                                                 │
│  3. EXCHANGE CODE FOR TOKENS                                   │
│     → POST to https://oauth2.googleapis.com/token              │
│     → Receive: access_token (1hr) + refresh_token (long-lived) │
│                                                                 │
│  4. STORE SECURELY                                             │
│     → Encrypt refresh_token with AES-256-GCM                   │
│     → Encryption key: env var (ENCRYPTION_KEY)                  │
│     → Store encrypted token in connected_accounts table         │
│     → Cache access_token in Cloudflare KV (TTL: 50 min)        │
│                                                                 │
│  5. ON EACH SCAN                                               │
│     → Check KV for cached access_token                          │
│     → If expired: decrypt refresh_token from DB                 │
│     → POST to Google to refresh → new access_token              │
│     → Cache new access_token in KV (TTL: 50 min)               │
│     → Use access_token for Gmail API calls                      │
│                                                                 │
│  6. TOKEN REVOCATION HANDLING                                  │
│     → If Gmail API returns 401 (token revoked)                  │
│     → Mark connected_account as inactive                        │
│     → Set scan_error = "Gmail access revoked"                   │
│     → Send user email: "Please reconnect your Gmail"            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Google Cloud Project Setup

```
Google Cloud Console → APIs & Services:

1. Create OAuth consent screen
   - App name: CleanInbox
   - User support email: support@cleaninbox.app
   - Scopes: gmail.modify, gmail.send, userinfo.email
   - App domain: cleaninbox.app
   - Authorized domains: cleaninbox.app

2. Create OAuth 2.0 Client ID
   - Application type: Web application
   - Authorized redirect URIs:
     - https://cleaninbox.app/api/auth/gmail/callback
     - http://localhost:3000/api/auth/gmail/callback (dev)

3. Enable APIs:
   - Gmail API
   - Generative Language API (Gemini)
```

---

## 7. Serverless Processing Pipeline

### Cloudflare Workers Architecture

```
┌─────────────────────────────────────────────────────────────┐
│               CLOUDFLARE WORKERS SETUP                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Worker 1: ORCHESTRATOR                                     │
│  ─────────────────────                                      │
│  Trigger: Cron (*/6 * * * * = every 6 hours)               │
│  Job:                                                       │
│    1. Query Supabase: SELECT * FROM connected_accounts      │
│       WHERE is_active = true                                │
│       AND (last_scanned_at IS NULL                          │
│            OR last_scanned_at < now() - interval '4 hours') │
│    2. For each account → publish message to Queue           │
│       Message: { account_id, user_id, email_address }       │
│    3. Log: "Published N scan jobs"                          │
│                                                             │
│  Worker 2: SCANNER (Queue Consumer)                         │
│  ────────────────────────────────                           │
│  Trigger: Cloudflare Queue (scan-jobs)                      │
│  Concurrency: max 10 concurrent (respect Gmail API quotas)  │
│  Job per message:                                           │
│    1. Decrypt refresh_token from Supabase                   │
│    2. Refresh OAuth access_token (cache in KV)              │
│    3. Gmail API: list today's emails                        │
│    4. Filter out already-processed (check email_actions)    │
│    5. Pass 1: Apply cleanup_rules                           │
│    6. Pass 2: Gemini API classify unknowns (batch of 20)   │
│    7. Gmail API: add TRASH label to trashed emails          │
│    8. Supabase: insert email_actions (batch upsert)         │
│    9. Supabase: update last_scanned_at                      │
│   10. Supabase: insert scan_log                             │
│                                                             │
│  Worker 3: SUMMARY (Cron)                                   │
│  ────────────────────────                                    │
│  Trigger: Cron (0 4 * * * = 4 AM UTC ≈ 11 PM EST)         │
│  Job:                                                       │
│    1. Query users WHERE summary_enabled = true              │
│    2. For each user: query today's email_actions            │
│    3. Build HTML summary email                              │
│    4. Sign Rescue/Flag URLs with HMAC-SHA256                │
│    5. Send via Gmail API (user's own account) or Resend     │
│                                                             │
│  Worker 4: WEBHOOKS                                         │
│  ──────────────────                                          │
│  Trigger: HTTP (GET /api/rescue, GET /api/flag-trash)       │
│  Job:                                                       │
│    1. Verify HMAC signature on URL                          │
│    2. Look up action in email_actions                       │
│    3. Create/delete rules in cleanup_rules                  │
│    4. Gmail API: modify labels (untrash or trash)           │
│    5. Log feedback in email_feedback                        │
│    6. Return styled HTML confirmation page                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Project Structure (Cloudflare Workers)

```
cleaninbox-workers/
├── wrangler.toml              # Cloudflare config (crons, queues, KV)
├── src/
│   ├── index.ts               # Worker entry point (routes + cron + queue)
│   ├── orchestrator.ts        # Cron: fan-out scan jobs
│   ├── scanner.ts             # Queue consumer: process one user's scan
│   ├── summary.ts             # Cron: send daily summaries
│   ├── webhooks.ts            # HTTP: rescue & flag-trash endpoints
│   ├── lib/
│   │   ├── gmail.ts           # Gmail API wrapper (list, get, modify, send)
│   │   ├── gemini.ts          # Gemini API wrapper (classify batch)
│   │   ├── supabase.ts        # Supabase client (queries)
│   │   ├── crypto.ts          # AES-256-GCM encrypt/decrypt tokens
│   │   ├── hmac.ts            # HMAC-SHA256 URL signing/verification
│   │   └── classifier.ts     # Two-pass classification engine
│   └── types.ts               # TypeScript interfaces
├── package.json
└── tsconfig.json
```

### wrangler.toml Configuration

```toml
name = "cleaninbox"
main = "src/index.ts"
compatibility_date = "2026-03-01"

[triggers]
crons = [
  "0 */6 * * *",   # Orchestrator: every 6 hours
  "0 4 * * *"      # Summary: 4 AM UTC (11 PM EST)
]

[[queues.producers]]
queue = "scan-jobs"
binding = "SCAN_QUEUE"

[[queues.consumers]]
queue = "scan-jobs"
max_batch_size = 10
max_retries = 3
dead_letter_queue = "scan-jobs-dlq"

[[kv_namespaces]]
binding = "TOKEN_CACHE"
id = "xxxxx"

[vars]
SUPABASE_URL = "https://your-project.supabase.co"
GEMINI_MODEL = "gemini-2.0-flash"

# Secrets (set via `wrangler secret put`):
# SUPABASE_SERVICE_KEY
# GEMINI_API_KEY
# ENCRYPTION_KEY
# HMAC_SECRET
# GOOGLE_CLIENT_ID
# GOOGLE_CLIENT_SECRET
```

---

## 8. Frontend Dashboard

### Next.js App Structure

```
cleaninbox-web/
├── app/
│   ├── page.tsx                    # Landing page (marketing)
│   ├── login/page.tsx              # Login (Supabase Auth)
│   ├── signup/page.tsx             # Signup
│   ├── dashboard/
│   │   ├── page.tsx                # Overview (stats, recent scans)
│   │   ├── accounts/page.tsx       # Connected Gmail accounts
│   │   ├── rules/page.tsx          # Manage cleanup rules
│   │   ├── history/page.tsx        # Scan history + email actions
│   │   └── settings/page.tsx       # Preferences (frequency, timezone)
│   ├── api/
│   │   ├── auth/
│   │   │   └── gmail/
│   │   │       └── callback/route.ts   # OAuth callback handler
│   │   └── scan/
│   │       └── trigger/route.ts        # Manual scan trigger
│   └── layout.tsx
├── components/
│   ├── RuleEditor.tsx              # Create/edit cleanup rules
│   ├── ScanHistory.tsx             # Table of past scans
│   ├── AccountCard.tsx             # Gmail account status card
│   └── StatsCards.tsx              # Dashboard stat cards
├── lib/
│   ├── supabase-client.ts          # Supabase browser client
│   └── supabase-server.ts          # Supabase server client
└── package.json
```

### Dashboard Wireframe

```
┌──────────────────────────────────────────────────────────────────┐
│  CleanInbox                           [Settings] [Logout]        │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │
│  │   247     │  │   189    │  │    58    │  │    12    │        │
│  │ Processed │  │ Trashed  │  │  Kept    │  │ Rules    │        │
│  │ this week │  │          │  │          │  │ learned  │        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │
│                                                                  │
│  Connected Accounts                        [+ Connect Gmail]     │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ ✅ abhishek15@gmail.com    Last scan: 2 hours ago        │   │
│  │    Status: Healthy | Rules: 34 | Scanned: 1,247 emails   │   │
│  │    [Pause] [Disconnect]                                   │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Recent Scan Activity                                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ Time        │ Scanned │ Trashed │ Kept │ AI Calls │ Status│   │
│  │─────────────│─────────│─────────│──────│──────────│───────│   │
│  │ Today 6 PM  │ 23      │ 18      │ 5    │ 2        │ ✅    │   │
│  │ Today 12 PM │ 31      │ 25      │ 6    │ 3        │ ✅    │   │
│  │ Today 6 AM  │ 15      │ 11      │ 4    │ 1        │ ✅    │   │
│  │ Yesterday   │ 67      │ 52      │ 15   │ 4        │ ✅    │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  Top Cleanup Rules                         [Manage All Rules]    │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │ 🗑️ TRASH  noreply@linkedin.com     (AI, 94 hits)        │   │
│  │ 🗑️ TRASH  marketing@hubspot.com    (User, 47 hits)      │   │
│  │ ✅ KEEP   boss@company.com          (User, 23 hits)      │   │
│  │ 🗑️ TRASH  *@promotions.amazon.com  (AI, 19 hits)        │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 9. Cost Analysis

### Monthly Cost by Scale

| Component | 100 Users | 500 Users | 1,000 Users | 5,000 Users |
|-----------|-----------|-----------|-------------|-------------|
| Cloudflare Workers (paid) | $5 | $5 | $10 | $25 |
| Supabase | $0 (free) | $25 (Pro) | $25 (Pro) | $100 (Team) |
| Gemini 2.0 Flash | $0 (free) | $17 | $34 | $168 |
| Gmail API | $0 | $0 | $0 | $0 |
| Vercel (Next.js hosting) | $0 (free) | $0 (free) | $20 (Pro) | $20 (Pro) |
| Domain + DNS | $1 | $1 | $1 | $1 |
| CASA Assessment (amortized) | $417 | $417 | $417 | $417 |
| **Total Infrastructure** | **$423** | **$465** | **$507** | **$731** |

### Revenue Model

| Plan | Price | Features |
|------|-------|----------|
| **Free** | $0/mo | 1 Gmail account, 2 scans/day, 50 rules max |
| **Pro** | $5/mo | 3 Gmail accounts, 4 scans/day, unlimited rules, priority AI |
| **Team** | $15/mo | 10 accounts, 6 scans/day, API access, webhook integrations |

### Break-Even Analysis

```
Fixed costs: ~$423/month (infrastructure + CASA amortized)
Variable cost per user: ~$0.05/month

Break-even at $5/mo plan: 423 / 5 = 85 paying users
Break-even at $10/mo plan: 423 / 10 = 43 paying users

At 500 paying Pro users: $2,500/mo revenue - $465/mo cost = $2,035/mo profit
At 1000 paying Pro users: $5,000/mo revenue - $507/mo cost = $4,493/mo profit
```

---

## 10. Implementation Roadmap

### Phase 1: MVP (Weeks 1-3)

**Goal**: Working product for 10 beta users

```
Week 1: Foundation
├── Set up Supabase project (new, dedicated)
├── Create database schema (tables + RLS)
├── Next.js app with Supabase Auth (login/signup)
├── Gmail OAuth flow (connect/disconnect)
└── Basic dashboard (connected accounts list)

Week 2: Core Engine
├── Cloudflare Worker: Scanner (rule-based only, no AI yet)
├── Gmail API integration (list, get, modify)
├── Token encryption/refresh logic
├── Manual scan trigger from dashboard
└── Email actions logging

Week 3: AI + Summary
├── Gemini integration for AI classification
├── Two-pass classifier (rules first, then AI)
├── Daily summary email with Rescue/Flag buttons
├── HMAC-signed webhook URLs
├── Webhook handlers (rescue + flag)
└── Beta launch to 10 users
```

### Phase 2: Polish (Weeks 4-5)

```
├── Rule management UI (create, edit, delete, toggle)
├── Scan history page
├── Settings page (frequency, timezone, aggressiveness)
├── Error handling + retry logic
├── Token revocation handling (re-auth flow)
└── Email notifications for scan failures
```

### Phase 3: Production (Weeks 6-10)

```
├── Google CASA security assessment application
├── Privacy policy + Terms of Service
├── Stripe billing integration
├── Rate limiting + abuse prevention
├── Sentry error monitoring
├── Landing page + SEO
├── Documentation / help center
└── Launch to public
```

### Phase 4: Growth (Months 3+)

```
├── Outlook/Microsoft 365 support
├── AI-suggested rules ("We noticed 47 newsletters...")
├── Bulk rule templates ("One-click unsubscribe mode")
├── Team/shared rules
├── API access for power users
└── Mobile-friendly dashboard
```

---

## 11. Key Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|-----------|
| CASA assessment cost ($5K-$15K) | Blocks scaling past 100 users | High | Budget for it; validate product-market fit first with beta |
| Google revokes user tokens | Users disconnected, no scanning | Medium | Monitor 401s, auto-email users to reconnect, graceful degradation |
| Gemini free tier rate limited | Delayed classification | Low | Aggressive batching (20/request); fallback to rule-only mode |
| User emails contain sensitive data | Privacy/legal liability | High | Never store email body; only store sender/subject/snippet; encrypt tokens |
| Gmail API quota exceeded at scale | Scans throttled | Low | Spread scans across day; exponential backoff; per-user rate limiting |
| Supabase free tier pauses DB | Complete outage | Medium | Upgrade to Pro ($25/mo) before launch; monitor usage |
| Competitor (Google Workspace, Clean Email) | Market pressure | Medium | Focus on simplicity + learning loop; free tier as moat |

### Security Best Practices

```
1. NEVER store full email bodies — only sender, subject, snippet
2. Encrypt OAuth refresh tokens with AES-256-GCM at rest
3. Sign all webhook URLs with HMAC-SHA256 (prevent forgery)
4. Use Supabase RLS — users can ONLY access their own data
5. Rotate encryption keys annually
6. Store secrets in Cloudflare Worker secrets (not code)
7. Implement CSRF protection on all form submissions
8. Rate-limit webhook endpoints (prevent abuse)
9. Log all token refresh failures for security monitoring
10. HTTPS everywhere (enforced by Cloudflare + Vercel)
```

---

## Quick Comparison: Current vs SaaS

| Aspect | Current (n8n) | SaaS (Proposed) |
|--------|--------------|-----------------|
| Users | 1-2 (manual setup) | Unlimited (self-service) |
| Onboarding | Manual workflow import | OAuth flow, 30 seconds |
| Cost per user | ~$50/mo (n8n Cloud) | ~$0.05/mo (serverless) |
| AI model | Gemini 2.0 Flash | Gemini 2.0 Flash (same) |
| Classification | Two-pass (rules + AI) | Two-pass (same logic) |
| Learning loop | Rescue/Flag buttons | Rescue/Flag + dashboard rules |
| Database | Shared Supabase | Dedicated Supabase with RLS |
| Security | Unsigned webhooks | HMAC-signed + encrypted tokens |
| Monitoring | n8n execution logs | Sentry + scan_logs table |
| Billing | None | Stripe (free/pro/team) |

---

## Next Steps

1. **Validate demand** — Share the concept with 10-20 potential users
2. **Set up Supabase project** — New dedicated project (not PropLedger)
3. **Scaffold Next.js app** — Landing page + auth + OAuth flow
4. **Build Scanner Worker** — Port existing n8n classification logic
5. **Beta launch** — 10 users, gather feedback, iterate

---

*Document created: 2026-03-05*
*Based on analysis of existing n8n Email Cleanup system (4 workflows, 4 tables)*
*Tech stack: Cloudflare Workers + Supabase + Gemini 2.0 Flash + Next.js*
