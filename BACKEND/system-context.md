# BACKEND -- System Context

> Generated: 2026-04-02

## 1. System Boundary

```
Mobile App (iOS/Android)
    |
    | HTTPS (AES-encrypted body) + SSE
    v
+--------------------------------------------------+
|              Memoket Backend                       |
|                                                    |
|  API Layer (HTTP)          RPC Layer (gRPC)        |
|  +--------------+          +--------------+        |
|  | user-api     |---gRPC-->| user-rpc     |        |
|  | :7001        |          | :6001        |        |
|  +--------------+          +--------------+        |
|  | audio-api    |---gRPC-->| audio-rpc    |        |
|  | :7005        |          |              |        |
|  +--------------+          +--------------+        |
|  | task-api     |---gRPC-->| task-rpc     |        |
|  +--------------+          +--------------+        |
|  | device-api   |---gRPC-->| device-rpc   |        |
|  +--------------+          +--------------+        |
|  | admin-api    |---gRPC-->| admin-rpc    |        |
|  +--------------+          +--------------+        |
|  | mcp-api      |---gRPC-->| mcp-rpc      |        |
|  +--------------+          +--------------+        |
|  | tc-api       |---gRPC-->| tc-rpc       |        |
|  +--------------+          +--------------+        |
|  | notif-api    |  (Redis only -- no RPC)          |
|  | :6025        |                                  |
|  +--------------+                                  |
|  | notif-disp   |                                  |
|  | :6015        |                                  |
|  +--------------+                                  |
|  | common-api   |  (static pages)                  |
|  +--------------+                                  |
|                                                    |
|  Worker Layer                                      |
|  +--------------+                                  |
|  | export-worker|  (Redis Stream consumer)         |
|  +--------------+                                  |
+--------------------------------------------------+
    |           |           |            |
    v           v           v            v
  AI Service  AWS S3    PostgreSQL    Redis
  (external)  CloudFront             (cache/stream)
    |
    v
  Firebase / Google APIs / Notion / Slack / Mailgun / Apple
```

## 2. Service Decomposition

### 2.1 Core Domain Services

| Service | Type | Port | Responsibility |
|---------|------|------|---------------|
| **user** | API + RPC | 7001 / 6001 | User auth, profile, settings, calendar OAuth, integrations management |
| **audio** | API + RPC | 7005 / -- | Recording CRUD, S3 presign, AI proxy (transcription, summary, agents, chat, insight), folders, sharing, export creation, notes |
| **task** | API + RPC | -- / -- | Task CRUD, Google Calendar two-way sync, webhook handling |

### 2.2 Supporting Domain Services

| Service | Type | Port | Responsibility |
|---------|------|------|---------------|
| **device** | API + RPC | -- / -- | IoT device binding/unbinding, firmware check, status reporting |
| **notification** | API + Dispatcher | 6025 / 6015 | SSE push to clients (API); FCM push + Redis Stream consumer (Dispatcher) |
| **template_community** | API + RPC | -- / -- | Template browsing, favorites, usage tracking |
| **mcp** | API + RPC + Server | -- / -- | MCP app listing, OAuth flow, MCP SSE server endpoint |

### 2.3 Infrastructure Services

| Service | Type | Port | Responsibility |
|---------|------|------|---------------|
| **admin** | API + RPC | -- / -- | Admin auth, system users/menus/dictionaries, firmware OTA, operation logs |
| **worker** | Standalone | -- | Export document generation: Redis Stream consumer, PDF/Word conversion, S3 upload |
| **common** | API only | -- | Static pages (privacy policy) |

## 3. Inter-Service gRPC Dependencies

| API Service | Depends On (gRPC) |
|------------|-------------------|
| **user-api** | user-rpc, task-rpc, audio-rpc |
| **audio-api** | audio-rpc (audio + folder + share), task-rpc, user-rpc, template_community-rpc |
| **task-api** | task-rpc, audio-rpc (audio + folder), user-rpc |
| **device-api** | device-rpc |
| **admin-api** | admin-rpc, device-rpc |
| **mcp-api** | mcp-rpc |
| **template_community-api** | template_community-rpc |
| **notification-api** | (none -- Redis only) |

## 4. External Dependencies

| Dependency | Protocol | Used By | Purpose |
|-----------|----------|---------|---------|
| AI Service | HTTP REST + SSE | audio-api, task-api, tc-api, worker | Transcription, summary, chat, agents, insight, templates, search |
| AWS S3 | HTTPS (presigned) | audio-api, device-api, admin-api, worker | Audio file storage, export document storage |
| CloudFront CDN | HTTPS (signed) | audio-api | CDN read access to S3 objects |
| PostgreSQL | TCP | All RPC services | Primary data store |
| Redis | TCP | All services | Session tokens, model cache, streams, SSE routing |
| etcd | TCP | All services | gRPC service discovery |
| Firebase (FCM) | HTTPS | notification-dispatcher | Push notifications to mobile devices |
| Google APIs | HTTPS (OAuth) | user-api, task-api, mcp-api | Calendar sync, OAuth |
| Notion API | HTTPS (OAuth) | audio-api, user-api | Note integration |
| Slack API | HTTPS (OAuth) | audio-api, user-api | Channel integration |
| Mailgun | HTTPS | user-rpc | Email verification codes |
| Apple Sign-In | HTTPS | user-rpc | Apple social login verification |

## 5. Data Flow Layers

Every request through the backend follows the go-zero layered architecture:

```
Client -> TraceHeader MW -> BaseMiddleware (AES decrypt, JWT parse) -> AuthMiddleware (Redis token check) -> Handler (generated) -> Logic (hand-written) -> [gRPC to RPC | External HTTP] -> Model (DB/Cache)
```

- **Middleware**: AES decrypt request body, parse JWT, validate token in Redis, inject context
- **Handler**: Parse HTTP request into typed struct, call Logic, write HTTP response
- **Logic**: Business orchestration -- RPC calls, external service calls, data transformation
- **RPC Server**: Database operations wrapped in business rules
- **Model**: SQL CRUD with go-zero CachedConn cache invalidation
