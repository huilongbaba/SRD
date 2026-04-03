# 05 - Transcript Tab

> Module Owner: APP (transcription module)
> Last Updated: 2026-04-02
> Source Reqs: srd_bitables "Transcript Tab" (24 reqs), srd_v1.2_draft Sec "Transcript Tab"
> Architecture Ref: Architecture/APP/L2_modules.md Sec 3.2, Architecture/OVERALL/L3_cross_repo_flows.md Flow 1

---

## 1. Purpose & Scope

The Transcript Tab displays the speech-to-text transcription result for a single recording. It is the primary content view within the transcription detail page, providing synchronized audio-text playback, speaker identification, outline navigation, and export capabilities.

**In-scope**: Transcript text rendering, audio playback with text sync, speaker labeling, outline navigation, waveform preview, transcript export, AI noise reduction toggle, transcription status polling.

**Out-of-scope**: Summary display (see `06-summary.md`), AI chat within file (see `07-ai-chat/infile-chat.md`), recording upload pipeline (see `04-recording`).

**Cross-repo boundary**: Transcription data is produced by the AI backend (ASR pipeline via Celery workers). APP fetches completed transcription via BACKEND REST API. The data flow is:

```
APP (TranscriptionController) --> BACKEND audio-api --> AI FastAPI (GET /records/{audio_id})
```

---

## 2. State Model

### 2.1 Transcription Processing State Machine

```
stateDiagram-v2
    [*] --> not_started : recording uploaded
    not_started --> processing : user taps "Generate" / auto-trigger
    processing --> complete : AI pipeline success (FCM/SSE notification)
    processing --> error : AI pipeline failure
    error --> processing : user taps "Retry"
    complete --> processing : user taps "Regenerate"
```

| State | Code | UI Presentation | Allowed Actions |
|-------|------|-----------------|-----------------|
| `not_started` (uploaded) | 1 | "Generate" button displayed | Generate transcription |
| `processing` | 2 | Purple-blue pulsing ring animation, progress hint | Cancel (leave page), wait |
| `complete` (processed) | 3 | Full transcript text, 3 tabs available (Transcript / Summary / InFileChat) | View, label speakers, export, share, regenerate |
| `error` | -1 | Error message with retry button | Retry, delete |

### 2.2 Audio Player State Machine

```
stateDiagram-v2
    [*] --> init
    init --> downloading : local file absent
    init --> ready : local file exists
    downloading --> ready : download success
    downloading --> download_error : download fail
    download_error --> downloading : retry
    ready --> playing : play
    playing --> paused : pause
    playing --> ready : playback complete
    paused --> playing : resume / speed change
    paused --> ready : stop
    playing --> seeking : drag progress
    seeking --> playing : seek complete
```

### 2.3 State Ownership

| State | Owner | Storage |
|-------|-------|---------|
| Processing status | `TranscriptionController` (`recordStatus`) | `RecordingRepository` (Hive encrypted) |
| Audio player state | `TranscriptionController` (playback part file) | In-memory reactive (GetX `.obs`) |
| Transcript segments | `TranscriptionController` (data part file) | `TranscriptionCacheService` (Hive) + server |
| Selected outline item | `TranscriptionController` | In-memory |

---

## 3. Functional Requirements

### 3.1 Transcription Core (2.1)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-TR-001 | The system MUST convert uploaded audio to text using AI ASR engine (Azure Speech / AssemblyAI fallback). | MUST | API test: POST transcription trigger returns conversation_id; AI logs confirm ASR completion |
| FR-TR-002 | The system MUST support recognition of 10+ languages (zh, en, ja, ko, etc.) with automatic language detection. | MUST | Test with audio files in 3+ languages; verify detected language matches |
| FR-TR-003 | The system MUST identify up to 6 distinct speakers per recording. | MUST | Multi-speaker audio test; verify speaker labels in segments |
| FR-TR-004 | The system MUST allow users to assign custom names to identified speakers (current segment or global scope). | MUST | UI test: rename speaker -> verify name persists across segments when global selected |
| FR-TR-005 | The system SHOULD auto-match speaker names from history for returning speakers. | SHOULD | Voiceprint matching test (beta): same speaker across recordings gets suggested name |
| FR-TR-006 | The system MUST attach precise timestamps (start_time, end_time) to each sentence. | MUST | Data inspection: every `TranscriptionSentence` has non-null `startTime` and `endTime` |
| FR-TR-007 | The system MUST synchronize text highlighting with audio playback position. | MUST | UI test: during playback, current sentence is visually highlighted and auto-scrolled |
| FR-TR-008 | The system MUST support tap-on-text to seek audio to that sentence's timestamp. | MUST | UI test: tap sentence -> audio jumps to `sentence.startTime` |
| FR-TR-009 | The system MUST display real-time transcription status with polling. | MUST | Verify `TranscriptionSummaryPollingService` queries status during processing state |
| FR-TR-010 | The system MUST support regeneration of transcription (with template & language params). | MUST | UI test: Regenerate menu -> confirm -> processing state -> new transcript displayed |
| FR-TR-011 | The system MUST cache transcription detail locally for offline viewing. | MUST | Test: load transcript online -> go offline -> re-enter detail page -> transcript still shown |
| FR-TR-012 | The system MUST support retry on transcription failure (manual button). | MUST | UI test: error state shows retry button; tap retry -> re-enters processing |

