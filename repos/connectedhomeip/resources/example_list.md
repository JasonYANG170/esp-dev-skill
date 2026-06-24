# connectedhomeip — Example Index

> 列出仓库 `examples/` 下真实示例目录（来源：`D:/esp-skill/espressif-repos/connectedhomeip/examples/`）。
> ESP32 支持列标记该示例是否含 `esp32/` 子目录（本仓库中真正可烧录到 ESP32/ESP32-C3 的示例）。

## 顶层示例（按目录）

| 路径 | 说明 | ESP32 支持 |
|---|---|---|
| `examples/all-clusters-app/` | 所有 ZCL 集群原型应用，演示配网与集群控制；含设备类型选择 | ✅ `esp32/`（DevKitC / WROVER-KIT / M5Stack / ESP32C3-DevKitM） |
| `examples/bridge-app/` | 桥接应用（聚合/代理其它设备） | 仅 `linux/` |
| `examples/chip-tool/` | 主机侧 CLI 客户端，配对设备并下发 ZCL 命令 | 主机侧（GN/ninja 构建） |
| `examples/chip-tool-darwin/` | macOS (Darwin) 版 chip-tool | 主机侧 |
| `examples/ipv6only-app/` | 仅 IPv6 传输（不含 BLE/Wi-Fi）的设备应用 | ✅ `esp32/` |
| `examples/lighting-app/` | 照明（灯）应用，多平台 | `efr32/`、`k32w/`、`linux/`、`mbed/`、`nrfconnect/`、`qpg/`、`telink/`（本版本无 esp32 子目录） |
| `examples/lock-app/` | 锁应用，把 OnOff 集群映射到锁/解锁逻辑 | ✅ `esp32/` |
| `examples/minimal-mdns/` | 最小 mDNS 实现（调试/学习） | 主机侧 |
| `examples/persistent-storage/` | 持久化存储示例 | ✅ `esp32/` |
| `examples/pigweed-app/` | Pigweed RPC 调试通道示例 | ✅ `esp32/` |
| `examples/pump-app/` | 泵应用 | 多平台（非 esp32） |
| `examples/pump-controller-app/` | 泵控制器应用 | 多平台（非 esp32） |
| `examples/shell/` | CHIP shell（命令行） | ✅ `esp32/` |
| `examples/temperature-measurement-app/` | 温度测量应用 | ✅ `esp32/` |
| `examples/tv-app/` | 电视（casting target）应用 | 多平台 |
| `examples/window-app/` | 窗帘/窗户应用 | 多平台 |
| `examples/common/` | 跨示例公共代码 | — |
| `examples/platform/` | 各平台适配层（含 `esp32/`、`efr32/`、`linux/`、`nrfconnect/`、`k32w/`、`qpg/`、`telink/`、`mbed/`、`cc13x2_26x2/`） | 平台层 |
| `examples/build_overrides/` | GN 构建覆盖 | — |

## 重点 ESP32 示例文件（最常用）

