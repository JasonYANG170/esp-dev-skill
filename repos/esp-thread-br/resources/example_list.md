# esp-thread-br 仓库示例与组件清单

> 全部路径取自仓库实际目录。示例均在 ESP-IDF 下用 `idf.py` 构建；RCP 镜像来自 ESP-IDF 的 `examples/openthread/ot_rcp`（不在本仓库）。

## 示例 (examples/)

| 路径 | 说明 |
|---|---|
| `examples/basic_thread_border_router/` | 官方 Thread Border Router 主示例。默认 ESP32-S3 + ESP32-H2，支持 Wi-Fi/Ethernet backbone、AUTO_START、Web GUI、RCP 自动更新、HTTPS OTA。 |
| `examples/basic_thread_border_router/main/esp_ot_br.c` | 示例 `app_main`：eventfd → init → start → backbone → auto_start，可选 Web/mDNS/OTA。 |
| `examples/basic_thread_border_router/main/esp_ot_config.h` | radio/host/port + RCP_UPDATE 默认配置宏（UART/SPI 两种）。 |
| `examples/basic_thread_border_router/partitions.csv` | nvs / otadata / phy / ota_0(2M) / ota_1(2M) / web_storage(200K) / rcp_fw(640K)。 |
| `examples/basic_thread_border_router/sdkconfig.defaults` | ESP32-S3 + DEV_KIT 主配置（lwIP hooks、OpenThread、OTA、Ethernet W5500）。 |
| `examples/basic_thread_border_router/sdkconfig.defaults.esp32c5` | ESP32-C5 Standalone 覆盖（`ESP_BR_BOARD_STANDALONE`、`ESP_CONSOLE_UART_DEFAULT`）。 |
| `examples/basic_thread_border_router/sdkconfig.defaults.esp32p4` | ESP32-P4 覆盖。 |
| `examples/basic_thread_border_router/sdkconfig.wifi.esp32p4` | ESP32-P4 Wi-Fi 配置。 |
| `examples/basic_thread_border_router/server_certs/` | OTA HTTPS 信任证书（`ca_cert.pem`，自建服务器需替换）。 |
| `examples/basic_thread_border_router/README.md` | 主示例说明：构建、配置、双向 IPv6、SRP、HTTPS OTA、手动 RCP 更新。 |
| `examples/basic_thread_border_router/README_standalone_RCP.md` | Standalone 模组接线与配置（UART/SPI）。 |
| `examples/basic_thread_border_router/README_esp32p4.md` | ESP32-P4 专用说明。 |
| `examples/m5stack_thread_border_router/` | M5Stack CoreS3 Thread Border Router 示例（带触摸屏、ePSKc 凭证共享、SoftAP 配网）。 |
| `examples/m5stack_thread_border_router/main/` | M5Stack `app_main`（结构同 basic，额外初始化屏幕）。 |
| `examples/common/thread_border_router/` | 被两示例复用的启动组件：`border_router_launch.c` + `Kconfig.projbuild`（板型/引脚/行为开关）。 |
| `examples/common/thread_border_router/include/border_router_launch.h` | `launch_openthread_border_router()` 声明。 |
| `examples/common/thread_border_router_m5stack/` | M5Stack 专用组件：UI、动画、ePSKc 页面、屏幕调光、电源管理 + `Kconfig.projbuild`。 |
| `examples/common/thread_border_router_m5stack/src/br_m5stack_epskc_page.c` | Credential Sharing 实现：`otBorderAgentEphemeralKeyStart/Stop`、`otVerhoeffChecksumCalculate`、meshcop-e 回调。 |
| `examples/common/thread_border_router_m5stack/Kconfig.projbuild` | M5Stack 专用 Kconfig：`OPENTHREAD_EPHEMERALKEY_LIFE_TIME`（默认 100）、`OPENTHREAD_EPHEMERALKEY_PORT`（默认 49180）。 |
| `examples/common/thread_border_router/src/border_router_launch.c` | BR 启动封装：含 `#if CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE` → `ot_external_coexist_init()`。 |
| `examples/common/thread_border_router/Kconfig.projbuild` | `OPENTHREAD_BR_START_WEB`（`select OPENTHREAD_COMMISSIONER/JOINER`）、`OPENTHREAD_BR_SOFTAP_SETUP`、`OPENTHREAD_BR_AUTO_START`。 |
| `docs/en/codelab/dhcpv6_pd.rst` | DHCPv6 PD Codelab（3.8）：Kea 配置、`ot br pd enable/state/omrprefix`、ping 验证。 |

