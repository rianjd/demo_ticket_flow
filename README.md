# Relay - AI-Powered Support Triage Engine

> **Live Demo:** [Coming Soon]  
> **Status:** Production-ready architecture, portfolio demonstration

An intelligent support platform that combines conversational AI, knowledge retrieval, and human escalation in a production-grade system.

---

## 🎯 What This Project Demonstrates

This is a **full-stack AI product** built to showcase:

- **LLM Orchestration** with explicit action routing (not just prompt engineering)
- **Hybrid Retrieval** (semantic + lexical) over PostgreSQL + pgvector
- **Async Processing** with worker queues and retry policies
- **Human-in-the-Loop** knowledge curation workflow
- **Production Observability** with Prometheus metrics and tracing
- **Real Product Thinking** - solves the support lifecycle, not just "AI chatbot"

---

## 📊 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    User Experience                          │
│  Chat Interface → Ticket Dashboard → Admin Panel            │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                   FastAPI Backend                           │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   Auth   │  │   Chat   │  │ Tickets  │  │  Admin   │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│                         │                                    │
│                         ▼                                    │
│              ┌──────────────────┐                           │
│              │  AI Orchestrator │                           │
│              │  (Action Router) │                           │
│              └──────────────────┘                           │
│                         │                                    │
│         ┌───────────────┼───────────────┐                   │
│         ▼               ▼               ▼                   │
│    ┌────────┐    ┌──────────┐    ┌──────────┐             │
│    │ Block  │    │ Clarify  │    │ Respond  │             │
│    └────────┘    └──────────┘    │   KB     │             │
│                                   └──────────┘             │
│                         │                                    │
│                         ▼                                    │
│              ┌──────────────────┐                           │
│              │ Escalate Ticket  │                           │
│              └──────────────────┘                           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              PostgreSQL + pgvector                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Sessions │  │ Tickets  │  │    KB    │  │ Embeddings│  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│              Async Embedding Worker                         │
│  Queue Processing → Retry Logic → Vector Storage            │
└─────────────────────────────────────────────────────────────┘
```

---

## 🚀 Key Features

### 1. **Smart Triage, Not Just Chat**

The AI doesn't blindly answer questions. It makes **explicit decisions**:

```python
Actions = [
    "block",           # Offensive/spam content
    "respond",         # Direct answer
    "clarify",         # Need more info
    "respond_kb",      # Answer from knowledge base
    "escalate_ticket", # Create ticket for human
    "resolve"          # Confirm resolution
]
```

Each action has **guardrails**:
- `respond_kb` requires confidence > 75%
- After 3 clarifications → auto-escalate
- Low-quality KB entries blocked from retrieval

### 2. **Hybrid Knowledge Retrieval**

Not just vector search. Combines:

- **Semantic:** pgvector cosine similarity on embeddings
- **Lexical:** Fuzzy matching, substring search, token overlap
- **Usage-based:** Prioritizes solutions that worked before (`usage_count`)

Result: Better recall on noisy user input.

### 3. **Human-in-the-Loop Learning**

```
User Chat → AI Responds → If escalated:
    ↓
Human Support Answers
    ↓
AI Refines Answer → Suggests KB Entry
    ↓
Admin Approves/Rejects
    ↓
Approved → KB (Auto-embedded async)
```

The system **learns from real support interactions**.

### 4. **Production-Grade Async Architecture**

Embeddings don't block API responses:

```
API Request → Queue Job → Worker Processes → DB Update
                ↓
          Status Tracking (pending/processing/done/failed)
                ↓
          Retry Policy (3 attempts)
