# 03 - 硬件连接（BLE 设备配对/OTA/设备管理）

## 1. Overview

- **Objective**: 管理 Memoket 录音笔硬件设备的蓝牙发现、配对、连接、OTA 固件升级及设备全生命周期，提供稳定的 BLE 通信基础设施。
- **Scope**: BLE 设备扫描与发现（APP-148）、RSSI 信号过滤（APP-149）、设备配对（APP-150）、连接管理（APP-151）、连接状态监控（APP-152）、自动重连（APP-153）、设备激活（APP-154）、多设备管理（APP-155）、WiFi 配置（APP-156/157）、OTA 固件升级（APP-160/240）、设备设置同步（APP-161）、电量显示（APP-162）、存储信息（APP-164）、设备信息（APP-165）、设备重命名（APP-166）、设备解绑（APP-167）、设备列表（APP-168）、设备切换（APP-169）、默认设备（APP-170）、设备状态（APP-171）、设备图标（APP-172）、用户引导页（APP-237）、OTA 成功/失败提示（APP-240）、录音增益调节（APP-241）、蓝牙权限管理。
- **Non-scope**: 设备端录音控制（属 `audio_file` 模块的 `DeviceRecordingControlService`）、设备音频下载/传输（属 `audio_file` 模块的 `DeviceAudioService`）、录音列表 UI 中的设备状态卡片（属 `audio_file` 模块）。

## 2. Definitions

| 术语 | 定义 | 备注 |
|------|------|------|
| BLE | Bluetooth Low Energy，低功耗蓝牙协议 | APP 通过 `flutter_blue_plus` + `memoket_ble` SDK 通信 |
| BleManager | `memoket_ble` SDK 提供的蓝牙管理器，封装扫描/连接底层操作 | `bleManager.startScan()` / `bleManager.connect()` |
| BleConnection | `memoket_ble` SDK 的连接对象，支持 `execute<T>(request)` 发送命令 | 通过 GATT 特征值读写 |
| BleDevice | `memoket_ble` SDK 的设备模型（id, name, rssi） | SDK 层模型，APP 层转换为 `BluetoothDevice` |
| BluetoothDevice | APP 层自定义设备模型，包含 connectionStatus, serverId, batteryLevel 等 | `my_device/models/entity/ble_device_model.dart` |
| DeviceBluetoothManager | 设备蓝牙管理器单例（ChangeNotifier），统一管理扫描、连接、状态监听 | `my_device/models/services/device_bluetooth_manager.dart` |
| DeviceCommands | 设备业务指令封装（电量、固件版本、RTC 时间同步、录音控制等） | `my_device/models/services/device_commands.dart` |
| OtaCommands | OTA 升级指令封装（start -> 分片传输 -> end -> 重启 -> 校验版本） | `my_device/models/services/ota_commands.dart` |
| DeviceService | 设备业务逻辑层，负责设备列表丰富化（电量/版本/文件/RTC） | `my_device/models/services/device_service.dart` |
| DeviceStorageService | 已连接设备本地持久化（支持跨 APP 重启恢复设备列表） | `my_device/models/services/device_storage_service.dart` |
| GATT | Generic Attribute Profile，BLE 通信的属性层协议 | APP 通过 Service/Characteristic UUID 通信 |
| OTA | Over-The-Air，空中固件升级 | 通过 BLE 传输固件文件到设备 |
| MTU | Maximum Transmission Unit，BLE 单次传输最大字节数 | 当前固定 511 bytes（514-3） |
| RSSI | Received Signal Strength Indication，信号强度指标 | >= -50: Excellent; >= -70: Good; >= -80: Fair; < -80: Weak |
| FirmwareModule | 固件模块类型枚举：all(0), bluetooth(1), wifi(2) | `device_enums.dart` |
| BoundDevice | 服务端绑定设备模型，包含服务端分配的 ID | `shared/models/bound_device_response.dart` |
| DeviceRecordingSession | 设备录音会话模型，跟踪录音时长/状态 | `my_device/models/entity/device_recording_session.dart` |
| 断连宽限期 | 设备断连后 1 分钟内仍显示录音灰态，超过后进入 Unknown 状态 | `_recordingDisconnectGrace = Duration(minutes: 1)` |

## 3. System Boundary

