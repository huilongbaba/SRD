# Module 02 -- User Service

> Generated: 2026-04-02

## 1. Overview

### Objective
Manage the complete user lifecycle: registration, authentication, profile management, settings, email verification, calendar OAuth, and third-party integration authorization management.

### Scope
- User registration via email, Google Sign-In, Apple Sign-In
- Login with JWT issuance and Redis session management
- Profile CRUD (avatar, nickname, gender, birthday, timezone)
- User settings (auto-transcribe, calendar sync, marketing flags)
- Email verification code (via Mailgun)
- Password reset and change
- Supported languages listing
- Google Calendar OAuth flow (auth URL, callback, disconnect, settings)
- Third-party integrations management (Slack, Google, Notion -- list, add, delete)
- Account deletion

### Non-Scope
- AI processing, audio management, task management (other services)
- Admin user management (admin service)

## 2. Definitions

| Term | Definition |
|------|-----------|
| login_type | Authentication method: email, Google, Apple |
| client_type | Device category (e.g., iOS, Android) for single-device-per-type session enforcement |
| calendar_auth | OAuth authorization record for Google Calendar integration |
| integration_auth | OAuth authorization record for third-party services (Notion, Slack) |
| captcha | Email verification code with rate-limited sending |
| account_id | Human-readable unique account identifier |

## 3. System Boundary

| This service handles | Other services handle |
|--------------------|---------------------|
| User CRUD and auth | Audio recordings (audio service) |
| JWT issuance | Task management (task service) |
| Calendar OAuth flow | Calendar event sync (task service) |
| Integration auth management | Integration content send (audio service) |
| Email sending | AI processing (AI service) |

### gRPC Dependencies
- **user-api** calls: user-rpc, task-rpc, audio-rpc
- **user-rpc** is called by: user-api, audio-api, task-api

## 4. Scenarios

### SC-USER-01: Email Registration
1. Client sends `POST /api/v1/users/email-captcha` with recipient email and type=Register
2. Backend sends verification code via Mailgun, stores code in Redis with TTL
3. Client sends `POST /api/v1/users/register` with email, password, captcha, profile fields
4. Backend verifies captcha, creates user + user_auth + user_password records
5. Returns user profile with userId

### SC-USER-02: Login (Email)
1. Client sends `POST /api/v1/users/login` with login_type=email, email, password
2. user-rpc validates credentials against user_password table
3. user-rpc generates JWT (HS256) with userId and configurable expiry
4. JWT stored in Redis: `user:token:{clientType}:{userId}` (replaces any existing token)
5. Returns user profile + JWT token in response

### SC-USER-03: Google Calendar OAuth
1. Client sends `GET /api/v1/calendar/google/auth/url`
2. Backend generates Google OAuth URL with state=userId, redirect_uri=callback
3. User completes OAuth in browser
4. Google redirects to `GET /api/v1/calendar/google/auth/callback?code=XXX&state=userId`
5. Backend exchanges code for access_token + refresh_token
6. Stores in user_calendar_auth table
7. Returns success

