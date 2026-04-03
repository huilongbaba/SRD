# 07-AI-Chat: Agent Mode (Structured Reports & Web Browsing)

> Module Owner: APP (ai_chat module -- Agent toggle), AI (agents.py, askai_tools, web_searching)
> Last Updated: 2026-04-02
> Parent: [07-ai-chat/README.md](README.md)
> Source Reqs: srd_bitables APP-074 (Agent presets), APP-075 (reports), APP-080 (export), APP-081 (report polling), APP-243 (follow-up)
> Architecture Ref: Architecture/AI/L2_modules.md Sec 1.4 agents.py + Sec 4.4 askai_tools, srd_v1.2_draft "Web Browsing Agent"

---

## 1. Purpose & Scope

Agent Mode enhances both InFileChat and CrossFileChat with structured AI analysis capabilities. When enabled, the AI performs intent recognition and planning before generating responses, can produce structured reports (not just freeform text), and optionally searches the web for external information.

Agent Mode operates as a toggle within the existing chat modes, not as a separate entry point.

**In-scope**: Agent toggle (enable/disable), intent recognition + planning pipeline, preset scene selection, structured report generation, web browsing integration, report export (PDF/Word/Clipboard), report polling.

**Out-of-scope**: Basic chat streaming without Agent (see `infile-chat.md`, `cross-file-chat.md`), template community (separate module).

**Cross-repo boundary**:

```
APP AiChatController (Agent enabled)
  --> BACKEND: POST /api/v1/insight/messages/sse {enable_modes: [agent, web?]}
    --> AI: POST /insights/messages/stream
      --> insight/pipeline.py:
          --> intent/identification.py: LLM intent recognition + tool selection
          --> Conditionally: web_searching.py (Google Serper API)
          --> askai_tools/tools.py: tool execution (search recordings, web search)
          --> response.py: structured report generation
      --> SSE events include reasoning_plan + report_content
    <-- SSE events
  <-- BACKEND SSE proxy
APP renders report in WebView / exports via backend
```

---

## 2. State Model

### 2.1 Agent Mode Toggle States

```
stateDiagram-v2
    [*] --> agent_off : default state
    agent_off --> agent_on : user enables Agent toggle
    agent_on --> agent_on_web : user enables Web toggle (addon)
    agent_on_web --> agent_on : user disables Web toggle
    agent_on --> agent_off : user disables Agent toggle
    agent_on_web --> agent_off : user disables Agent toggle
```

| Toggle | `enable_modes` value | Effect |
|--------|---------------------|--------|
| Agent OFF, Web OFF | `[]` | Direct answer (no intent recognition, no planning) |
| Agent ON, Web OFF | `[1]` (agent mode) | Intent recognition -> planning -> structured response |
| Agent ON, Web ON | `[1, 2]` (agent + web) | Intent recognition -> planning -> optional web search -> structured response |

### 2.2 Agent Processing Pipeline States

```
stateDiagram-v2
    [*] --> message_sent
    message_sent --> intent_recognition : AI receives message
    intent_recognition --> planning : intent identified, tools selected
    planning --> rag_retrieval : plan includes knowledge base search
    planning --> web_search : plan includes web search
    rag_retrieval --> response_generation : evidence gathered
    web_search --> fetch_pages : search results received
    fetch_pages --> snippet_matching : pages fetched
    snippet_matching --> response_generation : relevant snippets extracted
    response_generation --> report_generation : output type is report
    response_generation --> text_streaming : output type is text
    report_generation --> complete : report.gen -> chat.end with report_content
    text_streaming --> complete : chat.end
    complete --> [*]
```

### 2.3 Report Generation States (APP side)

```
stateDiagram-v2
    [*] --> no_report
    no_report --> generating : report.gen SSE event received
    generating --> available : chat.end with report_content
    generating --> failed : failed SSE event
    available --> exporting : user taps Export
    exporting --> export_done : export task completes
    exporting --> export_failed : export task fails
```

### 2.4 Pricing/Access Control

| User Tier | Agent | Web | Limit |
|-----------|-------|-----|-------|
| Free | ON by default | OFF by default | 3 uses/month |
| Paid | ON by default | OFF by default | Unlimited |

---

