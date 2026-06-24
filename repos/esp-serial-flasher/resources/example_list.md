# ESP Serial Flasher 示例清单

> 全部为仓库 `examples/` 下的真实示例目录（按 README 标题整理）。每个示例可直接作为起点复制改造。

## ESP32 主机示例

| 示例路径 | 标题 / 一句话说明 | 接口 | 连接方式 |
|---|---|---|---|
| `examples/esp32_example/` | Flash Multiple Partitions — UART 多分区烧录（bootloader+分区表+app），含 erase 演示 | UART | ROM bootloader |
| `examples/esp32_fast_reflash_example/` | Flash Multiple Partitions If MD5 Mismatch — 用已知 MD5 比对，仅烧变化分区 | UART | ROM bootloader |
| `examples/esp32_deflate_example/` | Flash Multiple Partitions with Compression (Deflate) — 预压缩 deflate 烧录 + 明文 MD5 校验 | UART | stub |
| `examples/esp32_load_ram_example/` | Load Program Into RAM — UART 把程序下载到 RAM 并运行 | UART | ROM bootloader |
| `examples/esp32_get_target_info_example/` | Get Target Info — 读 MAC / flash 容量 / security info / 芯片型号 | UART | ROM bootloader |
| `examples/esp32_read_flash_example/` | Read From Target Flash — 写后回读 flash 并比对 | UART | ROM bootloader |
| `examples/esp32_stub_example/` | Flashing Multiple Partitions While Using the Flasher Stub — stub 连接 + erase + 多分区烧录 | UART | stub |
| `examples/esp32_usb_cdc_acm_example/` | Flashing Multiple Partitions over USB CDC ACM Interface — USB Host CDC-ACM 烧录，含断开重连 | USB CDC-ACM | ROM bootloader |
| `examples/esp32_spi_load_ram_example/` | Loading the Program into RAM Through SPI — SPI 接口 RAM 下载 | SPI | ROM bootloader |
| `examples/esp32_sdio_example/` | Flashing through SDIO — SDIO 多分区烧录（ESP32-C5/C6 目标） | SDIO | 自动 stub |
| `examples/esp32_sdio_load_ram_example/` | Loading the Program into RAM Through SDIO — SDIO 接口 RAM 下载 | SDIO | 自动 stub |

## 其它主机平台示例

| 示例路径 | 标题 / 一句话说明 | 主机平台 | 内置 port | 关键引用 |
|---|---|---|---|---|
| `examples/linux_example/` | Linux example — 从任意 Linux 主机（PC / 树莓派）烧录 ESP，支持 DTR/RTS 自动复位与 libgpiod GPIO 复位 | Linux（libgpiod 可选） | `linux_port` | 见 recipe `linux_host.md` |
| `examples/pi_pico_example/` | Raspberry Pi Pico Example — RP2040 / RP2350 作主机烧录 ESP（Pico SDK v2.2.0，ARM 或 RISC-V 核） | Raspberry Pi Pico SDK | `pi_pico_port`（`PORT=PI_PICO`） | `port/pi_pico_port.h`；recipe `pi_pico_host.md` |
| `examples/stm32_example/` | STM32 Example — STM32 HAL 主机集成指南（STM32CubeMX 生成 CMake 工程，预初始化外设模型） | STM32 HAL | `stm32_port`（`PORT=STM32`） | `port/stm32_port.h`；recipe `stm32_host.md` |
| `examples/zephyr_example/` | ESP32 Zephyr Example — Zephyr RTOS（v4.4.0）集成，DTS 驱动 `espressif,esp-loader` 节点 + 可选 `esf` shell | Zephyr | `zephyr_port`（经 `esp_loader_from_device`） | `port/zephyr_port.h`、`zephyr/submanifest/esf.yaml`（用户创建）、`zephyr/Kconfig`、`examples/zephyr_example/boards/*.overlay`；recipe `zephyr_host.md` |

## 示例公共助手

| 路径 | 说明 |
|---|---|
| `examples/common/example_common.c` / `.h` | `connect_to_target` / `connect_to_target_with_stub` / `flash_binary` / `load_ram_binary` / `get_bootloader_address`；`bootloader_addresses[]` 表；`PARTITION_TABLE_ADDRESS=0x8000`、`APPLICATION_ADDRESS=0x10000` |
| `examples/common/bin2array.cmake` | 把 `target-firmware/*.bin` 转 C 数组，生成 `_bin` / `_bin_size` / `_bin_md5`（16 字节）符号 |
| `examples/common/require_target_firmware.cmake` | 校验 `target-firmware/` 下存在所需 .bin |
| `examples/README.md` | 示例总览（建议结合 `.gitlab-ci.yml` 看构建测试） |

## ESP32 UART 示例统一接线（来自 esp32_example/README.md）

| ESP32 (host) | Espressif SoC (target) |
|:---:|:---:|
| IO26 | BOOT |
| IO25 | RESET |
| IO4 | RX0 |
| IO5 | TX0 |

## 目标固件准备

各 ESP32 示例在 `target-firmware/` 放置：
- `bootloader.bin` — ESP bootloader
- `partition-table.bin` — 分区表
- `app.bin` — 主应用

可用自己的固件、esp-idf 示例，或 `test/target-example-src` 的源码构建。deflate 示例额外用 `target-firmware/compress_firmware.py` 生成压缩流。

## 构建/测试入口

- CI 构建测试见仓库根 `.gitlab-ci.yml`
- pytest 见各示例目录下的 `pytest_*.py`（如 `examples/esp32_example/pytest_esp32_example.py`），配置 `pytest.ini`
