# Non-Functional Requirements

> 所有指标从代码配置中提取，非估算值。每项均标注代码文件路径及行号，可直接溯源验证。

---

## 1. 性能 (Performance)

### 1.1 网络超时配置

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-P01 | HTTP 连接超时 (normal) | connect 超时 ≤ 30s | 30s | `network_timeout_config.dart:36` | 断网/弱网环境模拟，观察超时触发时间 |
| NFR-P02 | HTTP 发送超时 (normal) | send 超时 ≤ 60s | 60s | `network_timeout_config.dart:37` | 限速代理模拟慢上行，验证超时触发 |
| NFR-P03 | HTTP 接收超时 (normal) | receive 超时 ≤ 60s | 60s | `network_timeout_config.dart:38` | 限速代理模拟慢下行，验证超时触发 |
| NFR-P04 | 大文件上传超时 (generic) | send 超时 ≤ 15min | 15min | `network_timeout_config.dart:54` | 上传 >50MB 文件，模拟慢速网络验证 |
| NFR-P05 | 大文件上传超时 (iOS) | send 超时 ≤ 10min | 10min | `network_timeout_config.dart:109` | iOS 真机上传大文件，验证超时值 |
| NFR-P06 | 大文件上传超时 (Android) | send 超时 ≤ 20min | 20min | `network_timeout_config.dart:119` | Android 真机上传大文件，验证超时值 |
| NFR-P07 | 文件下载接收超时 | receive 超时 ≤ 10min | 10min | `network_timeout_config.dart:67` | 下载大文件，模拟慢速网络验证 |
| NFR-P08 | 日志上传超时 | connect: 60s, send: 2min, receive: 60s | 60s/120s/60s | `network_timeout_config.dart:76` | 上传大日志文件，模拟弱网验证各阶段超时 |
| NFR-P09 | 快速请求超时 | connect: 10s, send: 15s, receive: 15s | 10s/15s/15s | `network_timeout_config.dart:88` | 调用轻量级 API 端点，验证快速超时响应 |
| NFR-P10 | 动态超时上传速度估算 | 基于 50KB/s 估算，1.5x 缓冲，区间 [60s, 1800s] | 50KB/s, 1.5x, 60s-1800s | `network_timeout_config.dart:143` | 上传不同大小文件，验证动态超时计算值 |

### 1.2 SSE / 推送

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-P11 | SSE 断线重连延迟 | 断连后 ≤ 3s 发起重连 | 3s | `push_notification_service.dart:1467` | 断开服务端 SSE 连接，抓包验证重连间隔 |

### 1.3 预签名 URL 有效期

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-P12 | S3 上传预签名 URL 有效期 | URL 有效期 = 15min | 15min | `aws/client.go:114` | 生成 URL 后等待 15min+，验证 403 响应 |
| NFR-P13 | CloudFront 签名 URL 有效期 | URL 有效期 = 24h | 24h | `aws/client.go:156` | 生成 URL 后等待 24h+，验证过期拒绝 |

### 1.4 AI 处理超时

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-P14 | Insight 任务软超时 | soft_time_limit ≤ 300s | 300s | `askai_tasks.py:169` | 提交长文本 Insight 任务，监控 Celery 软超时 |
| NFR-P15 | 声纹处理超时常量 | VOICEPRINT_TIMEOUT_SEC = 300s | 300s | `voiceprint_tasks.py:32` | 提交长音频声纹任务，验证超时行为 |
| NFR-P16 | 声纹 Celery 任务超时 | soft_time_limit=840s, time_limit=900s | 840s/900s | `voiceprint_tasks.py:663` | 监控 Celery worker 日志，验证任务强制终止时间 |

### 1.5 Redis Stream 配置

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-P17 | Stream TTL | TTL = 600s (10min) | 600s | `insight_stream_buffer.py:24` | 写入 stream 后等待 10min+，验证自动过期 |
| NFR-P18 | Stream 最大事件数 | maxlen = 5000 events | 5000 | `insight_stream_buffer.py:25` | 连续写入 >5000 事件，验证旧事件裁剪 |
| NFR-P19 | Stream XREAD 阻塞时间 | block = 2000ms | 2000ms | `insight_stream_buffer.py:26` | 监控 XREAD 调用，验证阻塞等待时长 |

