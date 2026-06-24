---
name: connectedhomeip-skill
description: >-
  AI Skill for Matter / CHIP (Connected Home over IP) firmware development on the
  Espressif fork of project-chip/connectedhomeip. Used when users need to build,
  commission, or customize Matter device examples on ESP32/ESP32-C3, implement
  Zigbee Cluster Library (ZCL) attributes/callbacks, drive commissioning over
  BLE/Wi-Fi/Bypass, and control devices with chip-tool or the python controller.
  Trigger words: "Matter", "CHIP", "connectedhomeip", "project-chip", "chip-tool",
  "CHIPDeviceManager", "ZCL", "cluster", "commissioning", "Rendezvous",
 "OnOff", "Matter", "智能家居", "连接家庭", "Matter协议", "CHIP协议", "集群", "配网"
tags:
  - embedded
  - matter
  - chip
  - connectedhomeip
  - esp32
  - esp32-c3
  - zigbee-cluster
  - commissioning
  - ble
  - wifi
  - smart-home
  - firmware
license: Apache-2.0
compatibility: Build requires ESP-IDF v4.3 + xtensa-esp32-elf (ESP32) or riscv-esp32-elf (ESP32-C3) toolchain; targets ESP32-DevKitC, ESP32-WROVER-KIT, M5Stack, ESP32C3-DevKitM
metadata:
  author: Community
  version: "1.1.0"
---

# connectedhomeip-skill

AI Skill for Matter / CHIP (Connected Home over IP) firmware development on the Espressif fork of `project-chip/connectedhomeip`. Provides scenario-driven recipes, real API references, Kconfig/configuration guides, and common pitfalls, all grounded in the repository's own examples and headers. Supports the ESP32 family device examples (lock-app, all-clusters-app, etc.) and the chip-tool / python controller workflows.

## Core Principles

1. **Never invent APIs** — Every function, struct, cluster ID, or Kconfig symbol must come from `resources/` or the repo's `examples/`/`src/`. If it is not found, it does not exist.
2. **ESP32 examples use ESP-IDF v4.3 + `idf.py`** — Not GN/ninja directly for on-device firmware. GN is for host-side chip-tool and tests. Build firmware with `idf.py set-target esp32|esp32c3` then `idf.py build`.
3. **Entry point is `extern "C" void app_main()`** — ESP-IDF calls `app_main()`; CHIP stack init happens inside it: `nvs_flash_init()` → `CHIPDeviceManager::Init()` → `InitServer()` → `GetAppTask().StartAppTask()`.
4. **CHIPDeviceManager is the singleton driver** — `CHIPDeviceManager::GetInstance().Init(&callbacks)` boots the Matter stack; pass a `CHIPDeviceManagerCallbacks` subclass to receive device events and attribute changes.
5. **Rendezvous Mode is a Kconfig choice** — Configured under `Demo -> Rendezvous Mode` (BLE default, also Wi-Fi / Bypass / Thread / Ethernet). The numeric `CONFIG_RENDEZVOUS_MODE` value: Bypass=0, Wi-Fi=1, BLE=2, Thread=4, Ethernet=8.
6. **Default commissioning credentials are hard-coded in examples** — Discriminator `3840`, setup-pin-code `20202021`. These appear in chip-tool / python-controller commands verbatim.
7. **ZCL attribute writes use Ember API** — Server-side attribute updates go through `emberAfWriteAttribute(endpoint, clusterId, attrId, mask, &value, type)` and cluster IDs come from `<app/common/gen/cluster-id.h>` / `attribute-id.h`.
8. **Attribute changes flow back via `PostAttributeChangeCallback`** — Override `CHIPDeviceManagerCallbacks::PostAttributeChangeCallback` to react to cluster writes (e.g. OnOff → drive a relay/LED).
9. **Commissioning connectivity is observed through `ChipDeviceEvent`** — `DeviceEventType::kInternetConnectivityChange`, `kInterfaceIpAddressChanged`, `kSessionEstablished` are dispatched to your `DeviceEventCallback`.
10. **BLE must be enabled in sdkconfig** — `CONFIG_BT_ENABLED=y` and `CONFIG_BT_NIMBLE_ENABLED=y` (NimBLE is the Matter BLE stack on ESP32); IPv6 autoconfig must also be on.
11. **Custom partition table is required** — `CONFIG_PARTITION_TABLE_CUSTOM=y` with `partitions.csv`; the factory app needs ~1.9 MB (`1945K`).
12. **Always `git submodule update --init`** — Matter depends on many submodules (third_party); a partial checkout will not build.
13. **Sensors publish via cluster setters, not PostAttributeChangeCallback** — A sensor device (e.g. temperature) reports readings by calling the server's own setter (e.g. `emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(endpoint, celsius*100)`) from a timer/task. `PostAttributeChangeCallback` is for *actuator* devices reacting to controller writes, not for publishing sensor data. Sensor clusters also expose dedicated setter APIs (temperature/level-control/door-lock each have their own server headers under `src/app/clusters/`).
14. **Each cluster has its own server plugin and command path** — Beyond OnOff, DoorLock/LevelControl/etc. dispatch through their own server plugins (e.g. `emberAfPluginDoorLockServerActivateDoorLockCallback` for `LockDoor`/`UnlockDoor`, `emberAfPluginLevelControlClusterServerPostInitCallback` for startup hardware sync). Do not route every cluster through the OnOff `PostAttributeChangeCallback` branch — use the documented per-cluster callback.

