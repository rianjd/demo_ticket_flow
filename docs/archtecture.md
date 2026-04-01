# Relay - Technical Architecture

## System Overview

Relay is a production-grade AI support platform designed around three core principles:

1. **AI as L1 Triage** - Not a replacement for human support, but an intelligent filter
2. **Human-in-the-Loop Learning** - Continuously improves from real support interactions
3. **Observability-First** - Built to be debugged, monitored, and optimized

---

## Component Architecture

### High-Level Flow

```
┌──────────────────────────────────────────────────────────────────┐
│                         User Layer                               │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                 │
│  │   Chat     │  │  Tickets   │  │   Admin    │                 │
│  │ Interface  │  │ Dashboard  │  │   Panel    │                 │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                 │
└────────┼───────────────┼───────────────┼──────────────────────────┘
         │               │               │
         │               │               │
         ▼               ▼               ▼
┌──────────────────────────────────────────────────────────────────┐
│                      API Gateway Layer                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              FastAPI Application                        │    │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐     │    │
│  │  │ Auth │  │ Chat │  │Ticket│  │Admin │  │Metric│     │    │
│  │  │  API │  │  API │  │  API │  │  API │  │  API │     │    │
│  │  └──┬───┘  └───┬──┘  └───┬──┘  └───┬──┘  └──┬───┘     │    │
│  └─────┼──────────┼─────────┼─────────┼────────┼──────────┘    │
│        │          │         │         │        │                │
│        │          ▼         │         │        │                │
│        │   ┌──────────────┐ │         │        │                │
│        │   │AI Orchestrator│ │         │        │                │
│        │   │(Action Router)│ │         │        │                │
│        │   └──────┬───────┘ │         │        │                │
│        │          │         │         │        │                │
└────────┼──────────┼─────────┼─────────┼────────┼────────────────┘
         │          │         │         │        │
         ▼          ▼         ▼         ▼        ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Data Layer                                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │         PostgreSQL + pgvector                          │     │
│  │                                                         │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐│     │
│  │  │  Users   │  │ Sessions │  │ Tickets  │  │  Chat  ││     │
│  │  │          │  │          │  │          │  │  Logs  ││     │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────┘│     │
│  │                                                         │     │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐            │     │
│  │  │Knowledge │  │Suggested │  │Embedding │            │     │
│  │  │   Base   │  │Solutions │  │   Jobs   │            │     │
│  │  └──────────┘  └──────────┘  └──────────┘            │     │
│  └────────────────────────────────────────────────────────┘     │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│                    Worker Layer                                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │         Async Embedding Worker                         │     │
│  │                                                         │     │
│  │  Queue Poll → Process → Generate Vector → Store        │     │
│  │       ↓                                                 │     │
│  │  Retry Logic (3 attempts max)                          │     │
│  └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

---

## AI Orchestration Flow

### Decision Pipeline

```
User Message
    │
    ▼
┌─────────────────────────────────┐
│  Language Detection             │
│  (PT/EN/ES auto-detect)         │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Context Assembly               │
│  - Session history              │
│  - User profile                 │
│  - Company config               │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  LLM Decision Stage             │
│  (Groq llama-3.3-70b)           │
│                                 │
│  Returns: {                     │
│    action: "respond_kb",        │
│    reasoning: "...",            │
│    confidence: 0.87             │
│  }                              │
└────────────┬────────────────────┘
             │
             ▼
     ┌───────┴───────┐
     │ Action Router │
     └───────┬───────┘
             │
    ┌────────┼────────┐
    │        │        │
    ▼        ▼        ▼
┌────────┐┌────────┐┌────────┐
│ Block  ││Clarify ││Respond │
└────────┘└────────┘└────────┘
    │        │        │
    ▼        ▼        ▼
┌────────┐┌────────┐┌────────────┐
│ Log &  ││Ask for ││ KB Lookup  │
│ End    ││Details ││ (Hybrid)   │
└────────┘└────────┘└──────┬─────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Confidence   │
                    │ Check        │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              │                         │
              ▼                         ▼
      ┌──────────────┐         ┌──────────────┐
      │ > 75%        │         │ < 75%        │
      │ Respond KB   │         │ Escalate     │
      └──────────────┘         └──────────────┘
