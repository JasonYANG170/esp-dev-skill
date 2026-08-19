# I2C 总线通信

> **适用摘要**: 使用 `i2c_bus` 组件初始化 I2C 主机总线、创建总线上的设��句柄、进行字节/多字节/位级寄存器读写与总线扫描。`i2c_bus` 在 ESP-IDF >= 5.3 默认走新 `i2c_master` 驱动，旧版本走 `driver/i2c.h`，由 `CONFIG_I2C_BUS_BACKWARD_CONFIG` 控制。

> Evidence: `repos/esp-iot-solution/resources/`, source/examples in `repos/esp-iot-solution/`, and this recipe path `repos/esp-iot-solution/recipes/i2c_bus.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "初始化 I2C 总线"
- "I2C 读写寄存器"
- "i2c_bus 设备创建"
- "扫描 I2C 设备地址"
- "多个 I2C 设备挂同一总线"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+（master 分支）或 v4.4+（release/v2.0） |
| 组件依赖 | `espressif/i2c_bus` |
| 参考示例 | `examples/sensors/power_measure/main/power_measure_example.c` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/i2c_bus"
```

### 2. 创建总线与设备

```c
#include "i2c_bus.h"

#define I2C_MASTER_SCL_IO    9
#define I2C_MASTER_SDA_IO    8
#define I2C_MASTER_NUM       I2C_NUM_0
#define I2C_MASTER_FREQ_HZ   100000
#define SENSOR_ADDR          0x44    // 例如 SHT3x

static i2c_bus_handle_t s_i2c_bus = NULL;
static i2c_bus_device_handle_t s_dev = NULL;

void i2c_bus_init(void)
{
    i2c_config_t conf = {
        .mode = I2C_MODE_MASTER,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .sda_pullup_en = true,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .scl_pullup_en = true,
        .master.clk_speed = I2C_MASTER_FREQ_HZ,
    };
    s_i2c_bus = i2c_bus_create(I2C_MASTER_NUM, &conf);
    assert(s_i2c_bus != NULL);

    // 第三个参数 clk_speed=0 表示沿用总线速率
    s_dev = i2c_bus_device_create(s_i2c_bus, SENSOR_ADDR, 0);
    assert(s_dev != NULL);
}
```

### 3. 寄存器读写

```c
// 读单字节寄存器
uint8_t reg_val = 0;
i2c_bus_read_byte(s_dev, 0x00, &reg_val);

// 写单字节寄存器
i2c_bus_write_byte(s_dev, 0x06, 0x2C);

// 读多字节
uint8_t buf[6];
i2c_bus_read_bytes(s_dev, 0x2C, 6, buf);

// 写多字节
uint8_t out[2] = {0x24, 0x00};
i2c_bus_write_bytes(s_dev, 0x2C, 2, out);

// 设备无内部寄存器地址时，mem_address 传 NULL_I2C_MEM_ADDR
i2c_bus_read_byte(s_dev, NULL_I2C_MEM_ADDR, &reg_val);

// 16 位寄存器地址设备
i2c_bus_read_reg16(s_dev, 0x0000, 2, buf);
i2c_bus_write_reg16(s_dev, 0x0001, 2, out);
```

### 4. 总线扫描

```c
uint8_t addr_list[16];
uint8_t found = i2c_bus_scan(s_i2c_bus, addr_list, sizeof(addr_list));
ESP_LOGI(TAG, "found %u devices:", found);
for (int i = 0; i < found; i++) {
    ESP_LOGI(TAG, "  0x%02X", addr_list[i]);
}
```

### 5. 同一总线上挂多个不同速率设备

启用 `CONFIG_I2C_BUS_DYNAMIC_CONFIG`（默认开启）后，可在 `i2c_bus_device_create` 的第三参数为每个设备指定独立 clk_speed，组件会在每次传输前动态重装驱动。

```c
i2c_bus_device_handle_t dev_fast = i2c_bus_device_create(s_i2c_bus, 0x50, 400000);
i2c_bus_device_handle_t dev_slow = i2c_bus_device_create(s_i2c_bus, 0x48, 100000);
```

### 6. 释放资源

```c
i2c_bus_device_delete(&s_dev);
i2c_bus_delete(&s_i2c_bus);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `i2c_bus_create` 返回 NULL | SDA/SCL 引脚冲突或上拉未使能 | 确认引脚未被占用，硬件加 4.7k 上拉或 `sda_pullup_en=true` |
| `ESP_FAIL`（slave not ACK） | 设备地址错误或接线松动 | 用 `i2c_bus_scan` 确认地址；检查接线与电源 |
| 类型 `i2c_config_t` 重定义 | 在 IDF >= 5.3 同时 include `driver/i2c.h` 与 `i2c_bus.h` | 只 include `i2c_bus.h`；若必须用旧驱动，menuconfig 启用 `CONFIG_I2C_BUS_BACKWARD_CONFIG=y` |
| 多设备速率冲突 | `CONFIG_I2C_BUS_DYNAMIC_CONFIG` 未启用 | menuconfig → I2C Bus Options → enable dynamic configuration |
| 读 16 位寄存器地址设备乱码 | 用了 `i2c_bus_read_byte`（8 位地址） | 改用 `i2c_bus_read_reg16` / `i2c_bus_write_reg16` |

## 参考

- 组件头文件：`components/i2c_bus/include/i2c_bus.h`
- 组件 Kconfig：`components/i2c_bus/Kconfig`
- 真实示例：`examples/sensors/power_measure/main/power_measure_example.c`（INA236 经 `i2c_bus_create` 建总线）
