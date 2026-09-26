# WhatsMail

> AI-powered email assistant that identifies what deserves your attention and delivers personalized daily email digests directly to WhatsApp.

WhatsMail is a backend-focused application designed to reduce email overload.

Instead of manually checking dozens of messages every day, users can connect their Gmail account and let WhatsMail collect, classify, prioritize, and summarize incoming emails.

The most relevant information can then be delivered as a personalized daily digest through WhatsApp.

---

## The idea

A typical inbox contains a mixture of:

* important messages;
* newsletters;
* automated notifications;
* recruitment emails;
* bills and payment reminders;
* security alerts;
* university or work communications;
* marketing content;
* messages that require a response.

WhatsMail aims to automatically identify what actually deserves the user's attention.

The core pipeline is:

```text
Gmail
  ↓
Email Collection
  ↓
Rule-based Preprocessing
  ↓
AI Classification
  ↓
Priority Scoring
  ↓
Daily Digest
  ↓
WhatsApp
```

---

## Main goals

WhatsMail is being built around four main goals:

### Email collection

Connect securely to Gmail using OAuth 2.0 and retrieve incoming messages using the Gmail API.

### Intelligent classification

Analyze emails and classify them based on:

* category;
* priority;
* required action;
* relevance;
* deadlines;
* sender;
* message context.

### Daily digest

Generate a concise summary containing only the information that deserves the user's attention.

Example:

```text
Good morning!

You received 28 emails since your last digest.

🔴 3 high priority
🟡 5 medium priority
⚪ 20 low priority

Important messages:

1. Recruitment
You received an invitation for the next interview stage.

Action required:
Confirm your availability.

2. University
A new academic notification was published.

3. Security
A security notification was detected in your account.
```

### WhatsApp delivery

Send scheduled digests directly to the user through the WhatsApp Cloud API.

---

## Architecture

The application is designed as a backend-first system.

```text
                         WhatsMail

                            Client
                              │
                              ▼
                           FastAPI
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
       Gmail API          PostgreSQL            Redis
          │                                       │
          ▼                                       ▼
     Email Pipeline                         Task Queue
          │                                       │
          ▼                                       ▼
      Rule Engine                         Celery Workers
          │                                       │
          └────────────────┐                      │
                           ▼                      ▼
                       Local LLM  ◄───────────────┘
                           │
                           ▼
                    Classification
                           │
                           ▼
                     Daily Digest
                           │
                           ▼
                 WhatsApp Cloud API
```

The architecture separates HTTP requests, business logic, persistence, background processing, and third-party integrations.

---

## Tech stack

### Backend

* Python
* FastAPI
* Pydantic
* SQLAlchemy 2
* Alembic
* httpx

### Database

* PostgreSQL

### Background processing

* Redis
* Celery
* Celery Beat

### Integrations

* Google OAuth 2.0
* Gmail API
* WhatsApp Cloud API

### Artificial Intelligence

The project is designed around an AI provider abstraction.

The initial goal is to support local inference using lightweight models while keeping the possibility of using external LLM providers as fallback.

Possible architecture:

```text
Email
  ↓
Rule Engine
  ↓
Local LLM
  ↓
Confidence Evaluation
  │
  ├── confident → store classification
  │
  └── uncertain → external LLM fallback
```

This allows WhatsMail to reduce external API usage while keeping the AI layer replaceable.

### Frontend

Planned:

* React
* TypeScript
* Tailwind CSS

### Infrastructure

* Docker
* Docker Compose
* Nginx

---

## Project structure

```text
whatsmail/
│
├── backend/
│   ├── app/
│   │   ├── api/
│   │   ├── core/
│   │   ├── models/
│   │   ├── schemas/
│   │   ├── repositories/
│   │   ├── services/
│   │   │   ├── gmail/
│   │   │   ├── classification/
│   │   │   ├── ai/
│   │   │   ├── digest/
│   │   │   └── whatsapp/
│   │   ├── workers/
│   │   └── main.py
│   │
│   ├── alembic/
│   ├── tests/
│   └── pyproject.toml
│
├── frontend/
│
├── docker-compose.yml
├── .env.example
├── README.md
└── docs/
```

### Responsibilities

```text
api/
HTTP routes and request handling.

services/
Application and business logic.

repositories/
Database access.

models/
Database entities.

schemas/
Input/output validation using Pydantic.

workers/
Background and scheduled tasks.

services/gmail/
Google OAuth and Gmail API integration.

services/classification/
Email preprocessing and classification logic.

services/ai/
LLM providers and AI abstraction.

services/digest/
Daily digest generation.

services/whatsapp/
WhatsApp Cloud API integration.
```

---

## Planned email classification

The classifier is expected to return structured information instead of unrestricted text.

Example:

```json
{
  "category": "RECRUITMENT",
  "priority": "HIGH",
  "requires_action": true,
  "action": "Confirm interview availability",
  "summary": "The recruiter invited the user to the next interview stage."
}
```

Possible categories include:

```text
RECRUITMENT
WORK
EDUCATION
FINANCE
SECURITY
PERSONAL
NEWSLETTER
MARKETING
NOTIFICATION
OTHER
```

Possible priorities:

