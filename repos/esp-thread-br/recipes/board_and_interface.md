# 板型与通信接口配置

> **适用摘要**: 选择板型 (DEV_KIT / STANDALONE / M5STACK_CORES3)、配置 UART/SPI 通信接口与 RCP Reset/Boot 引脚、Standalone 模组接线。

## 触发意图
- "怎么选板型"
- "配置 SPI 接口连 RCP"
- "Standalone 接线"
- "RCP Reset Boot 引脚"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考 | `examples/common/thread_border_router/Kconfig.projbuild`、`examples/basic_thread_border_router/main/esp_ot_config.h`、`README_standalone_RCP.md` |

## 分步说明

### 1. 选择板型

`Kconfig.projbuild` 定义三种板型（决定默认引脚）：

```bash
idf.py menuconfig
# ESP Thread Border Router Example -> Border router board type
#   (X) Standalone dev kits           # ESP_BR_BOARD_STANDALONE
#   ( ) Border router dev kit         # ESP_BR_BOARD_DEV_KIT
#   ( ) Border router M5Stack CoreS3  # ESP_BR_BOARD_M5STACK_CORES3
```

| 配置项 | DEV_KIT | STANDALONE | M5STACK_CORES3 |
|---|---|---|---|
| `PIN_TO_RCP_RESET` | 7 | 7 | 7 |
| `PIN_TO_RCP_BOOT` | 8 | 8 | 18 |
| `PIN_TO_RCP_TX` | 17 | 4 | 10 |
| `PIN_TO_RCP_RX` | 18 | 5 | 17 |
| `PIN_TO_RCP_CS` | 10 | 20 | 13 |
| `PIN_TO_RCP_SCLK` | 12 | 22 | 36 |
| `PIN_TO_RCP_MISO` | 13 | 23 | 35 |
| `PIN_TO_RCP_MOSI` | 11 | 21 | 37 |

### 2. 选择 RCP 目标芯片

```bash
# ESP Thread Border Router Example -> Border router RCP target
#   (X) ESP32-H2   # CONFIG_ESP_BR_H2_TARGET
#   ( ) ESP32-C6   # CONFIG_ESP_BR_C6_TARGET
```

这会决定 `ESP_BR_RCP_TARGET_ID`（`ESP32H2_CHIP` 或 `ESP32C6_CHIP`），影响 `ESP_OPENTHREAD_RCP_UPDATE_CONFIG()`。

### 3. UART 模式（默认）

`sdkconfig.defaults` 默认 `CONFIG_OPENTHREAD_RADIO_SPINEL_UART=y`，对应 `esp_ot_config.h`：

```c
#define ESP_OPENTHREAD_DEFAULT_RADIO_CONFIG()              \
    {                                                      \
        .radio_mode = RADIO_MODE_UART_RCP,                 \
        .radio_uart_config = {                             \
            .port = 1,                                     \
            .uart_config = {                               \
                .baud_rate = 460800,                       \
                .data_bits = UART_DATA_8_BITS,             \
                .parity   = UART_PARITY_DISABLE,           \
                .stop_bits = UART_STOP_BITS_1,             \
                .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,     \
                .source_clk = UART_SCLK_DEFAULT,           \
            },                                             \
            .rx_pin = CONFIG_PIN_TO_RCP_TX,                \
            .tx_pin = CONFIG_PIN_TO_RCP_RX,                \
        },                                                 \
    }
```

### 4. SPI 模式（两端同步）

启用 SPI 需在 `ot_rcp` 与 BR 两端同时改：

```bash
# ot_rcp:   CONFIG_OPENTHREAD_RCP_SPI=y
# BR:       CONFIG_OPENTHREAD_RADIO_SPINEL_SPI=y
```

对应 `ESP_OPENTHREAD_DEFAULT_RADIO_CONFIG()`：

```c
.radio_mode = RADIO_MODE_SPI_RCP,
.radio_spi_config = {
    .host_device = SPI2_HOST,
    .dma_channel = 2,
    .spi_interface = {
        .mosi_io_num = CONFIG_PIN_TO_RCP_MOSI,
        .miso_io_num = CONFIG_PIN_TO_RCP_MISO,
        .sclk_io_num = CONFIG_PIN_TO_RCP_SCLK,
        .quadwp_io_num = -1, .quadhd_io_num = -1,
    },
    .spi_device = {
        .cs_ena_pretrans = 2, .input_delay_ns = 100,
        .mode = 0, .clock_speed_hz = 2500 * 1000,
        .spics_io_num = CONFIG_PIN_TO_RCP_CS, .queue_size = 5,
    },
    .intr_pin = CONFIG_PIN_TO_RCP_BOOT,
},
```

### 5. Standalone UART 接线（参考 README_standalone_RCP.md）

| 主控 | RCP (H2/C6) |
|---|---|
| GND | G |
| 5V | 5V |
| GPIO4 (UART RX) | TX |
| GPIO5 (UART TX) | RX |
| GPIO7 | RST |
| GPIO8 | GPIO9 (BOOT) |

### 6. Standalone SPI 接线

| 主控 | RCP (H2/C6) |
|---|---|
| GND | G |
| GPIO7 | RST |
| GPIO8 (SPI INTR) | GPIO9 (BOOT) |
| GPIO20 (SPI CS) | GPIO2 |
| GPIO21 (SPI MOSI) | GPIO3 |
| GPIO22 (SPI CLK) | GPIO0 |
| GPIO23 (SPI MISO) | GPIO1 |

### 7. C5 Standalone 特别覆盖

`sdkconfig.defaults.esp32c5`：
```bash
CONFIG_ESP_BR_BOARD_STANDALONE=y
CONFIG_ESP_CONSOLE_UART_DEFAULT=y
```
> ESP32-C5 仅一根天线，Wi-Fi 与 Thread 不能同时收发，故不使用片上 15.4，必须外接 RCP。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| SPI 模式 Spinel 超时 | 只在 BR 端开 SPI | ot_rcp 端同步 `OPENTHREAD_RCP_SPI=y` |
| ESP32-S3 UART 通信不稳 | 用了 GPIO17/18 | 改 GPIO4/GPIO5（见 qa.rst 5.2） |
| RCP 升级报芯片不匹配 | `ESP_BR_RCP_TARGET_ID` 与实际 RCP 不符 | 选对 `ESP_BR_H2_TARGET` / `ESP_BR_C6_TARGET` |
| M5Stack 改 Unit H2 后不工作 | 引脚与 AUTO_UPDATE 未调 | `PIN_TO_RCP_TX=18`、`PIN_TO_RCP_RX=17`，并 `AUTO_UPDATE_RCP=n` |

## 参考
- `examples/common/thread_border_router/Kconfig.projbuild`
- `examples/basic_thread_border_router/main/esp_ot_config.h`
- `examples/basic_thread_border_router/README_standalone_RCP.md`
- `docs/en/qa.rst`（5.2 Host 与 RCP 无法通信）
