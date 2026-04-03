# Module 04 -- Transcription Proxy

> Generated: 2026-04-02

## 1. Overview

### Objective
Proxy transcription, summary generation, and related AI operations between the mobile client and the external AI service. This module is a logical subset of the audio-api service, focused on AI content generation and retrieval.

### Scope
- Trigger transcription for uploaded audio (S3 URL -> AI service)
- Retrieve transcription detail (lines, speakers, timestamps)
- Edit transcription lines and speaker names
- Summary generation (standard, one-pager, custom templates)
- Report generation and regeneration
- AI agent scenes and generation
- Save AI message as summary
- Suggested questions
- Batch operations (batch trigger AI analysis, batch report generation)

### Non-Scope
- Audio file storage and upload (audio service presign flow)
- SSE chat and insight streaming (covered in audio service)
- Export document generation (worker service)
- AI processing itself (AI service)

## 2. Definitions

| Term | Definition |
|------|-----------|
| transcription | Speech-to-text result with speaker diarization, timestamps, and embedding vectors |
| summary | AI-generated text summary of a recording, rendered as HTML |
| report | AI-generated analysis report (may include multiple templates) |
| agent | AI-generated domain-specific analysis (e.g., meeting action items) |
| template_id | int64 identifier for a summary template (unified from string to int64 in 8b71ad7) |
| summary_language | ISO language code for summary generation; null/auto uses conversation language |
| TemplateNameValue | Custom type for flexible string/number JSON deserialization of template names (dd6567a) |

## 3. System Boundary

| This module handles | Other modules handle |
|--------------------|---------------------|
| Forwarding requests to AI service | AI processing (transcription, LLM inference) |
| Enriching requests with tenant_id, language, template_id | Audio upload and storage |
| Returning AI responses to client | SSE streaming (audio service module) |
| Transcription line/speaker editing proxy | Task extraction from transcription (task service) |

## 4. Scenarios

### SC-TRANS-01: Trigger Transcription After Upload
1. After upload complete, audio-api sends `POST /records/transcription/s3` to AI service
2. Request includes: `{s3_url, tenant_id (userId), audio_id}`
3. Auth: `X-Internal-API-Key` header
4. AI service creates conversation record, dispatches Celery chain
5. Returns `{conversation_id, status: processing}`
6. AI service sends notification via Redis when complete

### SC-TRANS-02: Summary Generation with Custom Template
1. Client sends `POST /api/v1/records/:id/summary` with `{template_id: <int64>, summary_language?}`
2. audio-api enriches with `template_ids: [default, onepager, custom]`, `tenant_id`, `summary_language`
3. Forwards to AI service `POST /records/{audio_id}/summary`
4. AI generates up to 3 summaries in parallel (default, one-pager, custom)
5. Returns summaries with HTML content
6. audio-api records template usage via template_community-rpc

### SC-TRANS-03: Transcription Line Edit
1. Client sends `PUT /api/v1/records/:id/transcription/:line_id` with updated text
2. audio-api proxies to AI service
3. AI service updates the transcription line

### SC-TRANS-04: Save AI Message as Summary
1. Client sends `POST /api/v1/records/messages/save` with message content
2. audio-api forwards to AI service `POST /records/{id}/chat/save-as-summary`
3. Uses `TemplateNameValue` type for `TemplateName` field (handles both string and number)

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-TRANS-01 | MUST | System MUST trigger AI transcription after upload complete with s3_url, tenant_id, audio_id | Test: AI service receives correct request |
| FR-TRANS-02 | MUST | Transcription retrieval MUST return lines with speaker labels, timestamps, and text | Test: response structure matches expected format |
| FR-TRANS-03 | MUST | Summary generation MUST support template_id as int64 | Test: int64 template_id accepted and forwarded |
| FR-TRANS-04 | MUST | Summary generation MUST support specifying summary_language (ISO code) | Test: language forwarded to AI; null defaults to auto |
| FR-TRANS-05 | MUST | AI requests MUST be enriched with tenant_id from JWT userId | Test: AI service receives correct tenant_id |
| FR-TRANS-06 | MUST | Report regeneration MUST trigger a new AI generation for the specified recording | Test: new report created |
| FR-TRANS-07 | MUST | Batch report generation MUST accept multiple recording IDs | Test: multiple reports triggered |
| FR-TRANS-08 | MUST | SaveAIMessage MUST handle TemplateName as both string and number (TemplateNameValue type) | Test: both string and numeric values accepted |
| FR-TRANS-09 | SHOULD | Transcription line edit SHOULD proxy changes to AI service and return updated line | Test: edited line reflected on subsequent retrieval |
| FR-TRANS-10 | SHOULD | Template usage SHOULD be recorded via template_community-rpc after summary generation | Test: usage record created |

