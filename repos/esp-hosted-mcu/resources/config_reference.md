# ESP-Hosted-MCU 配置（Kconfig）快速参考

> 所有符号来自仓库根 `Kconfig`（host 侧 menuconfig）与 `slave/` 工程的 menuconfig（部分 Example configuration）。`idf.py menuconfig` 路径以 `Component config -> ESP-Hosted config` 为入口。

## 1. 顶层与协处理器型号选择

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_HOSTED_ENABLED` | 启用 ESP-Hosted 组件 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32` | 协处理器 = ESP32 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32S2` | ESP32-S2 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32S3` | ESP32-S3 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32C2` | ESP32-C2 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32C3` | ESP32-C3 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32C6` | ESP32-C6 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32C5` | ESP32-C5 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32C61` | ESP32-C61 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32H2` | ESP32-H2 |
| `CONFIG_ESP_HOSTED_CP_TARGET_ESP32H4` | ESP32-H4 |
| `CONFIG_ESP_HOSTED_IDF_SLAVE_TARGET` | slave 基于 IDF 的目标 |

> host 的 `ESP_HOSTED_CP_TARGET_*` 必须与 slave 的 `idf.py set-target` 一致。

### ESP32-P4 开发板预设
`CONFIG_ESP_HOSTED_P4_DEV_BOARD_NONE` / `_FUNC_BOARD`、`CONFIG_ESP32P4_EYE_C6_BOARD`、`CONFIG_ESP_HOSTED_P4X_C5_DEV_BOARD_FUNC_BOARD`、`CONFIG_ESP_HOSTED_P4_C5_CORE_BOARD` / `_P4_C6_CORE_BOARD` / `_P4_C61_CORE_BOARD`。

## 2. 传输介质选择

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_HOSTED_SPI_HOST_INTERFACE` | SPI 全双工 |
| `CONFIG_ESP_HOSTED_SDIO_HOST_INTERFACE` | SDIO |
| `CONFIG_ESP_HOSTED_SPI_HD_HOST_INTERFACE` | SPI 半双工 |
| `CONFIG_ESP_HOSTED_UART_HOST_INTERFACE` | UART |

可选显示开关：`CONFIG_ESP_HOSTED_PRIV_SDIO_OPTION`、`CONFIG_ESP_HOSTED_PRIV_SPI_HD_OPTION`。

## 3. SPI 全双工（`SPI Configuration` 菜单）

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_HOSTED_SPI_MODE` | SPI mode（0/1/2/3），含 ESP32 与 ESP32XX 分组 |
| `CONFIG_ESP_HOSTED_SPI_CONTROLLER` / `CONFIG_ESP_HOSTED_SPI_HSPI` / `_VSPI` | SPI 控制器选择 |
| `CONFIG_ESP_HOSTED_SPI_GPIO_MOSI/MISO/CLK/CS` | 全双工引脚 |
| `CONFIG_ESP_HOSTED_SPI_GPIO_HANDSHAKE` | Handshake 引脚 |
| `CONFIG_ESP_HOSTED_SPI_GPIO_DATA_READY` | Data Ready 引脚 |
| `CONFIG_ESP_HOSTED_SPI_GPIO_RESET_SLAVE` | 复位 slave 引脚 |
| `CONFIG_ESP_HOSTED_HS_ACTIVE_HIGH/LOW` | Handshake 有效电平 |
| `CONFIG_ESP_HOSTED_DR_ACTIVE_HIGH/LOW` | Data Ready 有效电平 |
| `CONFIG_ESP_HOSTED_SPI_RESET_ACTIVE_HIGH/LOW` | Reset 有效电平 |
| `CONFIG_ESP_HOSTED_SPI_CLK_FREQ` | SPI 时钟（MHz） |
| `CONFIG_ESP_HOSTED_SPI_FREQ_ESP32` / `_ESP32C6` / `_ESP32H2` / `_ESP32XX` | 按协处理器默认频率（P4+C6 默认 40） |
| `CONFIG_ESP_HOSTED_SPI_TX_Q_SIZE` / `_RX_Q_SIZE` | 收发队列大小 |
| `CONFIG_ESP_HOSTED_SPI_CHECKSUM` | 启用 checksum（建议启用，SPI 无硬件检错） |

