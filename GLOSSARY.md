# Memoket 统一术语表

> SRD V1.2 | 按类别分组

## 文档与流程术语

| 术语 | 定义 | 使用场景 |
|------|------|----------|
| SRD | Software Requirements Document，软件需求规格说明书 | 本文档体系的根文档 |
| NFR | Non-Functional Requirement，非功能性需求 | 性能/安全/可用性等质量属性 |
| V1.0 | 初始发布版本，90 条功能需求 | SRD 基线 |
| V1.2 | 迭代版本，在 V1.0 基础上新增 10 模块 40 条需求 | 当前目标版本 |

## 录音状态机

| 状态 | 定义 | 触发条件 |
|------|------|----------|
| Idle | 空闲，等待用户操作 | 初始状态 / 录音结束后 |
| Recording | 录音中，麦克风采集或 BLE 设备录制 | 用户点击录音按钮 |
| Uploading | 音频上传中，S3 presigned PUT 直传 | 录音完成后自动触发 |
| Processing | 服务端处理中（转录 + 摘要 + 任务提取） | 上传成功后 BACKEND 触发 AI 流水线 |
| Complete | 处理完成，转录/摘要/任务全部就绪 | AI 全部任务完成回调 |
| Error | 处理异常（上传失败/转录失败/摘要失败） | 任一环节异常 |
| Regenerating | 重新生成中（转录或摘要） | 用户触发重新生成 |

## 通信与协议

| 术语 | 定义 | 使用场景 |
|------|------|----------|
| BLE | Bluetooth Low Energy，低功耗蓝牙 | APP 与 Memoket 录音笔的通信协议 |
| BLE GATT | Generic Attribute Profile，BLE 通用属性协议 | 音频数据传输 + 设备命令控制 |
| SSE | Server-Sent Events，服务端推送事件 | 前台实时通知 + AI 流式输出 |
| FCM | Firebase Cloud Messaging | 后台/杀死状态推送通知 |
| gRPC | Google Remote Procedure Call | BACKEND API 层 → RPC 层内部调用 |
| REST | Representational State Transfer | APP → BACKEND 主要通信方式 |
| OAuth | Open Authorization，开放授权协议 | Google/Apple 登录 + Slack/Notion/Google Calendar 集成 |
| Webhook | 回调通知，第三方主动推送变更事件 | Google Calendar 事件变更实时通知 |
| AES-CBC | 对称加密算法（CBC 模式） | APP ↔ BACKEND 请求/响应体加密 |
| JWT | JSON Web Token | 用户身份鉴权令牌 |
| Presigned URL | 预签名 URL | S3 直传音频文件，绕过 BACKEND |

## AI 与算法

| 术语 | 定义 | 使用场景 |
|------|------|----------|
| ASR | Automatic Speech Recognition，自动语音识别 | 音频转文字（Azure Speech / AssemblyAI） |
| RAG | Retrieval-Augmented Generation，检索增强生成 | AI Insight 跨录音问答的核心算法 |
| LLM | Large Language Model，大语言模型 | OpenAI / Azure OpenAI / Anthropic / Qwen |
| InFileChat | 文件内 AI 对话，上下文为当前录音文件 | 单录音 Q&A |
| CrossFileChat | 跨文件 AI 对话，上下文为多个录音 | AI Insight 功能 |
| Agent | AI 智能体模式，含意图识别、规划与工具调用 | Agent 报告 + Web Browsing |
| Voice Print | 声纹识别，自动识别说话人 | 转录结果中的说话人标注 |
| Knowledge Graph | 知识图谱，跨录音语义关系网络 | 片段关系识别与分组 |
| Function Calling | LLM 函数调用，AI 选择并执行外部工具 | Agent 模式的工具编排 |
| Celery | Python 分布式任务队列 | AI 异步任务执行（转录/分析/执行） |
| Redis Stream | Redis 流数据结构 | SSE 流式缓冲 + 通知事件传递 |
| pgvector | PostgreSQL 向量扩展 | RAG 语义检索的向量存储 |

## 产品功能术语

| 术语 | 定义 | 使用场景 |
|------|------|----------|
| Template Community | 模板社区，浏览/搜索/收藏摘要模板 | 发现页 + AI 推荐弹窗 |
| MCP | Model Context Protocol，AI 工具集成协议 | 第三方 AI 工具（如 ChatGPT/Claude）访问 Memoket 数据 |
| AI Insight | 跨录音智能问答 | 基于 RAG 的全局知识库查询 |
| Summary Template | 摘要模板，定义摘要输出格式 | Markdown / One-pager / 自定义模板 |
| OTA | Over-The-Air，空中升级 | BLE 录音笔固件远程更新 |
| Hotspot Transfer | 热点传输模式 | BLE 设备通过 WiFi 热点高速传输音频 |

## 基础设施

| 术语 | 定义 | 使用场景 |
|------|------|----------|
| S3 | Amazon Simple Storage Service | 音频文件 + 导出文件云存储 |
| CloudFront | AWS CDN 服务 | 音频/导出文件的签名分发 |
| etcd | 分布式键值存储 | BACKEND gRPC 服务注册与发现 |
| Hive | Flutter 本地加密存储 | APP 端缓存/配置/录音状态持久化 |
| GetX | Flutter 状态管理/路由/DI 框架 | APP 全局架构基础 |
| Dio | Dart HTTP 客户端 | APP 网络请求（含 AES 加密拦截器） |
| go-zero | Go 微服务框架 | BACKEND API + RPC 层框架 |
| FastAPI | Python 异步 Web 框架 | AI HTTP 服务端点 |
| LiteLLM | LLM 统一接口库 | AI 层多模型路由（OpenAI/Azure/Anthropic/Qwen） |
