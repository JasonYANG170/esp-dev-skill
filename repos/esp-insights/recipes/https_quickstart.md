# HTTPS 传输快速接入

> **适用摘要**: 用最少的代码把 ESP-Insights 通过 HTTPS 接入 ESP Insights 云端，采集错误/告警/事件日志。这是大多数新项目的默认起点。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-insights/resources/`, source/examples in `repos/esp-insights/`, and this recipe path `repos/esp-insights/recipes/https_quickstart.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "接入 ESP-Insights"
- "HTTPS 上报诊断"
- "获取 Auth Key"
- "最小可用 Insights 示例"
- "esp_insights_init 怎么用"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | `>= release/v5.1`（推荐 v5.5），已 `idf.py set-target <chip>` |
| 网络 | 设备能连互联网（Wi-Fi 或以太网） |
| 账号 | 已在 https://dashboard.insights.espressif.com 注册并登录 |
| Auth Key | Dashboard → Manage Auth Keys → 生成并下载 Auth Key 文本 |
| 参考工程 | `examples/minimal_diagnostics` |

## 分步说明

### 1. 拷贝最小工程作为起点

```bash
cp -r path/to/esp-insights/examples/minimal_diagnostics my_app
cd my_app
idf.py set-target esp32   # 或 esp32s3/esp32c3/...
```

### 2. 放入 Auth Key

把仪表盘下载的 Auth Key 内容写入 `main/insights_auth_key.txt`（文件名与 `main/CMakeLists.txt`、源码 `asm` 符号绑定）：

```bash
cp /path/to/downloaded/key.txt main/insights_auth_key.txt
```

`main/CMakeLists.txt`（来自 `examples/minimal_diagnostics/main/CMakeLists.txt`，已正确嵌入）：
```cmake
idf_component_register(SRCS "app_main.c"
                       INCLUDE_DIRS ".")
if (CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS)
    target_add_binary_data(${COMPONENT_TARGET} "insights_auth_key.txt" TEXT)
endif()
```

### 3. sdkconfig.defaults 关键项（来自 `examples/minimal_diagnostics/sdkconfig.defaults`）

```
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y

CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_OFFSET=0x8000
CONFIG_PARTITION_TABLE_MD5=y

CONFIG_ESP_INSIGHTS_ENABLED=y
CONFIG_ESP_INSIGHTS_META_VERSION_10=n     # 启用 metadata 2.0（推荐）

CONFIG_ESP32_ENABLE_COREDUMP=y
CONFIG_ESP32_ENABLE_COREDUMP_TO_FLASH=y
CONFIG_ESP32_COREDUMP_DATA_FORMAT_ELF=y
CONFIG_ESP32_COREDUMP_CHECKSUM_CRC32=y
CONFIG_ESP32_CORE_DUMP_MAX_TASKS_NUM=64
CONFIG_ESP32_CORE_DUMP_STACK_SIZE=1024

CONFIG_MBEDTLS_DYNAMIC_BUFFER=y
CONFIG_MBEDTLS_DYNAMIC_FREE_PEER_CERT=y
CONFIG_MBEDTLS_DYNAMIC_FREE_CONFIG_DATA=y

CONFIG_DIAG_ENABLE_METRICS=y
CONFIG_DIAG_ENABLE_HEAP_METRICS=y
CONFIG_DIAG_ENABLE_WIFI_METRICS=y
CONFIG_DIAG_ENABLE_VARIABLES=y
CONFIG_DIAG_ENABLE_NETWORK_VARIABLES=y
```

### 4. app_main 核心代码（来自 `examples/minimal_diagnostics/main/app_main.c`）

```c
#include "esp_log.h"
#include "esp_netif.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "protocol_examples_common.h"
#include "esp_insights.h"
#include "esp_diagnostics.h"
#include "esp_rmaker_utils.h"

#ifdef CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS
extern const char insights_auth_key_start[] asm("_binary_insights_auth_key_txt_start");
#endif

static const char *TAG = "my_app";

void app_main(void)
{
    /* NVS */
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    /* 网络 + 默认事件循环 */
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());

    /* 时间同步：未同步用相对启动时间(µs)，同步后用 epoch */
    esp_rmaker_time_sync_init(NULL);

    /* ESP-Insights：HTTPS 默认传输，log_type 覆盖 error/warning/event */
    esp_insights_config_t config = {
        .log_type = ESP_DIAG_LOG_TYPE_ERROR | ESP_DIAG_LOG_TYPE_WARNING | ESP_DIAG_LOG_TYPE_EVENT,
#ifdef CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS
        .auth_key = insights_auth_key_start,
#endif
    };
    ret = esp_insights_init(&config);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to init ESP Insights, err:0x%x", ret);
    }
    ESP_ERROR_CHECK(ret);

    /* 之后正常写日志/事件即可被采集 */
    ESP_LOGE(TAG, "this is a test error, will appear on dashboard");
    ESP_LOGW(TAG, "this is a test warning");
    ESP_DIAG_EVENT(TAG, "this is a custom event: boot=%" PRIu32, esp_log_timestamp());
}
```

### 5. 构建、烧录、监控

```bash
idf.py build
idf.py -p <serial-port> erase_flash flash monitor
```
启动日志中找：
```
I (xxxx) esp_insights: Insights enabled for Node ID XXXXXXXXXXXX
```

### 6. 仪表盘查看

1. 登录 https://dashboard.insights.espressif.com
2. 上传固件包 `build/minimal_diagnostics-v1.0.zip`（Dashboard → Firmware Images），用于 rodata 字符串交叉引用
3. Nodes → 用启动日志里的 Node ID 进入，查看错误/告警/事件、重启原因、metrics、variables

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 启动报错 `Failed to init ESP Insights` 且 auth 相关 | `auth_key` 为空或文件未嵌入 | 确认 `main/insights_auth_key.txt` 存在，且 `CMakeLists.txt` 含 `target_add_binary_data(... TEXT)` |
| 仪表盘看不到任何日志 | `Default log verbosity` 被设为 No output | 改为 >= Warning，或把 Console channel 设为 None |
| 仪表盘显示 Firmware Image missing | 未上传固件包 / 包与板上不一致 | 重新 `idf.py build` 后上传 `build/<project>-<ver>.zip` |
| 日志出现 "gaps" 或丢失 | RTC 存储满后丢弃 | 减少日志量，或调大 `CONFIG_RTC_STORE_CRITICAL_DATA_SIZE` |
| `ESP_DIAG_EVENT` 不显示 | `log_type` 未含 `EVENT` | config 加 `\| ESP_DIAG_LOG_TYPE_EVENT` |

## 参考

- `examples/minimal_diagnostics/` — 最小 HTTPS 示例（本配方主参考）
- `examples/minimal_diagnostics/main/app_main.c` — 入口源码
- `examples/minimal_diagnostics/sdkconfig.defaults` — 完整配置
- `components/esp_insights/include/esp_insights.h` — `esp_insights_init` / `esp_insights_config_t`
- `components/esp_diagnostics/include/esp_diagnostics.h` — `ESP_DIAG_LOG_TYPE_*` / `ESP_DIAG_EVENT`