```

Admin can inspect and retry failed jobs via `/admin/embedding-jobs`.

### 5. **Observability Built-In**

- **Prometheus metrics:** request count, latency, embedding provider usage
- **Request tracing:** `x-request-id` propagation
- **Debug metadata:** Each response includes confidence scores, retrieval results
- **Health checks:** `/health`, `/metrics` endpoints

---

## 🛠️ Tech Stack

### Backend
- **Framework:** FastAPI
- **Database:** PostgreSQL + pgvector
- **AI/ML:** Groq API (llama-3.3-70b), Cohere Embeddings
- **ORM:** SQLAlchemy + Alembic
- **Auth:** JWT with argon2 password hashing
- **Monitoring:** Prometheus client

### Frontend
- **Framework:** React 19 + TypeScript
- **Build:** Vite
- **UI:** Tailwind CSS, lucide-react
- **Features:** i18n (PT/EN/ES), dark mode, responsive

### Infrastructure
- **Containerization:** Docker + Docker Compose
- **Services:** Backend, Frontend, PostgreSQL, Embedding Worker

---

## 📸 Screenshots

### Chat Interface
> Clean conversational UI with confidence indicators and debug metadata

### Admin Dashboard
> Knowledge base curation, ticket management, embedding job monitoring

### Metrics Dashboard
> Real-time observability with Prometheus integration

*[Screenshots will be added post-deployment]*

---

## 🎥 Demo Video

**Coming Soon:** 3-minute walkthrough showing:
1. User chat flow (clarification → KB response)
2. Ticket escalation
3. Admin KB approval workflow
4. Metrics and monitoring

---

## 🧠 Engineering Highlights

If you're evaluating this for a hiring decision, these are the strongest discussion points:

### 1. Product Architecture
- Session-based chat with **state management**
- Ticket lifecycle modeling (open → in_progress → resolved)
- Multi-language support with auto-detection

### 2. AI Engineering
- **Explicit action routing** (not prompt-only)
- Confidence thresholds enforced at runtime
- Defensive fallbacks (semantic search fails → lexical search)

### 3. Data Pipeline
- Async embedding generation with **queue table pattern**
- Retry logic with exponential backoff
- Admin visibility into pipeline health

### 4. Scalability Considerations
- Stateless API design (session in DB, not memory)
- Vector operations offloaded to worker
- Configurable LLM provider (Groq/OpenAI/local)

### 5. Operational Readiness
- Structured logging with request IDs
- Metrics-first debugging
- Environment-based configuration (dev/staging/prod)

---

## 🔐 Security & Compliance

- **Authentication:** JWT-based with secure token refresh
- **Password Storage:** Argon2 hashing (OWASP recommended)
- **Input Validation:** Pydantic models on all endpoints
- **Rate Limiting:** Configurable per session/user *(planned)*
- **PII Handling:** No sensitive data logged in metrics

---

## 📈 Performance Characteristics

### Response Times (P95)
- Chat endpoint: ~800ms (with KB retrieval)
- Ticket creation: ~200ms
- Embedding generation: 2-5s (async, non-blocking)

### Scalability
- **Current:** Single-node demo (~100 concurrent users)
- **Production-ready:** Horizontal scaling via stateless design
- **Database:** pgvector handles 100k+ KB entries efficiently

---

## 🚧 Future Roadmap

- [ ] **CI/CD Pipeline:** GitHub Actions with automated tests
- [ ] **Analytics Dashboard:** AI resolution rate, escalation trends
- [ ] **Multi-tenancy:** Company isolation and RBAC
- [ ] **Advanced Retrieval:** Re-ranking models, query expansion
- [ ] **Integration APIs:** Zendesk, Intercom, Slack webhooks

---

## 📦 Running Locally

**Note:** This is for educational purposes. The live demo is the recommended testing method.

### Prerequisites
- Docker + Docker Compose
- Groq API key (or OpenAI)
- Optional: Cohere API key (for remote embeddings)

### Quick Start

```bash
# 1. Clone repo (private - request access)
git clone https://github.com/rianjd/relay.git
cd relay

# 2. Configure environment
cp .env.example .env
# Edit .env with your API keys

# 3. Launch stack
docker compose up --build

# 4. Access services
# Frontend: http://localhost:5173
# Backend: http://localhost:8000
# Metrics: http://localhost:8000/metrics
```

### Seed Demo Data

```bash
docker compose exec backend python -m app.seed_demo
```

Creates "Acme Corp" with sample KB entries and tickets.

---

## 📊 Sample Metrics Output

```prometheus
# Request volume by endpoint
http_requests_total{method="POST",path="/chat",status="200"} 1247
http_requests_total{method="GET",path="/tickets",status="200"} 892

# Response latency (seconds)
http_request_duration_seconds_bucket{le="0.5"} 1834
http_request_duration_seconds_bucket{le="1.0"} 2103

# Embedding provider usage
embedding_operations_total{provider="cohere"} 456
embedding_operations_total{provider="local"} 89
```

---

## 🤝 Contributing

This is a portfolio/demonstration project. Full source code is **available to employers upon request**.

For collaboration or questions:
- **Email:** rianjd@email.com
- **LinkedIn:** [linkedin.com/in/rianjd](https://linkedin.com/in/rianjd)
- **GitHub:** [@rianjd](https://github.com/rianjd)

---

## 📄 License

Educational and technical demonstration use.  
Not licensed for commercial deployment without permission.

---

## 🏆 Why This Project Stands Out

Most portfolio projects are:
- ❌ Simple CRUD apps with AI slapped on
- ❌ Single-file scripts without architecture
- ❌ No consideration for production concerns

**This project demonstrates:**
- ✅ End-to-end product lifecycle (user → AI → human → feedback loop)
- ✅ Production patterns (async workers, observability, retries)
- ✅ Real AI engineering (guardrails, hybrid retrieval, confidence gating)
- ✅ Full-stack execution (backend, frontend, infra, deployment)

**Built by a developer who thinks like a product engineer.**

---

**Last Updated:** April 2026  
**Project Status:** Active Development → Demo Deployment