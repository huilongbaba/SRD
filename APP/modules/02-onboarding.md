# 02 - 引导页面（Onboarding）

## 1. Overview

- **Objective**: 为首次启动 APP 的用户提供产品功能介绍与权限引导，确保用户了解核心功能后进入登录流程。
- **Scope**: 首次启动引导欢迎页（APP-224）、分步引导向导（APP-225）、跳过引导（APP-226）、功能提示气泡（APP-227）、权限申请引导（APP-228）、引导完成标记持久化（APP-229）。
- **Non-scope**: 登录/注册流程（见 `01-auth`）、硬件配对引导（见 `03-hardware`）、APP 内功能 Tooltip/Coachmark（属各功能模块自行管理）。

> **Status**: 本模块 6 项需求均标记为 🚧 V1.2 planned。当前代码中已有基础实现框架（`OnboardingController` + `OnboardingView` + `OnboardingService`），但内容为 mock 数据，UI 设计未启动。

## 2. Definitions

| 术语 | 定义 | 备注 |
|------|------|------|
| Onboarding | 首次启动引导流程，包含欢迎页、功能介绍滑页、权限申请 | 仅首次启动显示，完成后持久化标记 |
| OnboardingService | 引导页完成状态管理服务，基于 SharedPreferences 存储布尔值 | `shared/services/onboarding_service.dart` |
| OnboardingController | 引导页 GetX 控制器，管理 PageView、页面数据、导航逻辑 | `app/modules/onboarding/controllers/onboarding_controller.dart` |
| OnboardingPage | 引导页数据模型（title, description, imagePath） | 定义在 `OnboardingController` 内部 |
| PageController | Flutter PageView 控制器，支持滑动翻页与编程式跳页 | Flutter 原生组件 |
| SharedPreferences | 轻量级键值持久化，存储引导完成标记 | key: `onboarding_completed` |
| 分步引导 | 4 步向导：1.录音 2.导入 3.生成 4.AI 问答 | APP-225，具体步骤待 UED 确认 |
| 功能提示 | 新功能气泡提示（Tooltip/Coachmark） | APP-227，首次使用某功能时弹出 |

## 3. System Boundary

| 组件 | 职责 | 不负责 |
|------|------|--------|
| `OnboardingController` | 管理引导页页面数据（从 JSON/默认数据加载）、PageView 翻页控制、完成引导触发 | 权限申请的具体 OS 弹窗 |
| `OnboardingView` | 渲染引导页 UI：PageView（图片+标题+描述）、分页指示器、底部按钮（Get Started） | 引导页内容设计 |
| `OnboardingService` | 引导完成标记的读取（`isCompleted`）、写入（`setCompleted`）、重置（`reset`） | 引导页 UI 逻辑 |
| `RouteInitializer` | APP 启动时检查引导完成状态，决定首路由（Onboarding / Login / Home） | 引导页逻辑 |
| `OnboardingBinding` | GetX 懒绑定，注册 `OnboardingController` | 业务逻辑 |
| `PermissionUtils` | 🚧 权限申请引导中调用的权限检查/请求工具 | 引导流程编排 |

## 4. Scenarios

### S1: 首次启动 - 完整引导流程
- **Trigger**: 用户首次安装 APP 后启动
- **Steps**:
  1. `RouteInitializer.getInitialRoute()` 调用 `OnboardingService.isCompleted()` 返回 `false`
  2. 路由跳转至 `Routes.ONBOARDING`
  3. `OnboardingController.onInit()` 初始化 `PageController`，调用 `_loadMockData()` 加载引导页数据
  4. 从 `assets/mock/onboarding_pages.json` 加载页面数据（title + description + imagePath）
  5. 若 JSON 加载失败，使用默认 3 页数据作为后备
  6. 用户左右滑动 PageView 浏览各引导页
  7. 分页指示器实时更新当前页索引
  8. 用户点击 "Get Started" 按钮
  9. `OnboardingController.completeOnboarding()` 调用 `OnboardingService.setCompleted()`
  10. SharedPreferences 写入 `onboarding_completed = true`
  11. `Get.offAllNamed(Routes.LOGIN)` 跳转登录页
- **Expected**: 引导完成标记已持久化，后续启动不再显示引导页

### S2: 首次启动 - 跳过引导 🚧
- **Trigger**: 用户在引导页点击 "跳过" 按钮
- **Steps**:
  1. 直接调用 `completeOnboarding()` 
  2. 标记完成并跳转登录页
- **Expected**: 等同于完成引导，后续不再显示
- **Note**: 当前 UI 仅有 "Get Started" 按钮，"跳过" 按钮待 V1.2 UED 设计

### S3: 非首次启动 - 跳过引导页
- **Trigger**: 已完成过引导的用户启动 APP
- **Steps**:
  1. `RouteInitializer` 调用 `OnboardingService.isCompleted()` 返回 `true`
  2. 继续检查登录状态，跳转 Login 或 Home
