# 通过 SDIO 接口烧录（实验性）

> **适用摘要**: 用 ESP32 主机的 SDIO host（`esp32_sdio_port`）+ `esp_loader_init_sdio()` 烧录目标。SDIO 连接时自动上传 `esp-flasher-stub`，支持完整 stub 命令（含 deflate）。目前仅 ESP32-C5 / C6 作目标。SDIO 不支持改速率。

## 触发意图

- "SDIO 烧录"
- "SDIO flash"
- "esp_loader_init_sdio"
- "高速 SDIO 烧录 ESP"

## 前置条件

| 条件 | 要求 |
|---|---|
| port 编译 | `CONFIG_SERIAL_FLASHER_PORT_SDIO=y`（默认关） |
| 目标���持 SDIO | ESP32-C5 / ESP32-C6（见支持矩阵，实验性） |
| host | 具 SDIO host 支持的 ESP 芯片 |
| 接线 | SDIO CLK/CMD/D0（+D1-D3 四位模式）+ RESET + BOOT；可能需 10-47kΩ 上拉 |
| 参考示例 | `examples/esp32_sdio_example/`、`examples/esp32_sdio_load_ram_example/` |

## 分步说明

### 1. 构造 SDIO port（4-bit 模式示例）

```c
#include "esp_loader.h"
#include "esp32_sdio_port.h"
#include "example_common.h"
#include "driver/sdmmc_host.h"

esp32_sdio_port_t port = {
    .port.ops     = &esp32_sdio_ops,
    .slot         = SDMMC_HOST_SLOT_1,
    .max_freq_khz = SDMMC_FREQ_DEFAULT,
    .reset_pin    = GPIO_NUM_54,
    .boot_pin     = GPIO_NUM_53,
    .bus_width    = SDIO_4BIT,
    .sdio_d0_pin  = GPIO_NUM_50,
    .sdio_d1_pin  = GPIO_NUM_49,
    .sdio_d2_pin  = GPIO_NUM_48,
    .sdio_d3_pin  = GPIO_NUM_47,
    .sdio_clk_pin = GPIO_NUM_51,
    .sdio_cmd_pin = GPIO_NUM_52,
};
```

### 2. init_sdio + 连接（速率参数传 0）

```c
esp_loader_t loader;
if (esp_loader_init_sdio(&loader, &port.port) != ESP_LOADER_SUCCESS) {
    printf("SDIO init failed\n");
    abort();
}

// SDIO 不支持改速率；且 connect 时自动上传 stub
if (connect_to_target(&loader, 0) != ESP_LOADER_SUCCESS) {
    return;
}
```

### 3. 多分区烧录（与 UART 同一调用链）

```c
target_chip_t chip = esp_loader_get_target(&loader);
flash_binary(&loader, bootloader_bin, bootloader_bin_size, get_bootloader_address(chip));
flash_binary(&loader, partition_table_bin, partition_table_bin_size, 0x8000);
flash_binary(&loader, app_bin, app_bin_size, 0x10000);

esp_loader_reset_target(&loader);
```

### 4. 监听目标日志（另开 UART）

SDIO 示例用 UART2（如 GPIO46/45）转发目标 UART0 输出。

## SDIO 特性提醒

- 自动走 stub：`esp_loader_connect()` 即上传 `esp-flasher-stub`，无需 `connect_with_stub`
- 支持 deflate（stub 命令共享）
- 不支持 `esp_loader_change_transmission_rate`（host 驱动管理时钟）
- CMD/DAT 上拉：依硬件可能需 10-47kΩ 上拉到目标 VDD

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 连接失败 | CMD/D0 缺上拉 / 接线错 | 加 10-47kΩ 上拉；核对 CLK/CMD/D0 |
| 不支持速率变更 | SDIO 设计如此 | 不要调用 `change_transmission_rate` |
| 目标非 C5/C6 | SDIO 仅这两款目标 | 查支持矩阵；改用 UART |
| 时钟边沿问题 | PCB 走线/信号完整性 | 参考目标 datasheet 配置采样/驱动时钟沿 |

## 参考

- `examples/esp32_sdio_example/main/main.c` — SDIO 多分区烧录示例
- `examples/esp32_sdio_load_ram_example/` — SDIO RAM 下载示例
- `port/esp32_sdio_port.h` — `esp32_sdio_port_t`、`esp32_sdio_ops`、`sdio_bus_width_t`
- `docs/hardware-connections.md` — SDIO 接线与上拉指南
- README.md "Flash Size Footprint" — SDIO 仅链入相关 stub（+~20KB）
