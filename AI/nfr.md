# 5. Non-Functional Requirements

## NFR-01: Concurrency & Scalability

| ID | Priority | Requirement | Current Value | Source |
|----|----------|-------------|---------------|--------|
| NFR-SC-001 | MUST | Celery worker concurrency MUST be configurable | Default: `4` (via `CELERY_CONCURRENCY` env) | `celery_app.py` |
| NFR-SC-002 | MUST | Celery MUST use separate queues for transcription, analysis, execution, and default workloads to enable independent scaling | 4 queues: `transcription`, `analysis`, `execution`, `celery` | `celery_app.py` |
| NFR-SC-003 | MUST | Celery worker prefetch multiplier MUST be configurable and default to 1 (process one task at a time) | Default: `1` (via `CELERY_PREFETCH_MULTIPLIER`) | `celery_app.py` |
| NFR-SC-004 | MUST | Streaming insight generation MUST limit concurrent background threads | Default: `20` (via `INSIGHT_MAX_CONCURRENT_GENERATIONS`) | `insight_background_runner.py` |
| NFR-SC-005 | SHOULD | The system SHOULD support configurable Celery pool type (prefork/threads) | Via `CELERY_POOL` env var (optional) | `celery_app.py` |

## NFR-02: Reliability & Fault Tolerance

| ID | Priority | Requirement | Current Value | Source |
|----|----------|-------------|---------------|--------|
| NFR-RE-001 | MUST | Celery tasks MUST acknowledge only after completion (`task_acks_late`) | `True` (default, configurable via `CELERY_ACKS_LATE`) | `celery_app.py` |
| NFR-RE-002 | MUST | Celery tasks MUST be rejected on worker lost (`task_reject_on_worker_lost`) | `True` (default, configurable via `CELERY_REJECT_ON_WORKER_LOST`) | `celery_app.py` |
| NFR-RE-003 | MUST | ASR transcription MUST implement automatic provider fallback on failure | Azure Speech -> AssemblyAI fallback chain | `transcription_algos.py` |
| NFR-RE-004 | MUST | Streaming generation MUST implement idempotent launch (prevent duplicate tasks) | In-memory `_running_tasks` dict + Redis status key check | `insight_background_runner.py` |
| NFR-RE-005 | MUST | SSE streaming MUST support client reconnection via `last_entry_id` | Reconnect endpoint with Redis Stream resume | `insight_stream_buffer.py` |
| NFR-RE-006 | MUST | Every Celery task MUST create its own DB session and close it in `finally` block | Pattern enforced across all 7 task modules | All `*_tasks.py` |

## NFR-03: Timeouts

| ID | Priority | Requirement | Current Value | Source |
|----|----------|-------------|---------------|--------|
| NFR-TO-001 | MUST | Voiceprint extraction MUST have a soft time limit | `840` seconds (14 min) | `voiceprint_tasks.py` |
| NFR-TO-002 | MUST | Voiceprint extraction MUST have a hard time limit | `900` seconds (15 min) | `voiceprint_tasks.py` |
| NFR-TO-003 | MUST | Non-streaming insight processing MUST have a soft time limit | `300` seconds (5 min, configurable via `INSIGHT_TASK_SOFT_TIME_LIMIT`) | `askai_tasks.py` |
| NFR-TO-004 | MUST | Redis Stream TTL MUST prevent stale streams from accumulating | `600` seconds (10 min) | `insight_stream_buffer.py` |
| NFR-TO-005 | MUST | Redis Stream max length MUST be bounded | `5000` events per stream | `insight_stream_buffer.py` |
| NFR-TO-006 | MUST | S3 download timeout for pre-signed URLs MUST be enforced | `300` seconds (5 min) | `transcription_tasks.py` |
| NFR-TO-007 | SHOULD | Web page fetch timeout SHOULD be enforced | `120` seconds via WebParserClient | `web_searching.py` |

## NFR-04: Observability

