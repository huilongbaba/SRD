# Appendix A: Celery Task Chain Specification

## 1. Overview

The Memoket AI backend uses Celery for all heavy asynchronous processing. Tasks are organized into 7 modules across 4 queues, with the primary recording pipeline using a `chain()` for sequential execution and an optional `chord()` for parallel voiceprint extraction.

## 2. Queue Configuration

```python
task_queues = (
    Queue("transcription"),   # CPU/IO heavy: ASR + S3 download
    Queue("analysis"),        # LLM heavy: segmentation + extraction + summary
    Queue("execution"),       # LLM heavy: function calling
    Queue("celery"),          # Default: downloads, graphs, voiceprints, async insight
)

task_routes = {
    "app.workers.tasks.transcription_tasks.*": {"queue": "transcription"},
    "app.workers.tasks.analysis_tasks.*":      {"queue": "analysis"},
    "app.workers.tasks.execution_tasks.*":     {"queue": "execution"},
}
```

All unrouted tasks fall to the `celery` (default) queue.

## 3. Worker Configuration

| Setting | Default | Env Variable | Description |
|---------|---------|-------------|-------------|
| `worker_concurrency` | `4` | `CELERY_CONCURRENCY` / `CELERY_WORKER_CONCURRENCY` | Number of concurrent worker processes |
| `worker_prefetch_multiplier` | `1` | `CELERY_PREFETCH_MULTIPLIER` / `CELERY_WORKER_PREFETCH_MULTIPLIER` | Tasks prefetched per worker process |
| `task_acks_late` | `True` | `CELERY_ACKS_LATE` | Acknowledge only after completion |
| `task_reject_on_worker_lost` | `True` | `CELERY_REJECT_ON_WORKER_LOST` | Reject task if worker crashes |
| `worker_pool` | `prefork` (default) | `CELERY_POOL` | Pool type (prefork/threads) |
| `task_serializer` | `json` | -- | Serialization format |
| `timezone` | `UTC` | -- | Timezone |

## 4. Primary Recording Pipeline (Chain)

The main recording processing pipeline uses a Celery `chain()` for strict sequential execution:

```
chain(
    process_audio.s(tenant_id, conversation_id, s3_url, duration_ms, language_code, trace_id)
    |
    process_segments_analysis.s(tenant_id, conversation_id, trace_id)
    |
    process_items_extraction.s(tenant_id, conversation_id, trace_id)
    |
    process_unified_summary_generation.s(tenant_id, conversation_id, None, template_ids, language_code, trace_id)
    |
    process_function_calls.s(tenant_id, conversation_id, trace_id)
).apply_async()
```

### Chain Step Details

| Step | Task | Queue | Input | Output | Description |
|------|------|-------|-------|--------|-------------|
| 1 | `process_audio` | transcription | S3 URL, language code | `temp_file_path: str` | Download audio, ASR, embedding, persist lines |
| 2 | `process_segments_analysis` | analysis | conversation_id | `{status, conversation_id, lines}` | LLM segmentation, topic classification, speaker roles |
| 3 | `process_items_extraction` | analysis | conversation_id | `{status, conversation_id}` | Extract tasks, notes, memory atoms, potential questions |
| 4 | `process_unified_summary_generation` | analysis | template_ids, language_code | -- | Parallel template summary generation (2-3 templates) |
| 5 | `process_function_calls` | execution | conversation_id | `{status, message}` | LLM function call intent + execution |

### Parallel Voiceprint (Chord)

Optionally, `process_voiceprints` runs in parallel with the analysis chain via a `chord()`:

```
chord(
    process_audio.s(...),                    # Step 1
    process_voiceprints.s(audio_path, ...)   # Parallel: runs when Step 1 completes
)
|
chain(                                       # Sequential: Steps 2-5
    process_segments_analysis.s(...),
    process_items_extraction.s(...),
    process_unified_summary_generation.s(...),
    process_function_calls.s(...)
)
```

## 5. Task Catalog

### 5.1 transcription_tasks

| Task | Queue | ignore_result | Timeouts | Description |
|------|-------|--------------|----------|-------------|
| `process_audio` | transcription | `False` | None (unbounded) | S3 download, ASR transcription, embedding, DB persistence |

### 5.2 analysis_tasks

| Task | Queue | ignore_result | Timeouts | Description |
|------|-------|--------------|----------|-------------|
| `process_segments_analysis` | analysis | `False` | None | LLM conversation segmentation + topic classification |
| `process_items_extraction` | analysis | `False` | None | LLM extraction of tasks, notes, memory atoms, questions |
| `process_unified_summary_generation` | analysis | `False` | None | Parallel template-based summary generation (new) |
| `process_summary_generation` | analysis | `False` | None | Legacy summary generation |

