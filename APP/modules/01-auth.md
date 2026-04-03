# 01 - 用户认证（登录/注册）

## 1. Overview

- **Objective**: 提供用户注册、登录及身份验证功能，支持 Email/Google/Apple 三种认证方式，完成后进入 APP 工作台。
- **Scope**: 登录首屏、Google OAuth 登录、Apple OAuth 登录、Email 邮箱登录、Email 注册（邮箱+验证码+密码）、忘记密码/重置密码、国家/地区选择、Token 持久化与自动登录、Firebase Analytics 登录事件埋点。
- **Non-scope**: 引导页（见 `02-onboarding`）、硬件配对引导（见 `03-hardware`）、用户个人信息编辑（`user_center` 模块）、第三方集成 OAuth（Slack/Google/Notion 集成由 `integrations` 模块负责）。

## 2. Definitions

| 术语 | 定义 | 备注 |
|------|------|------|
| Firebase Auth | Google 提供的身份认证服务，APP 端 Google 登录通过 Firebase Auth 获取 ID Token | 仅 Google 登录使用 Firebase Auth，Apple/Email 不经过 Firebase |
| OAuth | 开放授权协议，用于 Google/Apple 第三方登录 | Google 使用 Firebase + Google Sign-In SDK；Apple 使用 `sign_in_with_apple` SDK |
| Backend Token | APP 后端签发的 `accessToken`，用于后续所有 API 调用的身份凭证 | 所有登录方式最终均调用后端 `/api/v1/users/login` 获取此 Token |
| LoginData | 后端登录接口返回的数据模型，包含 `accessToken`、`userId`、`clientId` | `shared/models/login_data.dart` |
| UserInfo | 用户基本信息模型（userId, avatar, nickname, email, status, gender, birthday） | `shared/models/user_info.dart` |
| UserService | 用户 Token 与信息管理服务，基于命名空间隔离的加密本地存储 | `shared/services/user_service.dart` |
| AuthService | Firebase Auth + Google/Apple Sign-In 封装的认证服务（GetxService 单例） | `shared/services/auth_service.dart` |
| ClientId | 设备唯一标识符，由 `DeviceClientIdService` 管理，登录时发送给后端 | 后端可能返回服务端分配的 clientId |
| login_type | 后端登录类型标识：1=Email, 2=Google, 3=Apple | 作为 `/api/v1/users/login` 请求参数 |

## 3. System Boundary

| 组件 | 职责 | 不负责 |
|------|------|--------|
| `LoginController` | 登录首屏逻辑：协议同意、Google/Apple 登录触发、国家选择、登录后跳转 | 邮箱登录表单逻辑 |
| `EmailLoginController` | 邮箱+密码登录、输入校验、防重复提交、跳转注册/忘记密码 | OAuth 登录 |
| `RegisterController` | 注册流程：邮箱+验证码+密码、校验规则、注册 API 调用 | 登录逻辑 |
| `AuthService` | Firebase Auth 管理、Google Sign-In SDK 调用、Apple Sign-In SDK 调用、ID Token 获取 | 后端 API 调用、Token 存储 |
| `UserApi` | 后端认证 API 抽象层（`loginWithEmail` / `loginWithGoogle` / `loginWithApple` / `register` / `sendRegisterEmailCaptcha`） | 本地状态管理 |
| `UserService` | Token 加密存储、UserInfo 持久化、`isLoggedIn` 判断 | 认证流程编排 |
| `RouteInitializer` | 启动时决定初始路由：Onboarding -> Login -> Home | 登录/注册业务逻辑 |
| `LoadingService` | 全局 Loading 遮罩 | 业务逻辑 |
| `ErrorHandlerService` | 全局错误处理与 Toast 提示 | 业务逻辑 |
| `AnalyticsService` | Firebase Analytics 登录事件打点 | 业务逻辑 |
| `PushNotificationService` | 登录成功后注册 FCM Token | 认证流程 |

## 4. Scenarios

