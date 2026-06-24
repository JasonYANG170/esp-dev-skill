# AGENTS.md — Supplementary Agent Guide

> 核心规则、Recipe 索引、陷阱清单、执行工作流都在 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未覆盖的工程约定与工具链说明，不重复内容。

## Project Context

- **Language**: C（部分 esp_modem C++ API、esp_mqtt_cxx/asio 为 C++） · **Target**: ESP32 全系列（ESP32/S2/S3/C3/C6/P4 等，按组件支持矩阵） · **Toolchain**: ESP-IDF v5.x + CMake + GCC（`idf.py`）
- **Repository**: Espressif [esp-protocols](https://github.com/espressif/esp-protocols)，一组独立的 ESP-IDF managed component

## 组件清单（components/）

| 目录 | 语言 | 一句话用途 |
|---|---|---|
| `esp_modem` | C/C++ | 蜂窝模组 AT 命令 + PPPoS 数据通信 |
| `mdns` | C | 组播 DNS 服务发现 |
| `esp_websocket_client` | C | WebSocket 客户端（ws/wss） |
| `eppp_link` | C | 双 MCU 间 PPP 组网（UART/SPI/SDIO/ETH） |
| `esp_dns` | C | DoT / DoH / TCP DNS |
| `esp_mqtt_cxx` | C++ | MQTT C++ 封装 |
| `mosquitto` | C | Mosquitto 移植 |
| `asio` | C++ | Asio 异步框架移植 |
| `mbedtls_cxx` | C++ | mbedTLS C++ 封装 |
| `libwebsockets` | C | libwebsockets 移植 |
| `sock_utils` | C | socket 辅助 |
| `eppp_link` | C | PPP 链路 |
| `console_simple_init` / `console_cmd_*` | C | console 命令组件 |

> 本 Skill 主要覆盖前 5 个有独立 docs 与完整 API 的核心组件；其余组件按 README 处理。

## 文件命名与 include 约定

### 头文件 include（来自真实头文件）

```c
// esp_modem C API
#include "esp_modem_api.h"          // AT 命令函数（esp_modem_sync, get_signal_quality, ...）
#include "esp_modem_config.h"       // esp_modem_dte_config_t + ESP_MODEM_DTE_DEFAULT_CONFIG()
#include "esp_modem_dce_config.h"   // esp_modem_dce_config_t + ESP_MODEM_DCE_DEFAULT_CONFIG(APN)
#include "esp_modem_c_api_types.h"  // esp_modem_dce_t, 枚举, ESP_MODEM_C_API_STR_BUF_SIZE
// USB DTE（按需）
#include "esp_modem_usb_c_api.h"
#include "esp_modem_usb_config.h"

// mDNS
#include "mdns.h"

// WebSocket
#include "esp_websocket_client.h"

// eppp_link
#include "eppp_link.h"

// esp_dns
#include "esp_dns.h"
```

### esp_modem C++ API（modem_console 等 C++ example）

```cpp
#include "esp_modem_api.hpp"          // C++ 包装
#include "esp_modem_dce_factory.hpp"  // dce_factory::Factory
#include "esp_modem_dte.hpp"
```

## 标准工程结构（managed component 方式）

```
MyProject/
├── CMakeLists.txt                 # 顶层：get_target_property / idf build
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml          # 声明依赖：espressif/esp_modem, espressif/mdns ...
│   └── main.c                     # app_main
├── sdkconfig.defaults             # 默认 Kconfig（如 CONFIG_ESP_MODEM_USE_PPP_MODE=y）
└── sdkconfig.defaults.<chip>      # 芯片特定默认
```

`main/idf_component.yml` 示例（按需声明）：
```yaml
dependencies:
  espressif/esp_modem: "^2.0.2"
  idf:
    version: ">=5.0"
```

## 标准入口 / 初始化模式（esp_modem PPPoS）

```c
void app_main(void)
{
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, ESP_EVENT_ANY_ID, &on_ip_event, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(NETIF_PPP_STATUS, ESP_EVENT_ANY_ID, &on_ppp_changed, NULL));

    esp_modem_dce_config_t dce_config = ESP_MODEM_DCE_DEFAULT_CONFIG(CONFIG_EXAMPLE_MODEM_PPP_APN);
    esp_netif_config_t netif_ppp_config = ESP_NETIF_DEFAULT_PPP();
    esp_netif_t *esp_netif = esp_netif_new(&netif_ppp_config);
    assert(esp_netif);

    esp_modem_dte_config_t dte_config = ESP_MODEM_DTE_DEFAULT_CONFIG();
    // 按 Kconfig 覆盖 UART 引脚、buffer、流控等

    esp_modem_dce_t *dce = esp_modem_new_dev(ESP_MODEM_DCE_SIM7600, &dte_config, &dce_config, esp_netif);
    assert(dce);

    // AT 交互 ...
    esp_modem_set_mode(dce, ESP_MODEM_MODE_DATA);   // 拨号，等待 IP_EVENT_PPP_GOT_IP
}
```

## Build Workflow

1. 设置 ESP-IDF 环境（`. $IDF_PATH/export.sh` / Windows 用 `export.bat` 或 IDF Eclipse/VSCode 插件）
2. 设置目标芯片：`idf.py set-target esp32s3`
3. 配置：`idf.py menuconfig`（按需调 `CONFIG_ESP_MODEM_*`、`CONFIG_MDNS_*`）
4. 编译：`idf.py build`
5. 烧录监视：`idf.py -p COMx flash monitor`
6. 调试：日志 `ESP_LOGI`；蜂窝模组可用 `examples/modem_console` 交互式发 AT

## 代码生成 Checklist

- [ ] `idf_component.yml` 仅声明实际用到的 espressif/* 组件
- [ ] esp_modem：先 `esp_netif_new()` 再 `esp_modem_new_dev()`，netif 句柄传入
- [ ] esp_modem：文本输出缓冲 >= `ESP_MODEM_C_API_STR_BUF_SIZE`
- [ ] esp_modem：注册了 `IP_EVENT` / `NETIF_PPP_STATUS` 事件处理
- [ ] esp_modem：HW 流控时调用了 `esp_modem_set_flow_control(dce, 2, 2)`
- [ ] esp_modem：CMUX 仅用于支持设备（非 SIM7000），A76xx 关 defrag
- [ ] mdns：`mdns_init()` → `mdns_hostname_set()` → `mdns_service_add()`
- [ ] mdns：查询结果 `mdns_query_results_free()`
- [ ] websocket：事件回调内不调用 stop/destroy/close；`crt_bundle_attach` 用于 wss
- [ ] eppp_link：server/client 用各自默认配置宏（IP 已交叉）
- [ ] 所有 API/结构体/Kconfig 符号都能在 `components/*/include` 或 docs 中找到

## Do Not Modify

- `D:/esp-skill/espressif-repos/esp-protocols/` 源码仓库本体（仅作为参考与 example 来源）
- `resources/` —— API 文档源（只读）
- `SKILL.md` front matter —— Skill 元数据
