# 07-AI-Chat: InFileChat (Single-File AI Q&A)

> Module Owner: APP (ai_chat module, per-recording context)
> Last Updated: 2026-04-02
> Parent: [07-ai-chat/README.md](README.md)
> Source Reqs: srd_bitables APP-083 (AI Chat streaming), srd_v1.2_draft "InfileChat"
> Architecture Ref: Architecture/APP/L2_modules.md Sec 3.3, Architecture/AI/L2_modules.md Sec 1.1 records.py

---

## 1. Purpose & Scope

InFileChat provides AI-powered Q&A within the context of a single recording. Users can ask questions about the recording content, and the AI responds using the transcription as context. It appears as the third tab in the transcription detail page.

**In-scope**: Per-recording chat, SSE streaming within single-file context, follow-up question suggestions, message history, retry/reconnect, source tracing to transcript lines.

**Out-of-scope**: Cross-file context (see `cross-file-chat.md`), Agent mode features (see `agent-mode.md`), chat session management (InFileChat is implicitly tied to the recording).

**Cross-repo boundary**:

```
APP AiChatController
  --> BACKEND: POST /api/v1/records/{audioId}/messages/stream
    --> AI: POST /records/{audio_id}/chat/stream
      --> AI ConversationService: per-recording chat with transcript context
      --> LLM streaming response
    <-- SSE events
  <-- BACKEND SSE proxy pass-through
```

---

## 2. State Model

### 2.1 InFileChat Page States

```
stateDiagram-v2
    [*] --> loading_history : page opens
    loading_history --> empty_chat : no history messages
    loading_history --> chat_loaded : history messages exist
    empty_chat --> thinking : user sends first message
    chat_loaded --> thinking : user sends message
    thinking --> streaming : first delta chunk received
    streaming --> message_complete : chat.end event
    streaming --> stream_error : timeout or network error
    stream_error --> reconnecting : auto-reconnect
    reconnecting --> streaming : reconnect success
    reconnecting --> message_failed : reconnect fails
    message_complete --> chat_loaded : ready for follow-up
    message_failed --> chat_loaded : error acknowledged
```

### 2.2 Thinking Status Enum (APP UI states)

| Status | Visual | Trigger |
|--------|--------|---------|
| `thinkingStart` | Pulsing "Thinking..." animation | Message sent, waiting for first chunk |
| `thinkingAnalyzing` | "Analyzing..." with reasoning text | Reasoning/plan chunks arriving |
| `thinkingGenerating` | Streaming text appearing in bubble | Response delta chunks arriving |
| `thinkingDone` | Complete message displayed | `chat.end` event received |
| `thinkingFail` | Red error card | `failed` event or timeout |

---

## 3. Functional Requirements

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-IFC-001 | The system MUST allow users to ask questions about the current recording's transcription content. | MUST | Send question about transcript content -> AI responds with contextually relevant answer |
| FR-IFC-002 | The system MUST stream AI responses via SSE with real-time text rendering (delta-by-delta). | MUST | UI test: text appears incrementally during streaming, not all-at-once |
| FR-IFC-003 | The system MUST display AI-suggested follow-up questions after each complete response. | MUST | After `chat.end`, `suggested_follow_ups` array is non-empty and rendered as tappable chips |
| FR-IFC-004 | The system MUST support recommended questions on first entry (before any user message). | MUST | API: `GET /api/v1/records/{audioId}/suggested_questions` returns relevant starter questions |
| FR-IFC-005 | The system MUST persist chat history for the recording and reload on re-entry. | MUST | Exit and re-enter chat -> previous messages displayed |
| FR-IFC-006 | The system MUST support retry for failed messages. | MUST | On `thinkingFail`, retry button calls `GET /api/v1/records/{audioId}/messages/stream/reconnect` |
| FR-IFC-007 | The system MUST auto-reconnect when SSE stream is interrupted mid-response. | MUST | Simulate network blip during streaming -> stream resumes from last received chunk |
| FR-IFC-008 | The system MUST support clearing all chat history for the recording. | MUST | Delete all -> `DELETE /api/v1/records/{audioId}/messages` -> chat reset to empty state |
| FR-IFC-009 | The system MUST display source references from the transcript when AI cites specific passages. | MUST | `source_audios` in `chat.end` payload -> rendered as tappable source cards linking to transcript positions |
| FR-IFC-010 | The system MUST check network connectivity before sending a message; display error if offline. | MUST | Offline test: attempt to send -> shows "No network connection" (not generic error) |