### S1: Google 登录 - 正常路径
- **Trigger**: 用户在登录首屏点击 "Continue with Google"
- **Steps**:
  1. `LoginController.signInWithGoogle()` 显示 Loading
  2. 检查 `AuthService` 是否已注册
  3. `AuthService.signInWithGoogle()` 触发 Google Sign-In 流程
  4. 用户在 Google 选择器中选择账号并授权
  5. 获取 `GoogleSignInAuthentication`（含 `idToken` + `accessToken`）
  6. 创建 `GoogleAuthProvider.credential` 并调用 `Firebase.signInWithCredential`
  7. 返回 `UserCredential` 给 `LoginController`
  8. `LoginController._handleLoginSuccess()` 调用 `UserApi.loginWithGoogle(googleIdToken)` 
  9. 后端返回 `LoginData`（含 `accessToken` + `userId` + `clientId`）
  10. 保存 Token（`UserService.saveToken`）、注册 FCM（`PushNotificationService.onUserLoggedIn`）
  11. 构建 `UserInfo` 并保存
  12. 记录 Analytics 事件 `logLogin(method: 'google')`
  13. `Get.offAllNamed(Routes.HOME)` 跳转首页
- **Expected**: 用户进入 APP 工作台，Token 已持久化

### S2: Apple 登录 - 正常路径
- **Trigger**: 用户在登录首屏点击 "Continue with Apple"
- **Steps**:
  1. `LoginController.signInWithApple()` 显示 Loading
  2. `AuthService.signInWithApple()` 调用 `SignInWithApple.getAppleIDCredential`
  3. 获取 `appleCredential`（含 `authorizationCode` + `email` + `givenName`）
  4. 返回 `AppleSignInResult` 给 `LoginController`
  5. `LoginController._handleAppleLoginSuccess()` 调用 `UserApi.loginWithApple(appleAuthCode)`
  6. 后端返回 `LoginData`，保存 Token 及 UserInfo
  7. 跳转首页
- **Expected**: 用户进入 APP 工作台
- **Note**: Apple 登录不经过 Firebase Auth，直接使用 `authorizationCode` 调后端

### S3: 邮箱登录 - 正常路径
- **Trigger**: 用户选择 "Continue with Email"，输入邮箱密码，点击登录
- **Steps**:
  1. `EmailLoginController.login()` 收起键盘，检查防重复 `_isLoggingIn`
  2. 调用 `UserApi.loginWithEmail(email, password)`
  3. 后端返回 `LoginData`，保存 Token
  4. 调用 `UserService.fetchUserInfo()` 从服务器获取完整用户信息
  5. 若 nickname 为空，使用邮箱前缀作为默认值
  6. 记录 Analytics 事件 `logLogin(method: 'email')`
  7. `Get.offAllNamed(Routes.HOME)`
- **Expected**: 用户进入 APP 工作台

### S4: 邮箱注册 - 正常路径
- **Trigger**: 用户在邮箱登录页点击 "注册"
- **Steps**:
  1. 跳转注册页面 `RegisterController`
  2. 用户输入邮箱 -> 点击发送验证码 -> `UserApi.sendRegisterEmailCaptcha(email)`
  3. 用户输入验证码 + 密码 -> 点击注册
  4. `RegisterController.register()` 调用 `UserApi.register(email, password, captcha)`
  5. 后端返回 `RegisterData`，保存 Token
  6. 跳转首页
- **Expected**: 新用户注册成功并进入 APP

### S5: Google 登录 - 用户取消
- **Trigger**: 用户在 Google 选择器中点击取消
- **Steps**:
  1. `GoogleSignIn.signIn()` 返回 `null`
  2. `AuthService.signInWithGoogle()` 返回 `null`
  3. `LoginController` 隐藏 Loading，不做任何跳转
- **Expected**: 停留在登录首屏，无错误提示

### S6: Google 登录 - Firebase Auth 超时
- **Trigger**: 网络不稳定，Firebase `signInWithCredential` 超过 30 秒
- **Steps**:
  1. `auth.signInWithCredential` 的 `.timeout(30s)` 触发
  2. 抛出 `FirebaseAuthException(code: 'timeout')`
  3. `AuthService` catch 并显示 Toast 错误
- **Expected**: 用户看到网络错误提示，停留在登录页

### S7: 邮箱登录 - 密码错误
- **Trigger**: 用户输入错误密码
- **Steps**:
  1. `UserApi.loginWithEmail` 返回 `isSuccess=false`
  2. 显示后端返回的错误消息或默认 `login_failed` 文案
