# Changelog

本文件记录 connectedhomeip-skill 的变更。

格式基于 [Keep a Changelog](https://keepachangelog.com/)。

## [1.1.0] — 2026-06-18

### Added
- 5 个新 recipe 填补审计确认的 cluster / 调试通道缺口：
  - `recipes/temperature_measurement_cluster.md` — TemperatureMeasurement 集群 sensor 上报模式（`emberAfTemperatureMeasurementClusterSetMeasuredValueCallback`）。来源：`examples/temperature-measurement-app/esp32/`、`src/app/clusters/temperature-measurement-server/temperature-measurement-server.h`、`examples/all-clusters-app/esp32/main/main.cpp`。
  - `recipes/door_lock_cluster.md` — DoorLock 集群服务端集成（LockState 属性、`ActivateDoorLockCallback`、PIN/RFID `ApplyPin/ApplyRfid`、用户表与日志）。来源：`src/app/clusters/door-lock-server/door-lock-server.h`、`examples/all-clusters-app/esp32/main/main.cpp`、`examples/chip-tool/README.md`。
  - `recipes/level_control_cluster.md` — Level Control 集群调光（`SetupInitialLevelControlValues`、`emberAfPluginLevelControlClusterServerPostInitCallback` 硬件同步、move/move-to-level/step/stop 命令）。来源：`src/app/clusters/level-control/level-control.h`、`examples/all-clusters-app/esp32/main/main.cpp`、`examples/chip-tool/README.md`。
  - `recipes/chip_shell_debug.md` — 设备端 CHIP Shell CLI bring-up（`chip::LaunchShell()`、`device config/get` 验证凭据、otcli）。来源：`examples/shell/esp32/`、`examples/shell/README.md` / `README_DEVICE.md` / `README_OTCLI.md`、`examples/platform/esp32/shell_extension/launch.{h,cpp}`。
  - `recipes/pigweed_rpc_debug.md` — Pigweed Echo RPC 远程调试通道（UART `pw_hdlc.rpc_console`、`rpcs.pw.rpc.EchoService.Echo`）。来源：`examples/pigweed-app/esp32/`、`examples/pigweed-app/esp32/README.md`、`examples/common/pigweed/RpcService.h`。
- `SKILL.md`：新增 "Debugging & Bring-up Tools" 分组；Clusters 表追加 3 条；新增 Core Principle #13（sensor 上报走 setter，不走 PostAttributeChangeCallback）与 #14（每个集群有独立 server 插件与命令路径，勿全走 OnOff 分支）。
- `resources/api_reference.md`：新增 TemperatureMeasurement / DoorLock / Level Control server 头文件签名，CHIP Shell `LaunchShell`，Pigweed `chip::rpc::Start`；扩展 ZCL 宏表（Level/DoorLock/Temp cluster & attribute ID、`emberAfWriteServerAttribute`/`emberAfReadAttribute`、`EMBER_ZCL_DOOR_LOCK_STATE_*`）。
- `resources/example_list.md`：重点文件表追加 cluster server 头文件、temperature-measurement-app / shell / pigweed 关键文件；选择建议表追加 DoorLock / Level Control 推荐起点。

### Grounding
- 所有新增 API 签名、宏、Kconfig、文件路径均取自 `src/app/clusters/`、`examples/{temperature-measurement-app,all-clusters-app,shell,pigweed-app}/esp32/`、`examples/platform/esp32/shell_extension/`、`examples/common/pigweed/`、`src/app/common/gen/`。
- 未发现的内容一律省略（如本仓库无 `lighting-app/esp32`，Level Control recipe 明确说明 all-clusters-app 是唯一参考）。

## [1.0.0] — 2026-06-18

### Added
- 初始版本：面向 Espressif `connectedhomeip`（Matter/CHIP 参考实现分叉）的 AI Skill。
- `SKILL.md`：核心原则（12 条）、When to Use、recipes 索引、芯片/设备类型支持表、关键配置表、配网状态机、关键回调签名、15 条 Critical Pitfalls（含错误/正确代码对比）、执行流程表、失败策略表。
- `AGENTS.md`：项目上下文、文件命名、include 模式、ESP32 示例标准结构、`app_main` 范式、构建工作流（设备固件 + chip-tool + python controller）、代码生成 checklist、Do Not Modify 清单。
- `recipes/`：9 个场景驱动 recipe —— 构建 ESP32 示例、创建自定义示例、BLE 配网、Wi-Fi/Bypass 配网、python controller、chip-tool CLI、OnOff 绑 GPIO、自定义属性回调、设备事件处理。
- `resources/api_reference.md`：`CHIPDeviceManager`、`InitServer`、`PlatformMgr/ConnectivityMgr/ConfigurationMgr`、`Mdns`、OnboardingCodes、Ember ZCL、`BoltLockManager`、`AppTask/AppEvent`、ESP-IDF 入口、日志辅助。
- `resources/config_reference.md`：Rendezvous 模式、Echo Client、PW RPC、设备类型、sdkconfig.defaults、CHIP Device Layer、构建开关、分区表、ESP-IDF 版本。
- `resources/pitfalls.md`：15 条陷阱汇总（含代码对比）。
- `resources/example_list.md`：`examples/` 全部顶层示例与 ESP32 支持标注、重点 ESP32 文件索引、选择建议。
- `README.md` / `CHANGELOG.md`。

### Grounding
- 所有 API 签名、Kconfig 符号、sdkconfig 项、文件路径、代码片段均取自仓库 `examples/`（lock-app / all-clusters-app / chip-tool）与 `src/`。
- 未发现的内容一律省略，未臆造任何 API 或示例。
