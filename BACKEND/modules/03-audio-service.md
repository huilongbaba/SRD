# Module 03 -- Audio Service

> Generated: 2026-04-02

## 1. Overview

### Objective
Manage audio recording metadata, S3 presigned URL upload pipeline, folder organization, sharing, audio notes, and act as the primary AI proxy for transcription, summary, agents, chat, and insight features.

### Scope
- Audio recording metadata CRUD (create via presign, list, search, rename, delete, batch operations)
- S3 presigned URL generation for upload (PUT) and CloudFront signed URLs for download (GET)
- Upload complete notification and status tracking
- AI proxy: transcription, summary, report, agents, chat (SSE streaming), insight (SSE streaming)
- Folder management (CRUD, move records)
- Audio sharing (create/update, revoke, public share by token)
- Audio notes (CRUD per recording)
- Export task creation (delegates to Redis Stream for worker)
- Todo sync records (Google Calendar, iOS)
- Template listing (proxy to AI service)

### Non-Scope
- AI processing itself (external AI service)
- Task management (task service, though audio-api calls task-rpc)
- User management (user service)
- Export document generation (worker service)
- Third-party integration auth management (user service)

## 2. Definitions

| Term | Definition |
|------|-----------|
| presign | S3 presigned URL for direct client-to-S3 upload |
| storage_path | S3 URI format `s3://bucket/key` stored in database |
| source_type | Origin of recording: phone microphone, BLE device, import |
| CloudFront signed URL | CDN URL signed with RSA key pair for read access |
| is_read | Whether the user has opened/viewed the recording details |
| denoise | Audio noise reduction via external denoise service |
| SSE proxy | Backend transparently forwards SSE streams from AI service to client |

## 3. System Boundary

| This service handles | Other services handle |
|--------------------|---------------------|
| Recording metadata CRUD | User auth/profile (user service) |
| S3 presigned URL generation | Task CRUD (task service) |
| AI proxy (transcription, summary, chat, insight) | AI processing (AI service) |
| Folder management | Export document generation (worker) |
| Sharing | Notification delivery (notification service) |
| Audio notes | |
| Export task creation + Redis Stream dispatch | |
| Todo sync records | |

### gRPC Dependencies
- **audio-api** calls: audio-rpc (audio + folder + share services), task-rpc, user-rpc, template_community-rpc
- **audio-rpc** is called by: audio-api, user-api, task-api

## 4. Scenarios

### SC-AUDIO-01: Recording Upload (Presigned URL Pattern)
1. Client sends `POST /api/v1/records/presign` with name, duration, source_type, storage_path
2. audio-api -> audio-rpc: `GetRecordPreSign` (INSERT audio row, status=pending)
3. audio-api generates S3 presigned PUT URL (15 min expiry)
4. Returns `{record_id, presigned_url, s3_url}`
5. Client uploads binary audio directly to S3 using presigned URL
6. Client sends `POST /api/v1/records/upload/complete` with upload_id, folder_id
7. audio-api -> audio-rpc: `RecordUploadComplete` (UPDATE status=uploaded, set folder relation)
8. Optionally triggers AI transcription

### SC-AUDIO-02: AI Chat SSE Streaming
1. Client sends `POST /api/v1/records/:id/messages/stream`
2. Route configured with `rest.WithTimeout(0)` -- no timeout
3. Logic sets SSE headers: Content-Type=text/event-stream, Cache-Control=no-cache
4. Logic forwards request to AI service using stream HTTP client (no timeout)
5. For each SSE chunk from AI, logic writes to client response and flushes
6. On `[DONE]` signal, closes connection
7. AES encryption bypassed for SSE responses

### SC-AUDIO-03: Share Creation
1. Client sends `POST /api/v1/records/:audio_id/shares` with include_audio, include_transcript, include_summary_ids, expire_date
2. audio-api -> share-rpc: `CreateOrUpdateShare` (generates unique token)
3. Returns share token
4. Public access via `GET /api/v1/public/shares/:token` (no auth, rate-limited)

