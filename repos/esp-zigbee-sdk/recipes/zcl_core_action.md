# 接收属性写入：ZCL Core Action 回调

> **适用摘要**: 讲解服务端如何接收并处理来自远端的属性写入/命令——通过 `ezb_zcl_core_action_handler_register` 注册回调，在 `EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID` 分支里取 `ezb_zcl_set_attr_value_message_t` 驱动外设（如点灯、关窗帘）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-zigbee-sdk/resources/`, source/examples in `repos/esp-zigbee-sdk/`, and this recipe path `repos/esp-zigbee-sdk/recipes/zcl_core_action.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "灯怎么响应开关命令"
- "Zigbee 接收属性变化"
- "set attribute value 回调"
- "处理 ZCL 命令"
- "ezb_zcl_core_action_handler"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_zigbee.h"`（含 `ezbee/zcl/zcl_core.h`） |
| 参考项目 | `examples/home_automation_devices/on_off_light/main/on_off_light.c` |

## 分步说明

### 1. 回调签名与注册

回调类型为 `ezb_zcl_core_action_callback_t`，接收 `(callback_id, message)`。在创建数据模型后注册。

```c
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
        ESP_LOGW(TAG, "Unhandled ZCL Core Action: ID(0x%04lx)", callback_id);
        break;
    }
}

// 在 esp_zigbee_create_*_device() 末尾：
ezb_zcl_core_action_handler_register(esp_zigbee_zcl_core_action_handler);
```

### 2. 处理 SET_ATTR_VALUE（点灯示例）

```c
static void zcl_core_set_attr_value_handler(ezb_zcl_set_attr_value_message_t *message)
{
    ESP_RETURN_ON_FALSE(message, , TAG, "message is empty");
    ESP_LOGI(TAG, "SetAttr ep(%d) cluster(0x%04x) %s status(0x%02x)",
             message->info.dst_ep, message->info.cluster_id,
             message->info.cluster_role == EZB_ZCL_CLUSTER_SERVER ? "server" : "client",
             message->info.status);

    if (message->info.cluster_id == EZB_ZCL_CLUSTER_ID_ON_OFF) {
        uint8_t on_off = *(uint8_t *)message->in.attribute.data.value;
        light_driver_set_power(on_off);
        ESP_LOGI(TAG, "Set On/Off: %d", on_off);
    } else {
        ESP_LOGW(TAG, "Unsupported cluster ID(0x%04x)", message->info.cluster_id);
    }
}
```

关键字段：
- `message->info.dst_ep` —— 目的 endpoint
- `message->info.cluster_id` —— cluster（如 `EZB_ZCL_CLUSTER_ID_ON_OFF`、`EZB_ZCL_CLUSTER_ID_LEVEL_CONTROL`、`EZB_ZCL_CLUSTER_ID_COLOR_CONTROL`）
- `message->info.cluster_role` —— `EZB_ZCL_CLUSTER_SERVER` / `EZB_ZCL_CLUSTER_CLIENT`
- `message->in.attribute.data.value` —— 新属性值指针

### 3. 其他常用 callback_id（`ezbee/zcl/zcl_core.h`，节选）

| callback_id | 用途 |
|---|---|
| `EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID` | 属性被写（主入口） |
| `EZB_ZCL_CORE_REPORT_ATTR_CB_ID` | 收到上报属性 |
| `EZB_ZCL_CORE_READ_ATTR_RSP_CB_ID` | 读属性响应 |
| `EZB_ZCL_CORE_DEFAULT_RSP_CB_ID` | 默认响应 |
| `EZB_ZCL_CORE_IDENTIFY_EFFECT_CB_ID` | identify 效果 |
| `EZB_ZCL_CORE_BASIC_RESET_TO_FACTORY_DEFAULT_CB_ID` | basic 恢复出厂 |
| `EZB_ZCL_CORE_ON_OFF_OFF_WITH_EFFECT_CB_ID` | 带效果关灯 |
| `EZB_ZCL_CORE_DOOR_LOCK_LOCK_DOOR_CB_ID` / `UNLOCK_DOOR_CB_ID` | 门锁开关 |
| `EZB_ZCL_CORE_WINDOW_COVERING_MOVEMENT_CB_ID` | 窗帘运动 |
| `EZB_ZCL_CORE_COLOR_CONTROL_COLOR_MODE_CHANGE_CB_ID` | 颜色模式切换 |
| `EZB_ZCL_CORE_MANUF_SPEC_CMD_CB_ID` | 厂商自定义命令 |

### 4. 注意线程上下文

该回调在 Zigbee 主任务上下文执行，因此**无需** `esp_zigbee_lock_acquire`；但不要在此阻塞或做耗时操作，需要时用 `esp_zigbee_task_queue_post` 把工作投递到自己的任务。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 命令收到但灯不变 | 没注册 handler 或没匹配 cluster_id | 确认 `ezb_zcl_core_action_handler_register` 已调；switch 匹配 cluster |
| 回调里调用阻塞 API 卡死 | 在主任务上下文阻塞 | 改用 `esp_zigbee_task_queue_post` 异步处理 |
| cluster_role 判断错 | 误用 client 分支 | server 设备只处理 `EZB_ZCL_CLUSTER_SERVER` |
| 值类型读错 | 把 8-bit on_off 当 16-bit | 按 cluster 属性类型解析（on/off=uint8, level=uint8, temp=int16） |
| 恢复出厂不生效 | 没处理 `BASIC_RESET_TO_FACTORY_DEFAULT_CB_ID` | 在该分支复位设备状态 |

## 参考项目

- `examples/home_automation_devices/on_off_light/main/on_off_light.c` — 点灯回调
- `examples/home_automation_devices/color_dimmable_light/main/color_dimmable_light.c` — on/off + level + color
- `examples/home_automation_devices/thermostat/main/thermostat.c` — thermostat cluster 处理
- `components/esp-zigbee-lib/include/ezbee/zcl/zcl_core.h` — 全部 callback_id
