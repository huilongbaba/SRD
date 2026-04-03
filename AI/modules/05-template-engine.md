# Module 05: Template Engine (Discovery & Recommendation)

## 1. Purpose

Manage summary template discovery, browsing, and content-aware recommendation. The system provides scene-based template organization with a topic-based recommendation algorithm that matches templates to conversation content via a 3-tier fallback strategy.

## 2. Actors

| Actor | Role |
|-------|------|
| Business Backend | Queries templates via `POST /templates/recommend`, `POST /templates/discover` |
| FastAPI Endpoint (templates.py) | Template CRUD, recommendation logic, scene mapping |
| PostgreSQL | Stores templates, template_locales, segments (for topic extraction) |

## 3. Functional Requirements

### 3.1 Template Discovery & Browsing

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-TE-001 | MUST | The system MUST list template scene categories via `POST /templates/scenes` | T |
| FR-TE-002 | MUST | The system MUST support browsing templates by scene/type with pagination via `POST /templates/discover` | T |
| FR-TE-003 | MUST | The system MUST support keyword search (by title via `crud_template_locale.search_template_ids_by_title()`) integrated into both `/discover` and `/recommend` endpoints | T |
| FR-TE-004 | MUST | The system MUST retrieve templates by IDs via `POST /templates` | T |
| FR-TE-005 | MUST | The system MUST retrieve template detail via `POST /templates/detail` | T |
| FR-TE-006 | MUST | The system MUST check template existence via `POST /templates/exists` | T |

### 3.2 Topic-Based Recommendation

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-TE-010 | MUST | For Normal templates (`audio_id` provided), the system MUST extract segment `main_topic` values from the conversation, map them to scene IDs via `_SCENE_NAME_MAP`, rank scenes by topic frequency, and return templates from matching scenes | T |
| FR-TE-011 | MUST | For Skill templates (`audio_id` is null), the system MUST use the tenant's 50 most recent conversations' topics for scene ranking | T |
| FR-TE-012 | MUST | The recommendation MUST implement a 3-tier fallback: (1) topic-matched scenes, (2) "General" scene (id=1), (3) random other scenes up to `target_count=30` | T |
| FR-TE-013 | MUST | The system MUST deduplicate templates via `_deduplicate_templates()` | T |
| FR-TE-014 | MUST | Templates within each scene MUST be shuffled using `random.Random(str(conversation_id))` (or `tenant_id` for skill templates) for deterministic variety per conversation | T |
| FR-TE-015 | MUST | The `_SCENE_NAME_MAP` (15-entry scene ID-to-name mapping in en/zh) MUST serve as the shared taxonomy source for both recommendation and segment topic classification in `analysis_conversation.py` | I |
| FR-TE-016 | MUST | The system MUST support Normal (per-recording summary) and Skill (cross-recording) template types | T |
| FR-TE-017 | MUST | The system MUST support multi-language locale (en, zh) for template content | T |

### 3.3 Template Resolution for Summary

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-TE-020 | MUST | The `_get_default_normal_template_id()` function MUST resolve the default template for summary generation | T |
| FR-TE-021 | MUST | The `_get_normal_template_content()` function MUST load template content from `template_locales` for the matching language code | T |

## 4. Data Contract

### 4.1 Recommendation Endpoint

**`POST /api/v1/templates/recommend`**

Request:
```json
{
  "tenant_id": "uuid-string",
  "audio_id": 12345,
  "language": "en",
  "keyword": "meeting",
  "page": 1,
  "page_size": 10
}
```

Response:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "templates": [
      {
        "template_id": 42,
        "title": "Meeting Minutes",
        "scene_name": "Business Meeting",
        "type": "normal"
      }
    ],
    "total_count": 25,
    "total_page": 3
  }
}
```

### 4.2 Scene Name Map

The `_SCENE_NAME_MAP` is a 15-entry dictionary mapping scene IDs to bilingual names (en/zh). It serves dual purpose:

1. **Recommendation**: Maps detected segment topics to scene IDs for template retrieval
2. **Topic Classification**: Used by `analysis_conversation.py` as the taxonomy for segment `main_topic` classification

This replaced the previous 553-line `dialogue_taxonomy_en_full.json` file, reducing maintenance surface and ensuring alignment between classification and recommendation.

## 5. Processing Flow

```
POST /templates/recommend {tenant_id, audio_id?, language, keyword?}
  |
  v
audio_id provided?
  |
  +-- Yes (Normal templates):
  |     Get conversation segments
  |     Extract main_topic from each segment
  |     _rank_scene_ids_from_topics():
  |       Map topic names -> scene IDs via _SCENE_NAME_MAP
  |       Rank by frequency, break ties by first occurrence
  |     Fetch templates per ranked scene (type=NORMAL)
  |
  +-- No (Skill templates):
        Get 50 most recent conversations for tenant
        Extract topics from conversations
        _rank_scene_ids_from_topics()
        Fetch templates per ranked scene (type=SKILL)
  |
  v
3-Tier Fallback:
  Tier 1: Topic-matched scene templates
  Tier 2: "General" scene (id=1) if empty
  Tier 3: Random other scenes up to target_count=30
  |
  v
_deduplicate_templates()
  |
  v
keyword provided? -> Filter by title search
  |
  v
Paginate and return
```

## 6. Configuration

No dedicated environment variables. The `_SCENE_NAME_MAP` and `target_count=30` are hardcoded constants in `templates.py`.

## 7. Error Handling

| Error Condition | Behavior |
|----------------|----------|
| Conversation not found (audio_id) | Returns empty recommendation list |
| No segments with topics | Falls back to General scene, then random fill |
| Template locale not found | Graceful skip |
| Keyword search yields no results | Returns empty page |

## 8. Performance Considerations

- **Multiple DB queries per scene**: The recommendation function may execute up to 15 DB queries (one per scene). Under high concurrency, this creates significant DB load
- **Deterministic shuffle**: Uses seeded random for consistent ordering without DB-level sorting
- **Pagination**: Applied after deduplication and keyword filtering, server-side

## 9. Dependencies

| Dependency | Direction | Description |
|-----------|-----------|-------------|
| `analysis_conversation.py` | Consumer | Imports `_SCENE_NAME_MAP` for topic classification (circular dependency risk) |
| `crud_template`, `crud_template_locale` | Internal | Template data access |
| `crud_segment`, `crud_conversation` | Internal | Topic extraction for recommendation |

## 10. Open Issues

| Issue | Status | Notes |
|-------|--------|-------|
| Circular dependency: algorithm layer -> API layer | Active | `analysis_conversation.py` imports `_SCENE_NAME_MAP` from `templates.py` via deferred import. Should be moved to `app/utils/constants.py` |
| No caching of scene-template mappings | Risk | High-concurrency recommendation creates many DB queries. Consider per-tenant cache |
| Hardcoded target_count=30 | Minor | Should be configurable |
