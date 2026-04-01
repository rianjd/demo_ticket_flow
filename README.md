# TicketFlow

TicketFlow is a production-style AI support platform that combines conversational triage, knowledge-base retrieval, human escalation, and operational observability in one full-stack system.

This repository demonstrates end-to-end product delivery: backend APIs, LLM orchestration, vector search, async workers, admin curation workflow, metrics, and a polished frontend experience.

## Recruiter Snapshot

- Full-stack product with real business workflow (L1 automation + human handoff)
- LLM orchestration with explicit action routing and guardrails
- Hybrid retrieval (semantic + lexical) over PostgreSQL + pgvector
- Async embedding pipeline with retries and operational endpoints
- Human-in-the-loop knowledge curation (support -> refined suggestion -> admin approval -> KB)
- Observability-first backend with Prometheus metrics and request-level tracing
- Multi-language chat behavior (PT/EN/ES detection)
- Containerized local environment with frontend, backend, DB, and worker

## What Makes This Project Strong

### 1. Product Thinking, Not Just API Endpoints

TicketFlow models a realistic support lifecycle:

1. User starts a chat session
2. AI decides next action (clarify/respond/use KB/escalate/block)
3. If needed, a support ticket is created automatically
4. Human support sends a response
5. AI refines that response into reusable knowledge
6. Admin approves/rejects KB suggestion
7. Approved content becomes retrievable in future sessions

This closes the loop between live support operations and continuously improving AI responses.

### 2. AI Engineering with Guardrails

The chat orchestrator does not blindly answer. It uses explicit action classes:

- block
- respond
- clarify
- respond_kb
- escalate_ticket
- resolve_confirmation

Hard constraints prevent low-confidence KB responses from leaking to users. Confidence thresholds are configurable and enforced before `respond_kb` is allowed.

### 3. Retrieval That Works in Real Data

KB search is hybrid:

- Semantic similarity via embeddings on `problem_summary_vector`
- Lexical relevance via substring, fuzzy matching, and token overlap
- Usage-based reranking (`usage_count`) for successful recurring answers

This balances precision and recall under noisy user input.

### 4. Operational Reliability

Embedding generation is offloaded to an async worker:

- Queue table (`embedding_jobs`)
- Status transitions: pending -> processing -> done/failed
- Retry policy with attempt caps
- Admin endpoints to inspect and retry failed jobs

The main API remains responsive while vector updates happen asynchronously.

### 5. Observability by Default

Backend includes middleware that emits:

- Request counters by method/path/status
- Request latency histogram
- Request ID propagation (`x-request-id`)
- Embedding provider usage metrics (`remote/local/none`)

This is the foundation expected in production environments.

## Architecture

```text
React + TypeScript + Vite Frontend
                |
                v
FastAPI Backend (Auth, Chat, Tickets, Admin, Metrics)
                |
                v
PostgreSQL + pgvector
  - chat_logs
  - tickets
  - suggested_solutions
  - knowledge_base
  - embedding_jobs
                |
                v
Embedding Worker (async vector processing + retries)
```

## Core Features

### Chat and Orchestration

- Session-based chat endpoint
- Multi-language detection (PT/EN/ES)
- LLM decision stage for next action
- KB retrieval with confidence/debug metadata
- Automatic escalation to ticket when needed

### Ticket and Support Flow

- Ticket creation and status handling
- Support response capture
- AI refinement of support text
- Extraction of `problem_pattern`, `problem_summary`, and example utterances

### Admin and Knowledge Operations

- List pending suggestions
- Approve or reject suggestions
- Block low-quality placeholder entries from entering KB
- List and retry embedding jobs

### Frontend Product Surface

- Marketing landing with product architecture and live demo simulation
- Auth flow (register/login)
- User chat interface
- Tickets dashboard
- Admin dashboard
- Debug/demo metadata badges on AI responses
- Light/dark mode and PT/EN language switch

## Tech Stack

### Backend

- Python
- FastAPI
- SQLAlchemy
- Alembic
- PostgreSQL
- pgvector
- Groq SDK
- Cohere SDK
- Prometheus client
- passlib (argon2)
- python-jose

### Frontend

- React 19
- TypeScript
- Vite
- lucide-react

### Infra

- Docker
- Docker Compose

## Key API Areas

- `POST /chat/` and `GET /chat/{session_id}`
- `POST /auth/register`, `POST /auth/login`
- `POST /tickets/`, `GET /tickets/`, `PATCH /tickets/{ticket_id}`
- `GET /admin/suggestions/pending`
- `POST /admin/suggestions/{id}/approve`
- `POST /admin/suggestions/{id}/reject`
- `POST /admin/tickets/{ticket_id}/support-response`
- `GET /admin/embedding-jobs`
- `POST /admin/embedding-jobs/retry`
- `GET /metrics`

## Environment Configuration

Use `.env.example` as base.

### LLM

- `LLM_PROVIDER`
- `LLM_API_KEY`
- `LLM_BASE_URL`
- `LLM_TIMEOUT_SECONDS`
- `LLM_MODEL`
- `LLM_MODEL_CHAT`
- `LLM_MODEL_DECISION`
- `LLM_MODEL_TICKET`
- `LLM_MODEL_ADMIN`

### Embeddings

- `EMBEDDING_PROVIDER` (`local | remote | auto | cohere`)
- `EMBEDDING_API_KEY`
- `COHERE_API_KEY`
- `EMBEDDING_MODEL`
- `EMBEDDING_MODELS`
- `COHERE_EMBEDDING_INPUT_TYPE`
- `LOCAL_EMBEDDING_FALLBACK`

### Retrieval Guardrails

- `KB_FORCE_CONFIDENCE`
- `KB_MIN_THRESHOLD_WITH_SEMANTIC`
- `KB_MIN_THRESHOLD_NO_SEMANTIC`
- `CLARIFICATION_ESCALATION_THRESHOLD`

## Quick Start

### 1. Configure environment

```bash
cp .env.example .env
```

Fill in keys (at minimum, one LLM provider key and optionally Cohere for remote embeddings).

### 2. Run everything with Docker Compose

```bash
docker compose up --build
```

Services:

- Frontend: `http://localhost:5173`
- Backend: `http://localhost:8000`
- Metrics: `http://localhost:8000/metrics`
- Postgres: `localhost:5432`

### 3. Apply migrations if needed

```bash
docker compose exec backend alembic upgrade head
```

## Local Testing

Embedding service tests:

```bash
python -m pytest backend/tests/test_embedding_service.py -v --tb=short
```

Skip integration scenarios:

```bash
python -m pytest backend/tests/test_embedding_service.py -m "not integration" -v
```

## Repository Structure

```text
backend/
  app/
    api/
    core/
    db/
    services/
    workers/
  alembic/
  tests/
frontend/
docker-compose.yml
.env.example
```

## Engineering Highlights for Interviews

If you are evaluating this project for hiring, these are the strongest discussion points:

1. Orchestrated AI behavior using explicit action classes instead of a single prompt-only chatbot
2. Hybrid retrieval strategy with confidence gating and defensive fallbacks
3. Async vectorization architecture that decouples ingestion from user response latency
4. Human-in-the-loop knowledge lifecycle with quality controls before KB promotion
5. Production-style observability and runtime diagnostics
6. Full-stack delivery with cohesive UX and bilingual interface support

## Future Evolution

- CI/CD pipeline with lint, test, and image publish stages
- End-to-end integration tests for full user/admin flow
- Analytics dashboard (AI resolution rate, escalation rate, KB hit ratio)
- Role-based authorization hardening for admin operations
- Rate limiting and abuse protection per session/user

## License

Educational and technical demonstration use.