| 组件 | 职责 | 不负责 |
|------|------|--------|
| `DeviceBluetoothManager` | 单例蓝牙管理：扫描/连接/断开/重连、设备列表维护、连接状态监听、录音状态监听、电量监听、GATT Service 发现 | 具体业务指令执行、UI 渲染 |
| `DeviceCommands` | 基于 `BleConnection` 发送业务命令：获取电量、固件版本、同步 RTC、录音控制（开始/暂停/恢复/停止）、录音增益 | 连接管理、扫描 |
| `OtaCommands` | OTA 升级全流程：加载固件文件 -> MD5 -> OTA start -> 分片传输 -> OTA end -> 重启 -> 重连验证 | 固件下载、UI 进度展示 |
| `DeviceService` | 设备列表业务逻辑：从 BluetoothManager 获取设备快照、丰富化已绑定设备信息（RTC/电量/版本/文件列表）、OTA 固件下载 | 蓝牙底层通信 |
| `AddDeviceController` | 添加设备流程编排：检查蓝牙权限/状态 -> 扫描 -> 选择设备 -> 检查绑定状态 -> 连接 -> 设备端绑定 -> 服务端绑定 | 蓝牙底层操作 |
| `MyDeviceController` | 我的设备列表页：加载设备、状态同步、设备丰富化触发 | 蓝牙通信 |
| `DeviceDetailController` | 设备详情页：设备信息展示、OTA 触发、解绑、传输模式切换 | 蓝牙底层 |
| `BluetoothPermissionController` | 蓝牙权限检查与请求 | 蓝牙连接 |
| `BleDeviceApi` | 后端设备 API：绑定/解绑/设备列表/固件检查/状态上报/绑定状态检查 | 蓝牙通信 |
| `DeviceStorageService` | 已连接设备本地持久化，支持 APP 重启后恢复设备列表 | 蓝牙通信 |
| `memoket_ble` SDK | BLE 底层封装：扫描、连接、GATT 通信、命令序列化/反序列化 | 业务逻辑 |
| `flutter_blue_plus` | Flutter 蓝牙插件：系统蓝牙适配器状态、GATT Service/Characteristic 操作 | 业务逻辑 |

## 4. Scenarios

### S1: 添加新设备 - 正常路径
- **Trigger**: 用户进入 "添加设备" 页面
- **Steps**:
  1. `AddDeviceController.onReady()` 调用 `checkBluetoothStatus()` 检查蓝牙权限和适配器状态
  2. 权限已授予且蓝牙已开启 -> 调用 `scanDevices()`
  3. `DeviceBluetoothManager.scanDevices()` 启动 BLE 扫描（timeout: 30s）
  4. 通过 `bleManager.devicesStream` 监听扫描结果
  5. 过滤设备名前缀 "Memoket"，转换为 `BluetoothDevice` 模型
  6. 自动选中第一个发现的设备
  7. 用户点击 "连接" -> `AddDeviceController.connectDevice()`
  8. 检查设备绑定状态：`BleDeviceApi.checkDeviceBoundStatus(sn)`
  9. 状态=未绑定(0) 或 已绑定当前用户(1) -> 继续
  10. 停止扫描 -> `DeviceBluetoothManager.connectDevice(device)`
  11. 检查蓝牙权限 + 适配器状态
  12. 获取或扫描原始 `BleDevice` -> `bleManager.connect(originalDevice)`
  13. 连接成功 -> 保存连接对象 -> 设置连接状态监听 -> 更新状态为 `connected`
  14. 添加到已连接设备列表 -> 持久化到 `DeviceStorageService`
  15. 发现 GATT 服务 -> 设置录音状态/电量全局监听
  16. 主动拉取一次电量
  17. 设备端绑定（模拟 3s）-> 服务端绑定 `BleDeviceApi.bindDevice()`
  18. 绑定成功 -> 用户点击 "Get Started" 关闭页面
- **Expected**: 设备已配对、已绑定、已持久化，在 "我的设备" 列表中显示

### S2: 添加设备 - 蓝牙权限未授予
- **Trigger**: 用户蓝牙权限未授权
- **Steps**:
  1. `checkBluetoothStatus()` 检测权限未授予
  2. 显示权限引导页，解释为何需要蓝牙权限
  3. 用户点击 "去设置" -> 跳转系统设置
  4. 返回 APP -> 重新检查权限
- **Expected**: 获取权限后自动开始扫描

### S3: 添加设备 - 扫描超时未找到设备
- **Trigger**: 30 秒扫描超时，未发现任何 Memoket 设备
- **Steps**:
  1. `scanDevices()` 的 `onScanComplete(false)` 触发
  2. `_checkScanResult()` 发送 `AddDeviceUiEvent.deviceNotFound`
  3. UI 显示 "未找到设备" 弹窗，提供重试和查看帮助选项
- **Expected**: 用户可重试扫描或查看帮助说明

### S4: 添加设备 - 设备已被其他用户绑定
- **Trigger**: `checkDeviceBoundStatus` 返回状态=2（已绑定其他用户）
- **Steps**:
  1. 检测到绑定状态为 `_boundStatusOtherUser(2)`
  2. 显示引导信息：长按硬件重置/解绑
  3. 用户重置设备后重试
- **Expected**: 用户了解需要先解绑设备

### S5: 连接失败与重试
- **Trigger**: `bleManager.connect()` 返回 null 或抛异常
- **Steps**:
  1. 连接失败 -> 更新设备状态为 `disconnected`
  2. 非静默模式下 Toast 提示 "连接失败，请检查蓝牙"
  3. 触发 `AddDeviceUiEvent.connectionFailed`
- **Expected**: 用户可重新尝试连接

