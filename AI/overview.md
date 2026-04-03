# 1. Overview

## 1.1 Objective

Memoket AI Backend (`CodeBase/AI`) is the intelligence layer of the Memoket (CapsoulAI) platform. It provides the AI/ML processing engine that transforms raw audio recordings into structured, searchable, and actionable knowledge. Core capabilities include:

- **Audio transcription** with multi-provider ASR and speaker diarization
- **Conversation analysis** with topic segmentation, entity extraction, and memory atom generation
- **Template-based summary generation** with source tracing and One-pager HTML rendering
- **Per-recording AI chat** (InFile mode) with full transcript context
- **Cross-recording RAG insight** (CrossFile mode) with intent recognition and multi-tool retrieval
- **Knowledge graph** for cross-conversation relationship discovery
- **Voiceprint identification** for speaker recognition across recordings
- **Audio noise reduction** via GTCRN deep learning model

## 1.2 Scope

| In Scope | Out of Scope |
|----------|-------------|
| ASR transcription (Azure Speech, AssemblyAI) | User authentication / registration |
| LLM-based conversation analysis | File/object storage management (S3 lifecycle) |
| RAG retrieval pipeline (pgvector + trigram) | Mobile/web UI rendering |
| Streaming SSE responses via Redis Streams | Push notification delivery (FCM) |
| Knowledge graph construction | Payment / subscription management |
| Template recommendation engine | BLE device protocol handling |
| Voiceprint extraction and matching | Export-to-PDF/DOCX rendering |
| Audio noise reduction (GTCRN) | OAuth token management for third-party integrations |

## 1.3 Tech Stack

| Layer | Technology | Version / Notes |
|-------|-----------|-----------------|
| Language | Python | 3.11+ |
| Web Framework | FastAPI (ASGI) | uvicorn server on :8000 |
| Task Queue | Celery | Redis broker + result backend |
| ORM | SQLAlchemy 2.0 | + Alembic migrations |
| Database | PostgreSQL + pgvector | Vector similarity search (1536-dim) |
| Cache / Stream | Redis | Celery broker + SSE stream buffer + notification bus |
| LLM Integration | LiteLLM Router | OpenAI, Azure OpenAI, Anthropic Claude, DashScope/Qwen |
| ASR Providers | Azure Speech Services, AssemblyAI | Language-based priority with automatic fallback |
| Embedding | OpenAI Embeddings API | 1536-dimension vectors |
| Audio Processing | torchaudio + ONNX Runtime | Voiceprint extraction, noise reduction |
| Object Storage | AWS S3 | Audio file storage |
| Monitoring | AWS CloudWatch | Metrics collection (module exists, partially integrated) |
| gRPC | grpcio + protobuf | InsightsService for inter-service RPC |
| MCP | Model Context Protocol | External AI agent integration via OAuth |

## 1.4 Entry Points

| Process | Command | Purpose |
|---------|---------|---------|
| FastAPI Server | `uvicorn app.main:app --port 8000` | HTTP/SSE API serving |
| Celery Worker | `celery -A app.workers.celery_app worker` | Async task processing |
| gRPC Server | Embedded in FastAPI startup | InsightsService RPC |
| MCP Server | Separate Starlette application | External AI agent integration |

## 1.5 Repository Structure

```
app/
  api/v1/endpoints/     8 FastAPI routers
  services/             10 service classes
  crud/                 20+ typed data access objects
  schemas/              22 SQLAlchemy ORM models
  models/               Pydantic request/response models
  workers/
    celery_app.py       Celery configuration
    tasks/              7 task modules
    algos/              Algorithm implementations
      insight/          RAG insight pipeline
      summary_template/ Summary generation engine
      askai_tools/      Tool execution for insight
  utils/                LLM selector, prompts, tools
  core/                 Database, security, logging
  grpc_services/        gRPC InsightsService
  mcp/                  Model Context Protocol server
```
