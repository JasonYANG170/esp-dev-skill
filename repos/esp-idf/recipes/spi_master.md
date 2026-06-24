# SPI 主机

> **适用摘要**: 初始化 SPI 主机总线（`spi_bus_initialize`）、添加设备（`spi_bus_add_device`）、阻塞/排队传输。

## 触发意图

- "配置 SPI"
- "SPI 主机"
- "驱动 SPI LCD / Flash / 传感器"
- "spi_bus_initialize"
- "spi_device_transmit"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `esp_driver_spi` |
| 头文件 | `driver/spi_master.h` |
| 参考 | `examples/peripherals/spi_master/hd_eeprom`、`examples/peripherals/spi_master/lcd` |

## 分步说明

### 1. 初始化总线（适配自 spi_master/hd_eeprom）

```c
#include "driver/spi_master.h"

#define EEPROM_HOST   SPI2_HOST
#define PIN_NUM_MISO  13
#define PIN_NUM_MOSI  12
#define PIN_NUM_CLK   11
#define PIN_NUM_CS    10

spi_bus_config_t buscfg = {
    .miso_io_num = PIN_NUM_MISO,
    .mosi_io_num = PIN_NUM_MOSI,
    .sclk_io_num = PIN_NUM_CLK,
    .quadwp_io_num = -1,
    .quadhd_io_num = -1,
    .max_transfer_sz = 32,
};
ESP_ERROR_CHECK(spi_bus_initialize(EEPROM_HOST, &buscfg, SPI_DMA_CH_AUTO));
```

### 2. 添加从设备

```c
spi_device_interface_config_t devcfg = {
    .clock_speed_hz = 10 * 1000 * 1000,   /* 10 MHz */
    .mode = 0,                            /* SPI mode 0 */
    .spics_io_num = PIN_NUM_CS,
    .queue_size = 7,
    /* .flags = SPI_DEVICE_HALFDUPLEX, 如需半双工 */
};
spi_device_handle_t spi;
ESP_ERROR_CHECK(spi_bus_add_device(EEPROM_HOST, &devcfg, &spi));
```

### 3. 阻塞式传输

```c
uint8_t tx[4] = { 0x01, 0x02, 0x03, 0x04 };
uint8_t rx[4] = {0};
spi_transaction_t t = {
    .length = 4 * 8,            /* 单位：bit */
    .tx_buffer = tx,
    .rx_buffer = rx,
};
ESP_ERROR_CHECK(spi_device_transmit(spi, &t));
```

### 4. 排队传输（高吞吐，多笔并发）

```c
spi_transaction_t t = { .length = len * 8, .tx_buffer = buf };
ESP_ERROR_CHECK(spi_device_queue_trans(spi, &t, portMAX_DELAY));

spi_transaction_t *ret_t;
ESP_ERROR_CHECK(spi_device_get_trans_result(spi, &ret_t, portMAX_DELAY));
```

### 关键 API

```c
esp_err_t spi_bus_initialize(spi_host_device_t host, const spi_bus_config_t *bus_config,
                             spi_dma_chan_t dma_chan);
esp_err_t spi_bus_add_device(spi_host_device_t host_id,
                             const spi_device_interface_config_t *dev_config,
                             spi_device_handle_t *handle);
esp_err_t spi_bus_remove_device(spi_device_handle_t handle);
esp_err_t spi_device_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);
esp_err_t spi_device_queue_trans(spi_device_handle_t handle, spi_transaction_t *trans_desc,
                                 uint32_t ticks_to_wait);
esp_err_t spi_device_get_trans_result(spi_device_handle_t handle, spi_transaction_t **trans_desc,
                                      uint32_t ticks_to_wait);
esp_err_t spi_device_polling_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);
```

主机枚举：`SPI1_HOST`（主 Flash，勿占用）/ `SPI2_HOST` / `SPI3_HOST`（视芯片）。
DMA：`SPI_DMA_CH_AUTO` 让驱动自动分配。

> `length` 字段单位是 **bit**，不是 byte：4 字节 → `.length = 4 * 8`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `SPI1_HOST` 不可用 | SPI1 接主 Flash | 用 `SPI2_HOST`/`SPI3_HOST` |
| 传输字节数不对 | `length` 写成了字节 | `length = N * 8`（bit） |
| `WIFI/LWIP` 冲突 | SPI DMA 通道被占 | 用 `SPI_DMA_CH_AUTO` |
| 时钟达不到设定值 | 桥接分频 | 查 `spi_device_get_actual_freq`；选芯片支持的档位 |
| MISO 读到全 0 | 接线/模式错误 | 核对 mode 0/3、CPOL/CPHA、CS 与 MISO 接线 |

## 参考

- `examples/peripherals/spi_master/hd_eeprom` — EEPROM 半双工读写
- `examples/peripherals/spi_master/lcd` — SPI LCD（含 DMA）
- ESP-IDF `components/esp_driver_spi/include/driver/spi_master.h`