### S6: 设备断连与自动重连
- **Trigger**: 已连接设备意外断开（距离过远、设备关机等）
- **Steps**:
  1. 连接状态监听回调检测到 `disconnected`
  2. 更新设备连接状态为 `disconnected`
  3. 如果设备在录音中：进入断连宽限期（1 分钟灰态）
  4. 宽限期内设备重新可发现 -> 自动重连
  5. 重连成功 -> 恢复连接状态 -> 重新设置监听
  6. 超过宽限期 -> 进入 Unknown 状态
- **Expected**: 短暂断连自动恢复；长时间断连给用户明确提示

### S7: OTA 固件升级 - 正常路径
- **Trigger**: 用户在设备详情页触发固件升级
- **Steps**:
  1. `BleDeviceApi.checkFirmwareVersion()` 检查是否有新版本
  2. 有新版本 -> 下载固件文件到本地
  3. `OtaCommands.otaUpgrade()` 开始升级
  4. Step 1: 加载固件文件 -> Step 2: 计算 MD5
  5. Step 3: 发送 OTA start 指令（含 MD5），设备确认 ready
  6. Step 4: 获取 MTU (511)，分片固件文件
  7. Step 5: 逐片发送数据（进度 15%~95%）
  8. Step 6: 发送 OTA end 指令，设备校验 MD5
  9. Step 7: 发送重启指令（设备会断开连接）
  10. Step 8: 等待设备重启并重新连接 -> 校验新版本号
  11. 进度回调 -> 100%
- **Expected**: 设备固件已更新到新版本

### S8: OTA 固件升级 - 传输失败
- **Trigger**: OTA 过程中数据传输中断或 MD5 校验失败
- **Steps**:
  1. `OtaEndResult.success != true` -> 日志 error
  2. OTA 流程异常终止
  3. UI 显示升级失败提示
- **Expected**: 设备回滚到旧固件，用户可重试

### S9: 设备解绑
- **Trigger**: 用户在设备详情页点击 "解绑"
- **Steps**:
  1. 二次确认弹窗
  2. `BleDeviceApi.unbindDevice(id)` 调用后端解绑
  3. 断开蓝牙连接
  4. 从本地存储移除设备
  5. 刷新设备列表
- **Expected**: 设备从列表消失，蓝牙连接已断开

### S10: 已连接设备恢复（APP 重启后）
- **Trigger**: APP 冷启动
- **Steps**:
  1. `DeviceStorageService` 加载本地持久化的设备信息
  2. 恢复设备列表（状态为 `disconnected`）
  3. 尝试自动扫描并重连历史设备
  4. 重连成功 -> 更新状态为 `connected`
- **Expected**: 历史设备在列表中显示，已连接设备自动恢复连接

## 5. Functional Requirements

