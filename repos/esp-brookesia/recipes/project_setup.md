# 项目搭建与第一次服务调用

> **适用摘要**: 新建 ESP-Brookesia 项目：声明组件依赖、初始化 ServiceManager、绑定服务、完成第一次服务调用。这是使用任何 Brookesia 服务（Wi-Fi / NVS / Audio 等）的统一前置步骤。

## 触发意图

- "新建 ESP-Brookesia 项目"
- "怎么添加 brookesia 组件依赖"
- "ServiceManager 怎么初始化"
- "板级工程和芯片工程怎么选"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | >= v5.5（master/v0.7）|
| 硬件 | Flash >= 8MB、PSRAM >= 4MB（Agent 示例需 16MB/8MB）|
| 参考示例 | `examples/service/wifi`（纯芯片）、`examples/service/console`（板级）|

## 分步说明

### 1. 声明依赖（`main/idf_component.yml`）

依赖名必须带 `espressif/` 前缀。以 Wi-Fi 服务为例：

```yaml
## IDF Component Manager Manifest File
dependencies:
  espressif/brookesia_service_wifi:
    version: "*"
  # 如需 NVS：
  espressif/brookesia_service_nvs:
    version: "*"
```

或命令行添加：

```bash
idf.py add-dependency "espressif/brookesia_service_wifi"
```

> 仅需“Helper 代码能编译”而不链接具体服务实现时，可只依赖 `espressif/brookesia_service_helper`。

### 2. 选择目标

**纯芯片工程**（如 `examples/service/wifi`，只用片上 Wi-Fi）：

```bash
idf.py set-target esp32s3
```

**板级工程**（如 `examples/service/console`、`examples/service/audio`、`examples/service/device`、`examples/agent/chatbot`，依赖板级 Audio/LCD/Touch）：

```bash
idf.py gen-bmgr-config -b esp_vocat_board_v1_2
```

板级工程根目录含 `idf_ext.py`，会自动下载 `esp_board_manager` 与 `brookesia_hal_boards` 并定位 `boards/`。

### 3. 最小 app_main（Wi-Fi 服务，纯芯片）

```cpp
#define BROOKESIA_LOG_TAG "Main"
#include "brookesia/lib_utils.hpp"
#include "brookesia/service_manager.hpp"
#include "brookesia/service_helper/wifi.hpp"

using namespace esp_brookesia;
using WifiHelper = service::helper::Wifi;

extern "C" void app_main(void)
{
    auto &service_manager = service::ServiceManager::get_instance();
    BROOKESIA_CHECK_FALSE_EXIT(service_manager.init(), "Failed to init ServiceManager");
    BROOKESIA_CHECK_FALSE_EXIT(service_manager.start(), "Failed to start ServiceManager");

    lib_utils::FunctionGuard shutdown_guard([&service_manager]() {
        service_manager.stop();
    });

    BROOKESIA_CHECK_FALSE_EXIT(WifiHelper::is_available(), "Wifi service is not available");

    auto binding = service_manager.bind(WifiHelper::get_name().data());
    BROOKESIA_CHECK_FALSE_EXIT(binding.is_valid(), "Failed to bind Wifi service");

    // 触发 Wi-Fi Start（start 会自动触发 init，多数情况无需显式 Init）
    auto start_result = WifiHelper::call_function_sync(
        WifiHelper::FunctionId::TriggerGeneralAction,
        BROOKESIA_DESCRIBE_TO_STR(WifiHelper::GeneralAction::Start));
    BROOKESIA_CHECK_FALSE_EXIT(start_result, "Failed to start WiFi");
}
```

### 4. 构建、烧录、监视

```bash
idf.py build
idf.py -p <PORT> flash
idf.py -p <PORT> monitor
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 组件管理器找不到依赖 | `idf_component.yml` 缺 `espressif/` 前缀 | 写成 `espressif/brookesia_service_wifi` |
| 板级工程编译缺组件 | 用了 `set-target` 而非 `gen-bmgr-config` | 改用 `idf.py gen-bmgr-config -b <board>` |
| VSCode 构建失败 | 用扩展装的 ESP-IDF 环境 | 改用命令行安装 ESP-IDF |
| `is_available()` 为 false | 服务组件未加入构建 | 确认 `idf_component.yml` 依赖了对应组件 |
| `binding.is_valid()` 为 false | 未先 `ServiceManager::start()` | 先 `init()`/`start()` 再 `bind()` |

## 参考

- `examples/service/wifi/main/main.cpp` — 纯芯片工程的完整 ServiceManager 范式
- `examples/service/wifi/main/idf_component.yml` — 含 `override_path` 的依赖示例
- `docs/en/getting_started.rst` — 版本、环境、组件获取、示例使用
