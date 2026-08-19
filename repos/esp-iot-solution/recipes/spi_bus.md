# SPI 总线通信

> **适用摘要**: 使用 `spi_bus` 组件初始化 SPI 主机总线、在总线上添加设备、进行单字节/多字节/16 位/32 位传输。该组件封装了 ESP-IDF `spi_master`，提供更简洁的句柄式接口。

> Evidence: `repos/esp-iot-solution/resources/`, source/examples in `repos/esp-iot-solution/`, and this recipe path `repos/esp-iot-solution/recipes/spi_bus.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "初始化 SPI 总线"
- "SPI 收发数据"
- "spi_bus 设备创建"
- "SPI 多设备共享总线"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+（master）或 v4.4+（release/v2.0） |
| 组件依赖 | `espressif/spi_bus` |
| 参考示例 | 组件 `components/spi_bus/test_apps/`；`examples/common_components/boards/board_common.c`（板级 SPI 总线初始化） |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/spi_bus"
```

### 2. 创建总线与设备

```c
#include "spi_bus.h"

static spi_bus_handle_t s_spi_bus = NULL;
static spi_bus_device_handle_t s_dev = NULL;

void spi_bus_init(void)
{
    spi_config_t bus_cfg = {
        .miso_io_num = 37,
        .mosi_io_num = 35,
        .sclk_io_num = 36,
        .max_transfer_sz = 4096,   // 小于 4096 会自动设为 4096
    };
    // host_id 用 SPI2_HOST 或 SPI3_HOST
    s_spi_bus = spi_bus_create(SPI2_HOST, &bus_cfg);
    assert(s_spi_bus != NULL);

    spi_device_config_t dev_cfg = {
        .cs_io_num = 34,
        .mode = 0,                  // SPI 模式 0~3
        .clock_speed_hz = 10 * 1000 * 1000,   // 10MHz，须为 80MHz 的整除
    };
    s_dev = spi_bus_device_create(s_spi_bus, &dev_cfg);
    assert(s_dev != NULL);
}
```

> 设备无 CS 引脚时，`spi_device_config_t.cs_io_num` 可设为 `NULL_SPI_CS_PIN`（-1）。

### 3. 数据传输

```c
// 单字节传输
uint8_t rx = 0;
spi_bus_transfer_byte(s_dev, 0x9F, &rx);   // 例如读 JEDEC ID

// 多字节传输（可半双工：data_out 或 data_in 之一为 NULL）
uint8_t out[4] = {0x03, 0x00, 0x00, 0x00};
uint8_t in[4] = {0};
spi_bus_transfer_bytes(s_dev, out, in, 4);

// 16 位（MSB 先发）
uint16_t rx16;
spi_bus_transfer_reg16(s_dev, 0xABCD, &rx16);

// 32 位（MSB 先发）
uint32_t rx32;
spi_bus_transfer_reg32(s_dev, 0x12345678, &rx32);
```

### 4. 自定义事务（spi_transaction_t）

当上述封装不够用时，可直接走底层 polling 事务：

```c
spi_transaction_t t = {
    .length = 8 * 4,           // 位数
    .tx_buffer = out,
    .rx_buffer = in,
};
spi_bus_transmit_begin(s_dev, &t);
```

### 5. 释放资源

```c
spi_bus_device_delete(&s_dev);
spi_bus_delete(&s_spi_bus);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `spi_bus_create` 返回 NULL | 引脚已被其他外设占用 | 查芯片引脚矩阵，换用空闲 GPIO |
| 时钟不准 / 数据错位 | `clock_speed_hz` 非 80MHz 整除、`mode` 不对 | 选 `SPI_MASTER_FREQ_*` 系列值；核对从机支持的 mode |
| 多设备 CS 互相干扰 | 多个设备共用同一 CS 或未声明 CS | 每个设备单独 `spi_bus_device_create`，各自 `cs_io_num` |
| `max_transfer_sz` 不足 | 单次传输超过缓冲 | 显式设置足够大的 `max_transfer_sz`（>= 4096） |

## 参考

- 组件头文件：`components/spi_bus/include/spi_bus.h`
- 组件目录：`components/spi_bus/`（含 `test_apps/` 自测代码）
- 真实使用：`examples/common_components/boards/board_common.c`（板级 `spi_bus_create` 初始化 SPI 总线供 LCD 等使用）
- SPI LCD 初始化参考：`docs/en/display/lcd/spi_lcd.rst`（`spi_bus_initialize` 走 IDF 原生 esp_lcd 路径）
