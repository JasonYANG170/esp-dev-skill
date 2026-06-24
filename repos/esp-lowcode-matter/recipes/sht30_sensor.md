# SHT30 温度传感器（I2C）周期上报

> **适用摘要**: 用 I2C 初始化 + `temperature_sensor_sht30_init` + `temperature_sensor_sht30_get_celsius` 周期读取温度，并通过 `LOW_CODE_FEATURE_ID_TEMPERATURE_SENSOR_VALUE` 上报到 Matter。

## 触发意图

- "温度传感器"
- "SHT30"
- "读取温度并上报"
- "I2C 传感器 lowcode"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `i2c_master.h`、`temperature_sensor_sht30.h` |
| 组件依赖 | REQUIRES 含 `temperature_sensor_sht30`（i2c 驱动随之带入） |
| 参考产品 | `products/temperature_sensor`、`products/temperature_sensor_with_display` |

## 分步说明

### 1. API（`temperature_sensor_sht30.h`）

```c
int temperature_sensor_sht30_init(int i2c_port);
int temperature_sensor_sht30_get_celsius(int i2c_port, float *temperature);   /* 0 成功，非0 失败 */
```

### 2. I2C 初始化（取自 `products/temperature_sensor`）

```cpp
#include <i2c_master.h>
#include <temperature_sensor_sht30.h>

#define I2C_PORT   I2C_NUM_0
#define I2C_SCL_IO (gpio_num_t)1
#define I2C_SDA_IO (gpio_num_t)2

int ret = i2c_master_init(I2C_PORT, I2C_SCL_IO, I2C_SDA_IO);
if (ret) {
    printf("%s: Failed to initialise master i2c\n", TAG);
    return -1;
}
ret = temperature_sensor_sht30_init(I2C_PORT);
if (ret) {
    printf("%s: Failed to initialise sht30 temperature sensor\n", TAG);
    return -1;
}
```

### 3. 周期读取并上报（10 秒）

```cpp
static void app_driver_report_temperature(float temp)
{
    int16_t temperature = temp * 100;   /* 带符号，°C*100 */
    low_code_feature_data_t update_data = {
        .details = { .endpoint_id = 1, .feature_id = LOW_CODE_FEATURE_ID_TEMPERATURE_SENSOR_VALUE },
        .value = {
            .type = LOW_CODE_VALUE_TYPE_INTEGER,        /* 温度可负，用 INTEGER */
            .value_len = sizeof(int16_t),
            .value = (uint8_t*)&temperature,
        },
    };
    low_code_feature_update_to_system(&update_data);
}

void app_driver_read_and_report_feature(system_timer_handle_t timer_handle, void *user_data)
{
    float temperature = 0.0;
    temperature_sensor_sht30_get_celsius(I2C_PORT, &temperature);
    system_delay_ms(100);
    app_driver_report_temperature(temperature);
}

/* app_driver_init 末尾：10s 周期 */
system_timer_handle_t timer = system_timer_create(app_driver_read_and_report_feature, NULL, 10000, true);
system_timer_start(timer);
```

### 4. 配合 OLED 显示（见 `recipes/ssd1306_display.md`）

`products/temperature_sensor_with_display` 在同一回调里把温度格式化后显示到 SSD1306/SSD1315。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 读数固定 0 / 失败 | I2C 未初始化或 SCL/SDA 接反 | 先 `i2c_master_init` 再 `sht30_init`；核对引脚 |
| 上报值异常 | 用了 UNSIGNED_INTEGER | 温度用 `LOW_CODE_VALUE_TYPE_INTEGER` |
| 单位错误 | 直接传 float | 传 `int16_t temp*100` |
| 读数偶发失败 | 总线时序 | 读后加 `system_delay_ms(100)` 再上报 |

## 参考

- `components/temperature_sensor_sht30/temperature_sensor_sht30.h`
- `products/temperature_sensor/main/app_driver.cpp`
- `products/temperature_sensor_with_display/main/app_driver.cpp`
- `recipes/system_timer.md`、`recipes/ssd1306_display.md`