### 3.2 Audio Playback (1.5)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-TR-020 | The system MUST support play/pause/stop/seek controls. | MUST | UI test: all 4 controls function correctly |
| FR-TR-021 | The system MUST support playback speed selection: 0.5x, 1x, 1.5x, 2x (default 1x). | MUST | UI test: select each speed -> verify playback rate changes |
| FR-TR-022 | The system MUST download audio from S3 if not cached locally. | MUST | Test: clear local cache -> open detail -> verify download starts -> playback works |
| FR-TR-023 | The system MUST display waveform visualization during playback. | MUST | UI test: waveform widget renders and animates during playback |
| FR-TR-024 | The system SHOULD support AI noise reduction toggle for playback. | SHOULD | UI test: toggle AI denoise -> audio reloads from `denoisedS3Url` |

### 3.3 Outline & Navigation (2.2)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-TR-030 | The system MUST auto-generate an outline with key topic segments. | MUST | Verify `segments` array in `TranscriptionDetailItem` is non-empty for processed recordings |
| FR-TR-031 | The system MUST support tap-on-outline to jump to corresponding audio time. | MUST | UI test: tap outline item -> audio seeks to `segment.startTime` |
| FR-TR-032 | The system MUST highlight the current outline item during playback. | MUST | UI test: during playback, active outline item is highlighted |

### 3.4 Export (1.4)

| ID | Requirement | Priority | Verification |
|----|-------------|----------|-------------|
| FR-TR-040 | The system MUST support full transcript export to PDF/Word format (backend-generated). | MUST | UI test: export -> poll status -> download file -> verify content matches transcript |
| FR-TR-041 | The system MUST display export progress via polling (batch export status API). | MUST | Verify APP polls `POST /api/v1/exports/status/batch` until status=done |

---

## 4. Data Contract

### 4.1 Core Models (APP side)

```dart
// TranscriptionDetailItem
{
  id: String,            // recording ID
  name: String,          // recording name
  s3_url: String,        // original audio S3 URL
  denoised_s3_url: String?, // AI-denoised audio URL (preferred for playback)
  date: String,
  duration: int,         // seconds
  location: String,
  hashtags: List<String>,
  status: int,           // AudioStatusType code
  segments: List<TranscriptionSegment>,
  speaker_version: int?
}

// TranscriptionSegment
{
  id: String,
  level: int,
  title: String,         // topic title
  summary: String,       // segment summary
  hash_tags: List<String>,
  category: String,
  start_time: int,       // seconds
  end_time: int,         // seconds
  sentences: List<TranscriptionSentence>
}

// TranscriptionSentence
{
  id: String,
  speaker: String,       // speaker label (e.g. "Speaker 1" or user-defined name)
  text: String,
  start_time: int,       // seconds
  end_time: int          // seconds
}
```

### 4.2 API Endpoints (APP -> BACKEND -> AI)

| Endpoint | Method | Purpose | Response |
|----------|--------|---------|----------|
| `/api/v1/records/{id}/report/regenerate` | POST | Trigger regeneration | `{audio_id, comment}` |
| `/api/v1/records/presign` | POST | Get S3 presigned URL for upload | `{record_id, presigned_url}` |
| `/api/v1/exports/{reportId}` | POST | Start export task | `{export_task_id}` |
| `/api/v1/exports/status/batch` | POST | Poll export status | `{status, download_url}` |

### 4.3 AI Backend Endpoints (BACKEND -> AI)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `POST /records/transcription/s3` | POST | Trigger transcription from S3 audio |
| `GET /records/{audio_id}` | GET | Fetch transcription detail (segments, sentences) |
| `PUT /records/{audio_id}/speaker` | PUT | Update speaker name assignment |
| `GET /reports?id={id}` | GET | Fetch report content for export |

---

## 5. Key Scenarios

### Scenario 1: View Transcript with Audio Sync