- **Expected**: Toast 提示登录失败

### S8: 自动登录 - Token 有效
- **Trigger**: APP 启动
- **Steps**:
  1. `RouteInitializer.getInitialRoute()` 检查 OnboardingService.isCompleted
  2. 检查 `UserService.isLoggedIn`（Token 存在）
  3. 返回 `Routes.HOME`
- **Expected**: 直接进入首页，无需重新登录

## 5. Functional Requirements

| ID | 描述 | 级别 | 验证方法 |
|----|------|------|----------|
| AUTH-FR-001 | 系统 MUST 支持通过 Google OAuth 2.0 登录，使用 Firebase Auth + Google Sign-In SDK 获取 `idToken` 后调用后端 `/api/v1/users/login` (login_type=2) | MUST | 点击 Google 登录 -> 授权 -> 验证跳转至首页且 Token 已持久化 |
| AUTH-FR-002 | 系统 MUST 支持通过 Apple Sign-In 登录，获取 `authorizationCode` 后调用后端 `/api/v1/users/login` (login_type=3) | MUST | 点击 Apple 登录 -> 授权 -> 验证跳转至首页 |
| AUTH-FR-003 | 系�� MUST 支持通过 Email + Password 登录，调用后端 `/api/v1/users/login` (login_type=1) | MUST | 输入有效邮箱+密码 -> 点击登录 -> 验证跳转至首页 |
| AUTH-FR-004 | 系统 MUST 支持 Email 注册流程：发送验证码 -> 输入验证码 + 密码 -> 调用 `/api/v1/users/register` | MUST | 完整注册流程 -> 验证新用户可登录 |
| AUTH-FR-005 | 系统 MUST 在登录成功后将 `accessToken` 加密持久化到本地存储（Hive），支持 APP 重启后自动登录 | MUST | 登录后关闭 APP -> 重新打开 -> 验证直接进入首页 |
| AUTH-FR-006 | 系统 MUST 在登录成功后调用 `PushNotificationService.onUserLoggedIn()` 注册 FCM Token | MUST | 登录后检查 FCM Token 注册请求已发送 |
| AUTH-FR-007 | 邮箱登录 MUST 实现防重复提交：`_isLoggingIn` 为 true 时忽略后续点击 | MUST | 快速双击登录按钮 -> 验证仅发送一次 API 请求 |
| AUTH-FR-008 | 邮箱+密码输入 MUST 实时校验：邮箱格式（`ValidationUtils.validateEmail`）和密码规则（`ValidationUtils.validatePassword`）均通过时才启用登录按钮 | MUST | 输入无效邮箱 -> 按钮置灰；输入有效邮箱+密码 -> 按钮可点击 |
| AUTH-FR-009 | 注册页面 MUST 支持发送邮箱验证码，调用 `UserApi.sendRegisterEmailCaptcha` | MUST | 输入邮箱 -> 点击发送 -> 验证收到验证码邮件 |
| AUTH-FR-010 | 登录首屏 SHOULD 支持国家/地区选择，通过 `CountryService` 持久化并同步到 HTTP Header | SHOULD | 选择国家 -> 关闭 APP -> 重新打开 -> 验证国家设置保留 |
| AUTH-FR-011 | 系统 MUST 在 Google 登录时根据环境（dev/qa/uat）使用对应的 `serverClientId` 和 `iosClientId` | MUST | 在 UAT 环境下 Google 登录 -> 验证使用 UAT 的 Client ID |
| AUTH-FR-012 | Apple 登录 MUST 请求 `email` 和 `fullName` scopes | MUST | Apple 首次登录 -> 验证获取到 email 和姓名 |
| AUTH-FR-013 | 系统 MUST 在登录成功后记录 Firebase Analytics 事件：`logLogin(method)` + `setUserProperties(userId, preferredLanguage)` | MUST | 登录后检查 Firebase Analytics 事件日志 |
| AUTH-FR-014 | 系统 MUST 支持忘记密码流程：验证邮箱 -> 发送验证码 -> 重置密码 | MUST | 完整忘记密码流程 -> 验证新密码可用 |
| AUTH-FR-015 | 当 `AuthService` 或 `UserApi` 未注册时，系统 MUST 显示 `service_not_initialized` 错误提示并终止流程 | MUST | 模拟服务未注册场景 -> 验证 Toast 提示且不 crash |

