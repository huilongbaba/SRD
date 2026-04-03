# Module 04: Insight RAG Pipeline (Cross-Recording)

## 1. Purpose

Power the cross-recording "Memochat" feature where users ask questions spanning multiple conversations. The system recognizes intent, selects retrieval tools, performs multi-granularity RAG retrieval (pgvector + trigram + RRF fusion), and generates informed responses with source attribution. Supports both streaming (SSE) and non-streaming (Celery task) modes.

## 2. Actors

| Actor | Role |
|-------|------|
| Business Backend | Sends insight messages via `POST /insights/messages/stream` or `POST /insights/messages` |
| FastAPI Endpoint (insights.py) | Manages sessions, creates messages, launches generation |
| Background Runner (ThreadPoolExecutor) | Executes pipeline, writes to Redis Stream (streaming mode) |
| Celery Worker (default queue) | Executes pipeline (non-streaming mode via `process_ai_insight`) |
| LLM API | Intent recognition, response generation, report generation, title generation |
| PostgreSQL + pgvector | Multi-granularity vector + trigram search |
| Redis Stream | SSE event buffer (streaming mode) |

## 3. Functional Requirements

### 3.1 Session & Message Management

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-IR-001 | MUST | The system MUST manage insight chat sessions (`InsightChat`) with creation, listing (paginated), deletion, and title updates | T |
| FR-IR-002 | MUST | The system MUST manage insight messages (`InsightMessage`) within sessions, tracking role, status, reasoning_plan, report_content, related_conversation_ids, external_webs, suggested_follow_ups, and enable_modes | T |
| FR-IR-003 | MUST | The system MUST support the `saved` boolean flag on insight messages, with idempotent save-as-summary behavior | T |
| FR-IR-004 | MUST | The system MUST provide suggested cross-conversation questions via `GET /insights/suggested-questions` | T |
| FR-IR-005 | MUST | The system MUST provide per-message follow-up suggestions via `GET /insights/suggested-follow-ups` | T |

### 3.2 Streaming Pipeline

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-IR-010 | MUST | The streaming endpoint (`POST /insights/messages/stream`) MUST launch generation in a background thread and serve SSE responses immediately | T |
| FR-IR-011 | MUST | Phase 1 (Prepare): The pipeline MUST extract recent messages within `MAX_CONTEXT_TOKENS=64000`, load session context summary, and build `SessionContext` | T |
| FR-IR-012 | MUST | Phase 2 (Intent): The pipeline MUST perform LLM-based intent recognition with streaming reasoning chunks delivered to the client in real-time (status: ANALYZING) | T |
| FR-IR-013 | MUST | Phase 2 (Intent): The intent output MUST be parsed from a `---JSON---` delimiter in the streaming LLM output, yielding `is_valid`, `response_format`, `selected_tools`, `entities`, `time_range` | T |
| FR-IR-014 | MUST | Phase 3 (Retrieval): If `rag_required`, the pipeline MUST execute multi-granularity retrieval in parallel via ThreadPoolExecutor | T |
| FR-IR-015 | MUST | Phase 3 (Retrieval): The system MUST perform both pgvector (semantic) and pg_trgm (lexical) search per data type (conversations, segments, memory_atoms, lines, tasks, reports) | T |
| FR-IR-016 | MUST | Phase 3 (Retrieval): Results MUST be merged via RRF (Reciprocal Rank Fusion) with k=60, then aggregated to conversation level with configurable weights and decay=0.7 | T |
| FR-IR-017 | MUST | Phase 3 (Retrieval): Each conversation's evidence MUST be packaged into a `ConversationEvidencePackage` containing transcript excerpts, summaries, memory atoms, and metadata | T |
| FR-IR-018 | MUST | Phase 4 (Response): The pipeline MUST generate a streaming response with RAG context and source attribution (status: GENERATING) | T |
| FR-IR-019 | SHOULD | Phase 5 (Report): If `response_format == 'report'`, the pipeline SHOULD generate a detailed report via a separate LLM call (status: REPORTING) | T |
| FR-IR-020 | MUST | Phase 6 (Persist): The background runner MUST persist the AI message with response, sources, follow-ups, and report to the database | T |
| FR-IR-021 | MUST | The system MUST generate a session title from the first user message via `gen_title()` | T |
| FR-IR-022 | MUST | Conversation source citations MUST be deduplicated by `(audio_id, segment_id, line_id)` via `_build_conversation_dedup_key()` | T |

### 3.3 Retrieval Null-Safety

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-IR-030 | MUST | The retrieval context builder MUST apply defensive `or` fallbacks when extracting conversation metadata (`title`, `participants`, `started_at`, `duration`) | T |
| FR-IR-031 | MUST | The retrieval pipeline MUST guard against `None` for `intent_results` to prevent `NoneType` errors | T |

## 4. Data Contract

### 4.1 Streaming Endpoint

**`POST /api/v1/insights/messages/stream`**

Request:
```json
{
  "session_id": 100,
  "message": "What decisions were made across my meetings last week?",
  "tenant_id": "uuid-string",
  "enable_modes": ["web_search"]
}
```

Response: `StreamingResponse` (SSE `text/event-stream`)
```
data: {"status": 0, "chunk": "Let me analyze...", "message_id": 456}
data: {"status": 1, "chunk": "Searching recordings..."}
data: {"status": 2, "chunk": "Based on your meetings...", "message_id": 456}
data: {"status": 6, "report_id": 789, "source_audios": [...], "suggested_follow_ups": [...]}
```

### 4.2 Celery Task (Non-Streaming)

```python
process_ai_insight(
    tenant_id: str,
    session_id: int,
    message_id: int,
    trace_id: Optional[str] = None
)
```

