# Module 08 -- Task Service

> Generated: 2026-04-02

## 1. Overview

### Objective
Manage user tasks (both AI-extracted and manually created), provide Google Calendar two-way synchronization, and handle calendar webhook events for real-time updates.

### Scope
- Task CRUD (create, read, update, delete, batch operations)
- Task listing with pagination, folder filtering, status filtering
- Task search by keyword
- Move tasks to folders (single and batch)
- Google Calendar two-way sync (batch upsert, batch delete)
- Calendar webhook handling (Google Calendar push notifications)
- Task association with audio recordings (via audio_id, ai_task_id)

### Non-Scope
- AI task extraction from recordings (AI service)
- Audio recording management (audio service)
- Calendar OAuth authorization (user service)
- Task notification delivery (notification service)

## 2. Definitions

| Term | Definition |
|------|-----------|
| task | A todo item with title, content, priority, due date, reminders, repeat rules |
| ai_task_id | Identifier linking a task to an AI-extracted task from a recording |
| audio_id | Recording from which the task was identified (0 for manual tasks) |
| google_calendar_sync | Record mapping a local task to a Google Calendar event |
| calendar_webhook | Active Google Calendar push notification subscription |
| sync_direction | Direction of sync: "from_google", "to_google", "bidirectional" |
| sync_status | Current sync state: "synced", "pending", "failed" |

## 3. System Boundary

| This service handles | Other services handle |
|--------------------|---------------------|
| Task CRUD and search | AI task extraction (AI service) |
| Google Calendar sync records | Calendar OAuth tokens (user service) |
| Webhook event processing | Audio recording metadata (audio service) |
| Task-folder relationships | Notification delivery (notification service) |

### gRPC Dependencies
- **task-api** calls: task-rpc, audio-rpc (audio + folder), user-rpc
- **task-rpc** is called by: task-api, audio-api, user-api

## 4. Scenarios

### SC-TASK-01: Create Task (Manual)
1. Client sends `POST /api/v1/tasks` with title, content, priority, due_time, etc.
2. task-api -> task-rpc: `TaskCreate`
3. task-rpc inserts task record (audio_id=0, no ai_task_id)
4. Returns task detail

### SC-TASK-02: Create Task from AI Todo
1. AI extracts todos from recording transcription
2. Client confirms todo via audio-api -> task-rpc: `TaskCreate` with audio_id and ai_task_id
3. Task created with link to source recording

### SC-TASK-03: Google Calendar Webhook Sync
1. Google sends `POST /api/v1/tasks/google/webhook` with channel_id, resource_id
2. task-api identifies the user via calendar_webhook table
3. task-api fetches incremental changes from Google Calendar API using sync_token
4. For each changed event:
   - New event: `BatchUpsertGoogleCalendarSync` (creates task + sync record)
   - Updated event: Update existing task fields
   - Deleted event: `BatchSoftDeleteGoogleCalendarSync` (soft-delete sync record + cancel task)
5. Updates sync_token for next incremental sync

### SC-TASK-04: Sync Task to Google Calendar
1. Client triggers sync for a specific todo: `POST /records/:audio_id/todos/:todo_id/sync/google` (via audio-api)
2. Backend creates Google Calendar event using stored OAuth tokens
3. Creates google_calendar_sync record linking local task to Google event
4. Returns success

### SC-TASK-05: Batch Move Tasks to Folder
1. Client sends `POST /api/v1/tasks/batch/move` with task_ids and folder_id
2. task-api -> audio-rpc (folder service): `BatchMoveRecordToFolder`
3. Folder relations updated for all tasks

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-TASK-01 | MUST | Task CRUD MUST support title, content, priority, start_time, due_time, remind_time, repeat, repeat_end_date, location, participants | Test: all fields persisted and returned |
| FR-TASK-02 | MUST | Task list MUST support pagination (page, page_size), folder filtering, and status filtering | Test: correct pagination and filtering |
| FR-TASK-03 | MUST | Task list response MUST include aggregated counts: total_task_count, in_progress_count, completed_count, overdue_count | Test: counts match actual task states |
| FR-TASK-04 | MUST | Task search MUST support keyword matching on title and content fields | Test: keyword matches return correct results |
| FR-TASK-05 | MUST | Batch delete MUST accept multiple task IDs and soft-delete all | Test: multiple tasks soft-deleted |
| FR-TASK-06 | MUST | Google Calendar webhook MUST process incremental sync events (create, update, delete) | Test: events correctly mapped to task operations |
| FR-TASK-07 | MUST | Google Calendar sync records MUST track sync_direction and sync_status | Test: fields correctly set |
| FR-TASK-08 | MUST | BatchUpsertGoogleCalendarSync MUST create tasks for new Google events with event fields | Test: tasks created from Google events |
| FR-TASK-09 | MUST | BatchSoftDeleteGoogleCalendarSync MUST soft-delete sync records and optionally update task status to cancelled (3) | Test: sync records deleted; task status updated |
| FR-TASK-10 | MUST | Calendar webhook records MUST track channel_id, resource_id, and expires_at | Test: webhook record persisted |
| FR-TASK-11 | MUST | Task move to folder MUST update folder_relation for the task | Test: folder relation created/updated |
| FR-TASK-12 | SHOULD | Batch task creation SHOULD support multiple tasks in a single request and return success/fail counts | Test: batch create returns correct counts |
| FR-TASK-13 | MUST | Webhook endpoint MUST be publicly accessible (no JWT auth) | Test: webhook without auth succeeds |
| FR-TASK-14 | SHOULD | Task association with audio SHOULD be queryable via FindTaskByAudioIdAndAiTaskId | Test: linked task found by audio_id + ai_task_id |
| FR-TASK-15 | MUST | Soft-delete tasks by audio_id MUST mark all tasks for that recording as deleted | Test: all linked tasks soft-deleted |