- **Expected**: 用户不会看到引导页

### S4: 引导页 JSON 加载失败
- **Trigger**: `assets/mock/onboarding_pages.json` 文件损坏或不存在
- **Steps**:
  1. `rootBundle.loadString()` 抛出异常
  2. `catch` 块使用默认 3 页数据（i18n key 驱动）
  3. 用户正常浏览默认引导页
- **Expected**: 不影响引导流程，使用后备数据

### S5: 权限申请引导 🚧
- **Trigger**: 引导流程中的权限申请步骤（APP-228）
- **Steps**:
  1. 引导页内展示权限说明（蓝牙、麦克风、通知等）
  2. 用户点击授权按钮
  3. 系统弹出 OS 级权限弹窗
  4. 授权完成后进入下一步
- **Expected**: 提前获取必要权限，减少后续使用中的中断
- **Note**: 4-30 版本计划实现（APP-228: `4-30版本=YES`）

### S6: 功能提示气泡 🚧
- **Trigger**: 用户首次进入某功能页面（APP-227）
- **Steps**:
  1. 检查该功能的提示是否已展示过
  2. 若未展示，显示气泡提示
  3. 用户点击关闭或自动消失
  4. 标记已展示
- **Expected**: 每个功能提示仅显示一次
- **Note**: V1.2 计划，UED 未开始

## 5. Functional Requirements

| ID | 描述 | 级别 | 验证方法 |
|----|------|------|----------|
| ONB-FR-001 | 系统 MUST 在首次启动时显示引导页，通过 `OnboardingService.isCompleted()` 判断 | MUST | 清除 APP 数据 -> 启动 -> 验证看到引导页 |
| ONB-FR-002 | 引导页 MUST 使用 PageView 支持左右滑动翻页，动画时长 300ms，曲线 easeInOut | MUST | 滑动引导页 -> 验证翻页动画流畅 |
| ONB-FR-003 | 引导页 MUST 显示分页指示器（圆点），实时反映当前页索引 | MUST | 翻页 -> 验证指示器同步高亮 |
| ONB-FR-004 | 引导页 MUST 支持通过 "Get Started" 按钮完成引导并跳转登录页 | MUST | 点击按钮 -> 验证跳转登录页 |
| ONB-FR-005 | 完成引导后系统 MUST 将 `onboarding_completed=true` 持久化到 SharedPreferences | MUST | 完成引导 -> 关闭 APP -> 重新打开 -> 验证不再显示引导 |
| ONB-FR-006 | 引导页内容 MUST 从 `assets/mock/onboarding_pages.json` 加载，加载失败时 MUST 使用默认数据（3 页 i18n 文案）作为后备 | MUST | 删除 JSON 文件 -> 启动 -> 验证显示默认引导页 |
| ONB-FR-007 🚧 | 引导页 SHOULD 包含 4 步向导（录音 -> 导入 -> 生成 -> AI 问答） | SHOULD | 验证 4 步内容正确展示 |
| ONB-FR-008 🚧 | 引导页 SHOULD 提供 "跳过" 按钮，点击后直接完成引导并跳转 | SHOULD | 点击跳过 -> 验证标记已持久化 |
| ONB-FR-009 🚧 | 引导页内 SHOULD 支持权限申请引导（蓝牙、麦克风、通知等） | SHOULD | 权限引导步骤 -> 点击授权 -> 验证权限已获取 |
| ONB-FR-010 🚧 | 系统 SHOULD 支持新功能气泡提示，每个提示仅显示一次，标记持久化 | SHOULD | 首次进入功能 -> 看到提示; 再次进入 -> 不再显示 |
| ONB-FR-011 🚧 | 引导完成标记 SHOULD 支持重置（`OnboardingService.reset()`），用于测试场景 | SHOULD | 调用 reset -> 重启 -> 验证引导页再次显示 |
| ONB-FR-012 | 引导页每页 MUST 显示插图区域（优先使用 imagePath 图片，无图片时显示默认 icon） | MUST | 有图引导页显示图片; 无图引导页显示默认手机 icon |

## 6. State Model

```stateDiagram-v2
    [*] --> CheckCompleted : APP 启动
    CheckCompleted --> Completed : isCompleted=true
    CheckCompleted --> Loading : isCompleted=false

    Completed --> [*] : 跳过引导 -> Login/Home

    Loading --> PageReady : 数据加载完成
    Loading --> PageReady_Fallback : 加载失败 (使用默认数据)

    PageReady --> Browsing : 显示第1页
    PageReady_Fallback --> Browsing : 显示第1页

    Browsing --> Browsing : 滑动翻页 (pageIndex++)
    Browsing --> Completing : 点击 Get Started / 跳过

    Completing --> Completed : setCompleted() 成功
    Completed --> [*] : offAllNamed(LOGIN)
```

### 状态定义表

