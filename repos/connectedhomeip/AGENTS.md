# AGENTS.md — Supplementary Agent Guide

> Core rules, recipe index, pitfalls, and execution workflow are all in `SKILL.md`.
> This file covers **only** conventions and tooling not present in `SKILL.md`. Do not duplicate content.

## Project Context

**Language**: C/C++ · **Target**: ESP32 / ESP32-C3（Xtensa / RISC-V） · **Toolchain**: ESP-IDF v4.3（`xtensa-esp32-elf` 或 `riscv-esp32-elf`）+ GN/ninja（主机侧 chip-tool）

本仓库是 Espressif 对上游 `project-chip/connectedhomeip`（CSA Matter/CHIP 参考实现）的分叉。设备固件走 ESP-IDF；主机侧工具（chip-tool、单元测试）走 GN + ninja。

## Code Generation Conventions

### File Naming
- 设备应用源文件：`*.cpp`，头文件：`*.h`
- 应用任务：`AppTask.cpp` / `include/AppTask.h`
- 设备回调：`DeviceCallbacks.cpp` / `include/DeviceCallbacks.h`
- 设备管理：`CHIPDeviceManager.cpp` / `.h`
- ZCL 生成头（不要手改）：`<app/common/gen/cluster-id.h>`、`attribute-id.h`、`attribute-type.h`、`att-storage.h`
- ESP-IDF 配置：`sdkconfig.defaults`（提交）+ `sdkconfig`（生成，不提交）
- 分区表：`partitions.csv`
- 应用 Kconfig：`main/Kconfig.projbuild`

### Include Pattern（ESP32 设备应用）
```cpp
#include <app/common/gen/attribute-id.h>
#include <app/common/gen/attribute-type.h>
#include <app/common/gen/cluster-id.h>
#include <app/server/OnboardingCodesUtil.h>
#include <app/server/Server.h>              // InitServer()
#include <app/util/af-enums.h>
#include <app/util/attribute-storage.h>
#include <platform/CHIPDeviceLayer.h>       // PlatformMgr / ConnectivityMgr / ConfigurationMgr
#include <platform/internal/CHIPDeviceLayerInternal.h>
#include <support/CodeUtils.h>              // VerifyOrExit
#include <support/ErrorStr.h>               // ErrorStr()
#include <core/CHIPCore.h>
#include <core/CHIPError.h>

// ESP-IDF
#include "esp_log.h"
#include "esp_system.h"
#include "nvs_flash.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
```

### Standard ESP32 Example Structure
```
examples/<app>/esp32/
├── CMakeLists.txt
├── README.md
├── partitions.csv                 # 自定义分区表（factory ~1.9MB）
├── sdkconfig.defaults             # BT/NimBLE/lwIP/分区默认
├── third_party/                   # 连接 CHIP 上游
└── main/
    ├── CMakeLists.txt
    ├── Kconfig.projbuild          # Demo 菜单（Rendezvous 模式 / 设备类型）
    ├── main.cpp                   # extern "C" void app_main()
    ├── AppTask.cpp                # 应用任务 + 事件队列
    ├── DeviceCallbacks.cpp        # CHIPDeviceManagerCallbacks 实现
    ├── CHIPDeviceManager.cpp
    ├── <Peripheral>Manager.cpp    # 如 BoltLockManager
    ├── LEDWidget.cpp / Button.cpp
    ├── Rpc.cpp                    # CONFIG_ENABLE_PW_RPC 时启用
    └── include/
        ├── AppConfig.h            # APP_TASK_NAME, 按钮GPIO, LED GPIO 等
        ├── AppEvent.h             # 事件类型枚举
        ├── AppTask.h
        ├── CHIPDeviceManager.h
        ├── DeviceCallbacks.h
        └── ...
```

