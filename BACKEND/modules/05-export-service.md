# Module 05 -- Export Service

> Generated: 2026-04-02

## 1. Overview

### Objective
Generate PDF and Word documents from AI transcriptions and reports, operating as an asynchronous Redis Stream consumer (export worker) with a REST API facade for task creation and status polling.

### Scope
- Export task creation (via audio-api): insert DB record + dispatch to Redis Stream
- Export worker: Redis Stream consumer group, content fetching from AI, document conversion, S3 upload
- Export status polling and download URL generation (via audio-api)
- Batch export creation and status queries

### Non-Scope
- AI content generation (AI service)
- Audio recording management (audio service)
- User authentication (gateway middleware)

## 2. Definitions

| Term | Definition |
|------|-----------|
| export_task | Database record tracking the lifecycle of a single export operation |
| export_task_stream | Redis Stream used to dispatch export jobs |
| consumer group | Redis Streams consumer group for reliable message processing |
| dead letter stream | Redis Stream where failed messages are moved after max retries |
| target_format | 1=PDF, 2=Word (DOCX) |
| is_transcription | If true, export raw transcription; if false, export AI report |
| wkhtmltopdf | External binary for HTML-to-PDF conversion |
| pandoc | External binary for HTML-to-DOCX conversion |
| goldmark | Go library for Markdown-to-HTML conversion |

## 3. System Boundary

| This module handles | Other modules handle |
|--------------------|---------------------|
| Export task creation and dispatch | Audio metadata (audio service) |
| Redis Stream consumption | AI content generation (AI service) |
| Content fetching from AI service | User authentication (gateway) |
| Document format conversion (PDF/Word) | Notification to client (notification service) |
| S3 upload of generated documents | |
| Task status tracking | |

## 4. Scenarios

### SC-EXPORT-01: Create Export Task
1. Client sends `POST /api/v1/exports/:id` with `{format: pdf|word}`
2. audio-api -> audio-rpc: `CreateExportTask` (INSERT export_task, status=pending)
3. audio-api -> Redis: `XADD export_task_stream {export_task_id, user_id}`
4. Returns `{export_task_id}`

### SC-EXPORT-02: Worker Processes Export
1. Worker reads from Redis Stream: `XREADGROUP export_worker_group` (block 5s)
2. Worker reclaims stuck pending messages first (`reclaimPending`)
3. Worker fetches export_task from PostgreSQL
4. Checks task status (skips if not `pending` -- idempotency)
5. Updates status to `processing`
6. Fetches content from AI service:
   - If `is_transcription=true`: `GET /records/{audio_id}` (transcription JSON -> formatted text)
   - If `is_transcription=false`: `GET /reports?id={report_id}` (HTML or Markdown)
7. Converts content:
   - Markdown -> HTML (goldmark with table extension)
   - HTML styling injection (CSS for fonts/layout, goquery cleanup)
   - PDF: HTML -> PDF via `wkhtmltopdf` subprocess
   - DOCX: HTML -> DOCX via `pandoc` subprocess
8. Uploads generated file to S3: `exports/{userId}/{taskId}.{ext}`
9. Updates status to `done`, sets `target_file_path`
10. Acknowledges Redis message

### SC-EXPORT-03: Client Polls Export Status
1. Client sends `GET /api/v1/exports/:id` or `POST /api/v1/exports/status/batch`
2. audio-api -> audio-rpc: `GetExportTaskById`
3. If status=done: generate CloudFront signed URL from target_file_path
4. Returns `{status, download_url}`

### SC-EXPORT-04: Export Failure and Dead Letter
1. Worker encounters error during processing
2. Error logged, status updated to `failed` with error_msg
3. Retry count incremented in Redis hash (`export_task_retries`)
4. If max retries exceeded: message moved to `export_task_dead` stream

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-EXPORT-01 | MUST | Export task creation MUST insert DB record (status=pending) and XADD to Redis Stream | Test: DB record + Stream entry created |
| FR-EXPORT-02 | MUST | Worker MUST consume from Redis Stream using consumer groups (XREADGROUP) | Test: message consumed and acknowledged |
| FR-EXPORT-03 | MUST | Worker MUST check task status before processing (skip if not pending) | Test: duplicate message does not reprocess |
| FR-EXPORT-04 | MUST | Worker MUST fetch content from AI service (transcription or report) | Test: correct AI endpoint called based on is_transcription |
| FR-EXPORT-05 | MUST | Worker MUST convert Markdown to HTML using goldmark | Test: valid HTML output from Markdown input |
| FR-EXPORT-06 | MUST | Worker MUST generate PDF via wkhtmltopdf subprocess | Test: valid PDF file produced |
| FR-EXPORT-07 | MUST | Worker MUST generate DOCX via pandoc subprocess | Test: valid DOCX file produced |
| FR-EXPORT-08 | MUST | Worker MUST upload generated file to S3 at `exports/{userId}/{taskId}.{ext}` | Test: file exists in S3 |
| FR-EXPORT-09 | MUST | Worker MUST update task status to done/failed after processing | Test: status reflects processing outcome |
| FR-EXPORT-10 | MUST | Worker MUST handle failures with retry and dead-letter mechanism | Test: failed task retried; exceeded retries moved to dead letter |
| FR-EXPORT-11 | MUST | Worker MUST reclaim stuck pending messages on startup | Test: orphaned messages reprocessed |
| FR-EXPORT-12 | MUST | Status endpoint MUST return CloudFront signed download URL when task is done | Test: valid signed URL returned |
| FR-EXPORT-13 | MUST | Batch status endpoint MUST accept multiple task IDs | Test: multiple statuses returned |
| FR-EXPORT-14 | MUST | Each export task MUST have an independent context timeout | Test: long-running task cancelled after timeout |
| FR-EXPORT-15 | SHOULD | Batch export creation SHOULD accept multiple report IDs in a single request | Test: multiple tasks created |

