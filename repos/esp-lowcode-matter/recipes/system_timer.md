# system_timer 软件定时器（周期上报模式）

> **适用摘要**: 用 `system_timer_create` / `system_timer_start` 创建周期或单次定时器，在回调里周期读取并上报特性（传感器读数的标准模式）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/system_timer.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "周期上报"
- "定时读取传感器"
- "system_timer"
- "periodic timer"
- "轮询上报数据"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `system.h`（`system_timer_handle_t`、`system_timer_*`） |
| 参考产品 | `products/temperature_sensor`（10s）、`products/occupancy_sensor`（2s）、`products/temperature_sensor_with_display`（10s） |

## 分步说明

### 1. API（`system.h`）

```c
typedef void *system_timer_handle_t;
typedef void (*system_timer_cb_t)(system_timer_handle_t timer_handle, void *user_data);

system_timer_handle_t system_timer_create(system_timer_cb_t callback, void *arg, int timeout_ms, bool periodic);
int  system_timer_start(system_timer_handle_t handle);
int  system_timer_stop(system_timer_handle_t handle);
int  system_timer_delete(system_timer_handle_t handle);
```

> 回调签名**必须**是 `(system_timer_handle_t, void*)`——即便不用这两个参数也要保留。

### 2. 标准模式：周期读取并上报

取自 `products/temperature_sensor/main/app_driver.cpp`（每 10 秒读 SHT30 并上报）：

```cpp
#include <system.h>
#include <low_code.h>
#include <temperature_sensor_sht30.h>

#define I2C_PORT I2C_NUM_0

static void app_driver_report_temperature(float temp)
{
    int16_t temperature = temp * 100;   /* °C*100，带符号 */
    low_code_feature_data_t update_data = {
        .details = { .endpoint_id = 1, .feature_id = LOW_CODE_FEATURE_ID_TEMPERATURE_SENSOR_VALUE },
        .value = {
            .type = LOW_CODE_VALUE_TYPE_INTEGER,
            .value_len = sizeof(int16_t),
            .value = (uint8_t*)&temperature,
        },
    };
    low_code_feature_update_to_system(&update_data);
}

/* 回调签名固定，timer_handle/user_data 故意未用 */
void app_driver_read_and_report_feature(system_timer_handle_t timer_handle, void *user_data)
{
    float temperature = 0.0;
    temperature_sensor_sht30_get_celsius(I2C_PORT, &temperature);
    system_delay_ms(100);
    app_driver_report_temperature(temperature);
}

/* 在 app_driver_init 末尾创建并启动（periodic=true） */
system_timer_handle_t timer = system_timer_create(app_driver_read_and_report_feature, NULL, 10000, true);
if (!timer) {
    printf("%s: Failed to create timer\n", TAG);
    return -1;
}
system_timer_start(timer);
```

### 3. 占用传感器（更短周期，取自 `products/occupancy_sensor`）

```cpp
system_timer_handle_t timer = system_timer_create(app_driver_read_and_report_feature, NULL, 2000, true);
system_timer_start(timer);
```

回调里读 LD2420 并以 `LOW_CODE_FEATURE_ID_OCCUPANCY_SENSOR_VALUE`（`UNSIGNED_INTEGER`）上报 `data.occupied`。

### 4. 单次定时器

```cpp
system_timer_handle_t one_shot = system_timer_create(one_shot_cb, NULL, 5000, false);
system_timer_start(one_shot);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 回调不执行 | 签名不是 `(system_timer_handle_t, void*)` | 严格按签名，未用参数也保留 |
| 定时器不跑 | 没调 `system_timer_start` 或漏 `system_loop()` | create 后 start；主循环跑 `system_loop()` |
| 读数跳变/NaN | 传感器未初始化就读 | 先 `temperature_sensor_sht30_init` 再建定时器 |
| 上报格式错 | 温度用了 UNSIGNED_INTEGER | 温度带符号用 `INTEGER`，占用 0/1 用 `UNSIGNED_INTEGER` |
| 创建返回 NULL | 内存/参数错 | 检查 arg 与 timeout；避免在回调里动态分配 |

## 参考

- `components/system/system.h`
- `products/temperature_sensor/main/app_driver.cpp`
- `products/occupancy_sensor/main/app_driver.cpp`
- `products/temperature_sensor_with_display/main/app_driver.cpp`
- `recipes/sht30_sensor.md`、`recipes/ld2420_occupancy.md`
