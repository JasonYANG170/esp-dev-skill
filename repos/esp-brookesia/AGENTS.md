# AGENTS.md — Supplementary Agent Guide

> 核心规则、组件/板支持表、调用范式、踩坑清单、Recipe 索引与执行流程均在 `SKILL.md`。
> 本文件仅补充 `SKILL.md` 未覆盖的工程约定与工具链指引，**不重复**其内容。

## Project Context

- **Language**: C++（C++17 及以上，使用 boost::json / boost::thread / CRTP；少量 C 桥接如 `app_main`）
- **Target**: 乐鑫 ESP SoC（ESP32-S3 / ESP32-P4 / ESP32-C5 / ESP32-S31 等，视板而定）
- **Framework**: ESP-Brookesia v0.7（组件化），构建于 ESP-IDF >= v5.5
- **Toolchain**: ESP-IDF + ESP Component Manager（`idf.py`、`idf.py add-dependency`）
- **Entry point**: `extern "C" void app_main(void)`（C 链接），内部使用 C++ 命名空间 `esp_brookesia`

## Code Generation Conventions

### 文件命名

- 应用源文件：`main.cpp`（或 `*.cpp`），头文件 `*.hpp`
- 示例的子模块按功能拆分到 `main/modules/*.cpp` / `*.hpp`（参考 `examples/agent/chatbot/main/modules/`）
- 组件内的服务实现头位于 `components/*/include/brookesia/...`，Helper 头位于 `brookesia/service_helper/*.hpp`、`brookesia/agent_helper/*.hpp`

### Include 模式（来自各示例）

```cpp
// 基础（日志、检查宏、TaskScheduler 等）
#define BROOKESIA_LOG_TAG "Main"
#include "brookesia/lib_utils.hpp"

// 服务框架
#include "brookesia/service_manager.hpp"
#include "brookesia/service_helper.hpp"            // 汇总所有通用服务 Helper
// 或按需包含单个：
// #include "brookesia/service_helper/wifi.hpp"
// #include "brookesia/service_helper/nvs.hpp"
// #include "brookesia/service_helper/audio.hpp"
// #include "brookesia/service_helper/device.hpp"
// #include "brookesia/service_helper/sntp.hpp"

// HAL（需要板级设备的服务才包含）
#include "brookesia/hal_interface.hpp"
#include "brookesia/hal_adaptor.hpp"

// AI Agent
#include "brookesia/agent_helper.hpp"
// 等价于按需：
// #include "brookesia/agent_helper/manager.hpp"
// #include "brookesia/agent_helper/xiaozhi.hpp" / coze.hpp / openai.hpp

// MCP（让 LLM 调用设备能力）
#include "brookesia/mcp_utils/mcp_utils.hpp"
```

### 命名空间与类型别名

```cpp
using namespace esp_brookesia;
// 服务 Helper 别名（CRTP 派生类）
using WifiHelper   = service::helper::Wifi;
using NVSHelper    = service::helper::NVS;
using AudioHelper  = service::helper::Audio;
using DeviceHelper = service::helper::Device;
// Agent Helper 别名
using AgentHelper    = agent::helper::Manager;
using XiaoZhiHelper  = agent::helper::XiaoZhi;
using CozeHelper     = agent::helper::Coze;
using OpenaiHelper   = agent::helper::Openai;
using EmoteHelper    = service::helper::ExpressionEmote;
```

### 标准项目结构（板级工程，参考 `examples/agent/chatbot`）

```
MyProject/
├── main/
│   ├── idf_component.yml        # 依赖声明（espressif/brookesia_*）
│   ├── main.cpp                 # app_main
│   └── modules/                 # 按功能拆分的子模块
│       ├── ai_agents.{cpp,hpp}
│       ├── general_services.{cpp,hpp}
│       ├── wifi_provisioning.{cpp,hpp}
│       └── profiler.{cpp,hpp}
├── boards/                      # 可选：自定义板（覆盖 brookesia_hal_boards）
├── partitions.csv               # 分区表（唤醒词模型等需 model 分区）
├── sdkconfig.defaults           # 默认配置
└── CMakeLists.txt
```

### 标准入口（app_main）范式

参考 `examples/service/wifi/main/main.cpp` 与 `examples/agent/chatbot/main/main.cpp`：

```cpp
#define BROOKESIA_LOG_TAG "Main"
#include "brookesia/lib_utils.hpp"
#include "brookesia/service_manager.hpp"
#include "brookesia/service_helper/wifi.hpp"

using namespace esp_brookesia;
using WifiHelper = service::helper::Wifi;

extern "C" void app_main(void)
{
    // 1) 启动 ServiceManager（单例）
    auto &service_manager = service::ServiceManager::get_instance();
    BROOKESIA_CHECK_FALSE_EXIT(service_manager.init(), "Failed to init ServiceManager");
    BROOKESIA_CHECK_FALSE_EXIT(service_manager.start(), "Failed to start ServiceManager");

    // 2) RAII 清理守卫（stop 在函数返回时执行）
    lib_utils::FunctionGuard shutdown_guard([&service_manager]() {
        service_manager.stop();
    });

    // 3) 可用性检查 + bind（binding 活到函数末尾）
    BROOKESIA_CHECK_FALSE_EXIT(WifiHelper::is_available(), "Wifi service not available");
    auto binding = service_manager.bind(WifiHelper::get_name().data());
    BROOKESIA_CHECK_FALSE_EXIT(binding.is_valid(), "Failed to bind Wifi service");

    // 4) 业务逻辑：call_function_* / subscribe_event / EventMonitor
    auto start_result = WifiHelper::call_function_sync(
        WifiHelper::FunctionId::TriggerGeneralAction,
        BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralAction::Start));
    BROOKESIA_CHECK_FALSE_EXIT(start_result, "Failed to start WiFi");
}
```