| 状态 | 含义 | 进入条件 | 退出条件 | 代码枚举 |
|------|------|----------|----------|----------|
| CheckCompleted | 检查引导是否已完成 | APP 启动时 RouteInitializer 调用 | 检查结果返回 | `OnboardingService.isCompleted()` |
| Loading | 引导页数据加载中 | 首次启动且未完成引导 | JSON 加载完成或失败 | `_loadMockData()` 执行中 |
| PageReady | 引导页数据就绪 | 数据加载成功 | 用户开始浏览 | `pages.value` 非空 |
| Browsing | 用户浏览引导页 | 引导页渲染完成 | 用户完成/跳过 | `currentPageIndex` 变化中 |
| Completing | 正在标记引导完成 | 用户点击完成/跳过 | SharedPreferences 写入完成 | `OnboardingService.setCompleted()` 执行中 |
| Completed | 引导已完成 | 标记已持久化 | 路由跳转 | `onboarding_completed=true` |

### 非法状态转移

| 不允许 | 原因 | 防御 |
|--------|------|------|
| Completed -> Browsing | 已完成的引导不应重新显示 | `RouteInitializer` 检查 `isCompleted()` |
| Loading -> Completed (跳过数据加载) | 必须先加载/回退数据才能展示 | `_loadMockData` 是 `onInit` 中的必经路径 |

## 7. Data Contract

### API

本模块无后端 API 调用。所有数据为本地资源或本地存储。

### 数据模型

**OnboardingPage（Controller 内部定义）**

| 字段 | 类型 | 必填 | 单位 | 示例 |
|------|------|------|------|------|
| title | String | Y | - | `"Smart Recording"` |
| description | String | Y | - | `"Record meetings and lectures..."` |
| imagePath | String? | N | asset path | `"assets/images/onboarding_1.png"` |

**SharedPreferences Key**

| Key | 类型 | 默认值 | 说明 |
|-----|------|--------|------|
| `onboarding_completed` | bool | `false` | 引导完成标记 |

**assets/mock/onboarding_pages.json 结构**

```json
{
  "pages": [
    {
      "title": "...",
      "description": "...",
      "imagePath": "assets/images/onboarding_1.png"
    }
  ]
}
```

### 协议分层

不适用（本模块无网络通信）。

## 8. Error Handling

| Case | 触发条件 | 系统行为 | 状态变化 | 用户感知 |
|------|----------|----------|----------|----------|
| JSON 资源加载失败 | `rootBundle.loadString()` 抛异常（文件不存在/损坏） | catch -> 使用默认 3 页数据，LoggerUtils.e 记录错误 | Loading -> PageReady_Fallback | 无感知（显示默认引导页） |
| JSON 解析失败 | `jsonDecode` 抛异常 | 同上 | Loading -> PageReady_Fallback | 无感知 |
| SharedPreferences 写入失败 | 存储空间不足或权限问题 | 异常向上传播（当前未 catch） | Completing 卡住 | 可能重复显示引导页 |
| PageView 索引越界 | `goToPage(index)` 传入无效索引 | 条件判断 `if (index >= 0 && index < totalPages)` 过滤 | 无变化 | 无感知 |
| pages 列表为空 | 加载与回退均失败（极端） | Obx 检查 `pages.isEmpty` 显示 CircularProgressIndicator | 停留在 Loading | 用户看到加载动画 |

## 9. Non-functional Requirements

| 指标 | 要求 | 实测值 | 来源 |
|------|------|--------|------|
| 引导页加载时间 | < 500ms（从路由进入到首页渲染） | 待测 | 本地 JSON + asset 图片 |
| 翻页动画帧率 | >= 60fps | 待测 | `PageController` + Curves.easeInOut |
| 翻页动画时长 | 300ms | 300ms（硬编码） | `onboarding_controller.dart:99` |
| SharedPreferences 写入延迟 | < 50ms | 待测 | 单个 bool 值写入 |
| 引导页内存占用 | < 50MB（含图片资源） | 待测 | asset 图片大小决定 |
| 国际化支持 | 支持 en / zh | 已实现 | i18n key: `onboarding.*` |

## 10. Observability

### Logs

| 事件 | 级别 | 携带字段 | 组件 |
|------|------|----------|------|
| Mock 数据加载失败 | ERROR | error, stackTrace | `OnboardingController` (tag: 'OnboardingController') |
| 路由初始化 - 用户已登录 | INFO | - | `RouteInitializer` (tag: 'RouteInitializer') |

### Metrics

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| onboarding_completion_rate | 引导页完成率（完成/跳过 vs 中途退出） | < 70%（需关注引导设计是否合理） |
| onboarding_skip_rate 🚧 | 跳过引导的用户占比 | 观测用，无阈值 |
| onboarding_avg_pages_viewed | 平均浏览引导页数 | 观测用（低于 2 页说明内容不吸引） |
| onboarding_load_fallback_rate | JSON 加载失败使用后备数据的比率 | > 1%（需排查 asset 打包问题） |