| ID | 描述 | 级别 | 验证方法 |
|----|------|------|----------|
| HW-FR-001 | 系统 MUST 支持 BLE 扫描，超时时间 30s，过滤设备名前缀 "Memoket" | MUST | 开启 Memoket 设备 -> 打开扫描 -> 验证 30s 内发现设备 |
| HW-FR-002 | 扫描结果 MUST 显示设备名称和信号强度（RSSI），并排重（按设备 ID 去重） | MUST | 多次扫描到同一设备 -> 验证列表中仅显示一次 |
| HW-FR-003 | 系统 MUST 在发起连接前检查蓝牙权限（`PermissionUtils.checkPermission`）和适配器状态（`FlutterBluePlus.adapterState`） | MUST | 关闭蓝牙 -> 尝试连接 -> 验证提示开启蓝牙 |
| HW-FR-004 | 连接流程 MUST 在连接前停止扫描，避免扫描与连接冲突 | MUST | 正在扫描时发起连接 -> 验证扫描已停止 |
| HW-FR-005 | 系统 MUST 在扫描结束未找到设备时发送 `AddDeviceUiEvent.deviceNotFound` 事件 | MUST | 无设备可发现 -> 验证弹出未找到设备提示 |
| HW-FR-006 | 连接前 MUST 检查设备绑定状态（`checkDeviceBoundStatus`），状态=2（已绑定其他用户）时 MUST 阻止连接并提示 | MUST | 用已绑定设备尝试连接 -> 验证提示已被其他用户绑定 |
| HW-FR-007 | 连接成功后系统 MUST 持久化设备信息到 `DeviceStorageService` | MUST | 连接设备 -> 关闭 APP -> 重启 -> 验证设备在列表中 |
| HW-FR-008 | 连接成功后系统 MUST 发现 GATT 服务并设置录音状态和电量通知监听 | MUST | 连接设备 -> 验证电量信息已获取 |
| HW-FR-009 | 系统 MUST 在连接成功后调用 `BleDeviceApi.bindDevice()` 完成服务端绑定 | MUST | 连接新设备 -> 验证服务端绑定接口已调用 |
| HW-FR-010 | 连接状态 MUST 实时监听并通过 `ChangeNotifier` 通知所有监听者 | MUST | 断开设备蓝牙 -> 验证 APP 状态同步更新为 disconnected |
| HW-FR-011 🚧 | 系统 SHOULD 在非手动断连场景下实现自动重连（息屏、APP 退出、距离过远、意外断连） | SHOULD | 设备短暂离开范围 -> 返回 -> 验证自动重连 |
| HW-FR-012 | 系统 MUST 支持设备录音断连宽限期：断连 1 分钟内显示灰态，超过后进入 Unknown 状态 | MUST | 录音中断开设备 -> 1 分钟内验证灰态; > 1 分钟验证 Unknown |
| HW-FR-013 | OTA 升级流程 MUST 按顺序执行：加载文件 -> MD5 -> OTA start -> 分片传输(MTU=511) -> OTA end -> 重启 -> 重连校验 | MUST | 触发 OTA -> 验证固件版本已更新 |
| HW-FR-014 | OTA 进度 MUST 通过回调实时汇报：0% -> 10%(MD5) -> 11%(start) -> 15%(分片开始) -> 95%(传输完成) -> 100%(校验通过) | MUST | OTA 过程中验证进度条递增 |
| HW-FR-015 | 系统 MUST 支持获取设备电量（`DeviceCommands.getBattery`），返回 `BatteryInfo` | MUST | 连接设备 -> 验证电量信息展示 |
| HW-FR-016 | 系统 MUST 支持获取设备固件版本（`DeviceCommands.getVersion`），返回 `VersionInfoResult` | MUST | 连接设备 -> 验证版本号展示 |
| HW-FR-017 | 系统 MUST 支持同步 RTC 时间（`DeviceCommands.syncRtcTime`），将 APP 当前时间下发到设备 | MUST | 连接设备后验证 RTC 同步日志 |
| HW-FR-018 | 系统 MUST 支持设备解绑：调用 `BleDeviceApi.unbindDevice` + 断开蓝牙 + 清除本地存储 | MUST | 解绑设备 -> 验证从列表消失 + 后端已删除 |
| HW-FR-019 | 系统 MUST 支持设备固件版本检查：`BleDeviceApi.checkFirmwareVersion(sn, model, firmware_version, module)` | MUST | 检查固件 -> 验证返回是否有更新 |
| HW-FR-020 | 系统 MUST 支持设备状态上报：`BleDeviceApi.reportDeviceStatus(id, firmware_version, online)` | MUST | 连接/断开时验证状态上报请求 |
| HW-FR-021 | 电量显示 MUST 使用三级制（低、中、高），不显示百分比 | MUST | 验证 UI 显示低/中/高图标而非百分比 |
| HW-FR-022 | 设备列表 MUST 显示设备连接状态图标：连接成功+电量、连接中、连接断开 三种状态 | MUST | 验证三种状态对应不同 icon |
| HW-FR-023 | 连接过程中设备未在本次会话扫描过时，系统 MUST 自动扫描该设备 ID（`_scanForDeviceById`），找到后再连接 | MUST | 从存储恢复设备 -> 点击连接 -> 验证先扫描后连接 |
| HW-FR-024 🚧 | 系统 SHOULD 支持 WiFi 配置功能：WiFi 网络设置（APP-156）和热点模式（APP-157） | SHOULD | V1.2 待实现 |
| HW-FR-025 🚧 | 系统 SHOULD 支持设备快速切换（APP-169），在主页提供快捷入口 | SHOULD | 多设备场景下快速切换 |

## 6. State Model

### 蓝牙连接状态机

```stateDiagram-v2
    [*] --> Disconnected

    Disconnected --> Connecting : connectDevice() 调用
    Connecting --> Connected : bleManager.connect() 成功
    Connecting --> ConnectedFailed : 连接失败/超时
    Connecting --> Disconnected : 蓝牙权限未授予/适配器未开启

    Connected --> Disconnected : 设备断开/手动断开
    ConnectedFailed --> Disconnected : 状态重置

    Disconnected --> Connecting : 自动重连触发
```

### 设备录音 UI 状态机

```stateDiagram-v2
    [*] --> NotRecording

    NotRecording --> ConnectedRecording : BLE 通知 recording_started
    ConnectedRecording --> NotRecording : BLE 通知 recording_stopped
    ConnectedRecording --> DisconnectedGrace : 设备断连 (< 1min)
    DisconnectedGrace --> ConnectedRecording : 重连成功且仍在录音
    DisconnectedGrace --> NotRecording : 重连成功但未在录音
    DisconnectedGrace --> StatusUnknown : 断连超过 1 分钟
    StatusUnknown --> ConnectedRecording : 重连成功且在录音
    StatusUnknown --> NotRecording : 重连成功但未在录音 / 用户关闭
```

### OTA 升级状态机

