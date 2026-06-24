# 灯具设备（On/Off / Dimmable / Color Temperature / Extended Color）

> **适用摘要**: 用 esp-matter 标准 device type 创建各类灯具 endpoint，绑定 LED 驱动，在 `app_attribute_update_cb` 里做 Matter 单位到 LED 单位的���映射，并用 `attribute::update()` 让按键反向写回数据模型。

## 触发意图

- "做一个 Matter 灯"
- "on_off_light / color_temperature_light / extended_color_light 怎么建"
- "Matter 亮度/色温怎么映射到 LED"
- "按键控制 Matter 灯"
- "extended color light"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/light/`（`main/app_main.cpp`、`main/app_driver.cpp`、`main/app_priv.h`） |
| 设备 HAL | `device_hal` 选定板子后自动带 `led_driver` / `button_driver` |

## 分步说明

### 1. 选择 device type 与 config_t

四种灯具 device type 命名空间（取自 `esp_matter_endpoint_impl.h`）层层继承，cluster 越来越多：

| device type | namespace | 含 cluster |
|---|---|---|
| On/Off Light | `endpoint::on_off_light` | OnOff |
| Dimmable Light | `endpoint::dimmable_light` | + LevelControl |
| Color Temperature Light | `endpoint::color_temperature_light` | + ColorControl(CT) |
| Extended Color Light | `endpoint::extended_color_light` | + ColorControl(HS+XY+CT) |

Extended Color Light 的 `config_t` 字段（来自 `examples/light/main/app_main.cpp`）：

```cpp
extended_color_light::config_t light_config;
light_config.on_off.on_off = DEFAULT_POWER;                       // bool
light_config.on_off_lighting.start_up_on_off = nullptr;           // nullable 用 nullptr
light_config.level_control.current_level = DEFAULT_BRIGHTNESS;    // 0..254
light_config.level_control.on_level = DEFAULT_BRIGHTNESS;
light_config.level_control_lighting.start_up_current_level = DEFAULT_BRIGHTNESS;
light_config.color_control.color_mode = (uint8_t)chip::app::Clusters::ColorControl::ColorMode::kColorTemperature;
light_config.color_control.enhanced_color_mode = (uint8_t)chip::app::Clusters::ColorControl::ColorMode::kColorTemperature;
light_config.color_control_color_temperature.start_up_color_temperature_mireds = nullptr;
```

### 2. 创建 endpoint 并记下 endpoint_id

```cpp
endpoint_t *endpoint = extended_color_light::create(node, &light_config,
                                                    ENDPOINT_FLAG_NONE, light_handle);
light_endpoint_id = endpoint::get_id(endpoint);
```

`light_handle` 作为 `priv_data` 传入，回调里再 cast 回来。

### 3. 为频繁变化的属性注册 deferred persistence

来自 `examples/light/main/app_main.cpp`：

```cpp
attribute_t *cur = attribute::get(light_endpoint_id, LevelControl::Id,
                                  LevelControl::Attributes::CurrentLevel::Id);
attribute::set_deferred_persistence(cur);

attribute_t *cx = attribute::get(light_endpoint_id, ColorControl::Id,
                                 ColorControl::Attributes::CurrentX::Id);
attribute::set_deferred_persistence(cx);
attribute_t *cy = attribute::get(light_endpoint_id, ColorControl::Id,
                                 ColorControl::Attributes::CurrentY::Id);
attribute::set_deferred_persistence(cy);
attribute_t *ct = attribute::get(light_endpoint_id, ColorControl::Id,
                                 ColorControl::Attributes::ColorTemperatureMireds::Id);
attribute::set_deferred_persistence(ct);
```

### 4. 单位重映射宏（`app_priv.h`）

Matter 数值范围与 LED 驱动范围不同，`examples/light/main/app_priv.h` 定义了重映射：

```c
#define STANDARD_BRIGHTNESS       100
#define STANDARD_HUE              360
#define STANDARD_SATURATION       100
#define STANDARD_TEMPERATURE_FACTOR 1000000

#define MATTER_BRIGHTNESS         254
#define MATTER_HUE                254
#define MATTER_SATURATION         254
#define MATTER_TEMPERATURE_FACTOR 1000000
```

### 5. `app_attribute_update_cb` → 驱动联动

回调里按 `endpoint_id → cluster_id → attribute_id` 三级分发（来自 `app_driver.cpp`）：

```cpp
esp_err_t app_driver_attribute_update(app_driver_handle_t driver_handle, uint16_t endpoint_id,
                                      uint32_t cluster_id, uint32_t attribute_id,
                                      esp_matter_attr_val_t *val) {
    esp_err_t err = ESP_OK;
    if (endpoint_id == light_endpoint_id) {
        led_driver_handle_t handle = (led_driver_handle_t)driver_handle;
        if (cluster_id == OnOff::Id) {
            if (attribute_id == OnOff::Attributes::OnOff::Id) {
                err = led_driver_set_power(handle, val->val.b);
            }
        } else if (cluster_id == LevelControl::Id) {
            if (attribute_id == LevelControl::Attributes::CurrentLevel::Id) {
                int v = REMAP_TO_RANGE(val->val.u8, MATTER_BRIGHTNESS, STANDARD_BRIGHTNESS);
                err = led_driver_set_brightness(handle, v);
            }
        } else if (cluster_id == ColorControl::Id) {
            if (attribute_id == ColorControl::Attributes::ColorTemperatureMireds::Id) {
                uint32_t v = REMAP_TO_RANGE_INVERSE(val->val.u16, STANDARD_TEMPERATURE_FACTOR);
                err = led_driver_set_temperature(handle, v);
            }
            // CurrentHue / CurrentSaturation / CurrentX / CurrentY 同理
        }
    }
    return err;
}
```

### 6. 按键反向写回数据模型

按键回调里读当前值、取反、`attribute::update()`（来自 `app_driver.cpp` 的 `app_driver_button_toggle_cb`）：

```cpp
static void app_driver_button_toggle_cb(void *arg, void *data) {
    uint16_t endpoint_id = light_endpoint_id;
    attribute_t *attr = attribute::get(endpoint_id, OnOff::Id, OnOff::Attributes::OnOff::Id);
    esp_matter_attr_val_t val;
    attribute::get_val(attr, &val);
    val.val.b = !val.val.b;
    attribute::update(endpoint_id, OnOff::Id, OnOff::Attributes::OnOff::Id, &val);
}
```

### 7. start 之后把默认值推给硬件

```cpp
err = esp_matter::start(app_event_cb);
app_driver_light_set_defaults(light_endpoint_id);   // 读数据库当前值，初始化 LED
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 灯不亮 / 不变色 | cluster_id/attribute_id 不匹配 | 严格按 `OnOff::Id` / `LevelControl::Id` / `ColorControl::Id` 判断 |
| 色温方向反 | mireds 没取反 | 用 `REMAP_TO_RANGE_INVERSE` |
| 亮度跳变 | 没设 deferred persistence | 对 CurrentLevel 调 `set_deferred_persistence` |
| 按键无反应 | `iot_button_register_cb` 回调里没 cast handle | 用 `priv_data` 而非全局变量 |
| 切色温模式后 XY 不变 | `color_mode` 没设对 | 按 `ColorControl::ColorMode::kColorTemperature` 设 `color_mode` 与 `enhanced_color_mode` |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/examples/light/main/app_main.cpp`
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/main/app_driver.cpp`
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/main/app_priv.h`
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h`（`on_off_light` / `dimmable_light` / `color_temperature_light` / `extended_color_light`）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Building a Color Temperature Lightbulb