### 1.6 Celery Worker 配置

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-P20 | Worker 池类型 | pool = threads | threads | `.env.prod:141` | 检查 Celery worker 启动日志，确认池类型 |
| NFR-P21 | Worker 并发数 | concurrency = 16 (prod), 4 (docker default) | 16/4 | `.env.prod:143` | `celery inspect active` 确认并发槽位数 |
| NFR-P22 | 预取乘数 | prefetch_multiplier = 1 | 1 | `.env.prod:144` | `celery inspect conf` 验证配置值 |
| NFR-P23 | 任务确认策略 | acks_late = true | true | `.env.prod:146` | 终止 worker 进程，验证任务重新入队 |
| NFR-P24 | Worker 丢失拒绝策略 | reject_on_worker_lost = true | true | `.env.prod:145` | 模拟 worker 崩溃，验证任务被拒绝并重新分发 |

### 1.7 数据库连接池

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-P25 | 连接池超时 | pool_timeout ≤ 30s | 30s | `.env.prod:47` | 耗尽连接池后发起新请求，验证等待超时 |
| NFR-P26 | 连接池大小 | pool_size = 10 | 10 | `.env.prod:45` | `SHOW PROCESSLIST` 验证稳态连接数 |
| NFR-P27 | 连接池溢出上限 | max_overflow = 20 | 20 | `.env.prod:46` | 并发压测，验证最大连接数 = 10 + 20 = 30 |

---

## 2. 可用性 (Usability)

### 2.1 蓝牙 (BLE) 体验

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-U01 | BLE 扫描超时 | 扫描超时 ≤ 30s | 30s | `device_bluetooth_manager.dart:231` | 开启扫描但无设备，验证 30s 后扫描停止 |
| NFR-U02 | BLE 自动重连策略 | 蓝牙适配器恢复 ON 时自动按序重连所有挂起设备 | 事件驱动，无上限重试 | `device_bluetooth_manager.dart:1105` | 关闭/重开蓝牙，验证自动重连所有挂起设备 |
| NFR-U03 | BLE 重连设备间隔 | 相邻设备重连间隔 = 500ms | 500ms | `device_bluetooth_manager.dart:1124` | 多设备挂起时重开蓝牙，抓包验证间隔时序 |

### 2.2 离线缓存

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-U04 | 本地存储引擎 | Hive 加密存储 + 内存 LRU 缓存 | Hive (hive_flutter ^1.1.0), in-memory Map cache, AES-256-CBC | `storage_service.dart:81` | 离线读写数据，验证 Hive 文件加密及内存缓存命中 |

### 2.3 上传失败自动重试

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-U05 | 冷启动自动重传 | APP 启动时自动重试所有 audioUploadError 状态项（本地文件仍存在） | 启动时全量重试 | `audio_file_controller_upload.dart:127` | 制造上传失败后杀进程重启，验证自动重新上传 |
| NFR-U06 | 网络恢复自动重传 | 网络连接恢复时自动重试所有失败上传 | 网络恢复即触发 | `network_connectivity_service.dart:79` | 断网后恢复，验证所有失败上传自动重试 |

### 2.4 网络请求重试策略

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-U07 | 默认重试策略 | maxRetries=3, delay=1000ms, factor=2.0 (指数退避); 可重试状态码: 408/429/500/502/503/504 | 3次/1s/2.0x; [408,429,500,502,503,504] | `retry_interceptor.dart:27` | 模拟 502 响应，抓包验证重试次数及间隔 (1s, 2s, 4s) |
| NFR-U08 | 激进重试策略 | maxRetries=5, delay=500ms, factor=1.5 | 5次/500ms/1.5x | `retry_interceptor.dart:46` | 模拟持续失败，验证重试 5 次，间隔 500ms→750ms→1125ms→1687ms→2531ms |

---

## 3. 安全性 (Security)

### 3.1 传输加密

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-S01 | 网络传输加密算法 | AES-256-CBC | AES-256-CBC (encrypt ^5.0.3) | `http_service.dart:958` | 抓包验证请求/响应 body 为密文，解密验证 |
| NFR-S02 | 加密密钥规格 | 32 bytes AES-256 key + 16 bytes IV (CBC) | 32B key + 16B IV | `encryption_key_provider.dart:6` | 断点检查密钥长度，验证 CBC 模式初始向量 |
| NFR-S03 | 加密完整性校验 | SHA-256 hash of (encryptedData + timestamp) | SHA-256 | `http_service.dart:872` | 篡改密文后发送，验证服务端拒绝 hash 不匹配请求 |