## When to Use

**Applicable:**
- Building or customizing ESP32 / ESP32-C3 Matter device examples (lock-app, all-clusters-app, lighting concepts)
- Implementing or extending Zigbee Cluster Library (ZCL) clusters and attribute callbacks
- Configuring commissioning / Rendezvous (BLE, Wi-Fi, Bypass) and onboarding codes
- Driving a device with chip-tool (CLI) or the python controller (`chip-device-ctrl`)
- Wiring cluster attribute changes to hardware (LED, relay, sensor)
- Setting up ESP-IDF sdkconfig / Kconfig / partition table for a Matter app

**Not applicable:**
- General ESP-IDF / FreeRTOS questions unrelated to Matter/CHIP
- The upstream (non-Espressif) Matter SDK specifics for non-ESP platforms (EFR32, nRF, K32W, QPG, telink, mbed) — only ESP32 examples are deeply covered here
- Hardware/PCB design or RF certification
- Apple HomeKit / Google Thread 1.x stacks outside of the Matter data model

---

## Scenario Quick Reference (Recipes)

When user intent matches a scenario below, **read the corresponding recipe first** — it contains the real call chain, step-by-step build/commission flow, common errors, and verified code.

### Build & Bring-up

| recipe | scenario |
|---|---|
| `recipes/build_esp32_example.md` | 在 ESP32/ESP32-C3 上构建并烧录一个 CHIP 示例（lock-app / all-clusters-app） |
| `recipes/new_esp32_example.md` | 基于 lock-app 创建自定义 ESP32 Matter 设备示例 |

### Commissioning

| recipe | scenario |
|---|---|
| `recipes/commissioning_ble.md` | 通过 BLE 完成 Rendezvous 配网（默认模式） |
| `recipes/commissioning_wifi_bypass.md` | 通过 Wi-Fi / Bypass 模式配网 |
| `recipes/python_controller.md` | 使用 python `chip-device-ctrl` 进行配网与集群控制 |
| `recipes/chip_tool_cli.md` | 使用 chip-tool CLI 配对并发送 ZCL 命令 |

### Clusters & Application Logic

| recipe | scenario |
|---|---|
| `recipes/onoff_cluster_hardware.md` | 将 OnOff 集群属性变化绑定到 GPIO（LED / 继电器） |
| `recipes/custom_attribute_callback.md` | 实现 `PostAttributeChangeCallback` 响应集群写入并回写属性 |
| `recipes/device_event_handling.md` | 处理网络连接 / IP 变更 / 会话建立等 `ChipDeviceEvent` 事件 |
| `recipes/temperature_measurement_cluster.md` | 上报温度传感器测量值到 TemperatureMeasurement 集群（sensor 上报模式） |
| `recipes/door_lock_cluster.md` | DoorLock 集群服务端集成（LockState / PIN / RFID / ActivateDoorLockCallback） |
| `recipes/level_control_cluster.md` | Level Control 集群调光（current-level 写入 + PostInit 硬件同步） |

