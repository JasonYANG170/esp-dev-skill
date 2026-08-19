# UART 主机连接并烧录 ESP 目标（ROM bootloader）

> **适用摘要**: 在 ESP-IDF 主机（或任意 serial port）上，用 UART 接口把目标 ESP 芯片置入下载模式、连接（ROM bootloader，非 stub）、提速、多分区烧录并复位。这是最常用、最基础的烧录场景。

> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/uart_connect_flash.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "用 MCU 烧录 ESP"
- "UART 烧录 ESP32"
- "从另一个 ESP32 给目标烧录"
- "esp_loader 基本用法"
- "跨 MCU 烧录"
- "主机烧录目标固件"

## 前置条件

| 条件 | 要求 |
|---|---|
| 主机平台 | ESP-IDF v5.5+（或任意带 `esp32_port`/`linux_port`/`pi_pico_port` 的 serial port） |
| port 编译 | `CONFIG_SERIAL_FLASHER_PORT_UART=y`（默认开） |
| 接线 | Host UART TX→Target RX0；Host UART RX→Target TX0；Host GPIO→Target RESET；Host GPIO→Target BOOT；共地 |
| 目标固件 | `target-firmware/bootloader.bin`、`partition-table.bin`、`app.bin`（经 bin2array 转 C 数组） |
| 参考示例 | `examples/esp32_example/` |

## 分步说明

### 1. 接线（ESP32 主机 → ESP 目标）

| ESP32 (host) | Espressif SoC (target) |
|:---:|:---:|
| IO26 | BOOT |
| IO25 | RESET |
| IO4 | RX0 |
| IO5 | TX0 |

### 2. 嵌入目标固件（bin2array）

`main/CMakeLists.txt`：
```cmake
set(target_firmware_dir ${CMAKE_SOURCE_DIR}/target-firmware)
include(${CMAKE_SOURCE_DIR}/../common/bin2array.cmake)
create_resources(${target_firmware_dir} ${CMAKE_BINARY_DIR}/target_firmware_data.c)
target_sources(${target} PRIVATE ${CMAKE_BINARY_DIR}/target_firmware_data.c)
```
生成符号 `bootloader_bin` / `bootloader_bin_size` / `bootloader_bin_md5` 等。

### 3. 构造 port 并 init

```c
#include "esp_loader.h"
#include "esp32_port.h"
#include "example_common.h"

extern const uint8_t bootloader_bin[];
extern const uint32_t bootloader_bin_size;
extern const uint8_t partition_table_bin[];
extern const uint32_t partition_table_bin_size;
extern const uint8_t app_bin[];
extern const uint32_t app_bin_size;

void app_main(void)
{
    esp32_port_t port = {
        .port.ops    = &esp32_uart_ops,
        .baud_rate   = 115200,
        .uart_port   = UART_NUM_1,
        .uart_rx_pin = GPIO_NUM_5,
        .uart_tx_pin = GPIO_NUM_4,
        .reset_pin   = GPIO_NUM_25,
        .boot_pin    = GPIO_NUM_26,
    };

    esp_loader_t loader;
    if (esp_loader_init_serial(&loader, &port.port) != ESP_LOADER_SUCCESS) {
        return;  // UART 驱动初始化失败
    }
    /* ...连接 + 烧录（见下）... */
}
```

### 4. 连接（ROM bootloader）+ 提速

```c
    esp_loader_connect_args_t connect_args = ESP_LOADER_CONNECT_DEFAULT();
    esp_loader_error_t err = esp_loader_connect(&loader, &connect_args);
    if (err != ESP_LOADER_SUCCESS) {
        printf("Cannot connect: %d\n", err);
        return;
    }
    printf("Connected to target\n");

    // 非 ESP8266 才提速（8266 不支持改波特率）
    if (esp_loader_get_target(&loader) != ESP8266_CHIP) {
        err = esp_loader_change_transmission_rate(&loader, 230400);
        if (err != ESP_LOADER_SUCCESS) {
            printf("change rate failed: %d\n", err);
            return;
        }
    }
```

### 5. 多分区烧录（用 example_common 的 helper）

```c
    target_chip_t chip = esp_loader_get_target(&loader);
    uint32_t bootloader_addr = get_bootloader_address(chip);  // 因芯片而异

    flash_binary(&loader, bootloader_bin,      bootloader_bin_size,      bootloader_addr);
    flash_binary(&loader, partition_table_bin, partition_table_bin_size, 0x8000);
    flash_binary(&loader, app_bin,             app_bin_size,             0x10000);

    esp_loader_reset_target(&loader);   // 复位目标，运行新固件
```

`flash_binary()` 内部完成 `flash_start`（含擦除）→ 循环 `flash_write` → `flash_finish`（默认 MD5 校验）。进度打印 `Progress: NN %`，成功后 `Flash verified`。

### 6. 转发目标串口日志（可选）

```c
    vTaskDelay(500 / portTICK_PERIOD_MS);  // 跳过目标启动横幅
    // 复用同一 UART1 在 115200 读目标日志（提速后需先降回 115200）
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_LOADER_ERROR_TIMEOUT`（连接） | TX/RX 接反、reset/boot 未接、共地缺失、目标未进下载模式 | 核对接线表；确认 host GPIO 能控制 target RESET/BOOT |
| `ESP_LOADER_ERROR_INVALID_RESPONSE` | 波特率过高 / 线缆过长 / 供电不足 | 降波特率（先用 115200 连接再提速）；缩短线缆；独立供电 |
| 提速后失败 | 线缆/供电扛不住高波特率 | 降低 `higher_transmission_rate`，或保持 115200 |
| 目标复位后无日志 | 复用了烧录 UART 但未降回 115200 | `uart_set_baudrate(UART_NUM_1, 115200)` |
| ESP8266 提速失败 | 8266 bootloader 不支持改速率 | 跳过 `change_transmission_rate`，或连接前在 port 设定波特率 |
| `INVALID_MD5` | 镜像不完整 / 传输错误 | 重新烧录；检查 bin 大小；ESP8266 设 `skip_verify=true` |

## 参考

- `examples/esp32_example/` — UART 多分区烧录完整示例
- `examples/common/example_common.c` — `connect_to_target` / `flash_binary` / `get_bootloader_address`
- `docs/hardware-connections.md` — UART 接线与各接口引脚
- `docs/platform-setup.md` — ESP-IDF 集成步骤
- `recipes/flash_partitions.md` — 手写 start/write/finish 的等价实现
