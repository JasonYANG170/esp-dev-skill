# 构建 ZHA 数据模型：device / endpoint / cluster / Basic 属性

> **适用摘要**: 讲解 ESP Zigbee SDK v2.x 的 ZHA 数据模型构建流程——device descriptor → endpoint descriptor（由 `ezb_zha_create_*` 一次性创建含 cluster）→ 补充 Basic 厂商/型号属性 → 注册到协议栈。这是所有 ZHA 设备共用的骨架。

> Evidence: `repos/esp-zigbee-sdk/resources/`, source/examples in `repos/esp-zigbee-sdk/`, and this recipe path `repos/esp-zigbee-sdk/recipes/zha_device_model.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "怎么创建一个 Zigbee 设备"
- "ZHA 数据模型"
- "添加 endpoint / cluster"
- "设置 ManufacturerName / ModelIdentifier"
- "on_off_light / temperature_sensor / thermostat 数据模型"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_zigbee.h"` + `#include "ezbee/zha.h"` |
| 参考项目 | 任一 `examples/home_automation_devices/<dev>/main/<dev>.c` |

## 分步说明

### 1. 数据模型概念（来自 `docs/en/developing.rst`）

```
Node (一个 ESP32 节点)
 └── Endpoint (1..240)
      └── Cluster (16-bit ID, input/output, server/client)
           └── Attribute (16-bit ID, 存状态) + Command (动作)
```

- 一个节点可暴露多个 endpoint（不同应用/设备类型）
- `ezb_zha_create_<device>(ep_id, &cfg)` 会按 HA 规范自动装配该设备的标准 server cluster（如 on/off light 自动含 basic、identify、groups、scenes、on_off）

### 2. 标准四步创建（以 on/off light 为例）

```c
esp_err_t esp_zigbee_create_zha_on_off_light_device(void)
{
    /* 1. 创建 device descriptor */
    ezb_af_device_desc_t dev_desc = ezb_af_create_device_desc();

    /* 2. 用 ZHA 宏创建 endpoint（自带标准 cluster） */
    ezb_zha_on_off_light_config_t light_cfg = EZB_ZHA_ON_OFF_LIGHT_CONFIG();
    ezb_af_ep_desc_t ep_desc = ezb_zha_create_on_off_light(ESP_ZIGBEE_HA_ON_OFF_LIGHT_EP_ID, &light_cfg);

    /* 3. 可选：补 Basic cluster 的厂商/型号属性（zigbee 字符串首字节为长度） */
    ezb_zcl_cluster_desc_t basic_desc =
        ezb_af_endpoint_get_cluster_desc(ep_desc, EZB_ZCL_CLUSTER_ID_BASIC, EZB_ZCL_CLUSTER_SERVER);
    ezb_zcl_basic_cluster_desc_add_attr(basic_desc, EZB_ZCL_ATTR_BASIC_MANUFACTURER_NAME_ID, (void *)ESP_MANUFACTURER_NAME);
    ezb_zcl_basic_cluster_desc_add_attr(basic_desc, EZB_ZCL_ATTR_BASIC_MODEL_IDENTIFIER_ID,  (void *)ESP_MODEL_IDENTIFIER);

    /* 4. 挂到 device 并注册到协议栈 */
    ESP_ERROR_CHECK(ezb_af_device_add_endpoint_desc(dev_desc, ep_desc));
    ESP_ERROR_CHECK(ezb_af_device_desc_register(dev_desc));

    /* 可选：注册 ZCL core action 回调 */
    ezb_zcl_core_action_handler_register(esp_zigbee_zcl_core_action_handler);
    return ESP_OK;
}
```

### 3. 可用的 ZHA 创建函数（`ezbee/zha.h`，节选）

| 函数 | 设备类型 ID | 默认 server cluster |
|---|---|---|
| `ezb_zha_create_on_off_light` | `EZB_ZHA_ON_OFF_LIGHT_DEVICE_ID`(0x0100) | basic, identify, groups, scenes, on_off |
| `ezb_zha_create_dimmable_light` | 0x0101 | + level_control |
| `ezb_zha_create_color_dimmable_light` | 0x0102 | + color_control |
| `ezb_zha_create_on_off_switch` | 0x0000 | basic, identify（client 侧） |
| `ezb_zha_create_temperature_sensor` | 0x0302 | + temperature_measurement |
| `ezb_zha_create_light_sensor` | 0x0106 | + illuminance_measurement |
| `ezb_zha_create_thermostat` | 0x0301 | + thermostat |
| `ezb_zha_create_door_lock` | 0x000A | + door_lock |
| `ezb_zha_create_window_covering` | 0x0202 | + window_covering |
| `ezb_zha_create_shade` | 0x0200 | + shade_config/on_off/level |
| `ezb_zha_create_mains_power_outlet` | 0x0009 | + groups/scenes/on_off |
| `ezb_zha_create_configuration_tool` | 0x0005 | basic, identify |
| `ezb_zha_create_custom_gateway` | 0xFF00 | basic, identify |

### 4. 配置宏覆盖默认值（温度传感器示例）

```c
ezb_zha_temperature_sensor_config_t cfg = EZB_ZHA_TEMPERATURE_SENSOR_CONFIG();
cfg.temp_meas_cfg.min_measured_value = zb_temperature_to_s16(-10); // 0.01℃/单位
cfg.temp_meas_cfg.max_measured_value = zb_temperature_to_s16(80);
ezb_af_ep_desc_t ep = ezb_zha_create_temperature_sensor(EP_ID, &cfg);
```

### 5. 多 endpoint：一个节点多种设备

对同一 `dev_desc` 多次 `ezb_zha_create_*` + `ezb_af_device_add_endpoint_desc`，最后一次性 `ezb_af_device_desc_register`。参考 `examples/all_device_types_app/`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ep_desc` 为 NULL | `ezb_zha_create_*` 失败（内存/参数） | 检查 `cfg` 初始化与 `ep_id` 范围 1..240 |
| 远端读不到 cluster | 漏 `ezb_af_device_desc_register` | 四步必须全部执行 |
| ManufacturerName 显示乱码 | 字符串未带长度前缀 | 用 `"\x09""ESPRESSIF"` 形式（首字节=长度） |
| 想加非标准 cluster | `ezb_zha_create_*` 不含它 | 用 `ezb_af_endpoint_add_cluster_desc` 手动加，或做自定义 cluster（见 `custom_cluster.md`） |
| Basic 属性加不上 | 取 cluster_desc 时角色写错 | server 角色用 `EZB_ZCL_CLUSTER_SERVER` |

## 参考项目

- `examples/home_automation_devices/on_off_light/main/on_off_light.c` — on/off light 数据模型
- `examples/home_automation_devices/temperature_sensor/main/temperature_sensor.c` — 传感器 + 默认值覆盖
- `examples/home_automation_devices/thermostat/main/thermostat.c` — thermostat
- `examples/all_device_types_app/main/all_device_types_app.c` — 多 endpoint
