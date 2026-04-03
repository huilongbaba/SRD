# 10 - Settings (设置页面总览)

> Module: Application Settings Hub
> Covers: SRD bitables "设置页面" (29 reqs) + 3 sub-modules (integrations, summary-language, vocabulary)
> Last updated: 2026-04-02

---

## 1. Overview

- **Objective**: Provide a centralized settings hub where users manage account, preferences, permissions, integrations, and application configuration.
- **Scope**: The Settings module (`app_setting`) is the largest settings surface in the APP, covering:
  - **Account & Auth** (6.2): Profile editing, account binding, account deletion, privacy, multi-device sync
  - **Legal & Policy** (6.3): Terms of service, privacy policy, cookie policy, agreement consent, update notices
  - **General Settings** (7.1): Language switch, region/timezone, about page, version check
  - **Advanced Settings** (7.2): Environment switch (dev/test/prod), auto-transcription toggle, upload network policy (WiFi only), audio quality, debug mode, log upload
  - **Permissions** (7.3): Microphone, storage, location, notification, permission guidance
  - **Help & Support** (9.2): FAQ, feedback, changelog, video tutorials
  - **Sub-modules** (documented separately):
    - [Integrations](integrations.md) -- Third-party OAuth (Slack/Google/Notion)
    - [Summary Language](summary-language.md) -- AI summary output language preference
    - [Vocabulary](vocabulary.md) -- Custom words and industry termbases
- **Non-scope**:
  - User registration / login flow (Module 01)
  - BLE device settings (Module 03)
  - Onboarding screens (Module 02)
  - Template selection (Module 11)

---

## 2. Sub-Module Navigation

| Sub-Module | File | Req Count | Key Features |
|-----------|------|-----------|--------------|
| [Integrations](integrations.md) | `integrations.md` | 6 (APP-320~325) | Slack/Google/Notion OAuth, backend-mediated auth flow |
| [Summary Language](summary-language.md) | `summary-language.md` | 3 (APP-330~332) | Language picker, global state sharing, failure rollback |
| [Vocabulary](vocabulary.md) | `vocabulary.md` | 4 (APP-335~338) | Custom word list, industry termbases, A-Z grouping |

---

## 3. Completion Status

### V1.0 Requirements (22 items within Settings)

| Status | Count | Percentage |
|--------|-------|-----------|
| Fully Implemented | 17 | 77.3% |
| Partially Implemented | 5 | 22.7% |
| Not Implemented | 0 | 0% |

### V1.0 Partial Implementation Gaps

| Req | Feature | Gap Description | Priority |
|-----|---------|----------------|----------|
| APP-183 | Profile editing (nickname/avatar) | V1.2 planned; now implemented via `my_profile_view.dart` | Done |
| APP-205 | Auto-transcription toggle | V1.2 planned; now implemented in `app_setting_view` | Done |
| APP-207 | Audio quality selector (High/Med/Low) | UI not implemented; UED not started | **P1 Gap** |
| APP-213 | Push notification permission | V1.2 planned; now implemented via `PushNotificationService` + FCM | Done |
| APP-233 | Video tutorials | V1.2 planned; UED not started | P3 |

### V1.2 Sub-Module Requirements (13 items)

| Sub-Module | Total | Implemented | Planned |
|-----------|-------|-------------|---------|
| Integrations (APP-320~325) | 6 | 6 | 0 |
| Summary Language (APP-330~332) | 3 | 3 | 0 |
| Vocabulary (APP-335~338) | 4 | 3 | 1 (APP-338: Industry termbase selection) |

---

## 4. Settings Hub Architecture

```
app_setting/
├── bindings/
│   ├── app_setting_binding.dart          # Main settings page bindings
│   ├── integrations_binding.dart         # Integrations sub-page
│   ├���─ vocabulary_binding.dart           # Vocabulary sub-page
│   └── custom_termbase_binding.dart      # Custom termbase sub-page
├── controllers/
│   ├── summary_language_controller.dart  # Permanent singleton (cross-page)
│   ├── integrations_controller.dart      # Lifecycle-aware OAuth manager
│   ├── vocabulary_controller.dart        # Word list CRUD
│   └── custom_termbase_controller.dart   # Termbase toggle + industry selection
├── services/
│   └── integrations_service.dart         # Integration domain logic
└── views/
    ├── app_setting_view.dart             # Main settings list
    ├── integrations_view.dart            # OAuth cards
    ├── summary_language_bottom_sheet.dart # Language picker
    ├── vocabulary_view.dart              # Word list A-Z
    └── custom_termbase_view.dart         # Termbase hub
```

### Routing

| Route | Page | Binding |
|-------|------|---------|
| `/app-setting` | `AppSettingView` | `AppSettingBinding` |
| `/integrations-view` | `IntegrationsView` | `IntegrationsBinding` |
| `/vocabulary` | `VocabularyView` | `VocabularyBinding` |
| `/custom-termbase` | `CustomTermbaseView` | `CustomTermbaseBinding` |

---

## 5. Main Settings Page Requirements (excluding sub-modules)

