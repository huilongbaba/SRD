# Module 07 -- Integration Service

> Generated: 2026-04-02

## 1. Overview

### Objective
Manage OAuth authorization flows and content delivery for third-party integrations (Notion, Slack), enabling users to send audio notes and transcriptions to external platforms.

### Scope
- OAuth flow for Notion and Slack (auth URL generation, callback handling, token storage)
- Integration auth status checking
- Resource listing from integrated platforms (Notion pages, Slack channels)
- Content send to integrated platforms
- Integration authorization management via user service (list, add, delete)

### Non-Scope
- Google Calendar integration (handled by user service calendar endpoints + task service sync)
- AI processing or content generation
- MCP tool integrations (MCP service)

## 2. Definitions

| Term | Definition |
|------|-----------|
| integration_auth | OAuth token record for a third-party service (Notion/Slack) |
| provider | Service identifier: "notion", "slack" (using constx constants: ServerNameNotion, ServerNameSlack) |
| provider_id | Numeric identifier for the integration provider |
| send-to-integration | Action of pushing content (notes, transcription) to an integrated platform |

## 3. System Boundary

| This module handles | Other modules handle |
|--------------------|---------------------|
| OAuth flow for Notion/Slack | User-level integration management (user service: add/delete/list) |
| Token storage in integration_auth | Audio content generation (AI service) |
| Resource listing (pages/channels) | Google Calendar OAuth (user service calendar endpoints) |
| Content delivery to platforms | |

### Split Across Services
- **User service** (`/api/v1/users/integrations/*`): Manages integration authorization records (list, add with OAuth URL, delete). Uses POST-based routes with `{provider}` in JSON body.
- **Audio service** (`/api/v1/integrations/:provider/*`): Handles content-facing operations (auth status, auth URL, resource list, send, OAuth callback).

## 4. Scenarios

### SC-INT-01: Slack OAuth Authorization
1. Client sends `POST /api/v1/users/integrations/add` with `{provider: "slack"}`
2. user-api -> user-rpc: Generate OAuth URL with state=userId, redirect_uri=callback
3. Returns `{auth_url: "https://slack.com/oauth/v2/authorize?..."}`
4. User opens URL in system browser, grants permission
5. Slack redirects to callback: `GET /api/v1/integrations/slack/callback?code=XXX&state=userId`
6. Backend exchanges code for access_token + refresh_token
7. Backend stores in integration_auth table via audio-rpc: `UpsertIntegrationAuth`
8. Browser shows success page
9. App resumes, refreshes integration list: `POST /api/v1/users/integrations`
10. Slack now shows as authorized

### SC-INT-02: Send to Notion
1. Client checks auth: `GET /api/v1/integrations/notion/auth-status`
2. Client lists pages: `GET /api/v1/integrations/notion/list`
3. Client selects page and sends content: `POST /api/v1/integrations/notion/send` with `{page_id, content}`
4. Backend uses stored OAuth token to call Notion API
5. Returns success/failure

### SC-INT-03: Integration Revocation
1. Client sends `POST /api/v1/users/integrations/delete` with `{provider: "slack"}`
2. user-api revokes integration auth record
3. Integration no longer shows in list

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-INT-01 | MUST | OAuth URL generation MUST keep client_secret server-side (never exposed to client) | Code review: no secrets in response |
| FR-INT-02 | MUST | OAuth callback MUST exchange authorization code for tokens and store in integration_auth | Test: tokens stored after callback |
| FR-INT-03 | MUST | Provider identification MUST use defined constants (ServerNameNotion, ServerNameSlack, GoogleProvider) | Code review: no magic strings |
| FR-INT-04 | MUST | Integration list endpoint MUST use POST with JSON body `{provider}` | Test: POST request with provider body succeeds |
| FR-INT-05 | MUST | Auth status endpoint MUST return current authorization state for the specified provider | Test: authorized provider returns status |
| FR-INT-06 | MUST | Resource listing MUST return platform-specific resources (Notion pages, Slack channels) | Test: correct resources returned for each provider |
| FR-INT-07 | MUST | Send-to-integration MUST deliver content using stored OAuth tokens | Test: content appears in target platform |
| FR-INT-08 | MUST | Integration deletion MUST revoke the authorization record | Test: deleted integration no longer returned in list |
| FR-INT-09 | SHOULD | Token refresh SHOULD be attempted before token expiry | Test: expired tokens trigger refresh flow |
| FR-INT-10 | SHOULD | Failed OAuth callback SHOULD log error and return appropriate error page | Test: error logged; browser shows error |

