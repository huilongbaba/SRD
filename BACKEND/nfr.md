# BACKEND -- Non-Functional Requirements

> Generated: 2026-04-02
> Source: YAML config files, code analysis

## 1. Performance

| Metric | Value | Source | Notes |
|--------|-------|--------|-------|
| HTTP request timeout (default) | 50,000 ms (50 s) | `audio-api: Timeout: 50000` | Go-zero server-level timeout |
| SSE streaming timeout | 0 (disabled) | `rest.WithTimeout(0)` in SSE routes | Prevents go-zero from closing long connections |
| gRPC client timeout (user-rpc) | 10,000 ms | `user-api: UserRpc.Timeout: 10000` | Per-RPC-client timeout |
| gRPC client timeout (mcp-rpc) | 5,000 ms | `audio-api: McpRpc.Timeout: 5000` | |
| Export task timeout | 300 s (5 min) | `worker: Export.Timeout: 300` | Per-task context timeout |
| Notification SSE heartbeat | 20 s | `notification: Notification.HeartbeatIntervalSec: 20` | Keep-alive interval |
| Notification write timeout | 10 s | `notification: Notification.WriteTimeoutSec: 10` | Per-SSE-write timeout |
| Notification connection TTL | 90 s | `notification: Notification.ConnTTLSeconds: 90` | Max connection lifetime |
| Notification channel buffer | 128 messages | `notification: Notification.ChannelBuffer: 128` | Per-connection buffer size |
| Dispatcher HTTP timeout | 3 s | `dispatcher: Dispatcher.HTTPTimeoutSec: 3` | For internal notification delivery |
| Dispatcher block read | 5 s | `dispatcher: Dispatcher.BlockSeconds: 5` | Redis XREADGROUP block time |
| Dispatcher idle reclaim | 60 s | `dispatcher: Dispatcher.ReclaimIdleSecond: 60` | Reclaim stuck pending messages |
| Export worker block read | 5 s | Redis XREADGROUP block time | From worker consumer loop |
| S3 presigned URL expiry | 15 min | `common/third_party/aws/` | For PUT upload URLs |

## 2. Reliability

| Metric | Value | Source |
|--------|-------|--------|
| Export dead-letter stream | `export_task_dead` | `worker: Export.DeadStream` |
| Export retry tracking | `export_task_retries` (Redis hash) | `worker: Export.RetryHashKey` |
| Notification max retry | 3 | `dispatcher: Dispatcher.MaxRetry: 3` |
| Notification dead-letter stream | `notification_push_dead` | `dispatcher: Dispatcher.DeadStreamName` |
| Consumer group (export) | `export_worker_group` | `worker: Export.ConsumerGroup` |
| Consumer group (notification) | `notification_push_group` | `dispatcher: Dispatcher.ConsumerGroup` |
| gRPC NonBlock mode | true | All RPC client configs | Non-blocking connection -- service starts even if RPC target unavailable |

## 3. Security

| Control | Implementation | Source |
|---------|---------------|--------|
| Transport encryption | AES-CBC on request/response bodies | `common/middleware/base_middleware.go` |
| AES key/IV | YAML config (per-environment) | Config files (not in repo) |
| JWT algorithm | HS256 | `common/jwtx/jwt.go` |
| JWT payload | userId only (minimal claims) | `UserClaims` struct |
| Session enforcement | Single-device-per-client-type | Redis key: `user:token:{clientType}:{userId}` |
| Token validation | Redis lookup on every authenticated request | `AuthMiddleware` |
| Response integrity | `X-Timestamp` + `X-Hash` headers | `BaseMiddleware` |
| Admin auth | Separate JWT + admin role check | `AdminMiddleware` + `AdminAuthMiddleware` |
| Internal API auth | API key header | `dispatcher: Dispatcher.InternalAPIKey` |
| CORS | Allowlist-based | `Cors.AllowOrigins: ["https://capsoul.space"]` |
| Share rate limiting | Redis-based rate limiter | Public share endpoint |

## 4. Scalability

| Aspect | Design | Notes |
|--------|--------|-------|
| API/RPC separation | Independent scaling of HTTP and gRPC tiers | go-zero microservice pattern |
| Service discovery | etcd-based | `Etcd.Hosts` in all configs |
| Export worker | Redis Streams consumer groups | Horizontal scaling via multiple worker instances |
| Notification dispatcher | Redis Streams consumer groups | Horizontal scaling via multiple dispatcher instances |
| Model caching | go-zero CachedConn | Auto-cache DB rows by primary key |
| ID generation | Snowflake | Distributed unique IDs without coordination |

## 5. Observability

| Signal | Implementation | Source |
|--------|---------------|--------|
| Tracing | OpenTelemetry trace propagation | `X-Trace-ID` header, `go.opentelemetry.io/otel/trace` |
| Logging | go-zero structured logging | `logx`, `logc` |
| Log rotation | File mode, 7-day retention, compressed | `worker: Log.Mode: file, KeepDays: 7, Compress: true` |
| Log level | info (production) | `worker: Log.Level: info` |
| Stat reporting | Disabled | `Log.Stat: false` across all services |
| gRPC metadata propagation | Client context headers forwarded | `BaseMiddleware` injects x-client-id, x-platform, x-user-id, etc. |
| Error tracking | Feishu (Lark) bot notifications | `common/third_party/feishu/` |

## 6. Maintainability

| Aspect | Design |
|--------|--------|
| Code generation | goctl generates handlers, types, models from `.api` and `.proto` definitions |
| Shared library | `common/` package with 15+ sub-packages (middleware, jwtx, cryptx, errorx, i18n, etc.) |
| i18n | `go-i18n/v2` with embedded locale files, error messages translated to user's language |
| Error codes | Unified `CodeError` with HTTP status code + i18n message key |
| Config management | YAML per environment, etcd for service discovery |
| Database migrations | Manual SQL scripts in `service/*/DB/` (no migration framework -- identified risk) |

## 7. Deployment

| Aspect | Value | Source |
|--------|-------|--------|
| Container runtime | Docker Compose | Per environment |
| Worker log path | `/app/logs/worker` | `worker: Log.Path` |
| Firebase credentials | JSON key file bundled | `dispatcher: Firebase.KeyPath` |
| Environment separation | Separate YAML configs + etcd clusters | Config per env |
