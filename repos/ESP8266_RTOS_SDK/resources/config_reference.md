# ESP8266_RTOS_SDK 配置项（Kconfig / menuconfig）速查

> 所有 `CONFIG_*` 符号取自仓库 `Kconfig`、`components/*/Kconfig`、`sdkconfig.rename`。`sdkconfig` 由 `make menuconfig` 生成，不要手改；用 `sdkconfig.defaults` 注入默认值。

## 工具链与路径

| 配置 / 环境变量 | 含义 |
|---|---|
| `IDF_PATH`（环境变量，非 Kconfig）| SDK 根目录绝对路径，构建必需；`include $(IDF_PATH)/make/project.mk` |
| 工具链 | `xtensa-lx106-elf-gcc` v8.4.0（esp-2020r3）；其 bin 目录需加入 `PATH` |
| `CONFIG_TOOLPREFIX` | 编译器前缀，默认 `xtensa-lx106-elf-`（menuconfig: Toolchain configuration）|

## Serial flasher config（烧录相关）

| Kconfig 符号 | 含义 |
|---|---|
| `CONFIG_ESPTOOLPY_PORT` | 串口（如 COM3 / /dev/ttyUSB0）|
| `CONFIG_ESPTOOLPY_BAUD` | 烧录波特率（默认 115200，可选 921600 等）|
| `CONFIG_ESPTOOLPY_FLASHSIZE` | Flash 容量：`1MB` / `2MB` / `4MB` / `2MB-c1` 等，必须与实际一致 |
| `CONFIG_ESPTOOLPY_FLASHMODE` | Flash 模式：`qio`/`qout`/`dio`/`dout` |
| `CONFIG_ESPTOOLPY_FLASHFREQ` | Flash 频率：`40m`/`26m`/`20m`/`80m` |
| `CONFIG_ESPTOOLPY_COMPRESSED` | 烧录用压缩传输（推荐开）|

## Partition Table（分区表）

| Kconfig 符号 | 含义 |
|---|---|
| `CONFIG_PARTITION_TABLE_SINGLE_APP` | "Single factory app, no OTA"（factory @0x10000，无 OTA）|
| `CONFIG_PARTITION_TABLE_TWO_OTA` | "Two OTA app"（ota_0 @0x10000、ota_1 @0x110000、otadata）|
| `CONFIG_PARTITION_TABLE_CUSTOM` | "Custom partition table CSV" |
| `CONFIG_PARTITION_TABLE_CUSTOM_FILENAME` | 自定义 CSV 文件名（工程相对路径）|
| `CONFIG_PARTITION_TABLE_FILENAME` | 当前生效的 CSV/内置表名（菜单自动填）|

> 分区表烧到 flash 0x8000；app 分区必须落在单个 1MB 集成分区内。详见 `docs/en/api-guides/partition-tables.rst`。

## Component config → PHY

| Kconfig 符号 | 含义 |
|---|---|
| `CONFIG_ESP_PHY_INIT_DATA_IN_PARTITION` | PHY 初始化数据从 `phy_init` 分区读取（否则编译进 app）|
| `CONFIG_PHY_DATA_OFFSET` | PHY 分区偏移 |
| `vdd33_const`（PHY 子项，0~255）| ADC 模式：255=测系统电压（TOUT 悬空）；[18,36]=测外部电压（0.1V 单位）；[0,18]或(36,255)=默认 3.3V 参考 |

## Component config → Wi-Fi（CONFIG_ESP8266_WIFI_*）

> 来源 `components/esp8266/Kconfig`。这些宏在 `esp_wifi.h` 里被转成 `WIFI_*` 默认值（见 `WIFI_INIT_CONFIG_DEFAULT`）。