### Debugging & Bring-up Tools

| recipe | scenario |
|---|---|
| `recipes/chip_shell_debug.md` | 设备端 CHIP Shell CLI（device config / get 验证凭据、功能测试） |
| `recipes/pigweed_rpc_debug.md` | Pigweed Echo RPC 远程调试通道（UART pw_hdlc rpc_console） |

---

## Chip & Device Type Support

本仓库（Espressif fork）的 ESP32 设备示例支持以下目标：

| 设备类型 (Kconfig) | 说明 | 目标芯片 | 状态 LED GPIO |
|---|---|---|---|
| `CONFIG_DEVICE_TYPE_ESP32_DEVKITC` | ESP32-DevKitC（默认） | ESP32 (Xtensa) | `GPIO_NUM_2` |
| `CONFIG_DEVICE_TYPE_ESP32_WROVER_KIT` | ESP32-WROVER-KIT_V4.1 | ESP32 (Xtensa) | 见 `main/main.cpp` |
| `CONFIG_DEVICE_TYPE_M5STACK` | M5Stack | ESP32 (Xtensa) | 虚拟 LED（屏幕） |
| `CONFIG_DEVICE_TYPE_ESP32_C3_DEVKITM` | ESP32C3-DevKitM | ESP32-C3 (RISC-V) | 见 `main/main.cpp` |

设置目标：`idf.py set-target esp32`（Xtensa）或 `idf.py set-target esp32c3`（RISC-V，需 `riscv-esp32-elf` 工具链）。

## Key Configuration (sdkconfig.defaults)

| 配置项 | 作用 | 典型值 |
|---|---|---|
| `CONFIG_BT_ENABLED` | 启用蓝牙（BLE 配网必需） | `y` |
| `CONFIG_BT_NIMBLE_ENABLED` | 启用 NimBLE（CHIP 使用的 BLE 协议栈） | `y` |
| `CONFIG_LWIP_IPV6_AUTOCONFIG` | lwIP IPv6 自动配置 | `y` |
| `CONFIG_PARTITION_TABLE_CUSTOM` | 自定义分区表 | `y` |
| `CONFIG_PARTITION_TABLE_FILENAME` | 分区表文件 | `"partitions.csv"` |
| `CONFIG_ESPTOOLPY_BAUD` | 烧录波特率 | `921600` |
| `CONFIG_ESPTOOLPY_MONITOR_BAUD` | 监视串口波特率 | `115200` |
| `CONFIG_RENDEZVOUS_MODE` | Rendezvous 配网模式（Bypass=0/Wi-Fi=1/BLE=2/Thread=4/Ethernet=8） | `2` (BLE) |
| `CONFIG_ENABLE_CHIP_SHELL` | 启用 CHIP shell | 可选 |
| `CONFIG_ENABLE_PW_RPC` | 启用 Pigweed RPC 调试通道 | 可选 |

---

## Matter Device State (Commissioning Lifecycle)

```
[boot] nvs_flash_init → CHIPDeviceManager::Init → InitServer
   │
   ├─ Rendezvous BLE 未配网 → BLE 广播（fast）等待 Commissioner
   │        ↓ Commissioner 连接 + 提供凭据
   ├─ NetworkCommissioning (AddWiFiNetwork / EnableNetwork)
   │        ↓ close-ble
   ├─ WiFi 关联 → kInternetConnectivityChange (Established)
   │        ↓
   ├─ kInterfaceIpAddressChanged → Mdns::StartServer()
   │        ↓
   └─ kSessionEstablished（Commissioner 检测）→ 可接收 ZCL 命令
            ↓
        PostAttributeChangeCallback → 应用逻辑（锁/灯）
```

### Key Callback / Class Signatures (real, from repo headers)

