# Module 09 -- Template Service

> Generated: 2026-04-02

## 1. Overview

### Objective
Provide a template community for browsable, favoritable, and trackable summary templates, enabling users to discover and apply templates to AI-generated summaries. Combines a local RPC service (favorites/usage data) with AI service proxy (template content discovery and recommendation).

### Scope
- Template listing by scene/type (proxy to AI service)
- Template recommendation based on recording topics (proxy to AI service)
- Template discovery with keyword search (proxy to AI service)
- Favorites management (toggle, list -- local RPC)
- Usage tracking (record usage, list by time/frequency -- local RPC)
- Global usage statistics aggregation (local RPC)

### Non-Scope
- Template content storage and management (AI service)
- Summary generation using templates (audio service / AI service)
- Template creation/editing (not exposed via API)

## 2. Definitions

| Term | Definition |
|------|-----------|
| template_id | int64 identifier for a template (unified from string to int64) |
| template_type | Classification: NORMAL (per-recording summary), SKILL (insight/global) |
| scene | Category/domain of templates (e.g., Meeting, Interview, Lecture) |
| favorite | User bookmark on a template (per language) |
| usage tracking | Recording each time a user applies a template |
| template_use_agg | Per-user aggregated template usage data |
| template_use_stats | Global per-template usage statistics |
| keyword search | Server-side title search via AI service's TemplatesDiscover endpoint |

## 3. System Boundary

| This service handles | Other services handle |
|--------------------|---------------------|
| Favorites management (local DB) | Template content storage (AI service) |
| Usage tracking (local DB) | Template recommendation algorithm (AI service) |
| Proxy to AI for template listing/discovery | Summary generation with template (audio service) |
| Global usage statistics | |

### Architecture
```
Client -> template_community-api -> template_community-rpc (favorites, usage)
                                 -> AI Service (template listing, discover, recommend, scenes)
```

## 4. Scenarios

### SC-TMPL-01: Browse Recommended Templates
1. Client sends `POST /api/v1/template-community/recommend/list` with `{audio_id?, language, keyword?, page, page_size}`
2. tc-api proxies to AI service: `POST /templates/recommend` with `{tenant_id, audio_id?, language, keyword?}`
3. AI service:
   - If audio_id provided: extracts topics from recording segments, ranks scenes by topic frequency
   - If audio_id null: uses recent 50 conversations for topic extraction
   - Falls back to General scene if no matches; fills with random scenes up to 30
   - Optional keyword filtering via title search
4. tc-api enriches response with favorites data from tc-rpc: `BatchCheckFavoriteTemplateIDs`
5. tc-api enriches with global usage counts: `SumTemplateUseCounts`
6. Returns paginated ranked template list

### SC-TMPL-02: Toggle Favorite
1. Client sends `POST /api/v1/template-community/favorite` with `{template_id, template_type, language}`
2. tc-api -> tc-rpc: `UpsertFavorite` (if not favorited) or `SoftDeleteFavorite` (if already favorited)
3. Returns success

### SC-TMPL-03: Record Template Usage
1. After summary generation with a template, audio-api -> tc-rpc: `RecordTemplateUsage`
2. tc-rpc writes usage log, updates per-user aggregate, increments global stats
3. Optional batch recording: `RecordTemplateUsageBatch` for multiple uses

### SC-TMPL-04: View Usage History (by day)
1. Client sends `POST /api/v1/template-community/usage/day` with `{start_time_ms, end_time_ms, language}`
2. tc-api -> tc-rpc: `ListUsageLogsByRange`
3. Returns usage logs within the specified time range

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-TMPL-01 | MUST | Template listing MUST proxy to AI service and return templates with scene/type classification | Test: templates returned with scene metadata |
| FR-TMPL-02 | MUST | Template recommendation MUST support both audio_id-based (Normal) and conversation-history-based (Skill) modes | Test: different results for with/without audio_id |
| FR-TMPL-03 | MUST | Keyword search MUST filter templates by title via AI service TemplatesDiscover endpoint | Test: keyword filter returns matching templates |
| FR-TMPL-04 | MUST | Favorite toggle MUST create or soft-delete a favorite record per user+template_id+language | Test: toggling creates/removes favorite |
| FR-TMPL-05 | MUST | Favorite list MUST return paginated template IDs for the user's language | Test: correct IDs with pagination |
| FR-TMPL-06 | MUST | Usage recording MUST write usage log, update user aggregate, and increment global stats | Test: all three data stores updated |
| FR-TMPL-07 | MUST | Batch usage recording MUST accept multiple items with unified sync flags | Test: batch items recorded correctly |
| FR-TMPL-08 | MUST | Usage list (by frequency) MUST sort by global total_use_count | Test: results ordered by global count |
| FR-TMPL-09 | MUST | Usage list (by recency) MUST sort by user's last_used_at | Test: results ordered by recent use |
| FR-TMPL-10 | MUST | Usage by day MUST return logs within specified time range (start_time_ms, end_time_ms) | Test: only logs in range returned |
| FR-TMPL-11 | MUST | Template IDs MUST be int64 throughout the system | Code review: no string template IDs |
| FR-TMPL-12 | SHOULD | Template list responses SHOULD include favorite status and global usage count enrichment | Test: favorite flag and count present |
| FR-TMPL-13 | SHOULD | Global usage stats SHOULD aggregate across all languages per template_id | Test: SumTemplateUseCounts returns cross-language totals |

