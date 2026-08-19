# 多设备节点

> **适用摘要**: 在单个节点上挂载多个设备（如 Switch + Light + Fan + Temperature Sensor），共享 write 回调并按设备名分发。参考 `examples/multi_device/`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "一个节点多个设备"
- "multi device"
- "网关/集线器多设备"
- "按设备名区分回调"

## 前置条件

| 条件 | 要求 |
|---|---|
| 节点 | 已 `esp_rmaker_node_init()` |
| 参考示例 | `examples/multi_device/main/app_main.c` |

## 分步说明

### 1. 共享 write 回调（按设备名 + 参数名分发）

```c
static esp_err_t write_cb(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
            const esp_rmaker_param_val_t val, void *priv_data, esp_rmaker_write_ctx_t *ctx)
{
    const char *device_name = esp_rmaker_device_get_name(device);
    const char *param_name  = esp_rmaker_param_get_name(param);

    if (strcmp(param_name, ESP_RMAKER_DEF_POWER_NAME) == 0) {
        ESP_LOGI(TAG, "Received value = %s for %s - %s",
                val.val.b ? "true" : "false", device_name, param_name);
        if (strcmp(device_name, "Switch") == 0) {
            app_driver_set_state(val.val.b);   /* 只 Switch 控物理 IO */
        }
    } else if (strcmp(param_name, ESP_RMAKER_DEF_BRIGHTNESS_NAME) == 0) {
        ESP_LOGI(TAG, "Brightness %d for %s", val.val.i, device_name);
    } else if (strcmp(param_name, ESP_RMAKER_DEF_SPEED_NAME) == 0) {
        ESP_LOGI(TAG, "Speed %d for %s", val.val.i, device_name);
    } else {
        return ESP_OK;   /* 忽略未知参数 */
    }
    esp_rmaker_param_update(param, val);
    return ESP_OK;
}
```

### 2. 创建并挂载多个设备

```c
esp_rmaker_node_t *node = esp_rmaker_node_init(&cfg, "ESP RainMaker Multi Device", "Multi Device");

/* Switch */
esp_rmaker_device_t *switch_device = esp_rmaker_switch_device_create("Switch", NULL, DEFAULT_SWITCH_POWER);
esp_rmaker_device_add_cb(switch_device, write_cb, NULL);
esp_rmaker_node_add_device(node, switch_device);

/* Light（含亮度 + 属性） */
esp_rmaker_device_t *light_device = esp_rmaker_lightbulb_device_create("Light", NULL, DEFAULT_LIGHT_POWER);
esp_rmaker_device_add_cb(light_device, write_cb, NULL);
esp_rmaker_device_add_param(light_device,
        esp_rmaker_brightness_param_create(ESP_RMAKER_DEF_BRIGHTNESS_NAME, DEFAULT_LIGHT_BRIGHTNESS));
esp_rmaker_device_add_attribute(light_device, "Serial Number", "012345");
esp_rmaker_device_add_attribute(light_device, "MAC", "xx:yy:zz:aa:bb:cc");
esp_rmaker_node_add_device(node, light_device);

/* Fan（含速度） */
esp_rmaker_device_t *fan_device = esp_rmaker_fan_device_create("Fan", NULL, DEFAULT_FAN_POWER);
esp_rmaker_device_add_cb(fan_device, write_cb, NULL);
esp_rmaker_device_add_param(fan_device,
        esp_rmaker_speed_param_create(ESP_RMAKER_DEF_SPEED_NAME, DEFAULT_FAN_SPEED));
esp_rmaker_node_add_device(node, fan_device);

/* Temperature Sensor（只读，温度由应用主动上报） */
esp_rmaker_device_t *temp_device = esp_rmaker_temp_sensor_device_create("Temperature Sensor", NULL, app_get_current_temperature());
esp_rmaker_node_add_device(node, temp_device);
```

### 3. 运行时按名查询设备/参数

```c
esp_rmaker_device_t *dev = esp_rmaker_node_get_device_by_name(node, "Light");
esp_rmaker_param_t  *pwr = esp_rmaker_device_get_param_by_type(dev, ESP_RMAKER_PARAM_POWER);
/* 或按参数名：esp_rmaker_device_get_param_by_name(dev, "Power"); */

/* 主动上报温度 */
esp_rmaker_param_t *temp = esp_rmaker_device_get_param_by_type(temp_device, ESP_RMAKER_PARAM_TEMPERATURE);
esp_rmaker_param_update_and_report(temp, esp_rmaker_float(26.3));
```

### 4. 运行时新增/删除设备（hub 场景）

```c
/* start 之后再动态加设备，需要 esp_rmaker_report_node_details() 让云端重拉配置 */
esp_rmaker_node_add_device(node, new_device);
esp_rmaker_report_node_details();

/* 删除：先从节点移除，再 delete */
esp_rmaker_node_remove_device(node, old_device);
esp_rmaker_device_delete(old_device);
esp_rmaker_report_node_details();
```

> `esp_rmaker_report_node_details()` 注释明确：仅在 `esp_rmaker_start()` 之后动态增删设备时使用（如 bridge/hub）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| App 看不到新加设备 | start 后加设备未重报 | 调用 `esp_rmaker_report_node_details()` |
| 回调里设备名/参数名混乱 | 未用 `esp_rmaker_device_get_name` 区分 | 回调内先取设备名再分发 |
| 多设备回调注册重复 | 每个设备单独 `add_cb` | 每设备都调一次，可共用同一回调函数 |
| 设备重名 | 名称在同节点必须唯一 | 用 Switch1/Switch2 等区分 |
| 删除设备内存泄漏 | 未先 `remove_device` | 先 `esp_rmaker_node_remove_device` 再 `device_delete` |

## 参考

- `examples/multi_device/main/app_main.c` — 四设备节点完整示例
- `examples/zigbee_gateway/` — Zigbee 子设备动态映射为 RainMaker 设备
- `resources/api_reference.md` — Node/Device/Param 查询 API