## 6. State Model

```stateDiagram-v2
    [*] --> AppLaunch
    AppLaunch --> OnboardingCheck : RouteInitializer
    OnboardingCheck --> Onboarding : 未完成引导
    OnboardingCheck --> TokenCheck : 已完成引导
    Onboarding --> LoginScreen : 完成引导
    TokenCheck --> Home : Token有效 (isLoggedIn=true)
    TokenCheck --> LoginScreen : 无Token

    LoginScreen --> GoogleSignIn : 点击 Google
    LoginScreen --> AppleSignIn : 点击 Apple
    LoginScreen --> EmailLogin : 点击 Email
    LoginScreen --> Register : 点击注册

    GoogleSignIn --> GoogleAuth : 触发 Google Sign-In SDK
    GoogleAuth --> FirebaseCredential : 获取 idToken
    GoogleAuth --> LoginScreen : 用户取消 (null)
    FirebaseCredential --> BackendLogin : signInWithCredential 成功
    FirebaseCredential --> LoginScreen : Firebase Auth 失败

    AppleSignIn --> AppleAuth : 触发 Apple Sign-In SDK
    AppleAuth --> BackendLogin : 获取 authorizationCode
    AppleAuth --> LoginScreen : 用户取消/失败

    EmailLogin --> BackendLogin : 邮箱+密码校验通过
    EmailLogin --> LoginScreen : 密码错误

    Register --> SendCaptcha : 输入邮箱
    SendCaptcha --> VerifyCaptcha : 发送验证码
    VerifyCaptcha --> BackendRegister : 验证通过+设置密码
    BackendRegister --> BackendLogin : 注册成功

    BackendLogin --> SaveToken : 后端返回 LoginData
    SaveToken --> SaveUserInfo : Token 已持久化
    SaveUserInfo --> RegisterFCM : UserInfo 已保存
    RegisterFCM --> LogAnalytics : FCM Token 已注册
    LogAnalytics --> Home : offAllNamed(HOME)

    BackendLogin --> LoginScreen : API 失败
```

### 状态定义表

| 状态 | 含义 | 进入条件 | 退出条件 | 代码枚举 |
|------|------|----------|----------|----------|
| LoginScreen | 登录首屏 | APP 启动且无有效 Token | 用户选择登录方式 | `Routes.LOGIN` |
| GoogleAuth | Google OAuth 进行中 | 用户点击 Google 登录 | 授权成功/取消/失败 | `GoogleSignIn.signIn()` 执行中 |
| AppleAuth | Apple OAuth 进行中 | 用户点击 Apple 登录 | 授权成功/取消/失败 | `SignInWithApple.getAppleIDCredential()` 执行中 |
| FirebaseCredential | Firebase 凭证验证中 | Google 授权成功 | Firebase signIn 成功/失败/超时 | `auth.signInWithCredential()` 执行中 |
| BackendLogin | 后端登录 API 调用中 | 第三方/邮箱认证成功 | 后端返回成功/失败 | `UserApi.loginWith*()` 执行中 |
| SaveToken | Token 持久化中 | 后端返回 LoginData | Token 保存完成 | `UserService.saveToken()` 执行中 |
| Home | APP 首页/工作台 | Token 已持久化 + UserInfo 已保存 | 用户登出 | `Routes.HOME` |
| EmailLogin_LoggingIn | 邮箱登录进行中 | 用户点击登录 | API 返回 | `_isLoggingIn.value = true` |

### 非法状态转移

| 不允许 | 原因 | 防御 |
|--------|------|------|
| LoginScreen -> Home (无 Token) | 必须完成认证才能进入 | `RouteInitializer` 检查 `UserService.isLoggedIn` |
| EmailLogin_LoggingIn -> EmailLogin_LoggingIn | 禁止重复提交 | `if (_isLoggingIn.value) return` |
| GoogleAuth -> BackendLogin (无 idToken) | 缺少必要凭证 | `if (googleAuth.idToken == null)` 返回 null |
| AppleAuth -> BackendLogin (无 authorizationCode) | 缺少必要凭证 | `if (appleCredential.authorizationCode.isEmpty)` 返回 null |