## 6. State Model

### Favorite State
```
not_favorited -> favorited (UpsertFavorite) -> soft_deleted (SoftDeleteFavorite) -> favorited (re-UpsertFavorite)
```

### Usage Tracking
Append-only log model. No state transitions -- each usage event is recorded independently.

## 7. Data Contract

### REST API Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | `/api/v1/template-community/detail` | Yes | `{template_id}` | Template detail (from AI) |
| POST | `/api/v1/template-community/scenes` | Yes | `{}` | Scene list (from AI) |
| POST | `/api/v1/template-community/list` | Yes | `{scene_id?, type?, language, page, page_size}` | Paginated template list |
| POST | `/api/v1/template-community/recommend/list` | Yes | `{audio_id?, language, keyword?, page, page_size}` | Ranked template list |
| POST | `/api/v1/template-community/favorite/list` | Yes | `{language, page, page_size}` | Favorite template IDs |
| POST | `/api/v1/template-community/favorite` | Yes | `{template_id, template_type, language}` | `{}` |
| POST | `/api/v1/template-community/usage/list` | Yes | `{language, sort, page, page_size}` | Usage aggregate list |
| POST | `/api/v1/template-community/usage/day` | Yes | `{language, start_time_ms, end_time_ms}` | Usage logs by day |

### gRPC Service: `TemplateCommunity` (9 RPCs)

```protobuf
service TemplateCommunity {
  rpc UpsertFavorite(UpsertFavoriteReq) returns (Empty);
  rpc SoftDeleteFavorite(SoftDeleteFavoriteReq) returns (Empty);
  rpc ListFavoriteTemplateIDs(ListFavoriteTemplateIDsReq) returns (ListFavoriteTemplateIDsResp);
  rpc BatchCheckFavoriteTemplateIDs(BatchCheckFavoriteTemplateIDsReq) returns (BatchCheckFavoriteTemplateIDsResp);
  rpc RecordTemplateUsage(RecordTemplateUsageReq) returns (Empty);
  rpc RecordTemplateUsageBatch(RecordTemplateUsageBatchReq) returns (Empty);
  rpc ListUserUsageAgg(ListUserUsageAggReq) returns (ListUserUsageAggResp);
  rpc ListUsageLogsByRange(ListUsageLogsByRangeReq) returns (ListUsageLogsByRangeResp);
  rpc SumTemplateUseCounts(SumTemplateUseCountsReq) returns (SumTemplateUseCountsResp);
}
```

### Key Proto Messages

- **UpsertFavoriteReq**: `{user_id, template_id, template_type, language}`
- **RecordTemplateUsageReq**: `{user_id, template_id, used_at_unix_ms, session_id?, audio_id?, ext_json, language, sync_user_usage?, sync_global_stats?}`
- **ListUserUsageAggReq**: `{user_id, language, sort ("recent"|"count"), page, page_size}`
- **UserUsageAggItem**: `{template_id, last_used_at_ms, total_use_count}` (total_use_count is global)

### Database Tables

| Table | Key Columns |
|-------|-------------|
| `template_community_base` | Template metadata |
| Favorites table | user_id, template_id, language, is_deleted |
| `template_usage_log` | user_id, template_id, language, used_at_ms, session_id, audio_id |
| `template_use_agg` | user_id, template_id, last_used_at_ms |
| `template_use_stats` | template_id, language, total_use_count |

## 8. Error Handling

| Error | Code | Handling |
|-------|------|---------|
| AI service unavailable (template listing) | 502 | Return "template service unavailable" |
| Template not found | 404 | Return "template not found" |
| Invalid template_id | 400 | Validate int64 before processing |
| Favorite toggle failure | 500 | Log error; return "operation failed" |
| Usage recording failure | 500 | Log error (non-blocking -- should not affect summary generation) |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| Template ID type | int64 | Unified in 8b71ad7 |
| Favorite uniqueness | user_id + template_id + language | Composite key |
| Global stats aggregation | Cross-language SUM per template_id | `SumTemplateUseCounts` |
| Recommendation target count | 30 templates | AI service default |
| AI endpoint (discover) | Supports `keyword` search parameter | `TemplatesDiscover` |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| Template usage tracking | Every usage recorded with timestamp, user, session, audio context |
| Favorite operations | Toggle events logged |
| AI proxy calls | Template listing/discovery requests logged |
| Trace propagation | X-Trace-ID through RPC and AI service calls |
