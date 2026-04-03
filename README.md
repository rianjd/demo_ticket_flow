# Relay - AI Support Triage Engine

**Live Demo:**  Pending
**Video Demo:** https://youtu.be/dWKuSng6J1g
**Focus:** AI decision systems, retrieval, async processing

---

## What this is

Relay is not just a chatbot.

It’s a support triage system where an LLM **decides what to do**, instead of always answering:

* answer directly
* search knowledge base
* ask for clarification
* escalate to a human
* block bad input

Built to simulate how a real support system behaves under uncertainty.

---

## Why I built this

Most “AI support” demos are just:

* prompt + response
* no state
* no failure handling

I wanted to build something closer to reality:

* decisions instead of raw text generation
* fallback strategies when AI fails
* async processing where needed
* visibility into what the system is doing

---

## Core idea

Instead of:

```
User → LLM → Response
```

This project does:

```
User → LLM → Action → System executes → Response
```

---

## Actions

```python
Actions = [
    "block",
    "respond",
    "clarify",
    "respond_kb",
    "escalate_ticket",
    "resolve"
]
```

Each action has rules:

* `respond_kb` only if confidence is high
* too many clarifications → escalate
* unsafe input → blocked

---

## Architecture (simplified)

```
Frontend (React)
      ↓
FastAPI Backend
      ↓
AI Orchestrator (decision layer)
      ↓
───────────────
| KB Search   |
| Ticketing   |
| Guardrails  |
───────────────
      ↓
PostgreSQL + pgvector

Async Worker:
Queue → Embeddings → DB
```

---

## Key parts

### 1. AI Orchestrator

This is the core.

The LLM does not generate final answers directly.
It first decides:

* what action to take
* with what confidence
* using what context

This avoids:

* blind answers
* hallucinations going straight to users

---

### 2. Knowledge Retrieval

Vector search alone wasn’t enough.

Problems:

* bad results with typos / messy input
* low recall for short queries

Solution:

* semantic search (pgvector)
* lexical fallback (text matching)
* ranking boost from previous successful answers

---

### 3. Async Embeddings

Initial version:

* embeddings generated during request
* response time: ~2–3 seconds

Fix:

* moved to async worker with queue table

Flow:

```
API → insert job → worker processes → update DB
```

Result:

* chat response ~800ms
* embeddings processed in background

---

### 4. Human-in-the-loop

When AI fails:

1. ticket is created
2. human answers
3. system suggests KB entry
4. admin approves/rejects

Only approved content is used later.

---

### 5. Observability

Added early to debug behavior:

* request IDs
* response metadata (confidence, source)
* Prometheus metrics
* basic tracing

This helped catch:

* bad retrieval results
* low-confidence loops
* unnecessary escalations

---

## Tech Stack

**Backend**

* FastAPI
* PostgreSQL + pgvector
* SQLAlchemy
* JWT auth

**AI**

* LLM via API (configurable)
* embeddings (local or external)

**Frontend**

* React + TypeScript
* Tailwind

**Infra**

* Docker + Compose
* worker process for async jobs

---

## Challenges

### Slow responses (initially)

* embeddings inside request
* fixed with async worker

---

### Weak retrieval quality

* vector search alone wasn’t enough
* added lexical + usage-based ranking

---

### Hallucinated answers

* LLM answering without certainty
* fixed with:

  * confidence thresholds
  * forced escalation

---

### Infinite clarification loops

* model kept asking for more info
* added limit → auto escalate

---

## What I’d improve next

* multi-tenant support (per company)
* better ranking (re-ranking models)
* analytics (resolution rate, failure modes)
* external integrations (Zendesk, Slack)

---

## Running locally

```bash
git clone https://github.com/rianjd/-----
cd -----

cp .env.example .env

docker compose up --build
```

---

## What this project shows

* I don’t treat LLMs as magic black boxes
* I design systems around failure, not just success
* I think in terms of product behavior, not just endpoints
* I care about performance and observability early

---

## Contact

* GitHub: https://github.com/rianjd
* LinkedIn: https://linkedin.com/in/rian-junckes-dilli-68201a239/

---

**Note:** This is a portfolio project, but built with real system constraints in mind.