## 4. SDIO（`Hosted SDIO Configuration` 菜单）

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_HOSTED_SDIO_BUS_WIDTH` + `_4_BIT_BUS` / `_1_BIT_BUS` | 总线宽度 |
| `CONFIG_ESP_HOSTED_SDIO_CLOCK_FREQ_KHZ` | 时钟（kHz），最高 50 MHz |
| `CONFIG_ESP_HOSTED_SDIO_SLOT` + `_SLOT_0` / `_SLOT_1` | SDIO slot（P4 Slot1 默认可重映射，Slot0 固定） |
| `CONFIG_ESP_HOSTED_SDIO_PIN_CMD/CLK/D0/D1/D2/D3` | SDIO 引脚（含 1-bit/4-bit 分组与 slot 分组变体） |
| `CONFIG_ESP_HOSTED_SDIO_GPIO_RESET_SLAVE` | 复位 slave 引脚 |
| `CONFIG_ESP_HOSTED_SDIO_TX_Q_SIZE` / `_RX_Q_SIZE` | 队列大小（默认 20） |
| `CONFIG_ESP_HOSTED_SDIO_RESET_DELAY_MS` | 复位延时 |
| `CONFIG_ESP_HOSTED_SDIO_CHECKSUM` | checksum |
| `CONFIG_ESP_HOSTED_SDIO_RESET_ACTIVE_HIGH/LOW` | Reset 极性 |
| `CONFIG_ESP_HOSTED_SDIO_OPTIMIZATION_RX_NONE` / `_RX_MAX_SIZE` / `_RX_STREAMING_MODE` | SDIO 接收优化（Streaming 默认 / Packet / 无优化） |
| `CONFIG_ESP_HOSTED_SD_PWR_CTRL_LDO_INTERNAL_IO` / `_LDO_IO_ID` | SDIO 电源 LDO 配置 |

## 5. SPI 半双工（`SPI Half-duplex Configuration` 菜单）

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_HOSTED_SPI_HD_MODE` | SPI HD 模式 |
| `CONFIG_ESP_HOSTED_SPI_HD_INTERFACE_NUM_DATA_LINES` + `_4_DATA_LINES` / `_2_DATA_LINES` / `_1_DATA_LINE` | 数据线数量（Quad/Dual/1-bit） |
| `CONFIG_ESP_HOSTED_SPI_HD_DATA_READY_ENABLED` | 启用 Data Ready |
| `CONFIG_ESP_HOSTED_SPI_HD_POLL_INTERVAL_MS` | 轮询间隔 |
| `CONFIG_ESP_HOSTED_SPI_HD_GPIO_CS/CLK/D0/D1/D2/D3/DATA_READY/RESET_SLAVE` | 引脚 |
| `CONFIG_ESP_HOSTED_SPI_HD_DR_ACTIVE_HIGH/LOW` | Data Ready 极性 |
| `CONFIG_ESP_HOSTED_SPI_HD_RESET_ACTIVE_HIGH/LOW` | Reset 极性 |
| `CONFIG_ESP_HOSTED_SPI_HD_CLK_FREQ` | 时钟（P4+C6 默认 40） |
| `CONFIG_ESP_HOSTED_SPI_HD_FREQ_ESP32C6` / `_ESP32XX` | 按协处理器默认 |
| `CONFIG_ESP_HOSTED_SPI_HD_TX_Q_SIZE` / `_RX_Q_SIZE` | 队列大小 |
| `CONFIG_ESP_HOSTED_SPI_HD_CHECKSUM` | checksum |

