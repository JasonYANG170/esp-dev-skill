# ESP-Insights Configuration Reference (Kconfig)

> 所有符号来自仓库 `components/*/Kconfig`。默认值即仓库内 default。grep `CONFIG_` 前缀的即 sdkconfig 键。

---

## ESP Insights（`components/esp_insights/Kconfig`）

| Kconfig | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_INSIGHTS_ENABLED` | bool | n | 总开关，自动 select `DIAG_ENABLE_WRAP_LOG_FUNCTIONS` |
| `CONFIG_ESP_INSIGHTS_DEBUG_ENABLED` | bool | n | Insights 调试打印（依赖 ENABLED） |
| `CONFIG_ESP_INSIGHTS_DEBUG_PRINT_JSON` | bool | y | dump 时打印 JSON 而非 hexdump（依赖 DEBUG_ENABLED） |
| `CONFIG_ESP_INSIGHTS_COREDUMP_ENABLE` | bool | y | core dump 摘要（依赖 coredump-to-flash **且** ELF） |
| `CONFIG_ESP_INSIGHTS_TRANSPORT_MQTT` | choice | — | 选 MQTT（与 HTTPS 二选一） |
| `CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS` | choice | 默认 | 选 HTTPS（默认） |
| `CONFIG_ESP_INSIGHTS_CMD_RESP_ENABLED` | bool | n | command-response（依赖 ENABLED **且** TRANSPORT_MQTT） |
| `CONFIG_ESP_INSIGHTS_TRANSPORT_HTTPS_HOST` | string | `https://client.insights.espressif.com` | HTTPS host（依赖 TRANSPORT_HTTPS） |
| `CONFIG_ESP_INSIGHTS_CLOUD_POST_MIN_INTERVAL_SEC` | int | 60 | 上报最小间隔（秒） |
| `CONFIG_ESP_INSIGHTS_CLOUD_POST_MAX_INTERVAL_SEC` | int | 240 | 上报最大间隔（秒） |
| `CONFIG_ESP_INSIGHTS_META_VERSION_10` | bool | y | 旧元数据（1.0，仅 key）；置 n 用 2.0（tag+key） |

> 动态上报逻辑：上一周期发了数据 → 间隔翻倍；否则减半，夹在 [MIN, MAX] 之间。

---

## Diagnostics（`components/esp_diagnostics/Kconfig`）