---

## 4. Data Contract

### 4.1 API Endpoints (InFileChat-specific)

| Endpoint | Method | Direction | Purpose |
|----------|--------|-----------|---------|
| `/api/v1/records/{audioId}/messages/stream` | POST | APP->BE->AI | Send message, receive SSE stream `{message, enable_modes[]}` |
| `/api/v1/records/{audioId}/messages/stream/reconnect` | GET | APP->BE->AI | Reconnect to interrupted stream |
| `/api/v1/records/{audioId}/messages` | GET | APP->BE->AI | Fetch message history `{page, page_size}` |
| `/api/v1/records/{audioId}/messages` | DELETE | APP->BE->AI | Clear all chat messages |
| `/api/v1/records/{audioId}/suggested_questions` | GET | APP->BE->AI | Get recommended starter questions |

### 4.2 AI Backend Endpoints

| Endpoint | Method | AI Module | Purpose |
|----------|--------|-----------|---------|
| `POST /records/{audio_id}/chat/stream` | POST | `records.py` | SSE streaming per-recording chat |
| `GET /records/{audio_id}/chat/reconnect` | GET | `records.py` | Reconnect to chat stream |
| `POST /records/{audio_id}/chat/retry` | POST | `records.py` | Retry failed chat message |
| `GET /records/{audio_id}/chat/messages` | GET | `records.py` | Get chat messages |
| `DELETE /records/{audio_id}/chat/{message_id}` | DELETE | `records.py` | Delete a chat message |

### 4.3 SSE Event Flow (InFileChat)

```
POST /records/{audioId}/messages/stream
  {message: "What were the key decisions?", enable_modes: []}

SSE Response:
  data: <Base64(AES({"type":"delta","delta":"The key","session_id":"abc123"}))>
  
  data: <Base64(AES({"type":"delta","delta":" decisions were:","session_id":"abc123"}))>
  
  data: <Base64(AES({"type":"chat.end","report_id":null,"session_id":"abc123",
    "report_content":null,
    "suggested_follow_ups":["What was the timeline discussed?","Who is responsible?"],
    "source_audios":[{"id":"seg1","title":"Decision segment","url":"memoket://..."}],
    "source_webs":[]}))>
```

---

## 5. Key Scenarios

### Scenario 1: First Question on a Recording

```
1. User opens transcription detail, taps AI Chat tab
2. AiChatController checks for existing history (getAiChatMessageList with audioId)
3. No history -> display recommended questions (getAiChatRecommendList)
4. User taps recommended question "Summarize the main topics"
5. AiChatController.sendAiChatMessage() called
6. POST /api/v1/records/{audioId}/messages/stream with {message, enable_modes: []}
7. SSE stream begins -> thinking animation shown
8. Delta chunks arrive -> text rendered incrementally in chat bubble
9. chat.end arrives -> follow-up suggestions displayed
10. Message persisted server-side; available on re-entry
```

### Scenario 2: SSE Timeout and Reconnection

```
1. User sends complex question requiring >30s AI processing
2. SSE stream established, timeout set to 120s (post Ticket-001488 fix)
3. AI is processing (RAG retrieval + LLM reasoning)
4. At 120s with no chunks: TimeoutException fires in onError
5. APP sets pendingStreamReconnect = true
6. APP calls GET /api/v1/records/{audioId}/messages/stream/reconnect
7. AI checks Redis Stream for pending events
8. If events exist: stream resumes from last event
9. If no events: AI still processing -> new SSE stream waits for events
10. User sees "Reconnecting..." then response resumes
```

