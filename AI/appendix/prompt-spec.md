# Appendix B: Key LLM Prompt Templates

## 1. Overview

All LLM prompts are centrally managed in `app/utils/prompts.py` via the `PromptBase` class and `PromptName` enum. Templates are accessed via `PromptBase.get_template(PromptName.XXX)` and formatted via `PromptBase.format_template()`.

## 2. Prompt Catalog

### 2.1 Conversation Analysis

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `ONE_IN_ALL_SYSTEM_PROMPT` | `analysis_conversation.py` | System prompt for dialogue segmentation + topic classification |
| `ONE_IN_ALL_USER_PROMPT` | `analysis_conversation.py` | User prompt with taxonomy + transcript |
| `ITEM_EXTRACTION_SYSTEM_PROMPT` | `analysis_conversation.py` | System prompt for task/note/memory extraction |
| `ITEM_EXTRACTION_USER_PROMPT` | `analysis_conversation.py` | User prompt with transcript + time context |

### 2.2 Summary Generation

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `MARKDOWN_SUMMARY_UNIFIED_SYSTEM_PROMPT` | `summary_template/summary.py` | System prompt for unified markdown summary |
| `MARKDOWN_SUMMARY_UNIFIED_USER_PROMPT` | `summary_template/summary.py` | User prompt with template instructions + transcript + related conversations |
| `SUMMARY_ONEPAGE_SYSTEM_PROMPT` | `summary_template/summary.py` | System prompt for One-pager JSON card generation |
| `SUMMARY_BLOCK_ONEPAGE_SYSTEM_PROMPT` | `summary_template/summary.py` | System prompt for block-level One-pager |
| `SUMMARY_DEFAULT_SYSTEM_PROMPT` | `summary_template/summary.py` | Default summary system prompt |
| `SUMMARY_USER_PROMPT` | `summary_template/summary.py` | Summary user prompt (basic) |
| `SUMMARY_WITH_TASKS_USER_PROMPT` | `summary_template/summary.py` | Summary user prompt with extracted tasks context |
| `SUMMARY_WITH_TEMPLATE_USER_PROMPT` | `summary_template/summary.py` | Summary user prompt with template instructions |
| `SUMMARY_WITH_TEMPLATE_AND_TASKS_USER_PROMPT` | `summary_template/summary.py` | Summary user prompt with both template and tasks |

### 2.3 AI Insight (Memochat)

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `AI_INSIGHT_INTENT_AND_TOOL_SELECTION_PROMPT` | `insight/intent/identification.py` | Intent recognition + tool selection (streaming, with ---JSON--- delimiter) |
| `CONVERSATION_INSIGHT_INTENT_AND_TOOL_SELECTION_PROMPT` | `insight/intent/identification.py` | Per-recording conversation intent (simplified) |
| `AI_INSIGHT_CANNOT_ANSWER_PROMPT` | `insight/response.py` | Response when query cannot be answered |
| `AI_INSIGHT_CLARIFICATION_PROMPT` | `insight/response.py` | Clarification request prompt |
| `AI_INSIGHT_CHAT_ANSWER_EXPLANATION_PROMPT` | `insight/response.py` | Response generation with RAG context |
| `AI_INSIGHT_CHAT_ANSWER_REPORT_PROMPT` | `insight/response.py` | Detailed report generation |
| `AI_INSIGHT_CHAT_ANSWER_FIRST_TURN_TITLE_PROMPT` | `insight/response.py` | Title generation for first message |
| `AI_INSIGHT_CHAT_TOPIC_BLOCK_SUMMARY_PROMPT` | `insight/prepare.py` | Session topic block summarization |
| `AI_INSIGHT_CHAT_GEN_TITLE_PROMPT` | `insight/pipeline.py` | Session title generation |
| `GENERATE_WEB_SEARCHING_QUERIES_PROMPT` | `askai_tools/tools.py` | Web search query generation |

### 2.4 Per-Recording Chat

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `CONVERSATION_MEMOCHAT_SYSTEM_PROMPT` | `insight/pipeline.py` | System prompt for per-recording chat |
| `CONVERSATION_MEMOCHAT_USER_PROMPT` | `insight/pipeline.py` | User prompt with transcript + summary + tasks |
| `CHAT_IN_CONVERSATION_SYSTEM_PROMPT` | `conversation_service.py` | Legacy chat system prompt |

### 2.5 Function Calling / Agents

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `FUNCTION_CALLS_IDENTIFY_PROMPT` | `process_execution.py` | Function call intent identification |
| `MUST_FUNCTION_CALLS_PROMPT` | `process_execution.py` | Force function call execution |
| `DRAFT_EMAIL_SYSTEM_PROMPT` | `process_execution.py` | Email drafting agent |
| `DEEP_RESEARCH_SYSTEM_PROMPT` | `process_execution.py` | Deep research agent |
| `DECISION_SUPPORT_SYSTEM_PROMPT` | `process_execution.py` | Decision support agent |
| `AGENT_EXECUTION_USER_PROMPT` | `process_execution.py` | Agent execution user prompt |