```

---

## Knowledge Base Retrieval

### Hybrid Search Strategy

```
User Query: "How to reset my password?"
    │
    ▼
┌─────────────────────────────────────────────────────┐
│  Step 1: Semantic Search (pgvector)                 │
│                                                      │
│  SELECT *                                            │
│  FROM knowledge_base                                 │
│  ORDER BY problem_summary_vector <=> query_vector   │
│  LIMIT 10                                            │
│                                                      │
│  Returns: Cosine similarity scores                  │
└────────────┬────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────┐
│  Step 2: Lexical Boost                              │
│                                                      │
│  - Substring match in title (+0.2)                  │
│  - Fuzzy match in problem_pattern (+0.15)           │
│  - Token overlap in example_utterances (+0.1)       │
│                                                      │
│  Adjusted Score = semantic * 0.7 + lexical * 0.3    │
└────────────┬────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────┐
│  Step 3: Usage-Based Reranking                      │
│                                                      │
│  If usage_count > 10:                               │
│    Final Score = Adjusted Score * 1.1               │
│                                                      │
│  Sort by Final Score DESC                           │
└────────────┬────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────┐
│  Step 4: Confidence Gating                          │
│                                                      │
│  If top_score >= 0.75:                              │
│    Return KB entry                                  │
│  Else:                                              │
│    Escalate to clarification or ticket             │
└─────────────────────────────────────────────────────┘
```

### Why Hybrid?

**Problem:** Pure semantic search fails on:
- Acronyms ("VPN" vs "Virtual Private Network")
- Specific error codes ("ERR_SSL_PROTOCOL_ERROR")
- Product names ("TicketFlow Premium Plan")

**Solution:** Lexical matching catches exact substring/token matches that embeddings miss.

---

## Embedding Pipeline

### Async Processing Architecture

```
KB Entry Created/Updated
    │
    ▼
┌─────────────────────────────────┐
│  Insert into embedding_jobs     │
│                                 │
│  {                              │
│    kb_entry_id: 123,            │
│    status: "pending",           │
│    attempt: 0                   │
│  }                              │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Worker Polls Queue             │
│  (Every 5 seconds)              │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Status → "processing"          │
│  Fetch KB entry text            │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Generate Embedding             │
│  (Cohere API or local model)    │
└────────────┬────────────────────┘
             │
      ┌──────┴──────┐
      │             │
      ▼             ▼
 ┌─────────┐   ┌─────────┐
 │ Success │   │  Error  │
 └────┬────┘   └────┬────┘
      │             │
      ▼             ▼
 ┌─────────────┐ ┌────────────────┐
 │ Update KB   │ │ attempt++      │
 │ with vector │ │ If attempt < 3 │
 │             │ │   retry_after  │
 │ status:     │ │ Else:          │
 │  "done"     │ │   status:      │
 └─────────────┘ │   "failed"     │
                 └────────────────┘
```

### Retry Strategy

| Attempt | Delay    | Action                           |
|---------|----------|----------------------------------|
| 1       | 0s       | Immediate processing             |
| 2       | 60s      | Retry after 1 minute             |
| 3       | 300s     | Retry after 5 minutes            |
| 4+      | -        | Mark as failed, admin visibility |

---

## Human-in-the-Loop Learning

### Knowledge Refinement Flow

```
1. User Chat Escalates → Ticket Created
    │
    ▼
2. Human Support Answers
    │
    ├─ "Here's how to reset: Go to settings..."
    │
    ▼
3. AI Refines Answer
    │
    ├─ Extracts:
    │   - problem_pattern: "password reset request"
    │   - problem_summary: "User cannot access account"
    │   - solution_text: "Navigate to Settings > Security..."
    │   - example_utterances: ["reset password", "forgot password"]
    │
    ▼
4. Creates Suggested Solution (status: "pending")
    │
    ▼
5. Admin Reviews
    │
    ├─ Option A: Approve → KB Entry Created
    │                     → Embedding Job Queued
    │                     → Future chats use this
    │
    └─ Option B: Reject → Discarded
                        → No KB pollution