## 3. Functional Requirements

### 3.1 Agent Preset Scenes (APP-074)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-AGT-001 | The system MUST provide N preset analysis scenes (auto-recommended based on user content and recording context). | MUST | API: `GET /agents` returns available agent functions; UI shows scene cards |
| FR-AGT-002 | The system MUST allow users to select a preset scene to trigger structured analysis. | MUST | UI test: tap scene card -> analysis triggered -> structured report returned |
| FR-AGT-003 | The system SHOULD auto-recommend relevant scenes based on recording content. | SHOULD | Recommendation based on conversation topics and user history |

### 3.2 Report Generation (APP-075)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-AGT-010 | The system MUST generate structured AI analysis reports when Agent mode is enabled. | MUST | Send message with `enable_modes: [1]` -> response includes `report_content` |
| FR-AGT-011 | The system MUST render reports in rich format (HTML with headings, lists, tables). | MUST | `report_content` rendered in WebView; verify formatting preserved |
| FR-AGT-012 | The system MUST display AI reasoning/planning text during processing. | MUST | `reasoning_plan` field shown in "Analyzing..." state before response |
| FR-AGT-013 | The system MUST poll report generation status when report generation is in progress. | MUST | `report.gen` event triggers polling indicator; `chat.end` completes |

### 3.3 Report Export (APP-080)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-AGT-020 | The system MUST support report export to PDF format (backend-generated). | MUST | Export flow: POST /api/v1/exports/{reportId} {target_format: pdf} -> poll -> download |
| FR-AGT-021 | The system MUST support report export to Word format (backend-generated). | MUST | Same flow with target_format for Word |
| FR-AGT-022 | The system MUST support copying report content to clipboard. | MUST | Copy button -> clipboard contains report text |

### 3.4 Web Browsing (APP-350..352)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-AGT-030 | The system MUST support web search when Agent+Web mode is enabled and AI determines external info is needed. | MUST | Send question needing external info with `enable_modes: [1, 2]` -> `source_webs` non-empty in response |
| FR-AGT-031 | The system MUST use Google Serper API for search queries, supporting concurrent multi-query search. | MUST | AI logs: verify Serper API calls with multiple queries |
| FR-AGT-032 | The system MUST concurrently fetch web pages (HTML/PDF, max 32 threads) and extract text content. | MUST | AI logs: verify concurrent page fetching with thread pool |
| FR-AGT-033 | The system MUST match search result snippets to the most relevant web page passages using F1 Score. | MUST | AI logs: verify `SnippetMatch` stage with F1 scoring |
| FR-AGT-034 | The system MUST display web source references in the response. | MUST | `source_webs` in `chat.end` contains web page references with titles and URLs |

### 3.5 Follow-up (APP-243)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-AGT-040 | The system MUST generate smart follow-up questions based on the AI response content. | MUST | `suggested_follow_ups` in `chat.end` is contextually relevant to the report/answer |

---

## 4. Data Contract

### 4.1 API Endpoints (Agent Mode specific)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/insight/messages/sse` | POST | Send message with Agent modes `{message, enable_modes: [1] or [1,2], session_id}` |
| `/api/v1/exports/{reportId}` | POST | Start export task for AI report `{is_transcription: false, target_format}` |
| `/api/v1/exports/{taskId}` | GET | Poll export status |

### 4.2 AI Backend Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `GET /agents` | GET | List available agent analysis functions |
| `POST /agents/{audio_id}` | POST | Execute specific agent analysis on recording |
| `GET /agents/result` | GET | Get single agent result |
| `GET /agents/{audio_id}` | GET | Get all agent results for recording |
| `PUT /agents/{audio_id}` | PUT | Update agent result content |
| `GET /reports?id={id}` | GET | Get report by ID (supports markdown-to-HTML conversion) |

### 4.3 SSE Event Flow (Agent Mode)