### Scenario 3: Follow-up Question Loop

```
1. AI completes first response with suggested_follow_ups: ["How long was the meeting?", "What action items were identified?"]
2. Follow-up chips displayed below response bubble
3. User taps "What action items were identified?"
4. Message sent as new POST to same audioId endpoint
5. AI uses conversation context (previous Q&A + transcript) for richer answer
6. New follow-ups generated -> cycle continues
```

---

## 6. Non-Functional Requirements

| ID | Requirement | Target | Source |
|----|-------------|--------|--------|
| NFR-IFC-001 | SSE stream idle timeout | 120s | Ticket-001488 fix |
| NFR-IFC-002 | First delta chunk latency (time from send to first visible text) | < 10s for simple questions, < 60s for complex RAG queries | UX target |
| NFR-IFC-003 | Message history page load | < 2s for 20 messages | Pagination: page_size=10 |
| NFR-IFC-004 | AES decryption per chunk | < 5ms | Performance critical for smooth streaming |
| NFR-IFC-005 | Chat context = full transcript of current recording | Mandatory | Per-recording scope |

---

## 7. Error Catalogue

| Error Code | Trigger | User-Facing Message | Recovery |
|------------|---------|---------------------|----------|
| E-IFC-001 | SSE timeout (120s no data) | "Connection timed out. Reconnecting..." | Auto-reconnect via `/reconnect` endpoint |
| E-IFC-002 | Network loss during streaming | "Connection lost. Will retry when online." | Auto-reconnect on `NetworkConnectivityService` recovery |
| E-IFC-003 | AI processing failure | Error detail from `failed` event | Retry button available on error card |
| E-IFC-004 | No network at send time | "No internet connection. Please check your network settings and try again." | User must restore network and retry |
| E-IFC-005 | AES decryption failure (corrupted chunk) | (Silent: synthetic chat.end sent) | Logged; user can retry if response incomplete |
| E-IFC-006 | Reconnect endpoint returns no data | "Unable to resume. Please try again." | User manually retries from chat input |

---

## 8. Observability

| Event Name | Trigger Point | Key Fields | Purpose |
|------------|---------------|------------|---------|
| `infile_chat_message_sent` | User sends message | `recording_id`, `message_length`, `is_follow_up`, `agent_enabled` | Track chat engagement |
| `infile_chat_response_complete` | chat.end received | `recording_id`, `response_time_ms`, `has_sources`, `follow_up_count` | Track AI response quality |
| `infile_chat_response_failed` | failed event or timeout | `recording_id`, `error_type` (timeout/network/ai_error) | Track failure rate |
| `infile_chat_reconnect` | Reconnection triggered | `recording_id`, `reconnect_reason` | Track stream reliability |
| `infile_chat_follow_up_tapped` | User taps follow-up suggestion | `recording_id`, `suggestion_index` | Track follow-up engagement |

---

## 9. Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| OQ-IFC-1 | Should InFileChat support message deletion (individual messages)? AI endpoint exists but APP does not expose it. | Product | Open |
| OQ-IFC-2 | Should InFileChat display AI reasoning/planning text (currently only in Agent mode)? | Product | Open |
| OQ-IFC-3 | Timeout value (120s) -- should this use heartbeat mechanism instead of fixed timeout? (Ticket-001488 architectural feedback) | Engineering | Open |
| OQ-IFC-4 | Rate limiting for chat messages -- is there a per-user or per-recording limit? | Product + Backend | Open |

---

## 10. Revision History

| Date | Author | Changes |
|------|--------|---------|
| 2026-04-02 | SRD Rewrite | Initial 10-section format. Source: srd_v1.2_draft InFileChat section, ai_chat_api.dart code (SSE parsing, 30s->120s timeout), ai_chat_model.dart, L3 Flow 3, Ticket-001488 analysis. |