```stateDiagram-v2
    [*] --> Idle
    Idle --> CheckingVersion : 用户触发固件检查
    CheckingVersion --> DownloadingFirmware : 有新版本
    CheckingVersion --> Idle : 已是最新

    DownloadingFirmware --> OtaStarting : 下载完成
    DownloadingFirmware --> Failed : 下载失败

    OtaStarting --> Transmitting : 设备 ready 确认
    OtaStarting --> Failed : 设备拒绝

    Transmitting --> Verifying : 所有分片发送完成
    Transmitting --> Failed : 传输中断

    Verifying --> Rebooting : MD5 校验通过
    Verifying --> Failed : 校验失败

    Rebooting --> VersionCheck : 设备重连成功
    VersionCheck --> Completed : 版本号正确
    VersionCheck --> Failed : 版本号不匹配

    Completed --> Idle : 重置
    Failed --> Idle : 重试/取消
```

### 状态定义表

**BluetoothConnectionStatus**

| 状态 | 含义 | 进入条件 | 退出条件 | 代码枚举 |
|------|------|----------|----------|----------|
| disconnected | 未连接 | 初始状态 / 连接断开 / 连接失败重置 | 发起连接 | `BluetoothConnectionStatus.disconnected` |
| connecting | 连接中 | `connectDevice()` 调用 | 连接成功/失败 | `BluetoothConnectionStatus.connecting` |
| connected | 已连接 | `bleManager.connect()` 成功 | 设备断开 | `BluetoothConnectionStatus.connected` |
| connectedFailed | 连接失败 | `bleManager.connect()` 返回 null/异常 | 状态重置 | `BluetoothConnectionStatus.connectedFailed` |

**DeviceRecordingUiState**

| 状态 | 含义 | 进入条件 | 退出条件 | 代码枚举 |
|------|------|----------|----------|----------|
| notRecording | 非录音状态 | 初始/录音停止/重连后未在录音 | BLE 录音通知 | `DeviceRecordingUiState.notRecording` |
| connectedRecording | 已连接且录音中 | BLE 录音状态通知 | 录音停止/断连 | `DeviceRecordingUiState.connectedRecording` |
| disconnectedGrace | 断连宽限期（灰态） | 录音中设备断连 | 重连成功/超时 | `DeviceRecordingUiState.disconnectedGrace` |
| statusUnknown | 断连超时（Unknown） | 断连超过 1 分钟 | 重连成功/用户关闭 | `DeviceRecordingUiState.statusUnknown` |

**DeviceManagerChangeType**

| 状态 | 含义 | 进入条件 | 退出条件 | 代码枚举 |
|------|------|----------|----------|----------|
| uiState | 纯 UI 状态变化 | isSearching/isConnecting/selectedDevice 变化 | - | `DeviceManagerChangeType.uiState` |
| scanResults | 扫描结果列表变化 | 扫描发现新设备 | - | `DeviceManagerChangeType.scanResults` |
| connection | 单设备连接状态变化 | connected/disconnected | - | `DeviceManagerChangeType.connection` |
| deviceList | 设备列表整体变化 | 设备新增/移除/存储恢复 | - | `DeviceManagerChangeType.deviceList` |
| adapterState | 蓝牙适配器状态变化 | 系统蓝牙开关 | - | `DeviceManagerChangeType.adapterState` |
| recordingState | 录音状态变化 | 开始/暂停/恢复/停止 | - | `DeviceManagerChangeType.recordingState` |
| battery | 电量信息变化 | 电量通知 | - | `DeviceManagerChangeType.battery` |

### 非法状态转移

| 不允许 | 原因 | 防御 |
|--------|------|------|
| Disconnected -> Connected（跳过 Connecting） | 必须经过连接过程 | `connectDevice()` 先设置 connecting 再执行连接 |
| Connecting -> Connecting（重复连接） | 可能导致多次订阅 | `if (state.value.isConnecting) return` 防护 |
| NotRecording -> StatusUnknown（跳过宽限期） | 只有在录音中断连才进入宽限期 | 断连时检查 `_recordingSessions` |
| OTA Transmitting -> Idle（跳过校验） | 必须完成 end + 重启 + 版本校验 | OTA 流程为顺序执行 |

## 7. Data Contract

### API

| 方法 | 路径 | 请求体 | 响应体 | 错误码 |
|------|------|--------|--------|--------|
| GET | `/api/v1/firmwares/check` | Query: `{ sn, model, firmware_version, module }` | `FirmwareVersionResponse` | 404: 设备未找到 |
| POST | `/api/v1/devices/bind` | `{ name, sn, model, firmware_version, mac }` | `BoundDeviceResponse` | 409: 设备已被绑定 |
| POST | `/api/v1/devices/{id}/unbind` | - | void | 404: 设备未找到 |
| GET | `/api/v1/devices` | - | `{ list: [BoundDeviceResponse] }` | - |
| GET | `/api/v1/devices/{id}` | - | `BoundDeviceResponse` | 404: 设备未找到 |
| POST | `/api/v1/devices/{id}/report` | `{ firmware_version?, online? }` | `ReportDeviceStatusResponse` | - |
| POST | `/api/v1/devices/check-bind` | `{ sn }` | `CheckDeviceBoundStatusResponse` | - |