### 5.3 execution_tasks

| Task | Queue | ignore_result | Timeouts | Description |
|------|-------|--------------|----------|-------------|
| `process_function_calls` | execution | `True` | None | Auto function call intent identification + execution |
| `process_function_call_by_user` | execution | -- | None | User-triggered function call execution |

### 5.4 askai_tasks

| Task | Queue | ignore_result | Timeouts | Description |
|------|-------|--------------|----------|-------------|
| `process_askai` | celery (default) | `False` | None | Legacy AI insight search (non-streaming) |
| `process_ai_insight` | celery (default) | `False` | soft_time_limit=`300`s | Non-streaming insight pipeline execution |

### 5.5 download_tasks

| Task | Queue | ignore_result | Timeouts | Description |
|------|-------|--------------|----------|-------------|
| `process_download_youtube` | celery (default) | -- | -- | YouTube audio download via yt-dlp |

### 5.6 graph_tasks

| Task | Queue | ignore_result | Timeouts | Description |
|------|-------|--------------|----------|-------------|
| `process_graph` | celery (default) | -- | -- | Cross-conversation relationship graph construction |

### 5.7 voiceprint_tasks

| Task | Queue | ignore_result | Timeouts | Description |
|------|-------|--------------|----------|-------------|
| `process_voiceprints` | celery (default) | `True` | soft=`840`s, hard=`900`s | Speaker voiceprint extraction via ONNX |

## 6. Error Handling Pattern

All Celery tasks follow a consistent error handling pattern:

```python
@celery_app.task(ignore_result=False)
def process_xxx(tenant_id, conversation_id, trace_id=None):
    if trace_id:
        set_trace_id(trace_id)        # 1. Trace ID propagation

    SessionLocal = get_session_factory()
    db = SessionLocal()                # 2. Own DB session
    try:
        conversation = crud_conversation.get(db, ...)
        if not conversation:
            return {"status": "skipped", "message": "not found"}

        # ... algorithm execution ...

        return {"status": "completed", "message": "done"}
    except Exception as e:
        import traceback, sys
        tb = traceback.format_exc()
        lineno = sys.exc_info()[2].tb_lineno    # 3. Line-level error
        logger.error(f"Task failed at line {lineno}: {e}\n{tb}")

        crud_conversation.update(db, ...,       # 4. Status persistence
            obj_in={"status": "failed"})
        return {"status": "failed", "error": f"{e} (line {lineno})"}
    finally:
        db.close()                              # 5. Always close session
```

Key aspects:
1. **Trace ID propagation**: Every task accepts and sets `trace_id`
2. **Own DB session**: Each task creates and closes its own `SessionLocal()`
3. **Line-level error logging**: Exact line number included in error messages
4. **Status persistence**: DB records updated to "failed" on error
5. **Graceful chain continuation**: Tasks return status dicts rather than raising, allowing downstream tasks to handle

## 7. Notification on Completion

When the full pipeline completes (or fails), `NotificationService` pushes an event to AWS ElastiCache Redis:

```python
# Stream: memoket:notifications (maxlen ~10000)
# Fields: event, tenant_id, audio_id, conversation_id, status, timestamp, error_detail

notify_analysis_complete(tenant_id, audio_id, conversation_id)
notify_analysis_failed(tenant_id, audio_id, conversation_id, error_detail)
```

The Business Backend consumes this stream via `XREADGROUP` to push real-time status updates to clients.

## 8. LLM Model Configuration per Task

| Task / Algorithm | Env Variable | Default Model |
|-----------------|-------------|---------------|
| Conversation analysis (segmentation) | `ONE_IN_ALL_ANALYSIS_MODEL` | `gpt-5-mini` |
| Items extraction | `ITEMS_EXTRACTION_MODEL` | `gpt-5` |
| Summary generation | `SUMMARY_MODEL` | `azure-gpt-5.1` |
| Function call intent | `FUNCTION_CALLS_IDENTIFY_MODEL` | `gpt-5-nano` |
| Agent execution | `AGENT_WRITING_MODEL` | `gpt-5-mini` |
| Insight intent recognition | `AI_INSIGHT_INTENT_AND_TOOL_SELECTION_MODEL` | `azure-gpt-5-mini` |
| Insight response writing | `AI_INSIGHT_WRITING_MODEL` | `gpt-5` |
| Legacy insight function calls | `AI_INSIGHT_FUNCTION_CALLS_IDENTIFY_MODEL` | `gpt-5` |
