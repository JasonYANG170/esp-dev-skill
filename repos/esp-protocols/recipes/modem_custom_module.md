# 自定义蜂窝模组（Custom Module）

> **适用摘要**: 当官方内置模组枚举（SIM7600/SIM800/BG96 等）不满足时，通过继承 `GenericModule` 自定义模组类，添加私有 AT 命令，并用 `ESP_MODEM_DCE_CUSTOM` 创建 DCE。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-protocols/resources/`, source/examples in `repos/esp-protocols/`, and this recipe path `repos/esp-protocols/recipes/modem_custom_module.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "esp_modem 添加自定义模组"
- "自定义 AT 命令"
- "ESP_MODEM_DCE_CUSTOM"
- "私有模组命令"
- "GenericModule 继承"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_ESP_MODEM_ADD_CUSTOM_MODULE=y` |
| 头文件 | `CONFIG_ESP_MODEM_CUSTOM_MODULE_HEADER` 指向自定义头（默认 `custom_module.hpp`） |
| C++ | 自定义模组用 C++ 实现（`GenericModule`），example main 用 `.cpp` |
| 参考示例 | `components/esp_modem/examples/pppos_client/main/custom_module.hpp` |

## 分步说明

### 1. 启用 custom module（menuconfig）

```
Component config → esp-modem → Add support for custom module in C-API = y
  → Header file name which defines custom DCE creation = "custom_module.hpp"
```

这会启用 C-API 中 `ESP_MODEM_DCE_CUSTOM` 的支持，并在创建时调用头文件中的 `esp_modem_create_custom_dce()`。

### 2. 编写自定义模组头 `custom_module.hpp`

继承 `GenericModule`，添加私有命令（如读取网络时间）。下面骨架对齐 pppos_client example 的写法：

```cpp
#pragma once
#include "esp_modem_api.hpp"
#include "esp_modem_dce_factory.hpp"
#include "esp_modem_command_helper.hpp"
#include "esp_modem_command_declare.inc"

// 自定义模组类：继承 GenericModule，添加私有 AT 命令
class CustomModule : public GenericModule {
public:
    using GenericModule::GenericModule;   // 继承构造

    // 私有命令示例：读取模组时间 AT+CCLK?
    command_result get_time(std::string &time)
    {
        // 调用 DCE 的 command() 发送私有 AT，并解析返回行
        return dce->command("AT+CCLK?\r", [&, this](uint8_t *data, size_t len) {
            // 解析 +CCLK: "yy/MM/dd,hh:mm:ss+zz"
            // ... 填充 time 字符串
            return command_result::OK;
        }, 1000);
    }
};

// C-API 创建入口（CONFIG_ESP_MODEM_CUSTOM_MODULE_HEADER 指向本文件）
struct esp_modem_dce_wrap;   // 前置声明
std::unique_ptr<DCE> esp_modem_create_custom_dce(const esp_modem_dce_config_t *dce_config,
                                                 std::unique_ptr<DTE> dte,
                                                 esp_netif_t *netif)
{
    return dce_factory::Factory::create_unique_dce_from<CustomModule, DCE*>(*dce_config,
                                                                            std::move(dte),
                                                                            netif);
}
```

> 具体私有命令的解析逻辑请直接参考 `components/esp_modem/examples/pppos_client/main/custom_module.hpp`，其中 `esp_modem_get_time` 即为自定义命令。

### 3. 声明自定义 C-API 原型（在 main 中）

```c
#ifdef CONFIG_EXAMPLE_MODEM_DEVICE_CUSTOM
// 由 custom_module.hpp 提供
esp_err_t esp_modem_get_time(esp_modem_dce_t *dce_wrap, char *p_time);
#endif
```

### 4. 用 ESP_MODEM_DCE_CUSTOM 创建 DCE 并调用自定义命令

```c
#include "esp_modem_api.h"

// ... 同 PPPoS recipe：先建 PPP netif ...
esp_modem_dce_t *dce = esp_modem_new_dev(ESP_MODEM_DCE_CUSTOM,
                                         &dte_config, &dce_config, esp_netif);
assert(dce);

// 调用自定义命令
char time[ESP_MODEM_C_API_STR_BUF_SIZE];
if (esp_modem_get_time(dce, time) == ESP_OK) {
    ESP_LOGI(TAG, "Modem time: %s", time);
}

// 之后照常 esp_modem_set_mode(dce, ESP_MODEM_MODE_DATA); ...
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 链接找不到 `esp_modem_create_custom_dce` | 未启用 custom module 或头文件名不对 | menuconfig 启用，且头文件名与 `CONFIG_ESP_MODEM_CUSTOM_MODULE_HEADER` 一致 |
| `GenericModule` 找不到 | 没包含 `esp_modem_api.hpp` | `#include "esp_modem_api.hpp"` |
| 自定义命令解析错误 | 返回行格式不符预期 | 在 lambda 中容错解析，必要时用 `esp_modem_at_raw` |
| 编译变慢/二进制变大 | 误开 development mode | 仅改私有命令时无需 `CONFIG_ESP_MODEM_ENABLE_DEVELOPMENT_MODE` |

## 参考

- `components/esp_modem/examples/pppos_client/main/custom_module.hpp` — 自定义模组示例
- `components/esp_modem/examples/pppos_client/main/pppos_client_main.c` — 使用 `ESP_MODEM_DCE_CUSTOM` 的完整流程
- `components/esp_modem/Kconfig` — `ESP_MODEM_ADD_CUSTOM_MODULE` / `ESP_MODEM_CUSTOM_MODULE_HEADER` / `ESP_MODEM_ENABLE_DEVELOPMENT_MODE`
- `docs/esp_modem/en/README.rst` — "Other devices" 与 Extensibility 小节
