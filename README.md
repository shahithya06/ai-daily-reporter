# 🤖 ai-daily-reporter

> An n8n automation pipeline that pulls live data from your database, CRM, and Google Analytics — then uses Claude AI to write a human-quality narrative report, delivered to Slack and email every morning at 7 AM.

---

![n8n](https://img.shields.io/badge/n8n-workflow-orange?logo=n8n&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-Sonnet_4.6-6B48FF?logo=anthropic&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![Trigger](https://img.shields.io/badge/Trigger-Daily_7AM_CRON-blue)
![Integrations](https://img.shields.io/badge/Integrations-Postgres_%7C_HubSpot_%7C_GA_%7C_Slack_%7C_Email-informational)

---

## Table of Contents

- [Overview](#overview)
- [Motivation](#motivation)
- [Key Features](#key-features)
- [Pipeline Architecture](#pipeline-architecture)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [Report Output Example](#report-output-example)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Author](#author)
- [Acknowledgements](#acknowledgements)

---

## Overview

**ai-daily-reporter** is a plug-and-play n8n workflow that replaces manual morning reporting. It wakes up every day at 7 AM, pulls yesterday's numbers from three sources — your PostgreSQL database, HubSpot CRM, and Google Analytics — structures them into a rich context payload, and sends the data to Claude (Anthropic) to generate a clear, executive-grade narrative report. The report is delivered to Slack daily, with a full version emailed to leadership every Friday.

---

## Motivation

Most teams spend the first hour of their day copying numbers from three different dashboards into a Slack message nobody reads carefully. This pipeline eliminates that entirely:

- **No more manual reporting** — the workflow runs while your team sleeps
- **No more data silos** — product metrics, sales, and web analytics are synthesised in one place
- **No more bland dashboards** — Claude writes a report that reads like it came from a senior analyst, with highlights, concerns, trends, and recommendations

---

## Key Features

- **Multi-source data aggregation** — Pulls from PostgreSQL, HubSpot, and Google Analytics in parallel, then merges them before AI processing
- **MCP-style context structuring** — A dedicated Code node shapes raw data into a rich, versioned context payload with explicit instructions for the AI (tone, format, required sections, word limit)
- **AI narrative generation via Claude** — Uses `claude-sonnet-4-6` at `temperature: 0.4` for consistent, professional prose with specific numbers and actionable recommendations
- **Smart delivery routing** — Sends a concise Slack pulse every weekday; on Fridays, also emails the full report to leadership
- **Fully credential-driven** — All secrets live in n8n credentials, never hardcoded in the workflow

---

## Pipeline Architecture

```
Daily Trigger (7 AM CRON)
        │
        ├──► Pull Database Metrics (Postgres)   ─┐
        ├──► Pull CRM Data (HubSpot)             ├──► Merge All Sources
        └──► Pull Google Analytics               ─┘
                                                        │
                                                MCP Context Builder (Code)
                                                        │
                                            AI Narrative Generation (Claude)
                                                        │
                                              Format Report Payload (Code)
                                                        │
                                              Is it Friday? (IF node)
                                               /                    \
                                    YES: Send Email          NO: Skip Email
                                      + Slack Full              Slack Pulse
                                        Report                   Only
```

---

## Prerequisites

| Tool / Service | Version / Notes |
|---|---|
| [n8n](https://n8n.io) | `>= 1.30.0` (self-hosted or cloud) |
| [Anthropic API key](https://console.anthropic.com) | Claude Sonnet 4.6 access |
| PostgreSQL database | With a `daily_metrics` table (schema below) |
| HubSpot account | API key or OAuth2 |
| Google Analytics | UA or GA4 view with OAuth2 credentials |
| Slack workspace | OAuth2 app with `chat:write` scope |
| SMTP server | Any provider (Gmail, SendGrid, Postmark, etc.) |

### Expected PostgreSQL schema

```sql
CREATE TABLE daily_metrics (
  date          DATE PRIMARY KEY,
  revenue       NUMERIC(12, 2),
  new_users     INTEGER,
  churn_rate    NUMERIC(5, 4),
  active_sessions INTEGER
);
```

---

## Installation

### 1. Clone the repository

```bash
git clone https://github.com/your-username/ai-daily-reporter.git
cd ai-daily-reporter
```

### 2. Import the workflow into n8n

**Option A — n8n UI:**

1. Open your n8n instance
2. Go to **Workflows → Import from file**
3. Select `ai_reporting_pipeline.json`

**Option B — n8n CLI:**

```bash
n8n import:workflow --input=ai_reporting_pipeline.json
```

### 3. Set up credentials in n8n

In your n8n instance, create the following credentials and note their IDs:

| Credential Name | Type |
|---|---|
| `Production DB` | Postgres |
| `HubSpot CRM` | HubSpot API |
| `Google Analytics` | Google Analytics OAuth2 |
| `Anthropic API` | Anthropic API |
| `Slack Workspace` | Slack OAuth2 |
| `SMTP Email` | SMTP |

### 4. Update placeholder values in the workflow

Search the JSON for the following strings and replace them with your real values before importing, or update them via the node editors after importing:

```
YOUR_POSTGRES_CREDENTIAL_ID     → Your n8n Postgres credential ID
YOUR_HUBSPOT_CREDENTIAL_ID      → Your n8n HubSpot credential ID
YOUR_GA_CREDENTIAL_ID           → Your n8n Google Analytics credential ID
YOUR_GA_VIEW_ID                 → Your Google Analytics View ID
YOUR_ANTHROPIC_CREDENTIAL_ID    → Your n8n Anthropic credential ID
YOUR_SLACK_CREDENTIAL_ID        → Your n8n Slack credential ID
YOUR_SMTP_CREDENTIAL_ID         → Your n8n SMTP credential ID
```

Also update email addresses in the **Send Email Report** node:

```
reports@yourcompany.com         → Your sender address
leadership@yourcompany.com      → Your recipients
```

### 5. Activate the workflow

Toggle the workflow to **Active** in n8n. It will fire automatically at 7 AM server time.

---

## Configuration

All behaviour is controlled inside the **MCP Context Builder** code node. Key settings:

```js
instructions: {
  tone: 'professional yet conversational',   // Adjust Claude's writing style
  format: 'executive summary with sections', // Report format hint
  sections_required: [                       // Sections Claude must include
    'highlights',
    'concerns',
    'trends',
    'recommendations'
  ],
  max_length: '400 words'                    // Word limit passed to Claude
}
```

Claude model and sampling settings are in the **AI Narrative Generation** node:

```json
{
  "modelId": "claude-sonnet-4-6",
  "maxTokens": 1024,
  "temperature": 0.4
}
```

Increase `temperature` (up to `1.0`) for more creative prose; lower it toward `0.1` for more deterministic, data-focused output.

The Slack channel is set to `#daily-reports` in both Slack nodes. Change `channel` to any channel name or ID.

---

## Usage

### Manual test run

In n8n, open the workflow and click **Execute Workflow**. All three data sources will be queried, the AI will generate a report, and delivery will follow the Friday/non-Friday routing logic based on today's actual date.

### Change the trigger time

Open the **Daily Trigger (7AM)** node and update the cron expression:

```
0 7 * * *   →  Every day at 7:00 AM (server timezone)
0 6 * * *   →  Every day at 6:00 AM
0 8 * * 1-5 →  Weekdays only at 8:00 AM
```

### Extend with a new data source

1. Add a new node after the trigger (e.g., Stripe, Mixpanel, Salesforce)
2. Wire it into **Merge All Data Sources** on a new index
3. In the **MCP Context Builder** code node, extract the new data with `items[3]?.json` and add it to the `data` object
4. Update the Claude prompt to reference the new section

---

## Report Output Example

```
📊 Daily Performance Report — 2026-03-15

**Executive Highlights**
Yesterday was a strong day: revenue hit $48,200 (+12% vs. the 7-day average),
new user signups reached 340, and the sales team closed 4 deals worth $92,000
in combined pipeline value.

**Areas of Concern**
Bounce rate climbed to 68% (up from 54% last week), suggesting a mismatch
between traffic sources and landing page content. Churn rate also ticked up
to 2.3% — worth monitoring over the next 48 hours.

**Key Trends**
Organic search continues to be the top acquisition channel (41% of sessions).
Goal completions from direct traffic are up 22% week-over-week.

**Recommendations**
1. A/B test the homepage hero against the paid traffic landing page.
2. Trigger a re-engagement email sequence for cohorts showing early churn signals.
3. Double down on the SEO content cluster driving organic growth.

---
This report was auto-generated by the AI Reporting Pipeline.
```

---

## Roadmap

- [ ] **Google Sheets export** — Write each daily report to a running Google Sheet for historical tracking
- [ ] **Anomaly detection pre-pass** — Add a Code node that flags statistical outliers before the AI step, injecting them as priority context
- [ ] **Slack slash command** — Trigger an on-demand report for any date via `/report 2026-03-10`
- [ ] **Multi-language support** — Parameterise the Claude prompt to generate reports in French, Spanish, or German
- [ ] **Error notification node** — Route workflow failures to a `#alerts` channel with a diagnostic summary

---

## Contributing

Contributions, bug reports, and feature requests are welcome. Please open an issue before submitting a pull request so we can discuss the change.

1. Fork the repository
2. Create a feature branch: `git checkout -b feat/my-new-source`
3. Commit your changes: `git commit -m 'feat: add Stripe revenue node'`
4. Push and open a PR against `main`

See [CONTRIBUTING.md](./CONTRIBUTING.md) for full guidelines.

---

## Author

Built and maintained by **[Your Name]**
- GitHub: [@your-username](https://github.com/your-username)
- Email: you@yourcompany.com
- LinkedIn: [linkedin.com/in/yourprofile](https://linkedin.com/in/yourprofile)

---

## Acknowledgements

- [n8n](https://n8n.io) — the open-source workflow automation platform that makes this possible
- [Anthropic](https://anthropic.com) — for Claude and the Messages API
- [Model Context Protocol](https://modelcontextprotocol.io) — inspiration for the structured context-passing pattern used in the MCP Context Builder node

---

*Last updated: March 2026 · Workflow version 1.0*