## 6. State Model

### Export Task State Machine
```
pending (0) --> processing (1) --> done (2)
                    |
                    +--> failed (3) --> [retry] --> processing (1)
                                            |
                                            +--> [max retries] --> dead letter
```

| State | Value | Description |
|-------|-------|-------------|
| pending | 0 | Task created, waiting for worker pickup |
| processing | 1 | Worker is actively processing |
| done | 2 | Document generated and uploaded to S3 |
| failed | 3 | Processing failed (error_msg populated) |

### Redis Stream Consumer Lifecycle
```
XREADGROUP (block 5s) -> process message -> XACK
                                         -> on error: retry count++ -> XADD dead letter (if max)
```

## 7. Data Contract

### REST API Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | `/api/v1/exports/:id` | Yes | `{format: 1\|2}` | `{export_task_id}` |
| GET | `/api/v1/exports/:id` | Yes | -- | `{status, download_url?, error_msg?}` |
| POST | `/api/v1/exports/create/batch` | Yes | `{items: [{report_id, format, is_transcription}]}` | `{task_ids}` |
| POST | `/api/v1/exports/status/batch` | Yes | `{task_ids: [int64]}` | `{tasks: [{id, status, download_url?}]}` |

### gRPC Messages (audio.proto -- export subset)

```protobuf
message CreateExportTaskReq {
  int64 user_id = 1;
  int64 report_id = 2;
  bool is_transcription = 3;
  int64 target_format = 4; // 1=PDF 2=Word
}

message ExportTaskDetail {
  int64 id = 1;
  int64 user_id = 2;
  int64 report_id = 3;
  bool is_transcription = 4;
  string target_file_path = 5;
  int64 target_format = 6;
  int64 status = 7; // 0=pending 1=processing 2=done 3=failed
  string error_msg = 8;
  int64 created_at = 9;
  int64 updated_at = 10;
}
```

### Redis Stream
- Stream name: `export_task_stream`
- Consumer group: `export_worker_group`
- Dead letter stream: `export_task_dead`
- Retry hash: `export_task_retries`
- Message fields: `{export_task_id, user_id}`

### Database Table

| Table | Key Columns |
|-------|-------------|
| `export_task` | id, user_id, report_id, is_transcription, target_format, status, target_file_path, error_msg, created_at, updated_at |

## 8. Error Handling

| Error | Handling |
|-------|---------|
| AI service unavailable (content fetch) | Update status=failed, error_msg; retry on next attempt |
| wkhtmltopdf subprocess failure | Update status=failed; log stderr output |
| pandoc subprocess failure | Update status=failed; log stderr output |
| S3 upload failure | Update status=failed; retry |
| Task not found in DB | Log warning; acknowledge message (no retry) |
| Task already processed (not pending) | Skip processing; acknowledge message (idempotent) |
| Context timeout (5 min) | Cancel processing; update status=failed |
| Max retries exceeded | Move to dead letter stream |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| Per-task timeout | 300 s (5 min) | `worker: Export.Timeout: 300` |
| Redis block read timeout | 5 s | XREADGROUP block parameter |
| Consumer group | `export_worker_group` | `worker: Export.ConsumerGroup` |
| Dead letter stream | `export_task_dead` | `worker: Export.DeadStream` |
| S3 upload path | `exports/{userId}/{taskId}.{ext}` | Worker logic |
| Log retention | 7 days, compressed | `worker: Log.KeepDays: 7, Compress: true` |
| Log level | info | `worker: Log.Level: info` |
| Worker log path | `/app/logs/worker` | `worker: Log.Path` |
| External tools | wkhtmltopdf, pandoc | System dependencies |
| Markdown engine | goldmark (with table extension) | Go library |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| Task status transitions | Logged at each state change (pending -> processing -> done/failed) |
| AI content fetch | Logged with report_id, is_transcription flag |
| Conversion subprocess | Logged with command, exit code, stderr |
| S3 upload | Logged with object key, file size |
| Dead letter events | Logged with task_id, retry count, error |
| File-based logging | `/app/logs/worker` with 7-day retention and compression |