**Queue**: `celery` (default)
**soft_time_limit**: `300` seconds (configurable via `INSIGHT_TASK_SOFT_TIME_LIMIT`)

### 4.3 SSE Status Codes

| Status | Name | Description |
|--------|------|-------------|
| 0 | ANALYZING | Intent recognition reasoning chunks |
| 1 | RETRIEVING | Retrieval progress status |
| 2 | GENERATING | Response text tokens |
| 3 | PREPARING | Context preparation status |
| 5 | REPORTING | Report generation chunks |
| 6 | COMPLETED | Final event with report, sources, follow-ups |
| 7 | FAILED | Error event |

## 5. Processing Flow

```
POST /insights/messages/stream
  |
  v
Get/create InsightChat session
Create user InsightMessage + pending AI InsightMessage
  |
  v
launch_insight_generation(chat_type="insight")
  |
  +-- Background Thread:                       +-- SSE Reader:
  |     Phase 1: Prepare Context               |     consume_stream("insight", msg_id)
  |       Extract recent messages (64K tokens)  |     XREAD from Redis Stream
  |       Load session context_summary          |     Pace-controlled yield to client
  |       Build SessionContext                  |
  |     Phase 2: Intent Recognition             |
  |       Stream LLM call (reasoning chunks)    |
  |       -> XADD {status:0} events             |
  |       Parse ---JSON--- intent result        |
  |     Phase 3: Retrieval (if rag_required)    |
  |       Parallel fetchers (ThreadPoolExecutor)|
  |         Vector search (pgvector)            |
  |         Trigram search (pg_trgm)            |
  |       RRF fusion (k=60)                     |
  |       Normalize scores [0,1]                |
  |       Aggregate to conversation level       |
  |       Build ConversationEvidencePackage     |
  |     Phase 4: Response Generation            |
  |       Stream LLM call with RAG context      |
  |       -> XADD {status:2} events             |
  |     Phase 5: Report (if needed)             |
  |       Generate detailed report              |
  |       -> XADD {status:5} events             |
  |     Phase 6: Persist                        |
  |       Update InsightMessage in DB           |
  |       Create Report (if applicable)         |
  |       Update session title (first message)  |
  |       XADD {status:6} COMPLETED event       |
```

## 6. Configuration

| Env Variable | Default | Description |
|-------------|---------|-------------|
| `AI_INSIGHT_INTENT_AND_TOOL_SELECTION_MODEL` | `"azure-gpt-5-mini"` | Model for intent recognition |
| `AI_INSIGHT_WRITING_MODEL` | `"gpt-5"` | Model for response generation |
| `AI_INSIGHT_FUNCTION_CALLS_IDENTIFY_MODEL` | `"gpt-5"` | Model for legacy function call identification |
| `INSIGHT_TASK_SOFT_TIME_LIMIT` | `300` | Celery soft time limit in seconds |
| `INSIGHT_MAX_CONCURRENT_GENERATIONS` | `20` | Max concurrent background generation threads |
| `CHAT_MAX_CONTEXT_TOKENS` | `64000` | Max tokens for context window |

### Pace Control (Insight Mode)

| Status | Pace (seconds/chunk) |
|--------|---------------------|
| PENDING | 0.8 |
| ANALYZING | 0.1 |
| RETRIEVING | 1.5 |
| GENERATING | 0.08 |
| REPORTING | 3.0 |
| COMPLETED | 0.01 |
| FAILED | 0.01 |

## 7. Error Handling

| Error Condition | Behavior |
|----------------|----------|
| Intent recognition fails | Pipeline catches exception, writes FAILED event |
| No relevant conversations found | Generates response with "no relevant data found" message |
| Retrieval null values | Defensive fallbacks prevent NoneType crashes |
| Background thread crashes | Consumer times out after grace period; FAILED status |
| LLM timeout | Pipeline exception -> FAILED event to stream |
| Celery soft_time_limit exceeded | `SoftTimeLimitExceeded` exception raised, message status set to `"failed"` |

## 8. Performance Considerations

- **Parallel retrieval**: Multi-granularity fetchers execute concurrently via ThreadPoolExecutor
- **RRF fusion**: O(n) merge complexity per granularity; no score calibration needed
- **Weighted decay aggregation**: Multiple hits from the same conversation receive diminishing returns (decay=0.7) preventing over-representation
- **Stream maxlen**: 5000 events per stream to prevent memory issues
- **Pace jitter**: 10% random jitter for natural output feel

## 9. Dependencies

| Dependency | Direction | Description |
|-----------|-----------|-------------|
| `insight/pipeline.py` | Internal | Core pipeline orchestrator |
| `insight/prepare.py` | Internal | Context preparation |
| `insight/intent/identification.py` | Internal | LLM-based intent recognition |
| `insight/retrieval/` | Internal | Multi-granularity RAG retrieval |
| `insight/response.py` | Internal | Streaming response generation + dedup |
| `insight_background_runner.py` | Internal | ThreadPoolExecutor management |
| `insight_stream_buffer.py` | Internal | Redis Stream read/write |
| `askai_tools/tools.py` | Internal | Tool execution (search, web) |
| All CRUD modules | Internal | Database access for retrieval |

## 10. Open Issues

| Issue | Status | Notes |
|-------|--------|-------|
| Thread pool exhaustion | Risk | 20+ concurrent insight requests will block |
| No per-tenant retrieval budget | Risk | Heavy queries may overload pgvector indexes |
| Legacy `process_askai` task coexists | Technical debt | Old non-streaming pipeline remains active |
| No caching of retrieval results | Enhancement | Repeated similar queries re-execute full retrieval |
