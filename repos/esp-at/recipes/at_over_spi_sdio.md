# 通过 SPI 或 SDIO 承载 AT 指令

> **适用摘要**: 不使用 UART，改为通过 SPI 或 SDIO 接口在 ESP-AT 设备与主机 MCU 之间传输 AT 指令与数据，适用于需要更高吞吐或主机 MCU 已占用 UART 的场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-at/resources/`, source/examples in `repos/esp-at/`, and this recipe path `repos/esp-at/recipes/at_over_spi_sdio.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "SPI AT"
- "SDIO AT"
- "AT via SPI"
- "AT via SDIO"
- "非 UART 承载 AT"
- "at_spi_master"
- "at_sdio_host"

## 前置条件

| 条件 | 要求 |
|---|---|
| 设备端模块 | 选择对应 SDIO/SPI 模块（如 `ESP32-SDIO`、`ESP32C3-SPI`、`ESP32C5-SDIO`、`ESP32C5-SPI`） |
| 主机 MCU | STM32/ESP32 等，运行 SDIO/SPI 主机驱动 |
| 工程文档 | `main/interface/spi/README.md`、`main/interface/sdio/README.md`、`main/interface/socket/README.md` |

## 分步说明

ESP-AT 支持四种通信方式（`AT_COMMUNICATION_METHOD` choice，见 `main/Kconfig`）：`AT_BASE_ON_UART`、`AT_BASE_ON_SPI`、`AT_BASE_ON_SDIO`、`AT_BASE_ON_SOCKET`。

### 设备端：选择对应模块并烧录

`factory_param_data.csv` 中 SDIO/SPI 模块的 `uart_port` 与各 pin 列均为 `-1`，表示不走 UART：

```csv
# 节选自 factory_param_data.csv
PLATFORM_ESP32,ESP32-SDIO,"4MB, Wi-Fi + BLE, OTA, communicate with MCU via SDIO",4,78,-1,1,13,CN,-1,-1,-1,-1,-1,1
PLATFORM_ESP32C3,ESP32C3-SPI,"4MB, Wi-Fi + BLE, OTA, communicate with MCU via SPI",4,78,-1,1,13,CN,-1,-1,-1,-1,-1,1
PLATFORM_ESP32C5,ESP32C5-SDIO,"4MB, Wi-Fi + BLE, OTA, communicate with MCU via SDIO",4,78,-1,1,13,CN,-1,-1,-1,-1,-1,1
PLATFORM_ESP32C5,ESP32C5-SPI,"4MB, Wi-Fi + BLE, OTA, communicate with MCU via SPI",4,78,1,1,13,CN,-1,-1,-1,-1,-1,1
```

构建时：

```bash
./build.py install
# Platform name 选 PLATFORM_ESP32C3
# Module name 选 ESP32C3-SPI
./build.py build
./build.py -p /dev/ttyUSB0 flash
```

> 也可在 menuconfig 的 `AT` → `communicate method for AT command` 中改通信方式（`AT_BASE_ON_SPI`/`AT_BASE_ON_SDIO`/`AT_BASE_ON_SOCKET`）。

### 主机端：使用官方主机驱动示例

ESP-AT 提供 SPI/SDIO 主机端示例：

| 示例 | 说明 |
|---|---|
| `examples/at_spi_master/` | SPI 主机端示例，含 `spi/`（通用）与 `sdspi/`（SD over SPI） |
| `examples/at_sdio_host/` | SDIO 主机端示例，含 `ESP32/` 与 `STM32/` 子目录及 `res/` 资源 |

```bash
# SPI 主机端
ls examples/at_spi_master/
# README.md  sdspi  spi

# SDIO 主机端
ls examples/at_sdio_host/
# ESP32  README.md  STM32  res
```

按对应平台（ESP32 主机或 STM32 主机）把示例集成到主机工程，按 README 接线与初始化。

### socket 方式

也可用 socket 承载 AT（`AT_BASE_ON_SOCKET`），见 `main/interface/socket/README.md`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 选了 SPI/SDIO 模块但仍走 UART | menuconfig 通信方式未改 | 改 `AT_BASE_ON_SPI`/`AT_BASE_ON_SDIO`，或确认选了 SPI/SDIO 模块 |
| 主机端无响应 | 接线/时序/电平不匹配 | 参照 `examples/at_spi_master/README.md`、`at_sdio_host/README.md` 接线 |
| SDIO/SPI 模块引脚显示 `-1` | 这是正常的，因不走 UART | 不需配 uart 引脚，按 SPI/SDIO 硬件接线 |
| 切换通信方式后启动失败 | factory_param 与通信方式不一致 | module 选择与 `AT_COMMUNICATION_METHOD` 一致 |
| 主机端编译报错 | 平台不匹配（ESP32 vs STM32） | 选对应子目录示例 |

## 参考

- SPI 主机示例：`examples/at_spi_master/`
- SDIO 主机示例：`examples/at_sdio_host/`
- 设备端接口源码：`main/interface/`（`sdio/`、`spi/`、`socket/`、`uart/`）
- 模块参数：`components/customized_partitions/raw_data/factory_param/factory_param_data.csv`
- Kconfig：`main/Kconfig`（`AT_COMMUNICATION_METHOD` choice）
- 仓库文档：`docs/en/Compile_and_Develop/How_to_implement_SDIO_AT.rst`、`How_to_implement_SPI_AT.rst`
