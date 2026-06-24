# sensor_hub 传感器集线器

> **适用摘要**: 使用 `sensor_hub` 组件统一管理传感器：创建传感器实例、按类型注册事件回调、启动后周期采集并接收 `SENSOR_XXX_DATA_READY` 事件。sensor_hub 通过链接脚本机制自动加载被加入工程的传感器驱动组件。

## 触发意图

- "传感器采集"
- "sensor_hub"
- "温湿度 / IMU / 光照 传感器"
- "iot_sensor_create"
- "统一管理多个传感器"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+ |
| 组件依赖 | `espressif/sensor_hub` 及所需传感器驱动（如 `espressif/sht3x`、`espressif/lis2dh12`、`espressif/veml6040`） |
| 参考示例 | `examples/sensors/sensor_hub_monitor/main/app_main.c` |

## 分步说明

### 1. 添加依赖

```bash
idf.py add-dependency "espressif/sensor_hub"
idf.py add-dependency "espressif/sht3x"        # 温湿度
# 按需追加 lis2dh12（IMU）、veml6040（光照）等
```

> sensor_hub 利用 IDF 链接脚本生成机制（linker-script-generation）把传感器驱动注册到特定段；只要工程 include 了对应驱动组件，`iot_sensor_scan()` 即可发现。

### 2. 获取总线句柄

sensor_hub 需要一个 I2C 总线句柄。可用 `i2c_bus` 建总线，或用 board 组件（`iot_board_get_handle`）。

```c
#include "i2c_bus.h"

i2c_config_t conf = { /* sda/scl/clk ... */ .mode = I2C_MODE_MASTER };
i2c_bus_handle_t bus = i2c_bus_create(I2C_NUM_0, &conf);
```

### 3. 扫描已加载驱动

```c
#include "iot_sensor_hub.h"

iot_sensor_scan();   // 打印当前工程已注册的传感器名
```

### 4. 创建传感器实例并注册回调

```c
static sensor_handle_t sht3x_handle = NULL;
static sensor_event_handler_instance_t sht3x_handler = NULL;

static void sensor_event_handler(void *arg, sensor_event_base_t base,
                                 int32_t id, void *event_data)
{
    sensor_data_t *d = (sensor_data_t *)event_data;
    switch (id) {
    case SENSOR_STARTED:
        ESP_LOGI(TAG, "%s STARTED", d->sensor_name);
        break;
    case SENSOR_TEMP_DATA_READY:
        ESP_LOGI(TAG, "%s temp=%.2f", d->sensor_name, d->temperature);
        break;
    case SENSOR_HUMI_DATA_READY:
        ESP_LOGI(TAG, "%s humi=%.2f", d->sensor_name, d->humidity);
        break;
    default:
        break;
    }
}

void sensor_init(i2c_bus_handle_t bus)
{
    sensor_config_t cfg = {
        .bus = bus,
        .addr = 0x44,                 // SHT3x 默认地址
        .type = HUMITURE_ID,
        .mode = MODE_POLLING,
        .min_delay = 200,             // 采集间隔 ms
    };
    ESP_ERROR_CHECK(iot_sensor_create("sht3x", &cfg, &sht3x_handle));
    ESP_ERROR_CHECK(iot_sensor_handler_register(sht3x_handle, sensor_event_handler, &sht3x_handler));
    ESP_ERROR_CHECK(iot_sensor_start(sht3x_handle));
}
```

### 5. 按类型注册（跨实例）

```c
// 所有 HUMITURE_ID 传感器的 TEMP 事件统一回调
iot_sensor_handler_register_with_type(HUMITURE_ID, SENSOR_TEMP_DATA_READY,
                                      type_wide_handler, &inst);
```

### 6. 停止 / 注销 / 删除

```c
iot_sensor_stop(sht3x_handle);
iot_sensor_handler_unregister(sht3x_handle, sht3x_handler);
iot_sensor_delete(sht3x_handle);
```

## 常用传感器类型与事件

| 传感器类型 (`sensor_type_t`) | 典型芯片 | 数据就绪事件 |
|---|---|---|
| `HUMITURE_ID` | sht3x, aht20, bme280 | `SENSOR_TEMP_DATA_READY`、`SENSOR_HUMI_DATA_READY` |
| `IMU_ID` | lis2dh12 | `SENSOR_ACCE_DATA_READY`、`SENSOR_GYRO_DATA_READY` |
| `LIGHT_SENSOR_ID` | veml6040, veml6075 | `SENSOR_LIGHT_DATA_READY`、`SENSOR_RGBW_DATA_READY`、`SENSOR_UV_DATA_READY` |

> 公共事件：`SENSOR_STARTED`、`SENSOR_STOPED`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `iot_sensor_create` 返回非 OK | 驱动组件未加入工程 | `idf.py add-dependency` 加对应芯片驱动；运行 `iot_sensor_scan` 看是否被发现 |
| 回调不触发 | 未调用 `iot_sensor_start` | 创建后必须 `start` |
| 找不到传感器名 | 名字拼写错或大小写不符 | 用 `iot_sensor_scan()` 输出的精确名字（如 `"sht3x"`、`"lis2dh12"`） |
| `bus` 为 NULL | 未先建总线 | 先 `i2c_bus_create` 取得 `bus_handle_t` |
| `min_delay` 太小采集过频 | 设为 0 或极小 | 设合理间隔（如 100~1000ms） |

## 参考

- 组件头文件：`components/sensors/sensor_hub/include/iot_sensor_hub.h`、`sensor_type.h`、`sensor_event.h`
- 在线文档：`docs/en/sensors/sensor_hub.rst`
- 真实示例：`examples/sensors/sensor_hub_monitor/main/app_main.c`、`examples/sensors/sensor_control_led/main/app_main.c`