## 6. State Model

Stateless proxy. The backend does not maintain transcription/summary state -- all state is managed by the AI service. The backend simply proxies requests and enriches them with user context.

## 7. Data Contract

### REST API Endpoints (Transcription/Summary subset)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/records/:id/detail` | Yes | Get AI detail (transcription, summary, etc.) |
| GET | `/api/v1/records/:id/transcription` | Yes | Get transcription detail |
| PUT | `/api/v1/records/:id/transcription/:line_id` | Yes | Edit transcription line |
| PUT | `/api/v1/records/:id/transcription/:line_id/speaker` | Yes | Edit speaker name |
| POST | `/api/v1/records/:id/report` | Yes | Generate AI report |
| POST | `/api/v1/records/batch` | Yes | Batch trigger AI analysis |
| POST | `/api/v1/records/:id/report/regenerate` | Yes | Regenerate report |
| GET | `/api/v1/records/:id/summaries` | Yes | Get all summaries |
| GET | `/api/v1/records/:id/summaries/:summary_id` | Yes | Get summary by ID |
| POST | `/api/v1/records/:id/summary` | Yes | Generate summary (template) |
| PUT | `/api/v1/records/:id/summary/:summary_id` | Yes | Update summary |
| GET | `/api/v1/records/:id/participants` | Yes | Get recording speakers |
| GET | `/api/v1/records/participants` | Yes | Get speaker naming history |
| GET | `/api/v1/records/:id/agents/scenes` | Yes | AI agent scenes |
| POST | `/api/v1/records/:id/agents` | Yes | Generate AI agent |
| GET | `/api/v1/records/:id/agents` | Yes | Get agent reports |
| GET | `/api/v1/records/:id/agents/:agent_id` | Yes | Get agent by ID |
| PUT | `/api/v1/records/:id/agents/:agent_id` | Yes | Update agent report |
| GET | `/api/v1/records/:id/suggested_questions` | Yes | Suggested questions |
| POST | `/api/v1/records/messages/save` | Yes | Save AI message as summary |
| GET | `/api/v1/templates` | Yes | List templates |

### AI Service Endpoints (proxied)

| Category | AI Endpoint | Protocol |
|----------|------------|----------|
| Transcription | `POST /records/transcription/s3` | REST |
| Transcription | `POST /records/batch/transcription/s3` | REST |
| Record detail | `GET /records/{id}`, `GET /records/{id}/all` | REST |
| Summary | `POST /records/{id}/summary` | REST |
| Report | `GET /reports?id={id}` | REST |
| Agents | `POST /agents/{audio_id}`, `GET /agents/{audio_id}` | REST |
| Templates | `POST /templates/scenes` | REST |
| Save message | `POST /records/{id}/chat/save-as-summary` | REST |
| Search | `GET /records/search` | REST |

## 8. Error Handling

| Error | Code | Handling |
|-------|------|---------|
| AI service unavailable | 502 | Log with aiHost, requestURL, requestMethod; return "service unavailable" |
| AI service timeout | 504 | Log error; return "service timeout" |
| AI service returns error | Pass-through | Forward AI error to client |
| Invalid template_id | 400 | Validate int64 before forwarding |
| Recording not found | 404 | Check record ownership before proxying |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| AI client standard timeout | Configurable per endpoint | YAML config |
| AI client stream timeout | 0 (no timeout) | Stream client |
| AI endpoint count | 50+ | `common/third_party/ai/config.go` |
| AI auth | X-Internal-API-Key header | Internal API key |
| Template ID type | int64 | Unified in 8b71ad7 |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| AI request logging | Enhanced with aiHost, requestURL, requestMethod (dd6567a) |
| AI error logging | Full request context for triage |
| Template usage tracking | Recorded via template_community-rpc |
| Trace propagation | X-Trace-ID forwarded to AI service |
