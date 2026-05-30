# AI Research Assistant — Antigravity + n8n Automation Guide

A practical reference for building an AI-powered research digest pipeline using n8n as the workflow engine and Antigravity as the deployment/hosting layer.

---

## Overview

This automation fetches research papers from arXiv, summarises them using an AI model (Claude or GPT-4o), and delivers a formatted digest to Notion and/or email — all triggered on a schedule or on demand.

```
Trigger → Fetch arXiv → Parse XML → AI Summarise → Format → Notion / Gmail
```

---

## Prerequisites

| Tool | Purpose | Notes |
|------|---------|-------|
| n8n | Workflow engine | Self-host via Docker or use n8n Cloud |
| Antigravity | Deployment / env management | Hosts your n8n instance and secrets |
| OpenAI or Claude API | AI summarisation | Requires API key |
| Notion API | Output storage | Optional |
| Gmail | Email delivery | Optional |

---

## Antigravity Setup

Antigravity manages your environment variables and hosts the n8n instance so you do not hardcode secrets into your workflows.

### 1. Install and initialise

```bash
pip install antigravity
antigravity init my-research-bot
cd my-research-bot
```

### 2. Set secrets

```bash
antigravity secrets set OPENAI_API_KEY=sk-...
antigravity secrets set NOTION_TOKEN=secret_...
antigravity secrets set GMAIL_USER=you@gmail.com
antigravity secrets set N8N_WEBHOOK_URL=https://your-n8n-instance/webhook/research
```

### 3. Deploy n8n with environment injection

```bash
antigravity deploy --env production --service n8n
```

Antigravity injects the secrets as environment variables at runtime. Reference them inside n8n using the `$env` object:

```javascript
// Inside any n8n Code or Expression node
const apiKey = $env.OPENAI_API_KEY;
```

### 4. Expose the webhook (optional)

If you want to trigger the workflow from outside:

```bash
antigravity expose webhook --port 5678 --path /webhook/research
```

---

## n8n Workflow — Node-by-Node Setup

### Node 1 — Trigger

**Type:** Schedule Trigger (daily digest) or Webhook (on-demand)

**Schedule example — every day at 7:00 AM:**

```
Cron expression: 0 7 * * *
```

**Webhook example:**

- Method: `POST`
- Path: `/research`
- Body (JSON):
  ```json
  {
    "topic": "large language models",
    "max_results": 5
  }
  ```

---

### Node 2 — HTTP Request (arXiv API)

**Type:** HTTP Request

| Field | Value |
|-------|-------|
| Method | GET |
| URL | `https://export.arxiv.org/api/query` |
| Query params | `search_query=ti:{{$json.topic}}&max_results={{$json.max_results}}&sortBy=submittedDate&sortOrder=descending` |

> No API key required. arXiv is publicly accessible.

---

### Node 3 — Code Node (Parse XML)

**Type:** Code (JavaScript)

Paste the following into the Code node to extract paper titles, authors, and abstracts from the arXiv Atom XML response:

```javascript
const xml = $input.first().json.data;

// Simple regex-based parser for arXiv Atom feed
const entries = [];
const entryBlocks = xml.match(/<entry>([\s\S]*?)<\/entry>/g) || [];

for (const block of entryBlocks) {
  const get = (tag) => {
    const match = block.match(new RegExp(`<${tag}[^>]*>([\\s\\S]*?)<\\/${tag}>`));
    return match ? match[1].trim() : '';
  };

  entries.push({
    title: get('title').replace(/\n/g, ' '),
    abstract: get('summary').replace(/\n/g, ' ').substring(0, 800),
    authors: [...block.matchAll(/<name>(.*?)<\/name>/g)].map(m => m[1]).join(', '),
    link: (block.match(/href="(https:\/\/arxiv\.org\/abs\/[^"]+)"/) || [])[1] || '',
    published: get('published').substring(0, 10),
  });
}

return entries.map(e => ({ json: e }));
```

---