```
POST /api/v1/insight/messages/sse
  {message: "Analyze the key risks discussed", enable_modes: [1, 2], session_id: "s123"}

SSE Response (ordered):

1. Reasoning phase:
  data: <encrypted({"type":"delta","delta":"[Analyzing intent...]","session_id":"s123"})>
  data: <encrypted({"type":"delta","delta":"[Planning: search recordings + web]","session_id":"s123"})>

2. Web search phase (if web enabled):
  data: <encrypted({"type":"delta","delta":"[Searching web for risk analysis...]","session_id":"s123"})>

3. Response generation:
  data: <encrypted({"type":"delta","delta":"Based on your recordings and ","session_id":"s123"})>
  data: <encrypted({"type":"delta","delta":"web research, the key risks are:","session_id":"s123"})>

4. Report indicator:
  data: <encrypted({"type":"report.gen"})>

5. Completion:
  data: <encrypted({"type":"chat.end",
    "report_id": 456,
    "session_id": "s123",
    "report_content": "<html>...structured report...</html>",
    "suggested_follow_ups": ["How to mitigate risk #1?", "What timeline was proposed?"],
    "source_audios": [{"id":"rec1", "title":"Q3 Review Meeting"}],
    "source_webs": [{"id":"web1", "title":"Risk Management Best Practices", "url":"https://..."}]
  })>
```

### 4.4 Web Search Pipeline (AI side)

```
web_searching.py:
  Input:  user question + enable_modes includes web
  Step 1: LLM generates 1-3 search queries from user question
  Step 2: Google Serper API call (concurrent queries, auto-retry)
  Step 3: Extract URLs from search results (top 10 per query)
  Step 4: Concurrent page fetch (ThreadPoolExecutor, max 32 workers)
          - HTML: extract text via readability
          - PDF: extract text via PDF parser
  Step 5: F1 Score matching: search snippet vs page full text
          - Find best matching sentence + surrounding context
  Step 6: Package as web evidence for LLM context
  Output: List of {url, title, snippet, matched_passages}
```

---

## 5. Key Scenarios

### Scenario 1: Agent Analysis with Web Search

```
1. User in MemoChat, enables Agent + Web toggles
2. Types: "What are the latest industry trends related to topics in my recent meetings?"
3. POST with enable_modes: [1, 2]
4. AI Intent Recognition: detects "cross-recording analysis + external research" intent
5. Planning: (a) search user recordings for meeting topics, (b) web search for industry trends
6. RAG retrieval: finds meeting recordings with industry discussion segments
7. Web search: generates queries ["industry trends 2026", "tech industry outlook"]
8. Serper API returns results -> concurrent page fetch (HTML/PDF)
9. F1 Score matching extracts relevant paragraphs from web pages
10. LLM generates structured report combining recording insights + web research
11. report.gen event -> chat.end with report_content + source_audios + source_webs
12. APP renders report in WebView with dual source sections (recordings + web)
```

### Scenario 2: Export Agent Report to PDF

```
1. Agent generates report (report_id = 456 in chat.end)
2. User taps Export -> selects PDF
3. APP calls POST /api/v1/exports/456 {is_transcription: false, target_format: pdf}
4. BACKEND dispatches export task to Export Worker via Redis Stream
5. Export Worker fetches report from AI: GET /reports?id=456
6. AI returns HTML content with speaker name replacement
7. Worker: goldmark (Markdown->HTML) -> CSS injection -> wkhtmltopdf (HTML->PDF)
8. PDF uploaded to S3
9. APP polls /api/v1/exports/{taskId} -> status=done, download_url
10. User downloads PDF via CloudFront signed URL
```

### Scenario 3: Per-Recording Agent Analysis

```
1. User in InFileChat for a specific recording, enables Agent toggle
2. Taps preset scene card: "Extract Meeting Action Items"
3. POST /agents/{audio_id} triggers specific agent function
4. AI ConversationService executes function calling:
   - Intent identification from preset scene
   - Argument generation from transcript context
   - Function execution (e.g., extract tasks, generate summary)
5. Result stored as agent report
6. User views structured report with action items
7. Optionally exports to PDF/Word
```

---

## 6. Non-Functional Requirements