## 7. Data Contract

### API

| 方法 | 路径 | 请求体 | 响应体 | 错误码 |
|------|------|--------|--------|--------|
| POST | `/api/v1/users/login` | `{ login_type: 1, client_id, email, password }` | `LoginData { accessToken, userId, clientId }` | 401: 密码错误; 403: 账号禁用 |
| POST | `/api/v1/users/login` | `{ login_type: 2, client_id, google_id_token }` | `LoginData { accessToken, userId, clientId }` | 401: Token 无效 |
| POST | `/api/v1/users/login` | `{ login_type: 3, client_id, apple_auth_code, nickname? }` | `LoginData { accessToken, userId, clientId }` | 401: AuthCode 无效 |
| POST | `/api/v1/users/register` | `{ email, password, captcha }` | `RegisterData { accessToken, userId }` | 409: 邮箱已注册; 400: 验证码错误 |
| POST | `/api/v1/users/register/email-captcha` | `{ email, reconfirm? }` | `EmailCaptchaData` | 429: 频率限制 |

### 数据模型

**LoginData**

| 字段 | 类型 | 必填 | 单位 | 示例 |
|------|------|------|------|------|
| accessToken | String | Y | - | `"eyJhbGciOi..."` |
| userId | String | Y | - | `"usr_abc123"` |
| clientId | String? | N | - | `"cli_xyz789"` |

**UserInfo**

| 字段 | 类型 | 必填 | 单位 | 示��� |
|------|------|------|------|------|
| userId | String | Y | - | `"usr_abc123"` |
| avatar | String | Y | URL | `"https://lh3.googleusercontent.com/..."` |
| nickname | String | Y | - | `"John Doe"` |
| email | String | Y | - | `"john@example.com"` |
| status | int | Y | - | `1` (正常) |
| gender | int | Y | - | `0` (未知) |
| birthday | String? | N | ISO 8601 | `"1990-01-01"` |

**AppleSignInResult**

| 字段 | 类型 | 必填 | 单位 | 示例 |
|------|------|------|------|------|
| appleAuthCode | String | Y | - | `"c1234..."` |
| email | String? | N | - | `"user@privaterelay.appleid.com"` |
| userId | String? | N | - | `"001234.abcdef..."` |
| givenName | String? | N | - | `"John"` |
| familyName | String? | N | - | `"Doe"` |

## 8. Error Handling

| Case | 触发条件 | 系统行为 | 状态变化 | 用户感知 |
|------|----------|----------|----------|----------|
| Google 用户取消 | `GoogleSignIn.signIn()` 返回 null | 隐藏 Loading，不做操作 | 保持 LoginScreen | 无感知 |
| Apple 用户取消 | `AuthorizationErrorCode.canceled` | catch 并显示 `login_cancelled` Toast | 保持 LoginScreen | Toast: 登录已取消 |
| Firebase Auth 网络错误 | `FirebaseAuthException(code: 'network-request-failed')` | 显示 `auth.network_error` Toast | 保持 LoginScreen | Toast: 网络错误 |
| Firebase Auth 超时 | `signInWithCredential` 超过 30s | 抛出 timeout 异常，显示错误 | 保持 LoginScreen | Toast: 登录失败 |
| Firebase Auth 连接关闭 | `e.message.contains('connection closed')` | 日志记录可能原因，显示网络错误 | 保持 LoginScreen | Toast: 网络错误 |
| 后端 API 失败 | `result.isSuccess == false` | 显示 `result.message` 或默认错误 | 保持 LoginScreen | Toast: 具体错误信息 |
| 邮箱密码错误 | 后端返回登录失败 | 显示 `login_failed` 提示 | 保持 EmailLogin | Toast: 登录失败 |
| AuthService 未注册 | `Get.isRegistered<AuthService>()` 为 false | 显示 `service_not_initialized` Toast | 保持 LoginScreen | Toast: 服务未初始化 |
| UserApi 未注册 | `Get.isRegistered<UserApi>()` 为 false | 日志 error + Toast | 保持 LoginScreen | Toast: 服务未初始化 |
| UserService 未注册 | `Get.isRegistered<UserService>()` 为 false | ErrorHandlerService 处理 | 保持 LoginScreen | Toast: 服务未初始化 |
| 邮箱重复提交 | `_isLoggingIn.value == true` 时再次调用 | `return` 忽略 | 保持 LoggingIn | 无感知（按钮已禁用） |
| 未知异常 | 任何未捕获的 Exception | catch-all -> `ErrorHandlerService.handleError` | 保持当前 | Toast: 登录失败 |
| Google idToken 为 null | Google Auth 返回但无 idToken | 日志 error + Toast + 返回 null | 保持 LoginScreen | Toast: 登录失败 |
| Apple authorizationCode 为空 | Apple 返回但 code 为空 | 日志 error + Toast + 返回 null | 保持 LoginScreen | Toast: 登录失败 |

