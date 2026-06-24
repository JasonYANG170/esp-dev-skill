# 自定义设备与参数

> **适用摘要**: 用 `esp_rmaker_device_create()` + `esp_rmaker_param_create()` 创建自定义设备，添加自定义参数、UI Type、数值范围（bounds）、有效字符串列表，并编写 write 回调。

## 触发意图

- "自定义 RainMaker 设备"
- "自定义参数 param"
- "加 UI Type"
- "参数范围 bounds"
- "esp_rmaker_param_create"
- "下拉框/滑块参数"

## 前置条件

| 条件 | 要求 |
|---|---|
| 节点 | 已 `esp_rmaker_node_init()` |
| 参考示例 | `examples/switch/main/app_main.c`（手写设备+参数） |

## 分步说明

### 1. 值类型 helper（构造 `esp_rmaker_param_val_t`）

```c
esp_rmaker_param_val_t esp_rmaker_bool(bool bval);
esp_rmaker_param_val_t esp_rmaker_int(int ival);
esp_rmaker_param_val_t esp_rmaker_float(float fval);
esp_rmaker_param_val_t esp_rmaker_str(const char *sval);
esp_rmaker_param_val_t esp_rmaker_obj(const char *val);   // JSON 对象，如 {"name":"value"}
esp_rmaker_param_val_t esp_rmaker_array(const char *val); // JSON 数组，如 [1,2,3]
```

参数属性 flags（`esp_param_property_flags_t`，逻辑或）：

```c
PROP_FLAG_WRITE              // 可写
PROP_FLAG_READ               // 可读
PROP_FLAG_TIME_SERIES        // 时间序列
PROP_FLAG_PERSIST            // 持久化（重启后从 NVS 恢复）
PROP_FLAG_SIMPLE_TIME_SERIES // 简单时间序列
```

### 2. 创建设备 + 自定义参数

```c
/* 创建设备（type 可用自定义字符串或标准 ESP_RMAKER_DEVICE_* 宏） */
esp_rmaker_device_t *dev = esp_rmaker_device_create("MyDevice", "custom.device.demo", NULL);
esp_rmaker_device_add_cb(dev, write_cb, NULL);

/* 自定义参数：开关（布尔，可读可写，持久化） */
esp_rmaker_param_t *power = esp_rmaker_param_create(
        "Power", ESP_RMAKER_PARAM_POWER,
        esp_rmaker_bool(false),
        PROP_FLAG_READ | PROP_FLAG_WRITE | PROP_FLAG_PERSIST);
esp_rmaker_param_add_ui_type(power, ESP_RMAKER_UI_TOGGLE);
esp_rmaker_device_add_param(dev, power);
esp_rmaker_device_assign_primary_param(dev, power);

/* 自定义参数：亮度（整数，加范围） */
esp_rmaker_param_t *brightness = esp_rmaker_param_create(
        "Brightness", ESP_RMAKER_PARAM_BRIGHTNESS,
        esp_rmaker_int(50),
        PROP_FLAG_READ | PROP_FLAG_WRITE);
esp_rmaker_param_add_ui_type(brightness, ESP_RMAKER_UI_SLIDER);
esp_rmaker_param_add_bounds(brightness, esp_rmaker_int(0), esp_rmaker_int(100), esp_rmaker_int(5));
esp_rmaker_device_add_param(dev, brightness);

/* 自定义参数：模式（字符串下拉框） */
static const char *modes[] = {"Off", "Low", "High"};
esp_rmaker_param_t *mode = esp_rmaker_param_create(
        "Mode", ESP_RMAKER_PARAM_MODE,
        esp_rmaker_str("Off"),
        PROP_FLAG_READ | PROP_FLAG_WRITE);
esp_rmaker_param_add_ui_type(mode, ESP_RMAKER_UI_DROPDOWN);
esp_rmaker_param_add_valid_str_list(mode, modes, 3);
esp_rmaker_device_add_param(dev, mode);

/* 挂到节点 */
esp_rmaker_node_add_device(node, dev);
```

