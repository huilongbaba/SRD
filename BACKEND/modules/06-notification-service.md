# Module 06 -- Notification Service

> Generated: 2026-04-02

## 1. Overview

### Objective
Deliver real-time push notifications to mobile clients via Server-Sent Events (SSE) long-connections and Firebase Cloud Messaging (FCM), using Redis Streams for reliable message dispatch.

### Scope
- **Notification API (port 6025)**: SSE long-connection for real-time push to connected clients
- **Notification Dispatcher (port 6015)**: Redis Stream consumer that routes notifications to SSE connections and FCM
- FCM device token registration
- Internal notification delivery endpoint (API-key authenticated)

### Non-Scope
- Notification content generation (AI service emits events)
- User authentication (handled by gateway middleware)
- Email notifications (user service via Mailgun)
- In-app notification center / read receipts (not implemented)

## 2. Definitions

| Term | Definition |
|------|-----------|
| SSE | Server-Sent Events -- HTTP long-connection for server-to-client push |
| FCM | Firebase Cloud Messaging -- push notification service for mobile devices |
| notification-api | HTTP service that maintains SSE connections to mobile clients |
| notification-dispatcher | Standalone service that consumes Redis Stream and routes to SSE/FCM |
| heartbeat | Periodic SSE comment sent to keep the connection alive |
| connection TTL | Maximum lifetime of an SSE connection before forced reconnection |
| channel buffer | Per-connection message buffer size |

## 3. System Boundary

| This module handles | Other modules handle |
|--------------------|---------------------|
| SSE connection management | Notification event emission (AI service via Redis) |
| FCM push delivery | FCM token management on device side (mobile app) |
| Redis Stream consumption | User auth (gateway middleware) |
| Message routing (user -> device) | Notification content/payload design (varies by source) |

### Architecture
```
AI Service -> Redis Stream (memoket:notifications)
                    |
                    v
          Notification Dispatcher
          (XREADGROUP consumer)
                    |
         +----------+-----------+
         |                      |
         v                      v
   SSE Delivery            FCM Push
   (HTTP POST to           (Firebase API)
    notification-api)
         |
         v
   SSE Connection
   (to mobile client)
```

## 4. Scenarios

### SC-NOTIF-01: SSE Connection Establishment
1. Client sends `GET /api/v1/notification/sse` with auth token
2. Server sets headers: Content-Type=text/event-stream, Cache-Control=no-cache, Connection=keep-alive
3. Server registers connection in Redis: `notification:user:{userId}:devices` and `notification:device:{deviceId}`
4. Server starts heartbeat loop (every 20s)
5. Connection stays open until TTL (90s) or client disconnect

### SC-NOTIF-02: Notification Dispatch (AI -> Client)
1. AI service writes event to Redis Stream: `XADD memoket:notifications {event, audio_id, tenant_id}`
2. Dispatcher consumes: `XREADGROUP notification_push_group` (block 5s)
3. Dispatcher resolves target user/device
4. Dispatcher sends to notification-api: `POST /internal/notifications/deliver` (API-key auth)
5. notification-api writes to user's SSE channel buffer
6. Client receives SSE event

### SC-NOTIF-03: FCM Push (Offline Users)
1. Dispatcher detects no active SSE connection for target user
2. Dispatcher sends FCM push via Firebase API
3. Mobile device receives push notification
4. App opens and establishes SSE connection for subsequent events

### SC-NOTIF-04: FCM Token Registration
1. Client sends `POST /api/v1/notification/dispatcher/token/register` with FCM token
2. Dispatcher stores token associated with user/device

### SC-NOTIF-05: Dead Letter Handling
1. Dispatcher fails to deliver notification (SSE and FCM both fail)
2. Retry count incremented (max 3)
3. After max retries, message moved to `notification_push_dead` stream

## 5. Functional Requirements