## 6. State Model

### Integration Auth Status
```
0 (not authorized) -> 1 (authorized) -> 0 (revoked/expired)
```

### OAuth Flow States
```
initiated (URL generated) -> user_consenting (in browser) -> callback_received -> token_stored (authorized)
                                    |
                                    +--> user_cancelled (no callback fires)
```

## 7. Data Contract

### REST API Endpoints (User Service -- Integration Management)

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| POST | `/api/v1/users/integrations` | Yes | `{}` | `[{provider, account, expiresAt}]` |
| POST | `/api/v1/users/integrations/add` | Yes | `{provider: "slack"\|"notion"\|"google"}` | `{auth_url}` |
| POST | `/api/v1/users/integrations/delete` | Yes | `{provider: "slack"\|"notion"\|"google"}` | `{}` |

### REST API Endpoints (Audio Service -- Integration Operations)

| Method | Path | Auth | Request | Response |
|--------|------|------|---------|----------|
| GET | `/api/v1/integrations/:provider/auth-status` | Yes | -- | `{authorized, account?}` |
| GET | `/api/v1/integrations/:provider/auth-url` | Yes | -- | `{auth_url}` |
| GET | `/api/v1/integrations/:provider/list` | Yes | -- | `[{id, name, ...}]` (platform resources) |
| POST | `/api/v1/integrations/:provider/send` | Yes | `{resource_id, content}` | `{success}` |
| GET | `/api/v1/integrations/:provider/callback` | No | `?code=XXX&state=userId` | Redirect/success page |

### gRPC Messages (audio.proto -- integration subset)

```protobuf
message UpsertIntegrationAuthReq {
  int64 user_id = 1;
  int64 provider_id = 2;
  string access_token = 3;
  string refresh_token = 4;
  int64 expires_at = 5;
  int64 status = 6;
}

message GetIntegrationAuthByProviderReq {
  int64 user_id = 1;
  int64 provider_id = 2;
}

message GetIntegrationAuthByProviderResp {
  bool found = 1;
  int64 provider_id = 2;
  string access_token = 3;
  string refresh_token = 4;
  int64 expires_at = 5;
  int64 status = 6;
}
```

### Database Tables

| Table | Key Columns |
|-------|-------------|
| `integration_auth` | user_id, provider_id, access_token, refresh_token, expires_at, status |

### External Service Clients

| Client | Package | Purpose |
|--------|---------|---------|
| Notion | `common/third_party/notion/` | OAuth & page API |
| Slack | `common/third_party/slack/` | OAuth & channel API |

## 8. Error Handling

| Error | Code | Handling |
|-------|------|---------|
| OAuth code exchange failure | 500 | Log error; browser shows error page |
| Integration not authorized | 401 | Return "not authorized for this integration" |
| Token expired | 401 | Attempt refresh; if refresh fails, return "re-authorization required" |
| Notion API error | 502 | Log error; return "Notion service error" |
| Slack API error | 502 | Log error; return "Slack service error" |
| Unknown provider | 400 | Return "unsupported provider" |
| Send content failure | 500 | Log error; return "failed to send content" |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| Supported providers | Notion, Slack (+ Google for calendar only) | `constx.ServerNameNotion`, `constx.ServerNameSlack` |
| OAuth secrets | Server-side only | Config files |
| Token storage | Encrypted in DB | integration_auth table |
| Concurrent OAuth guard | `_providerBusyMap` on client side | Prevents duplicate OAuth flows |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| OAuth flow logging | Provider, userId, success/failure logged |
| Token refresh logging | Refresh attempts and outcomes logged |
| Send operation logging | Provider, resource_id, success/failure logged |
| Error logging | Full context for failed API calls to external platforms |
