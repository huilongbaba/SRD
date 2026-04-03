# Module 01 -- API Gateway & Middleware

> Generated: 2026-04-02

## 1. Overview

### Objective
Provide a unified HTTP entry layer for all backend services, handling transport encryption, authentication, request routing, CORS, and trace propagation.

### Scope
- AES-CBC request/response body encryption and decryption
- JWT parsing and Redis-backed session validation
- gRPC metadata propagation to downstream RPC services
- CORS policy enforcement
- Trace ID injection and propagation
- Unified error response formatting with i18n translation

### Non-Scope
- Business logic (delegated to Logic layer and RPC services)
- Database access (handled by RPC services)
- AI processing (external service)

## 2. Definitions

| Term | Definition |
|------|-----------|
| BaseMiddleware | Middleware that decrypts AES-CBC request body, parses JWT, injects RequestHeader into context, encrypts response body |
| AuthMiddleware | Middleware that validates the JWT token against Redis session store |
| AdminMiddleware | Variant of BaseMiddleware for admin routes with admin role check |
| AdminAuthMiddleware | Admin session validation middleware |
| RequestHeader | Struct extracted from HTTP headers containing userId, language, token, clientType, platform, etc. |
| CorsMiddleware | CORS policy middleware with configurable allowed origins |
| TraceHeaderMiddleware | Injects/propagates X-Trace-ID header |

## 3. System Boundary

| This module handles | Other modules handle |
|--------------------|---------------------|
| HTTP request/response lifecycle | Business logic (Logic layer) |
| AES encryption/decryption | Database access (RPC/Model) |
| JWT parsing and session validation | External API calls |
| Error response formatting | AI service proxy |
| gRPC metadata injection | SSE streaming content |

## 4. Scenarios

### SC-GW-01: Authenticated Request Flow
1. Client sends HTTPS request with AES-encrypted body and `Authorization: Bearer <JWT>` header
2. TraceHeaderMiddleware injects X-Trace-ID
3. BaseMiddleware decrypts AES-CBC body using configured key/IV
4. BaseMiddleware parses JWT from Authorization header, extracts userId
5. BaseMiddleware injects RequestHeader into request context
6. BaseMiddleware injects gRPC metadata (x-client-id, x-platform, x-user-id, etc.)
7. AuthMiddleware reads userId from context
8. AuthMiddleware checks Redis: `GET user:token:{clientType}:{userId}`
9. If token matches: forward to Handler
10. Handler returns response JSON
11. BaseMiddleware AES-CBC encrypts response body
12. BaseMiddleware sets `X-Timestamp` and `X-Hash` response headers

### SC-GW-02: SSE Bypass
1. SSE route configured with `rest.WithTimeout(0)`
2. BaseMiddleware detects `text/event-stream` Content-Type on response
3. Response body written raw (no AES encryption)
4. Connection kept alive until stream ends or client disconnects

### SC-GW-03: Unauthenticated Endpoint
1. Request hits route without AuthMiddleware (e.g., `/api/v1/users/login`)
2. BaseMiddleware still decrypts body and parses JWT if present
3. No Redis token validation
4. Handler processes request directly

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-GW-01 | MUST | BaseMiddleware MUST decrypt AES-CBC encrypted request bodies using the configured key and IV | Unit test: send encrypted body, verify decrypted output |
| FR-GW-02 | MUST | BaseMiddleware MUST encrypt non-SSE response bodies with AES-CBC | Unit test: verify response body is encrypted |
| FR-GW-03 | MUST | BaseMiddleware MUST bypass AES encryption for `text/event-stream` responses | Integration test: SSE endpoint returns raw data |
| FR-GW-04 | MUST | AuthMiddleware MUST validate JWT token against Redis session store on every authenticated request | Test: expired/mismatched token returns 401 |
| FR-GW-05 | MUST | AuthMiddleware MUST return i18n-translated 401 error on token mismatch or absence | Test: verify error message in requested language |
| FR-GW-06 | MUST | BaseMiddleware MUST inject RequestHeader (userId, language, clientType, platform, etc.) into request context | Test: downstream handler can read userId from context |
| FR-GW-07 | MUST | BaseMiddleware MUST inject gRPC metadata (x-client-id, x-platform, x-user-id, etc.) into context for downstream RPC calls | Test: RPC server receives metadata |
| FR-GW-08 | MUST | Every response MUST include `X-Timestamp` (Unix ms) and `X-Hash` (hash of encrypted body + timestamp) headers | Test: verify headers present on every response |
| FR-GW-09 | SHOULD | Request `X-Timestamp` and `X-Hash` SHOULD be verified when provided (logged but not blocking) | Log check: verification logged |
| FR-GW-10 | MUST | CorsMiddleware MUST enforce the configured AllowOrigins whitelist | Test: cross-origin request from disallowed origin is rejected |
| FR-GW-11 | MUST | TraceHeaderMiddleware MUST propagate X-Trace-ID across the request lifecycle | Test: trace ID appears in response headers |

## 6. State Model

Stateless. Each request is independently processed through the middleware chain. No per-request state is persisted (session state is in Redis).

## 7. Data Contract

### Middleware Chain Order (Client Routes)
```
Request -> TraceHeader -> BaseMiddleware (AES decrypt, JWT parse, logging) -> AuthMiddleware (Redis token check) -> Handler
```

### Middleware Chain Order (Admin Routes)
```
Request -> AdminMiddleware (AES decrypt, JWT parse, admin role check) -> AdminAuthMiddleware (Redis session) -> Handler
```

### RequestHeader (injected into context)
```go
type RequestHeader struct {
    UserId      string
    Language    string
    Token       string
    ClientID    string
    ClientType  string
    PhoneModel  string
    OS          string
    OsVersion   string
    VersionCode string
    Region      string
}
```

### gRPC Metadata Propagation
```
x-client-id, x-device-model, x-platform, x-os, x-os-version, x-version, x-region, x-user-id
```

### Redis Session Key
```
user:token:{clientType}:{userId} -> JWT string
```

## 8. Error Handling

| Error | HTTP Code | Handling |
|-------|-----------|---------|
| AES decryption failure | 400 | Log error, return bad request |
| JWT parse failure | 401 | Return unauthorized with i18n message |
| JWT expired | 401 | Return unauthorized with i18n message |
| Token mismatch (Redis) | 401 | Return unauthorized (`UnauthorizedError`) |
| CORS violation | 403 | Request blocked by CorsMiddleware |

### Error Response Format
```json
{
  "code": 401,
  "msg": "Unauthorized (translated)"
}
```
All errors flow through `common/respx/` for consistent JSON structure, with `common/i18n/` translating error keys to the user's language.

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| AES algorithm | AES-CBC | `common/cryptx/` |
| JWT algorithm | HS256 | `common/jwtx/jwt.go` |
| Session model | Single-device-per-client-type | Redis key pattern |
| CORS allowed origins | `https://capsoul.space` | YAML config |
| Default HTTP timeout | 50 s | `Timeout: 50000` |
| SSE timeout | 0 (disabled) | `rest.WithTimeout(0)` |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| Trace ID | `X-Trace-ID` header injected by TraceHeaderMiddleware, propagated to all downstream services |
| Structured logging | `logx`/`logc` with trace ID context |
| Request/response logging | BaseMiddleware logs decrypted request body (debug level) |
| Auth failure logging | AuthMiddleware logs token mismatch events |
| Error translation | Error codes mapped to i18n keys for user-facing messages |