## 组件 (components/)

| 路径 | 说明 |
|---|---|
| `components/esp_rcp_update/` | RCP 固件更新组件：初始化、序列号/回滚、SPIFFS 打包、`create_ota_image.py`。 |
| `components/esp_rcp_update/include/esp_rcp_update.h` | `esp_rcp_update_config_t`、`esp_rcp_update_init/update/reset/...`。 |
| `components/esp_rcp_update/include/esp_rcp_ota.h` | RCP OTA 流式接口（begin/receive/end/abort）。 |
| `components/esp_rcp_update/include/esp_rcp_firmware.h` | filetag enum、`esp_rcp_subfile_info_t`。 |
| `components/esp_rcp_update/include/esp_ot_rcp_update.h` | `esp_ot_try_update_rcp`、`esp_ot_register_rcp_handler`、`esp_ot_update_rcp_if_different`。 |
| `components/esp_br_http_ota/` | BR HTTPS OTA 辅助组件。 |
| `components/esp_br_http_ota/include/esp_br_http_ota.h` | `esp_br_http_ota()`、`OTA_MAX_WRITE_SIZE`。 |
| `components/esp_ot_br_server/` | BR Web Server（REST + GUI，兼容 ot-br-posix）。 |
| `components/esp_ot_br_server/include/esp_br_web.h` | `esp_br_web_start()`。 |
| `components/esp_ot_br_server/include/esp_br_wifi_config.h` | SoftAP 配网 API（start/get/stop/is_active/get_softap_info）。 |
| `components/esp_ot_br_server/src/openapi.yaml` | Thread REST API OpenAPI 定义。 |
| `components/esp_ot_br_server/frontend/` | Web GUI 前端资源。 |
| `components/esp_ot_cli_extension/` | OpenThread CLI 扩展命令组件。 |
| `components/esp_ot_cli_extension/include/esp_ot_cli_extension.h` | `esp_cli_custom_command_init()`、`esp_wifi_address_event_t`。 |
| `components/esp_ot_cli_extension/src/esp_ot_cli_extension.c` | 命令注册表（curl/dns64server/heapdiag/ip/loglevel/mcast/nvsdiag/ota/otrcp/tcp*/udp*/wifi/brlibcheck）。 |
| `components/esp_ot_cli_extension/include/esp_ot_wifi_cmd.h` | `wifi` CLI + NVS 配置 API。 |
| `components/esp_ot_cli_extension/include/esp_ot_ota_commands.h` | `ota` CLI + `esp_set_ota_server_cert`。 |
| `components/esp_ot_cli_extension/include/esp_ot_rcp_commands.h` | `otrcp` CLI。 |
| `components/esp_ot_cli_extension/include/esp_ot_dns64.h` | `dns64server` CLI。 |
| `components/esp_ot_cli_extension/include/esp_ot_curl.h` | `curl` CLI。 |
| `components/esp_ot_cli_extension/include/esp_ot_ip.h` | `ip` CLI。 |
| `components/esp_ot_cli_extension/include/esp_ot_udp_socket.h` | `mcast`/`udpsockserver`/`udpsockclient` CLI + socket helper。 |
| `components/esp_ot_cli_extension/include/esp_ot_tcp_socket.h` | `tcpsockserver`/`tcpsockclient` CLI。 |
| `components/esp_ot_cli_extension/include/esp_ot_heap_diag.h` | `heapdiag` CLI + init。 |
| `components/esp_ot_cli_extension/include/esp_ot_loglevel.h` | `loglevel` CLI。 |
| `components/esp_ot_cli_extension/include/esp_ot_nvs_diag.h` | `nvsdiag` CLI。 |
| `components/esp_ot_cli_extension/include/esp_ot_br_lib_compati_check.h` | `brlibcheck` CLI。 |

## 外部依赖（ESP-IDF，不在本仓库）

| 来源 | 说明 |
|---|---|
| ESP-IDF `examples/openthread/ot_rcp` | RCP 固件（ESP32-H2/C6），被 BR 构建期打包。 |
| ESP-IDF `examples/openthread/ot_cli` | Thread CLI 设备固件（用于入网验证）。 |
| ESP-IDF `examples/openthread/ot_trel` | TREL Wi-Fi 设备固件。 |
| ESP-IDF `examples/openthread/ot_br` | IDF 自带最小 BR 示例（Standalone 快速上手推荐）。 |