## 6. UART（`UART Configuration` 菜单）

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_HOSTED_UART_PORT` | UART 端口号 |
| `CONFIG_ESP_HOSTED_UART_PIN_TX` / `_PIN_RX` | TX/RX 引脚 |
| `CONFIG_ESP_HOSTED_UART_BAUDRATE` | 波特率（参考 921600） |
| `CONFIG_ESP_HOSTED_UART_NUM_DATA_BITS` + `_5/_6/_7/_8` | 数据位 |
| `CONFIG_ESP_HOSTED_UART_PARITY` + `_NONE/_EVEN/_ODD` | 校验 |
| `CONFIG_ESP_HOSTED_UART_STOP_BITS` + `_1/_1_5/_2` | 停止位 |
| `CONFIG_ESP_HOSTED_UART_GPIO_RESET_SLAVE` | 复位 slave 引脚 |
| `CONFIG_ESP_HOSTED_UART_TX_Q_SIZE` / `_RX_Q_SIZE` | 队列大小 |
| `CONFIG_ESP_HOSTED_UART_CHECKSUM` | checksum |
| `CONFIG_ESP_HOSTED_UART_RESET_ACTIVE_HIGH/LOW` | Reset 极性 |

## 7. 公共复位策略（`Common Slave Reset Strategy` 菜单）

| 符号 | 说明 |
|---|---|
| `CONFIG_ESP_HOSTED_SLAVE_RESET_ON_EVERY_HOST_BOOTUP` | 每次 host 启动都复位 slave（默认，最稳） |
| `CONFIG_ESP_HOSTED_SLAVE_RESET_ONLY_IF_NECESSARY` | 仅在必要时复位 |
| `CONFIG_ESP_HOSTED_HOST_RESTART_NO_COMMUNICATION_WITH_SLAVE` | host 重启时不与 slave 通信 |
| `CONFIG_ESP_HOSTED_HOST_RESTART_NO_COMMUNICATION_WITH_SLAVE_TIMEOUT` | 上述超时 |
| `CONFIG_ESP_HOSTED_GPIO_SLAVE_RESET_SLAVE` | 用于复位 slave 的 host GPIO |

## 8. 功能开关（features / 电源 / 共存 / GPIO 复位）

由对应功能文档与 Kconfig 控制，常见项：
- `CONFIG_ESP_HOSTED_CP_EXT_COEX` — 外部共存 API（见 `host/api/include/esp_hosted_cp_ext_coex.h`）
- "Allow host to power save" — host 省电（host 侧 + slave 侧各开一次，含 deep sleep、Host Wakeup GPIO、Level）
- Network Split — 见 `docs/feature_network_split.md`
- iTWT — 仅 Wi-Fi 6 协处理器（C6/C5）支持，见 `examples/host_wifi_itwt/`

## 9. slave 侧 menuconfig（`Example configuration`）

slave 工程在 `Example configuration -> Bus Config in between Host and Co-processor` 下选择传输介质（SPI Full-duplex / SDIO / SPI HD / UART），并配置对应子菜单（引脚、时钟、checksum、队列大小、复位引脚等）。OpenThread RCP 在 `Example Configuration -> Enable OpenThread RCP` 下配置专用 UART。

## 10. 高性能 lwIP / Wi-Fi Remote（host 侧）

P4+C6 推荐合并到 `sdkconfig.defaults.<host>`：
```text
CONFIG_WIFI_RMT_STATIC_RX_BUFFER_NUM=16
CONFIG_WIFI_RMT_DYNAMIC_RX_BUFFER_NUM=64
CONFIG_WIFI_RMT_DYNAMIC_TX_BUFFER_NUM=64
CONFIG_WIFI_RMT_AMPDU_TX_ENABLED=y
CONFIG_WIFI_RMT_TX_BA_WIN=32
CONFIG_WIFI_RMT_AMPDU_RX_ENABLED=y
CONFIG_WIFI_RMT_RX_BA_WIN=32
CONFIG_LWIP_TCP_SND_BUF_DEFAULT=65534
CONFIG_LWIP_TCP_WND_DEFAULT=65534
CONFIG_LWIP_TCP_RECVMBOX_SIZE=64
CONFIG_LWIP_UDP_RECVMBOX_SIZE=64
CONFIG_LWIP_TCPIP_RECVMBOX_SIZE=64
CONFIG_LWIP_TCP_SACK_OUT=y
```
UART + IDF v5.5 IRAM 不足时：`CONFIG_RINGBUF_PLACE_FUNCTIONS_INTO_FLASH=y`。

> `sdkconfig` / `sdkconfig.h` 由 menuconfig 生成，**不要手改**；持久化请写入 `sdkconfig.defaults.<target>`。详细优化见 `docs/performance_optimization.md`。
