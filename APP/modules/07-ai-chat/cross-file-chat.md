# 07-AI-Chat: CrossFileChat / MemoChat (Multi-File AI Q&A)

> Module Owner: APP (ai_chat module + ai_insight module, cross-recording context)
> Last Updated: 2026-04-02
> Parent: [07-ai-chat/README.md](README.md)
> Source Reqs: srd_v1.2_draft "CrossFileChat", "AI chat模块"
> Architecture Ref: Architecture/APP/L2_modules.md Sec 3.5, Architecture/AI/L2_modules.md Sec 1.2 insights.py + Sec 4.2 insight/

---

## 1. Purpose & Scope

CrossFileChat (branded as "MemoChat" in the UI) provides AI-powered Q&A across the user's entire recording library. Unlike InFileChat which is scoped to a single recording, CrossFileChat queries a cross-recording knowledge base using RAG (Retrieval-Augmented Generation) with vector similarity search and reciprocal rank fusion.

**In-scope**: Multi-recording context Q&A, session management (create/list/delete/rename), session history browsing, SSE streaming, RAG-powered evidence retrieval, source attribution to multiple recordings, follow-up suggestions.

**Out-of-scope**: Single-recording chat (see `infile-chat.md`), Agent mode/web search (see `agent-mode.md`), recording upload (see `04-recording`).

**Cross-repo boundary**:

```
APP AiChatController (CrossFileChat mode)
  --> BACKEND: POST /api/v1/insight/messages/sse
    --> AI: POST /insights/messages/stream {message, session_id, tenant_id, enable_modes}
      --> AI insight/pipeline.py: Intent recognition -> RAG retrieval -> Response generation
      --> Background Runner -> Redis Stream -> SSE consumer
    <-- SSE events (same format as InFileChat, encrypted)
  <-- BACKEND SSE proxy pass-through (no timeout)
```

---

## 2. State Model

### 2.1 Session Lifecycle

```
stateDiagram-v2
    [*] --> no_sessions : first use
    no_sessions --> active_session : first message sent (server auto-creates session)
    active_session --> active_session : follow-up messages
    active_session --> idle : 5min inactivity (server-side timeout)
    idle --> new_session : next message creates new session
    active_session --> browsing_history : user opens session list
    browsing_history --> active_session : user selects session
    browsing_history --> no_sessions : user deletes all sessions
```

### 2.2 CrossFileChat Page States

```
stateDiagram-v2
    [*] --> home : enter CrossFileChat
    home --> loading_sessions : check for existing sessions
    loading_sessions --> session_list : sessions exist
    loading_sessions --> empty_home : no sessions
    empty_home --> thinking : user sends message or taps recommendation
    session_list --> session_detail : user selects a session
    session_detail --> thinking : user sends message
    thinking --> streaming : first delta arrives
    streaming --> complete : chat.end event
    streaming --> error : timeout / failure
    error --> reconnecting : auto-reconnect
    reconnecting --> streaming : success
    reconnecting --> failed : reconnect fails
    complete --> waiting_input : ready for next message
    waiting_input --> thinking : user sends follow-up
```

### 2.3 State Ownership

| State | Owner | Storage |
|-------|-------|---------|
| Session list | `AiChatHistoryController` | Server-side (paginated API) |
| Current session messages | `AiChatController` | In-memory + server-side |
| Active session_id | `AiChatController.currentSessionId` | In-memory (received from SSE events) |
| Streaming state | `AiChatController` (ThinkingStatus) | In-memory reactive |

---

## 3. Functional Requirements

### 3.1 CrossFileChat Core

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-CFC-001 | The system MUST allow users to ask questions across all their recordings' content. | MUST | Send question referencing content from 2+ recordings -> AI responds with cross-recording context |
| FR-CFC-002 | The system MUST use RAG (vector similarity + trigram search + RRF fusion) to retrieve relevant evidence from the recording knowledge base. | MUST | AI logs: verify `retrieval/fetchers.py` runs vector search + trigram search; `fusion.py` merges results |
| FR-CFC-003 | The system MUST stream AI responses via SSE with delta-by-delta text rendering. | MUST | Same SSE infrastructure as InFileChat; verify incremental text rendering |
| FR-CFC-004 | The system MUST display source attribution showing which recordings were referenced. | MUST | `source_audios` in `chat.end` contains recording references with titles and links |
| FR-CFC-005 | The system MUST display AI-suggested follow-up questions after each response. | MUST | `suggested_follow_ups` in `chat.end` is non-empty |
| FR-CFC-006 | The system MUST display recommended starter questions on first entry. | MUST | API: `GET /api/v1/insight/suggested_questions` returns cross-recording questions |
| FR-CFC-007 | The system MUST check network status before entering chat; block if offline. | MUST | Offline: show "No network" error and prevent chat entry |

