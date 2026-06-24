# FOC 无刷电机控制 esp_simplefoc

> **适用摘要**: 使用 `esp_simplefoc` 组件（基于 Arduino-FOC，适配 ESP 的 LEDC/MCPWM）驱动三相无刷电机（BLDC），覆盖开环速度（`velocity_openloop`）与闭环速度（`velocity` + 角度传感器 AS5600/MT6701/AS5048A + PID）。入口仍是 ESP-IDF 的 `app_main`，但电机代码是 C++（`BLDCMotor` / `BLDCDriver3PWM` / `AS5600`）。

## 触发意图

- "FOC 无刷电机"
- "BLDC 控制"
- "esp_simplefoc"
- "BLDCDriver3PWM"
- "AS5600 / MT6701 / AS5048A"
- "开环 / 闭环速度控制"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | 带 LEDC 或 MCPWM 的 ESP32 系列（ESP32 / S3 / C6 / H2 等；P4 带 MCPWM） |
| IDF 环境 | ESP-IDF v5.3+ |
| 组件依赖 | `espressif/esp_simplefoc`（速度闭环额外需 `espressif/iqmath`） |
| FreeRTOS | menuconfig 设 `CONFIG_FREERTOS_HZ=1000`（默认 100 不够，FOC 控制环要 1ms） |
| 硬件 | 三相 MOSFET 驱动（3PWM/6PWM）+ 电机独立供电；闭环需磁编码器 |
| 参考示例 | `examples/motor/foc_openloop_control`、`foc_velocity_control`、`foc_knob_example` |

## 分步说明

### 1. 添加组件依赖并设 tick rate

```bash
idf.py add-dependency "espressif/esp_simplefoc"
```

`menuconfig → Component config → FreeRTOS → Kernel → configTICK_RATE_HZ` 改为 `1000`。

### 2. 声明电机、驱动、（可选）角度传感器

`BLDCMotor(极对数)` —— 14 极对（即 28 极）电机用 `BLDCMotor(14)`。
`BLDCDriver3PWM(phA, phB, phC)` —— 三相 PWM 引脚。
`AS5600(I2C_NUM_0, SDA, SCL)` —— 闭环才需要。

```cpp
#include "esp_simplefoc.h"

BLDCMotor      motor   = BLDCMotor(14);              // 14 极对
BLDCDriver3PWM driver  = BLDCDriver3PWM(17, 16, 15); // phA/phB/phC 引脚
// AS5600 as5600 = AS5600(I2C_NUM_0, GPIO_NUM_1, GPIO_NUM_2);  // 闭环用
```

### 3. 开环速度控制（硬件验证起步）

`MotionControlType::velocity_openloop` 不需传感器，电机按电角度开环旋转，适合先验证驱动硬件。

```cpp
extern "C" void app_main(void)
{
    SimpleFOCDebug::enable();            // 打开 SimpleFOC 调试输出
    Serial.begin(115200);

    driver.voltage_power_supply = 12;    // MOSFET 驱动供电
    driver.voltage_limit        = 11;    // 限幅，保护驱动

#if CONFIG_SOC_MCPWM_SUPPORTED
    driver.init(0);                       // MCPWM 芯片：传 timer 组号 0
#else
    driver.init({1, 2, 3});               // LEDC 芯片：传 3 个 LEDC channel
#endif
    motor.linkDriver(&driver);

    motor.velocity_limit = 200.0;         // rad/s
    motor.voltage_limit  = 12.0;          // V
    motor.controller     = MotionControlType::velocity_openloop;

    motor.init();

    while (1) {
        motor.move(1.2f);                 // 目标速度 rad/s
        vTaskDelay(1 / portTICK_PERIOD_MS);   // 1ms 节拍
    }
}
```

> `CONFIG_SOC_MCPWM_SUPPORTED` 决定走 MCPWM（传整数 timer id）还是 LEDC（传 channel 数组）。两种芯片的 `driver.init(...)` 参数不同，按上面宏分支。

### 4. 闭环速度控制（接角度传感器 + PID）

接 AS5600（或 MT6701/AS5048A），用 `initFOC` 对齐传感器与电极方向，再在循环里跑 `loopFOC` + `move`。

