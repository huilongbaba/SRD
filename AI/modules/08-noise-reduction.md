# Module 08: Audio Noise Reduction

## 1. Purpose

Provide real-time audio noise reduction using the GTCRN (Gated Temporal Convolutional Recurrent Network) deep learning model. This is a tenant-facing endpoint (JWT Bearer auth) that denoises audio files stored in S3.

## 2. Actors

| Actor | Role |
|-------|------|
| Mobile App / Web Frontend | Requests denoising via `POST /audio/denoise/s3` (JWT Bearer) |
| FastAPI Endpoint (audio.py) | Authentication, S3 download/upload orchestration |
| NoisereductionService | GTCRN model singleton for inference |
| AWS S3 | Source and destination for audio files |

## 3. Functional Requirements

| ID | Priority | Description | Verification |
|----|----------|-------------|-------------|
| FR-NR-001 | MUST | The system MUST accept audio denoising requests via `POST /api/v1/audio/denoise/s3` with JWT Bearer authentication | T |
| FR-NR-002 | MUST | The system MUST download the source audio from S3, run GTCRN model inference, and upload the denoised result back to S3 | T |
| FR-NR-003 | MUST | The `NoisereductionService` MUST maintain the GTCRN model as a singleton to avoid repeated model loading | I |
| FR-NR-004 | MUST | The endpoint MUST use JWT Bearer auth (`get_current_tenant`) -- this is the only tenant-facing endpoint in the AI backend | I |
| FR-NR-005 | SHOULD | The denoised audio SHOULD maintain the same sample rate and format compatibility as the input | T |

## 4. Data Contract

### 4.1 Denoise Endpoint

**`POST /api/v1/audio/denoise/s3`**

Authentication: JWT Bearer (`Authorization: Bearer <token>`)

Request:
```json
{
  "s3_url": "s3://bucket/audio/original.m4a",
  "output_s3_key": "audio/denoised/output.wav"
}
```

Response:
```json
{
  "code": 0,
  "msg": "success",
  "data": {
    "denoised_s3_url": "s3://bucket/audio/denoised/output.wav"
  }
}
```

## 5. Processing Flow

```
POST /audio/denoise/s3 (JWT Bearer)
  |
  v
Authenticate tenant via JWT
  |
  v
Download source audio from S3 to temp file
  |
  v
NoisereductionService.denoise(audio_path)
  |  (GTCRN model inference)
  v
Upload denoised audio to S3
  |
  v
Return denoised S3 URL
```

## 6. Configuration

GTCRN model configuration is managed within the `NoisereductionService` singleton. No dedicated environment variables are exposed.

## 7. Error Handling

| Error Condition | Behavior |
|----------------|----------|
| JWT authentication fails | 401 Unauthorized |
| S3 download fails | Exception, error response |
| GTCRN model inference fails | Exception, error response |
| S3 upload fails | Exception, error response |

## 8. Performance Considerations

- **Singleton model**: The GTCRN model is loaded once and reused across requests, avoiding expensive model initialization
- **Synchronous execution**: Runs in the FastAPI thread pool (not Celery) since it is a user-facing, latency-sensitive endpoint
- **Memory**: GTCRN model stays in memory for the lifetime of the FastAPI process

## 9. Dependencies

| Dependency | Direction | Description |
|-----------|-----------|-------------|
| `noisereduction_service.py` | Internal | GTCRN model singleton |
| AWS S3 (boto3) | External | Audio file I/O |

## 10. Open Issues

| Issue | Status | Notes |
|-------|--------|-------|
| No audio format validation | Minor | Accepted formats are not explicitly validated before inference |
| No rate limiting | Risk | As the only tenant-facing endpoint, it is exposed to potential abuse |
