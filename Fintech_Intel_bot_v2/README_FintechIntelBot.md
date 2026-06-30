# Fintech Competitor Intelligence Bot v2 (Groq) — n8n

An automated competitive intelligence pipeline that monitors **5 Indian fintech companies** across 4 data sources every 6 hours, synthesises findings using Groq's LLM, and delivers threat-rated reports to Gmail, Discord, Telegram, and Google Sheets.

---

## Table of Contents

- [Overview](#overview)
- [Monitored Companies](#monitored-companies)
- [Architecture](#architecture)
- [Pipeline Breakdown](#pipeline-breakdown)
  - [1. News Pipeline](#1-news-pipeline)
  - [2. Product Intelligence Pipeline](#2-product-intelligence-pipeline)
  - [3. Social Sentiment Pipeline](#3-social-sentiment-pipeline)
  - [4. Jobs & Hiring Pipeline](#4-jobs--hiring-pipeline)
  - [5. Master Synthesis](#5-master-synthesis)
  - [6. Output & Alert Distribution](#6-output--alert-distribution)
- [Prerequisites](#prerequisites)
- [Credentials Required](#credentials-required)
- [Setup Instructions](#setup-instructions)
- [Placeholders to Replace](#placeholders-to-replace)
- [Output Format](#output-format)
- [Known Bug Fixes (v2 Changes)](#known-bug-fixes-v2-changes)
- [Troubleshooting](#troubleshooting)

---

## Overview

This workflow runs automatically on a **6-hour schedule** and collects competitive signals from Google News RSS, Reddit, and job listings for Razorpay, PhonePe, Paytm, Zepto, and CRED. Each data source is analysed independently by Groq's `llama-3.3-70b-versatile` model, then a final **Master Groq call** synthesises all four intelligence streams into a single scored, actionable briefing.

A **High Threat filter** decides who gets notified:

- **HIGH threat** → Gmail report + Discord alert + Telegram message + Sheets log
- **Low/Medium** → Sheets log only

---

## Monitored Companies

| Company | Focus |
|---------|-------|
| **Razorpay** | Payments infrastructure, B2B fintech |
| **PhonePe** | UPI payments, super-app expansion |
| **Paytm** | Payments, financial services, recovery |
| **Zepto** | Quick commerce + fintech crossover |
| **CRED** | Credit card rewards, lending, cashback |

---

## Architecture

```
⏰ Every 6 Hours
        │
        ├─────────────────────────────────────────────────────┐
        │                                                     │
 ┌──────▼──────┐  ┌──────────────┐  ┌──────────┐  ┌────────▼──────┐
 │ NEWS        │  │ PRODUCT      │  │ SOCIAL   │  │ JOBS          │
 │ (Google RSS)│  │ (Google RSS) │  │ (Reddit) │  │ (Google RSS)  │
 │ 5 companies │  │ 5 companies  │  │ 5 subs   │  │ 5 companies   │
 └──────┬──────┘  └──────┬───────┘  └────┬─────┘  └────────┬──────┘
        │                │               │                  │
   Merge+Groq       Merge+Groq      Merge+Groq         Merge+Groq
        │                │               │                  │
        └────────────────┴───────────────┴──────────────────┘
                                  │
                          🔀 Master Merge (4 inputs)
                                  │
                        📝 Build Master Prompt
                                  │
                          🤖 Groq Master
                                  │
                          ⚙️ Parse Master
                                  │
                        🔍 High Threat? (IF node)
                         │               │
                       TRUE            FALSE
                    ┌────┴────┐          │
              Gmail + Discord   Only → 📊 Sheets
              + Telegram
              + Sheets
```

---

## Pipeline Breakdown

### 1. News Pipeline

Collects **general news** about each company from Google News RSS.

| Step | Node | What it does |
|------|------|-------------|
| 1 | **📰 [Company] News** × 5 | Reads Google News RSS for each company (India-localised, `hl=en-IN`) |
| 2 | **🔀 Merge News** | Combines all 5 feeds into one stream (5-input merge) |
| 3 | **📝 Build News Prompt** | Takes the top 20 items, strips HTML, builds a structured Groq prompt asking for competitive findings per company |
| 4 | **🤖 Groq News** | Posts to Groq API (`llama-3.3-70b-versatile`, temp 0.3, 1024 tokens) |
| 5 | **⚙️ Parse News** | Parses JSON response; tags with `source: 'news'`; returns safe fallback on parse error |

RSS Queries used:

| Company | Query |
|---------|-------|
| Razorpay | `Razorpay fintech` |
| PhonePe | `PhonePe India fintech` |
| Paytm | `Paytm fintech India` |
| Zepto | `Zepto India startup fintech` |
| CRED | `CRED India fintech startup` |

---

### 2. Product Intelligence Pipeline

Collects **product launches, features, pricing changes, and partnerships** using targeted RSS queries.

| Step | Node | What it does |
|------|------|-------------|
| 1 | **🔍 [Company] Products** × 5 | RSS queries filtered with `"new feature" OR "product launch" OR "pricing" OR "announced"` |
| 2 | **🔀 Merge Websites** | Combines all 5 feeds (5-input merge) |
| 3 | **📝 Build Website Prompt** | Takes top 25 items; asks Groq to identify pricing changes, UX shifts, integrations |
| 4 | **🤖 Groq Website** | Groq API call (same model and settings) |
| 5 | **⚙️ Parse Website** | Parses JSON; tags with `source: 'website'`; graceful fallback on error |

> **Note:** This pipeline previously scraped competitor homepages via HTTP GET, which was unreliable. v2 replaced this with RSS product-keyword queries.

---

### 3. Social Sentiment Pipeline

Scrapes **Reddit discussions** to surface user sentiment, complaints, and emerging issues.

| Step | Node | What it does |
|------|------|-------------|
| 1 | **💬 [Company] Reddit** × 5 | Calls Reddit's JSON API (`/search.json`) with `sort=new&limit=10` per company |
| 2 | **🔀 Merge Social** | Combines all 5 Reddit results (5-input merge) |
| 3 | **📝 Build Social Prompt** | Extracts post titles and snippets from `data.children`; prompts Groq for sentiment analysis |
| 4 | **🤖 Groq Social** | Groq API call |
| 5 | **⚙️ Parse Social** | Parses JSON; tags with `source: 'social'` |

Reddit queries used:

| Company | Query |
|---------|-------|
| Razorpay | `razorpay fintech` |
| PhonePe | `phonepe upi india` |
| Paytm | `paytm india` |
| Zepto | `zepto india` |
| CRED | `CRED app india fintech` |

> All Reddit HTTP nodes send a custom `User-Agent: n8n-fintech-intel-bot/2.0 (open-source automation)` header to avoid being blocked by Reddit's API.

---

### 4. Jobs & Hiring Pipeline

Monitors **hiring and restructuring signals** to detect strategic direction changes.

| Step | Node | What it does |
|------|------|-------------|
| 1 | **💼 [Company] Jobs** × 5 | RSS queries filtered with `hiring OR layoffs OR "head of" OR "VP of" OR CTO OR engineer` |
| 2 | **🔀 Merge Jobs** | Combines all 5 feeds (5-input merge) |
| 3 | **📝 Build Jobs Prompt** | Extracts top 20 job titles; asks Groq to infer strategic priorities from hiring patterns |
| 4 | **🤖 Groq Jobs** | Groq API call |
| 5 | **⚙️ Parse Jobs** | Parses JSON; tags with `source: 'jobs'` |

---

### 5. Master Synthesis

Combines all four intelligence streams into one final briefing.

| Step | Node | What it does |
|------|------|-------------|
| 1 | **🔀 Master Merge** | Waits for all 4 pipeline outputs (news, website, social, jobs) — 4-input merge |
| 2 | **📝 Build Master Prompt** | Aggregates all findings; builds a comprehensive cross-source prompt for final synthesis |
| 3 | **🤖 Groq Master** | Final Groq call — synthesises all intelligence into one structured report |
| 4 | **⚙️ Parse Master** | Parses the master report; formats outputs for Telegram (Markdown), Discord, Gmail (HTML), and Google Sheets; sets `is_high_threat` boolean |

---

### 6. Output & Alert Distribution

| Node | Condition | What it sends |
|------|-----------|---------------|
| **🔍 High Threat?** | Checks `is_high_threat === true` | Routes to full alerts or Sheets-only |
| **📧 Gmail Report** | HIGH threat only | HTML email — subject includes `threat_level` and today's Indian date |
| **📡 Discord Webhook** | HIGH threat only | Sends formatted intelligence summary to Discord channel |
| **📤 Telegram Alert** | HIGH threat only | Sends Markdown-formatted message to specified chat ID |
| **📊 Log to Sheets** | Always (both paths) | Logs every run to Google Sheets for historical tracking |

---

## Prerequisites

- **n8n** instance (self-hosted or cloud)
- **Groq** account — [console.groq.com](https://console.groq.com) — free tier available
- **Gmail** OAuth2 account (for sending reports)
- **Telegram Bot** — create via [@BotFather](https://t.me/BotFather); get your Chat ID
- **Discord Webhook URL** — created in your Discord server's channel settings
- **Google Sheets** — a sheet to log intelligence data into

---

## Credentials Required

| Credential | Node(s) | Setup |
|------------|---------|-------|
| **Groq API Key** (`httpHeaderAuth`) | All 4 Groq nodes (News, Website, Social, Jobs, Master) | Header name: `Authorization`, value: `Bearer YOUR_GROQ_KEY` |
| **Gmail OAuth2** | 📧 Gmail Report | Requires `gmail.send` scope |
| **Telegram API** | 📤 Telegram Alert | Bot token from @BotFather |
| **Google Sheets OAuth2** | 📊 Log to Sheets | Read/write access to target spreadsheet |
| **Discord** *(no credential)* | 📡 Discord Webhook | Webhook URL embedded directly in the HTTP Request node |

> ⚠️ **Groq API key:** v2 removed the hardcoded API key present in v1. You must create an `httpHeaderAuth` credential in n8n named `Groq API Key` before activating.

---

## Setup Instructions

### 1. Import the Workflow
- In n8n, go to **Workflows → Import from File**
- Select `Fintech_Intel_Bot_v2.json`

### 2. Create the Groq Credential
- Go to **Credentials → New → Header Auth**
- Name: `Groq API Key`
- Header Name: `Authorization`
- Header Value: `Bearer YOUR_GROQ_API_KEY`
- Assign this credential to all 5 Groq HTTP Request nodes

### 3. Configure Gmail
- Connect your Gmail OAuth2 credential to the **📧 Gmail Report** node
- Replace `YOUR_EMAIL@example.com` in the `sendTo` field with the recipient address

### 4. Configure Telegram
- Create a Telegram bot via [@BotFather](https://t.me/BotFather) and get the bot token
- Add the bot to your target chat/group and get the Chat ID
- Connect the credential to **📤 Telegram Alert** and replace `YOUR_TELEGRAM_CHAT_ID`

### 5. Configure Discord
- In your Discord server, go to **Channel Settings → Integrations → Webhooks → New Webhook**
- Copy the webhook URL
- In the **📡 Discord Webhook** node, replace `REPLACE_WITH_YOUR_DISCORD_WEBHOOK_URL`

### 6. Configure Google Sheets
- Connect your Google Sheets OAuth2 credential to **📊 Log to Sheets**
- Set the spreadsheet ID and sheet name where logs should be written

### 7. Activate the Workflow
- Click **Activate** — the scheduler will begin running every 6 hours automatically
- To test immediately, manually trigger **⏰ Every 6 Hours** via the "Execute Node" button

---

## Placeholders to Replace

Search the workflow for these values and replace them before activating:

| Placeholder | Node | Replace With |
|-------------|------|-------------|
| `YOUR_TELEGRAM_CHAT_ID` | 📤 Telegram Alert | Your Telegram chat/group ID |
| `YOUR_EMAIL@example.com` | 📧 Gmail Report | Recipient email address |
| `REPLACE_WITH_YOUR_DISCORD_WEBHOOK_URL` | 📡 Discord Webhook | Your Discord webhook URL |
| `YOUR_GROQ_CREDENTIAL_ID` | 🤖 Groq News | Resolved automatically once credential is created |
| `YOUR_TELEGRAM_CREDENTIAL_ID` | 📤 Telegram Alert | Resolved automatically once credential is created |
| `YOUR_GMAIL_CREDENTIAL_ID` | 📧 Gmail Report | Resolved automatically once credential is created |

---

## Output Format

Each pipeline (news, website, social, jobs) returns a structured JSON object:

```json
{
  "source": "news",
  "headline": "Single most important finding",
  "overall_threat": "HIGH",
  "importance_score": 8,
  "findings": [
    { "company": "Razorpay", "news": "key finding", "impact": "business impact", "score": 8 },
    { "company": "PhonePe",  "news": "key finding", "impact": "business impact", "score": 7 },
    { "company": "Paytm",    "news": "key finding", "impact": "business impact", "score": 6 },
    { "company": "Zepto",    "news": "key finding", "impact": "business impact", "score": 5 },
    { "company": "CRED",     "news": "key finding", "impact": "business impact", "score": 5 }
  ],
  "recommendation": "Single most important action this week"
}
```

The **Parse Master** node additionally produces:
- `is_high_threat` — boolean flag driving the IF node
- `threat_level` — string used in the Gmail subject line
- `telegram_message` — pre-formatted Markdown for Telegram
- `gmail_html` — HTML-formatted email body

---

## Known Bug Fixes (v2 Changes)

The following issues from v1 were resolved in this version (documented in sticky notes inside the workflow):

| Fix | Description |
|-----|-------------|
| **Groq credential hardcoded** | API key was embedded directly in node parameters. v2 uses a proper `httpHeaderAuth` credential |
| **CRED port collision** | CRED Site was wired to port 3 (same as Zepto) on Merge Websites, causing data to be dropped. Fixed by using unique merge ports |
| **Reddit HTML scraping** | Old Reddit nodes used browser search URLs that returned HTML, not data. v2 uses the Reddit JSON API (`/search.json`) |
| **Product website scraping** | HTTP GET requests to competitor homepages failed due to JS rendering and bot detection. Replaced with Google News RSS product-keyword queries |
| **Gemini AI Agent removed** | A Gemini-powered AI Agent was receiving an already-formatted `telegram_message` string and had no useful role. Removed to simplify the pipeline |
| **Sheets field undefined** | `$json.strategic_recommendation` was always undefined. Parse Master was updated to use the correct field name |

---

## Troubleshooting

**All Groq nodes fail with 401 Unauthorized**
- The `httpHeaderAuth` credential must have the value `Bearer YOUR_KEY` (include the word `Bearer`)
- Confirm the credential is assigned to all 5 Groq HTTP Request nodes

**No data coming from Reddit nodes**
- Reddit's JSON API may throttle requests without a proper `User-Agent` header — these are already set in the nodes
- If Reddit returns a 429, increase the schedule interval or add a Wait node between Reddit calls

**Master Merge never fires**
- The Master Merge uses a 4-input merge and waits for all branches. If any one Groq call fails, the merge stalls
- Add error handling nodes (n8n Error Trigger) to catch and pass a fallback object when a branch fails

**Gmail report not sending**
- Ensure the Gmail OAuth2 credential has the `gmail.send` scope
- Check that `YOUR_EMAIL@example.com` is replaced with a real address

**Telegram not delivering messages**
- Verify the bot has been added to the target chat/group
- Confirm `YOUR_TELEGRAM_CHAT_ID` is a numeric ID (not a username). Use `@userinfobot` on Telegram to find it

**Google Sheets not logging**
- Confirm the spreadsheet ID and sheet tab name are correctly set in the **📊 Log to Sheets** node
- Ensure the column headers in the sheet match what Parse Master is outputting
