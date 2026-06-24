# AGENTS.md — Supplementary Agent Guide

> 核心规则、recipe 索引、陷阱、执行工作流均在 `SKILL.md`。本文件仅补充 `SKILL.md` 未覆盖的约定与工具说明，不重复内容。

## Project Context

- **语言**：C（部分自定义 outbox 示例使用 C++）
- **目标平台**：ESP32 系列 SoC（ESP32, ESP32-C2/C3/C5/C6/C61, ESP32-H2, ESP32-P4, ESP32-S2/S3）
- **工具链 / 构建**：ESP-IDF >= 5.3，`idf.py`，CMake
- **组件来源**：ESP-IDF Component Manager（`espressif/mqtt`）或本地克隆为 `mqtt`

## Code Generation Conventions

### 文件命名
- 应用入口：`main/app_main.c`
- 头文件：`*.h`，源文件：`*.c`
- 自定义 outbox 实现可放 `custom_outbox.c`（C）或 `*.cpp`（C++ 示例）

### Include 顺序（来自仓库示例）
```c
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include "esp_system.h"
#include "nvs_flash.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "protocol_examples_common.h"   // 示例联网辅助
#include "esp_log.h"
#include "mqtt_client.h"                 // 核心头（include/mqtt_client.h）
// MQTT 5 时（CONFIG_MQTT_PROTOCOL_5=y）由 mqtt_client.h 自动 #include "mqtt5_client.h"
```

### 嵌入证书 / 密钥（embed）
```c
// 在 main/CMakeLists.txt：
// idf_component_register(... EMBED_TXTFILES server_ca.pem client.crt client.key)
// 或 EMBED_BINFILES 用于 DER
extern const uint8_t server_cert_pem_start[] asm("_binary_server_ca_pem_start");
extern const uint8_t server_cert_pem_end[]   asm("_binary_server_ca_pem_end");
extern const uint8_t client_cert_pem_start[] asm("_binary_client_crt_start");
extern const uint8_t client_key_pem_start[]  asm("_binary_client_key_start");
```
> 符号名规则：`_binary_<文件名去扩展名、点转下划线>_start/_end`。文件 `server_ca.pem` → `_binary_server_ca_pem_start`。

### 标准 app_main 启动序列（来自 `examples/tcp/main/app_main.c`）
```c
void app_main(void)
{
    ESP_LOGI(TAG, "[APP] Startup..");
    ESP_LOGI(TAG, "[APP] Free memory: %" PRIu32 " bytes", esp_get_free_heap_size());
    ESP_LOGI(TAG, "[APP] IDF version: %s", esp_get_idf_version());
    esp_log_level_set("*", ESP_LOG_INFO);
    esp_log_level_set("mqtt_client", ESP_LOG_VERBOSE);
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect());   // 示例：Wi-Fi 或 Ethernet
    mqtt_app_start();
}
```

### 标准 MQTT 启动模式（mqtt_app_start）
```c
static void mqtt_app_start(void)
{
    esp_mqtt_client_config_t mqtt_cfg = {
        .broker.address.uri = CONFIG_BROKER_URL,   // 或直接字符串
    };
    esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(client);
}
```

### 日志约定
- 模块 TAG：`static const char *TAG = "mqtt_example";`
- 错误辅助（来自示例）：
```c
static void log_error_if_nonzero(const char *message, int error_code)
{
    if (error_code != 0) {
        ESP_LOGE(TAG, "Last error %s: 0x%x", message, error_code);
    }
}
```

### 依赖声明
- `idf_component.yml`（项目级或组件级）：
  ```yaml
  dependencies:
    espressif/mqtt: "*"
    idf:
      version: ">=5.3"
  ```
- `main/CMakeLists.txt`：
  ```cmake
  idf_component_register(SRCS "app_main.c"
                         PRIV_REQUIRES mqtt nvs_flash esp_netif
                         INCLUDE_DIRS ".")
  ```
  TLS 示例常额外 `PRIV_REQUIRES esp_partition app_update`（用于 DS / secure cert）。

## Build Workflow

1. `idf.py set-target esp32`（或对应芯片）
2. `idf.py menuconfig` → `Component config` > `ESP-MQTT Configurations` 调整 Kconfig；`Example Connection Configuration` 配网
3. `idf.py build`
4. `idf.py -p PORT flash monitor`（退出 monitor：`Ctrl-]`）
5. 调试：观察 `MQTT_EVENT_ERROR` 中 `error_handle->error_type` 与 `connect_return_code`；提升 `esp_log_level_set("mqtt_client", ESP_LOG_VERBOSE)`

## MQTT 代码生成 Checklist

- [ ] `idf_component.yml` 声明 `espressif/mqtt`，CMake `PRIV_REQUIRES mqtt`
- [ ] `app_main` 顺序：`nvs_flash_init` → `esp_netif_init` → `esp_event_loop_create_default` → 联网 → `mqtt_app_start`
- [ ] `esp_mqtt_client_config_t` 用指定式初始化填充（`.broker.address.uri` 等）
- [ ] 调用顺序：`init` → `register_event(ESP_EVENT_ANY_ID,...)` → `start`
- [ ] 事件回调覆盖 `MQTT_EVENT_CONNECTED/DISCONNECTED/SUBSCRIBED/UNSUBSCRIBED/PUBLISHED/DATA/ERROR`
- [ ] topic/data 用 `%.*s` + `_len` 打印
- [ ] mqtts:// 必配 `broker.verification`（certificate / use_global_ca_store / crt_bundle_attach / psk_hint_key）
- [ ] 双向认证同时设 `credentials.authentication.certificate` + `.key`
- [ ] MQTT5：`CONFIG_MQTT_PROTOCOL_5=y` + `session.protocol_ver = MQTT_PROTOCOL_V_5`
- [ ] user_property 调用 `esp_mqtt5_client_set_user_property` 后必须 `esp_mqtt5_client_delete_user_property`
- [ ] 不在事件回调中调用 `esp_mqtt_client_stop` / `esp_mqtt_client_destroy`
- [ ] 大消息场景：处理 `current_data_offset` / `total_data_len` 分片
- [ ] 不稳定网络：配置 `outbox.limit` 并处理 publish 返回 `-2`

## Do Not Modify

- `resources/` —— 文档来源，勿改
- `SKILL.md` front matter —— Skill 元数据
- ESP-MQTT 仓库源码（`mqtt_client.c` / `mqtt5_client.c` / `lib/`）—— 如需自定义 outbox，通过 CMake `set_property(TARGET ${mqtt} PROPERTY SOURCES ... APPEND)` 追加，而非改动库本体