## 6. State Model

### Task Status
```
0 (pending) -> 1 (in_progress) -> 2 (completed)
                                -> 3 (cancelled)
                                -> (is_deleted soft-delete)
```

### Google Calendar Sync Status
```
pending -> synced -> (soft-deleted)
             |
             +--> failed
```

### Calendar Webhook Status
```
1 (active) -> 0 (expired/deleted)
```
- Webhooks have `expires_at` timestamp; must be renewed before expiry

## 7. Data Contract

### REST API Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| GET | `/api/v1/tasks` | Yes | `?page=&page_size=&folder_id=&status=` | `{list, total, page, page_size, total_page, total_task_count, in_progress_count, completed_count, overdue_count}` |
| POST | `/api/v1/tasks` | Yes | `{title, content, priority, start_time, due_time, remind_time, repeat, tag, location, participants}` | `{detail: TaskDetail}` |
| PUT | `/api/v1/tasks/:id` | Yes | `{title?, content?, priority?, ...}` | `{}` |
| DELETE | `/api/v1/tasks/:id` | Yes | -- | `{}` |
| DELETE | `/api/v1/tasks` | Yes | `{task_ids: [int64]}` | `{success_count}` |
| GET | `/api/v1/tasks/:id` | Yes | -- | `{detail: TaskDetail}` |
| POST | `/api/v1/tasks/:id/move` | Yes | `{folder_id}` | `{}` |
| POST | `/api/v1/tasks/batch/move` | Yes | `{task_ids, folder_id}` | `{}` |
| POST | `/api/v1/tasks/google/webhook` | No | Google webhook payload | `{}` |

### gRPC Service: `task` (21 RPCs)

Key RPCs:
- `TaskList`, `TaskCreate`, `TaskBatchCreate`, `TaskUpdate`, `TaskDelete`, `SearchTasks`, `GetTaskById`
- `FindTaskByAudioIdAndAiTaskId`
- `BatchGetGoogleCalendarSync`, `GetGoogleCalendarSyncByLocalEventId`
- `BatchUpsertGoogleCalendarSync`, `BatchSoftDeleteGoogleCalendarSync`
- `InsertGoogleCalendarSync`, `UpdateGoogleCalendarSync`
- `CreateCalendarWebhook`, `GetCalendarWebhooksByAuthId`, `BatchSoftDeleteCalendarWebhooksByAuthId`
- `BatchDeleteTasks`, `BatchSoftDeleteTasksByAudioId`
- `BatchSoftDeleteGoogleCalendarSyncByUserId`

### TaskDetail (proto message)
```protobuf
message TaskDetail {
  int64 id = 1;
  int64 audio_id = 3;
  optional int64 ai_task_id = 19;
  string title = 4;
  string content = 5;
  string source = 6;
  int64 remind_time = 7;
  string repeat = 8;
  optional int64 repeat_end_date = 18;
  string priority = 9;
  string tag = 10;
  int64 folder_id = 11;
  string folder_name = 12;
  int64 created_at = 13;
  int64 updated_at = 14;
  int64 status = 15;
  int64 start_time = 16;
  int64 due_time = 17;
  optional string location = 20;
  repeated string participants = 21;
}
```

### Database Tables

| Table | Key Columns |
|-------|-------------|
| `task` | id, user_id, audio_id, ai_task_id, title, content, tag[], start_time, due_time, reminder_time, repeat, priority, location, participants[], status, is_deleted |
| `google_calendar_sync` | user_id, local_event_id, google_calendar_id, google_event_id, sync_direction, sync_status |
| `calendar_webhook` | user_id, auth_id, provider_calendar_id, channel_id, resource_id, expires_at |

## 8. Error Handling

| Error | Code | Handling |
|-------|------|---------|
| Task not found | 404 | Return "task not found" |
| Unauthorized (not task owner) | 403 | Return "forbidden" |
| Google Calendar API failure | 502 | Log error; return "calendar sync failed" |
| Webhook verification failure | 400 | Log error; ignore webhook |
| Batch operation partial failure | 200 | Return success_count and fail_count |
| Calendar auth expired | 401 | Trigger token refresh via user service |
| Folder not found | 404 | Return "folder not found" |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| Time fields | Millisecond timestamps (int64) | Proto definitions |
| Batch upsert | Supports multiple sync items per request | `BatchUpsertGoogleCalendarSyncReq` |
| Webhook auth | None (Google sends directly) | Route configured without AuthMiddleware |
| Google Calendar client | `common/third_party/google/` | OAuth + Calendar API |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| Task CRUD | Operations logged with userId, taskId |
| Google Calendar sync | Sync direction, status, event counts logged |
| Webhook processing | channel_id, resource_id, event count logged |
| Batch operation results | success_count, fail_count logged |
| Trace propagation | X-Trace-ID through all RPC calls |