| 文件 | 作用 |
|---|---|
| `examples/lock-app/esp32/main/main.cpp` | `app_main` 入口，标准初始化顺序 |
| `examples/lock-app/esp32/main/AppTask.cpp` | 应用任务循环、`TryLockChipStack` 用法、`emberAfWriteAttribute` 回写 |
| `examples/lock-app/esp32/main/DeviceCallbacks.cpp` | `PostAttributeChangeCallback` + `DeviceEventCallback`（含 mDNS 重启） |
| `examples/lock-app/esp32/main/include/CHIPDeviceManager.h` | 设备管理单例与回调基类 |
| `examples/lock-app/esp32/main/include/BoltLockManager.h` | 设备逻辑模板（锁动作/状态/回调） |
| `examples/lock-app/esp32/main/Kconfig.projbuild` | `Demo -> Rendezvous Mode` choice 定义 |
| `examples/lock-app/esp32/sdkconfig.defaults` | BT/NimBLE/lwIP/分区表默认 |
| `examples/lock-app/esp32/partitions.csv` | 自定义分区表（factory 1945K） |
| `examples/all-clusters-app/esp32/main/main.cpp` | 全集群入口 + 设备类型 GPIO 映射 + OnOff/DoorLock/Temp/LevelControl server 注册（`SetupPretendDevices` / `SetupInitialLevelControlValues` / `chip::LaunchShell`） |
| `examples/all-clusters-app/esp32/main/Kconfig.projbuild` | `Demo -> Device Type` choice |
| `examples/temperature-measurement-app/esp32/main/main.cpp` | 最小 sensor 示例 `app_main`（标准初始化 + UI 循环） |
| `examples/temperature-measurement-app/esp32/main/temperature-measurement.zap` | TemperatureMeasurement server zap 配置 |
| `examples/temperature-measurement-app/esp32/README.md` | sensor 示例构建 + 配网 + python controller 温度集群读取 |
| `examples/shell/esp32/sdkconfig.defaults` | `CONFIG_ENABLE_CHIP_SHELL=y` |
| `examples/shell/README.md` / `README_DEVICE.md` / `README_OTCLI.md` | CHIP Shell 命令参考（device config/get、base64/echo/ping/version、otcli） |
| `examples/platform/esp32/shell_extension/launch.h` / `launch.cpp` | `chip::LaunchShell()` 实现（`chip_cli` 任务） |
| `examples/pigweed-app/esp32/README.md` | Pigweed Echo RPC 构建 + `pw_hdlc.rpc_console` 用法 |
| `examples/pigweed-app/esp32/main/main.cpp` | `EchoService` 注册与 `chip::rpc::Start` 调用 |
| `examples/pigweed-app/esp32/main/Kconfig.projbuild` | `PW RPC Example Configuration`（UART 端口/引脚/波特） |
| `examples/common/pigweed/RpcService.h` | `chip::rpc::Start` 与 `Mutex` 接口 |
| `examples/chip-tool/README.md` | chip-tool 构建与 `pairing`/集群命令用法（含 `doorlock` / `levelcontrol` / `temperaturemeasurement` 命令集） |
| `src/app/clusters/temperature-measurement-server/temperature-measurement-server.h` | `SetMeasuredValueCallback` 等 setter/getter 签名 |
| `src/app/clusters/door-lock-server/door-lock-server.h` | `ActivateDoorLockCallback`、`ApplyPin/ApplyRfid`、`DOOR_LOCK_SERVER_ENDPOINT`、表容量宏 |
| `src/app/clusters/level-control/level-control.h` | `PostInitCallback`、`EMBER_AF_PLUGIN_LEVEL_CONTROL_TICKS_PER_SECOND` |

## 选择建议

| 目标 | 推荐起点 |
|---|---|
| 通用集群验证 / 设备类型选择 | `examples/all-clusters-app/esp32/` |
| OnOff → 物理动作（锁/继电器）模板 | `examples/lock-app/esp32/` |
| 温度上报（TemperatureMeasurement server） | `examples/temperature-measurement-app/esp32/` |
| 门锁集群（DoorLock server：LockState/PIN/RFID） | `examples/all-clusters-app/esp32/`（仓库无独立 lock cluster 示例） |
| 调光（Level Control server） | `examples/all-clusters-app/esp32/`（仓库无 lighting-app/esp32） |
| 持久化存储 | `examples/persistent-storage/esp32/` |
| CHIP shell 调试（验证凭据 / 功能测试） | `examples/shell/esp32/` |
| Pigweed RPC 调试通道（主机调用设备函数） | `examples/pigweed-app/esp32/` |
| 仅 IPv6 传输 | `examples/ipv6only-app/esp32/` |
| 主机侧控制设备 | `examples/chip-tool/` 或 `src/controller/python/` |
