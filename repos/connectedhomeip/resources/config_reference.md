# connectedhomeip — Configuration Reference

> 所有 Kconfig 符号与 sdkconfig 项均来自仓库真实 `sdkconfig.defaults`、`Kconfig.projbuild` 与设备层配置。

## Rendezvous 模式（Demo 菜单）

来源：`examples/lock-app/esp32/main/Kconfig.projbuild`

| choice 项 | 数值 (`CONFIG_RENDEZVOUS_MODE`) | 说明 |
|---|---|---|
| `RENDEZVOUS_MODE_BYPASS` | `0` | 跳过安全配对，仅调试用 |
| `RENDEZVOUS_MODE_WIFI` | `1` | Wi-Fi Rendezvous |
| `RENDEZVOUS_MODE_BLE` | `2`（默认） | BLE Rendezvous（示例默认） |
| `RENDEZVOUS_MODE_THREAD` | `4` | Thread |
| `RENDEZVOUS_MODE_ETHERNET` | `8` | 以太网 |

`CONFIG_RENDEZVOUS_MODE` 为 `int`，`range 0 8`。在 menuconfig：`Demo -> Rendezvous Mode`。

## Echo Client（Demo 菜单，可选）

来源：`examples/lock-app/esp32/main/Kconfig.projbuild`

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_USE_ECHO_CLIENT` | bool | `n` | 启用内置 Echo Client |
| `CONFIG_ECHO_HOST_IP` | string | `127.0.0.1` | Echo 服务器 IPv4（依赖 `USE_ECHO_CLIENT`） |

## PW RPC 调试通道（Demo 菜单）

来源：`examples/lock-app/esp32/main/Kconfig.projbuild`

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_EXAMPLE_UART_PORT_NUM` | int | `0`（ESP32） | UART 端口号 |
| `CONFIG_EXAMPLE_UART_BAUD_RATE` | int | `115200` | UART 速率（1200–115200） |
| `CONFIG_EXAMPLE_UART_RXD` | int | `3` | UART RXD GPIO |
| `CONFIG_EXAMPLE_UART_TXD` | int | `1` | UART TXD GPIO |

## 设备类型（all-clusters-app Demo 菜单）

来源：`examples/all-clusters-app/esp32/main/Kconfig.projbuild`

| 符号 | 说明 |
|---|---|
| `CONFIG_DEVICE_TYPE_ESP32_DEVKITC` | ESP32-DevKitC（默认） |
| `CONFIG_DEVICE_TYPE_ESP32_WROVER_KIT` | ESP32-WROVER-KIT_V4.1 |
| `CONFIG_DEVICE_TYPE_M5STACK` | M5Stack |
| `CONFIG_DEVICE_TYPE_ESP32_C3_DEVKITM` | ESP32C3-DevKitM |

## sdkconfig.defaults（ESP32 设备应用）

来源：`examples/lock-app/esp32/sdkconfig.defaults`、`examples/all-clusters-app/esp32/sdkconfig.defaults`

| 配置项 | 作用 | 典型值 |
|---|---|---|
| `CONFIG_ESPTOOLPY_BAUD_921600B` | 烧录波特率档 | `y` |
| `CONFIG_ESPTOOLPY_BAUD` | 烧录波特率 | `921600` |
| `CONFIG_ESPTOOLPY_COMPRESSED` | 压缩烧录 | `y` |
| `CONFIG_ESPTOOLPY_MONITOR_BAUD_115200B` | 监视波特率档 | `y` |
| `CONFIG_ESPTOOLPY_MONITOR_BAUD` | 监视波特率 | `115200` |
| `CONFIG_BT_ENABLED` | 启用蓝牙（BLE 配网必需） | `y` |
| `CONFIG_BT_NIMBLE_ENABLED` | 启用 NimBLE（CHIP BLE 栈） | `y` |
| `CONFIG_LWIP_IPV6_AUTOCONFIG` | lwIP IPv6 自动配置 | `y` |
| `CONFIG_PARTITION_TABLE_CUSTOM` | 自定义分区表 | `y` |
| `CONFIG_PARTITION_TABLE_FILENAME` | 分区表文件名 | `"partitions.csv"` |

## CHIP Device Layer（menuconfig 子菜单）

来源：`Component config -> CHIP Device Layer`

| 路径 | 说明 |
|---|---|
| `WiFi Station Options` | Wi-Fi 站点 SSID/密码（Bypass/Wi-Fi 模式必填） |
| `WiFi AP Options` | Wi-Fi AP 配置 |
| 一般含 Device Type / Rendezvous 相关透传选项 | 见各示例 Kconfig |

## 构建开关（CMakeLists / 来源 main.cpp）

来源：`examples/lock-app/esp32/main/main.cpp`、`AppTask.cpp`

| 宏/开关 | 作用 |
|---|---|
| `CONFIG_ENABLE_PW_RPC` | 启用 Pigweed RPC（调用 `chip::rpc::Init()`） |
| `CONFIG_ENABLE_CHIP_SHELL` | 启用 CHIP shell（调用 `chip::LaunchShell()`） |
| `CONFIG_DEVICE_TYPE_*` | 决定 `main/main.cpp` 中的 GPIO/LED 映射 |

## 分区表（partitions.csv）

来源：`examples/lock-app/esp32/partitions.csv`

```csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     ,        0x6000,
phy_init, data, phy,     ,        0x1000,
factory,  app,  factory, ,        1945K,
```

要点：factory 分区约 1.9 MB；如增大 bootloader 需同步更新 offset 避免重叠。

## 目标设置命令

```bash
idf.py set-target esp32      # Xtensa，工具链 xtensa-esp32-elf
idf.py set-target esp32c3    # RISC-V，工具链 riscv-esp32-elf
```

## ESP-IDF 版本

设备示例要求 ESP-IDF `v4.3`（`examples/*/esp32/README.md` 均指向 `esp-idf/releases/v4.3`）。