**Bug 参考**: Ticket-000672 — Notion/Slack OAuth 在嵌入式 WebView 中被 Google 拒绝（403 disallowed_useragent）。此 bug 属于第三方集成模块，但提示认证模块应避免在 WebView 中进行 OAuth。当前 Google/Apple 登录使用原生 SDK，不受此影响。

## 9. Non-functional Requirements

| 指标 | 要求 | 实测值 | 来源 |
|------|------|--------|------|
| 登录 API 响应时间 | < 2000ms (P95) | 待测 | 后端 SLA |
| Google Sign-In SDK 超时 | 30s hard timeout | 30s (`auth.signInWithCredential.timeout`) | 代码 `auth_service.dart:311` |
| Token 持久化延迟 | < 100ms | 待测 | Hive 本地写入 |
| 冷启动自动登录路由判断 | < 200ms | 待测 | `RouteInitializer.getInitialRoute()` |
| 邮箱校验实时性 | 每次键入即时校验 | 同步 | `TextEditingController.addListener` |
| 防重复提交间隔 | 0ms（基于 flag 硬防护） | N/A | `_isLoggingIn` flag |
| Token 存储安全 | AES 加密 (Hive encrypted box) | 已实现 | `UserService` + Storage Package |
| 密码传输安全 | HTTPS + 请求体加密 | 已实现 | HttpService 拦截器 |

## 10. Observability

### Logs

| 事件 | 级别 | 携带字段 | 组件 |
|------|------|----------|------|
| Google Sign-In 开始 | INFO | - | `AuthService` (tag: 'AuthService') |
| Google Sign-In 账号获取 | DEBUG | email | `AuthService` |
| Google Auth token 获取 | DEBUG | hasAccessToken, hasIdToken | `AuthService` |
| Google idToken payload 解码 | DEBUG | payload JSON | `AuthService` |
| Firebase signInWithCredential 成功 | INFO | email | `AuthService` |
| Firebase Auth 错误 | ERROR | code, message | `AuthService` |
| Apple Sign-In 成功 | INFO | email, userId | `AuthService` |
| Apple Sign-In 授权错误 | ERROR | code, message | `AuthService` |
| Google 登录后端成功 | INFO | email, userId, backendToken(前20字符) | `LoginController` |
| Apple 登录后端成功 | INFO | email, userId, backendToken(前20字符) | `LoginController` |
| 邮箱登录开始 | INFO | - | `EmailLoginController` (tag: implicit) |
| UserApi 未注册 | ERROR | - | `LoginController` |
| 国家列表加载 | DEBUG | count, language | `LoginController` (tag: 'ChoiceAreaController') |
| 国家保存 | DEBUG | countryCode | `LoginController` (tag: 'ChoiceAreaController') |
| 签出成功 | INFO | - | `AuthService` |
| Analytics 打点失败 | WARN | error | `LoginController` / `EmailLoginController` |

### Metrics

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| login_success_rate | 各方式登录成功率 (Google/Apple/Email) | < 95% |
| login_latency_p95 | 从点击到进入首页的端到端耗时 | > 5000ms |
| firebase_auth_timeout_rate | Firebase signInWithCredential 超时率 | > 5% |
| registration_completion_rate | 注册流程完成率（发送验证码 -> 注册成功） | < 60% |
| auto_login_rate | 冷启动自动登录占比 | 观测用，无阈值 |
