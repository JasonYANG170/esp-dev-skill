# 协调器（ZC）on/off 灯：网络形成与开放入网

> **适用摘要**: 实现一个 Zigbee Coordinator（ZC）角色的 HA on/off 灯，完成网络形成（FORMATION）、开放网络（open network）与入网引导（STEERING），并通过 ZCL Core Action 接收 on/off 属性变化驱动 LED。

## 触发意图

- "做一个 Zigbee 协调器"
- "ZC 形成网络"
- "on/off light 协调器示例"
- "Zigbee 开放入网 / open network"
- "ESP32-H2 做网关/中心设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP32-H2-DevKitM-1 或 ESP32-C6-DevKitM-1（native 802.15.4） |
| 软件 | ESP-IDF v5.2+（示例对齐 v5.5.4），`espressif/esp-zigbee-lib >=2.0.0` |
| 参考项目 | `examples/home_automation_devices/on_off_light/` |
| 分区 | `partitions.csv` 含 `zb_storage`(16K) 与 `zb_fct`(1K) |

## 分步说明

### 1. 设备配置宏（`main/on_off_light.h`，源自示例）

ZC 角色用 `EZB_NWK_DEVICE_TYPE_COORDINATOR`，`max_children` 控制可挂载子设备数。

```c
#pragma once

#define ESP_ZIGBEE_PRIMARY_CHANNEL_MASK   ((1U << 13))
#define ESP_ZIGBEE_SECONDARY_CHANNEL_MASK (0x07FFF800U)
#define ESP_ZIGBEE_HA_ON_OFF_LIGHT_EP_ID  (10)
#define ESP_ZIGBEE_STORAGE_PARTITION_NAME "zb_storage"
#define ESP_MANUFACTURER_NAME             "\x09""ESPRESSIF"
#define ESP_MODEL_IDENTIFIER              "\x07"CONFIG_IDF_TARGET

#define ESP_ZIGBEE_ZC_CONFIG()                          \
    {                                                   \
        .device_type = EZB_NWK_DEVICE_TYPE_COORDINATOR, \
        .install_code_policy = false,                   \
        .zczr_config = { .max_children = 10, },         \
    }

#if CONFIG_SOC_IEEE802154_SUPPORTED
#define ESP_ZIGBEE_PLATFORM_CONFIG()                                 \
    {                                                                \
        .storage_partition_name = ESP_ZIGBEE_STORAGE_PARTITION_NAME, \
        .radio_config = { .radio_mode = ESP_ZIGBEE_RADIO_MODE_NATIVE }, \
    }
#else
#warning "非 15.4 SoC 请参考 zigbee_gateway RCP 配置"
#endif

#define ESP_ZIGBEE_DEFAULT_CONFIG()                      \
    {                                                    \
        .device_config = ESP_ZIGBEE_ZC_CONFIG(),         \
        .platform_config = ESP_ZIGBEE_PLATFORM_CONFIG(), \
    };
```

### 2. ZHA on/off 灯数据模型

```c
#include "esp_zigbee.h"
#include "ezbee/zha.h"
#include "on_off_light.h"

esp_err_t esp_zigbee_create_zha_on_off_light_device(void)
{
    ezb_af_device_desc_t          dev_desc  = ezb_af_create_device_desc();
    ezb_zha_on_off_light_config_t light_cfg = EZB_ZHA_ON_OFF_LIGHT_CONFIG();
    ezb_af_ep_desc_t              ep_desc   = ezb_zha_create_on_off_light(ESP_ZIGBEE_HA_ON_OFF_LIGHT_EP_ID, &light_cfg);
    ezb_zcl_cluster_desc_t        basic_desc = {0};

    basic_desc = ezb_af_endpoint_get_cluster_desc(ep_desc, EZB_ZCL_CLUSTER_ID_BASIC, EZB_ZCL_CLUSTER_SERVER);
    ezb_zcl_basic_cluster_desc_add_attr(basic_desc, EZB_ZCL_ATTR_BASIC_MANUFACTURER_NAME_ID, (void *)ESP_MANUFACTURER_NAME);
    ezb_zcl_basic_cluster_desc_add_attr(basic_desc, EZB_ZCL_ATTR_BASIC_MODEL_IDENTIFIER_ID,  (void *)ESP_MODEL_IDENTIFIER);

    ESP_ERROR_CHECK(ezb_af_device_add_endpoint_desc(dev_desc, ep_desc));
    ESP_ERROR_CHECK(ezb_af_device_desc_register(dev_desc));
    ezb_zcl_core_action_handler_register(esp_zigbee_zcl_core_action_handler);
    return ESP_OK;
}
```

### 3. ZCL Core Action：收 on/off 属性变化驱动灯

```c
static void zcl_core_set_attr_value_handler(ezb_zcl_set_attr_value_message_t *message)
{
    ESP_RETURN_ON_FALSE(message, , TAG, "message is empty");
    if (message->info.cluster_id == EZB_ZCL_CLUSTER_ID_ON_OFF) {
        light_driver_set_power(*(uint8_t *)message->in.attribute.data.value);
        ESP_LOGI(TAG, "Set On/Off: %d", *(uint8_t *)message->in.attribute.data.value);
    }
}

static void esp_zigbee_zcl_core_action_handler(ezb_zcl_core_action_callback_id_t callback_id, void *message)
{
    switch (callback_id) {
    case EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID:
        zcl_core_set_attr_value_handler(message);
        break;
    case EZB_ZCL_CORE_DEFAULT_RSP_CB_ID: {
        ezb_zcl_cmd_default_rsp_message_t *rsp = (ezb_zcl_cmd_default_rsp_message_t *)message;
        ESP_LOGI(TAG, "Default Response status(0x%02x)", rsp->in.status_code);
    } break;
    default:
        ESP_LOGW(TAG, "ZCL Core Action: ID(0x%04lx)", callback_id);
        break;
    }
}
```

