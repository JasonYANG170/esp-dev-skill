# 修改 AT 端口引脚

> **适用摘要**: 修改 AT 命令端口（默认 UART1）与日志端口（默认 UART0）的 TX/RX/CTS/RTS 引脚，适配自定义硬件布局。

## 触发意图

- "改 AT 串口引脚"
- "set AT port pin"
- "自定义 UART 引脚"
- "AT command port pin"
- "修改命令端口"

## 前置条件

| 条件 | 要求 |
|---|---|
| 方式 A（不改源码） | 已有 `factory_XXX.bin` + `tools/at.py` |
| 方式 B（改源码重编） | 已按 `recipes/build_and_flash.md` 克隆工程 |

## 分步说明

ESP-AT 默认用两个 UART：日志端口（UART0，输出日志）与命令端口（UART1，收发 AT 指令）。

### 默认引脚（来自 `factory_param_data.csv` 与 `How_to_set_AT_port_pin.rst`）

| 芯片 | 命令口 TX/RX | 日志口 TX/RX |
|---|---|---|
| ESP32 (WROOM-32) | GPIO17 / GPIO16 | GPIO1 / GPIO3 |
| ESP32-C3 (MINI-1) | GPIO7 / GPIO6 | GPIO21 / GPIO20 |
| ESP32-C2 | GPIO7 / GPIO6 | GPIO20 / GPIO19 |
| ESP32-C5 | GPIO23 / GPIO24 | GPIO11 / GPIO12 |
| ESP32-C6 | GPIO7 / GPIO6 | GPIO16 / GPIO17 |
| ESP32-C61 | GPIO7 / GPIO6 | GPIO11 / GPIO10 |
| ESP32-S2 (MINI) | GPIO17 / GPIO21 | GPIO43 / GPIO44 |

### 方式 A：用 `at.py` 改打包好的 factory bin（无需重编）

```bash
# 1. 下载 tools/at.py（仓库内 tools/at.py）
# 2. 查看用法
python at.py modify_bin --help

# 3. 修改命令口引脚（以 ESP32-C3 为例：TX=7, RX=6）
python at.py modify_bin --input factory_MINI-1.bin \
    --tx_pin 7 --rx_pin 6 --cts_pin 5 --rts_pin 4

# 4. 烧录修改后的 bin
esptool.py -p /dev/ttyUSB0 write_flash 0x0 factory_MINI-1.bin
```

### 方式 B：改源码重编（命令端口）

命令口引脚由 `factory_param_data.csv` 对应模块行的 `uart_port`、`uart_tx_pin`、`uart_rx_pin`、`uart_cts_pin`、`uart_rts_pin` 列决定。

```csv
# components/customized_partitions/raw_data/factory_param/factory_param_data.csv
# platform,module_name,...,uart_port,...,uart_tx_pin,uart_rx_pin,uart_cts_pin,uart_rts_pin,...
# 把 MINI-1 行的引脚改成你需要的（示例：TX=8 RX=9，不使用流控置 -1）
PLATFORM_ESP32C3,MINI-1,"4MB, Wi-Fi + BLE, OTA, TX:8 RX:9",4,78,1,1,13,CN,115200,8,9,-1,-1,1
```

改完后重新编译，生成 `build/customized_partitions/factory_param.bin`，烧录：

```bash
./build.py -p /dev/ttyUSB0 flash
```

### 方式 C：改源码重编（日志端口）

日志口引脚在 menuconfig 中改，不在 CSV 里：

```
./build.py menuconfig
  → Component config
    → ESP System Settings
      → Channel for console output → Custom UART
      → UART TX on GPIO#
      → UART RX on GPIO#
```

> 若想让命令口与日志口共用同一组引脚，需把 CSV 的 `uart_port` 改成日志口 UART 号，且 `uart_tx_pin/uart_rx_pin` 与日志口一致。

### SDIO/SPI 模块（不走 UART）

`factory_param_data.csv` 中 `ESP32-SDIO`、`ESP32C3-SPI`、`ESP32C5-SDIO`、`ESP32C5-SPI` 等模块的 `uart_port` 与各 pin 列均为 `-1`，表示通过 SDIO/SPI 承载 AT。改用这些接口见 `recipes/at_over_spi_sdio.md`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 改引脚后启动失败/无响应 | 命令引脚与日志引脚或其他外设冲突 | 查技术参考手册确认引脚可用，避免冲突 |
| `at.py` 报参数不支持 | factory bin 不是 2MB/4MB 生产固件 | 用 `build/factory/factory_XXX.bin` |
| 改了 `partitions_at.csv` | 改错文件，可能无法启动 | 引脚改 `factory_param_data.csv`，不要碰分区主表 |
| 流控引脚占用冲突 | CTS/RTS 引脚被其他功能占用 | 不用流控时把对应列置 `-1` |
| 引脚改了但没生效 | 只改 CSV 没重新生成 `factory_param.bin` | 重新 `./build.py build` 或用 `gen_esp32part.py` |

## 参考

- 仓库文档：`docs/en/Compile_and_Develop/How_to_set_AT_port_pin.rst`
- 引脚定义：`components/customized_partitions/raw_data/factory_param/factory_param_data.csv`
- at.py 工具：`tools/at.py`、`docs/en/Compile_and_Develop/tools_at_py.rst`
