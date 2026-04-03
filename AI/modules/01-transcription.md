# Module 01: Transcription Pipeline

## 1. Purpose

Convert raw audio files (from S3) into structured, speaker-diarized, embedding-enriched transcript data. This is the foundational pipeline step that all downstream analysis depends on.

## 2. Actors

| Actor | Role |
|-------|------|
| Business Backend | Triggers transcription via `POST /records/transcription/s3` |
| Celery Worker (transcription queue) | Executes `process_audio` task |
| Azure Speech Services | Primary ASR provider |
| AssemblyAI | Fallback ASR provider |
| OpenAI Embeddings API | Generates 1536-dim vectors per transcript line |
| PostgreSQL | Persists Lines, Participants, Conversation status |

## 3. Functional Requirements

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-TR-001 | MUST | The system MUST download audio from S3 URLs (s3://, https://, http:// schemes) to a local temp file | T |
| FR-TR-002 | MUST | The system MUST validate and convert audio to 16kHz mono WAV via ffmpeg if not already in that format | T |
| FR-TR-003 | MUST | The system MUST route ASR to providers based on language priority: `LANGUAGE_PROVIDER_PRIORITY` maps language codes to ordered provider lists (Chinese defaults to `["azure", "assemblyai"]`) | T |
| FR-TR-004 | MUST | The system MUST automatically fall back to the next ASR provider if the primary fails | T |
| FR-TR-005 | MUST | The system MUST apply punctuation restoration (PunctCapSegModelONNX) to ASR output | T |
| FR-TR-006 | MUST | The system MUST convert Traditional Chinese to Simplified Chinese (OpenCC t2s) when Chinese is detected | T |
| FR-TR-007 | MUST | The system MUST generate 1536-dim embedding vectors for each transcript line via `text_batch_embedding()` at transcription time | T |
| FR-TR-008 | MUST | The system MUST create Participant records for each unique `speaker_id_in_audio` returned by the ASR provider | T |
| FR-TR-009 | MUST | The system MUST bulk-create Line records with `embedding_vector`, `started_offset_ms`, `ended_offset_ms`, `speaker_id_in_audio`, `text`, and `confidence` | T |
| FR-TR-010 | MUST | The system MUST update `conversation.transcription_status` to `"transcribing"` at start and `"complete"` on success | T |
| FR-TR-011 | MUST | The system MUST update `conversation.analysis_status` to `"analyzing"` upon successful transcription completion | T |
| FR-TR-012 | MUST | On failure, the system MUST update both `transcription_status` and `analysis_status` to `"failed"` and clean up the temp file | T |
| FR-TR-013 | SHOULD | The system SHOULD validate the `default_language_code` against `get_supported_languages()` and fall back to `"en"` if unsupported | T |
| FR-TR-014 | SHOULD | The system SHOULD pass confirmed participant names as `keyterms`/`phrases` to the ASR provider for improved recognition accuracy | T |
| FR-TR-015 | SHOULD | The system SHOULD correct audio duration discrepancies (>10% difference between recorded and actual) by adjusting `started_at` | I |
| FR-TR-016 | MUST | The system MUST return `temp_file_path` from `process_audio` so that `process_voiceprints` (running in parallel) can access the audio file | T |

## 4. Data Contract

### 4.1 API Endpoint

**`POST /api/v1/records/transcription/s3`**

Request:
```json
{
  "s3_url": "s3://bucket/key.m4a",
  "tenant_id": "uuid-string",
  "audio_id": 12345,
  "duration_ms": 180000,
  "language_code": "en",
  "template_id": 42,
  "summary_language": "en"
}
```

Response:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "conversation_id": 67890,
    "status": "processing"
  }
}
```

### 4.2 Celery Task Signature

```python
process_audio(
    tenant_id: str,
    conversation_id: int,
    s3_url: str,
    duration_ms: int = 0,
    default_language_code: str = "en",
    trace_id: str = None
) -> str  # returns temp_file_path
```

**Queue**: `transcription`
**ignore_result**: `False`

### 4.3 Database Writes

| Table | Operation | Fields |
|-------|-----------|--------|
| `conversations` | UPDATE | `transcription_status`, `analysis_status`, `language_code`, `transcript_version`, `speaker_version`, `started_at` (duration fix) |
| `participants` | CREATE (per speaker) | `tenant_id`, `conversation_id`, `speaker_id_in_audio` |
| `lines` | BULK CREATE | `tenant_id`, `conversation_id`, `participant_id`, `speaker_id_in_audio`, `started_offset_ms`, `ended_offset_ms`, `text`, `confidence`, `embedding_vector` |

## 5. Processing Flow

```
POST /records/transcription/s3
  |
  v
