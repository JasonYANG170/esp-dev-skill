# I2C 主机（新总线 API）

> **适用摘要**: 使用 ESP-IDF v5.x 新总线式句柄 API（`i2c_new_master_bus` + `i2c_master_bus_add_device`）创建 I2C 主机并读写传感器寄存器。

## 触发意图

- "配置 I2C"
- "I2C 主机"
- "读取 MPU9250 / BMP280 / SHT30"
- "i2c_new_master_bus"
- "i2c_master_transmit_receive"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `esp_driver_i2c` |
| 头文件 | `driver/i2c_master.h` |
| 参考 | `examples/peripherals/i2c/i2c_basic` |

## 分步说明

### 初始化总线与设备（适配自 i2c_basic）

```c
#include "driver/i2c_master.h"
#include "esp_log.h"

#define I2C_MASTER_SDA_IO    8
#define I2C_MASTER_SCL_IO    9
#define I2C_MASTER_FREQ_HZ   100000
#define SENSOR_ADDR          0x68   /* MPU9250 */

static i2c_master_bus_handle_t s_bus;
static i2c_master_dev_handle_t s_dev;

static void i2c_init(void)
{
    i2c_master_bus_config_t bus_cfg = {
        .i2c_port = I2C_NUM_0,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .clk_source = I2C_CLK_SRC_DEFAULT,
        .glitch_ignore_cnt = 7,
        .flags.enable_internal_pullup = true,
    };
    ESP_ERROR_CHECK(i2c_new_master_bus(&bus_cfg, &s_bus));

    i2c_device_config_t dev_cfg = {
        .dev_addr_length = I2C_ADDR_BIT_LEN_7,
        .device_address = SENSOR_ADDR,
        .scl_speed_hz = I2C_MASTER_FREQ_HZ,
    };
    ESP_ERROR_CHECK(i2c_master_bus_add_device(s_bus, &dev_cfg, &s_dev));
}
```

### 读寄存器（先写寄存器地址再读）

```c
static esp_err_t sensor_read_reg(uint8_t reg, uint8_t *val)
{
    return i2c_master_transmit_receive(s_dev, &reg, 1, val, 1, -1);
}
```

### 写寄存器

```c
static esp_err_t sensor_write_reg(uint8_t reg, uint8_t val)
{
    uint8_t buf[2] = { reg, val };
    return i2c_master_transmit(s_dev, buf, sizeof(buf), -1);
}
```

### 扫描总线（探测地址）

```c
/* 探测某地址是否存在应答 */
esp_err_t r = i2c_master_probe(s_bus, addr, -1);
```

### 反初始化

```c
i2c_master_bus_rm_device(s_dev);
i2c_del_master_bus(s_bus);
```

### 关键 API

```c
esp_err_t i2c_new_master_bus(const i2c_master_bus_config_t *bus_config,
                             i2c_master_bus_handle_t *ret_bus_handle);
esp_err_t i2c_master_bus_add_device(i2c_master_bus_handle_t bus_handle,
                                    const i2c_device_config_t *dev_config,
                                    i2c_master_dev_handle_t *ret_handle);
esp_err_t i2c_master_bus_rm_device(i2c_master_dev_handle_t handle);
esp_err_t i2c_master_transmit(i2c_master_dev_handle_t i2c_dev,
                              const uint8_t *write_buffer, size_t write_size,
                              int xfer_timeout_ms);
esp_err_t i2c_master_receive(i2c_master_dev_handle_t i2c_dev,
                             uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
esp_err_t i2c_master_transmit_receive(i2c_master_dev_handle_t i2c_dev,
                              const uint8_t *write_buffer, size_t write_size,
                              uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
esp_err_t i2c_master_probe(i2c_master_bus_handle_t bus_handle, uint16_t address,
                           int xfer_timeout_ms);
```
> `xfer_timeout_ms` 传 `-1` 表示 `portMAX_DELAY`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 找不到 `i2c_param_config` | 用了 v4 旧 API | 改用 `i2c_new_master_bus` + `i2c_master_bus_add_device` |
| `ESP_ERR_TIMEOUT` | 无应答 | 检查地址、上拉、SDA/SCL 接线与速率 |
| 同一地址多设备冲突 | 多器件挂在同一总线 | 每个器件 `i2c_master_bus_add_device` 各得独立 handle |
| 内部上拉不足 | 上拉太弱 | `flags.enable_internal_pullup=true` 或外加 4.7k 上拉 |
| 速率 400k 报错 | 线长/容性大 | 降到 100k 或缩短走线 |

## 参考

- `examples/peripherals/i2c/i2c_basic` — 总线初始化、读写、deinit
- `examples/peripherals/i2c/i2c_eeprom` — EEPROM 读写（含分页）
- ESP-IDF `components/esp_driver_i2c/include/driver/i2c_master.h`