```cpp
// examples/lock-app/esp32/main/include/CHIPDeviceManager.h
class CHIPDeviceManagerCallbacks {
public:
    virtual void DeviceEventCallback(const chip::DeviceLayer::ChipDeviceEvent * event, intptr_t arg);
    virtual void PostAttributeChangeCallback(chip::EndpointId endpoint, chip::ClusterId clusterId,
                                             chip::AttributeId attributeId, uint8_t mask,
                                             uint16_t manufacturerCode, uint8_t type,
                                             uint16_t size, uint8_t * value) {}
};

class CHIPDeviceManager {
public:
    static CHIPDeviceManager & GetInstance();
    CHIP_ERROR Init(CHIPDeviceManagerCallbacks * cb);
};

// src/app/server/Server.h
namespace chip::app::Server {  // (InitServer is in the global app namespace)
void InitServer(AppDelegate * delegate = nullptr);
}

// examples/lock-app/esp32/main/include/BoltLockManager.h
class BoltLockManager {
public:
    enum Action_t { LOCK_ACTION = 0, UNLOCK_ACTION, INVALID_ACTION };
    enum State_t  { kState_LockingInitiated, kState_LockingCompleted,
                    kState_UnlockingInitiated, kState_UnlockingCompleted };
    int  Init();
    bool IsUnlocked();
    bool InitiateAction(int32_t aActor, Action_t aAction);
    void SetCallbacks(Callback_fn_initiated, Callback_fn_completed);
};
```

---

## Critical Pitfalls (Must Read)

这些是该仓库中最常见的错误。违反任何一条都会导致固件无法工作或配网失败。

### 1. 必须在 `app_main()` 内初始化 CHIP 栈（顺序固定）

```cpp
// ❌ 错误 — 跳过 nvs 初始化，CHIP 栈持久化失败
extern "C" void app_main() {
    CHIPDeviceManager::GetInstance().Init(&cb);  // NVS 未就绪
}

// ✅ 正确 — nvs_flash_init 先于 CHIP 栈
extern "C" void app_main() {
    nvs_flash_init();                              // 1. ESP NVS
    CHIPDeviceManager & mgr = CHIPDeviceManager::GetInstance();
    mgr.Init(&EchoCallbacks);                      // 2. CHIP DeviceLayer
    InitServer();                                  // 3. Matter Server (mDNS等)
    GetAppTask().StartAppTask();                   // 4. 应用任务
}
```
依据：`examples/lock-app/esp32/main/main.cpp`。

### 2. ESP32 设备固件用 `idf.py`，不是 GN/ninja

```bash
# ❌ 错误 — 用 GN 直接构建设备固件
gn gen out/esp32 && ninja -C out/esp32

# ✅ 正确 — ESP-IDF 构建系统
idf.py set-target esp32
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```
GN/ninja 仅用于主机侧的 chip-tool 与单元测试。

### 3. Rendezvous 模式是 Kconfig 选项，需重新 menuconfig + 重新烧录

```bash
# ❌ 错误 — 期望改代码立即生效而不烧录
# 在 menuconfig 改 Bypass 后直接 commissioning

# ✅ 正确 — 改完 menuconfig 必须重新 build + flash
idf.py menuconfig      # Demo -> Rendezvous Mode
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

### 4. BLE 配网必须启用 NimBLE

```ini
# ❌ 错误 — 用默认 bluedroid 或未启用 BT
# sdkconfig.defaults 缺少 BT 配置

# ✅ 正确 — sdkconfig.defaults
CONFIG_BT_ENABLED=y
CONFIG_BT_NIMBLE_ENABLED=y
```
依据：`examples/lock-app/esp32/sdkconfig.defaults`。

### 5. 属性变化必须经 `PostAttributeChangeCallback`，且要校验 endpoint/attr

```cpp
// ❌ 错误 — 不校验 endpoint/attribute 就动作
void PostAttributeChangeCallback(EndpointId ep, ClusterId c, AttributeId a,
                                 uint8_t m, uint16_t mc, uint8_t t, uint16_t s, uint8_t * v) {
    BoltLockMgr().InitiateAction(0, BoltLockManager::LOCK_ACTION);  // 任何写入都触发
}

