# ESP Serial Flasher 配置项速查

> 全部来自 `Kconfig`、`zephyr/Kconfig`、`docs/configuration.md`、`idf_component.yml`。v2 协议在运行时选择，无编译期接口开关；port 编译由 `CONFIG_SERIAL_FLASHER_PORT_*` 控制（可同时开多个）。

## 配置方式对照

| 构建系统 | 配置载体 |
|---|---|
| ESP-IDF | Kconfig（`menuconfig` / `sdkconfig` / `sdkconfig.defaults`），前缀 `CONFIG_` |
| Zephyr | Kconfig（`prj.conf`），前缀 `CONFIG_` |
| 纯 CMake（Linux/Pico/自定义） | CMake cache 变量（`-D`），无 `CONFIG_` 前缀 |

## Port 编译选择（ESP-IDF / Zephyr）

| Kconfig | 默认 | 说明 |
|---|:---:|---|
| `CONFIG_SERIAL_FLASHER_PORT_UART` | y | 编译 `esp32_port.c`，暴露 `esp32_uart_ops` |
| `CONFIG_SERIAL_FLASHER_PORT_SPI` | n | 编译 `esp32_spi_port.c`，`esp32_spi_ops`（仅 RAM 下载） |
| `CONFIG_SERIAL_FLASHER_PORT_SDIO` | n | 编译 `esp32_sdio_port.c`，`esp32_sdio_ops`（实验性，需 SDIO host） |
| `CONFIG_SERIAL_FLASHER_PORT_USB_CDC_ACM` | n | 编译 `esp32_usb_cdc_acm_port.c`，`esp32_usb_cdc_acm_ops`（`depends on SOC_USB_OTG_SUPPORTED`，需 `usb_host_cdc_acm` ^2） |

> 可同时启用多个 port，实现同一固件运行时切换接口。

## 日志级别

| Kconfig choice | 内部数值（`CONFIG_SERIAL_FLASHER_LOG_LEVEL`） | CMake 变量 `SERIAL_FLASHER_LOG_LEVEL` | 含义 |
|---|:---:|---|---|
| `..._LOG_LEVEL_NONE` | 0 | NONE / 0 | 全部静默，零开销 |
| `..._LOG_LEVEL_ERROR` | 1 | ERROR / 1 | 协议错误、MD5 不匹配 |
| `..._LOG_LEVEL_WARN`（默认） | 2 | WARN / 2 | + flash 容量回退、SDIO 重试 |
| `..._LOG_LEVEL_INFO` | 3 | INFO / 3 | + 连接成功、芯片、flash 容量、写开始 |
| `..._LOG_LEVEL_DEBUG` | 4 | DEBUG / 4 | + 每命令跟踪、hex dump、SPI 轮询 |

```bash
# 纯 CMake
cmake -DSERIAL_FLASHER_LOG_LEVEL=DEBUG ..
# ESP-IDF / Zephyr
# sdkconfig / prj.conf
CONFIG_SERIAL_FLASHER_LOG_LEVEL_DEBUG=y
```

> 日志由 port 的 `log` / `log_hex` 回调输出；回调置 NULL 则静默。DEBUG 的命令跟踪与 hex dump 仅在选定 DEBUG 时编译进来。

## 重试与时序

| 项 | Kconfig | CMake 变量 | 默认 | 说明 |
|---|---|---|---:|---|
| 写块重试次数 | `CONFIG_SERIAL_FLASHER_WRITE_BLOCK_RETRIES` | `SERIAL_FLASHER_WRITE_BLOCK_RETRIES` | 3 | flash/RAM 写块失败重试 |
| 复位保持时间 | `CONFIG_SERIAL_FLASHER_RESET_HOLD_TIME_MS` | `SERIAL_FLASHER_RESET_HOLD_TIME_MS` | 100 ms | 硬复位 assert 时长 |
| Boot 保持时间 | `CONFIG_SERIAL_FLASHER_BOOT_HOLD_TIME_MS` | `SERIAL_FLASHER_BOOT_HOLD_TIME_MS` | 50 ms | boot assert 时长 |