```cpp
BLDCMotor      motor   = BLDCMotor(14);
BLDCDriver3PWM driver  = BLDCDriver3PWM(4, 5, 6);
AS5600         as5600  = AS5600(I2C_NUM_0, GPIO_NUM_1, GPIO_NUM_2);

float target_value = 0.0f;
Commander command = Commander(Serial);
void doTarget(char *cmd) { command.scalar(&target_value, cmd); }

extern "C" void app_main(void)
{
    SimpleFOCDebug::enable();
    Serial.begin(115200);

    as5600.init();
    motor.linkSensor(&as5600);

    driver.voltage_power_supply = 12;
    driver.voltage_limit        = 11;
#if CONFIG_SOC_MCPWM_SUPPORTED
    driver.init(0);
#else
    driver.init({1, 2, 3});
#endif
    motor.linkDriver(&driver);

    motor.controller            = MotionControlType::velocity;
    motor.PID_velocity.P        = 0.9f;
    motor.PID_velocity.I        = 2.2f;
    motor.voltage_limit         = 11;
    motor.voltage_sensor_align  = 2;       // 对齐用电压
    motor.LPF_velocity.Tf       = 0.05;    // 速度低通滤波
    motor.velocity_limit        = 200;

    motor.useMonitoring(Serial);
    motor.init();
    motor.initFOC();                       // 对齐传感器（注意极对数正确）

    command.add('T', doTarget, const_cast<char *>("target angle"));

    while (1) {
        motor.loopFOC();                   // 高频 FOC 内环
        motor.move(target_value);          // 速度外环
        command.run();
        vTaskDelay(1 / portTICK_PERIOD_MS);
    }
}
```

> `motor.controller` 取值：`torque` / `velocity` / `angle` / `velocity_openloop` / `angle_openloop`。硬件验证阶段优先用 `*_openloop`。

### 5. 用 SimpleFOCStudio 调参（可选）

`motor.useMonitoring(Serial)` 把 PID/状态通过串口输出，可在 PC 上跑 [SimpleFOCStudio](https://github.com/simplefoc/SimpleFOCStudio) 实时改 PID 与目标值。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 电机抖动/不转 | 极对数填错 | 用万用表/规格书确认极对数（极数/2），`BLDCMotor(pp)` 传极对数 |
| `initFOC` 报极对数不匹配 | 磁环离传感器太远或极对数错 | 缩短磁环-IC 距离；核对极对数 |
| 控制环卡顿/电机异响 | FreeRTOS tick=100Hz 太低 | menuconfig 设 `CONFIG_FREERTOS_HZ=1000` |
| MCPWM 芯片编译报 channel 错 | `driver.init({1,2,3})` 用错芯片 | MCPWM 芯片传 `driver.init(0)`（timer id），用 `CONFIG_SOC_MCPWM_SUPPORTED` 宏分支 |
| 驱动过热/复位 | 从 3V3 取大电流 | MOSFET 用独立电源（如 12V），共地；`voltage_limit` 留余量 |
| AS5600 读不到 | I2C 引脚/上拉缺失 | 确认 SDA/SCL 引脚与 4.7k 上拉；`AS5600(I2C_NUM_0, sda, scl)` 引脚要对应 |
| 6PWM 驱动 | 用了 `BLDCDriver3PWM` | 半桥驱动用 `BLDCDriver6PWM`（3 对高/低侧） |
| 闭环速度不稳 | PID 没调或 LPF 太强 | 用 SimpleFOCStudio 调 `PID_velocity`；`LPF_velocity.Tf` 典型 0.01~0.05 |

## 参考项目

- 组件文档：`docs/en/motor/foc/esp_simplefoc.rst`
- BLDC 总览：`docs/en/motor/bldc/index.rst`
- 开环示例：`examples/motor/foc_openloop_control/main/main.cpp`
- 闭环示例：`examples/motor/foc_velocity_control/main/main.cpp`
- 力矩旋钮示例：`examples/motor/foc_knob_example/main/main.cpp`
- API 参考（与 Arduino-FOC 一致）：https://docs.simplefoc.com/
