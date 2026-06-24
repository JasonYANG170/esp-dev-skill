# esp-thread-br 配置参考

> 全部取自仓库 `examples/basic_thread_border_router/sdkconfig.defaults`、`examples/common/thread_border_router/Kconfig.projbuild`、`examples/common/thread_border_router_m5stack/Kconfig.projbuild`、各 `sdkconfig.defaults.*`。OpenThread/ESP-IDF 原生 Kconfig 仅列本仓库直接相关项。

## 1. 工程专属 Kconfig（`ESP Thread Border Router Example`）

### 板型 choice
| 配置 | 含义 |
|---|---|
| `CONFIG_ESP_BR_BOARD_STANDALONE` | 独立模组（含 ESP32-P4-Function-EV） |
| `CONFIG_ESP_BR_BOARD_DEV_KIT` | 官方 BR 开发板（ESP32-S3 + ESP32-H2） |
| `CONFIG_ESP_BR_BOARD_M5STACK_CORES3` | M5Stack CoreS3 |

### RCP 目标 choice
| 配置 | 含义 | 影响 |
|---|---|---|
| `CONFIG_ESP_BR_H2_TARGET` | RCP = ESP32-H2 | `ESP_BR_RCP_TARGET_ID = ESP32H2_CHIP` |
| `CONFIG_ESP_BR_C6_TARGET` | RCP = ESP32-C6 | `ESP_BR_RCP_TARGET_ID = ESP32C6_CHIP` |

### 引脚（默认值随板型）
| 配置 | DEV_KIT | STANDALONE | M5STACK |
|---|---|---|---|
| `CONFIG_PIN_TO_RCP_RESET` | 7 | 7 | 7 |
| `CONFIG_PIN_TO_RCP_BOOT` | 8 | 8 | 18 |
| `CONFIG_PIN_TO_RCP_TX` | 17 | 4 | 10 |
| `CONFIG_PIN_TO_RCP_RX` | 18 | 5 | 17 |
| `CONFIG_PIN_TO_RCP_CS` (SPI) | 10 | 20 | 13 |
| `CONFIG_PIN_TO_RCP_SCLK` (SPI) | 12 | 22 | 36 |
| `CONFIG_PIN_TO_RCP_MISO` (SPI) | 13 | 23 | 35 |
| `CONFIG_PIN_TO_RCP_MOSI` (SPI) | 11 | 21 | 37 |

### 行为开关
| 配置 | 含义 |
|---|---|
| `CONFIG_OPENTHREAD_BR_AUTO_START` | 自动连 Wi-Fi/Eth + 自动起网 |
| `CONFIG_OPENTHREAD_BR_START_WEB` | 启 Web Server（`select OPENTHREAD_COMMISSIONER/JOINER`） |
| `CONFIG_OPENTHREAD_BR_SOFTAP_SETUP` | SoftAP 首次配网（AP `ESP-ThreadBR-XXXX`） |

## 2. M5Stack 工程额外 Kconfig

| 配置 | 含义 |
|---|---|
| `CONFIG_RADIO_CO_PROCESSOR_MODULE_H2` | 使用 Module Gateway H2 |
| `CONFIG_RADIO_CO_PROCESSOR_UNIT_H2` | 使用 Unit Gateway H2（需 `AUTO_UPDATE_RCP=n`、`PIN_TO_RCP_TX=18`、`PIN_TO_RCP_RX=17`） |
| `CONFIG_OPENTHREAD_EPHEMERALKEY_LIFE_TIME` | ephemeral key 寿命（秒，默认 100） |
| `CONFIG_OPENTHREAD_EPHEMERALKEY_PORT` | ePSKc UDP 端口（默认 49180） |

## 3. OpenThread 相关（`sdkconfig.defaults` 默认开）
| 配置 | 作用 |
|---|---|
| `CONFIG_OPENTHREAD_ENABLED` | 启用 OpenThread |
| `CONFIG_OPENTHREAD_BORDER_ROUTER` | BR 角色 |
| `CONFIG_OPENTHREAD_CLI_OTA` | `ota` CLI + `esp_set_ota_server_cert` |
| `CONFIG_OPENTHREAD_RCP_COMMAND` | `otrcp` CLI |
| `CONFIG_OPENTHREAD_RADIO_SPINEL_UART` / `..._SPI` | 主控↔RCP 接口 |
| `CONFIG_OPENTHREAD_TASK_SIZE` | OT 任务栈（默认 8192） |
| `CONFIG_OPENTHREAD_DNS64_CLIENT` | `dns64server` CLI |
| `CONFIG_OPENTHREAD_NVS_DIAG` | `nvsdiag` CLI |
| `CONFIG_OPENTHREAD_BR_LIB_CHECK` | `brlibcheck` CLI |
| `CONFIG_OPENTHREAD_CLI_WIFI` | `wifi` CLI + NVS |
| `CONFIG_OPENTHREAD_RADIO_TREL` | TREL（eventfd +1） |

