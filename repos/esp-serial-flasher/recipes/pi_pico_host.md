# Raspberry Pi Pico 主机烧录（Pico SDK / RP2040 / RP2350 ARM 与 RISC-V）

> **适用摘要**: 用 Raspberry Pi Pico（RP2040）或 Pico 2（RP2350）作主机经 `uart1` 烧录 ESP 目标，走内置 `pi_pico_port`（`PORT=PI_PICO`）。Pico 2 的 RP2350 可选 ARM 或 RISC-V 核，由 `PICO_PLATFORM` 决定，两套交叉编译器**不可互换**。镜像经 `.uf2` 拖拽烧入 Pico。

## 触发意图

- "Pico 烧录 ESP"
- "RP2040 烧录 ESP"
- "RP2350 / Pico 2 烧录 ESP"
- "pi_pico_port"
- "PORT=PI_PICO"
- "PICO_PLATFORM RISC-V"

## 前置条件

| 条件 | 要求 |
|---|---|
| 主机 | Raspberry Pi Pico（RP2040）或 Pico 2（RP2350），≥2MB flash |
| SDK | Raspberry Pi Pico SDK v2.2.0（平台 README 测试版本） |
| 工具链 | **ARM**：arm-gnu-toolchain-15.2（RP2040 必需，RP2350 ARM 核）；**RISC-V**：raspberrypi pico-sdk-tools 提供的 RISC-V GCC（仅 RP2350 RISC-V 核） |
| 工具 | CMake ≥3.22、`PICO_SDK_PATH`（或 `PICO_SDK_FETCH_FROM_GIT`） |
| 参考示例 | `examples/pi_pico_example/`（含 README、CMakeLists、target-firmware/、wiring） |
| 参考头 | `port/pi_pico_port.h`、`docs/platform-setup.md` |

## 分步说明

### 1. 选板与平台（RP2040 vs RP2350 / ARM vs RISC-V）

按 `docs/platform-setup.md` 的 Pico 章节：**原版 Pico（RP2040）只需 ARM 工具链**；**Pico 2（RP2350）** 可跑 ARM 或 RISC-V 核，由 `PICO_PLATFORM` 选择，每种对应一个不可互换的交叉编译器。

```bash
export PICO_SDK_PATH=<sdk_location>          # 或用 PICO_SDK_FETCH_FROM_GIT
export PICO_TOOLCHAIN_PATH=<arm 或 risc-v gcc 目录>

# RP2040（原版 Pico）— 仅 ARM
cmake -DPICO_BOARD=pico ..

# Pico 2 RP2350 — ARM 核
cmake -DPICO_BOARD=pico2 -DPICO_PLATFORM=rp2350-arm-s ..

# Pico 2 RP2350 — RISC-V 核（用 RISC-V 工具链）
cmake -DPICO_BOARD=pico2 -DPICO_PLATFORM=rp2350-risc-v ..
```

> `PICO_PLATFORM` 与 `PICO_TOOLCHAIN_PATH` 必须匹配：ARM 平台指 ARM gcc，RISC-V 平台指 RISC-V gcc。单次构建不能混用两套编译器。

### 2. 接线（来自示例 README）

Pico 的 `uart1` 专用于与 ESP 通信；`uart0` 或 USB 可作调试输出。

| Pi Pico (host) | Espressif SoC (target) |
|:---:|:---:|
| GP18 | BOOT |
| GP19 | RESET |
| GP20 | RX0 |
| GP21 | TX0 |

### 3. 构造 pi_pico_port（Pico SDK 模型）

`pi_pico_port_t`（`port/pi_pico_port.h`）与 STM32 port 类似，由调用者填 UART 实例、波特率、引脚号。port 的 `init` 回调**会自动初始化外设**（与 STM32 的 NULL-init 不同）。

```c
#include "pi_pico_port.h"
#include "esp_loader.h"

pi_pico_port_t port = {
    .port.ops             = &pi_pico_uart_ops,
    .uart_inst            = uart1,
    .baudrate             = 115200,
    .uart_rx_pin_num      = 21,
    .uart_tx_pin_num      = 20,
    .reset_pin_num        = 19,
    .boot_pin_num         = 18,
};

esp_loader_t loader;
esp_loader_init_serial(&loader, &port.port);   // 自动调用 Pico UART/GPIO 初始化
```

`dont_initialize_peripheral = true` 可在 UART 已被外部初始化时跳过 port 内部的 `uart_init`（默认 false，port 自己初始化）。

### 4. 连接 + 烧录

示例 `main.c` 复用 `examples/common/example_common.c` 的标准流程：

