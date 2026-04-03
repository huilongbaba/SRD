# 04 - Recording Module (录音模块)

> SRD Version: 1.2 | Last Updated: 2026-04-02

## Module Overview

Recording is the largest and most complex APP module, responsible for the entire audio acquisition lifecycle -- from capturing audio through three distinct sources, through upload to S3, to server-side transcription processing completion.

The module comprises **4 sub-modules** unified by a **core state machine**:

| Sub-Module | File | Scope |
|-----------|------|-------|
| Local Recording | [local-recording.md](local-recording.md) | Phone microphone recording: start/pause/resume/stop, background recording, interruption handling |
| Device Recording | [device-recording.md](device-recording.md) | BLE hardware recording control, Opus audio download via BLE/WiFi, file transfer protocol |
| File Import | [file-import.md](file-import.md) | Audio import from file picker/album/share intent, format validation, transcoding |
| Core State Machine | [state-machine.md](state-machine.md) | `AudioStatusType` lifecycle governing ALL recording items from creation to transcription completion |

## Architecture Position

```
AudioFileController (orchestrator, 4 part-file mixins)
  |-- AudioFileRecordingMixin    --> RecordingService, RecordingFlowService
  |-- AudioFileUploadMixin       --> AudioUploadService, AudioUploadQueueService
  |-- AudioFileListMixin         --> AudioFileListService, AudioDataService
  |-- AudioFileEditMixin         --> RecordingEditService
  |
  |-- shared/services/           (12 recording-pipeline services)
  |-- shared/repositories/       RecordingRepository (Hive + in-memory RxList)
  |-- app/modules/my_device/     BLE device management, DeviceAudioService
```

## Key Code References

| Concept | Source File |
|---------|------------|
| AudioStatusType enum | `shared/repositories/recording_repository.dart:20` |
| AudioSourceType enum | `shared/repositories/recording_repository.dart:13` |
| RecordingService | `shared/services/recording_service.dart` |
| RecordingFlowService | `shared/services/recording_flow_service.dart` |
| RecordingStateManager | `shared/services/recording_state_manager.dart` |
| AudioUploadService | `shared/services/audio_upload_service.dart` |
| AudioUploadQueueService | `shared/services/audio_upload_queue_service.dart` |
| AudioImportService | `shared/services/audio_import_service.dart` |
| AudioTranscodeService | `shared/services/audio_transcode_service.dart` |
| DeviceRecordingControlService | `app/modules/audio_file/services/device_recording_control_service.dart` |
| DeviceAudioService | `app/modules/my_device/models/services/device_audio_service.dart` |
| StatusPollingService | `shared/services/status_polling_service.dart` |

## SRD Requirement Coverage

| Bitable Table | Requirement Count | Sub-Module |
|--------------|-------------------|------------|
| 本地录音 | 10 (APP-001 ~ APP-010) | local-recording.md |
| 设备录音 | 9 (APP-237 ~ APP-248) | device-recording.md |
| 文件导入 | 4 (APP-031, APP-033, APP-034, APP-243) | file-import.md |
| 核心状态流转 | 8 (APP-023 ~ APP-030, APP-243) | state-machine.md |
