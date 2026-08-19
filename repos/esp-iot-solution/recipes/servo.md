# LEDC 舵机角度控制（iot_servo）

> **适用摘要**: 使用 `iot_servo` 组件基于 ESP-IDF LEDC 生成 PWM 控制舵机（如 SG90/MG996R），初始化通道、写入目标角度并读取当前角度。组件用 `servo_config_t` 配置最大角度、脉宽范围（典型 500~2500µs）与 PWM 频率（典型 50Hz）。

> Evidence: `repos/esp-iot-solution/resources/`, source/examples in `repos/esp-iot-solution/`, and this recipe path `repos/esp-iot-solution/recipes/servo.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "舵机控制"
- "SG90 / MG996R"
- "iot_servo"
- "LEDC 舵机角度"
- "iot_servo_write_angle"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | 全系 ESP32（带 LEDC） |
| IDF 环境 | ESP-IDF v4.4+（master 需 v5.3+） |
| 组件依赖 | `espressif/iot_servo` |
| 硬件 | 舵机信号线接 GPIO；独立 5V 供电（勿从芯片 3V3 取大电流） |
| 参考示例 | `examples/motor/servo_control/` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/iot_servo"
```

### 2. 初始化舵机

`servo_config_t.channels` 含 `servo_pin[LEDC_CHANNEL_MAX]` 与 `ch[LEDC_CHANNEL_MAX]` 两个数组，`channel_number` 必须等于实际填入的通道项数。典型脉宽 `min_width_us=500`、`max_width_us=2500`，频率 `50`Hz。

```c
#include "iot_servo.h"

servo_config_t servo_cfg = {
    .max_angle    = 180,
    .min_width_us = 500,
    .max_width_us = 2500,
    .freq         = 50,
    .timer_number = LEDC_TIMER_0,
    .channels = {
        .servo_pin = { 2 },                 // 通道 0 接 GPIO2
        .ch        = { LEDC_CHANNEL_0 },
    },
    .channel_number = 1,                    // 与上面填的项数一致
};
ESP_ERROR_CHECK(iot_servo_init(LEDC_LOW_SPEED_MODE, &servo_cfg));
```

> 多路舵机时把多个 `servo_pin` / `ch` 填进数组，`channel_number` 设为实际路数。

### 3. 写入角度

```c
// 通道 0 转到 90 度（注意：本 API 非线程安全，多任务调用需自加锁）
iot_servo_write_angle(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, 90.0f);
```

### 4. 读取当前角度

```c
float angle = 0;
iot_servo_read_angle(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, &angle);
ESP_LOGI(TAG, "current angle = %.1f", angle);
```

### 5. 缓动到目标（应用层示例）

`iot_servo` 只提供点位控制；平滑扫描可在应用层分步写角度：

```c
for (float a = 0; a <= 180; a += 1.0f) {
    iot_servo_write_angle(LEDC_LOW_SPEED_MODE, LEDC_CHANNEL_0, a);
    vTaskDelay(pdMS_TO_TICKS(20));
}
```

### 6. 反初始化

```c
iot_servo_deinit(LEDC_LOW_SPEED_MODE);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 舵机抖动 / 不动 | `channel_number` 与实际填的通道项数不一致 | 让 `channel_number` 严格等于数组中有效项数 |
| 角度范围对不上 | `min/max_width_us` 与舵机规格不符 | 查舵机数据手册，常见 500~2500µs，部分 SG90 用 500~2400µs |
| `ESP_ERR_INVALID_ARG` | `channel` 非 `LEDC_CHANNEL_x` 或角度越界 | 角度限制在 `[0, max_angle]`；通道用枚举值 |
| 多任务调用写角度出错 | `iot_servo_write_angle` 非线程安全 | 用 mutex 保护写操作 |
| 舵机供电重启 | 从 3V3/开发板取电导致掉电 | 舵机用独立 5V 电源，共地 |

## 参考

- 组件头文件：`components/motor/servo/include/iot_servo.h`
- 真实示例：`examples/motor/servo_control/`
- 在线文档（电机总览）：`docs/en/motor/servo.rst`