> 标准 UI Type 宏见 `esp_rmaker_standard_types.h`：`ESP_RMAKER_UI_TOGGLE`/`_SLIDER`/`_DROPDOWN`/`_TEXT`/`_HUE_SLIDER`/`_HUE_CIRCLE`/`_PUSHBUTTON`/`_TRIGGER`/`_HIDDEN`/`_QR_SCAN`。

### 3. write 回调（按参数名分发）

```c
static esp_err_t write_cb(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
            const esp_rmaker_param_val_t val, void *priv_data, esp_rmaker_write_ctx_t *ctx)
{
    const char *name = esp_rmaker_param_get_name(param);
    if (strcmp(name, "Power") == 0) {
        app_driver_set_power(val.val.b);
    } else if (strcmp(name, "Brightness") == 0) {
        app_driver_set_brightness(val.val.i);
        /* 注：core 不检查 bounds，应用自行校验 */
    } else if (strcmp(name, "Mode") == 0) {
        app_driver_set_mode(val.val.s);
    }
    esp_rmaker_param_update(param, val);   /* 回写 */
    return ESP_OK;
}
```

### 4. 添加设备属性（attributes，仅启动时上报一次）

```c
esp_rmaker_device_add_attribute(dev, "Serial Number", "0123456789");
esp_rmaker_device_add_attribute(dev, "MAC", "aa:bb:cc:dd:ee:ff");
esp_rmaker_device_add_subtype(dev, "custom.subtype.rgb");
esp_rmaker_device_add_model(dev, "Demo-001");
```

### 5. 主动上报与通知

```c
/* 仅更新（不立即上报），适合批量上报 */
esp_rmaker_param_update(brightness, esp_rmaker_int(80));

/* 更新并上报 */
esp_rmaker_param_update_and_report(brightness, esp_rmaker_int(80));

/* 更新并触发手机通知（如报警/温度超限） */
esp_rmaker_param_update_and_notify(temp_param, esp_rmaker_float(42.5));

/* 批量更新后一次性上报 */
esp_rmaker_param_update(p1, esp_rmaker_int(1));
esp_rmaker_param_update(p2, esp_rmaker_int(2));
esp_rmaker_report_updated_params();
```

### 6. 简单时间序列（TTL）

```c
esp_rmaker_param_t *ts = esp_rmaker_param_create("Energy", "esp.param.energy",
        esp_rmaker_float(0.0),
        PROP_FLAG_READ | PROP_FLAG_SIMPLE_TIME_SERIES);
esp_rmaker_param_add_simple_time_series_ttl(ts, 30); /* 30 天 */

/* 直接上报带时间戳的时间序列点 */
esp_rmaker_param_report_simple_ts_data(ts, esp_rmaker_float(1.23), 0 /* 0=当前时间 */, 30);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| App 不显示控件 | 未加 UI Type | 调用 `esp_rmaker_param_add_ui_type()` |
| bounds 不生效 | core 不检查范围 | 在 write 回调内自行校验 `val.val.i` |
| 字符串参数任意值 | 未加有效字符串列表 | `esp_rmaker_param_add_valid_str_list()` |
| 持久化参数开机乱码 | 类型不匹配 | `esp_rmaker_bool()/int()` 类型要与初值一致 |
| `valid_strs` 数组失效 | 数组生命周期不够 | 用 `static const char *` 全局数组 |
| 上报丢失 | MQTT 预算耗尽 | 合并上报或调大 `CONFIG_ESP_RMAKER_MQTT_*_BUDGET` |

## 参考

- `components/esp_rainmaker/include/esp_rmaker_core.h` — Param/Device 全部 API
- `components/esp_rainmaker/include/esp_rmaker_standard_types.h` — UI Type / Param Type 宏
- `components/esp_rainmaker/include/esp_rmaker_standard_params.h` — 标准 param helper
- `examples/switch/main/app_main.c` — 手写设备+参数
- `resources/api_reference.md`
