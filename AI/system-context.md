# 2. System Context

## 2.1 System Position

Memoket AI sits between the Business Backend (Go/Node) and external AI providers. It receives requests from the Business Backend via REST/gRPC (authenticated with Internal API Key) and from end-user-facing endpoints via JWT Bearer.

```
Mobile App / Web Frontend
        |
        v
  Business Backend (Go/Node)
        |
        +-- REST (X-Internal-API-Key) --> AI FastAPI (:8000)
        +-- gRPC (Internal API Key)  --> AI gRPC InsightsService
        |
        v
  AI Backend
        |
        +-- LLM APIs (OpenAI, Azure OpenAI, Anthropic, Qwen)
        +-- ASR Providers (Azure Speech, AssemblyAI)
        +-- AWS S3 (audio files)
        +-- PostgreSQL + pgvector (data store)
        +-- Redis (Celery broker + stream buffer)
        +-- AWS ElastiCache Redis (notification bus)
```

## 2.2 Internal Architecture

The AI backend follows a **layered architecture with async worker offloading**:

| Layer | Components | Responsibility |
|-------|-----------|----------------|
| **Presentation** | 8 FastAPI routers, gRPC InsightsService, MCP Server | Request handling, auth, SSE streaming |
| **Auth & Middleware** | TraceIdMiddleware, APIMonitoringMiddleware, JWT/API Key | Authentication, trace propagation, monitoring |
| **Business Logic** | 10 service classes, 20+ CRUD modules | Orchestration, validation, data access |
| **Async Processing** | Celery Workers (4 queues), ThreadPoolExecutor background runner | Heavy processing offloading |
| **Algorithm** | Transcription, analysis, summary, insight RAG, function calling, voiceprint | Core AI/ML logic |
| **Streaming** | Redis Streams + SSE Generator | Real-time token-by-token delivery |
| **Data** | PostgreSQL + pgvector, Redis, AWS ElastiCache | Persistence, caching, messaging |

## 2.3 External Interfaces

### 2.3.1 Inbound (AI receives)

| Interface | Protocol | Auth | Caller | Endpoints |
|-----------|----------|------|--------|-----------|
| REST API | HTTP/1.1 | X-Internal-API-Key | Business Backend | ~60 endpoints across 8 routers |
| gRPC | HTTP/2 (gRPC) | Internal API Key | Business Backend | `StartAiInsight`, `GetAiInsightResult` |
| JWT Bearer | HTTP/1.1 | JWT (python-jose) | Mobile/Web (audio.py only) | `POST /audio/denoise/s3` |
| MCP | HTTP/SSE | OAuth | External AI agents | `create_insight`, `get_insight_result` |

### 2.3.2 Outbound (AI calls)

| Interface | Protocol | Purpose | Configuration |
|-----------|----------|---------|---------------|
| LLM APIs | HTTPS | Text generation, JSON structured output, function calling | LiteLLM Router via YAML config |
| OpenAI Embeddings | HTTPS | 1536-dim text embedding vectors | Via LiteLLM |
| Azure Speech Services | HTTPS/WebSocket | Primary ASR provider | Language-based priority |
| AssemblyAI | HTTPS | Fallback ASR provider | Automatic fallback on Azure failure |
| AWS S3 | HTTPS (boto3) | Audio file download/upload | AWS credentials |
| PostgreSQL | TCP (SQLAlchemy) | Data persistence | Connection pool (configurable) |
| Redis | TCP (redis-py) | Celery broker, stream buffer | `CELERY_BROKER_URL` env var |
| AWS ElastiCache | TCP | Notification events to Business Backend | Separate Redis instance |
| Serper API | HTTPS | Web search for Agent mode | `SERPER_API_KEY` env var |

## 2.4 Authentication Model

Two authentication mechanisms coexist:

| Mechanism | Implementation | Scope |
|-----------|---------------|-------|
| **JWT Bearer** | `get_current_tenant` dependency (python-jose) | End-user facing: `audio.py` |
| **X-Internal-API-Key** | `get_internal_api_key` dependency (header check) | Service-to-service: all other endpoints |

The Internal API Key is a shared secret between Business Backend and AI Backend. All 7 non-audio routers require this key in the `X-Internal-API-Key` header.

## 2.5 API Endpoint Summary

| Prefix | Router | Auth | Endpoints | Primary Purpose |
|--------|--------|------|-----------|-----------------|
| `/api/v1/records` | records.py | Internal API Key | ~20 | Full recording lifecycle + per-recording chat |
| `/api/v1/insights` | insights.py | Internal API Key | ~18 | Cross-recording AI insight + chat sessions |
| `/api/v1/tasks` | tasks.py | Internal API Key | 6 | Task CRUD + source tracing |
| `/api/v1/agents` | agents.py | Internal API Key | 5 | AI agent function execution |
| `/api/v1/chats` | chats.py | Internal API Key | 3 | Legacy per-recording chat |
| `/api/v1/reports` | reports.py | Internal API Key | 1 | Report retrieval |
| `/api/v1/templates` | templates.py | Internal API Key | 6 | Template discovery + recommendation |
| `/api/v1/audio` | audio.py | JWT Bearer | 1 | Audio noise reduction |

## 2.6 Data Model Overview

22 SQLAlchemy ORM models with composite primary keys `(tenant_id, id)`:

| Category | Models |
|----------|--------|
| **Core Conversation** | Conversation, Segment, Line, Participant, Speaker |
| **Extracted Items** | Task, Schedule, Note, Reminder, MemoryAtom, PotentialQuestion |
| **Chat (Per-Recording)** | ChatHistory (legacy), ConversationMessage (new streaming) |
| **Insight (Cross-Recording)** | InsightChat, InsightMessage, SearchHistory (legacy) |
| **Reports & Templates** | Report, Template, TemplateLocale |
| **Graph & Activity** | Relationship, UserActivityLog, UserProfile |

### Vector-Enabled Fields (pgvector, 1536 dimensions)

| Table | Column | Usage |
|-------|--------|-------|
| `lines` | `embedding_vector` | Per-utterance semantic search + summary source tracing |
| `conversations` | `title_embedding_vector` | Cross-conversation title similarity |
| `conversations` | `summary_embedding_vector` | Cross-conversation summary similarity |
| `memory_atoms` | `embedding_vector` | Fine-grained knowledge retrieval |
| `segments` | `summary_embedding_vector` | Topic-level semantic search |