### SC-USER-04: Third-Party Integration (Slack/Notion)
1. Client sends `POST /api/v1/users/integrations` to list current integrations
2. Client sends `POST /api/v1/users/integrations/add` with `{provider: "slack"}`
3. Backend generates OAuth URL, returns to client
4. User completes OAuth in browser
5. Callback saves tokens to integration_auth table
6. Client refreshes integration list on app resume

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-USER-01 | MUST | System MUST support user registration via email with captcha verification | Test: register with valid captcha succeeds; invalid captcha returns error |
| FR-USER-02 | MUST | System MUST support login via Google Sign-In (id_token verification) | Test: valid Google id_token creates/retrieves user |
| FR-USER-03 | MUST | System MUST support login via Apple Sign-In (auth_code verification) | Test: valid Apple auth code creates/retrieves user |
| FR-USER-04 | MUST | Login MUST issue JWT (HS256) and store it in Redis per user+clientType | Test: login returns JWT; Redis key set |
| FR-USER-05 | MUST | Login MUST invalidate previous sessions for the same user+clientType | Test: previous token no longer valid after new login |
| FR-USER-06 | MUST | Profile update MUST allow changing avatar, nickname, gender, birthday, timezone | Test: PATCH fields and verify persisted |
| FR-USER-07 | MUST | Password reset MUST require email captcha verification before allowing new password | Test: reset without valid captcha fails |
| FR-USER-08 | MUST | Account deletion MUST hard-delete user, user_auth, and user_password records | Test: deleted user cannot log in; records removed |
| FR-USER-09 | MUST | Email captcha MUST enforce rate limiting (configurable resend interval) | Test: rapid resend returns `next_send_in_seconds` |
| FR-USER-10 | MUST | Integration endpoints MUST use POST-based routes with JSON body `provider` field | Test: `POST /integrations/add {provider: "slack"}` succeeds |
| FR-USER-11 | MUST | Provider matching MUST use defined constants (GoogleProvider, ServerNameNotion, ServerNameSlack) | Code review: no magic strings |
| FR-USER-12 | SHOULD | Batch user info query SHOULD support up to 100 IDs per request | Test: 100 IDs returns all results |
| FR-USER-13 | MUST | Calendar OAuth callback MUST store access_token, refresh_token, and expiry | Test: verify fields stored after callback |
| FR-USER-14 | MUST | Calendar disconnect MUST revoke the authorization record | Test: disconnected auth no longer returned in list |
| FR-USER-15 | SHOULD | Supported language list SHOULD return recently used languages for the user | Test: after recording language usage, recent_items populated |

## 6. State Model

### User Status
```
0 (inactive) -> 1 (active) -> (deleted via hard-delete)
```

### Calendar Auth Status
```
0 (revoked) <-> 1 (active)
```

### Integration Auth Status
```
0 (revoked) <-> 1 (active)
```

## 7. Data Contract

### REST API Endpoints

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | `/api/v1/users/login` | No | `{login_type, email, password, google_id_token, apple_auth_code, timezone, nickname}` | `{user_id, avatar, nickname, status, gender, birthday, email, created_at, account_id}` + JWT |
| POST | `/api/v1/users/register` | No | `{email, password, captcha, nickname, gender, birthday, timezone}` | `{user_id, ...profile}` |
| POST | `/api/v1/users/reset-password` | No | `{email, password}` | `{user_id}` |
| POST | `/api/v1/users/email-captcha` | No | `{recipient_name, recipient_email, subject, body, type, reconfirm}` | `{next_send_in_seconds, last_sent_minutes_ago}` |
| POST | `/api/v1/users/supported-languages` | No | `{}` | `{items: [{code, app_name, native_name, notes}], recent_items: [...]}` |
| GET | `/api/v1/users/profile` | Yes | -- | `{id, avatar, nickname, status, gender, birthday, email, created_at, account_id}` |
| POST | `/api/v1/users/update-profile` | Yes | `{avatar, nickname, gender, birthday, timezone}` | `{}` |
| POST | `/api/v1/users/change-password` | Yes | `{old_password, new_password}` | `{}` |
| DELETE | `/api/v1/users/delete` | Yes | -- | `{success}` |
| POST | `/api/v1/users/logout` | Yes | -- | `{}` |
| POST | `/api/v1/users/base/info` | Yes | `{ids: [int64]}` | `{items: [{id, avatar, nickname}]}` |
| POST | `/api/v1/users/used-language` | Yes | `{code}` | `{}` |
| GET | `/api/v1/users/setting` | Yes | -- | `{auto_transcribe, calendar_sync, popup_marketing, email_marketing, push_marketing, ext}` |
| PATCH | `/api/v1/users/setting` | Yes | `{auto_transcribe?, calendar_sync?, ...}` | `{}` |
| POST | `/api/v1/users/integrations` | Yes | `{}` | `[{provider, account, expiresAt}]` |
| POST | `/api/v1/users/integrations/add` | Yes | `{provider}` | `{auth_url}` |
| POST | `/api/v1/users/integrations/delete` | Yes | `{provider}` | `{}` |