### 3.2 JWT 认证

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-S04 | 后端 JWT 算法 | HS256 (HMAC-SHA256) | HS256 | `jwtx/jwt.go:20` | 解析 JWT header 验证 alg 字段 |
| NFR-S05 | 后端 JWT 有效期 | access token 有效期 = 30 天 | 2592000s (30d) | `common.yaml:239` | 签发 token 后等待 30d+，验证过期拒绝 |
| NFR-S06 | AI 服务 access token 有效期 | 有效期 = 30min | 30min | `security.py:18`, `.env.prod:30` | 签发 token 后等待 30min+，验证 401 响应 |
| NFR-S07 | AI 服务 refresh token 有效期 | 有效期 = 30 天 | 30d | `security.py:47`, `.env.prod:31` | 签发 refresh token 后等待 30d+，验证过期 |
| NFR-S08 | AI 服务 JWT 算法 | HS256 | HS256 | `.env.prod:29` | 解析 JWT header 验证 alg 字段 |
| NFR-S09 | MCP OAuth token 有效期 | 有效期 = 1h | 3600s (1h) | `server_oauth.py:510` | 签发 OAuth token 后等待 1h+，验证失效 |

### 3.3 本地数据加密

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-S10 | 后端 AES 密钥长度 | AES-128 (16 bytes) | 16B | `common.yaml:242` | 检查配置中 Aes.Key 长度 = 16 字符 |
| NFR-S11 | APP 本地存储加密 | AES-256-CBC, 密钥由 KeyManagementService 管理 | AES-256-CBC, Hive 加密存储 | `key_management_service.dart:414` | 读取 Hive 文件原始字节，验证为 AES 密文 |
| NFR-S12 | SQLite 数据库加密 | SQLCipher 加密 | SQLCipher (libsqlcipher.so) | `MANIFEST.MF:24` | 尝试用普通 SQLite 打开数据库文件，验证失败 |

---

## 4. 兼容性 (Compatibility)

### 4.1 BLE 协议

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-C01 | BLE MTU 大小 | MTU = 511 bytes (514 - 3 overhead) | 511B | `ota_commands.dart:173` | 抓取 BLE 数据包，验证 MTU 协商值 |

### 4.2 音频格式支持

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-C02 | 支持的音频格式 | 支持 6 种格式: mp3, m4a, aac, wav, ogg, opus | [mp3, m4a, aac, wav, ogg, opus] | `file_type_utils.dart:18` | 逐格式导入音频文件，验证播放及转写正常 |
| NFR-C03 | 默认录音格式 | 录音输出 = m4a (AAC-LC) | m4a (AAC-LC) | `recording_service.dart:200` | 录制音频，检查输出文件格式及编码器 |
| NFR-C04 | 音频转码编解码器 | 转码输出 = AAC (.m4a 容器) | AAC → .m4a | `audio_transcode_service.dart:47` | 导入 wav/opus 文件，验证转码后为 m4a/AAC |
| NFR-C05 | BLE 设备音频格式 | BLE 设备输出 = Opus (raw → Ogg Opus .opus) | Opus → Ogg Opus | `audio_data_converter.dart:8` | 连接 BLE 设备录音，验证输出 .opus 文件可播放 |

### 4.3 平台最低版本

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-C06 | iOS 最低部署版本 (Xcode) | IPHONEOS_DEPLOYMENT_TARGET ≥ 13.0 | 13.0 | `project.pbxproj:705` | Xcode Build Settings 检查 |
| NFR-C07 | iOS Pod 最低版本覆盖 | Podfile post_install 覆盖各 Pod 最低版本 = 14.0 | 14.0 | `Podfile:57` | `pod install` 后检查生成的 xcconfig 文件 |
| NFR-C08 | Android 最低 SDK 版本 | minSdkVersion = flutter.minSdkVersion (typically 21) | 21 (Android 5.0) | `build.gradle.kts:95` | `./gradlew :app:dependencies` 检查 minSdk |
| NFR-C09 | Flutter SDK 版本约束 | SDK ≥ 3.8.0, < 4.0.0 | >=3.8.0 <4.0.0 | `pubspec.yaml:7` | `flutter --version` 确认在约束范围内 |

### 4.4 第三方依赖

| ID | 指标 | 要求 | 实测值 | 代码来源 | 验证方法 |
|----|------|------|--------|----------|----------|
| NFR-C10 | FFmpeg Kit 依赖版本 | ffmpeg_kit_flutter_new_audio ^2.0.0 | ^2.0.0 | `pubspec.yaml:75` | `flutter pub deps` 验证解析版本 |

---

## 附录: 指标汇总统计

| 类别 | 数量 | ID 范围 |
|------|------|---------|
| 性能 (Performance) | 27 | NFR-P01 ~ NFR-P27 |
| 可用性 (Usability) | 8 | NFR-U01 ~ NFR-U08 |
| 安全性 (Security) | 12 | NFR-S01 ~ NFR-S12 |
| 兼容性 (Compatibility) | 10 | NFR-C01 ~ NFR-C10 |
| **合计** | **57** | — |

> 文档生成日期: 2026-04-02
> 数据来源: `SRD/nfr_config_values.json` (从代码库自动提取的 55 项原始配置值，部分含子项展开为 57 行)
