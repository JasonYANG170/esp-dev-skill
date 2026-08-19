# 在 ESP-IDF 项目中集成（managed component）

> **适用摘要**: 把 ESP Serial Flasher 作为 managed component 添加到 ESP-IDF 项目，配置 port 编译选项，设置 sdkconfig.defaults，并接入目标固件 bin2array 流程。

> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/idf_component_setup.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-IDF 集成 esp-serial-flasher"
- "add-dependency esp-serial-flasher"
- "配置 PORT_UART / PORT_SPI / PORT_SDIO / PORT_USB_CDC_ACM"
- "sdkconfig 配置 serial flasher"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.5 或更新 |
| 参考示例 | `examples/esp32_example/`、`examples/esp32_usb_cdc_acm_example/`、`examples/esp32_spi_load_ram_example/`、`examples/esp32_sdio_example/` |

## 分步说明

### 1. 添加依赖

```bash
cd my_project
idf.py add-dependency "espressif/esp-serial-flasher"
```
会写入 `idf_component.yml`：
```yaml
dependencies:
  espressif/esp-serial-flasher: "^2"
```
> `usb_host_cdc_acm` / `usb_host_cp210x_vcp` / `usb_host_ch34x_vcp` 在 S2/S3/P4 target 上由该组件 `idf_component.yml` 的 rules 自动拉取，无需手动加。

### 2. 选择 port 编译（sdkconfig.defaults）

```ini
# sdkconfig.defaults
CONFIG_SERIAL_FLASHER_PORT_UART=y          # 默认开
# 按需开启其它 port（可同时开多个，运行时切换）：
# CONFIG_SERIAL_FLASHER_PORT_SPI=y
# CONFIG_SERIAL_FLASHER_PORT_SDIO=y
# CONFIG_SERIAL_FLASHER_PORT_USB_CDC_ACM=y   # 依赖 SOC_USB_OTG_SUPPORTED

# 日志级别
CONFIG_SERIAL_FLASHER_LOG_LEVEL_WARN=y     # 推荐（默认）
# 重试/时序
CONFIG_SERIAL_FLASHER_WRITE_BLOCK_RETRIES=3
CONFIG_SERIAL_FLASHER_RESET_HOLD_TIME_MS=100
CONFIG_SERIAL_FLASHER_BOOT_HOLD_TIME_MS=50
```
或 `idf.py menuconfig → ESP serial flasher`。

### 3. main/CMakeLists.txt 接入 bin2array

```cmake
set(srcs main.c ../../common/example_common.c)
set(include_dirs . ../../common)
idf_component_register(SRCS ${srcs} INCLUDE_DIRS ${include_dirs})

set(target ${COMPONENT_LIB})
set(target_firmware_dir ${CMAKE_SOURCE_DIR}/target-firmware)
include(${CMAKE_SOURCE_DIR}/../common/require_target_firmware.cmake)
esp_serial_flasher_require_target_firmware("my_flasher" "${target_firmware_dir}"
    bootloader.bin partition-table.bin app.bin)

include(${CMAKE_SOURCE_DIR}/../common/bin2array.cmake)
create_resources(${target_firmware_dir} ${CMAKE_BINARY_DIR}/target_firmware_data.c)
set_property(SOURCE ${CMAKE_BINARY_DIR}/target_firmware_data.c PROPERTY GENERATED 1)
target_sources(${target} PRIVATE ${CMAKE_BINARY_DIR}/target_firmware_data.c)
```
> `require_target_firmware.cmake` 校验 `target-firmware/` 下存在指定 .bin；`bin2array.cmake` 生成 `_bin` / `_bin_size` / `_bin_md5` 符号。

### 4. main.c 引用与编译

```c
#include "esp_loader.h"
#include "esp32_port.h"          // 或对应 port 头
#include "example_common.h"
// ...
```
```bash
idf.py set-target esp32s3
idf.py build
idf.py -p PORT flash monitor
```

## 配置项速查（来自 Kconfig / docs/configuration.md）

| Kconfig | CMake 变量 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_SERIAL_FLASHER_PORT_UART` | — | y | 编译 `esp32_port.c`，暴露 `esp32_uart_ops` |
| `CONFIG_SERIAL_FLASHER_PORT_SPI` | — | n | 编译 `esp32_spi_port.c`，`esp32_spi_ops`（仅 RAM 下载） |
| `CONFIG_SERIAL_FLASHER_PORT_SDIO` | — | n | 编译 `esp32_sdio_port.c`，`esp32_sdio_ops`（实验性） |
| `CONFIG_SERIAL_FLASHER_PORT_USB_CDC_ACM` | — | n | 编译 `esp32_usb_cdc_acm_port.c`（需 USB OTG） |
| `CONFIG_SERIAL_FLASHER_LOG_LEVEL_*` | `SERIAL_FLASHER_LOG_LEVEL` | WARN(2) | NONE/ERROR/WARN/INFO/DEBUG |
| `CONFIG_SERIAL_FLASHER_WRITE_BLOCK_RETRIES` | `SERIAL_FLASHER_WRITE_BLOCK_RETRIES` | 3 | 写块重试次数 |
| `CONFIG_SERIAL_FLASHER_RESET_HOLD_TIME_MS` | `SERIAL_FLASHER_RESET_HOLD_TIME_MS` | 100 | 复位保持 ms |
| `CONFIG_SERIAL_FLASHER_BOOT_HOLD_TIME_MS` | `SERIAL_FLASHER_BOOT_HOLD_TIME_MS` | 50 | boot 保持 ms |
| `CONFIG_SERIAL_FLASHER_RESET_INVERT` | `SERIAL_FLASHER_RESET_INVERT` | n | 反相 reset（UART） |
| `CONFIG_SERIAL_FLASHER_BOOT_INVERT` | `SERIAL_FLASHER_BOOT_INVERT` | n | 反相 boot（UART） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 找不到 `esp_loader.h` | 未 add-dependency / include 路径 | `add-dependency`；`INCLUDE_DIRS` 含 `../../common` |
| USB CDC port 灰显 | target 不支持 USB OTG | 换 S2/S3/P4 target；`depends on SOC_USB_OTG_SUPPORTED` |
| `*_bin_md5` 未定义 | 未用 bin2array | 按 Step 3 接入 bin2array.cmake |
| 编译体积大 | 默认含 stub rodata | 不用 stub 即可；链接器 GC 会剥除（见 README Flash Size Footprint） |

## 参考

- `examples/esp32_example/`、`examples/esp32_usb_cdc_acm_example/` — 各 port 的 ESP-IDF 集成
- `idf_component.yml` — 组件依赖与 USB 组件 rules
- `Kconfig` — 所有 `CONFIG_SERIAL_FLASHER_*` 选项定义
- `docs/configuration.md` — CMake/Kconfig 配置完整说明
- `docs/platform-setup.md` — ESP-IDF setup