### SC-AUDIO-04: Folder Operations
1. Client creates folder: `POST /api/v1/folders` with name, folder_type
2. Client moves record: `POST /api/v1/records/:id/folder` with folder_id
3. Folder types: "audio" for recordings, "task" for tasks

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-AUDIO-01 | MUST | Presign endpoint MUST create audio record with status=pending and return presigned PUT URL | Test: record created in DB; presigned URL valid |
| FR-AUDIO-02 | MUST | Upload complete MUST update audio status to uploaded and create folder relation if folder_id provided | Test: status transitions; folder relation created |
| FR-AUDIO-03 | MUST | Record list MUST support pagination (page, page_size), date range filtering, and folder filtering | Test: verify pagination math and filtering |
| FR-AUDIO-04 | MUST | Record deletion MUST soft-delete (is_deleted=true), not hard-delete | Test: deleted record excluded from list but exists in DB |
| FR-AUDIO-05 | MUST | Batch delete MUST process multiple record IDs in a single request | Test: multiple records soft-deleted |
| FR-AUDIO-06 | MUST | CloudFront signed URLs MUST be generated for all audio file read access | Test: returned URLs are valid CloudFront signed URLs |
| FR-AUDIO-07 | MUST | SSE streaming endpoints MUST set timeout to 0 and bypass AES encryption | Test: SSE connection stays open; raw data delivered |
| FR-AUDIO-08 | MUST | AI proxy MUST enrich requests with tenant_id (userId), language_code, template_id before forwarding | Test: AI service receives enriched request |
| FR-AUDIO-09 | MUST | Share MUST generate unique token and support optional expiry date | Test: token uniqueness; expired share returns error |
| FR-AUDIO-10 | MUST | Public share endpoint MUST be rate-limited (Redis-based) | Test: exceeding rate limit returns 429 |
| FR-AUDIO-11 | MUST | Export task creation MUST insert task in DB (status=pending) and XADD to Redis Stream | Test: DB record + Redis Stream entry created |
| FR-AUDIO-12 | MUST | Audio notes MUST support batch create, list, update, and delete per recording | Test: CRUD operations on notes |
| FR-AUDIO-13 | MUST | Search MUST proxy to AI service for keyword-based record search | Test: search returns matching records |
| FR-AUDIO-14 | SHOULD | Reconnection endpoints SHOULD allow clients to resume interrupted SSE streams | Test: reconnect returns remaining stream data |
| FR-AUDIO-15 | MUST | Todo sync record MUST support both Google Calendar and iOS platforms | Test: sync records created for both platforms |
| FR-AUDIO-16 | MUST | Mark-as-read MUST update is_read flag when user views record details | Test: is_read transitions from false to true |
| FR-AUDIO-17 | MUST | All TemplateId/TemplateID fields MUST use int64 type | Code review: no string template IDs |

## 6. State Model

### Audio Record Status
```
0 (pending) -> 1 (uploaded) -> [AI processing states managed by AI service]
```
- `pending`: Record created, waiting for S3 upload
- `uploaded`: S3 upload confirmed

### Export Task Status
```
0 (pending) -> 1 (processing) -> 2 (done)
                               -> 3 (failed)
```

### Share Status
```
1 (active) -> 2 (revoked)
```

### Todo Sync Status
```
0 (not synced) -> 1 (synced) -> 2 (failed)
```

## 7. Data Contract

### Key REST API Endpoints (85+ total)

**Recording CRUD:**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/records` | Yes | List recordings (paginated, filterable) |
| POST | `/api/v1/records/presign` | Yes | Get S3 presigned URL |
| POST | `/api/v1/records/upload/complete` | Yes | Upload complete notification |
| DELETE | `/api/v1/records/:id` | Yes | Delete record |
| PATCH | `/api/v1/records/:id` | Yes | Rename record |
| GET | `/api/v1/records/search` | Yes | Search records and tasks |
| GET | `/api/v1/records/status` | Yes | Batch get record status |
| DELETE | `/api/v1/records/delete/batch` | Yes | Batch delete |

**AI Proxy (SSE):**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/v1/records/:id/messages/stream` | Yes | Stream chat (SSE) |
| GET | `/api/v1/records/:id/messages/stream/reconnect` | Yes | Resume stream (SSE) |
| POST | `/api/v1/records/:id/messages/:message_id/stream/retry` | Yes | Retry stream (SSE) |
| POST | `/api/v1/insight/messages/sse` | Yes | Stream insight (SSE) |
| GET | `/api/v1/insight/:session_id/stream/resume` | Yes | Resume insight stream |
| POST | `/api/v1/insight/:session_id/stream/:message_id/retry` | Yes | Retry insight stream |

