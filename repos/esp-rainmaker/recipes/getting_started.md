# 从零创建 RainMaker 节点

> **适用摘要**: 从零搭建一个 ESP RainMaker 节点工程，完成 NVS、网络初始化、节点创建、设备挂载、服务启用与 Agent 启动的完整流程。这是所有 RainMaker 应用的骨架。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-rainmaker/resources/`, source/examples in `repos/esp-rainmaker/`, and this recipe path `repos/esp-rainmaker/recipes/getting_started.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "创建 RainMaker 节点"
- "新建 RainMaker 工程"
- "RainMaker 入门"
- "esp_rmaker_node_init 怎么用"
- "RainMaker 初始化顺序"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.1 或更高 |
| 组件依赖 | `esp_rainmaker`（见 `idf_component.yml`） |
| 参考示例 | `examples/switch/main/app_main.c` |

## 分步说明

### 1. 工程依赖（`main/idf_component.yml`）

```yaml
dependencies:
  esp_rmaker:
    version: "*"
```

### 2. 包含头文件

```c
#include <string.h>
#include <freertos/FreeRTOS.h>
#include <freertos/task.h>
#include <esp_log.h>
#include <esp_event.h>
#include <nvs_flash.h>

#include <esp_rmaker_core.h>
#include <esp_rmaker_standard_types.h>
#include <esp_rmaker_standard_params.h>
#include <esp_rmaker_standard_devices.h>
#include <esp_rmaker_ota.h>
#include <esp_rmaker_schedule.h>
#include <esp_rmaker_scenes.h>
#include <esp_rmaker_console.h>
#include <esp_rmaker_common_events.h>

#include <app_network.h>
#include <app_insights.h>
```

### 3. write 回调（处理云端下发）

```c
static const char *TAG = "app_main";

static esp_err_t write_cb(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
            const esp_rmaker_param_val_t val, void *priv_data, esp_rmaker_write_ctx_t *ctx)
{
    if (ctx) {
        ESP_LOGI(TAG, "Received write request via : %s", esp_rmaker_device_cb_src_to_str(ctx->src));
    }
    if (strcmp(esp_rmaker_param_get_name(param), ESP_RMAKER_DEF_POWER_NAME) == 0) {
        ESP_LOGI(TAG, "Received value = %s for %s",
                val.val.b ? "true" : "false", esp_rmaker_param_get_name(param));
        app_driver_set_state(val.val.b);
        esp_rmaker_param_update(param, val);
    }
    return ESP_OK;
}
```

### 4. 事件处理（监听 RainMaker 事件）

```c
static void event_handler(void *arg, esp_event_base_t event_base,
                          int32_t event_id, void *event_data)
{
    if (event_base == RMAKER_EVENT) {
        switch (event_id) {
            case RMAKER_EVENT_INIT_DONE:
                ESP_LOGI(TAG, "RainMaker Initialised.");
                break;
            case RMAKER_EVENT_CLAIM_STARTED:
                ESP_LOGI(TAG, "RainMaker Claim Started.");
                break;
            case RMAKER_EVENT_CLAIM_SUCCESSFUL:
                ESP_LOGI(TAG, "RainMaker Claim Successful.");
                break;
            case RMAKER_EVENT_CLAIM_FAILED:
                ESP_LOGI(TAG, "RainMaker Claim Failed.");
                break;
            case RMAKER_EVENT_STARTED:
                ESP_LOGI(TAG, "RainMaker Started.");
                break;
            default:
                break;
        }
    } else if (event_base == RMAKER_COMMON_EVENT) {
        switch (event_id) {
            case RMAKER_MQTT_EVENT_CONNECTED:
                ESP_LOGI(TAG, "MQTT Connected.");
                break;
            case RMAKER_MQTT_EVENT_DISCONNECTED:
                ESP_LOGI(TAG, "MQTT Disconnected.");
                break;
            default:
                break;
        }
    }
}
```

### 5. app_main() 完整骨架

```c
void app_main(void)
{
    esp_rmaker_console_init();
    app_driver_init();

    /* NVS 初始化（处理无空闲页情况） */
    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        err = nvs_flash_init();
    }
    ESP_ERROR_CHECK(err);

    /* 网络初始化（必须在 node_init 之前） */
    app_network_init();

    /* 注册事件 */
    ESP_ERROR_CHECK(esp_event_handler_register(RMAKER_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(RMAKER_COMMON_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));

    /* 初始化节点（第一个 RainMaker API） */
    esp_rmaker_config_t rainmaker_cfg = { .enable_time_sync = false };
    esp_rmaker_node_t *node = esp_rmaker_node_init(&rainmaker_cfg, "ESP RainMaker Device", "Switch");
    if (!node) {
        ESP_LOGE(TAG, "Could not initialise node. Aborting!!!");
        vTaskDelay(5000 / portTICK_PERIOD_MS);
        abort();
    }

    /* 创建标准 Switch 设备 */
    esp_rmaker_device_t *switch_device = esp_rmaker_device_create("Switch", ESP_RMAKER_DEVICE_SWITCH, NULL);
    esp_rmaker_device_add_cb(switch_device, write_cb, NULL);
    esp_rmaker_device_add_param(switch_device,
            esp_rmaker_name_param_create(ESP_RMAKER_DEF_NAME_PARAM, "Switch"));
    esp_rmaker_param_t *power_param = esp_rmaker_power_param_create(ESP_RMAKER_DEF_POWER_NAME, false);
    esp_rmaker_device_add_param(switch_device, power_param);
    esp_rmaker_device_assign_primary_param(switch_device, power_param);
    esp_rmaker_node_add_device(node, switch_device);

    /* 启用服务（必须在 esp_rmaker_start 之前） */
    esp_rmaker_ota_enable_default();
    esp_rmaker_timezone_service_enable();
    esp_rmaker_schedule_enable();
    esp_rmaker_scenes_enable();
    app_insights_enable();

    /* 启动 Agent */
    esp_rmaker_start();

    /* 启动网络连接（在 esp_rmaker_start 之后） */
    err = app_network_start(POP_TYPE_RANDOM);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "Could not start Wifi. Aborting!!!");
        abort();
    }
}
```

### 6. 编译烧录

```bash
idf.py set-target esp32s3
idf.py build
idf.py flash monitor
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_rmaker_node_init` 返回 NULL | NVS 未初始化或内存不足 | 确保 `nvs_flash_init()` 在前，必要时 `abort()` 重试 |
| 节点连不上云 | Claiming 未完成或芯片不支持所选 Claim 类型 | ESP32/ESP32-C2 不能 Self Claim；改 Assisted Claim |
| 设备不在 App 里显示 | 设备未 `esp_rmaker_node_add_device()` | 创建后必须挂到节点 |
| App 上 Power 不更新 | write 回调未 `esp_rmaker_param_update()` | 回调内回写参数 |
| `undefined reference to app_network_*` | 未链接 `rmaker_app_network` 组件 | `main/idf_component.yml` 加依赖，或用 examples/common |

## 参考

- `examples/switch/main/app_main.c` — 标准 Switch 节点完整示例
- `examples/multi_device/main/app_main.c` — 多设备节点
- `resources/api_reference.md` — Node/Device/Param API
- `recipes/claiming_and_provisioning.md` — Claiming 与配网