| ID | Priority | Requirement | Verification |
|----|----------|-------------|--------------|
| FR-NOTIF-01 | MUST | Notification API MUST maintain SSE long-connections with connected clients | Test: SSE connection stays open; heartbeat received |
| FR-NOTIF-02 | MUST | SSE heartbeat MUST be sent every 20 seconds | Test: heartbeat received within expected interval |
| FR-NOTIF-03 | MUST | SSE connection MUST be closed after TTL (90s), requiring client reconnection | Test: connection drops after 90s |
| FR-NOTIF-04 | MUST | SSE write operations MUST timeout after 10 seconds | Test: slow client does not block server |
| FR-NOTIF-05 | MUST | Each SSE connection MUST have a 128-message channel buffer | Test: buffer overflow handling |
| FR-NOTIF-06 | MUST | Dispatcher MUST consume from Redis Stream using consumer groups | Test: messages consumed and acknowledged |
| FR-NOTIF-07 | MUST | Dispatcher MUST attempt SSE delivery first, then FCM for offline users | Test: online user gets SSE; offline user gets FCM |
| FR-NOTIF-08 | MUST | Dispatcher MUST retry failed deliveries up to 3 times | Test: retry count incremented; max retry respected |
| FR-NOTIF-09 | MUST | Dispatcher MUST move failed messages to dead letter stream after max retries | Test: message appears in dead letter stream |
| FR-NOTIF-10 | MUST | Internal delivery endpoint MUST require API key authentication | Test: request without API key returns 401 |
| FR-NOTIF-11 | MUST | FCM token registration MUST associate token with user and device | Test: token stored correctly |
| FR-NOTIF-12 | MUST | Dispatcher MUST reclaim stuck pending messages after idle timeout (60s) | Test: orphaned messages reprocessed |
| FR-NOTIF-13 | SHOULD | Dispatcher SHOULD handle multiple simultaneous SSE connections per user | Test: notification delivered to all active connections |

## 6. State Model

### Notification Delivery State Machine
```
received (Redis Stream) --> dispatching --> delivered (SSE)
                                |          --> delivered (FCM)
                                |
                                +--> failed --> retry (1..3) --> dispatching
                                                    |
                                                    +--> dead letter
```

### SSE Connection Lifecycle
```
connecting --> connected (heartbeat loop active) --> TTL expired --> disconnected
                    |                                                     |
                    +--> client disconnect --------------------------------+
                    |
                    +--> write timeout --> connection closed
```

### Redis State (per connection)
```
notification:user:{userId}:devices -> Set of device connection IDs
notification:device:{deviceId} -> Connection metadata
```

## 7. Data Contract

### REST API Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/v1/notification/sse` | Yes (JWT) | SSE long-connection |
| POST | `/internal/notifications/deliver` | Internal (API key) | Deliver notification to connected client |
| POST | `/api/v1/notification/dispatcher/token/register` | Yes (JWT) | Register FCM device token |

### SSE Event Format
```
data: {"event": "ANALYSIS_COMPLETE", "audio_id": 12345, "status": "completed"}

: heartbeat
```

### Redis Streams
- **Input stream**: `memoket:notifications`
- **Consumer group**: `notification_push_group`
- **Dead letter**: `notification_push_dead`
- **Retry hash**: `notification_push_retries`

### Database Tables
None -- notification service is Redis-only (no persistent storage).

## 8. Error Handling

| Error | Handling |
|-------|---------|
| SSE write timeout (10s) | Close connection; client will reconnect |
| SSE channel buffer full (128) | Drop oldest message or close connection |
| FCM send failure | Retry up to 3 times; then dead letter |
| Redis Stream read error | Log error; retry on next iteration |
| Internal API key mismatch | Return 401 |
| Client disconnect mid-stream | Clean up Redis connection tracking |

## 9. NFR (Quantified)

| Metric | Value | Source |
|--------|-------|--------|
| SSE heartbeat interval | 20 s | `Notification.HeartbeatIntervalSec: 20` |
| SSE connection TTL | 90 s | `Notification.ConnTTLSeconds: 90` |
| SSE write timeout | 10 s | `Notification.WriteTimeoutSec: 10` |
| SSE channel buffer | 128 messages | `Notification.ChannelBuffer: 128` |
| Notification API port | 6025 | `notification: Port: 6025` |
| Dispatcher API port | 6015 | `dispatcher: Port: 6015` |
| Dispatcher HTTP timeout | 3 s | `Dispatcher.HTTPTimeoutSec: 3` |
| Dispatcher block read | 5 s | `Dispatcher.BlockSeconds: 5` |
| Dispatcher idle reclaim | 60 s | `Dispatcher.ReclaimIdleSecond: 60` |
| Dispatcher max retry | 3 | `Dispatcher.MaxRetry: 3` |
| Redis Stream (input) | `memoket:notifications` | `Dispatcher.StreamName` |
| Firebase key | JSON file path | `Firebase.KeyPath` |

## 10. Observability

| Signal | Implementation |
|--------|---------------|
| Connection tracking | Redis keys track active SSE connections per user/device |
| Delivery logging | Each notification delivery attempt logged with outcome |
| Dead letter monitoring | Dead letter stream entries for failed deliveries |
| Heartbeat health | Heartbeat loop errors logged |
| FCM response logging | Firebase API responses logged for push delivery |
| Connection lifecycle | Connect/disconnect events logged with userId, deviceId |
