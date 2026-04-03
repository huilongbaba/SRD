# Module 06: Knowledge Graph

## 1. Purpose

Build and query a cross-conversation relationship graph that connects segments across different conversations based on semantic similarity. Enables discovery of thematic connections and information flow between recordings.

## 2. Actors

| Actor | Role |
|-------|------|
| Celery Worker (default queue) | Executes `process_graph` task |
| RetrievalService | LLM-based query planning + multi-entity keyword search |
| LLM API | Relationship analysis between segments |
| PostgreSQL | Stores `Relationship` edges between segments |

## 3. Functional Requirements

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-KG-001 | MUST | The system MUST build relationship edges between segments across conversations via the `process_graph` Celery task | T |
| FR-KG-002 | MUST | Relationships MUST be stored in the `relationships` table with source/target segment references | T |
| FR-KG-003 | MUST | The `RetrievalService` MUST provide LLM-based query planning for graph traversal | T |
| FR-KG-004 | MUST | The `RetrievalService` MUST support multi-entity keyword search across the graph | T |
| FR-KG-005 | MUST | Segments MUST have an `is_relationship_analyzed` flag to track whether graph analysis has been performed | I |
| FR-KG-006 | SHOULD | The system SHOULD use `schema_generator.py` to dynamically generate DB schema descriptions for LLM-based query planning | I |

## 4. Data Contract

### 4.1 Celery Task

```python
process_graph(
    tenant_id: str,
    conversation_id: int,
    trace_id: str = None
)
```

**Queue**: `celery` (default)

### 4.2 Relationship Model

| Field | Type | Description |
|-------|------|-------------|
| `tenant_id` | text | Tenant identifier |
| `source_segment_id` | bigint | Source segment |
| `target_segment_id` | bigint | Target segment |
| `relationship_type` | string | Type of relationship |

## 5. Processing Flow

```
process_graph task
  |
  v
Load segments for conversation (where is_relationship_analyzed = false)
  |
  v
For each unanalyzed segment:
  Find related segments in other conversations (embedding similarity)
  |
  v
  LLM analysis: determine relationship type and strength
  |
  v
  Create Relationship records
  |
  v
  Mark segment as is_relationship_analyzed = true
```

## 6. Configuration

No dedicated environment variables. Uses the default Celery queue.

## 7. Error Handling

| Error Condition | Behavior |
|----------------|----------|
| No unanalyzed segments | Task completes without creating relationships |
| LLM analysis fails | Error logged; segment remains unanalyzed for retry |

## 8. Performance Considerations

- Runs on the default Celery queue to avoid blocking transcription/analysis pipelines
- Segment-level granularity reduces the number of relationship comparisons vs line-level

## 9. Dependencies

| Dependency | Direction | Description |
|-----------|-----------|-------------|
| `process_segments_analysis` | Upstream | Segments must exist before graph analysis |
| `retrieval_service.py` | Consumer | Uses graph for query planning |
| `schema_generator.py` | Internal | DB schema descriptions for LLM |
| `crud_segment` | Internal | Segment data access |

## 10. Open Issues

| Issue | Status | Notes |
|-------|--------|-------|
| Graph query integration with insight pipeline | Enhancement | Currently separate from the main RAG retrieval path |