| Kconfig 符号 | 含义 / 默认 |
|---|---|
| `CONFIG_ESP8266_WIFI_AMPDU_RX_ENABLED` | 启用 AMPDU RX（影响单包最大长度 `WIFI_RX_MAX_SINGLE_PKT_LEN`）|
| `CONFIG_ESP8266_WIFI_RX_BA_WIN_SIZE` | Block Ack RX 窗口大小 |
| `CONFIG_ESP8266_WIFI_AMSDU_ENABLED` | 启用 AMSDU RX（单包最大长度升到 3000）|
| `CONFIG_ESP8266_WIFI_RX_BUFFER_NUM` | RX 缓冲数量 |
| `CONFIG_ESP8266_WIFI_RX_PKT_NUM` | RX 包数量 |
| `CONFIG_ESP8266_WIFI_LEFT_CONTINUOUS_RX_BUFFER_NUM` | 连续 RX 左缓冲数 |
| `CONFIG_ESP8266_WIFI_TX_PKT_NUM` | TX 包数量 |
| `CONFIG_ESP8266_WIFI_QOS_ENABLED` | WiFi QoS |
| `CONFIG_ESP8266_WIFI_NVS_ENABLED` | WiFi 用 NVS 存储配置（默认开，影响 `WIFI_NVS_ENABLED`）|
| `CONFIG_ESP8266_WIFI_CONNECT_OPEN_ROUTER_WHEN_PWD_IS_SET` | 设了密码仍允许连开放路由 |
| `CONFIG_ESP8266_WIFI_ENABLE_WPA3_SAE` | 启用 WPA3 SAE |
| `CONFIG_ESP8266_WIFI_DEBUG_LOG_ENABLE` | 启用 WiFi 调试日志（含若干子模块：CORE/SCAN/PM/NVS/TRC/EBUF/NET80211/TIMER/ESPNOW/MAC/WPA/WPS）|
| `CONFIG_ESP8266_WIFI_DEBUG_LOG_ERROR` .. `_VERBOSE` | WiFi 调试日志级别 |

> 内存紧张（`esp_wifi_init` 报 NO_MEM）时，调小 `RX_BUFFER_NUM`/`RX_PKT_NUM`/`TX_PKT_NUM`。

## Component config → Log output

| Kconfig 符号 | 含义 |
|---|---|
| `CONFIG_LOG_DEFAULT_LEVEL` | 默认日志级别：`NONE`/`ERROR`/`WARN`/`INFO`/`DEBUG`/`VERBOSE` |
| `CONFIG_LOG_COLORS` | 日志着色 |
| `CONFIG_LOG_TIMESTAMP` | 日志加时间戳 |

## Example Configuration（项目级，由 Kconfig.projbuild 提供）

> 不同示例提供不同项，常见：

| Kconfig 符号 | 含义 | 来源示例 |
|---|---|---|
| `CONFIG_ESP_WIFI_SSID` | 目标 AP SSID | station / softAP |
| `CONFIG_ESP_WIFI_PASSWORD` | 密码 | station / softAP |
| `CONFIG_ESP_MAXIMUM_RETRY` | Station 最大重试次数 | station |
| `CONFIG_ESP_MAX_STA_CONN` | SoftAP 最大接入站点数 | softAP |
| `CONFIG_ESPNOW_CHANNEL` | ESPNOW 信道 | espnow |
| `CONFIG_ESPNOW_PMK` | ESPNOW 主密钥（16 字节）| espnow |
| `CONFIG_ESPNOW_LMK` | ESPNOW 本地主密钥 | espnow |
| `CONFIG_ESPNOW_SEND_COUNT` / `_SEND_DELAY` / `_SEND_LEN` | ESPNOW 发送参数 | espnow |
| `CONFIG_FIRMWARE_UPGRADE_URL` | OTA 固件下载 URL | simple_ota / native_ota |

## 常用 sdkconfig.defaults 示例

> 把下列内容写入工程根 `sdkconfig.defaults`，首次 `make menuconfig` 后会自动注入（无需手改 sdkconfig）：

```
# Flash 与分区
CONFIG_ESPTOOLPY_FLASHSIZE="4MB"
CONFIG_ESPTOOLPY_FLASHMODE="dio"
CONFIG_ESPTOOLPY_FLASHFREQ="40m"

# 日志
CONFIG_LOG_DEFAULT_LEVEL_INFO=y

# 用自定义分区表（含 SPIFFS storage）
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions_example.csv"

# WiFi
CONFIG_ESP8266_WIFI_NVS_ENABLED=y

# 关闭 vdd33_const=255 测外部电压（按需）
```

## sdkconfig.rename（旧→新符号映射）

仓库根 `sdkconfig.rename` 列出已重命名的 Kconfig 符号映射；升级 SDK 版本时若发现旧符号不存在，先查此文件。