```text
CRITICAL
HIGH
MEDIUM
LOW
```

---

## Hybrid classification

Not every email needs to be sent directly to an LLM.

WhatsMail is designed to combine deterministic rules with semantic classification.

Examples:

```text
List-Unsubscribe header
→ likely newsletter

known marketing sender
→ lower priority

security keywords
→ higher priority

payment / expiration date
→ possible action required

interview / recruitment keywords
→ recruitment category
```

The AI layer can then handle messages where context and semantics are necessary.

This approach reduces:

* unnecessary LLM calls;
* processing cost;
* latency;
* resource consumption.

---

## Background processing

Long-running operations should not block HTTP requests.

WhatsMail will use background workers for tasks such as:

```text
fetch emails
classify messages
generate digests
send WhatsApp notifications
retry failed integrations
```

Example:

```text
Celery Beat
     │
     ▼
Schedule daily digest
     │
     ▼
Redis Queue
     │
     ▼
Celery Worker
     │
     ├── Gmail
     ├── Classification
     ├── AI
     └── PostgreSQL
             │
             ▼
         Digest ready
             │
             ▼
        WhatsApp API
```

---

## Privacy and security

Email content can contain highly sensitive information.

Security and privacy are therefore treated as core architectural concerns.

Planned measures include:

* Google OAuth 2.0 instead of storing Gmail passwords;
* minimum required Gmail permissions;
* encrypted OAuth tokens;
* environment-based secrets;
* HTTPS in production;
* restricted internal services;
* no publicly exposed LLM endpoint;
* minimum necessary email storage;
* data deletion support;
* logging without sensitive message content.

The local LLM service should only be accessible through the private application network.

```text
Internet
   │
   ▼
Nginx
   │
   ▼
FastAPI
   │
   ├── PostgreSQL
   ├── Redis
   └── Local LLM

Internal services are not exposed publicly.
```

---

## Roadmap

### Phase 1 — Foundation

* [x] Create repository
* [x] Define initial project structure
* [ ] Configure FastAPI
* [ ] Configure PostgreSQL
* [ ] Configure SQLAlchemy
* [ ] Configure Alembic
* [ ] Create initial database models
* [ ] Add Docker Compose environment

### Phase 2 — Gmail integration

* [ ] Configure Google OAuth 2.0
* [ ] Connect Gmail account
* [ ] Retrieve latest emails
* [ ] Normalize email data
* [ ] Persist email metadata

Initial milestone:

```http
GET /api/v1/emails?limit=10
```

Expected response:

```json
[
  {
    "sender": "company@example.com",
    "subject": "Interview invitation",
    "received_at": "2026-09-26T10:32:00",
    "snippet": "Hello..."
  }
]
```

### Phase 3 — Classification

* [ ] Implement rule-based classifier
* [ ] Define categories
* [ ] Define priority scoring
* [ ] Create structured AI classification schema
* [ ] Integrate local LLM
* [ ] Evaluate classification quality
* [ ] Add optional cloud LLM fallback

### Phase 4 — Daily digest

* [ ] Group emails by priority
* [ ] Detect emails requiring action
* [ ] Extract deadlines when possible
* [ ] Generate daily digest
* [ ] Store generated digests

### Phase 5 — Background processing

* [ ] Add Redis
* [ ] Add Celery
* [ ] Add Celery workers
* [ ] Add retry strategies
* [ ] Add idempotent jobs
* [ ] Add Celery Beat

### Phase 6 — WhatsApp

* [ ] Configure WhatsApp Cloud API
* [ ] Create approved message template
* [ ] Send test notifications
* [ ] Send scheduled daily digest
* [ ] Track delivery status
* [ ] Handle delivery failures

### Phase 7 — Dashboard

* [ ] React application
* [ ] Authentication
* [ ] Inbox overview
* [ ] Email classifications
* [ ] Digest history
* [ ] Notification preferences
* [ ] AI classification feedback

---

## Future ideas

Possible future improvements include:

* multiple Gmail accounts;
* Outlook integration;
* Telegram notifications;
* Discord notifications;
* custom webhooks;
* n8n / Node-RED integrations;
* user feedback for classifications;
* personalized priority rules;
* semantic email search;
* phishing detection;
* email thread summarization;
* attachment analysis;
* automatic deadline extraction;
* suggested email responses;
* calendar event detection;
* multilingual summaries;
* configurable digest schedules.

---

## Current status

WhatsMail is currently under active development.

The initial repository and project structure have been created.

The first development milestone is:

> Connect a Gmail account through OAuth 2.0 and retrieve the latest emails through the WhatsMail API.

---

## Why this project?

WhatsMail is also an engineering study focused on backend concepts such as:

* REST APIs;
* OAuth 2.0;
* external API integrations;
* asynchronous I/O;
* background jobs;
* task queues;
* distributed workers;
* scheduling;
* retry strategies;
* idempotency;
* structured LLM outputs;
* local AI inference;
* event-driven processing;
* database modeling;
* observability;
* application security.

The goal is not only to build an AI email summarizer, but to design a reliable backend system around it.

---

## License

This project is currently intended for educational and portfolio purposes.

License information will be added as the project evolves.
