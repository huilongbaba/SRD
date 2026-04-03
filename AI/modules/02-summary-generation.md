# Module 02: Summary Generation Pipeline

## 1. Purpose

Generate structured summaries of audio conversations using configurable templates. Supports two rendering modes (Markdown with source tracing, One-pager HTML card layout) and up to 3 templates generated in parallel. This is the most complex pipeline in the system due to the Celery chain with chord pattern and parallel `asyncio.gather()` execution.

## 2. Actors

| Actor | Role |
|-------|------|
| Business Backend | Triggers summary via transcription chain or manual `POST /records/{audio_id}/summary` |
| Celery Worker (analysis queue) | Executes `process_unified_summary_generation` / `process_summary_generation` |
| LLM API (LiteLLM Router) | Generates markdown summary, JSON card structure |
| OpenAI Embeddings API | Source tracing via embedding similarity |
| PostgreSQL | Reads templates, writes reports |

## 3. Functional Requirements

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-SG-001 | MUST | The system MUST generate summaries for up to 3 templates in parallel via `asyncio.gather()`: default Standard template, One-pager template, and optional custom template | T |
| FR-SG-002 | MUST | For Markdown templates, the system MUST generate markdown content via `chat_with_llm()` and convert to HTML via `markdown_to_html()` | T |
| FR-SG-003 | MUST | For the One-pager template, the system MUST generate a JSON card structure via `ask_llm_for_json_response()`, validate it, apply layout optimization rules, and render to styled HTML | T |
| FR-SG-004 | MUST | The system MUST perform source tracing for Markdown summaries: extract `<p>` and `<li>` tags (30-150 chars), compute embedding similarity with transcript lines, and insert `memoket://outline/{line_index}` links for matches above threshold (0.75) | T |
| FR-SG-005 | MUST | The system MUST resolve `summary_language` correctly: if null/empty/"auto", use `from_lang_code_to_name(conversation.language_code)`; otherwise, convert via `from_lang_code_to_name(summary_language)`. The LLM receives the human-readable name (e.g., "English", "Chinese") | T |
| FR-SG-006 | MUST | The language code-to-name conversion MUST be applied in all four processing locations: `process_segments_analysis`, `process_items_extraction`, `process_unified_summary_generation`, and `_run_generation` (background runner) | T |
| FR-SG-007 | MUST | When a user provides a `template_id` during transcription, the system MUST append it as a third template if it differs from the default template ID | T |
| FR-SG-008 | MUST | Custom template names MUST be resolved from `template_locale.title` (not hardcoded) | T |
| FR-SG-009 | MUST | The default template MUST retain the name "Standard" for frontend compatibility | I |
| FR-SG-010 | MUST | The system MUST create a `Report` record per generated template with `content`, `format`, `name`, `version`, and `status` | T |
| FR-SG-011 | MUST | The system MUST skip automatic summary generation for recordings shorter than 3 minutes (180 seconds) and instead write a "transcript too short" message with full transcript | T |
| FR-SG-012 | MUST | The "transcript too short" skip message MUST be localized to the conversation's language code (10 languages supported: en, zh, zh-Hant, de, ja, fr, es, it, ko, pt) | I |
| FR-SG-013 | SHOULD | The system SHOULD include related conversations (up to 5, from last 30 days, embedding similarity threshold 1.1) in the summary prompt for context | T |
| FR-SG-014 | MUST | Summary prompts MUST include `PRIMARY_SOURCE`, `RELATED_CONTEXT_USAGE`, `main_summary_basis`, `related_section_position`, and `related_conversation_details_inside_main_body` constraints to prevent related conversation content from contaminating the main summary body | I |
| FR-SG-015 | SHOULD | Related conversation date format in prompts SHOULD be simplified to date only (no time range) | I |

## 4. Data Contract

### 4.1 API Endpoint (Manual Trigger)

**`POST /api/v1/records/{audio_id}/summary`**

Request:
```json
{
  "tenant_id": "uuid-string",
  "template_ids": [1, 2, 42],
  "summary_language": "en"
}
```

