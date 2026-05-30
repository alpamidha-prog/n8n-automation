# AI Research Assistant — n8n & Antigravity Automation

An automated, AI-powered research pipeline that fetches the latest academic papers from arXiv on scheduled intervals, generates concise summaries with key takeaways using OpenAI or Claude, and sends a formatted digest to Notion and/or email.

This repository contains the complete n8n workflow configuration and a step-by-step implementation guide.

---

## 📂 Project Structure

* **[n8n_automation.md](file:///d:/XAMK/Courses/AI%20in%20practice/Lessons/n8n%20automation/n8n_automation.md)**: A complete, node-by-node setup guide explaining how the HTTP request parser, XML processing, LLM prompts, and integrations work.
* **[research_assistant_workflow.json](file:///d:/XAMK/Courses/AI%20in%20practice/Lessons/n8n%20automation/research_assistant_workflow.json)**: The exported n8n workflow file. You can import this JSON directly into your own n8n instance to get started immediately.

---

## ⚡ Quick Start

### 1. Import Workflow to n8n
1. Open your n8n workspace.
2. Create a new workflow.
3. Click the menu in the top-right corner and select **Import from File**, then upload `research_assistant_workflow.json` (or copy-paste the raw JSON text).

### 2. Configure Environment & Secrets
For secure hosting and environment management, we use **Antigravity**. Configure your secrets as follows:

```bash
# Set up your Antigravity environment
antigravity secrets set OPENAI_API_KEY=sk-...
antigravity secrets set NOTION_TOKEN=secret_...
antigravity secrets set GMAIL_USER=your_email@gmail.com
```

### 3. Setup Triggers
* **Schedule Trigger**: Runs automatically at set intervals (e.g., daily at 7:00 AM).
* **Webhook Trigger**: Can be invoked on-demand via a POST request with your target topic and limits:
  ```json
  {
    "topic": "large language models",
    "max_results": 5
  }
  ```

---

## 📘 Detailed Setup & Guide
For a deep dive into how each node works, how to configure the Notion database schema, and custom JavaScript parsers, refer to the full **[Setup Guide](file:///d:/XAMK/Courses/AI%20in%20practice/Lessons/n8n%20automation/n8n_automation.md)**.