### 4. Signal Handler：FORMATION → STEERING（ZC 关键分支）

```c
static void esp_zigbee_alarm_bdb_commissioning(alarm_timer_arg_t arg)
{
    esp_zigbee_lock_acquire(portMAX_DELAY);
    (void)ezb_bdb_start_top_level_commissioning(arg);
    esp_zigbee_lock_release();
}

static bool esp_zigbee_app_signal_handler(const ezb_app_signal_t *app_signal)
{
    ezb_app_signal_type_t signal_type = ezb_app_signal_get_type(app_signal);
    switch (signal_type) {
    case EZB_ZDO_SIGNAL_SKIP_STARTUP:
        ESP_LOGI(TAG, "Initialize Zigbee stack");
        ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_INITIALIZATION);
        break;
    case EZB_BDB_SIGNAL_DEVICE_FIRST_START:
    case EZB_BDB_SIGNAL_DEVICE_REBOOT: {
        ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
        if (status == EZB_BDB_STATUS_SUCCESS) {
            deferred_driver_init();
            if (ezb_bdb_is_factory_new()) {
                ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_NETWORK_FORMATION); // ZC 形成网络
            } else {
                ezb_bdb_open_network(180);   // 非首次启动：开放 180 秒
            }
        } else {
            alarm_timer_schedule(esp_zigbee_alarm_bdb_commissioning, EZB_BDB_MODE_INITIALIZATION, 1000);
        }
    } break;
    case EZB_BDB_SIGNAL_FORMATION: {
        ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
        if (status == EZB_BDB_STATUS_SUCCESS) {
            ESP_LOGI(TAG, "Formed: PAN(0x%04hx) Channel(%d) Short(0x%04hx)",
                     ezb_nwk_get_panid(), ezb_nwk_get_current_channel(), ezb_nwk_get_short_address());
            ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_NETWORK_STEERING); // 形成后引导
        } else {
            alarm_timer_schedule(esp_zigbee_alarm_bdb_commissioning, EZB_BDB_MODE_NETWORK_FORMATION, 1000);
        }
    } break;
    case EZB_BDB_SIGNAL_STEERING: {
        ezb_bdb_comm_status_t status = *((ezb_bdb_comm_status_t *)ezb_app_signal_get_params(app_signal));
        ESP_LOGI(TAG, "Network steering %s", status == EZB_BDB_STATUS_SUCCESS ? "completed" : "failed");
    } break;
    case EZB_ZDO_SIGNAL_DEVICE_ANNCE: {
        const ezb_zdo_signal_device_annce_params_t *p = ezb_app_signal_get_params(app_signal);
        ESP_LOGI(TAG, "New device joined(0x%04hx)", p->short_addr);
    } break;
    default:
        ESP_LOGI(TAG, "Signal: %s(0x%02x)", ezb_app_signal_to_string(signal_type), signal_type);
        break;
    }
    return true;
}
```

### 5. setup + 主任务 + app_main

```c
esp_err_t esp_zigbee_setup_commissioning(void)
{
    ezb_aps_secur_enable_distributed_security(false);
    ESP_ERROR_CHECK(ezb_bdb_set_primary_channel_set(ESP_ZIGBEE_PRIMARY_CHANNEL_MASK));
    ESP_ERROR_CHECK(ezb_bdb_set_secondary_channel_set(ESP_ZIGBEE_SECONDARY_CHANNEL_MASK));
    ESP_ERROR_CHECK(ezb_app_signal_add_handler(esp_zigbee_app_signal_handler));
    return ESP_OK;
}

static void esp_zigbee_stack_main_task(void *pvParameters)
{
    esp_zigbee_config_t config = ESP_ZIGBEE_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(esp_zigbee_init(&config));
    ESP_ERROR_CHECK(esp_zigbee_setup_commissioning());
    ESP_ERROR_CHECK(esp_zigbee_create_zha_on_off_light_device());
    ESP_ERROR_CHECK(esp_zigbee_start(false));   // no-autostart
    esp_zigbee_launch_mainloop();
    esp_zigbee_deinit();
    vTaskDelete(NULL);
}

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(nvs_flash_init_partition(ESP_ZIGBEE_STORAGE_PARTITION_NAME));
    xTaskCreate(esp_zigbee_stack_main_task, "Zigbee_main", 4096, NULL, 5, NULL);
}
```

### 6. 编译烧录

```bash
idf.py set-target esp32h2
idf.py -p PORT erase_flash flash monitor
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `EZB_BDB_STATUS_NOT_PERMITTED` | 角色非 ZC/ZR 却用 FORMATION | 确认 `device_type = EZB_NWK_DEVICE_TYPE_COORDINATOR` |
| 一直停留在 SKIP_STARTUP | 没在 FIRST_START 启动 FORMATION | 检查 factory-new 分支调用 FORMATION |
| 子设备入不了网 | 网络未开放或时长过短 | 用 `ezb_bdb_open_network(180)` 开放足够秒数 |
| 断言/崩溃 | 主任务栈不足 | 栈设为 4096；`uxTaskGetStackHighWaterMark` 监控 |
| 找不到 `esp_zigbee.h` | 未声明依赖 | `idf_component.yml` 加 `espressif/esp-zigbee-lib >=2.0.0` |
| 抓包看不到 payload | 未配置密钥 | Wireshark 加 key `ZigbeeAlliance09` |

## 参考项目

- `examples/home_automation_devices/on_off_light/` — ZC on/off 灯完整示例
- `examples/home_automation_devices/color_dimmable_light/` — ZC 彩光灯（含 level + color_control）
- `examples/all_device_types_app/` — 多设备类型组合