| Kconfig | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_DIAG_LOG_MSG_ARG_FORMAT_TLV` | choice | 默认 | 日志参数存为 TLV（Type/Len/Value） |
| `CONFIG_DIAG_LOG_MSG_ARG_FORMAT_STRING` | choice | — | 日志参数存为完整 vsnprintf 字符串 |
| `CONFIG_DIAG_LOG_MSG_ARG_MAX_SIZE` | int | 64 | 日志参数缓冲上限（32–255） |
| `CONFIG_DIAG_LOG_DROP_WIFI_LOGS` | bool | y | 丢弃 Wi-Fi 日志（每条 Wi-Fi 日志会衍生 3 条诊断日志） |
| `CONFIG_DIAG_ENABLE_WRAP_LOG_FUNCTIONS` | bool | n | 包装 esp_log_write/writev（ENABLED 时自动选中） |
| `CONFIG_DIAG_ENABLE_METRICS` | bool | y | metrics 总开关 |
| `CONFIG_DIAG_METRICS_MAX_COUNT` | int | 20 | 可注册 metric 上限 |
| `CONFIG_DIAG_ENABLE_HEAP_METRICS` | bool | y | heap metrics（free/最大块/历史最小） |
| `CONFIG_DIAG_HEAP_POLLING_INTERVAL` | int | 30 | heap 采集间隔（秒，30–86400） |
| `CONFIG_DIAG_ENABLE_WIFI_METRICS` | bool | y | Wi-Fi RSSI / 历史最小 RSSI |
| `CONFIG_DIAG_WIFI_POLLING_INTERVAL` | int | 30 | Wi-Fi 采集间隔（秒，30–86400） |
| `CONFIG_DIAG_ENABLE_VARIABLES` | bool | y | variables 总开关 |
| `CONFIG_DIAG_VARIABLES_MAX_COUNT` | int | 20 | 可注册 variable 上限 |
| `CONFIG_DIAG_ENABLE_NETWORK_VARIABLES` | bool | y | 网络 variables（SSID/BSSID/channel/auth/IP/netmask/gateway/断连原因） |
| `CONFIG_DIAG_MORE_NETWORK_VARS` | bool | n | 进阶网络 variables |
| `CONFIG_DIAG_USE_EXTERNAL_LOG_WRAP` | bool | n | 由外部包装日志，本组件改用 `esp_diag_log_write/writev` 数据摄入 |

---

## Diagnostics data store（`components/esp_diag_data_store/Kconfig`）

| Kconfig | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_DIAG_DATA_STORE_RTC` | choice | 默认 | 存 RTC memory（依赖 `SOC_RTC_MEM_SUPPORTED`） |
| `CONFIG_DIAG_DATA_STORE_RAM` | choice | — | 存内部 RAM（断电丢） |
| `CONFIG_DIAG_DATA_STORE_FLASH` | choice | — | 占位，当前不支持 |
| `CONFIG_DIAG_DATA_STORE_DBG_PRINTS` | bool | false | 数据存储调试打印 |
| `CONFIG_DIAG_DATA_STORE_REPORTING_WATERMARK_PERCENT` | int | 80 | 触发低内存事件的水位线（50–90） |
| `CONFIG_RTC_STORE_DATA_SIZE` | int | 6144（ESP32: 3072） | RTC 总大小（512–7168；依赖 DIAG_DATA_STORE_RTC） |
| `CONFIG_RTC_STORE_CRITICAL_DATA_SIZE` | int | 4096（ESP32: 2048） | critical 区，余下给 non-critical（512–DATA_SIZE） |
| `CONFIG_RAM_STORE_DATA_SIZE` | int | 6144 | RAM 总大小（依赖 DIAG_DATA_STORE_RAM） |
| `CONFIG_RAM_STORE_CRITICAL_DATA_SIZE` | int | 4096 | RAM critical 区 |
| `CONFIG_FLASH_STORE_PARTITION_LABEL` | string | `diag_data` | 诊断数据分区名（依赖 DIAG_DATA_STORE_FLASH，暂不支持） |

---

## 应用侧配套（来自 examples/minimal_diagnostics/sdkconfig.defaults）

下列非 Insights 自身 Kconfig，但接入时通常必须：

```
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_FILENAME="partitions.csv"
CONFIG_PARTITION_TABLE_OFFSET=0x8000
CONFIG_PARTITION_TABLE_MD5=y

# core dump（新版 IDF 用 ESP_COREDUMP_* 符号；旧版用 ESP32_COREDUMP_*）
CONFIG_ESP32_ENABLE_COREDUMP=y
CONFIG_ESP32_ENABLE_COREDUMP_TO_FLASH=y
CONFIG_ESP32_COREDUMP_DATA_FORMAT_ELF=y
CONFIG_ESP32_COREDUMP_CHECKSUM_CRC32=y
CONFIG_ESP32_CORE_DUMP_MAX_TASKS_NUM=64
CONFIG_ESP32_CORE_DUMP_STACK_SIZE=1024

# 省 TLS 内存
CONFIG_MBEDTLS_DYNAMIC_BUFFER=y
CONFIG_MBEDTLS_DYNAMIC_FREE_PEER_CERT=y
CONFIG_MBEDTLS_DYNAMIC_FREE_CONFIG_DATA=y
```

## 组件依赖版本（`components/esp_insights/idf_component.yml`）

```yaml
version: "1.3.3"
dependencies:
  idf: { version: ">=5.1" }
  espressif/rmaker_common: { version: "^1.5" }
  espressif/esp_diag_data_store: { version: "~1.1.0", override_path: '../esp_diag_data_store/' }
  espressif/esp_diagnostics:     { version: ">=1.3.0", override_path: '../esp_diagnostics/' }
  espressif/cbor:                { version: "~0.6" }
```
