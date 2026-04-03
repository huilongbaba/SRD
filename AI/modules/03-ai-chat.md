# Module 03: AI Chat (Per-Recording)

## 1. Purpose

Provide per-recording AI chat (InFile mode) where users can ask questions about a specific recording's transcript. Supports real-time streaming responses via SSE (Server-Sent Events) backed by Redis Streams, with reconnection and retry capabilities.

## 2. Actors

| Actor | Role |
|-------|------|
| Business Backend | Sends chat messages via `POST /records/{audio_id}/chat/stream` |
| FastAPI Endpoint | Creates DB records, launches background generation, serves SSE stream |
| Background Runner (ThreadPoolExecutor) | Executes insight pipeline in thread, writes events to Redis Stream |
| LLM API | Generates response with full transcript context |
| Redis Stream | Buffers events between writer (background thread) and reader (SSE endpoint) |

## 3. Functional Requirements

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-CH-001 | MUST | The system MUST accept streaming chat requests per recording via `POST /records/{audio_id}/chat/stream` | T |
| FR-CH-002 | MUST | The system MUST create a user `ConversationMessage` and a pending AI `ConversationMessage` in the database before starting generation | T |
| FR-CH-003 | MUST | The system MUST launch generation in a `ThreadPoolExecutor` background thread via `launch_insight_generation(chat_type="conversation")` | T |
| FR-CH-004 | MUST | The system MUST stream response tokens to the client via SSE (NDJSON format for conversation mode) using Redis Streams as the event buffer | T |
| FR-CH-005 | MUST | The per-recording chat pipeline MUST skip intent recognition and use the full transcript + summary + tasks as context (simplified pipeline) | T |
| FR-CH-006 | MUST | The system MUST support SSE reconnection via `GET /records/{audio_id}/chat/reconnect` with `last_entry_id` to resume from where the client left off | T |
| FR-CH-007 | MUST | The system MUST support retry of failed messages via `POST /records/{audio_id}/chat/retry` | T |
| FR-CH-008 | MUST | The system MUST provide chat message history via `GET /records/{audio_id}/chat/messages` | T |
| FR-CH-009 | MUST | The system MUST support deleting individual chat messages via `DELETE /records/{audio_id}/chat/{message_id}` | T |
| FR-CH-010 | MUST | The system MUST support "save as summary" functionality: `POST /records/{audio_id}/chat/save-as-summary` creates a Report from the AI message content | T |
| FR-CH-011 | MUST | The "save as summary" endpoint MUST validate: role=assistant, status=completed, saved=false. It MUST set `saved=True` and reject duplicate saves (return code 2) | T |
| FR-CH-012 | MUST | Chat message responses MUST include the `saved` boolean field | I |
| FR-CH-013 | MUST | The system MUST implement idempotent launch: check both in-memory `_running_tasks` dict and Redis status key before submitting a new generation task | T |
| FR-CH-014 | MUST | The background runner MUST persist the completed AI message (response text, sources, follow-ups) to the database | T |

## 4. Data Contract

### 4.1 Chat Stream Endpoint

**`POST /api/v1/records/{audio_id}/chat/stream`**

Request:
```json
{
  "message": "What were the key decisions made?",
  "tenant_id": "uuid-string",
  "template_id": 42,
  "language_code": "en"
}
```

Response: `StreamingResponse` (NDJSON, `text/event-stream`)
```
{"status": 0, "chunk": "Based on the transcript...", "tenant_id": "...", "message_id": 123}
{"status": 2, "chunk": "...the key decisions were:", "tenant_id": "...", "message_id": 123}
{"status": 6, "report_id": 456, "source_audios": [...], "session_title": "Key Decisions"}
```

### 4.2 Save as Summary Endpoint

**`POST /api/v1/records/{audio_id}/chat/save-as-summary`**

Request:
```json
{
  "message_id": 123,
  "tenant_id": "uuid-string"
}
```

