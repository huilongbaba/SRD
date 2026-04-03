# Module 07: Voiceprint Identification

## 1. Purpose

Extract speaker voiceprint embeddings from audio recordings using an ONNX model, enabling cross-recording speaker identification. Voiceprints are matched against a tenant-level speaker registry to automatically assign confirmed speaker names.

## 2. Actors

| Actor | Role |
|-------|------|
| Celery Worker (default queue) | Executes `process_voiceprints` task |
| ONNX Runtime | Voiceprint embedding extraction model |
| torchaudio | Audio loading and preprocessing |
| PostgreSQL | Stores voiceprints in Speaker and Participant tables |

## 3. Functional Requirements

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-VP-001 | MUST | The system MUST extract voiceprint embeddings from audio segments using the ONNX model | T |
| FR-VP-002 | MUST | The system MUST match extracted voiceprints against the tenant-level `speakers` registry | T |
| FR-VP-003 | MUST | The system MUST update `participants` with matched speaker identities | T |
| FR-VP-004 | MUST | The system MUST run voiceprint extraction in parallel with the analysis chain via Celery `chord()` | T |
| FR-VP-005 | MUST | The system MUST use the `temp_file_path` returned by `process_audio` to access the audio file | T |
| FR-VP-006 | SHOULD | The system SHOULD segment audio by speaker `started_offset_ms`/`ended_offset_ms` from transcript lines, with a minimum segment duration of 8000ms (`MIN_SEGMENT_DURATION_MS`) | I |
| FR-VP-007 | SHOULD | The system SHOULD use DBSCAN clustering (eps=0.4) for grouping similar voiceprint embeddings | I |
| FR-VP-008 | SHOULD | The system SHOULD target 16kHz sampling rate for audio preprocessing | I |

## 4. Data Contract

### 4.1 Celery Task

```python
process_voiceprints(
    audio_path: str,
    tenant_id: str,
    conversation_id: int,
    trace_id: str = None
)
```

**Queue**: `celery` (default)
**ignore_result**: `True`
**soft_time_limit**: `840` seconds (14 minutes)
**time_limit**: `900` seconds (15 minutes)

### 4.2 Database Operations

| Table | Operation | Fields |
|-------|-----------|--------|
| `speakers` | READ/CREATE | Tenant-level speaker registry with voiceprint embeddings |
| `participants` | UPDATE | Match voiceprint to confirmed speaker name |
| `lines` | READ | Get transcript line timestamps for audio segmentation |

## 5. Processing Flow

```
process_voiceprints (runs in parallel with analysis chain)
  |
  v
Load audio from temp_file_path (torchaudio)
  |
  v
Get transcript lines for conversation
  |
  v
Segment audio by speaker timestamps
  (minimum segment duration: 8000ms)
  |
  v
For each segment:
  ONNX model inference -> voiceprint embedding
  |
  v
DBSCAN clustering (eps=0.4) of embeddings
  |
  v
Match clusters against tenant speaker registry
  |
  v
Update participants with matched speaker identities
```

## 6. Configuration

| Constant | Value | Description |
|----------|-------|-------------|
| `MIN_SEGMENT_DURATION_MS` | `8000` | Minimum audio segment duration for voiceprint extraction |
| `TARGET_SR` | `16000` | Target sample rate for audio preprocessing |
| `DBSCAN_EPS` | `0.4` | DBSCAN clustering radius for voiceprint grouping |

## 7. Error Handling

| Error Condition | Behavior |
|----------------|----------|
| Audio file not found | Task returns early with error log |
| ONNX model inference fails | Error logged; voiceprint extraction skipped for that segment |
| soft_time_limit (840s) exceeded | `SoftTimeLimitExceeded` raised; graceful shutdown |
| time_limit (900s) exceeded | Worker forcefully terminates the task |

## 8. Performance Considerations

- **Parallel execution**: Runs via `chord()` alongside the analysis chain, not blocking summary generation
- **Time limits**: 14-minute soft limit + 15-minute hard limit prevent runaway tasks
- **Segment filtering**: Minimum 8-second segments ensure sufficient audio data for reliable voiceprint extraction
- **ONNX Runtime**: Optimized inference compared to PyTorch for production deployment

## 9. Dependencies

| Dependency | Direction | Description |
|-----------|-----------|-------------|
| `process_audio` | Upstream | Provides `temp_file_path` for audio access |
| `crud_speaker` | Internal | Speaker registry lookup |
| `crud_participant` | Internal | Participant identity update |
| `crud_line` | Internal | Transcript line timestamps |

## 10. Open Issues

| Issue | Status | Notes |
|-------|--------|-------|
| ONNX model path configuration | Minor | Model path appears to be hardcoded |
| No voiceprint enrollment API | Enhancement | Speakers are registered implicitly; no explicit enrollment workflow |
