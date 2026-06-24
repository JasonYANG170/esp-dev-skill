# AGENTS.md — Supplementary Agent Guide

> 核心规则、配方索引、陷阱清单、执行工作流均在 `SKILL.md`。
> 本文件仅补充 `SKILL.md` 未涵盖的工程约定与工具链指引，**不重复**其内容。

## Project Context

- **语言**: C · **目标**: Espressif ESP32 SoC 全系（ESP32 / S2 / S3 / C2 / C3 / C6 等）
- **框架**: ESP-IDF（`>= release/v5.1`，推荐 release/v5.5；v6.0 已兼容；4.x 仅 `idf_4_x_compat` 分支）
- **构建工具**: `idf.py`（CMake + Ninja/GNU Make）
- **依赖管理**: ESP-IDF Component Manager（`idf_component.yml`）
- **协议/编码**: 默认 HTTPS 上报到 `client.insights.espressif.com`，可选 MQTT(TLS)，载荷 CBOR 编码
- **License**: Apache-2.0（仓库根 `LICENSE`）

## Code Generation Conventions

### File Naming
- 应用入口源文件：`app_main.c`（约定名，`app_main(void)` 为入口）
- 头文件一律 `.h`，源文件 `.c`
- Auth Key 文件名约定：`insights_auth_key.txt`（HTTPS 嵌入）；改名需同步改 `main/CMakeLists.txt` 中的 `target_add_binary_data` 与源码里的 `asm("_binary_insights_auth_key_txt_start")` 符号

### Include Pattern
ESP-Insights 是 ESP-IDF 组件，**不要**写 `#include "../../components/..."` 这类相对路径。按组件 include：

```c
/* 核心 */
#include "esp_insights.h"                  // esp_insights_init / config / transport
#include "esp_diagnostics.h"               // log hook, ESP_DIAG_EVENT, device_info, data types

/* 可选（按 Kconfig 守卫，未启用时这些头内符号不可用） */
#include "esp_diagnostics_metrics.h"       // esp_diag_metrics_*  (需 CONFIG_DIAG_ENABLE_METRICS)
#include "esp_diagnostics_variables.h"     // esp_diag_variable_* (需 CONFIG_DIAG_ENABLE_VARIABLES)
#include "esp_diagnostics_system_metrics.h"// esp_diag_heap_metrics_* / esp_diag_wifi_metrics_*
#include "esp_diagnostics_network_variables.h" // esp_diag_network_variables_init (需 CONFIG_DIAG_ENABLE_NETWORK_VARIABLES)
#include "esp_diag_data_store.h"           // 数据存储事件 / 直接读写（高级用法）

/* 配套基础库（应用侧通常需要） */
#include "esp_log.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "nvs_flash.h"
```

### HTTPS Auth Key 嵌入模式（来自 examples/*/main/app_main.c）
```c
#ifdef CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS
extern const char insights_auth_key_start[] asm("_binary_insights_auth_key_txt_start");
extern const char insights_auth_key_end[]   asm("_binary_insights_auth_key_txt_end");
#endif
```

### Standard Project Structure（基于 examples）
```
my_insights_app/
├── CMakeLists.txt                  # 顶层：cmake_minimum_required + include($ENV{IDF_PATH}/tools/cmake/project.cmake)
├── partitions.csv                  # 自定义分区表（含 coredump，MQTT 还需 fctry）
├── sdkconfig.defaults              # 预置 Kconfig（ESP_INSIGHTS_ENABLED、coredump、metrics...）
└── main/
    ├── CMakeLists.txt              # idf_component_register + (HTTPS) target_add_binary_data
    ├── idf_component.yml           # 依赖 espressif/esp_insights（override_path 指向本地组件或 registry 版本）
    ├── app_main.c                  # app_main(void)
    └── insights_auth_key.txt       # 仅 HTTPS：粘贴仪表盘 Auth Key（勿提交到公开仓库）
```

`main/idf_component.yml` 引用本地组件示例（来自 `examples/minimal_diagnostics/main/idf_component.yml`）：
```yaml
dependencies:
  espressif/esp_insights:
    version: ">=1.1.0"
    override_path: "../../../components/esp_insights"   # 仓库内开发时用
```