### 数据模型

**BluetoothDevice (APP 层)**

| 字段 | 类型 | 必填 | 单位 | 示例 |
|------|------|------|------|------|
| id | String | Y | MAC 地址 | `"AA:BB:CC:DD:EE:FF"` |
| serverId | String? | N | 服务端 ID | `"dev_abc123"` |
| name | String | Y | - | `"Memoket-X1"` |
| rssi | int | Y | dBm | `-65` |
| discoveredAt | DateTime | Y | ISO 8601 | `"2026-04-02T10:30:00Z"` |
| connectionStatus | BluetoothConnectionStatus | Y | 枚举 | `connected` |
| firmwareVersion | String? | N | semver | `"1.2.3"` |
| batteryLevel | int? | N | 0-100 | `75` |
| batteryStatus | BatteryStatus? | N | 枚举 | `notCharging` |
| connectable | bool | Y | - | `true` |
| imagePath | String? | N | asset path | `"assets/images/device.png"` |
| manufacturerData | Map<int, List<int>>? | N | - | BLE 广播数据 |
| serviceData | Map<String, List<int>>? | N | - | BLE 服务数据 |
| serviceUuids | List<String>? | N | UUID | BLE 服务 UUID 列表 |
| txPowerLevel | int? | N | dBm | 发射功率 |

**BatteryStatus 枚举**

| 值 | 含义 | 数值 |
|----|------|------|
| unknown | 未知 | 0 |
| charging | 充电中 | 1 |
| notCharging | 未充电 | 2 |
| full | 已充满 | 3 |

**FirmwareModule 枚举**

| 值 | 含义 | 数值 |
|----|------|------|
| all | 全部模块 | 0 |
| bluetooth | 蓝牙模块 | 1 |
| wifi | WiFi 模块 | 2 |

**DeviceModel 枚举**

| 值 | 名称 | 数值 |
|----|------|------|
| m01m1 | M01M1 | 1 |
| m01m2 | M01M2 | 2 |

**AddDeviceUiEvent 枚举**

| 值 | 含义 |
|----|------|
| deviceNotFound | 扫描结束未找到设备 |
| connectionFailed | 设备连接失败 |
| bindFailed | 服务端绑定失败 |
| closePage | 页面可关闭 |

### 协议分层

```
+-------------------------------------------------------------+
| APP 业务层                                                   |
| AddDeviceController / MyDeviceController / DeviceDetail...   |
+-------------------------------------------------------------+
| APP 服务层                                                   |
| DeviceService / DeviceCommands / OtaCommands                |
+-------------------------------------------------------------+
| APP 蓝牙管理层                                               |
| DeviceBluetoothManager (单例 ChangeNotifier)                |
| - 扫描管理 (scanDevices / stopScan / _scanForDeviceById)    |
| - 连接管理 (connectDevice / disconnectDevice)               |
| - 状态监听 (connection / recording / battery subscriptions) |
| - 设备列表 (foundDevices / connectedDevices)                |
+-------------------------------------------------------------+
| memoket_ble SDK                                              |
| BleManager / BleConnection / BleCommandExecutor             |
| - 扫描: startScan() / devicesStream                         |
| - 连接: connect(BleDevice) -> BleConnection                 |
| - 命令: connection.execute<T>(request) -> BleResponse<T>    |
| - 请求类型: GetBatteryRequest, VersionQueryRequest,         |
|   SyncTimeRequest, OtaStartRequest, OtaEndRequest, etc.     |
+-------------------------------------------------------------+
| flutter_blue_plus                                            |
| - 系统蓝牙适配器状态 (BluetoothAdapterState)                |
| - GATT Service/Characteristic 发现与操作                     |
| - 特征值通知订阅 (setNotifyValue / onValueReceived)          |
+-------------------------------------------------------------+
| BLE 物理层                                                   |
| SERVICE_UUID -> CHAR_RESPONSE (响应特征值)                   |
| 通知类型: 录音状态 / 电量变化 / 文件传输                     |
+-------------------------------------------------------------+
```

**BLE 命令协议摘要**

| 命令 | 请求类 | 响应类 | 方向 | 说明 |
|------|--------|--------|------|------|
| 获取电量 | `GetBatteryRequest` | `BatteryInfo` | APP -> 设备 | 返回电量值+充电状态 |
| 获取版本 | `VersionQueryRequest` | `VersionInfoResult` | APP -> 设备 | 返回固件版本号 |
| 同步 RTC | `SyncTimeRequest(DateTime)` | void | APP -> 设备 | 将 APP 时间下发 |
| OTA 启动 | `OtaStartRequest(module, md5)` | `OtaStartReadyResult` | APP -> 设备 | 设备确认 ready |
| OTA 数据 | 分片 bytes | - | APP -> 设备 | MTU=511, 逐片发送 |
| OTA 结束 | `OtaEndRequest` | `OtaEndResult` | APP -> 设备 | MD5 校验 |
| 设备重启 | reboot command | - | APP -> 设备 | OTA 完成后重启 |
| 录音状态通知 | - | `RecordingStateNotification` | 设备 -> APP | GATT Notify |
| 电量通知 | - | `BatteryInfo` | 设备 -> APP | GATT Notify |

