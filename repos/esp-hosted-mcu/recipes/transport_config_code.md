# 运行期配置传输介质（esp_hosted_*_set_config）

> **适用摘要**: 不依赖 menuconfig 静态选择，而是在代码里通过 `esp_hosted_<transport>_set_config()` 自定义 SDIO / SPI / SPI-HD / UART 的引脚、时钟、队列大小等参数，再调用 `esp_hosted_init()`。适用于引脚重映射、运行期切换、或在非 ESP host 上移植。

## 触发意图

- "运行时改 ESP-Hosted 引脚"
- "不用 menuconfig 配置 SPI/SDIO"
- "esp_hosted_sdio_set_config"
- "INIT_DEFAULT_HOST_SPI_CONFIG"
- "自定义 ESP-Hosted 传输"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_hosted.h"`（已包含 `esp_hosted_transport_config.h`） |
| 时序 | 必须在 `esp_hosted_init()` **之前**调用 set_config |
| 参考例程 | `examples/host_transport_config/` |

## 分步说明

### 1. 关键类型（来自 `host/api/include/esp_hosted_transport_config.h`）

```c
// 引脚（port + pin），port 在 ESP 上一般为 0
typedef struct { void *port; int pin; } gpio_pin_t;

// 顶层配置：transport_in_use 选 SDIO/SPI_HD/SPI/UART，union 持有对应结构
struct esp_hosted_transport_config {
    uint8_t transport_in_use;
    union {
        struct esp_hosted_sdio_config   sdio;
        struct esp_hosted_spi_hd_config spi_hd;
        struct esp_hosted_spi_config    spi;
        struct esp_hosted_uart_config   uart;
    } u;
};
```

各结构的关键字段（节选，全部字段见头文件）：

- `esp_hosted_sdio_config`：`clock_freq_khz`、`bus_width`、`slot`、`pin_clk/cmd/d0..d3/reset`、`rx_mode`、`block_mode`、`iomux_enable`、`tx_queue_size`、`rx_queue_size`
- `esp_hosted_spi_config`（全双工）：`pin_mosi/miso/sclk/cs/handshake/data_ready/reset`、`mode`、`clk_mhz`、`tx/rx_queue_size`
- `esp_hosted_spi_hd_config`（半双工）：`num_data_lines`、`pin_cs/clk/data_ready/d0..d3/reset`、`clk_mhz`、`mode`、`checksum_enable`、`num_command/address/dummy_bits`、`tx/rx_queue_size`
- `esp_hosted_uart_config`：`port`、`pin_tx/rx/reset`、`num_data_bits`、`parity`、`stop_bits`、`flow_ctrl`、`clk_src`、`checksum_enable`、`baud_rate`、`tx/rx_queue_size`

### 2. 便捷宏（取平台默认配置）

```c
#define INIT_DEFAULT_HOST_SDIO_CONFIG()         esp_hosted_get_default_sdio_config()
#define INIT_DEFAULT_HOST_SDIO_IOMUX_CONFIG()   esp_hosted_get_default_sdio_iomux_config()
#define INIT_DEFAULT_HOST_SPI_HD_CONFIG()       esp_hosted_get_default_spi_hd_config()
#define INIT_DEFAULT_HOST_SPI_CONFIG()          esp_hosted_get_default_spi_config()
#define INIT_DEFAULT_HOST_UART_CONFIG()         esp_hosted_get_default_uart_config()
```

### 3. SDIO 自定义示例

```c
#include "esp_hosted.h"

// 1. 取默认配置（含平台默认引脚/时钟/队列）
struct esp_hosted_sdio_config config = INIT_DEFAULT_HOST_SDIO_CONFIG();

// 2. 改成自定义
config.clock_freq_khz = 25000;     // 25 MHz
config.bus_width      = 4;         // 4-bit
config.tx_queue_size  = 20;
config.rx_queue_size  = 20;

// 3. 应用（返回值必须检查——函数标注了 warn_unused_result）
esp_hosted_transport_err_t ret = esp_hosted_sdio_set_config(&config);
if (ret != ESP_TRANSPORT_OK) {
    ESP_LOGE(TAG, "SDIO set_config failed: %d", ret);
    return;
}

// 4. 再初始化 ESP-Hosted
esp_hosted_init();
esp_hosted_connect_to_slave();
```

### 4. SPI 全双工自定义示例

```c
struct esp_hosted_spi_config spi_cfg = INIT_DEFAULT_HOST_SPI_CONFIG();
spi_cfg.clk_mhz        = 10;       // 先用低时钟验证
spi_cfg.mode           = 0;
spi_cfg.checksum_enable = true;    // 注意：SPI 无硬件检错，建议启用
// pin_miso/mosi/sclk/cs/handshake/data_ready/reset 可在此覆盖默认

esp_hosted_transport_err_t ret = esp_hosted_spi_set_config(&spi_cfg);
if (ret != ESP_TRANSPORT_OK) { /* handle */ }

esp_hosted_init();
esp_hosted_connect_to_slave();
```

### 5. UART 自定义示例

```c
struct esp_hosted_uart_config uart_cfg = INIT_DEFAULT_HOST_UART_CONFIG();
uart_cfg.baud_rate       = 921600;
uart_cfg.checksum_enable = true;
// pin_tx / pin_rx / pin_reset 按板子改

esp_hosted_transport_err_t ret = esp_hosted_uart_set_config(&uart_cfg);
if (ret != ESP_TRANSPORT_OK) { /* handle */ }

esp_hosted_init();
esp_hosted_connect_to_slave();
```

### 6. 其他相关 API

```c
// 取回当前配置指针
esp_hosted_transport_err_t esp_hosted_transport_get_config(struct esp_hosted_transport_config **config);
// 取 reset 引脚配置
esp_hosted_transport_err_t esp_hosted_transport_get_reset_config(gpio_pin_t *pin_config);
// 一次性设全部为平台默认
esp_hosted_transport_err_t esp_hosted_transport_set_default_config(void);
bool esp_hosted_transport_is_config_valid(void);

// SDIO IO_MUX 专用
esp_hosted_transport_err_t esp_hosted_sdio_iomux_set_config(struct esp_hosted_sdio_config *config);
// SPI-HD 2 线专用
esp_hosted_transport_err_t esp_hosted_spi_hd_2lines_set_config(struct esp_hosted_spi_hd_config *config);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译警告 `ignoring return value` | set_config 标了 `warn_unused_result` | 检查并处理返回的 `esp_hosted_transport_err_t` |
| 配置不生效 | 在 `esp_hosted_init()` 之后才 set_config | set_config 必须在 init 之前 |
| 自定义引脚通信失败 | 引脚不支持该外设功能 | 优先选 IO_MUX 引脚；SDIO 在 ESP32 仅支持固定 IO_MUX |
| 默认宏未定义 | 未包含 `esp_hosted.h` | `esp_hosted.h` 会引入 `esp_hosted_transport_config.h` |
| SDIO 上拉缺失仍自定义时钟 | 高时钟 + 无上拉 = 不稳 | 上拉到位后再提时钟 |

## 参考

- `host/api/include/esp_hosted_transport_config.h`（全部结构与函数签名）
- `examples/host_transport_config/`（每种传输的引脚表与代码范式）
- `docs/spi_full_duplex.md`、`docs/sdio.md`、`docs/uart.md`、`docs/spi_half_duplex.md`