Response:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "summaries": [
      {
        "template_name": "Standard",
        "format": "markdown",
        "content": "<html>...</html>"
      },
      {
        "template_name": "One Pager",
        "format": "html",
        "content": "<html>...</html>"
      }
    ]
  }
}
```

### 4.2 Celery Task Signatures

**Unified (new)**:
```python
process_unified_summary_generation(
    tenant_id: str,
    conversation_id: int,
    report_id: Optional[int] = None,
    template_ids: List[int] = [],
    language_code: str = None,
    trace_id: Optional[str] = None
)
```

**Legacy**:
```python
process_summary_generation(
    tenant_id: str,
    conversation_id: int,
    report_id: Optional[int] = None,
    template_ids: Tuple[str, List[str]] = [],
    trace_id: str = None
)
```

**Queue**: `analysis`
**ignore_result**: `False`

### 4.3 Database Operations

| Table | Operation | Fields |
|-------|-----------|--------|
| `reports` | CREATE (per template) | `tenant_id`, `conversation_id`, `format`, `content`, `name`, `version`, `status` |
| `reports` | UPDATE (manual trigger) | `content`, `version`, `status` |
| `conversations` | UPDATE (on failure) | `analysis_status` = `"failed"` |
| `template_locales` | READ | Template content + title by locale |

## 5. Processing Flow

```
Input: lines, template_ids, related_conversations, summary_language
  |
  v
Check duration < 3 min? --> Yes: write skip message, return
  |
  No
  |
  v
Resolve summary_language:
  null/auto -> from_lang_code_to_name(conversation.language_code)
  else -> from_lang_code_to_name(summary_language)
  |
  v
Fetch related conversations (embedding similarity, last 30 days, top 5)
  |
  v
asyncio.gather(*template_runs):
  |
  +-- Template 1 (Standard Markdown):
  |     Load template from DB -> Build prompt -> chat_with_llm()
  |     -> Postprocess: markdown_to_html -> extract text blocks
  |     -> Compute embeddings -> cosine similarity with line embeddings
  |     -> Insert memoket:// source links (threshold >= 0.75)
  |
  +-- Template 2 (One Pager HTML):
  |     Build card prompt with card_type registry
  |     -> ask_llm_for_json_response()
  |     -> validate_onepage_json_structure()
  |     -> apply_layout_rules()
  |     -> render_html_from_json()
  |
  +-- Template 3 (Custom, optional):
        Load custom template from DB -> Build prompt
        -> chat_with_llm() -> Postprocess (same as Standard)
  |
  v
Create Report records per template result
  |
  v
Update conversation.summary_json (if applicable)
```

## 6. Configuration

| Env Variable | Default | Description |
|-------------|---------|-------------|
| `SUMMARY_MODEL` | `"azure-gpt-5.1"` | LLM model for markdown summary generation |
| `ONE_IN_ALL_ANALYSIS_MODEL` | `"gpt-5-mini"` | LLM model for conversation analysis/segmentation |
| `ITEMS_EXTRACTION_MODEL` | `"gpt-5"` | LLM model for items extraction |

## 7. Error Handling

| Error Condition | Behavior |
|----------------|----------|
| LLM call fails | Exception caught; if manual trigger, report status set to `"failed"` with error message; if auto, `analysis_status` set to `"failed"` |
| No transcript lines | Summary skipped, skip message written |
| Template not found in DB | Graceful skip for that template |
| Source tracing embedding fails | Summary generated without source links |
| One-pager JSON validation fails | Fallback behavior in rendering |

## 8. Performance Considerations

- **Parallel generation**: Up to 3 templates generated simultaneously via `asyncio.gather()` -- significant wall-clock time savings vs sequential
- **Source tracing cost**: Embedding computation for summary text blocks adds latency; batched to minimize API calls
- **Related conversation lookup**: Uses pre-computed `title_embedding_vector` and `summary_embedding_vector` for efficient similarity search

## 9. Dependencies

| Dependency | Direction | Description |
|-----------|-----------|-------------|
| `process_segments_analysis` | Upstream | Must complete before summary generation |
| `process_items_extraction` | Upstream | Must complete before summary generation |
| `summary_template/summary.py` | Internal | Core summary generation algorithm |
| `summary_template/html_blocks/` | Internal | One-pager card rendering |
| `templates.py` endpoint | Internal | `_get_normal_template_content()` for template resolution |
| `crud_report`, `crud_line` | Internal | Database access |

## 10. Open Issues

| Issue | Status | Notes |
|-------|--------|-------|
| Ticket-001492 (fixed) | Resolved | Summary language code-to-name conversion now applied in all 4 locations. Previously, explicit language codes (e.g., "en") were passed raw to LLM prompts instead of human-readable names ("English") |
| Circular import risk | Active | `analysis_conversation.py` imports `_SCENE_NAME_MAP` from `templates.py` (API layer) at call time. Deferred import mitigates but is fragile under refactoring. Consider moving to `app/utils/constants.py` |
| No cost tracking for parallel LLM calls | Risk | 3 parallel template generations = 3+ LLM calls per recording. No per-tenant budget enforcement |