**AI Proxy (REST):**

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/records/:id/detail` | Yes | Get AI detail (transcription, summary) |
| POST | `/api/v1/records/:id/report` | Yes | Generate AI report |
| POST | `/api/v1/records/:id/summary` | Yes | Generate summary |
| POST | `/api/v1/records/:id/agents` | Yes | Generate AI agent |
| POST | `/api/v1/records/:id/chats` | Yes | Ask AI (record context) |
| POST | `/api/v1/records/messages/save` | Yes | Save AI message as summary |

**Folder / Share / Notes / Export:** See full endpoint list in L3_data_flow.md.

### gRPC Services (audio.proto)

- **service `audio`**: 24 RPCs -- recording CRUD, presign, upload complete, export task, integration auth, todo sync, audio notes
- **service `folder`**: 7 RPCs -- folder CRUD, move records
- **service `share`**: 4 RPCs -- create/update, query, revoke, public share by token

### Database Tables

| Table | Key Columns |
|-------|-------------|
| `audio` | id, user_id, name, storage_path, denoised_storage_path, status, source_type, duration, device_id, is_read, is_deleted |
| `audio_note` | id, user_id, audio_id, content, note_time |
| `folder` | id, user_id, name, folder_type, color, icon, is_deleted |
| `folder_relation` | folder_id, relation_id (polymorphic: audio or task) |
| `export_task` | id, user_id, report_id, is_transcription, target_format, status, target_file_path, error_msg |
| `share` | id, token, audio_id, user_id, include_audio/transcript/summary_ids, expire_date, status |
| `integration_auth` | user_id, provider_id, access_token, refresh_token, expires_at, status |
| `todo_sync_record` | user_id, audio_id, todo_id, sync_status, google_event_id, ios_sync_status |

## 8. Error Handling

| Error | Code | Handling |
|-------|------|---------|
| Record not found | 404 | Return "record not found" |
| Unauthorized access (not record owner) | 403 | Return "forbidden" |
| S3 presign generation failure | 500 | Log error, return "storage error" |
| AI service unavailable | 502 | Log error, return "AI service unavailable" |
| AI service timeout | 504 | Log error, return "AI service timeout" |
| SSE connection dropped | -- | Client uses reconnect endpoint with last message ID |
| Export task creation failure | 500 | Log error, return "export creation failed" |
| Share expired | 410 | Return "share expired" |
| Rate limit exceeded (public share) | 429 | Return "rate limit exceeded" |
| Folder not found | 404 | Return "folder not found" |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| HTTP timeout | 50,000 ms | `audio-api: Timeout: 50000` |
| SSE timeout | 0 (disabled) | SSE routes use `rest.WithTimeout(0)` |
| S3 presigned PUT URL expiry | 15 min | `common/third_party/aws/` |
| AI client standard timeout | Configurable | Standard HTTP client |
| AI client stream timeout | 0 (no timeout) | Stream HTTP client |
| gRPC client timeout (task-rpc) | default | `audio-api: TaskRpc` |
| gRPC client timeout (mcp-rpc) | 5,000 ms | `audio-api: McpRpc.Timeout: 5000` |
| Export Redis Stream | `export_task_stream` | `audio-api: ExportTaskStream` |
| CORS origins | `https://capsoul.space` | `audio-api: Cors.AllowOrigins` |
| Share URL prefix | `https://dev.capsoulbeta.space/share/` | `audio-api: ShareUrlPrefix` |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| AI proxy logging | Enhanced logging with aiHost, requestURL, requestMethod (dd6567a) |
| SSE streaming debug | Per-field debug logging in HandleStreamPayload (session_id, message_id, report_id, etc.) |
| Record lifecycle | Status transitions logged |
| S3 operations | Presign generation and upload complete logged |
| Export dispatch | Redis XADD logged with export_task_id and user_id |
| Trace propagation | X-Trace-ID through all RPC and AI service calls |