## GPIO 反相（仅 UART）

| 项 | Kconfig | CMake 变量 | 默认 | 说明 |
|---|---|---|:---:|---|
| 反相 reset | `CONFIG_SERIAL_FLASHER_RESET_INVERT` | `SERIAL_FLASHER_RESET_INVERT` | n | 硬件有反相电路时启用 |
| 反相 boot | `CONFIG_SERIAL_FLASHER_BOOT_INVERT` | `SERIAL_FLASHER_BOOT_INVERT` | n | 同上 |

## CMake 多选项示例

```bash
cmake \
  -DSERIAL_FLASHER_WRITE_BLOCK_RETRIES=5 \
  -DSERIAL_FLASHER_RESET_HOLD_TIME_MS=200 \
  -DSERIAL_FLASHER_LOG_LEVEL=INFO \
  .. && cmake --build .
```

## Linux port 构建变量

| 变量 | 默认 | 说明 |
|---|---|---|
| `PORT` | — | 必须为 `LINUX` |
| `LINUX_PORT_GPIO` | OFF | 启用 libgpiod 字符设备复位（需 libgpiod ≥2.0） |

## 协议/接口运行时选择（无编译期开关）

```c
esp_loader_init_serial(&loader, &port.port); // UART / USB CDC-ACM / Linux tty
esp_loader_init_spi   (&loader, &port.port); // SPI（仅 RAM 下载）
esp_loader_init_sdio  (&loader, &port.port); // SDIO（自动 stub）
```
不支持某协议的功能返回 `ESP_LOADER_ERROR_UNSUPPORTED_FUNC`。

## idf_component.yml 依赖规则（自动按 target 拉取）

```yaml
dependencies:
  usb_host_cdc_acm:    { version: "^2",    if: "target in [esp32s2, esp32s3, esp32p4]" }
  usb_host_cp210x_vcp: { version: "^2.2",  if: "target in [esp32s2, esp32s3, esp32p4]" }
  usb_host_ch34x_vcp:  { version: "^2.2.1",if: "target in [esp32s2, esp32s3, esp32p4]" }
```
> 仅在启用 USB CDC-ACM port 且 target 为 S2/S3/P4 时需要；通常由组件自动管理。

## Zephyr 相关 Kconfig

| Kconfig | 说明 |
|---|---|
| `CONFIG_ESP_SERIAL_FLASHER_UART_BUFSIZE` | Zephyr port 的 TTY 收发缓冲大小（`zephyr_port_t._tty_rx_buf` / `_tty_tx_buf`） |

Zephyr port 经 device tree 节点（`compatible = "espressif,esp-loader"`）配置 `uart` / `default-baudrate` / `higher-baudrate` / `reset-gpios` / `boot-gpios` / `sync-timeout-ms` / `num-trials`，并经 `chosen { zephyr,esp-loader = &esp_loader0; }` 选定。

## Stub flash 占用（README "Flash Size Footprint"）

| 连接模式 | flash rodata 开销 | 触发 |
|---|:---:|---|
| ROM bootloader（`esp_loader_connect()`） | ~0 KB | 不引用 stub，`--gc-sections` 剥除 |
| With stub（`esp_loader_connect_with_stub()`） | **+~87 KB** | 全部 11 个 per-chip stub 链入 |
| SDIO（`CONFIG_SERIAL_FLASHER_PORT_SDIO`） | +~20 KB | 仅 SDIO 相关 stub 链入 |

降低占用：不用 stub 就只调 `esp_loader_connect()`；SDIO 构建关闭 `CONFIG_SERIAL_FLASHER_PORT_SDIO`；纯 CMake 可在源列表剔除 stub `.c`。

## 参考

- `Kconfig` — ESP-IDF/Zephyr 全部选项定义
- `zephyr/Kconfig` — Zephyr 选项
- `docs/configuration.md` — 完整配置文档
- `docs/platform-setup.md` — 各平台构建变量
- `idf_component.yml` — 组件依赖规则