```

### Quality Gates

Before entering KB:
- ✅ Human-verified solution (came from support response)
- ✅ Admin approval required
- ✅ Placeholder text detection ("will get back to you") → auto-reject
- ✅ Minimum content length (> 50 chars)
- ✅ No PII in example utterances

---

## Observability Stack

### Metrics Exposed

```prometheus
# Request volume
http_requests_total{method, path, status}

# Latency distribution
http_request_duration_seconds{method, path}

# Embedding operations
embedding_operations_total{provider, status}
embedding_operation_duration_seconds{provider}

# KB retrieval
kb_lookups_total{result_type}  # hit/miss/low_confidence
kb_lookup_confidence{quantile}

# Business metrics
tickets_created_total{category}
escalations_total{reason}
```

### Request Tracing

Every request generates:
```
x-request-id: 7a3f2c9b-1e8d-4a5c-9f3e-2d1a6b8c4e7f
```

Logged at:
- API entry
- LLM call start/end
- DB query start/end
- Worker job pickup

Enables full request path reconstruction for debugging.

---

## Configuration System

### Environment-Based Config

```python
# .env
ENVIRONMENT=production  # dev/staging/production

# LLM Configuration
LLM_PROVIDER=groq
LLM_MODEL_DECISION=llama-3.3-70b-versatile
LLM_MODEL_CHAT=llama-3.3-70b-versatile
LLM_TIMEOUT_SECONDS=30

# Embedding Configuration
EMBEDDING_PROVIDER=cohere  # local/cohere/auto
EMBEDDING_MODEL=embed-english-v3.0
LOCAL_EMBEDDING_FALLBACK=true

# Retrieval Guardrails
KB_MIN_THRESHOLD_WITH_SEMANTIC=0.75
KB_MIN_THRESHOLD_NO_SEMANTIC=0.60
CLARIFICATION_ESCALATION_THRESHOLD=3

# Security
JWT_SECRET_KEY=<random-256-bit-key>
JWT_ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=30
```

### Dynamic Company Config

Each company has JSON config:
```json
{
  "company_profile": {
    "name": "Acme Corp",
    "industry": "SaaS",
    "timezone": "America/Sao_Paulo"
  },
  "ticket_categories": [
    {
      "id": "tech_issue",
      "name": "Technical Problem",
      "sla_hours": 4,
      "auto_escalate_keywords": ["production down", "data loss"]
    }
  ],
  "agent_behavior": {
    "tone": "professional",
    "max_clarifications": 3,
    "confidence_threshold": 0.75
  }
}
```

Loaded at runtime, no code changes needed.

---

## Database Schema

### Core Tables

```sql
-- Authentication
users (
  id, email, hashed_password, role, created_at
)

-- Chat Sessions
chat_sessions (
  id, user_id, created_at, updated_at, language
)

chat_logs (
  id, session_id, role, content, action, 
  confidence, created_at
)

-- Ticketing
tickets (
  id, session_id, category, status, priority,
  title, description, created_at
)

ticket_messages (
  id, ticket_id, sender_type, content, created_at
)

-- Knowledge Base
knowledge_base (
  id, title, problem_pattern, problem_summary,
  solution_text, example_utterances, usage_count,
  problem_summary_vector vector(1024),
  created_at, updated_at
)

suggested_solutions (
  id, ticket_id, problem_pattern, solution_text,
  status, reviewed_by, reviewed_at
)

-- Async Processing
embedding_jobs (
  id, kb_entry_id, status, attempt, 
  error_message, retry_after, created_at
)
```

### Key Indexes

```sql
-- Fast session lookup
CREATE INDEX idx_sessions_user ON chat_sessions(user_id);

-- Vector search performance
CREATE INDEX idx_kb_vector ON knowledge_base 
  USING ivfflat (problem_summary_vector vector_cosine_ops);

-- Embedding job queue
CREATE INDEX idx_jobs_pending ON embedding_jobs(status, retry_after)
  WHERE status IN ('pending', 'failed');