```
1. User opens recording detail page (status = complete)
2. TranscriptionController._initData() fetches from cache; if miss, calls API
3. Transcript segments rendered in scrollable list
4. User taps Play -> audio player starts
5. As audio plays, current sentence highlighted (binary search on timestamps)
6. User taps sentence text -> audio seeks to sentence.startTime
```

### Scenario 2: Speaker Labeling

```
1. User taps speaker label on a sentence
2. Speaker label panel opens with options:
   a. Scope: current segment only OR global (all segments for this speaker)
   b. Name: user types custom name OR picks from recent labels
   c. Beta options: enable voiceprint auto-recognition for future recordings
   d. AI content replacement: optionally replace speaker labels in Summary/Agent content
3. User confirms -> PUT /records/{audio_id}/speaker
4. UI refreshes with new speaker name across affected segments
```

### Scenario 3: Export Transcript to PDF

```
1. User taps Export -> selects PDF format
2. APP calls POST /api/v1/exports/{reportId} {is_transcription: true, target_format: pdf}
3. BACKEND creates export task, dispatches to Export Worker via Redis Stream
4. Export Worker fetches transcript JSON from AI, renders HTML, converts to PDF via wkhtmltopdf
5. PDF uploaded to S3 -> export_task status = done
6. APP polls batch status -> receives CloudFront signed download URL
7. User downloads PDF
```

---

## 6. Non-Functional Requirements

| ID | Requirement | Target | Source |
|----|-------------|--------|--------|
| NFR-TR-001 | AI transcription pipeline MUST complete within soft_time_limit | 300s (Celery) | NFR-8.3 |
| NFR-TR-002 | Transcript detail page MUST load from local cache within | 500ms | Cache-first pattern |
| NFR-TR-003 | Audio-text sync highlight MUST update within | 100ms of playback position change | Real-time UX |
| NFR-TR-004 | Export task MUST complete (PDF generation) within | 60s for typical recording | Export Worker SLA |
| NFR-TR-005 | Transcript data MUST be AES-256-CBC encrypted at rest in Hive | Mandatory | Security requirement |

---

## 7. Error Catalogue

| Error Code | Trigger | User-Facing Message | Recovery |
|------------|---------|---------------------|----------|
| E-TR-001 | ASR provider failure (both Azure + AssemblyAI) | "Transcription failed. Please try again." | Retry button; re-dispatches Celery chain |
| E-TR-002 | Audio file download failure (S3) | "Unable to download audio. Check your connection." | Auto-retry on network recovery |
| E-TR-003 | Notification link broken (SSE/FCM miss) | UI stuck in processing animation | Polling compensation (see Ticket-001493 fix); resume-from-background reconciliation |
| E-TR-004 | Export task timeout | "Export failed. Please try again." | Re-submit export task |
| E-TR-005 | Speaker label save failure (network) | Toast: "Failed to save speaker name" | Local-first with server sync on recovery |

---

## 8. Observability

| Event Name | Trigger Point | Key Fields | Purpose |
|------------|---------------|------------|---------|
| `transcription_triggered` | User taps Generate/Regenerate | `recording_id`, `template_id`, `language` | Track generation requests |
| `transcription_complete` | AI pipeline finishes (via notification) | `recording_id`, `duration_s`, `language_detected` | Track completion rate and latency |
| `speaker_labeled` | User assigns speaker name | `recording_id`, `speaker_id`, `scope` (segment/global), `voiceprint_enabled` | Track speaker labeling usage |
| `transcript_export_started` | User initiates export | `recording_id`, `format` (pdf/word) | Track export volume |
| `audio_playback_started` | User taps play | `recording_id`, `speed` | Track playback engagement |
| `outline_navigated` | User taps outline item | `recording_id`, `segment_id` | Track outline usage |

---

## 9. Open Questions

| # | Question | Owner | Status |
|---|----------|-------|--------|
| OQ-TR-1 | Voiceprint auto-recognition accuracy threshold for production readiness (currently beta) | AI Team | Open |
| OQ-TR-2 | Should transcript text support inline editing by users? (Currently read-only, Note: "Transcript不能编辑") | Product | Decided: read-only in v1.2 |
| OQ-TR-3 | Export formats -- should TXT export be added alongside PDF/Word? | Product | Open |
| OQ-TR-4 | Processing progress indicator -- show estimated time or percentage? (Currently fuzzy state) | UX | Open: "UI详情页加预计时间描述" |

---

## 10. Revision History

| Date | Author | Changes |
|------|--------|---------|
| 2026-04-02 | SRD Rewrite | Initial 10-section format. Consolidated from srd_bitables Transcript Tab (24 reqs) + srd_v1.2_draft Transcript Tab section + Architecture L2/L3 cross-repo flow analysis. |