### 3.2 Session Management

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-CFC-010 | The system MUST auto-create sessions when the first message is sent (server-managed). | MUST | Send first message -> `session_id` returned in SSE events |
| FR-CFC-011 | The system MUST support listing all chat sessions with pagination. | MUST | `GET /api/v1/insight/sessions?page=1&page_size=10` returns session list |
| FR-CFC-012 | The system MUST support deleting individual sessions. | MUST | `DELETE /api/v1/insight/sessions/{session_id}` removes session and its messages |
| FR-CFC-013 | The system MUST support loading a session's message history. | MUST | Select session -> `GET /api/v1/insight/sessions/{session_id}?page=1` returns messages |
| FR-CFC-014 | The system MUST support session title display (auto-generated from first message). | MUST | Session list shows meaningful titles derived from conversation content |
| FR-CFC-015 | The system SHOULD allow server-side session expiration after 5 minutes of inactivity. | SHOULD | After 5min idle, next message gets a new session_id |

### 3.3 History & Navigation

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-CFC-020 | The system MUST load historical messages when entering an existing session. | MUST | Check for history on entry -> download and display previous messages |
| FR-CFC-021 | The system MUST support clearing all chat data. | MUST | `DELETE /api/v1/insight` clears all sessions and messages |

---

## 4. Data Contract

### 4.1 API Endpoints (CrossFileChat-specific)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/insight/messages/sse` | POST | Send message, receive SSE stream `{message, enable_modes, session_id}` |
| `/api/v1/insight/{session_id}/stream/resume` | GET | Reconnect to interrupted stream |
| `/api/v1/insight/{session_id}/stream/{message_id}/retry` | GET | Retry failed message |
| `/api/v1/insight/sessions` | GET | List sessions `{page, page_size}` |
| `/api/v1/insight/sessions/{session_id}` | GET | Get session messages `{page, page_size}` |
| `/api/v1/insight/sessions/{session_id}` | DELETE | Delete session |
| `/api/v1/insight/suggested_questions` | GET | Get cross-recording recommended questions |
| `/api/v1/insight/suggested-follow-ups` | GET | Get per-message follow-up suggestions |
| `/api/v1/insight` | DELETE | Clear all chat data |

### 4.2 AI Backend RAG Pipeline

```
insights.py (POST /insights/messages/stream)
  --> Create user_message + pending ai_message in DB
  --> launch_insight_generation() via ThreadPoolExecutor
      --> insight/pipeline.py:
          1. prepare.py: gather recent messages + user profile for context
          2. intent/identification.py: LLM intent recognition + tool selection
          3. retrieval/:
             a. fetchers.py: parallel vector search (pgvector) + trigram search
             b. fusion.py: RRF (Reciprocal Rank Fusion) score merging
             c. postprocess.py: evidence packaging + conversation metadata
          4. response.py: LLM streaming generation with retrieved context
      --> Events written to Redis Stream (XADD)
  --> SSE consumer: XREAD from Redis Stream, pace-controlled delivery
```

### 4.3 Session Data Model

```dart
// SessionHistoryModel (from API)
{
  total_count: int,
  total_page: int,
  chat_sessions: [
    {
      id: int,
      title: String,     // auto-generated from first message
      created_at: String,
      updated_at: String
    }
  ]
}
```

---

## 5. Key Scenarios

### Scenario 1: Cross-Recording Question

```
1. User enters MemoChat from home page
2. APP checks network -> OK
3. APP checks for existing sessions (getAiChatSessionHistory)
4. User types: "What were all the budget discussions across my meetings this month?"
5. POST /api/v1/insight/messages/sse {message: "...", enable_modes: [], session_id: null}
6. AI creates new session, dispatches to insight pipeline
7. Pipeline: intent recognition -> detects "cross-recording search" intent
8. RAG retrieval: vector search across all user's recording embeddings
   - Finds 3 recordings with budget-related segments
   - Trigram search for "budget" keyword matches
   - RRF fusion merges and ranks evidence
9. LLM generates response using retrieved evidence, streaming via Redis Stream
10. APP receives SSE events: thinking -> delta chunks -> chat.end
11. Response shows synthesized budget summary with source_audios linking to 3 recordings
12. Follow-up suggestions: ["What were the final budget numbers?", "Who raised budget concerns?"]
```

### Scenario 2: Session Continuation