```c
esp_loader_connect_args_t args = ESP_LOADER_CONNECT_DEFAULT();
esp_loader_connect(&loader, &args);

if (esp_loader_get_target(&loader) != ESP8266_CHIP) {
    esp_loader_change_transmission_rate(&loader, 230400);   // 非 8266 提速
}

flash_binary(&loader, bootloader_bin,      bootloader_bin_size,      boot_offset);
flash_binary(&loader, partition_table_bin, partition_table_bin_size, 0x8000);
flash_binary(&loader, app_bin,             app_bin_size,             0x10000);

esp_loader_reset_target(&loader);
```

`boot_offset` 按 `example_common.c` 的 `bootloader_addresses[]` 取（ESP8266/C3/S3/C2/H2/C6/C61=0x0，ESP32/S2=0x1000，C5/P4=0x2000）。

### 5. 目标固件准备（bin2array）

示例 `CMakeLists.txt` 用 `bin2array.cmake` 把 `target-firmware/*.bin` 编译进固件：

```cmake
set(target_firmware_dir ${CMAKE_SOURCE_DIR}/target-firmware)
include(${CMAKE_SOURCE_DIR}/../common/require_target_firmware.cmake)
esp_serial_flasher_require_target_firmware("pi_pico_example" "${target_firmware_dir}"
    bootloader.bin partition-table.bin app.bin)

include(${CMAKE_SOURCE_DIR}/../common/bin2array.cmake)
create_resources(${target_firmware_dir} ${CMAKE_CURRENT_BINARY_DIR}/target_firmware_data.c)
```

用 ESP-IDF 构建目标固件后拷入：

```bash
cd test/target-example-src/hello-world-ESP32-src
idf.py set-target esp32s3
idf.py -D SDKCONFIG_DEFAULTS=sdkconfig.defaults.flash reconfigure build
cp build/bootloader/bootloader.bin      ../../pi_pico_example/target-firmware/
cp build/partition_table/partition-table.bin ../../pi_pico_example/target-firmware/
cp build/hello_world.bin                ../../pi_pico_example/target-firmware/app.bin
```

### 6. 构建与 uf2 烧入

```bash
cd examples/pi_pico_example
mkdir build && cd build
cmake .. && cmake --build .
```

生成 `.uf2` 后，按住 Pico 的 **BOOTSEL** 键接 USB，把它当作 Mass Storage 设备，把 `build/*.uf2` 拷进去即完成 Pico 自身烧录。

调试输出（minicom 连 Pico 虚拟串口）：

```bash
minicom -b 115200 -o -D /dev/ttyACM0
```

预期输出（README）：

```text
Connected to target
Transmission rate changed
Loading bootloader...
Start programming
Progress: 100 %
Finished programming
...
Done!
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| RISC-V 构建报 ARM 汇编错 | `PICO_PLATFORM` 与 `PICO_TOOLCHAIN_PATH` 不匹配 | ARM 平台指 ARM gcc；RISC-V 平台指 RISC-V gcc，不可混用 |
| `PICO_SDK_PATH` not set | 环境变量未设 | `export PICO_SDK_PATH=...`，或加 `-DPICO_SDK_FETCH_FROM_GIT=ON` |
| 连接超时 | TX/RX 接反或引脚号错 | GP20→ESP RX0，GP21→ESP TX0（Pico TX 接目标 RX） |
| 外设初始化冲突 | UART 已被别处 init 又没设 `dont_initialize_peripheral` | 设 `port.dont_initialize_peripheral = true` |
| uf2 烧不进 | 没按 BOOTSEL 上电 | 断电，按住 BOOTSEL 再接 USB，等出现盘符 |
| 提速失败（ESP8266 目标） | 8266 不支持改波特率 | 目标若是 8266 则跳过 `change_transmission_rate` |
| 看不到调试输出 | stdio 未启用 | 示例 `pico_enable_stdio_usb(... 1)` 与 `pico_enable_stdio_uart(... 1)` 默认都开 |

## 参考项目

- `examples/pi_pico_example/` — Pico 主机完整示例（README、CMakeLists、main.c、target-firmware/）
- `port/pi_pico_port.h` — `pi_pico_port_t`、`pi_pico_uart_ops`、`dont_initialize_peripheral` 字段
- `docs/platform-setup.md` — Pico Setup（`PICO_PLATFORM`/`PICO_TOOLCHAIN_PATH`/`PICO_SDK_PATH`）
- `examples/common/example_common.c` / `bin2array.cmake` — `flash_binary`、`bootloader_addresses[]`
- `docs/hardware-connections.md` — UART 接线