| ID | Priority | Requirement | Current Value | Source |
|----|----------|-------------|---------------|--------|
| NFR-OB-001 | MUST | Every request MUST receive a trace ID (from `X-Trace-Id` header or auto-generated UUID) propagated through all layers | `TraceIdMiddleware` + `set_trace_id()` in Celery tasks + background threads | `middleware`, all tasks |
| NFR-OB-002 | MUST | Celery task errors MUST include traceback with exact line number | `sys.exc_info()[2].tb_lineno` pattern | All `*_tasks.py` |
| NFR-OB-003 | MUST | All log entries MUST include structured `extra_fields` with `trace_id`, `tenant_id`, and context-specific fields | `logger.info/error(msg, extra={"extra_fields": {...}})` | All modules |
| NFR-OB-004 | SHOULD | API monitoring middleware SHOULD track request/response metrics | `APIMonitoringMiddleware` present | `main.py` |
| NFR-OB-005 | SHOULD | CloudWatch metrics collection SHOULD be active for LLM call tracking | Module exists (`cloudwatch_metrics.py`) but partially integrated | `utils/cloudwatch_metrics.py` |

## NFR-05: Streaming Performance

| ID | Priority | Requirement | Current Value | Source |
|----|----------|-------------|---------------|--------|
| NFR-SP-001 | MUST | SSE pace control MUST be configurable per `(chat_type, status)` combination | Insight: 0.01-3.0s; Conversation: 0.01-1.0s | `insight_stream_buffer.py` |
| NFR-SP-002 | MUST | Status transitions MUST trigger immediate flush of pending events (`_FORCE_FLUSH_ON_NEW_STATUS`) | ANALYZING, GENERATING, COMPLETED, FAILED trigger flush | `insight_stream_buffer.py` |
| NFR-SP-003 | SHOULD | Pace timing SHOULD include 10% random jitter for natural-feeling output | Jitter applied in consumer | `insight_stream_buffer.py` |
| NFR-SP-004 | MUST | Chat context window MUST be bounded | `MAX_CONTEXT_TOKENS=64000` (via `CHAT_MAX_CONTEXT_TOKENS`) | `insight/prepare.py` |

## NFR-06: Security

| ID | Priority | Requirement | Current Value | Source |
|----|----------|-------------|---------------|--------|
| NFR-SE-001 | MUST | All non-audio endpoints MUST require `X-Internal-API-Key` header | `get_internal_api_key` dependency on 7 routers | `security.py`, all endpoints |
| NFR-SE-002 | MUST | The audio endpoint MUST require JWT Bearer authentication | `get_current_tenant` dependency on `audio.py` | `audio.py` |
| NFR-SE-003 | MUST | gRPC InsightsService MUST be protected by the same `INTERNAL_API_KEY` mechanism | Metadata check in `InsightsServicer` | `insights_service.py` |
| NFR-SE-004 | SHOULD | The `SECRET_KEY` for JWT signing SHOULD be validated at startup | Currently uses `os.getenv("SECRET_KEY")` with no startup validation | `security.py` |

## NFR-07: Data Integrity

| ID | Priority | Requirement | Current Value | Source |
|----|----------|-------------|---------------|--------|
| NFR-DI-001 | MUST | All ORM models MUST use composite primary keys `(tenant_id, id)` for tenant isolation | 22 models with composite PKs | All `schemas/*.py` |
| NFR-DI-002 | MUST | Segment creation and line update MUST use two-phase commit to prevent deadlocks | Phase 1: create segments + commit; Phase 2: update lines (sorted by id) + commit | `analysis_tasks.py` |
| NFR-DI-003 | MUST | Error response codes MUST follow the established convention | 0=success, 2=validation, 3=not found, 4=internal, 6=service failure, 8=version mismatch | All endpoints |

## NFR-08: Notification

| ID | Priority | Requirement | Current Value | Source |
|----|----------|-------------|---------------|--------|
| NFR-NO-001 | MUST | Pipeline completion/failure MUST emit notification events to AWS ElastiCache Redis Stream | `XADD memoket:notifications` with maxlen ~10000 | `notification_service.py` |
| NFR-NO-002 | MUST | Notification events MUST include `event`, `tenant_id`, `audio_id`, `conversation_id`, `status`, `timestamp` | Structured fields in Redis Stream entry | `notification_service.py` |
