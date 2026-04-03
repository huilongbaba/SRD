# 07 - AI Chat Module Overview

> Module Owner: APP (ai_chat module + ai_insight module)
> Last Updated: 2026-04-02
> Source Reqs: srd_bitables "AI Chat模块" (6 reqs), srd_v1.2_draft Sec "AI chat模块"
> Architecture Ref: Architecture/APP/L2_modules.md Sec 3.3 + 3.5, Architecture/OVERALL/L3_cross_repo_flows.md Flow 3
> Known Bugs: Ticket-001488 (false "no internet" due to SSE timeout)

---

## 1. Purpose & Scope

The AI Chat module provides conversational AI capabilities across the Memoket app. It operates in three distinct modes, each documented in its own sub-module file:

| Sub-module | File | Context | Entry Point |
|------------|------|---------|-------------|
| **InFileChat** | `infile-chat.md` | Single recording | Transcription detail page, 3rd tab |
| **CrossFileChat (MemoChat)** | `cross-file-chat.md` | Multi-recording knowledge base | Home page AI Chat entry |
| **Agent Mode** | `agent-mode.md` | Structured analysis + web search | Toggle within InFileChat or CrossFileChat |

All three modes share a common SSE streaming infrastructure, message model, and controller architecture.

---

## 2. Shared Architecture

### 2.1 SSE Streaming Pipeline (common to all modes)

```
APP (AiChatController)
  --> POST request (ResponseType.stream) to BACKEND
    --> BACKEND SSE proxy (rest.WithTimeout(0)) to AI FastAPI
      --> AI creates DB records, launches background generation
      --> AI Background Runner writes events to Redis Stream (XADD)
      --> AI SSE consumer reads Redis Stream (XREAD, pace-controlled 0.04-0.1s/chunk)
    <-- SSE events: data: {Base64(AES-128-CBC(JSON))}
  <-- BACKEND passes through SSE events (bypasses AES response encryption)
APP (AiChatApiImpl.parseStreamData)
  --> UTF-8 decode -> buffer -> split on \n\n -> extract data: field
  --> Base64 decode -> AES-128-CBC decrypt -> JSON parse
  --> Dispatch by event type:
      - delta -> append text to chat bubble
      - report.gen -> show report generation indicator
      - chat.end -> finalize message (report_content, sources, follow_ups)
      - failed -> show error with error_detail
```

### 2.2 SSE Event Types

| Event Type | Payload | APP Action |
|------------|---------|------------|
| `delta` | `{type, delta, session_id}` | Append text chunk to streaming bubble |
| `report.gen` | `{type}` | Show "generating report" indicator |
| `chat.end` | `{type, report_id, session_id, report_content, suggested_follow_ups, source_audios, source_webs}` | Finalize message; show follow-up suggestions and sources |
| `failed` | `{type, error_detail}` | Display error; set `thinkingFail` status |
| `chat.onDone` | (stream closed) | Clean up stream subscription |
| `error` (APP-side) | (timeout or exception) | Trigger reconnection logic |

### 2.3 Shared State Model: Chat Session + Message + SSE Connection

```
stateDiagram-v2
    state "SSE Connection States" as SSE {
        [*] --> idle
        idle --> connecting : send message
        connecting --> streaming : first chunk received
        streaming --> streaming : more chunks
        streaming --> completed : chat.end event
        streaming --> error : timeout / network loss
        error --> reconnecting : auto-reconnect
        reconnecting --> streaming : reconnect success
        reconnecting --> failed : reconnect fails
        completed --> idle : ready for next message
        failed --> idle : user acknowledges error
    }

    state "Message States" as MSG {
        [*] --> user_sent
        user_sent --> thinking : waiting for AI
        thinking --> analyzing : reasoning chunks arriving
        analyzing --> generating : response chunks arriving
        generating --> complete : chat.end received
        thinking --> thinking_fail : error/timeout
        complete --> [*]
    }

    state "Session States" as SESSION {
        [*] --> no_session
        no_session --> active : first message sent
        active --> active : follow-up messages
        active --> expired : 5min inactivity (server-side)
        expired --> active : new message creates new session
    }
```

### 2.4 Common Data Models