### Calendar Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/calendar/:provider/auth/url` | Yes | Get OAuth URL |
| GET | `/api/v1/calendar/:provider/auth/callback` | No | OAuth callback |
| GET | `/api/v1/calendar/:provider/auths` | Yes | List provider auths |
| DELETE | `/api/v1/calendar/:provider/auths/:id` | Yes | Disconnect auth |
| GET | `/api/v1/calendar/:provider/settings` | Yes | Get calendar settings |
| PATCH | `/api/v1/calendar/:provider/settings` | Yes | Update calendar settings |
| POST | `/api/v1/calendar/:provider/sync/:id` | Yes | Trigger sync |
| GET | `/api/v1/calendar/providers` | Yes | List providers |

### gRPC Service: `user` (26 RPCs)
Key RPCs: `UserLogin`, `UserRegister`, `UserById`, `UsersBaseInfo`, `UserUpdate`, `UserResetPassword`, `UserChangePassword`, `DeleteUser`, `GetClient`, `EmailCaptchaSend`, `EmailCaptchaCheck`, `UserSettingByUserId`, `UserSettingUpdate`, `CalendarAuthCallback`, `GetCalendarAuthById`, `DisconnectCalendarAuth`, `ListCalendarAuthsByProvider`, `ListCalendarAuthsByProviderPage`, `UpdateCalendarAuthLastSyncAt`, `UpdateCalendarAuthSyncToken`, `SupportedLanguageList`, `RecordLanguageUsage`

### Database Tables

| Table | Key Columns |
|-------|-------------|
| `user` | id, nickname, email, status, avatar, gender, birthday, timezone, account_id |
| `user_auth` | user_id, auth_type, auth_key |
| `user_password` | user_id, password_hash |
| `user_setting` | user_id, auto_transcribe, calendar_sync, marketing flags, ext |
| `user_calendar_auth` | user_id, provider, email, access_token, refresh_token, sync_token |
| `email_log` | recipient, subject, status |
| `client` | client_id |
| `supported_language` | code, app_name, native_name |

## 8. Error Handling

| Error | Code | Handling |
|-------|------|---------|
| Invalid credentials | 401 | Return i18n translated "invalid email or password" |
| Email already registered | 409 | Return "email already exists" |
| Invalid captcha | 400 | Return "invalid verification code" |
| Captcha rate limited | 429 | Return `next_send_in_seconds` |
| Google/Apple token invalid | 401 | Return "social login failed" |
| Calendar OAuth code exchange failure | 500 | Log error, return "authorization failed" |
| User not found | 404 | Return "user not found" |
| Old password incorrect | 400 | Return "old password incorrect" |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| JWT algorithm | HS256 | `common/jwtx/jwt.go` |
| JWT claims | userId only | Minimal payload |
| Session enforcement | Single-device-per-client-type | Redis key pattern |
| gRPC timeout (user-rpc from user-api) | 10,000 ms | `user-api: UserRpc.Timeout: 10000` |
| Email provider | Mailgun | `common/third_party/mailgun/` |
| Apple verification | Apple Sign-In REST API | `common/third_party/apple/` |
| Batch user info limit | 100 IDs | Caller-enforced constraint |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| Login events | Logged with userId, login_type, clientType |
| Auth failures | Logged with reason (invalid credentials, expired token, etc.) |
| Email sending | email_log table records all sent emails |
| Calendar OAuth | Logged with provider, userId, success/failure |
| Trace propagation | X-Trace-ID from middleware through all RPC calls |
