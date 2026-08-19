# 功率计量（BL0937 / BL0942 / INA236）

> **适用摘要**: 使用 `power_measure` 组件通过工厂模式创建功率计量设备（BL0937 脉冲型、BL0942 UART/SPI 型、INA236 I2C 型），读取电压、电流、有功功率、功率因数、电能。三种芯片各有独立 config 结构体与 `power_measure_new_xxx_device` 工厂函数。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-iot-solution/resources/`, source/examples in `repos/esp-iot-solution/`, and this recipe path `repos/esp-iot-solution/recipes/power_measure.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "功率 / 电参数测量"
- "BL0937 / BL0942 / INA236"
- "电压电流功率采集"
- "智能插座计量"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+ |
| 组件依赖 | `espressif/power_measure` |
| 参考示例 | `examples/sensors/power_measure/main/power_measure_example.c` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/power_measure"
```

> 示例通过 Kconfig（`CONFIG_POWER_MEASURE_CHIP_BL0937` / `_BL0942` / `_INA236`）选择芯片，对应 menuconfig 中的选项。

### 2. 公共配置

```c
#include "power_measure.h"

power_measure_config_t common_config = {
    .overcurrent = 15,                 // 过流阈值（A）
    .undervoltage = 180,               // 欠压阈值（V）
    .overvoltage  = 260,               // 过压阈值（V）（示例中使用，结构体实际字段以上述头文件为准）
    .enable_energy_detection = true,   // 启用电能累计
};
static power_measure_handle_t handle = NULL;
```

> 注：`power_measure_config_t` 真实字段为 `overcurrent`、`undervoltage`、`enable_energy_detection`（见 `components/sensors/power_measure/include/power_measure.h`）。示例 main 中出现的 `overvoltage` 字段是示例自定义扩展，以头文件为准。

### 3a. INA236（I2C）—— 需先建 i2c_bus

```c
#include "i2c_bus.h"
#include "power_measure_ina236.h"

static i2c_bus_handle_t s_i2c_bus = NULL;

static esp_err_t init_i2c_bus(void)
{
    i2c_config_t conf = {
        .mode = I2C_MODE_MASTER,
        .sda_io_num = CONFIG_INA236_I2C_SDA_GPIO,
        .sda_pullup_en = true,
        .scl_io_num = CONFIG_INA236_I2C_SCL_GPIO,
        .scl_pullup_en = true,
        .master.clk_speed = 100000,
    };
    s_i2c_bus = i2c_bus_create(I2C_NUM_0, &conf);
    return s_i2c_bus ? ESP_OK : ESP_FAIL;
}

static esp_err_t init_ina236(void)
{
    common_config.enable_energy_detection = false;   // INA236 不支持电能检测
    power_measure_ina236_config_t ina236_config = {
        .i2c_bus = s_i2c_bus,
        .i2c_addr = 0x41,
        .alert_en = false,
        .alert_pin = -1,
        .alert_cb = NULL,
    };
    return power_measure_new_ina236_device(&common_config, &ina236_config, &handle);
}
```

### 3b. BL0937（GPIO 脉冲）

```c
#include "power_measure_bl0937.h"

power_measure_bl0937_config_t bl0937_config = {
    .sel_gpio = CONFIG_BL0937_SEL_GPIO,
    .cf1_gpio = CONFIG_BL0937_CF1_GPIO,
    .cf_gpio  = CONFIG_BL0937_CF_GPIO,
    .pin_mode = 0,
    .sampling_resistor = 0.001f,    // 1mΩ
    .divider_resistor  = 2010.0f,   // 分压电阻
    .ki = 1.0f, .ku = 1.0f, .kp = 1.0f,
};
power_measure_new_bl0937_device(&common_config, &bl0937_config, &handle);
```

### 3c. BL0942（UART 或 SPI）

```c
#include "power_measure_bl0942.h"

power_measure_bl0942_config_t bl0942_config = {
    .addr = CONFIG_BL0942_DEVICE_ADDRESS,
    .shunt_resistor = 0.001f,
    .divider_ratio = 3760.0f,
    .use_spi = 0,                   // 0 = UART，1 = SPI
    .uart = {
        .uart_num = UART_NUM_1,
        .tx_io = CONFIG_BL0942_UART_TX_GPIO,
        .rx_io = CONFIG_BL0942_UART_RX_GPIO,
        .sel_io = -1,
        .baud = CONFIG_BL0942_UART_BAUD_RATE,
    },
};
power_measure_new_bl0942_device(&common_config, &bl0942_config, &handle);
```

### 4. 读取电参数

```c
float voltage, current, power, pf, energy;

power_measure_get_voltage(handle, &voltage);          // V
power_measure_get_current(handle, &current);          // A
power_measure_get_active_power(handle, &power);       // W
power_measure_get_power_factor(handle, &pf);          // 0~1
power_measure_get_energy(handle, &energy);            // kWh（BL0937/BL0942）
power_measure_get_apparent_power(handle, &apparent);  // VA

ESP_LOGI(TAG, "U=%.2fV I=%.2fA P=%.2fW PF=%.2f", voltage, current, power, pf);
```

### 5. 校准与电能复位

```c
power_measure_calibrate_voltage(handle, 220.0f);   // 用已知电压校准
power_measure_calibrate_current(handle, 1.0f);
power_measure_calibrate_power(handle, 220.0f);
power_measure_reset_energy_calculation(handle);
```

### 6. 释放

```c
power_measure_delete(handle);
```

## 各芯片适用与差异

| 芯片 | 接口 | 电能累计 | 典型应用 |
|---|---|---|---|
| BL0937 | GPIO 脉冲（CF/CF1/SEL） | 支持（软件） | 低成本智能插座 |
| BL0942 | UART 或 SPI | 支持 | 智能插座、配电 |
| INA236 | I2C | 不支持 | 直流低压侧电流检测 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| INA236 创建失败 | 未先建 i2c_bus | 先 `i2c_bus_create`，再把句柄传入 `i2c_bus` 字段 |
| 数值偏差大 | 校准系数（ki/ku/kp）未设或采样电阻不符 | 用 `power_measure_calibrate_*` 标定；核对硬件电阻值 |
| `get_energy` 始终 0 | `enable_energy_detection=false` 或芯片不支持 | BL0937/BL0942 设 `true`；INA236 不支持电能 |
| BL0942 UART 无响应 | 波特率 / 地址不匹配 | 确认 `baud` 与模块配置；`addr` 用十进制地址 |
| 编译报缺结构体字段 | 拷贝示例时混入示例自定义字段 | 以 `components/sensors/power_measure/include/*.h` 头文件为准 |

## 参考

- 组件头文件：`components/sensors/power_measure/include/power_measure.h`、`power_measure_bl0937.h`、`power_measure_bl0942.h`、`power_measure_ina236.h`
- 在线文档：`docs/en/sensors/power_measure.rst`
- 真实示例：`examples/sensors/power_measure/main/power_measure_example.c`
