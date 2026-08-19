# 标准设备与参数 Helper

> **适用摘要**: 使用 RainMaker 标准 helper API（`esp_rmaker_switch_device_create` / `lightbulb` / `fan` / `temp_sensor` 等）快速创建符合规范的设备，以及标准参数 helper（power/brightness/hue/saturation/temperature/speed 等）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-rainmaker/resources/`, source/examples in `repos/esp-rainmaker/`, and this recipe path `repos/esp-rainmaker/recipes/standard_devices.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "标准设备 switch/lightbulb/fan"
- "esp_rmaker_switch_device_create"
- "标准参数 power/brightness"
- "RainMaker 设备类型 esp.device.*"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `esp_rmaker_standard_devices.h`、`esp_rmaker_standard_params.h`、`esp_rmaker_standard_types.h` |
| 参考示例 | `examples/switch/`、`examples/led_light/`、`examples/fan/`、`examples/temperature_sensor/` |

## 标准设备类型宏（`esp_rmaker_standard_types.h`）

| 宏 | 设备类型字符串 |
|---|---|
| `ESP_RMAKER_DEVICE_SWITCH` | esp.device.switch |
| `ESP_RMAKER_DEVICE_LIGHTBULB` | esp.device.lightbulb |
| `ESP_RMAKER_DEVICE_FAN` | esp.device.fan |
| `ESP_RMAKER_DEVICE_TEMP_SENSOR` | esp.device.temperature-sensor |
| `ESP_RMAKER_DEVICE_LIGHT` | esp.device.light |
| `ESP_RMAKER_DEVICE_OUTLET` | esp.device.outlet |
| `ESP_RMAKER_DEVICE_PLUG` | esp.device.plug |
| `ESP_RMAKER_DEVICE_SOCKET` | esp.device.socket |
| `ESP_RMAKER_DEVICE_LOCK` | esp.device.lock |
| `ESP_RMAKER_DEVICE_BLINDS_INTERNAL` | esp.device.blinds-internal |
| `ESP_RMAKER_DEVICE_BLINDS_EXTERNAL` | esp.device.blinds-external |
| `ESP_RMAKER_DEVICE_GARAGE_DOOR` | esp.device.garage-door |
| `ESP_RMAKER_DEVICE_SPEAKER` | esp.device.speaker |
| `ESP_RMAKER_DEVICE_AIR_CONDITIONER` | esp.device.air-conditioner |
| `ESP_RMAKER_DEVICE_THERMOSTAT` | esp.device.thermostat |
| `ESP_RMAKER_DEVICE_TV` | esp.device.tv |
| `ESP_RMAKER_DEVICE_WASHER` | esp.device.washer |
| `ESP_RMAKER_DEVICE_OTHER` | esp.device.other |
| `ESP_RMAKER_DEVICE_ZIGBEE_GATEWAY` | esp.device.zigbee_gateway |
| `ESP_RMAKER_DEVICE_THREAD_BR` | esp.device.thread-br |

## 分步说明

### 1. 标准设备 helper（自动加 name + 主参数）

```c
#include <esp_rmaker_standard_devices.h>

/* Switch：自动加 Power 参数并设为主参数 */
esp_rmaker_device_t *sw = esp_rmaker_switch_device_create("Switch", NULL, false);

/* Lightbulb：自动加 Power 主参数 */
esp_rmaker_device_t *light = esp_rmaker_lightbulb_device_create("Light", NULL, true);

/* Fan：自动加 Power 主参数 */
esp_rmaker_device_t *fan = esp_rmaker_fan_device_create("Fan", NULL, false);

/* Temperature Sensor：自动加 Temperature 主参数 */
esp_rmaker_device_t *ts = esp_rmaker_temp_sensor_device_create("Temperature Sensor", NULL, 25.0);
```

> 这些 helper 内部已加 `Name` 参数与主参数（Power/Temperature），并 `assign_primary_param`，无需手写。

### 2. 标准参数 helper（补加额外参数）