> 需要 HAL 设备的服务（Audio/Device/Agent）：在步骤 1 之前加
> `hal::init_device(hal::AudioDevice::DEVICE_NAME);` 或 `hal::init_all_devices();`。

## Build Workflow

1. **安装 ESP-IDF**（命令行方式；官方不建议用 VSCode 扩展安装，否则依赖 `esp_board_manager` 的示例可能构建失败）。
2. **声明依赖**：`idf.py add-dependency "espressif/brookesia_service_wifi"`，或直接编辑 `main/idf_component.yml`。
3. **选择目标**：
   - 纯芯片工程：`idf.py set-target esp32s3`（或 `esp32p4` / `esp32c5` 等）
   - 板级工程（依赖板级外设）：`idf.py gen-bmgr-config -b <board>`（板名见 SKILL.md 板支持表）
4. **可选配置**：`idf.py menuconfig`（如 Agent 示例需在 `Example Configuration` 填 API Key/Bot ID；或开启各组件 `BROOKESIA_SERVICE_*_ENABLE_DEBUG_LOG`）。
5. **构建**：`idf.py build`
6. **烧录**：`idf.py -p <PORT> flash`
7. **监视**：`idf.py -p <PORT> monitor`（`Ctrl+]` 退出）

### 板级工程的 idf_ext.py 约定

板级示例（`examples/service/console`、`examples/service/audio`、`examples/service/device`、`examples/agent/chatbot`）项目根含 `idf_ext.py`，提供：

- 无需手动设置 `IDF_EXTRA_ACTIONS_PATH`
- 选板时自动按 `idf_component.yml` 下载 `esp_board_manager` 与 `brookesia_hal_boards`
- 执行 `idf.py gen-bmgr-config` 时自动加 `-c` 指向 `boards/`（先找工程本地，再找组件内 `boards/`）

## CRTP Helper 扩展约定（自定义服务/Agent）

如需自定义 Helper，按 `docs/en/service/helper/base.rst` 的 CRTP 契约：

```cpp
class MyHelper : public esp_brookesia::service::helper::Base<MyHelper> {
public:
    enum class FunctionId { Ping };
    enum class EventId    { Ready };
    static std::string_view get_name();
    static std::vector<FunctionSchema> get_function_schemas();
    static std::vector<EventSchema>    get_event_schemas();
};
```

派生类须定义 `FunctionId`/`EventId`、`get_name`、`get_function_schemas`、`get_event_schemas`；基类提供 `call_function_sync/async`、`subscribe_event`、`is_available`、`get_function_schema`、`get_event_schema`。

## 代码生成检查清单（Codegen Checklist）

- [ ] `main/idf_component.yml` 依赖均带 `espressif/` 前缀
- [ ] `#define BROOKESIA_LOG_TAG "X"` 在 include `brookesia/lib_utils.hpp` 之前
- [ ] 需要 HAL 的服务在 `ServiceManager::start()` 前调用 `hal::init_*`
- [ ] 调用顺序：`ServiceManager::start()` → `bind()` → Helper 调用/订阅
- [ ] `binding` 与长生命周期的 `connection` 存入成员/容器，不在窄作用域内析构
- [ ] 同步调用按 schema 顺序传参，需要时带 `service::helper::Timeout(ms)`
- [ ] 返回 Array/Object 的函数显式给模板参数（如 `<boost::json::array>`）
- [ ] 枚举参数用 `BROOKESIA_DESCRIBE_TO_STR(...)`，返回值用 `BROOKESIA_DESCRIBE_FROM_JSON`/`BROOKESIA_DESCRIBE_STR_TO_ENUM`
- [ ] 自定义 struct/enum 用 `BROOKESIA_DESCRIBE_STRUCT`/`BROOKESIA_DESCRIBE_ENUM` 描述
- [ ] `subscribe_event` 回调签名（item 数量与类型）与事件 schema 一致
- [ ] 多核 SPI LCD 用 `BROOKESIA_THREAD_CONFIG_GUARD({.core_id=N})` 锁核
- [ ] 错误检查统一用 `BROOKESIA_CHECK_FALSE_EXIT/RETURN`、`BROOKESIA_CHECK_NULL_RETURN` 等

## Do Not Modify

- 仓库源码目录 `D:/esp-skill/espressif-repos/esp-brookesia/` 下的所有文件（只读参考，改动不要提交）
- 本 Skill 的 `SKILL.md` frontmatter（Skill 元数据）
- `resources/` 是文档速查，如需更正应基于仓库 `docs/` 重新核对
