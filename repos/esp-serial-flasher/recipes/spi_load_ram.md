# 通过 SPI 接口下载 RAM（仅 RAM 下载）

> **适用摘要**: 用 ESP32 主机的 SPI 外设（`esp32_spi_port`）+ `esp_loader_init_spi()` 把程序下载到目标 RAM 并运行。SPI 接口**只支持 RAM 下载**，不支持 flash 写/读/erase。支持标准 SPI 与 Quad-SPI。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/spi_load_ram.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "SPI 下载 RAM"
- "SPI 烧录 ESP"
- "Quad-SPI load to RAM"
- "esp_loader_init_spi"

## 前置条件

| 条件 | 要求 |
|---|---|
| port 编译 | `CONFIG_SERIAL_FLASHER_PORT_SPI=y` |
| 目标支持 SPI | ESP32-S3 / C2 / C3 / H2（见支持矩阵） |
| 接线 | SPI CLK/CS/MISO/MOSI + RESET + 4 个 strap bit 引脚（可选 QUADWP/QUADHD） |
| 参考示例 | `examples/esp32_spi_load_ram_example/` |

## 分步说明

### 1. 构造 SPI port（含 strap 引脚）

```c
#include "esp_loader.h"
#include "esp32_spi_port.h"
#include "example_common.h"

esp32_spi_port_t port = {
    .port.ops       = &esp32_spi_ops,
    .spi_bus        = SPI2_HOST,
    .frequency      = 20 * 1000000,      // 20MHz
    .reset_pin      = GPIO_NUM_5,
    .spi_clk_pin    = GPIO_NUM_12,
    .spi_cs_pin     = GPIO_NUM_10,
    .spi_miso_pin   = GPIO_NUM_13,
    .spi_mosi_pin   = GPIO_NUM_11,
    .spi_quadwp_pin = GPIO_NUM_14,        // Quad-SPI（可选）
    .spi_quadhd_pin = GPIO_NUM_9,
    .strap_bit0_pin = GPIO_NUM_13,
    .strap_bit1_pin = GPIO_NUM_2,
    .strap_bit2_pin = GPIO_NUM_3,
    .strap_bit3_pin = GPIO_NUM_4,
};
```

> strap bit 引脚对应目标下载模式的 strapping，详见目标芯片 TRM 的 "Boot Configuration" 章节。

### 2. 用 SPI init + 连接（速率参数传 0）

```c
esp_loader_t loader;
if (esp_loader_init_spi(&loader, &port.port) != ESP_LOADER_SUCCESS) {
    printf("SPI init failed\n");
    abort();
}

// SPI 不支持改速率，connect_to_target 第二参数传 0
if (connect_to_target(&loader, 0) != ESP_LOADER_SUCCESS) {
    return;
}
```

### 3. RAM 下载（与 UART 同一调用链）

```c
extern const uint8_t app_bin[];

ESP_LOGI(TAG, "Loading app to RAM ...");
esp_loader_error_t err = load_ram_binary(&loader, app_bin);
if (err != ESP_LOADER_SUCCESS) {
    ESP_LOGE(TAG, "Loading to RAM failed ...");
}
```

### 4. 监听目标日志（另开 UART）

SPI 占用了与目标通信的引脚，目标日志需经目标 UART0 转发到主机另一路 UART（示例用 UART_NUM_2 / GPIO6,7）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `UNSUPPORTED_FUNC`（flash 类调用） | SPI 仅支持 RAM 下载 | 不要对 SPI 调用 flash_start/write/read/erase |
| 连接失败 | strap 引脚接错 / 目标未进 SPI 下载模式 | 核对 strap bit 接线；查目标 TRM |
| Quad-SPI 不工作 | QUADWP/QUADHD 未接 | 接好 quad 引脚，或仅用标准 SPI |
| `connect_to_target` 提速相关报错 | SPI 不支持改速率 | 第二参数传 0 |

## 参考

- `examples/esp32_spi_load_ram_example/main/main.c` — SPI RAM 下载完整示例
- `port/esp32_spi_port.h` — `esp32_spi_port_t` 字段、`esp32_spi_ops`
- `docs/hardware-connections.md` — SPI 接线与 strap 说明
- `recipes/load_ram_uart.md` — RAM 下载调用链细节