```c
#include <esp_rmaker_standard_params.h>

esp_rmaker_param_t *brightness = esp_rmaker_brightness_param_create(ESP_RMAKER_DEF_BRIGHTNESS_NAME, 80);
esp_rmaker_param_t *hue        = esp_rmaker_hue_param_create(ESP_RMAKER_DEF_HUE_NAME, 0);
esp_rmaker_param_t *saturation = esp_rmaker_saturation_param_create(ESP_RMAKER_DEF_SATURATION_NAME, 100);
esp_rmaker_param_t *intensity  = esp_rmaker_intensity_param_create(ESP_RMAKER_DEF_INTENSITY_NAME, 0);
esp_rmaker_param_t *cct        = esp_rmaker_cct_param_create(ESP_RMAKER_DEF_CCT_NAME, 50);
esp_rmaker_param_t *speed      = esp_rmaker_speed_param_create(ESP_RMAKER_DEF_SPEED_NAME, 0);
esp_rmaker_param_t *direction  = esp_rmaker_direction_param_create(ESP_RMAKER_DEF_DIRECTION_NAME, 0);
esp_rmaker_param_t *temperature= esp_rmaker_temperature_param_create(ESP_RMAKER_DEF_TEMPERATURE_NAME, 25.0f);
esp_rmaker_param_t *name       = esp_rmaker_name_param_create(ESP_RMAKER_DEF_NAME_PARAM, "Light");
esp_rmaker_param_t *power      = esp_rmaker_power_param_create(ESP_RMAKER_DEF_POWER_NAME, false);

esp_rmaker_device_add_param(light, brightness);
esp_rmaker_device_add_param(light, hue);
esp_rmaker_device_add_param(light, saturation);
```

> 默认参数名宏见 `esp_rmaker_standard_params.h`：`ESP_RMAKER_DEF_POWER_NAME`（"Power"）、`ESP_RMAKER_DEF_BRIGHTNESS_NAME`（"Brightness"）等。名称非强制，可用自定义名。

### 3. 为彩灯补 brightness/hue/saturation（参考 led_light 示例）

```c
esp_rmaker_device_t *light_device = esp_rmaker_lightbulb_device_create("Light", NULL, DEFAULT_POWER);
esp_rmaker_device_add_bulk_cb(light_device, bulk_write_cb, NULL);

esp_rmaker_device_add_param(light_device,
        esp_rmaker_brightness_param_create(ESP_RMAKER_DEF_BRIGHTNESS_NAME, DEFAULT_BRIGHTNESS));
esp_rmaker_device_add_param(light_device,
        esp_rmaker_hue_param_create(ESP_RMAKER_DEF_HUE_NAME, DEFAULT_HUE));
esp_rmaker_device_add_param(light_device,
        esp_rmaker_saturation_param_create(ESP_RMAKER_DEF_SATURATION_NAME, DEFAULT_SATURATION));

esp_rmaker_node_add_device(node, light_device);
```

### 4. 标准服务类型宏

| 宏 | 服务类型 |
|---|---|
| `ESP_RMAKER_SERVICE_OTA` | esp.service.ota |
| `ESP_RMAKER_SERVICE_TIME` | esp.service.time |
| `ESP_RMAKER_SERVICE_SCHEDULE` | esp.service.schedule |
| `ESP_RMAKER_SERVICE_SCENES` | esp.service.scenes |
| `ESP_RMAKER_SERVICE_SYSTEM` | esp.service.system |
| `ESP_RMAKER_SERVICE_LOCAL_CONTROL` | esp.service.local_control |
| `ESP_RMAKER_SERVICE_GROUPS` | esp.service.groups |
| `ESP_RMAKER_SERVICE_CONNECTIVITY` | esp.service.connectivity |
| `ESP_RMAKER_SERVICE_USER_AUTH` | esp.service.rmaker-user-auth |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| helper 创建后无 Power | 用了自定义 type 字符串覆盖 | helper 已内置；不要再重复加同名 Power |
| 主参数不可控 | 未 `assign_primary_param`（手写设备时） | helper 自动设；手写时显式调用 |
| App 图标不对 | device type 字符串拼错 | 用 `ESP_RMAKER_DEVICE_*` 宏，不要手打 |
| 子类型不显示 | 未 `esp_rmaker_device_add_subtype` | 加 `esp.subtype.*` 子类型字符串 |

## 参考

- `components/esp_rainmaker/include/esp_rmaker_standard_devices.h`
- `components/esp_rainmaker/include/esp_rmaker_standard_params.h`
- `components/esp_rainmaker/include/esp_rmaker_standard_types.h`
- `examples/led_light/main/app_main.c`、`examples/fan/`、`examples/temperature_sensor/`
