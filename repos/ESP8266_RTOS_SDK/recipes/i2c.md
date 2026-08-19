# I2C 主机读写

> **适用摘要**: 用 I2C 主机驱动（仅 `I2C_NUM_0`）通过命令链（command link）对从设备发起 start/write/read/stop 序列，以 MPU6050 为实例演示寄存器写与读。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/i2c.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "I2C 读写"
- "读传感器"
- "MPU6050"
- "I2C 主机"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/peripherals/i2c/` |
| 端口 | 仅 `I2C_NUM_0`（ESP8266 只有 1 个 I2C，仅 master 模式） |
| 引脚 | 软件实现，SDA/SCL 可任意可用 GPIO；驱动可启用内部上拉 |
| 模式 | 仅 `I2C_MODE_MASTER` |

## 分步说明

### 1. 初始化（改编自示例）

```c
#include "driver/i2c.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

#define I2C_MASTER_SCL_IO  2
#define I2C_MASTER_SDA_IO  14
#define I2C_MASTER_NUM     I2C_NUM_0

static esp_err_t i2c_master_init(void)
{
    i2c_config_t conf = {
        .mode = I2C_MODE_MASTER,
        .sda_io_num = I2C_MASTER_SDA_IO,
        .sda_pullup_en = GPIO_PULLUP_ENABLE,
        .scl_io_num = I2C_MASTER_SCL_IO,
        .scl_pullup_en = GPIO_PULLUP_ENABLE,
        .clk_stretch_tick = 300,    // 时钟拉长时间，约 210us，按需调整
    };
    ESP_ERROR_CHECK(i2c_driver_install(I2C_MASTER_NUM, conf.mode));
    ESP_ERROR_CHECK(i2c_param_config(I2C_MASTER_NUM, &conf));
    return ESP_OK;
}
```

### 2. 写寄存器

```c
#define MPU6050_ADDR        0x68
#define ACK_CHECK_EN        0x1

static esp_err_t i2c_write_reg(uint8_t dev_addr, uint8_t reg, uint8_t *data, size_t len)
{
    i2c_cmd_handle_t cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, (dev_addr << 1) | I2C_MASTER_WRITE, ACK_CHECK_EN);
    i2c_master_write_byte(cmd, reg, ACK_CHECK_EN);
    i2c_master_write(cmd, data, len, ACK_CHECK_EN);
    i2c_master_stop(cmd);
    esp_err_t ret = i2c_master_cmd_begin(I2C_MASTER_NUM, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    return ret;
}
```

### 3. 读寄存器（先写寄存器地址，再 restart 读）

```c
static esp_err_t i2c_read_reg(uint8_t dev_addr, uint8_t reg, uint8_t *data, size_t len)
{
    i2c_cmd_handle_t cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, (dev_addr << 1) | I2C_MASTER_WRITE, ACK_CHECK_EN);
    i2c_master_write_byte(cmd, reg, ACK_CHECK_EN);
    i2c_master_stop(cmd);
    esp_err_t ret = i2c_master_cmd_begin(I2C_MASTER_NUM, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    if (ret != ESP_OK) return ret;

    cmd = i2c_cmd_link_create();
    i2c_master_start(cmd);
    i2c_master_write_byte(cmd, (dev_addr << 1) | I2C_MASTER_READ, ACK_CHECK_EN);
    i2c_master_read(cmd, data, len, I2C_MASTER_LAST_NACK);   // 最后一字节 NACK
    i2c_master_stop(cmd);
    ret = i2c_master_cmd_begin(I2C_MASTER_NUM, cmd, 1000 / portTICK_RATE_MS);
    i2c_cmd_link_delete(cmd);
    return ret;
}
```

### 4. 使用

```c
void app_main(void)
{
    i2c_master_init();
    uint8_t who = 0;
    i2c_read_reg(MPU6050_ADDR, 0x75, &who, 1);   // WHO_AM_I 应为 0x68
    ESP_LOGI("main", "WHO_AM_I = 0x%02x", who);
    // 读 14 字节加速度/温度/陀螺仪
    uint8_t buf[14];
    i2c_read_reg(MPU6050_ADDR, 0x3B, buf, 14);
    // ...
    i2c_driver_delete(I2C_MASTER_NUM);
}
```

### 5. 关键 API / 类型

| API / 类型 | 作用 |
|---|---|
| `i2c_param_config(i2c_num, &i2c_config_t)` | 配置引脚/上拉/拉伸 |
| `i2c_driver_install(i2c_num, I2C_MODE_MASTER)` | 安装主机驱动 |
| `i2c_driver_delete(i2c_num)` | 卸载 |
| `i2c_cmd_link_create()` / `i2c_cmd_link_delete(h)` | 命令链生命周期 |
| `i2c_master_start / stop(h)` | 起停信号 |
| `i2c_master_write_byte(h, byte, ack_en)` | 写一字节 |
| `i2c_master_write(h, data, len, ack_en)` | 写多字节 |
| `i2c_master_read_byte(h, &byte, ack)` / `i2c_master_read(h, buf, len, ack)` | 读 |
| `i2c_master_cmd_begin(i2c_num, h, ticks)` | 提交执行整条命令链 |
| `i2c_ack_type_t` | `I2C_MASTER_ACK / NACK / LAST_NACK` |
| `i2c_config_t` 字段：`mode`、`sda_io_num`、`scl_io_num`、`sda_pullup_en`、`scl_pullup_en`、`clk_stretch_tick` | 配置 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_FAIL`（无 ACK） | 从设备地址错 / 未上电 / 接线 | 核对 7 位地址左移 1；检查 SDA/SCL/电源 |
| 读最后一字节卡死 | 用错 ACK 类型 | 最后一字节用 `I2C_MASTER_LAST_NACK` |
| 重复 init 报错 | 未 delete 再 install | 先 `i2c_driver_delete` |
| 想用 slave 模式 | 本仓库不支持 | ESP8266 I2C 仅 master |
| 速率上不去 | `clk_stretch_tick` 太小 | 调大拉伸时间 |
| 端口写错 | 用了 `I2C_NUM_1` | 仅 `I2C_NUM_0` |

## 参考

- `examples/peripherals/i2c/` — 官方 I2C 示例（`main/user_main.c`，MPU6050）
- `components/esp8266/include/driver/i2c.h` — 完整原型与枚举
- `docs/en/api-reference/peripherals/i2c.rst`
