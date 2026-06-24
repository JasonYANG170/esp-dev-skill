# esp-protocols 真实 Example 索引

> 以下路径均为仓库内真实存在的 example 目录（相对 `components/`）。每个 example 可作为改造起点。路径可加前缀 `D:/esp-skill/espressif-repos/esp-protocols/components/`。

## esp_modem

| 路径 | 说明 |
|---|---|
| `esp_modem/examples/pppos_client/` | 最常用的 PPPoS 客户端（C），支持 UART/USB、多种模组、SIM PIN、SMS、自定义模组 |
| `esp_modem/examples/modem_console/` | C++ 交互控制台，可逐条发 AT / 内置命令（ping、httpget）调试模组 |
| `esp_modem/examples/simple_cmux_client/` | CMUX 模式客户端：数据通道跑 PPP 的同时发 AT |
| `esp_modem/examples/ap_to_pppos/` | WiFi AP ↔ PPPoS NAPT 网关（把蜂窝流量转发给 WiFi 客户端） |
| `esp_modem/examples/modem_tcp_client/` | 在模组链路上跑 TCP 客户端（含 mbedTLS transport） |
| `esp_modem/examples/modem_psm/` | 模组 PSM（Power Saving Mode）低功耗示例 |
| `esp_modem/examples/linux_modem/` | Linux 主机上用 esp_modem（VFS terminal） |

## mdns

| 路径 | 说明 |
|---|---|
| `mdns/examples/query_advertise/` | mDNS 服务发布 + 查询一体示例（hostname/service/TXT/subtype/delegate/query） |

## esp_websocket_client

| 路径 | 说明 |
|---|---|
| `esp_websocket_client/examples/target/` | ESP 目标板 WebSocket 客户端：ws/wss、双向认证、分片帧、含测试用 websocket_server.py |
| `esp_websocket_client/examples/linux/` | Linux 主机上的 WebSocket 客户端示例 |

## eppp_link

| 路径 | 说明 |
|---|---|
| `eppp_link/examples/host/` | 主控侧（PPP client），从通信协处理器获取联网能力 |
| `eppp_link/examples/slave/` | 通信协处理器侧（PPP server + WiFi/NAT），为 host 提供联网 |

## esp_dns

| 路径 | 说明 |
|---|---|
| `esp_dns/examples/esp_dns_basic/` | esp_dns 基础示例（DoT/DoH/TCP/UDP 解析） |

## esp_mqtt_cxx

| 路径 | 说明 |
|---|---|
| `esp_mqtt_cxx/examples/tcp/` | `idf::mqtt::Client` C++ 客户端（明文 TCP），订阅 `$SYS/broker/...`，`Insecure` 安全策略 |
| `esp_mqtt_cxx/examples/ssl/` | `idf::mqtt::Client` C++ 客户端（TLS），用嵌入的 `mqtt_eclipseprojects_io.pem` 做 `CryptographicInformation{PEM{...}}` |

## mosquitto（板载 MQTT Broker）

| 路径 | 说明 |
|---|---|
| `mosquitto/examples/broker/` | ESP32 板载 Mosquitto broker：TCP/TLS（`esp_tls_cfg_server_t`）、basic auth（`handle_connect_cb`）、可选本地客户端 loopback 自测（`CONFIG_EXAMPLE_BROKER_RUN_LOCAL_MQTT_CLIENT`） |
| `mosquitto/examples/serverless_mqtt/` | 双 ESP 板载 broker 经 ICE/WebRTC 跨 NAT 同步：`handle_message_cb` 转发远端消息、`message_wrap_t` 自定义封装 |

## 顶层 examples/（跨组件演示）

| 路径 | 说明 |
|---|---|
| `examples/esp_netif/multiple_netifs/` | 多网络接口管理 |
| `examples/esp_netif/slip_custom_netif/` | SLIP 自定义 netif 客户端 |
| `examples/mqtt/` | Linux 上的 MQTT demo（含 esp32h2 / linux 默认配置） |

> 其余组件（asio / libwebsockets / mbedtls_cxx / sock_utils / console_* / net_connect）请参阅各自 `README.md` 与 `examples/`（如有）。本 Skill 聚焦 esp_modem、mdns、esp_websocket_client、eppp_link、esp_dns、esp_mqtt_cxx、mosquitto 等核心协议组件。