// ✅ 正确 — 校验 OnOff 集群 + 属性 + endpoint
void OnOnOffPostAttributeChangeCallback(EndpointId endpointId, AttributeId attributeId, uint8_t * value) {
    VerifyOrExit(attributeId == ZCL_ON_OFF_ATTRIBUTE_ID, ...);
    VerifyOrExit(endpointId == 1 || endpointId == 2, ...);
    if (*value) BoltLockMgr().InitiateAction(AppEvent::kEventType_Lock, BoltLockManager::LOCK_ACTION);
    else        BoltLockMgr().InitiateAction(AppEvent::kEventType_Lock, BoltLockManager::UNLOCK_ACTION);
exit:
    return;
}
```
依据：`examples/lock-app/esp32/main/DeviceCallbacks.cpp`。

### 6. 服务端属性回写用 `emberAfWriteAttribute`，集群/属性 ID 来自 `gen/`

```cpp
// ❌ 错误 — 自造集群/属性 ID 数字
emberAfWriteAttribute(1, 0x0006, 0x0000, CLUSTER_MASK_SERVER, &v, ZCL_BOOLEAN_ATTRIBUTE_TYPE);

// ✅ 正确 — 使用 <app/common/gen/cluster-id.h> 宏
#include <app/common/gen/cluster-id.h>
#include <app/common/gen/attribute-id.h>
#include <app/common/gen/attribute-type.h>

EmberAfStatus status = emberAfWriteAttribute(
    1, ZCL_ON_OFF_CLUSTER_ID, ZCL_ON_OFF_ATTRIBUTE_ID,
    CLUSTER_MASK_SERVER, (uint8_t *) &newValue, ZCL_BOOLEAN_ATTRIBUTE_TYPE);
```
依据：`examples/lock-app/esp32/main/AppTask.cpp::UpdateClusterState`。

### 7. 默认配网凭据写死在示例中：discriminator 3840 / pin 20202021

```bash
# ❌ 错误 — chip-tool 使用错误的 discriminator/pin
chip-tool pairing ble 11111111 1234

# ✅ 正确 — 使用示例默认值（除非在 menuconfig 中改了）
chip-tool pairing ble 20202021 3840
```
依据：`examples/chip-tool/README.md`。

### 8. IP 变化 / 连接事件要重启 mDNS 服务

```cpp
// ❌ 错误 — 拿到 IP 后不重启 mDNS，commissioner 找不到设备
case DeviceEventType::kInterfaceIpAddressChanged:
    ESP_LOGI(TAG, "IP changed, but mDNS not restarted");
    break;

// ✅ 正确 — IP 分配后调用 chip::app::Mdns::StartServer()
case DeviceEventType::kInterfaceIpAddressChanged:
    if (event->InterfaceIpAddressChanged.Type == InterfaceIpChangeType::kIpV4_Assigned ||
        event->InterfaceIpAddressChanged.Type == InterfaceIpChangeType::kIpV6_Assigned) {
        chip::app::Mdns::StartServer();
    }
    break;
```
依据：`examples/lock-app/esp32/main/DeviceCallbacks.cpp`。

### 9. 应用任务循环中查询 CHIP 栈要加锁

```cpp
// ❌ 错误 — 在独立任务中无锁读取连接状态
sHaveServiceConnectivity = ConnectivityMgr().HaveServiceConnectivity();  // 数据竞争

// ✅ 正确 — TryLockChipStack()
if (PlatformMgr().TryLockChipStack()) {
    sHaveBLEConnections      = (ConnectivityMgr().NumBLEConnections() != 0);
    sHaveServiceConnectivity = ConnectivityMgr().HaveServiceConnectivity();
    PlatformMgr().UnlockChipStack();
}
```
依据：`examples/lock-app/esp32/main/AppTask.cpp::AppTaskMain`。

### 10. 自定义分区表，factory 分区约 1.9 MB

```csv
# ❌ 错误 — 默认分区表空间不足，CHIP 固件链接失败
# partitions.csv 不存在或 factory 太小

