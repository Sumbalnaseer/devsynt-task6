# Task 6 — AI Email Triage & RAG Assistant

An n8n-based automation that monitors a Gmail inbox, classifies every incoming email by intent and priority using an LLM, and routes it automatically: grounded RAG replies for general/sales questions, forwarding + Discord alerts for HR/Manager/Urgent cases, human notification for meetings, and archive/spam handling for the rest. Built as part of the DevSynt AI Internship program.

## Overview

The system watches an inbox, extracts and cleans each email, checks for duplicates, classifies it with an LLM into one of 11 categories with a priority and confidence score, then routes it through a decision engine. General and sales questions are answered by a Retrieval-Augmented Generation (RAG) pipeline reused from Task 5, grounded in a real-estate knowledge base — the system never invents an answer that isn't in the documents. Every processed email is logged, and the relevant human channel is notified via Discord.

## Architecture

```
Gmail Trigger
     |
Parse & Clean (Code node)
     |
Duplicate Check (Google Sheets lookup + IF)
     |
AI Classification (Groq LLM — openai/gpt-oss-20b)
     |
Parse Classification (Code node)
     |
   Switch (routes on category)
     |
  ┌──┴───────────────────────────────────────────────┐
  |         |         |         |        |      |     |
 RAG    Job App     Project/  Meeting  Urgent  Promo  Spam
  |     (→ HR)      Internal    |        |       |      |
  |         |       (→Mgr)      |        |       |      |
Create    Forward   Forward   Discord  Discord  Remove  Add
Session   + Discord + Discord  notify   alert   label   label
  |
Query RAG
(Task 5 backend)
  |
Reply to
sender
  └──────────────────────┬─────────────────────────────┘
                          |
                  Log to Google Sheets
                  (every branch appends a row)
```

## Tech Stack

- **Automation platform:** n8n (self-hosted)
- **Email:** Gmail API (Gmail Trigger + Gmail action nodes)
- **AI classification:** Groq API, model `openai/gpt-oss-20b`
- **RAG backend:** reused Task 5 FastAPI service (session-based chat API), ChromaDB vector store, sentence-transformers embeddings
- **Notifications:** Discord webhooks (separate channels for HR, Manager, Urgent)
- **Logging:** Google Sheets (Task6_Email_Log)

## Setup

### Prerequisites
- n8n instance (self-hosted or cloud)
- Gmail account with OAuth credential configured in n8n
- Groq API key ([console.groq.com](https://console.groq.com))
- Discord server with 3 webhook-enabled text channels (HR, Manager, Urgent)
- Google Sheets document for logging
- Task 5 RAG backend running locally (or deployed) and reachable from n8n

### Environment / Credentials
| Name | Where used | Notes |
|---|---|---|
| Gmail OAuth | Gmail Trigger + Gmail action nodes | Standard n8n Gmail credential |
| Groq API key | HTTP Request (classification) node | Passed as `Authorization: Bearer <key>` header credential |
| Discord webhook URLs (×3) | HTTP Request nodes per branch | One per channel: HR, Manager, Urgent |
| Google Sheets OAuth | Google Sheets nodes (dedup + logging) | Same account owning the log sheet |
| Task 5 backend URL | RAG branch HTTP Request nodes | `http://127.0.0.1:8000` in local testing |

### Import & Run
1. Import `workflow.json` into n8n
2. Reconnect all credentials (Gmail, Groq header auth, Google Sheets)
3. Update the 3 Discord webhook URLs and the Google Sheet URL/ID for your own accounts
4. Start the Task 5 backend (`uvicorn app.main:app --reload --port 8000`) for RAG to work
5. Activate the workflow

## Email Classification & Routing Logic

Every email is classified into one of: `general_query`, `sales_inquiry`, `job_application`, `project_related`, `internal_communication`, `meeting_request`, `urgent_request`, `complaint`, `promotional`, `spam`, `other`.

| Category | Action |
|---|---|
| general_query, sales_inquiry | RAG-grounded email reply |
| job_application | Forward to HR + Discord alert |
| project_related, internal_communication, complaint | Forward to Manager + Discord alert |
| meeting_request | Discord notification, no auto-reply |
| urgent_request | High-priority Discord alert |
| promotional | Archived (label removed from inbox) |
| spam | Labeled as spam |
| other / unclassified | Fallback Discord notification for human review |

The classification prompt includes explicit guardrails: legitimate transactional emails (e.g. bank alerts) are never misclassified as spam even when they contain fraud-warning boilerplate, and ambiguous cases default to `other` rather than aggressive spam-marking.

## RAG Implementation

The RAG branch calls the Task 5 backend's session-based chat API:
1. `POST /api/chat/sessions` — creates a new chat session
2. `POST /api/chat/message` — sends the email body as the question, receives a grounded answer with source citations (document name + page number)

The answer is embedded into an email reply sent back to the original sender via the Gmail thread.

## Human-in-the-Loop & Reliability

- Duplicate protection via Gmail message ID lookup against the log sheet before any processing occurs
- Every branch logs: timestamp, message/thread ID, sender, subject, category, priority, confidence, requires_human flag, action taken, RAG usage, response status, forwarded-to, Discord status
- Low-confidence or unclassifiable emails fall through to a dedicated human-review Discord channel rather than being silently dropped

## Limitations

- Classification occasionally misjudges tone-driven urgency (e.g. deadline language can push a routine update into the Urgent branch instead of Manager)
- RAG branch depends on the Task 5 backend running and reachable; if it's down, in-flight requests to that branch will fail rather than gracefully degrading to a "contact us" fallback
- Gmail's own spam filter can move genuinely spam-classified test emails into the Spam folder before n8n's trigger fully processes them, which can affect testing but not production behavior
- Meeting requests are only notified, not auto-scheduled — this is by design per the task's human-in-the-loop requirement

## Demo Video
[link here]

## LinkedIn Post
[link here]