```

---

## Deployment Architecture

### Docker Compose Stack

```yaml
services:
  frontend:
    build: ./frontend
    ports: ["5173:5173"]
    environment:
      - VITE_API_URL=http://backend:8000
  
  backend:
    build: ./backend
    ports: ["8000:8000"]
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/relay
      - GROQ_API_KEY=${GROQ_API_KEY}
    depends_on:
      - db
  
  worker:
    build: ./backend
    command: python -m app.workers.embedding_worker
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/relay
      - COHERE_API_KEY=${COHERE_API_KEY}
    depends_on:
      - db
  
  db:
    image: pgvector/pgvector:pg16
    ports: ["5432:5432"]
    volumes:
      - postgres_data:/var/lib/postgresql/data
```

### Production Considerations

**Scaling:**
- Backend: Horizontal (stateless)
- Worker: Horizontal (queue-based)
- Database: Vertical + read replicas

**High Availability:**
- Load balancer → N backend instances
- Database failover (Patroni/Stolon)
- Worker redundancy (multiple instances)

**Monitoring:**
- Prometheus scraping `/metrics`
- Grafana dashboards
- Alert manager for SLA breaches

---

## Security Model

### Authentication Flow

```
1. User Registration
   ├─ Email + Password
   ├─ Password hashed (argon2)
   └─ User record created

2. Login
   ├─ Credentials verified
   ├─ JWT access token issued (30min TTL)
   ├─ JWT refresh token issued (7d TTL)
   └─ Tokens returned to client

3. API Requests
   ├─ Authorization: Bearer <access_token>
   ├─ Token validation
   ├─ User ID extracted from claims
   └─ Request processed
```

### Role-Based Access

| Role       | Permissions                                     |
|------------|-------------------------------------------------|
| `user`     | Chat, view own tickets                          |
| `support`  | View all tickets, respond, create suggestions   |
| `admin`    | Approve/reject KB, manage users, view metrics   |

### Input Validation

All endpoints use Pydantic models:
```python
class ChatRequest(BaseModel):
    message: str = Field(min_length=1, max_length=2000)
    session_id: Optional[UUID] = None
    
    @validator('message')
    def sanitize_message(cls, v):
        # Strip HTML tags, prevent injection
        return bleach.clean(v, strip=True)
```

---

## Testing Strategy

### Unit Tests
- Service layer logic (embedding, retrieval, orchestration)
- Mock external API calls (Groq, Cohere)
- 80%+ coverage target

### Integration Tests
- API endpoint contracts
- Database operations
- Auth flow end-to-end

### Performance Tests
- Load testing with Locust
- Target: 100 concurrent users
- P95 latency < 1s for chat endpoint

---

## Cost Analysis

### Demo Deployment (Railway)

| Service      | Cost/Month |
|--------------|------------|
| Backend      | $5         |
| Worker       | $5         |
| PostgreSQL   | $10        |
| Frontend     | Free (CDN) |
| **Total**    | **$20**    |

### API Costs (10k messages/month)

| Provider     | Cost       |
|--------------|------------|
| Groq         | $0 (free tier) |
| Cohere       | ~$2        |
| **Total**    | **~$2**    |

**Grand Total:** ~$22/month for production-ready demo

---

## Comparison to Alternative Approaches

### Why Not Just Use ChatGPT API Directly?

| Aspect              | ChatGPT Alone | Relay Architecture |
|---------------------|---------------|--------------------|
| **Knowledge Retrieval** | Hallucinations | Grounded in KB |
| **Escalation Logic**    | Manual prompts | Explicit actions |
| **Learning**            | None | Human-in-loop |
| **Observability**       | Black box | Full metrics |
| **Cost Control**        | Per-token | Cached KB |

### Why Not Use Existing Helpdesk AI?

Most commercial solutions (Zendesk AI, Intercom) are:
- ❌ Black-box integrations
- ❌ No control over decision logic
- ❌ Expensive ($100-500/month)

Relay demonstrates:
- ✅ Custom orchestration
- ✅ Full code ownership
- ✅ Extensible architecture

---

**Document Version:** 1.0  
**Last Updated:** April 2026