### Canonical `app_main` Pattern（ESP32）
```cpp
extern "C" void app_main()
{
    // 1. ESP NVS（CHIP 持久化依赖）
    if (nvs_flash_init() != ESP_OK) { return; }

#if CONFIG_ENABLE_PW_RPC
    chip::rpc::Init();
#endif
#if CONFIG_ENABLE_CHIP_SHELL
    chip::LaunchShell();
#endif

    // 2. CHIP DeviceLayer 初始化（注册回调）
    CHIPDeviceManager & deviceMgr = CHIPDeviceManager::GetInstance();
    deviceMgr.Init(&EchoCallbacks);   // CHIPDeviceManagerCallbacks 子类

    // 3. Matter Server（mDNS / 安全会话 / 数据模型）
    InitServer();

    // 4. 启动应用任务（事件循环，永不返回）
    GetAppTask().StartAppTask();
}
```

### ESP32 Logging Convention
```cpp
static const char * TAG = "lock-app";
ESP_LOGI(TAG, "...");
ESP_LOGE(TAG, "...: %s", ErrorStr(err));
```
监视串口默认 115200 8N1（`CONFIG_ESPTOOLPY_MONITOR_BAUD`）。退出 monitor：`Ctrl+]`。

### 回调注册顺序（关键）
1. 定义 `CHIPDeviceManagerCallbacks` 子类（重写 `DeviceEventCallback` 与 `PostAttributeChangeCallback`）。
2. `CHIPDeviceManager::Init(&callbacks)` 注册。
3. `InitServer()` 启动应用服务层。
4. 应用任务内通过 `PlatformMgr().TryLockChipStack() / UnlockChipStack()` 安全查询栈状态。

## Build Workflow（设备固件）

1. 准备 ESP-IDF v4.3：`git clone esp-idf` → `checkout v4.3` → `./install.sh` → `. ./export.sh`
2. 激活 CHIP 工具：`source ./scripts/activate.sh`
3. 设置目标：`idf.py set-target esp32`（或 `esp32c3`）
4. 配置：`idf.py menuconfig`（`Demo -> Rendezvous Mode`、`Demo -> Device Type`、`Component config -> CHIP Device Layer -> WiFi Station Options`）
5. 构建：`idf.py build`
6. 烧录监视：`idf.py -p /dev/ttyUSB0 flash monitor`
7. 清除（重置配网）：`idf.py -p /dev/ttyUSB0 erase_flash`

## Build Workflow（chip-tool，主机侧）

```bash
cd examples/chip-tool
git submodule update --init
source third_party/connectedhomeip/scripts/activate.sh
gn gen out/debug
ninja -C out/debug
# 产物：out/debug/chip-tool
```

## Python Controller Workflow

```bash
./scripts/build_python.sh -m platform
source ./out/python_env/bin/activate
chip-device-ctrl
```

## Code Generation Checklist（ESP32 Matter 应用）

- [ ] `sdkconfig.defaults` 含 `CONFIG_BT_ENABLED=y` + `CONFIG_BT_NIMBLE_ENABLED=y`
- [ ] `CONFIG_LWIP_IPV6_AUTOCONFIG=y`
- [ ] 自定义分区表：`CONFIG_PARTITION_TABLE_CUSTOM=y` + `partitions.csv`（factory ≥ 1.9MB）
- [ ] `app_main()` 顺序：`nvs_flash_init` → `CHIPDeviceManager::Init` → `InitServer` → `StartAppTask`
- [ ] `CHIPDeviceManagerCallbacks` 重写 `PostAttributeChangeCallback` 并校验 endpoint/cluster/attribute
- [ ] IP 变化事件中调用 `chip::app::Mdns::StartServer()`
- [ ] 应用任务查询栈状态使用 `TryLockChipStack()/UnlockChipStack()`
- [ ] Rendezvous 模式经 menuconfig 选择并重新 build + flash
- [ ] 服务端属性回写使用 `emberAfWriteAttribute` + `gen/cluster-id.h` 宏
- [ ] 子模块已初始化（`git submodule update --init --recursive`）

## Do Not Modify

- `<app/common/gen/*.h>` — ZAP 生成的集群/属性 ID，改了会与上游数据模型不一致
- `src/`（CHIP 核心实现）、`third_party/`、`examples/platform/esp32/`（平台适配层）—— 视为只读；改动只在 `examples/<app>/esp32/main/` 内
- `LICENSE`、`CONTRIBUTING.md`、`CODE_OF_CONDUCT.md`
