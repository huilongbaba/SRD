# APP Overview

## 1. 目标

Memoket APP 是一款跨平台（iOS + Android）智能录音应用，为用户提供 **录音采集 → 云端上传 → AI 转录与摘要 → 智能对话与导出** 的端到端体验，同时支持与自研 BLE 录音笔硬件配对使用。

## 2. 范围

本 SRD 覆盖 APP 端全部功能需求，包括：

- **用户系统**：注册/登录（Google/Apple/Email）、个人资料、设置
- **录音采集**：本地麦克风录音、BLE 设备录音、音频文件导入
- **内容管理**：录音列表、分类管理、搜索、编辑、导出（PDF/DOCX/分享链接）
- **AI 功能展示**：转录详情、摘要展示、InFileChat、CrossFileChat (AI Insight)、Agent 报告
- **设备管理**：BLE 配对/连接/OTA 升级、音频传输
- **V1.2 新增**：模板社区、第三方集成（Slack/Google/Notion）、摘要语言偏好、自定义词汇、录音笔记、MCP 服务入口、Calendar 同步展示

## 3. 非范围（Non-scope）

以下内容 **不在** APP SRD 范围内：

| 非范围项 | 归属 |
|----------|------|
| 后端业务逻辑（用户服务、音频服务、通知分发等） | BACKEND SRD |
| AI 算法实现（ASR 流水线、RAG 检索、摘要模板引擎、Agent 执行） | AI SRD |
| BLE 硬件固件（录音笔固件逻辑、OTA 包构建） | 硬件团队 |
| 运维部署（K8s 编排、CI/CD、监控告警） | DevOps |
| 后台管理系统（admin-api 相关功能） | Admin SRD |

## 4. 技术栈

| 关注点 | 技术选型 |
|--------|----------|
| 框架 | Flutter 3.8+ / Dart |
| 状态管理 | GetX（reactive `.obs` + `GetxController`） |
| 路由 | GetX Router（`GetPage` + lazy bindings，40+ 页面） |
| 依赖注入 | GetX DI（`Get.put` / `Get.find` / `Get.putAsync`） |
| HTTP 客户端 | Dio（本地 `network` 包，含 AES 加密/重试/缓存拦截器） |
| 本地存储 | Hive 加密存储（本地 `storage` 包） |
| BLE 通信 | flutter_blue_plus + 自研 `memoket_ble` SDK |
| 录音 | `record` 包（AAC-LC, 44.1kHz, mono） |
| 播放 | `just_audio` + `audioplayers` |
| 认证 | Firebase Auth（Google/Apple Sign-In）+ 自定义 JWT |
| 推送 | FCM（后台）+ SSE（前台）+ flutter_local_notifications |
| 分析 | Firebase Analytics + Microsoft Clarity |
| 崩溃报告 | Firebase Crashlytics |
| 国际化 | flutter_i18n（JSON asset bundles） |
| 代码生成 | json_serializable + flutter_gen + build_runner |
| 架构模式 | **MVVM**（View ↔ Controller/ViewModel ↔ Service/Repository） |

## 5. 成功指标

| 指标 | 当前值 | 目标 |
|------|--------|------|
| V1.0 功能完成率 | **92.2%**（83/90 完全实现） | 100% |
| V1.0 覆盖率（含部分实现） | **100%**（90/90） | 100% |
| V1.2 提前实现率 | **63.5%**（33/52 已提前完成） | 100%（发布时） |
| V1.2 新增模块完成率 | **97.5%**（39/40 已实现） | 100% |
| NFR 满足率 | **92.3%**（12/13 满足） | 100% |
| 部分实现收尾项 | 7 条（P1 × 3, P2 × 4） | 0 条 |
