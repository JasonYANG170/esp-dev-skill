# ZCL 属性写本地 + 主动上报（Report Attribute）

> **适用摘要**: 讲解两类属性操作——服务端用 `ezb_zcl_set_attr_value` 更新本地属性值（如温度传感器刷新读数），以及已绑定的客户端用 `ezb_zcl_report_attr_cmd_req` 主动上报当前属性给绑定目的端。

## 触发意图

- "更新 Zigbee 属性值"
- "温度传感器上报"
- "report attribute"
- "set attribute value"
- "ZCL 属性变化通知"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_zigbee.h"`（含 `ezbee/zcl.h`） |
| 参考项目 | `examples/home_automation_devices/temperature_sensor/` |

## 分步说明

### 1. 服务端写本地属性（`ezb_zcl_set_attr_value`）

温度值需转换为 Zigbee 的 `int16_t`（单位 0.01℃）。注意线程安全——加锁包裹。

```c
static int16_t zb_temperature_to_s16(float temp) { return (int16_t)(temp * 100); }

static void esp_app_temp_sensor_handler(float temperature)
{
    int16_t measured_value = zb_temperature_to_s16(temperature);
    esp_zigbee_lock_acquire(portMAX_DELAY);
    ezb_zcl_set_attr_value(ESP_ZIGBEE_HA_TEMPERATURE_SENSOR_EP_ID,
                           EZB_ZCL_CLUSTER_ID_TEMPERATURE_MEASUREMENT,
                           EZB_ZCL_CLUSTER_SERVER,
                           EZB_ZCL_ATTR_TEMPERATURE_MEASUREMENT_MEASURED_VALUE_ID,
                           EZB_ZCL_STD_MANUF_CODE,
                           (uint8_t *)&measured_value, false);
    esp_zigbee_lock_release();
}
```

参数顺序：`ep_id, cluster_id, cluster_role, attr_id, manuf_code, value_ptr, force`。

### 2. 客户端主动上报（`ezb_zcl_report_attr_cmd_req`）

温度传感器在按键时上报当前 measured value。`dst_addr.addr_mode = EZB_ADDR_MODE_NONE` 表示走绑定表（须先 bind）；`fc.direction = EZB_ZCL_CMD_DIRECTION_TO_CLI`（client→server 方向）。

```c
static void button_event_handler(switch_driver_handle_t handle)
{
    ezb_err_t ret = EZB_ERR_NONE;
    ezb_zcl_report_attr_cmd_t report_attr_cmd = {
        .cmd_ctrl = {
            .fc.direction       = EZB_ZCL_CMD_DIRECTION_TO_CLI,
            .dst_addr.addr_mode = EZB_ADDR_MODE_NONE,
            .src_ep             = ESP_ZIGBEE_HA_TEMPERATURE_SENSOR_EP_ID,
            .cluster_id         = EZB_ZCL_CLUSTER_ID_TEMPERATURE_MEASUREMENT,
        },
        .payload = {
            .attr_id = EZB_ZCL_ATTR_TEMPERATURE_MEASUREMENT_MEASURED_VALUE_ID,
        },
    };
    esp_zigbee_lock_acquire(portMAX_DELAY);
    ret = ezb_zcl_report_attr_cmd_req(&report_attr_cmd);
    esp_zigbee_lock_release();
    ESP_LOGI(TAG, "Report %s", ret == EZB_ERR_NONE ? "sent" : "failed");
}
```

### 3. 配置周期性上报（`ezb_zcl_config_report_cmd_req`）

通过配置 report 让服务端在属性变化/超时时自动上报，避免主动轮询。仓库提供 `ezb_zcl_reporting_info_find` / `ezb_zcl_reporting_start_attr_report` / `ezb_zcl_reporting_stop_attr_report` 管理上报句柄。

```c
/* 找到某属性的 reporting info 并启动周期上报（默认最小 5s，最大 0=禁用） */
ezb_zcl_reporting_info_t info = ezb_zcl_reporting_info_find(
    EP_ID, EZB_ZCL_CLUSTER_ID_TEMPERATURE_MEASUREMENT, EZB_ZCL_CLUSTER_SERVER,
    EZB_ZCL_ATTR_TEMPERATURE_MEASUREMENT_MEASURED_VALUE_ID, EZB_ZCL_STD_MANUF_CODE);
if (info != EZB_ZCL_INVALID_REPORTING_INFO) {
    esp_zigbee_lock_acquire(portMAX_DELAY);
    ezb_zcl_reporting_start_attr_report(info);
    esp_zigbee_lock_release();
}
```

### 4. 接收远端上报（ZCL Core Action）

在 `ezb_zcl_core_action_handler_register` 注册的回调里处理 `EZB_ZCL_CORE_REPORT_ATTR_CB_ID`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ezb_zcl_set_attr_value` 返回错误 | cluster_role/attr_id 不存在 | 确认该 cluster 已在 endpoint 注册且 attr_id 合法 |
| 温度上报对端收不到 | 未绑定 / 方向错 | 先 ZDO bind；`fc.direction = EZB_ZCL_CMD_DIRECTION_TO_CLI` |
| 数值单位不对 | 直接传浮点 | 必须转 `int16_t`（×100） |
| 周期上报不生效 | 未 start | 用 `ezb_zcl_reporting_start_attr_report` 启动 |
| 上报过频被丢弃 | 小于最小间隔 | 默认最小 `EZB_ZCL_MIN_REPORTING_INTERVAL_DEFAULT`(5s) |

## 参考项目

- `examples/home_automation_devices/temperature_sensor/main/temperature_sensor.c` — set_attr_value + report_attr_cmd
- `docs/en/user-guide/zcl_general_report.rst` — 上报机制说明
- `components/esp-zigbee-lib/include/ezbee/zcl/zcl_reporting.h` — reporting 句柄 API
