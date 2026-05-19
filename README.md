# BookLeaf Author Query Bot

An AI-powered, multi-channel-ready customer support bot for BookLeaf Publishing — built for the AI Automation Specialist technical assignment.

---

## What It Does

Authors send natural language queries like *"Is my book live yet?"* or *"When will I get my royalty?"* — the bot understands the intent, fetches their personal data from a Supabase database, and responds with accurate, personalised information. If it can't understand the query, it escalates to a human agent automatically.

---

## Tech Stack

| Layer | Tool | Why |
|---|---|---|
| Workflow Engine | **n8n** (cloud) | Visual, auditable, no-code-first — aligns with the role requirement |
| LLM | **OpenAI GPT-4o-mini** | Fast, cost-efficient, reliable JSON output |
| Database | **Supabase** (PostgreSQL) | Real-time, free tier, REST API natively supported |
| Chat UI | **n8n Chat Trigger** | Built-in, zero frontend code, public URL out of the box |
| Memory | **n8n Window Buffer Memory** | Maintains conversation context across multiple turns |

---

## Architecture

```
User Message
      ↓
[Chat Trigger] — built-in chat UI, session tracking
      ↓
[Build Prompt] — constructs system prompt with full KB + rules
      ↓
[AI Agent + Window Buffer Memory] — GPT-4o-mini, 12-message context window
      ↓
[Parse GPT JSON] — robust extraction with safe fallbacks
      ↓
[Smart Router] — 4 routes based on confidence + intent
      ├── confidence < 0.8         → Escalation Response
      ├── needs_db + email found   → Supabase Lookup → DB Result
      ├── needs_db + no email      → Ask Email Response
      └── KB / Greeting            → KB or Greeting Response
      ↓
[Log to Supabase] — every query logged regardless of route
      ↓
[Send Response] — returned to chat UI
```

---

## Supabase Schema

### `authors` table
```sql
create table authors (
  id uuid default gen_random_uuid() primary key,
  email text unique not null,
  name text,
  book_title text,
  final_submission_date date,
  book_live_date date,
  royalty_status text,
  isbn text,
  add_on_services text[],
  author_copy_status text,
  phone text
);
```

### `query_logs` table
```sql
create table query_logs (
  id uuid default gen_random_uuid() primary key,
  query text,
  email_used text,
  intent text,
  response text,
  confidence float,
  escalated boolean default false,
  route text,
  created_at timestamptz default now()
);
```

### Mocked Author Data (6 records)
| Email | Book | Status | Add-ons |
|---|---|---|---|
| sara.johnson@xyz.com | Petals of Silence | Live | Global Distribution |
| rahul.mehta@gmail.com | Echoes of Tomorrow | In Review | None |
| priya.sharma@outlook.com | Rivers of Light | Live | Bestseller Package, PR |
| amit.verma@yahoo.com | Voices in the Wind | In Review | Award Nomination |
| nisha.kapoor@gmail.com | Midnight Bloom | Live | Global Distribution, Bestseller |
| dev.anand@proton.me | The Last Verse | Live | None |

---

## Key Features

### Natural Language Understanding
The AI Agent classifies 8 intents: `greeting`, `book_live_status`, `royalty_status`, `author_copy`, `add_on_status`, `dashboard_access`, `general_faq`, `unknown`.

### Conversation Memory
Uses n8n's Window Buffer Memory (12-message context window, session-keyed). The bot remembers what was said earlier — if it asked for your email and you reply with it in the next message, it correctly picks it up without asking again.

### Confidence-Based Escalation
Every response includes a confidence score. If confidence < 0.8, the query is routed to escalation and the user is directed to the human support team with the ticket link.

### Knowledge Base Integration
The full BookLeaf FAQ is embedded directly in the system prompt — covering challenge details, refund policy, publishing timeline, distribution, royalties, dashboard access, copyright, and add-on services. General queries are answered without any DB call.

### Error Handling
| Scenario | Handling |
|---|---|
| AI Agent fails | AI Error Handler catches it, returns safe fallback |
| Supabase unreachable | `neverError: true` + `continueErrorOutput`, graceful fallback message |
| No author record found | DB Result Processor returns "account not found" message |
| Multiple records | Handled in DB Result Processor, asks user to verify via support |
| Gibberish / off-topic | Confidence 0.2 → escalation route |
| Greeting with no question | Confidence 0.95 → warm reply, no DB call, no escalation |

### Full Query Logging
Every interaction — regardless of route — is logged to `query_logs` with: original query, extracted email, intent, confidence score, route taken, escalation flag, and response sent.

---

## Supported Query Examples

```
"hii"                                          → Warm greeting
"How long does it take to publish my book?"    → KB answer (no email needed)
"Can I get a refund?"                          → KB answer
"Is my book live yet?"                         → Asks for email → fetches DB
"Is my book live? My email is x@y.com"         → Direct DB fetch, no email prompt
"When will I get my royalty?"                  → DB query
"Where is my author copy?"                     → DB query
"What add-ons do I have?"                      → DB query
"asdfghjkl"                                    → Escalation
"What is the meaning of life"                  → Escalation
```

---

## What I Would Improve With More Time

1. **WhatsApp + Email channel integration** — n8n has native WhatsApp Business and Gmail nodes. I'd add webhook triggers for both so the same workflow handles all channels without code changes.

2. **pgvector RAG for KB** — Instead of embedding the KB in the system prompt, I'd store it in Supabase with vector embeddings and do semantic search. This scales as the KB grows and reduces token usage per query.

3. **Instagram DM webhook** — Connect Instagram Graph API webhook to n8n so DMs trigger the same workflow.

4. **Author identity via phone lookup** — Add a secondary lookup by phone number for WhatsApp queries where email isn't available.

5. **Admin dashboard** — A simple read view on `query_logs` to monitor escalation rates, common intents, and low-confidence patterns over time.

6. **Rate limiting** — Add a check per session to prevent spam/abuse before hitting the LLM.

---

## Self-Rating

| Skill | Rating | Notes |
|---|---|---|
| Zapier / Make / n8n | 6/10 | Comfortable building complex n8n workflows; less hands-on with Zapier/Make but understand the mental model |
| LangChain / OpenAI integrations | 8/10 | Shipped Artificial Mufti (500+ users, iOS + Android) using RAG pipelines, multi-LLM integration, pgvector |
| System design & troubleshooting | 8/10 | Designed and debugged this full workflow end-to-end including memory, routing, error handling, and DB integration |

---

## Repository Structure

```
bookleaf-author-query-bot/
├── README.md
├── workflow/
│   └── BookLeaf_Author_Query_Bot_v3.json   ← import directly into n8n
├── supabase/
│   └── schema.sql                           ← run in Supabase SQL editor
└── identity-unification/
    ├── flowchart.png                        ← Excalidraw export
    └── identity-unification.md             ← design + pseudocode
```

---

## How to Run

1. Import `workflow/BookLeaf_Author_Query_Bot_v3.json` into n8n
2. Add your OpenAI API key to the OpenAI Chat Model node credentials
3. Update Supabase URL and anon key in the Supabase Lookup and Log nodes
4. Run `supabase/schema.sql` in your Supabase SQL editor
5. Activate the workflow
6. Open the Chat Trigger public URL and start querying