## 8. Error Handling

| Case | 触发条件 | 系统行为 | 状态变化 | 用户感知 |
|------|----------|----------|----------|----------|
| 蓝牙权限未授予 | `PermissionUtils.checkPermission(bluetooth)` 返回 false | 日志 warn + Toast "需要蓝牙权限" + 更新状态为 disconnected | Connecting -> Disconnected | Toast: 请授予蓝牙权限 |
| 系统蓝牙未开启 | `FlutterBluePlus.adapterState != on` | 日志 warn + Toast "请开启系统蓝牙" + 更新状态为 disconnected | Connecting -> Disconnected | Toast: 请开启蓝牙 |
| 扫描启动失败 | `bleManager.startScan()` 异常 | 日志 error + 设置 isSearching=false + 回调 onScanComplete(false) | isSearching -> false | 可能无设备显示 |
| 扫描超时无设备 | 30s 扫描结束 foundDevices 为空 | 发送 `deviceNotFound` UI 事件 | isSearching -> false | "未找到设备" 弹窗 |
| 连接目标设备未扫描过 | `_originalDevicesMap[id]` 为 null | 自动调用 `_scanForDeviceById` 补扫 30s | 保持 Connecting | 用户等待（有 loading） |
| 补扫未找到设备 | `_scanForDeviceById` 返回 null | 更新为 disconnected + 回调 onFailure | Connecting -> Disconnected | "未找到设备，请确保设备已开启" |
| BLE 连接失败 | `bleManager.connect()` 返回 null | 更新为 disconnected + Toast（非静默模式） | Connecting -> Disconnected | Toast: 连接失败 |
| BLE 连接异常 | `bleManager.connect()` 抛异常 | catch -> 更新为 disconnected + onFailure 回调 | Connecting -> Disconnected | Toast: 连接失败 |
| 设备已被他人绑定 | checkDeviceBoundStatus 返回 status=2 | 阻止连接 + 提示重置设备 | 保持 Disconnected | "设备已被其他用户绑定" |
| 服务端绑定失败 | `BleDeviceApi.bindDevice` 返回 isSuccess=false | 发送 `bindFailed` UI 事件 | 已连接但未绑定 | "绑定失败" 提示 |
| OTA 文件不存在 | `File(firmwareFilePath).exists()` 为 false | 日志 error + 返回 null，OTA 中止 | Idle | "固件文件不存在" |
| OTA start 被拒 | startResponse.data?.ready != true | 日志 error + OTA 中止 | Failed | "设备拒绝升级" |
| OTA MD5 校验失败 | endResponse.data?.success != true | 日志 error + OTA 中止 | Failed | "固件校验失败" |
| OTA 传输中断 | 分片传输异常 | catch -> rethrow + 日志 error | Failed | "升级失败" |
| 设备断连时录音中 | 连接状态监听检测到断开 + 有活跃录音 session | 进入断连宽限期（1 min）-> 尝试重连 | connectedRecording -> disconnectedGrace | 灰态录音指示器 |
| 幽灵录音状态 | 设备断连后录音 session 未清理（Bug Ticket-000719） | 应检查 `RecordingStateManager` 持久化状态并清理孤立 session | 应回到 notRecording | 录音指示器无法删除(已知 bug) |
| 获取电量失败 | `DeviceCommands.getBattery` 失败 | 日志 error，不中断流程 | 无变化 | 电量显示为空/旧值 |
| 获取版本失败 | `DeviceCommands.getVersion` 失败 | 日志 error，不中断流程 | 无变化 | 版本号显示为空/旧值 |
| 正在扫描中重复扫描 | `scanAgain()` 在 isSearching=true 时调用 | 日志 "上一个扫描结束之前不能进行新的扫描" + return | 无变化 | 无感知 |

**已知 Bug 参考**:
- **Ticket-000719**: 幽灵录音状态 -- BLE 设备断连后录音 session 未清理，导致录音指示器持续显示。根因：`RecordingStateManager` 的 Hive 持久化状态在设备断连时未被正确清除。
- **Ticket-000563/000645**: 设备空状态 Widget 被误渲染到录音列表/搜索页面。根因：条件渲染逻辑错误，`DeviceEmptyStateWidget` 出现在非设备管理页面。

## 9. Non-functional Requirements

