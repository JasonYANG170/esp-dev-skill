# esp-protocols 配置参考（Kconfig）

> 取自仓库 `components/*/Kconfig` 与 `idf_component.yml`。在 menuconfig 中位于 "Component config" 下各组件菜单。

## esp_modem（`components/esp_modem/Kconfig`）

| Kconfig | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_MODEM_USE_PPP_MODE` | bool | y | 启用 lwip PPP netif（自动 select `LWIP_PPP_SUPPORT`）；仅用 AT 命令时可关 |
| `CONFIG_ESP_MODEM_CMUX_DEFRAGMENT_PAYLOAD` | bool | y | CMUX 内部重组分片。**A76xx（2 字节 payload）需设 n** |
| `CONFIG_ESP_MODEM_USE_INFLATABLE_BUFFER_IF_NEEDED` | bool | n | DTE 缓冲不足时动态扩展；CMUX defrag=n 时可避免命令分片丢失 |
| `CONFIG_ESP_MODEM_CMUX_DELAY_AFTER_DLCI_SETUP` | int(ms) | 0 | 建立下一个虚拟终端前的等待（SIM800 MSC 问题） |
| `CONFIG_ESP_MODEM_CMUX_DELAY_AFTER_NETIF_DISCONNECT` | int(ms) | 0 | PPP 断开后关闭虚拟终端前的等待 |
| `CONFIG_ESP_MODEM_CMUX_USE_SHORT_PAYLOADS_ONLY` | bool | n | 仅用 1 字节 CMUX 长度字段（<128 字节），嘈杂环境更稳 |
| `CONFIG_ESP_MODEM_ADD_CUSTOM_MODULE` | bool | n | 启用自定义模组 C-API |
| `CONFIG_ESP_MODEM_CUSTOM_MODULE_HEADER` | string | `custom_module.hpp` | 自定义模组头文件名（依赖上面） |
| `CONFIG_ESP_MODEM_C_API_STR_MAX` | int | 128 | C-API 文本输出缓冲最小字节数（IMSI/IMEI/operator/at 等） |
| `CONFIG_ESP_MODEM_URC_HANDLER` | bool | n | 启用 `esp_modem_set_urc()` |
| `CONFIG_ESP_MODEM_PPP_ESCAPE_BEFORE_EXIT` | bool | n | PPP→CMD 时发 `+++`（Quectel 友好；SIMCOM 可能出错） |
| `CONFIG_ESP_MODEM_ADD_DEBUG_LOGS` | bool | n | 打印 UART 收发原始数据（调试用） |
| `CONFIG_ESP_MODEM_ENABLE_DEVELOPMENT_MODE` | bool | n | 开发模式：直接展开 AT 命令宏（仅库开发者） |

## mdns（`components/mdns/Kconfig`，节选）

| Kconfig | 类型 | 默认 | 说明 |
|---|---|---|---|
| `CONFIG_MDNS_MAX_INTERFACES` | int | 3（范围 1-9） | mDNS 服务的最大接口数，降低可省 RAM |
| `CONFIG_MDNS_MAX_SERVICES` | int | 10 | 最大服务数 |
| `CONFIG_MDNS_TASK_PRIORITY` | int | 1（范围 1-255） | mDNS 任务优先级（勿高于系统任务） |
| `CONFIG_MDNS_TASK_STACK_SIZE` | int | 4096 | mDNS 任务栈大小 |
| `CONFIG_MDNS_ACTION_QUEUE_LEN` | int | 16（范围 8-64） | 动作队列长度 |
| `CONFIG_MDNS_MULTIPLE_INSTANCE` | bool | — | 允许同类型多实例（`mdns_service_add` 多次） |
| `CONFIG_MDNS_PUBLISH_DELEGATE_HOST` | bool | — | 启用委托主机发布 |
| `CONFIG_MDNS_RESOLVE_TEST_SERVICES` | bool | — | 解析测试服务（example 用） |
| `CONFIG_MDNS_RESPOND_REVERSE_QUERIES` | bool | — | 响应 IPv6 反向查询（影响 `MDNS_NAME_MAX_LEN`） |

## esp_websocket_client

主要在 example 的 `Kconfig.projbuild` 提供 `CONFIG_WEBSOCKET_URI`、`CONFIG_WEBSOCKET_URI_FROM_STDIN`，以及 TLS 模式开关（如 `CONFIG_WS_OVER_TLS_SERVER_AUTH` / `CONFIG_WS_OVER_TLS_MUTUAL_AUTH` / `CONFIG_WS_OVER_TLS_SKIP_COMMON_NAME_CHECK`）。

组件级 TLS 行为依赖 ESP-IDF 的 `esp_tls` / `MBEDTLS_CERTIFICATE_BUNDLE` 配置；`client_ds_data` 需要 `CONFIG_ESP_TLS_USE_DS_PERIPHERAL`。

## eppp_link（`components/eppp_link/Kconfig`，传输层选择）

| Kconfig | 说明 |
|---|---|
| `CONFIG_EPPP_LINK_DEVICE_UART` | 使用 UART 传输（`EPPP_DEFAULT_TRANSPORT_CONFIG` → `EPPP_DEFAULT_UART_CONFIG()`） |
| `CONFIG_EPPP_LINK_DEVICE_SPI` | 使用 SPI 传输 |
| `CONFIG_EPPP_LINK_DEVICE_SDIO` | 使用 SDIO 传输 |
| `CONFIG_EPPP_LINK_DEVICE_ETH` | 使用 Ethernet 传输 |

> 选定传输层后，`EPPP_DEFAULT_TRANSPORT_CONFIG()` / `EPPP_TRANSPORT_INIT()` / `EPPP_TRANSPORT_DEINIT()` 宏会自动指向对应实现。

## 组件版本（`idf_component.yml`，截至本 Skill 撰写）

| 组件 | version |
|---|---|
| esp_modem | `2.0.2` |
| mdns | `1.11.2` |
| esp_websocket_client | `1.7.0` |

> 在工�� `idf_component.yml` 中声明依赖时使用上述版本号或范围。

## 相关 ESP-IDF Kconfig（间接）

| Kconfig | 何时需要 |
|---|---|
| `LWIP_PPP_SUPPORT` | esp_modem 自动 select；用 PPPoS 必需 |
| `MBEDTLS_CERTIFICATE_BUNDLE` | esp_websocket_client 用 `crt_bundle_attach` / esp_dns DoT/DoH 用证书包 |
| `CONFIG_ESP_TLS_USE_DS_PERIPHERAL` | esp_websocket_client 双向认证用 DS 外设（`client_ds_data`） |