| ID | Requirement | Target | Source |
|----|-------------|--------|--------|
| NFR-AGT-001 | Agent mode total response time (intent + retrieval + web + generation) | < 60s typical, < 120s with web search | UX target |
| NFR-AGT-002 | Web search: Serper API call latency | < 3s per query | External SLA |
| NFR-AGT-003 | Web search: concurrent page fetch | max 32 threads, < 10s total | AI web_searching.py config |
| NFR-AGT-004 | Report HTML rendering in WebView | < 2s after data received | UX target |
| NFR-AGT-005 | Export task completion (PDF/Word) | < 60s for typical report | Export Worker SLA |
| NFR-AGT-006 | Free tier: 3 Agent uses per month, enforced server-side | Mandatory | Business rule from SRD note 6 |
| NFR-AGT-007 | SSE stream idle timeout | 120s (shared) | Ticket-001488 fix |

---

## 7. Error Catalogue

| Error Code | Trigger | User-Facing Message | Recovery |
|------------|---------|---------------------|----------|
| E-AGT-001 | Agent processing timeout (Celery hard limit) | "Analysis took too long. Please try a simpler question." | User retries with shorter/simpler query |
| E-AGT-002 | Web search API failure (Serper down) | "Unable to search the web. Responding from recordings only." | Graceful degradation: response generated without web context |
| E-AGT-003 | Web page fetch failure (all pages unreachable) | (Transparent: response generated with available data) | Log error; partial web sources shown |
| E-AGT-004 | Report generation failure | `failed` SSE event with error_detail | Retry button available |
| E-AGT-005 | Export task failure | "Export failed. Please try again." | Re-submit export task |
| E-AGT-006 | Free tier limit reached | "You've used all 3 free Agent analyses this month. Upgrade for unlimited access." | Upgrade prompt; disable Agent toggle until next month |
| E-AGT-007 | Intent recognition failure | AI falls back to direct answer (no Agent pipeline) | Transparent fallback; response still generated |

---

## 8. Observability

| Event Name | Trigger Point | Key Fields | Purpose |
|------------|---------------|------------|---------|
| `agent_mode_toggled` | User toggles Agent or Web switch | `mode` (agent_only/agent_web/off), `context` (infile/crossfile) | Track Agent adoption |
| `agent_analysis_started` | Agent mode message sent | `session_id`, `recording_id?`, `modes_enabled`, `is_preset_scene` | Track Agent usage |
| `agent_analysis_complete` | chat.end with report_content | `session_id`, `response_time_ms`, `has_web_sources`, `report_size_chars` | Track Agent success |
| `agent_analysis_failed` | failed event | `session_id`, `error_type`, `modes_enabled` | Track Agent failures |
| `agent_web_search_executed` | Web search triggered | `session_id`, `query_count`, `results_count` | Track web search volume |
| `agent_report_exported` | Export task completes | `report_id`, `format` (pdf/word) | Track export volume |
| `agent_preset_selected` | User selects preset scene | `scene_name`, `recording_id` | Track preset popularity |
| `agent_free_tier_limit_hit` | Free user reaches 3/month | `user_id` | Track conversion opportunity |

---

## 9. Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| OQ-AGT-1 | Number and types of preset Agent scenes -- what N scenes will be available at launch? | Product + AI | Open |
| OQ-AGT-2 | AI-to-PPT conversion mentioned in SRD flow -- is this in scope for v1.2? | Product | Open |
| OQ-AGT-3 | Web search billing -- does Serper API have per-query cost implications? Rate limits? | Engineering + Finance | Open |
| OQ-AGT-4 | Agent mode for InFileChat vs CrossFileChat -- are the same preset scenes available in both contexts? | Product | Open |
| OQ-AGT-5 | Free tier limit enforcement -- is it per-calendar-month or rolling 30 days? | Product | Open |
| OQ-AGT-6 | Report editing -- can users edit Agent-generated reports like they can edit summaries? | Product | Open |
| OQ-AGT-7 | Save-as-summary -- AI endpoint `POST /records/{id}/chat/save-as-summary` exists. Should Agent reports be saveable as summaries? | Product | Open |

---

## 10. Revision History

| Date | Author | Changes |
|------|--------|---------|
| 2026-04-02 | SRD Rewrite | Initial 10-section format. Source: srd_bitables AI Chat (APP-074/075/080/081/083/243), srd_v1.2_draft AI chat + Web Browsing Agent sections, AI L2 agents.py + web_searching.py + askai_tools, L3 Flow 3 + Flow 4, AiChatMessage model. |