### 2.6 Report & HTML Generation

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `HTML_DOC_WRITING_SYSTEM_PROMPT` | Multi-agent pipeline | HTML document generation |
| `HTML_EMAIL_WRITING_SYSTEM_PROMPT` | Multi-agent pipeline | HTML email generation |
| `MARKDOWN_WRITER_AGENT_SYSTEM_PROMPT` | Multi-agent pipeline | Markdown writing agent |
| `HTML_DESIGNER_AGENT_SYSTEM_PROMPT` | Multi-agent pipeline | HTML design agent |
| `RETRIEVER_AGENT_SYSTEM_PROMPT` | Multi-agent pipeline | Retrieval agent |
| `COORDINATOR_AGENT_SYSTEM_PROMPT` | Multi-agent pipeline | Multi-agent coordinator |

### 2.7 Knowledge Graph & Moments

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `SEGMENTS_RELATIONSHIP_PROMPT` | `graph_tasks.py` | Cross-segment relationship analysis |
| `RECAP_SUMMARY_PROMPT` | `moment_service.py` | Dashboard recap summary |
| `KEYPOINTS_GENERATION_PROMPT` | `moment_service.py` | Key points extraction for dashboard |
| `AI_MEANINGFUL_LINE_PROMPT` | `moment_service.py` | Meaningful quote selection |
| `USER_PROFILE_EXTRACTION_SYSTEM_PROMPT` | Profile extraction | User profile analysis system prompt |
| `USER_PROFILE_EXTRACTION_USER_PROMPT` | Profile extraction | User profile analysis user prompt |

### 2.8 Query & Retrieval

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `QUERY_PLANNER_PROMPT` | `retrieval_service.py` | Knowledge graph query planning |
| `QUERY_ANSWER_PROMPT` | `retrieval_service.py` | Knowledge graph query answering |
| `QUERY_INTENT_RECOGNITION_PROMPT` | Legacy intent pipeline | Query intent classification |
| `DEEP_SEARCH_RESPONSE_PROMPT` | Legacy deep search | Deep search response generation |
| `ORIGIN_STORY_PROMPT` | Origin tracing | Event origin story generation |
| `EVENT_ORIGIN_PROMPT` | Origin tracing | Event origin identification |

### 2.9 Legacy Insight

| Prompt Name | Used By | Purpose |
|-------------|---------|---------|
| `AI_INSIGHT_FUNCTION_CALLS_IDENTIFY_PROMPT` | `askai_tasks.py` | Legacy function call identification for insight |
| `AI_INSIGHT_WRITING_SYSTEM_PROMPT` | `askai_tasks.py` | Legacy insight response writing |

## 3. Key Prompt Design Patterns

### 3.1 Streaming Intent + JSON Delimiter

The insight intent recognition prompt uses a `---JSON---` delimiter to separate streaming reasoning from structured output:

```
[Reasoning chunks streamed to client in real-time]
---JSON---
{
  "is_valid": true,
  "response_format": "explanation",
  "selected_tools": ["search_recording_facts"],
  "entities": ["project alpha", "Q4 review"],
  "time_range": {"start": "2026-03-01", "end": "2026-03-31"}
}
```

### 3.2 Summary Language Instruction

All summary-related prompts include a language instruction:

```
Generate a summary in {language}
```

Where `{language}` is the human-readable name (e.g., "English", "Chinese") resolved via `from_lang_code_to_name()`.

### 3.3 Source Integrity Constraints

Summary prompts include explicit constraints to prevent related conversation content from contaminating the main summary:

- `PRIMARY_SOURCE`: Main conversation is the primary source
- `RELATED_CONTEXT_USAGE`: Related conversations provide supplementary context only
- `main_summary_basis`: Summary body must be based on the main conversation
- `related_section_position`: Related content goes in a separate section
- `related_conversation_details_inside_main_body`: Prohibited

### 3.4 Taxonomy Injection

The segmentation prompt injects the 15-entry `_SCENE_NAME_MAP` as the `Taxonomy` input, replacing the previous 553-line external JSON file. This ensures topic classification aligns with the template recommendation system.

## 4. LLM Call Patterns

| Pattern | Function | Used For |
|---------|----------|----------|
| `chat_with_llm()` | Synchronous single-turn chat | Markdown summary, per-recording chat |
| `chat_with_llm_stream()` | Streaming chat (generator) | Intent recognition, response generation |
| `ask_llm_for_json_response()` | JSON-structured output | One-pager card generation, item extraction |
| `func_call_with_llm()` | Function calling mode | Agent intent identification, legacy insight |
| `text_batch_embedding()` | Batch embedding generation | Transcript lines, summary blocks, atoms |