### Node 4 — AI Summarise (OpenAI or Claude)

#### Option A — OpenAI node (built-in n8n credential)

| Field | Value |
|-------|-------|
| Resource | Message a model |
| Model | `gpt-4o` |
| Prompt | See below |

#### Option B — HTTP Request to Claude API

| Field | Value |
|-------|-------|
| Method | POST |
| URL | `https://api.anthropic.com/v1/messages` |
| Header `x-api-key` | `{{$env.ANTHROPIC_API_KEY}}` |
| Header `anthropic-version` | `2023-06-01` |
| Body (JSON) | See below |

**Prompt template (use in either option):**

```
You are a research assistant helping a master's student in AI.

Paper title: {{$json.title}}
Authors: {{$json.authors}}
Published: {{$json.published}}

Abstract:
{{$json.abstract}}

Please provide:
1. A 2-sentence plain-English summary of the paper's contribution
2. Three bullet points of key takeaways
3. One sentence on why this matters for AI research today

Be concise. Use clear academic language.
```

---

### Node 5 — Set Node (Format Digest)

**Type:** Set

Assemble the AI output into a clean Markdown digest block:

```javascript
// Expression field value
`## {{$json.title}}

**Authors:** {{$json.authors}} | **Published:** {{$json.published}}

{{$json.summary}}

[Read paper →]({{$json.link}})

---`
```

---

### Node 6a — Notion Node (Save to workspace)

| Field | Value |
|-------|-------|
| Resource | Page |
| Operation | Create |
| Database ID | Your Notion DB ID |
| Title | `{{$json.title}}` |
| Content | `{{$json.formatted_digest}}` |

Add properties: `Authors`, `Published Date`, `arXiv Link`, `AI Summary`.

---

### Node 6b — Gmail Node (Send digest email)

| Field | Value |
|-------|-------|
| Operation | Send |
| To | `{{$env.GMAIL_USER}}` |
| Subject | `Research Digest — {{$now.format('YYYY-MM-DD')}}` |
| Body type | HTML |
| Body | Wrap `$json.formatted_digest` in `<pre>` or convert Markdown to HTML |

---

## Antigravity + n8n Integration Tips

- Use `antigravity logs --service n8n` to stream workflow execution logs
- Use `antigravity env pull` to sync your `.env` file locally for testing
- To run the workflow locally before deploying, use `antigravity run local`
- Tag workflow versions: `antigravity deploy --tag v1.0-research-bot`

---

## Folder Structure (recommended)

```
my-research-bot/
├── antigravity.yaml          # Service config
├── .env.example              # Non-secret env template
├── workflows/
│   └── research_assistant.json   # Exported n8n workflow (importable)
├── scripts/
│   └── parse_arxiv.js        # Standalone XML parser for testing
└── README.md
```

---

## Extending the Workflow

| Extension | How |
|-----------|-----|
| Multiple topics | Add a `Split in Batches` node before arXiv fetch |
| PDF download | Add an HTTP node fetching `arxiv.org/pdf/{id}` |
| Vector storage | Add a Pinecone or Qdrant node after summarisation |
| Slack notification | Add a Slack node in parallel with Gmail |
| Weekly summary | Add a second Schedule trigger + aggregation Code node |

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| arXiv returns no results | Check the `search_query` URL encoding — spaces must be `+` |
| AI node times out | Reduce `max_results` to 3 or add a `Wait` node between calls |
| Notion page not created | Verify the database ID and that the integration is added to the DB |
| Secrets not available | Run `antigravity secrets list` to confirm they are set |
| n8n webhook not reachable | Run `antigravity expose` and check the tunnel URL |

---

## Resources

- [n8n documentation](https://docs.n8n.io)
- [arXiv API user manual](https://info.arxiv.org/help/api/user-manual.html)
- [Anthropic API reference](https://docs.anthropic.com)
- [Notion API reference](https://developers.notion.com)
- Antigravity CLI: `antigravity --help`