## 4. RCP 更新相关
| 配置 | 作用 |
|---|---|
| `CONFIG_AUTO_UPDATE_RCP` | 启动时自动更新 RCP（需挂 `rcp_fw`） |
| `CONFIG_CREATE_OTA_IMAGE_WITH_RCP_FW` | 构建期生成 `build/ota_with_rcp_image` |
| `CONFIG_RCP_PARTITION_NAME` | RCP 镜像分区名（默认 `rcp_fw`） |
| `CONFIG_RCP_PATH_NAME` | RCP 镜像子路径（默认 `ot_rcp`） |
| `CONFIG_RCP_SRC_DIR` | ot_rcp 构建目录（CMake 用） |
| `CONFIG_DEFAULT_PIN_TO_RCP_TX/RX/RESET/BOOT` | `ESP_RCP_UPDATE_DEFAULT_CONFIG()` 用 |

## 5. lwIP（双向 IPv6 / 路由必需）
```bash
CONFIG_LWIP_IPV6_NUM_ADDRESSES=12      # IDF v5.3.1+ 必须 12；早期为 8
CONFIG_LWIP_IPV6_FORWARD=y
CONFIG_LWIP_MULTICAST_PING=y
CONFIG_LWIP_NETIF_STATUS_CALLBACK=y
CONFIG_LWIP_HOOK_IP6_ROUTE_DEFAULT=y
CONFIG_LWIP_HOOK_ND6_GET_GW_DEFAULT=y
CONFIG_LWIP_HOOK_IP6_INPUT_CUSTOM=y
CONFIG_LWIP_IPV6_AUTOCONFIG=y
CONFIG_LWIP_HOOK_IP6_SELECT_SRC_ADDR_CUSTOM=y
CONFIG_LWIP_FORCE_ROUTER_FORWARDING=y
```

## 6. mbedTLS（Thread DTLS/EC-JPAKE）
```bash
CONFIG_MBEDTLS_CMAC_C=y
CONFIG_MBEDTLS_SSL_PROTO_DTLS=y
CONFIG_MBEDTLS_KEY_EXCHANGE_ECJPAKE=y
CONFIG_MBEDTLS_ECJPAKE_C=y
```

## 7. mDNS
```bash
CONFIG_MDNS_MULTIPLE_INSTANCE=y
CONFIG_MDNS_MAX_SERVICES=200
```

## 8. 优化（OTA 推荐）
```bash
CONFIG_COMPILER_OPTIMIZATION_SIZE=y
CONFIG_NEWLIB_NANO_FORMAT=y
```

## 9. Flash / 分区
```bash
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y       # 4MB 变体改 _4MB 且 ota 分区改 1500K
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
```

## 10. Ethernet（官方 BR 板 Sub-Ethernet / W5500，仅 Ethernet backbone）
```bash
CONFIG_EXAMPLE_CONNECT_ETHERNET=y
CONFIG_EXAMPLE_CONNECT_WIFI=n
CONFIG_EXAMPLE_USE_W5500=y
CONFIG_EXAMPLE_ETH_SPI_HOST=2
CONFIG_EXAMPLE_ETH_SPI_SCLK_GPIO=21
CONFIG_EXAMPLE_ETH_SPI_MOSI_GPIO=45
CONFIG_EXAMPLE_ETH_SPI_MISO_GPIO=38
CONFIG_EXAMPLE_ETH_SPI_CS_GPIO=41
CONFIG_EXAMPLE_ETH_SPI_CLOCK_MHZ=36
CONFIG_EXAMPLE_ETH_SPI_INT_GPIO=39
CONFIG_EXAMPLE_ETH_PHY_RST_GPIO=40
CONFIG_EXAMPLE_ETH_PHY_ADDR=1
```

## 11. HTTP Server（Web GUI）
```bash
CONFIG_HTTPD_MAX_REQ_HDR_LEN=1024
```

## 12. 控制台
```bash
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y    # 官方 BR 板默认
# 其它硬件可改 CONFIG_ESP_CONSOLE_UART_DEFAULT=y
```
> 启用 `ESP_CONSOLE_SECONDARY_USB_SERIAL_JTAG` 时需手动指定 `OPENTHREAD_CONSOLE_TYPE_*`。

## 13. 外部 RF 共存
```bash
CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE=y   # basic_thread_border_router 与 ot_rcp 两端同时开
```
> 3-wire + 第 4 根 TX 信号；仅在 Wi-Fi 与 15.4 临近信道干扰显著时才有意义。

## 14. 组件依赖版本（`idf_component.yml`）
```yaml
espressif/mdns: "^1.0.0"
espressif/esp_ot_cli_extension: "~2.0.0"
espressif/esp_rcp_update: "~1.6.0"
idf: ">=5.1.0"
```