# ✅ 正确 — examples/lock-app/esp32/partitions.csv
nvs,      data, nvs,     ,        0x6000,
phy_init, data, phy,     ,        0x1000,
factory,  app,  factory, ,        1945K,
```

### 11. 子模块必须初始化，否则编译失败

```bash
# ❌ 错误 — 浅克隆后直接 build，third_party 缺失
idf.py build   # 大量 "file not found"

# ✅ 正确 — 初始化所有子模块
git submodule update --init --recursive
```

### 12. ESP32 仅支持 2.4 GHz Wi-Fi（不支持 5 GHz）

```
# ❌ 错误 — 将 ESP32 接入 5 GHz AP，配网后无法关联

# ✅ 正确 — 使用 2.4 GHz SSID；如需清除已存网络配置
idf.py -p /dev/ttyUSB0 erase_flash
```
依据：`examples/all-clusters-app/esp32/README.md`（"The ESP32 does not support 5GHz networks"）。

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 确认目标芯片（ESP32/ESP32-C3）、设备类型、Rendezvous 模式、需要的集群 |
| 2 | Recipe | 匹配 `recipes/` 中的场景，按其调用链/命令执行 |
| 3 | Query | recipes 未覆盖的 API/Kconfig 查 `resources/` |
| 4 | Validate | 校验头文件包含（`<app/common/gen/*>`、`<platform/CHIPDeviceLayer.h>`）、sdkconfig、分区表 |
| 5 | Confirm | 向用户说明：includes、app_main 初始化顺序、回调、menuconfig 选项 |
| 6 | Execute | **新项目**：复制最接近的示例（lock-app/all-clusters-app）再改；**已有项目**：就地编辑 |
| 7 | Build | `idf.py set-target` → `idf.py build`；修正编译错误 |
| 8 | Flash | `idf.py -p <port> flash monitor`；按 boot 键如需 |
| 9 | Commission | 用 chip-tool / python controller 配网并下发 ZCL 命令验证 |

### Step 6 Detail — 选择最接近的示例

- 通用所有集群 / 设备类型选择 → `examples/all-clusters-app/esp32/`
- OnOff → 物理动作（锁）映射 → `examples/lock-app/esp32/`
- 温度上报 → `examples/temperature-measurement-app/esp32/`
- 持久化存储示例 → `examples/persistent-storage/esp32/`
- CHIP shell → `examples/shell/esp32/`
- Pigweed RPC 调试 → `examples/pigweed-app/esp32/`
- 仅 IPv6（无 BLE/Wi-Fi） → `examples/ipv6only-app/esp32/`

复制整个示例目录后，修改 `main/main.cpp`、`AppTask.cpp`、`DeviceCallbacks.cpp`、`sdkconfig.defaults`、`partitions.csv` 与 `Kconfig.projbuild`。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API/struct 在 resources 中找不到 | 停止，告知用户该 API 不存在，不要臆造 |
| `idf.py build` 报 third_party 缺失 | `git submodule update --init --recursive` |
| 配网失败 / commissioner 连不上 | 检查 Rendezvous 模式与 BT/NimBLE 配置；确认 SSID 为 2.4 GHz |
| 拿到 IP 但 mDNS 不通 | 确认在 `kInterfaceIpAddressChanged` 中调用了 `chip::app::Mdns::StartServer()` |
| chip-tool 报 unpair / 找不到 | 确认 discriminator/pin（默认 3840 / 20202021），必要时 `erase_flash` 重置 |
| 链接报 factory 分区不足 | 增大 `partitions.csv` 中 factory 尺寸或精简集群 |
| 不确定目标芯片 | ESP32 用 Xtensa `idf.py set-target esp32`；ESP32-C3 用 RISC-V `set-target esp32c3` |

## References

- Scenario recipes → `recipes/` directory
- API quick reference → `resources/api_reference.md`
- Configuration reference → `resources/config_reference.md`
- Consolidated pitfalls → `resources/pitfalls.md`
- Example index → `resources/example_list.md`
- 仓库本体 → `D:/esp-skill/espressif-repos/connectedhomeip`