| ID | Description | Level | SRD Req | Status | Verification |
|----|------------|-------|---------|--------|-------------|
| FR-ST-001 | System MUST allow users to edit profile (nickname/avatar) | MUST | APP-183 | V1.2 Done | Edit nickname; verify persisted |
| FR-ST-002 | System MUST allow email/phone account binding | MUST | APP-184 | V1.0 Done | Bind email; verify login works |
| FR-ST-003 | System MUST allow permanent account deletion with data purge | MUST | APP-186 | V1.0 Done | Delete account; verify data inaccessible |
| FR-ST-004 | System MUST provide privacy/data permission management | MUST | APP-187 | V1.0 Done | Review privacy settings |
| FR-ST-005 | System MUST sync account data across multiple devices | MUST | APP-188 | V1.0 Done | Login on second device; verify data |
| FR-ST-006 | System MUST display Terms of Service via WebView | MUST | APP-191 | V1.0 Done | Tap ToS; verify content loads |
| FR-ST-007 | System MUST display Privacy Policy via WebView | MUST | APP-192 | V1.0 Done | Tap privacy; verify content loads |
| FR-ST-008 | System MUST display Cookie Policy | MUST | APP-193 | V1.0 Done | Tap cookie policy; verify display |
| FR-ST-009 | System MUST require agreement checkbox during registration | MUST | APP-194 | V1.0 Done | Register without check; verify blocked |
| FR-ST-010 | System MUST show agreement update notification when policy changes | MUST | APP-195 | V1.0 Done | Trigger update; verify popup |
| FR-ST-011 | System MUST support multi-language switching (zh/en/ja/ko/etc.) | MUST | APP-197 | V1.0 Done | Switch to Japanese; verify UI |
| FR-ST-012 | System MUST support region/timezone/date format selection | MUST | APP-198 | V1.0 Partial | **Gap: region format setting incomplete** |
| FR-ST-013 | System MUST display About page with version/team/licenses | MUST | APP-202 | V1.0 Done | Tap About; verify content |
| FR-ST-014 | System MUST check for app updates automatically | MUST | APP-203 | V1.0 Done | Force old version; verify update prompt |
| FR-ST-015 | System MUST support environment switching (Dev/QA/UAT/Prod) in debug mode | MUST | APP-204 | V1.0 Done | Toggle env; verify API endpoint changes |
| FR-ST-016 | System MUST provide auto-transcription toggle (on/off) | MUST | APP-205 | V1.2 Done | Toggle off; record; verify no auto-transcription |
| FR-ST-017 | System MUST allow upload network policy: WiFi-only or WiFi+Cellular | MUST | APP-206 | V1.0 Done | Set WiFi-only; disable WiFi; verify upload paused |
| FR-ST-018 | System SHOULD provide audio quality selector (High/Medium/Low) | SHOULD | APP-207 | **Not implemented** | UI missing |
| FR-ST-019 | System MUST provide developer debug mode toggle | MUST | APP-208 | V1.0 Done | 5-tap version; verify debug panel |
| FR-ST-020 | System MUST support crash log upload | MUST | APP-209 | V1.0 Done | Trigger crash; verify log uploaded |
| FR-ST-021 | System MUST request microphone permission with guidance | MUST | APP-210 | V1.0 Done | Deny permission; verify guidance shown |
| FR-ST-022 | System MUST request storage permission with guidance | MUST | APP-211 | V1.0 Done | Deny permission; verify guidance |
| FR-ST-023 | System MUST request location permission with guidance | MUST | APP-212 | V1.0 Done | Deny permission; verify guidance |
| FR-ST-024 | System MUST request push notification permission | MUST | APP-213 | V1.2 Done | FCM token registered on grant |
| FR-ST-025 | System MUST provide permission-denied guidance that jumps to OS settings | MUST | APP-214 | V1.0 Done | Deny perm; tap guide; verify OS settings opens |
| FR-ST-026 | System MUST provide FAQ / Help Center | MUST | APP-230 | V1.0 Done | Tap Help; verify FAQ content |
| FR-ST-027 | System MUST provide user feedback form | MUST | APP-231 | V1.0 Done | Submit feedback; verify sent |
| FR-ST-028 | System MUST display version changelog | MUST | APP-232 | V1.0 Done | Tap changelog; verify version history |
| FR-ST-029 | System SHOULD provide video tutorials | SHOULD | APP-233 | **Not implemented** | UED not started |

---

## 6. Key Dependencies

| Dependency | Used By | Purpose |
|-----------|---------|---------|
| `UserApi` | Profile, integrations, language | Account CRUD, integration endpoints, language endpoints |
| `UserService` | Account management | Token + user info singleton |
| `SummaryLanguageController` | Settings + Generate flows | Permanent singleton shared across pages |
| `IntegrationsController` | Integrations page | Lifecycle-aware OAuth state management |
| `VocabularyController` | Vocabulary page | Word CRUD with A-Z grouping |
| `NetworkConnectivityService` | Upload settings | Online/offline detection |
| `PushNotificationService` | Notification permission | FCM + SSE management |

---

## 7. Related Documents

- [Integrations](integrations.md) -- OAuth flow for Slack/Google/Notion
- [Summary Language](summary-language.md) -- AI summary language preference
- [Vocabulary](vocabulary.md) -- Custom words and industry termbases
- [System Context](../../system-context.md) -- APP boundary with BACKEND
- [Module 01: Auth](../01-auth.md) -- Registration and login
