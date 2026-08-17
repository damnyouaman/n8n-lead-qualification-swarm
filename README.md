# Inbound Lead Qualification & Routing Swarm

An n8n workflow that catches inbound leads from a website form, researches the company with an AI agent, scores it against an ICP rubric, and routes it — hot leads to a sales rep, cold leads to a nurture list.

**Workflow ID:** `5EjpPLqvYf9l75W7` · **14 nodes** · Endpoint: `POST /webhook/inbound-lead`

📄 **[Block diagram (PDF)](docs/workflow-diagram.pdf)** — one page, shows every node and what it's for.

---

## How it works

### Stage 1 — Enrich & Score

| Node | Type | Purpose |
|---|---|---|
| **Lead Webhook** | `webhook` | Catches the form POST. Reads `name`, `email`, `company` from `$json.body`. Replies `200` immediately so the form never waits on the AI. |
| **Enrichment Agent** | `langchain.agent` | Researches the company and writes a 3-paragraph brief: what they do, firmographics, recent buying signals. |
| ↳ OpenAI — Enrichment | `lmChatOpenAi` | `gpt-5-mini`. Drives the research loop. |
| ↳ Search Google for company info | `toolSerpApi` | SerpAPI. The agent's only source of facts. |
| **Score Lead** | `chainLlm` | Grades the brief against the ICP rubric, returns a number out of 100. |
| ↳ OpenAI — Scoring | `lmChatOpenAi` | `gpt-5`. Shared with the parser and the email writer. |
| ↳ Score Output Parser | `outputParserStructured` | Forces JSON `{score, reasoning, data_confidence}`. Auto-fix retries malformed output. |

### Stage 2 — Route & Act

| Node | Type | Purpose |
|---|---|---|
| **Normalize Lead + Score** | `set` | Flattens everything into one clean item. **The single place to fix** if the score output shape changes. |
| **Hot Lead?** | `if` | Numeric test: `score >= 80`. True → sales path, false → nurture. |
| **HubSpot — Assign to AE** | `hubspot` (contact: upsert) | Creates/updates the contact, assigns the AE via `contactOwner`. |
| **Slack — Alert Sales Team** | `slack` (message: post) | Posts to `#sales-alerts` with score, reasoning and the full brief. |
| **Write Intro Email** | `chainLlm` | Drafts a <120-word intro opening on one real detail from the brief. |
| **Gmail — Draft Intro Email** | `gmail` (draft: create) | **Creates a draft only — never sends.** Waits for human review. |
| **HubSpot — Add to Nurture List** | `hubspot` (contactList: add) | Cold path. Drops the lead into the 30-day marketing list. No rep is alerted. |

### The ICP rubric

Lives in the **Score Lead** system prompt. 100 points across five dimensions:

| Dimension | Points |
|---|---|
| Company size | 30 (50–500 employees scores full) |
| Industry fit | 25 (B2B SaaS / software / services scores full) |
| Buying signals, last 24 months | 25 (funding, acquisition, senior hire, expansion) |
| Email quality | 10 (corporate domain matching the company) |
| Geography | 10 (US / CA / UK / EU / AU / NZ) |

**Hard disqualifiers cap the score at 20:** unidentifiable company, job seeker, direct competitor, staffing/SEO spam.

**Unknown-data rule:** if research can't find a dimension, it scores the *midpoint*, not zero — a lead is never punished for a gap in our research. `data_confidence` drops instead.

---

## Design decisions worth knowing

- **SerpAPI is the only enrichment source.** No Clearbit/Apollo, so headcount and revenue are web-search-derived and often unknown. Both prompts are written to emit an explicit `"unknown"` rather than hallucinate a number, and the rubric degrades gracefully.
- **Both LLM steps and all three integrations retry twice.** HubSpot, Slack and Gmail use `onError: continueRegularOutput`, so a CRM hiccup can never swallow the Slack alert for a hot lead.
- **Gmail drafts, never sends.** A human always reviews before anything reaches a prospect.
- **`toolSerpApi` is deprecated upstream.** It still works and is built in. The replacement is the verified community node `n8n-nodes-serpapi.serpApi`, which requires a community-node install.

---

## Setup

### 1. Configure the n8n MCP server

```bash
cp .mcp.json.example .mcp.json
# then edit .mcp.json with your instance URL and API key
```

Get the API key from n8n → **Settings → n8n API**. The public API must be enabled — it is **not available on the n8n Cloud free trial** (every `/api/v1/*` path 404s), so use a paid plan or a self-hosted instance.

### 2. Install the n8n skill pack

```bash
/plugin install czlonkowski/n8n-skills
```

Or manually: `git clone https://github.com/czlonkowski/n8n-skills.git && cp -r n8n-skills/skills/* .claude/skills/`

### 3. Import the workflow

Import `workflows/inbound-lead-qualification-swarm.json` via the n8n UI, or deploy it with the MCP tools.

### 4. Add credentials in the n8n UI

| Credential | Used by |
|---|---|
| OpenAI | OpenAI — Enrichment, OpenAI — Scoring |
| SerpApi | Search Google for company info |
| HubSpot (App Token) | both HubSpot nodes |
| Slack | Slack — Alert Sales Team |
| Gmail (OAuth2) | Gmail — Draft Intro Email |

### 5. Fill the three config placeholders

These are **not** credentials — the workflow will fail at runtime until they're set:

| Node | Field | Set it to |
|---|---|---|
| HubSpot — Assign to AE | `contactOwner` | The AE's numeric HubSpot owner ID (currently `REPLACE_WITH_AE_OWNER_ID`) |
| HubSpot — Add to Nurture List | `listId` | Your 30-day nurture list ID (currently `0`) |
| Slack — Alert Sales Team | channel | Confirm `sales-alerts` is right |

### 6. Point your form at the webhook

```
POST https://<your-n8n-host>/webhook/inbound-lead
Content-Type: application/json

{ "name": "Jane Doe", "email": "jane@company.com", "company": "Company Inc" }
```

---

## Current status

⚠️ **Not yet verified end-to-end.** The first live run failed at the Enrichment Agent with
`OpenAI: You exceeded your current quota` — the OpenAI API key has no credits. OpenAI does not
grant free API credits to new accounts; billing must be funded at
[platform.openai.com billing](https://platform.openai.com/settings/organization/billing).

What that run **did** confirm: the webhook fires, the payload lands at `$json.body` exactly as
expected, and the Enrichment Agent is reached and correctly wired. Everything downstream of the
first OpenAI call — the score output shape, Slack, Gmail, HubSpot — is still unproven.

---

## Repo layout

```
├── workflows/
│   └── inbound-lead-qualification-swarm.json   # portable export, credentials stripped
├── docs/
│   ├── workflow-diagram.pdf                    # one-page block diagram
│   └── workflow-diagram.html                   # diagram source
├── .mcp.json.example                           # MCP config template (copy to .mcp.json)
└── .gitignore                                  # blocks .mcp.json — it holds the live API key
```

**`.mcp.json` is gitignored** because it contains a live n8n API key. Never commit it.