| 指标 | 要求 | 实测值 | 来源 |
|------|------|--------|------|
| BLE 扫描超时 | 30s | 30s（硬编码） | `device_bluetooth_manager.dart:231` |
| BLE 连接超时 | SDK 默认（通常 10-15s） | SDK 控制 | `memoket_ble` SDK |
| 扫描结果去重 | 按设备 ID 实时去重 | 已实现 | `_handleScanResults` indexWhere 检查 |
| OTA 分片大小 | MTU=511 bytes | 511（硬编码） | `ota_commands.dart:172` |
| OTA 传输可靠性 | 失败后可重试，设备保持旧固件 | 已实现 | OTA 异常不影响设备正常运行 |
| 设备持久化延迟 | < 100ms | 待测 | `DeviceStorageService` |
| 录音断连宽限期 | 1 分钟 | 1 min（硬编码） | `_recordingDisconnectGrace = Duration(minutes: 1)` |
| 并发扫描防护 | 单次仅允许一个扫描任务 | 已实现 | `if (isSearching) return` 防护 |
| 并发连接防护 | 单次仅允许一个连接任务 | 已实现 | `if (isConnecting) return` 防护 |
| 设备列表上限 | 未限制（多设备支持） | N/A | 支持绑定多个设备 |
| 电量刷新频率 | 连接成功后立即拉取一次 + GATT 通知驱动 | 已实现 | `_fetchBatteryAfterConnected` + 订阅 |

## 10. Observability

### Logs

| 事件 | 级别 | 携带字段 | 组件 |
|------|------|----------|------|
| 发现新设备 | INFO | name, id, foundDevices.length | `DeviceBluetoothManager` |
| 启动扫描失败 | ERROR | error | `DeviceBluetoothManager` |
| 已停止扫描 | INFO | - | `DeviceBluetoothManager` |
| 停止扫描失败 | ERROR | error | `DeviceBluetoothManager` |
| 上一个扫描未结束 | INFO | - | `DeviceBluetoothManager` |
| 设备连接成功 | INFO | name | `DeviceBluetoothManager` |
| 设备连接失败 | WARN | name | `DeviceBluetoothManager` |
| 连接设备出错 | ERROR | error | `DeviceBluetoothManager` |
| 蓝牙权限未授予 | WARN | - | `DeviceBluetoothManager` |
| 系统蓝牙未开启 | WARN | adapterState | `DeviceBluetoothManager` |
| 未选中设备 | WARN | - | `DeviceBluetoothManager` |
| 设备未在本次扫描过 | INFO | name, id | `DeviceBluetoothManager` |
| GATT 服务发现 | INFO | deviceId, services.length | `DeviceBluetoothManager` |
| canStartDeviceRecording | INFO | isRecordingFromDevice, hasConnectedDevice | `DeviceBluetoothManager` |
| 获取电量结果 | INFO | deviceId, isSuccess, batteryInfo | `DeviceCommands` |
| 获取电量失败 | ERROR | description | `DeviceCommands` |
| 获取电量出错 | ERROR | deviceId, error | `DeviceCommands` |
| 获取版本结果 | INFO | deviceId, isSuccess, version | `DeviceCommands` |
| 获取版本信息失败 | ERROR | description | `DeviceCommands` |
| 同步 RTC 时间 | INFO | deviceId, time | `DeviceCommands` |
| OTA MD5 计算 | INFO | md5 hex | `OtaCommands` |
| OTA start 请求/响应 | INFO | serialize, isSuccess, ready | `OtaCommands` |
| OTA 分片数量 | INFO | chunks.length | `OtaCommands` |
| OTA 传输进度 | INFO | progress% | `OtaCommands` |
| OTA 校验失败 | ERROR | description | `OtaCommands` |
| OTA 设备重启 | INFO | - | `OtaCommands` |
| OTA 升级成功 | INFO | - | `OtaCommands` |
| OTA 升级出错 | ERROR | deviceId, error | `OtaCommands` |
| 固件文件不存在 | ERROR | firmwareFilePath | `OtaCommands` |
| 发现设备 | INFO | name | `AddDeviceController` |
| 扫描完成 | INFO | hasFoundDevices | `AddDeviceController` |
| 绑定设备结果 | INFO | result, isSuccess | `AddDeviceController` |
| 尝试绑定无选中设备 | WARN | - | `AddDeviceController` |
| 重复点击连接 | WARN | - | `AddDeviceController` |
| 设备信息丰富化 | INFO | name, id, isConnected, hasServerId | `DeviceService` |

### Metrics

| 指标 | 含义 | 告警阈值 |
|------|------|----------|
| ble_scan_success_rate | BLE 扫描发现至少一个设备的成功率 | < 80% |
| ble_connect_success_rate | BLE 连接成功率（连接成功 / 连接尝试） | < 90% |
| ble_connect_latency_p95 | 从点击连接到连接成功的耗时 | > 15s |
| device_bind_success_rate | 服务端设备绑定成功率 | < 95% |
| ota_success_rate | OTA 升级成功率 | < 90% |
| ota_duration_p50 | OTA 升级中位耗时 | > 300s |
| ble_disconnect_rate | 已连接设备意外断连率（每小时） | > 5 次/小时 |
| ghost_recording_incidents | 幽灵录音状态出现次数（Ticket-000719） | > 0 |
| scan_timeout_rate | 扫描超时（30s 内无设备）比率 | > 30% |
| reconnect_success_rate | 自动重连成功率 | < 70% |