```dart
// AiChatMessage (shared across all modes)
{
  message_id: int?,
  role: String?,           // "user" | "assistant"
  message: String?,        // user question or AI response text
  version: int?,
  status: int?,            // 0=pending, 1=complete, 2=failed
  created_at: String?,
  updated_at: String?,
  reasoning_plan: String?, // AI reasoning/planning text (Agent mode)
  report_id: int?,         // linked report for structured output
  report_format: String?,  // "markdown" | "html"
  report_content: String?, // full report content
  source_audios: List<AiChatBubbleModel>?,    // referenced recordings
  source_webs: List<AiChatBubbleModel>?,      // referenced web pages
  suggested_follow_ups: List<String>?          // AI-suggested next questions
}

// ChatSession
{
  id: int?,
  title: String?,
  created_at: String?,
  updated_at: String?
}

// AiChatBubbleModel (source reference)
{
  // Represents a source recording or web page cited by AI
  // Fields: id, title, url, snippet, etc.
}
```

### 2.5 Critical NFRs (shared)

| ID | Requirement | Value | Source |
|----|-------------|-------|--------|
| NFR-CHAT-001 | SSE stream idle timeout | **120s** (fixed from 30s in Ticket-001488) | `ai_chat_api.dart:139` |
| NFR-CHAT-002 | AI processing soft limit | 300s (Celery soft_time_limit) | NFR-8.3 |
| NFR-CHAT-003 | SSE reconnection delay | 3s | NFR-8.4 |
| NFR-CHAT-004 | Redis Stream TTL | 600s | NFR-8.4 |
| NFR-CHAT-005 | Redis Stream max length | 5000 events | NFR-8.4 |
| NFR-CHAT-006 | SSE pace control | 0.04-0.1s per chunk | AI `insight_stream_buffer.py` |
| NFR-CHAT-007 | BACKEND SSE proxy timeout | 0 (no timeout) | `rest.WithTimeout(0)` |

### 2.6 Ticket-001488 Summary (SSE Timeout Bug)

**Problem**: SSE stream had a 30-second hard timeout (`stream.timeout(Duration(seconds: 30))`). AI Thinking phase (LLM reasoning + RAG retrieval) commonly exceeds 30s for complex questions. When timeout fires, APP's `onError` does not distinguish `TimeoutException` from real network errors, and the fallback error message is hardcoded to "No internet connection."

**Root cause**: Aggressive timeout + missing error classification + misleading fallback message.

**Fix**: (1) Increase timeout to 120s. (2) Distinguish `TimeoutException` from `HttpException` in `onError`. (3) Change fallback message from "No internet connection" to "Something went wrong. Please try again."

---

## 3. Sub-module Index

| File | Scope | Key Reqs |
|------|-------|----------|
| [`infile-chat.md`](infile-chat.md) | Single-file Q&A, SSE streaming, follow-ups | APP-083 (AI Chat streaming), context = current recording |
| [`cross-file-chat.md`](cross-file-chat.md) | Multi-file MemoChat, session management, history | Context = cross-recording knowledge base, RAG pipeline |
| [`agent-mode.md`](agent-mode.md) | Structured reports, web browsing, PDF/Word export | APP-074 (Agent presets), APP-075 (reports), APP-080 (export) |

---

## 4. Error Handling Strategy (shared)

| Error Type | Detection | APP Behavior | Recovery |
|------------|-----------|--------------|----------|
| SSE timeout (120s idle) | `TimeoutException` in `onError` | Set `pendingStreamReconnect=true` | Auto-reconnect via `/reconnect` or `/retry` endpoint |
| Network loss mid-stream | `SocketException` or stream error | Show "Connection lost" error | Auto-reconnect on network recovery |
| AI processing failure | `failed` SSE event with `error_detail` | Show error detail in red error card | Retry button re-sends message |
| Session expired (5min) | Server returns new session_id | Transparent to user | New session auto-created |
| AES decryption failure | `catch` in parseStreamData | Send synthetic `chat.end` to clean up | Log error; user can retry |

---

## 5. Revision History

| Date | Author | Changes |
|------|--------|---------|
| 2026-04-02 | SRD Rewrite | Initial 10-section format. Module overview + shared architecture extracted from srd_bitables AI Chat (6 reqs), srd_v1.2_draft AI chat section, L3 Flow 3, Ticket-001488 analysis, and ai_chat_api.dart code review. |