```
1. User previously had a conversation in session "Meeting Review" 
2. User opens MemoChat -> session list displayed
3. User taps "Meeting Review" session
4. AiChatHistoryDetailController loads messages for that session
5. Previous Q&A displayed chronologically
6. User types follow-up: "What happened after the Q3 review?"
7. POST /api/v1/insight/messages/sse {message: "...", session_id: existing_id}
8. AI uses session context (previous messages) + recording knowledge base
9. Richer, contextually-aware response generated
```

### Scenario 3: Session Cleanup

```
1. User opens session history list
2. User swipes to delete a session -> DELETE /api/v1/insight/sessions/{id}
3. Session removed from list; messages permanently deleted
4. Alternatively: user clears all -> DELETE /api/v1/insight -> all sessions removed
```

---

## 6. Non-Functional Requirements

| ID | Requirement | Target | Source |
|----|-------------|--------|--------|
| NFR-CFC-001 | SSE stream idle timeout | 120s (shared with InFileChat) | Ticket-001488 fix |
| NFR-CFC-002 | RAG retrieval latency (vector + trigram + fusion) | < 5s for typical knowledge base | AI pipeline target |
| NFR-CFC-003 | Total response time (intent + retrieval + generation) | < 30s typical, < 120s for complex multi-recording queries | UX target |
| NFR-CFC-004 | Session list page load | < 2s for 20 sessions | Pagination |
| NFR-CFC-005 | Redis Stream event delivery pace | 0.04-0.1s per chunk | `insight_stream_buffer.py` pace control |
| NFR-CFC-006 | BACKEND SSE proxy | No timeout (`rest.WithTimeout(0)`) | L3 Flow 3 architecture |

---

## 7. Error Catalogue

| Error Code | Trigger | User-Facing Message | Recovery |
|------------|---------|---------------------|----------|
| E-CFC-001 | SSE timeout (120s idle) | "Connection timed out. Reconnecting..." | Auto-reconnect via `/insight/{session_id}/stream/resume` |
| E-CFC-002 | AI background thread crash | No COMPLETED event -> consumer times out -> FAILED event | Error shown; user retries |
| E-CFC-003 | LLM call timeout in pipeline | `failed` SSE event with error detail | Retry available |
| E-CFC-004 | No recordings in knowledge base | AI returns generic response without sources | Informational: "I don't have enough recordings to answer this" |
| E-CFC-005 | Session not found (expired/deleted) | Server creates new session transparently | Invisible to user |
| E-CFC-006 | Network offline at entry | "No internet connection. Chat requires network." | Block entry; re-check on retry |

---

## 8. Observability

| Event Name | Trigger Point | Key Fields | Purpose |
|------------|---------------|------------|---------|
| `crossfile_chat_message_sent` | User sends message | `session_id`, `message_length`, `is_new_session`, `agent_enabled` | Track cross-file chat usage |
| `crossfile_chat_response_complete` | chat.end received | `session_id`, `response_time_ms`, `source_count`, `follow_up_count` | Track response quality |
| `crossfile_chat_response_failed` | failed event | `session_id`, `error_type` | Track failure rate |
| `crossfile_chat_session_created` | New session auto-created | `session_id`, `trigger` (first_message) | Track session creation |
| `crossfile_chat_session_deleted` | User deletes session | `session_id` | Track session lifecycle |
| `crossfile_chat_source_tapped` | User taps source reference | `session_id`, `source_recording_id` | Track source navigation |
| `crossfile_chat_reconnect` | Reconnection triggered | `session_id`, `reason` | Track stream reliability |

---

## 9. Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| OQ-CFC-1 | Session expiration policy: 5 minutes is mentioned in SRD notes but not confirmed as server implementation. What is the actual timeout? | AI Team | Open |
| OQ-CFC-2 | Should APP display the number of recordings in the user's knowledge base as context? | Product | Open |
| OQ-CFC-3 | Session title: auto-generated from first message. Should users be able to edit session titles? (AI endpoint `PUT /insights/sessions/{id}/title` exists) | Product | Open |
| OQ-CFC-4 | AI error messages are not i18n-compatible (returned in English from AI backend). Should they be mapped to i18n keys on APP side? | Engineering | Open |
| OQ-CFC-5 | Suggested questions API (`/suggested_questions`) -- should questions be refreshed periodically as new recordings are added? | Product | Open |

---

## 10. Revision History

| Date | Author | Changes |
|------|--------|---------|
| 2026-04-02 | SRD Rewrite | Initial 10-section format. Source: srd_v1.2_draft CrossFileChat section, ai_chat_api.dart (SSE endpoints), AI L2 insights.py + insight/ pipeline, L3 Flow 3, session_history_model.dart. |