Response:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "report_id": 456,
    "report_name": "Chat Summary"
  }
}
```

### 4.3 Reconnect Endpoint

**`GET /api/v1/records/{audio_id}/chat/reconnect?message_id=123&last_entry_id=1234567890-0`**

Response: Resumes SSE stream from `last_entry_id`.

## 5. Processing Flow

```
POST /records/{audio_id}/chat/stream
  |
  v
Get conversation by audio_id
  |
  v
Create user ConversationMessage (DB)
Create pending AI ConversationMessage (DB)
  |
  v
launch_insight_generation(chat_type="conversation")
  |
  +-- Background Thread:
  |     mark_stream_running() -> Redis status key
  |     Load transcript lines + summary + tasks
  |     run_insight_streaming(chat_type="conversation")
  |       Skip intent recognition
  |       Load full transcript context
  |       LLM generation (streaming)
  |         yield {status: GENERATING, chunk} -> XADD to Redis Stream
  |     Persist AI message to DB
  |     XADD {status: COMPLETED} -> Redis Stream
  |     mark_stream_completed()
  |
  +-- SSE Reader (FastAPI endpoint):
        consume_stream("conversation", message_id)
          XREAD from Redis Stream (blocking)
          Apply pace control (0.04s/chunk for conversation)
          Yield NDJSON lines to client
```

## 6. Configuration

| Env Variable | Default | Description |
|-------------|---------|-------------|
| `INSIGHT_MAX_CONCURRENT_GENERATIONS` | `20` | Max concurrent background generation threads |
| `CHAT_MAX_CONTEXT_TOKENS` | `64000` | Max tokens for chat context window |

### Pace Control (Conversation Mode)

| Status | Pace (seconds/chunk) | Notes |
|--------|---------------------|-------|
| PENDING | 1.0 | Slow status message |
| GENERATING | 0.04 | Fast response streaming |
| COMPLETED | 0.01 | Instant final event |
| FAILED | 0.01 | Instant error event |

## 7. Error Handling

| Error Condition | Behavior |
|----------------|----------|
| Background thread crashes | Redis Stream gets no COMPLETED event; consumer times out after grace period; returns FAILED event to client |
| LLM call times out | Pipeline catches exception, writes FAILED event to Redis Stream |
| Client disconnects mid-stream | Generation continues (decoupled via Redis); stream data available for reconnection |
| Stream timeout (client 30s idle) | Client triggers reconnection via `/reconnect` endpoint |
| Duplicate save-as-summary | Returns code 2 (validation error); `saved` flag prevents re-creation |

## 8. Performance Considerations

- **Thread pool limit**: `_MAX_CONCURRENT=20` concurrent streaming requests; overflow blocks on `_executor.submit()`
- **Pace control jitter**: 10% random jitter applied to pace timing for natural-feeling output
- **Stream TTL**: Redis Streams expire after 600 seconds; `cleanup_stream()` available for immediate cleanup
- **Force-flush on status change**: Status transitions (ANALYZING, GENERATING, COMPLETED, FAILED) trigger immediate flush of pending events

## 9. Dependencies

| Dependency | Direction | Description |
|-----------|-----------|-------------|
| `insight_background_runner` | Internal | ThreadPoolExecutor management |
| `insight_stream_buffer` | Internal | Redis Stream read/write |
| `insight/pipeline.py` | Internal | Simplified pipeline (no intent recognition) |
| `crud_conversation_message` | Internal | Message CRUD |
| `crud_report` | Internal | Save-as-summary report creation |

## 10. Open Issues

| Issue | Status | Notes |
|-------|--------|-------|
| Thread pool exhaustion under high concurrency | Risk | 20+ simultaneous requests will block; no backpressure mechanism |
| No cost tracking for chat LLM calls | Risk | Each chat message triggers an LLM call with full transcript context |
| Legacy `chats.py` still exists | Technical debt | Non-streaming chat endpoints in `chats.py` coexist with streaming endpoints in `records.py` |