Create conversation record (DB)
  |
  v
Dispatch Celery chain:
  process_audio (transcription queue)
    -> process_segments_analysis (analysis queue)
      -> process_items_extraction (analysis queue)
        -> process_summary_generation (analysis queue)
          -> process_function_calls (execution queue)
  |
  v [process_audio]
Download S3 -> temp file
  |
  v
Validate language code
  |
  v
Update status -> "transcribing"
  |
  v
ffmpeg convert to 16kHz mono WAV (if needed)
  |
  v
ASR: try primary provider -> fallback if fails
  |
  v
Punctuation restoration (PunctCapSegModelONNX)
  |
  v
Chinese T2S conversion (OpenCC, if detected)
  |
  v
Generate embeddings (text_batch_embedding)
  |
  v
Create Participants (per unique speaker)
  |
  v
Bulk create Lines (with embeddings)
  |
  v
Update status -> "complete" / "analyzing"
  |
  v
Return temp_file_path (for voiceprint task)
```

## 6. Configuration

| Env Variable | Default | Description |
|-------------|---------|-------------|
| `CELERY_BROKER_URL` | (required) | Redis broker URL |
| `CELERY_RESULT_BACKEND` | (required) | Redis result backend URL |

### ASR Provider Configuration

Provider selection is determined by `LANGUAGE_PROVIDER_PRIORITY` in `transcription_algos.py`. The priority map routes languages to ordered provider lists. If the primary provider fails, the system automatically tries the next provider in the list.

### Audio Preprocessing

- Accepted extensions: `.mp3`, `.wav`, `.m4a`, `.aac`, `.flac`, `.ogg`, `.wma`, `.opus`, `.webm`
- Default extension (if unrecognized): `.mp3`
- Target format: 16kHz mono WAV (via ffmpeg)
- Chinese keyword replacements for brand names are applied post-ASR

## 7. Error Handling

| Error Condition | Behavior |
|----------------|----------|
| S3 download fails | Exception raised, status set to `"failed"`, temp file cleaned up |
| ASR primary provider fails | Automatic fallback to next provider |
| All ASR providers fail | Exception raised, status set to `"failed"` |
| Embedding generation fails | Exception raised, status set to `"failed"` |
| DB write fails | Exception raised, status set to `"failed"` |
| Invalid language code | Falls back to `"en"` |

## 8. Performance Considerations

- **Batch embedding**: All transcript lines are embedded in a single batch call to minimize API round-trips
- **Bulk DB insert**: Lines are created via `crud_line.bulk_create()` in a single transaction
- **Temp file cleanup**: Always performed in the `except` block on failure; on success, the file is retained for voiceprint extraction running in parallel

## 9. Dependencies

| Dependency | Direction | Description |
|-----------|-----------|-------------|
| `analysis_tasks.process_segments_analysis` | Downstream | Next step in Celery chain |
| `voiceprint_tasks.process_voiceprints` | Parallel | Runs concurrently via `chord()` |
| `transcription_algos.transcribe` | Internal | Core ASR algorithm |
| `transcription_algos.text_embedding` | Internal | Embedding generation |
| `crud_conversation`, `crud_line`, `crud_participant` | Internal | Database access |

## 10. Open Issues

| Issue | Status | Notes |
|-------|--------|-------|
| No idempotency check | Risk | Duplicate `POST /records/transcription/s3` creates duplicate conversations for the same `audio_id` |
| No ASR cost tracking | Risk | ASR provider usage is not metered per tenant |
| Hardcoded keyword replacements | Technical debt | Chinese homophone brand name replacements are hardcoded in `transcription_algos.py` |
