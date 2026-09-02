# Customer Email Support Agent

An AI agent, built in n8n, that reads incoming customer support emails, checks a real knowledge base / SOP document, and either replies automatically or routes the email to a human for one-click approval.

Built as a working proof-of-concept for a larger project: an "AI Agent Builder" that generates agents like this one from a single natural-language prompt.

## What it does

1. **Watches** a Gmail inbox for new customer emails.
2. **Checks confidence** against a knowledge base (shipping policy, refund policy, escalation rules) using Google Gemini.
3. **If confident** → drafts a formal reply and sends it automatically.
4. **If not confident** (hostile tone, high-value refund, missing info, etc.) → drafts a reply anyway, flags the reason, and logs it to a Google Sheet + Slack alert instead of sending blind.
5. **Human review** happens through a small web dashboard (`approval-queue-dashboard.html`) with tick/cross buttons that actually call back into n8n to send or discard the queued reply.

## Architecture

```
Gmail Trigger
     │
     ▼
AI Agent (Gemini) ── consults knowledge base in system prompt
     │
     ▼
Confidence check (IF node)
     │
   ┌─┴─────────────┐
 high             low
   │                │
   ▼                ▼
Send Reply    Google Sheet (queue) → Slack alert
                    │
                    ▼
        Approval dashboard (tick/cross)
           │                    │
       Approve → send      Discard → mark done
      (via webhook)        (via webhook)
```

## Files in this repo

| File | What it is |
|---|---|
| `customer-email-reply-agent.json` | The main n8n workflow: Gmail trigger → AI triage agent → auto-send or queue. Import into n8n and fill in your own Google Sheet ID, Slack channel ID, and credentials. |
| `approval-queue-dashboard.html` | Standalone web dashboard for reviewing queued emails. Open directly in a browser. Calls a small companion n8n workflow (webhooks) to actually approve/reject. |
| `system-prompt.md` | The full instructions given to the AI agent, including the knowledge base format and confidence rules. |

## What I learned building this

- AI agent output from a structured parser often comes back nested (e.g. `output.confidence` rather than `confidence`) — every downstream node needs to reference the right path, or conditions silently fail.
- Different LLM providers have different quirks with structured output / function-calling schemas (e.g. Gemini rejects JSON Schema union types like `["string","null"]` that OpenAI accepts).
- Building the human-in-the-loop approval step as a *real* interactive dashboard (not just a spreadsheet) was necessary to make this genuinely usable day-to-day, not just a demo.

## Status

Fully functional and tested end-to-end (auto-send path and human-review path both verified). Not yet running continuously in production — the main workflow is intentionally left unpublished until ready to go live.