### Canonical Init Pattern（来自 examples/minimal_diagnostics/main/app_main.c）
```c
#include "esp_log.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "protocol_examples_common.h"
#include "esp_insights.h"
#include "esp_rmaker_utils.h"

#ifdef CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS
extern const char insights_auth_key_start[] asm("_binary_insights_auth_key_txt_start");
#endif

void app_main(void)
{
    /* 1. NVS */
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    /* 2. 网络 + 默认事件循环 */
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());   /* 或用户自己的 Wi-Fi 连接 */

    /* 3. 时间同步（可选）：未同步则用相对启动时间(µs) */
    esp_rmaker_time_sync_init(NULL);

    /* 4. ESP-Insights */
    esp_insights_config_t config = {
        .log_type = ESP_DIAG_LOG_TYPE_ERROR | ESP_DIAG_LOG_TYPE_WARNING | ESP_DIAG_LOG_TYPE_EVENT,
#ifdef CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS
        .auth_key = insights_auth_key_start,
#endif
    };
    ret = esp_insights_init(&config);
    if (ret != ESP_OK) {
        ESP_LOGE("app", "Failed to init ESP Insights, err:0x%x", ret);
    }
    ESP_ERROR_CHECK(ret);

    /* 之后即可正常用 ESP_LOGE/W、ESP_DIAG_EVENT、metrics/variables API */
}
```
要点顺序：NVS → netif/event loop → 联网 → 时间同步 → `esp_insights_init` → 业务循环。

## Build Workflow
```bash
# 1. 设置目标芯片
idf.py set-target esp32        # 或 esp32s3 / esp32c3 / esp32c6 ...

# 2. 调参（传输、coredump、metrics、分区表、Auth Key 等）
idf.py menuconfig
#   Component config → ESP Insights
#   Example Configuration → WiFi SSID / Password（用 protocol_examples 时）

# 3. 构建（生成 build/<project>-<ver>.zip 固件包供仪表盘上传）
idf.py build

# 4. 擦除并烧录（首次或换分区表后必须 erase_flash）
idf.py -p <serial-port> erase_flash flash monitor
```
启动日志中查找：`Insights enabled for Node ID <XXXXXX>`，用该 Node ID 在仪表盘定位节点。

## Codegen Checklist
- [ ] `main/idf_component.yml` 声明 `espressif/esp_insights` 依赖（registry 版本或 `override_path`）
- [ ] `sdkconfig.defaults` 含 `CONFIG_ESP_INSIGHTS_ENABLED=y` 及所需 metrics/variables 开关
- [ ] HTTPS：`insights_auth_key.txt` 已放入 `main/`；`CMakeLists.txt` 含 `target_add_binary_data(... TEXT)`
- [ ] MQTT：`partitions.csv` 含 `fctry` 分区；已通过 RainMaker CLI claim
- [ ] core dump：`*_COREDUMP_TO_FLASH=y` + `*_COREDUMP_DATA_FORMAT_ELF=y` + `coredump` 分区
- [ ] `partitions.csv` 启用自定义分区表（`CONFIG_PARTITION_TABLE_CUSTOM=y`）
- [ ] `app_main` 顺序：NVS → netif → event loop → 联网 → 时间同步 → `esp_insights_init`
- [ ] `esp_insights_config_t.log_type` 覆盖需要的类型（error/warning/event）
- [ ] 自定义 metrics/variables 先 `register` 再 `report`/`add`；`key` 全局唯一
- [ ] metadata 版本（`ESP_INSIGHTS_META_VERSION_10`）与所选 API（`add_*` vs `report_*`）一致
- [ ] Default log verbosity 未设为 No output（否则 error/warn 不上报）
- [ ] 监听 `ESP_DIAG_DATA_STORE_EVENT_*_LOW_MEM` 以应对 RTC 满载
- [ ] 上传 `build/<project>-<ver>.zip` 到仪表盘 Firmware Images（用于 rodata 交叉引用）

## Do Not Modify
- `D:/esp-skill/espressif-repos/esp-insights/components/**` — 仓库源码（含各组件 `include/` 头文件、`Kconfig`、`src/`）
- 仓库 `examples/` 与 `unit_test_app/` — 仅作参考/拷贝起点，勿原地改
- `SKILL.md` frontmatter — Skill 元数据
- 本 skill 的 `resources/` 文档应与头文件保持一致；如仓库升级，需同步核对 `esp_insights.h` 等真实签名
