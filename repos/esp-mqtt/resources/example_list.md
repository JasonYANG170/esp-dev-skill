# ESP-MQTT 示例清单

> 全部路径来自仓库 `examples/`。每个示例含 `README.md` 与 `main/app_main.c`。

| 示例 | 路径 | 传输 | 说明 |
|---|---|---|---|
| TCP | `examples/tcp/` | mqtt:// | 最小 MQTT over TCP 连接，订阅/取消订阅/发布演示，broker URI 由 `CONFIG_BROKER_URL` 配置 |
| SSL（服务端 CA） | `examples/ssl/` | mqtts:// | 用服务端 CA 证书单向校验的 TLS 连接（默认 8883） |
| 双向认证 | `examples/ssl_mutual_auth/` | mqtts:// | 客户端证书 + 私钥 + 服务端 CA 的 mTLS（test.mosquitto.org:8884） |
| PSK 认证 | `examples/ssl_psk/` | mqtts:// | 使用预共享密钥（PSK）认证，无需证书 |
| 数字签名外设 | `examples/ssl_ds/` | mqtts:// | 用 Digital Signature 外设 + secure cert 进行 TLS 认证（test.mosquitto.org:8884）。支持 ESP32-S2/S3/C3/C5/C6/H2/P4。依赖 `espressif/esp_secure_cert_mgr: "^2.0.2"`；顶层 `CMakeLists.txt` 注册 `esp_secure_cert` 分区烧录；provisioning 用 `esp-secure-cert-tool`（`configure_esp_secure_cert.py --configure_ds`）。参见 `examples/ssl_ds/README.md`、`examples/ssl_ds/CMakeLists.txt`、`examples/ssl_ds/main/idf_component.yml` |
| WebSocket | `examples/ws/` | ws:// | MQTT over WebSocket（默认 80，`CONFIG_BROKER_URI`） |
| WebSocket Secure | `examples/wss/` | wss:// | MQTT over WebSocket Secure（默认 443） |
| MQTT 5 | `examples/mqtt5/` | mqtt:// 或 mqtts:// | MQTT v5.0：连接/发布/订阅/disconnect 属性、用户属性、共享订阅 |
| 自定义 outbox | `examples/custom_outbox/` | mqtt:// | 启用 `CONFIG_MQTT_CUSTOM_OUTBOX`，用 C++ 重写 outbox（polymorphic memory resource） |

## 各示例 CMake 依赖（main/CMakeLists.txt，PRIV_REQUIRES）

| 示例 | 额外依赖 |
|---|---|
| tcp | `mqtt nvs_flash esp_netif` |
| ssl | `mqtt esp_partition nvs_flash esp_netif app_update` |
| ssl_mutual_auth | `mqtt esp_wifi nvs_flash`（含 lwip 等） |
| ssl_psk | `mqtt nvs_flash esp_netif` |
| ssl_ds | `mqtt esp_partition nvs_flash esp_netif app_update`（用 `esp_secure_cert_*`） |
| ws | `mqtt esp_wifi nvs_flash` |
| wss | `mqtt nvs_flash esp_netif` |
| mqtt5 | `mqtt nvs_flash esp_netif` |
| custom_outbox | `mqtt nvs_flash esp_netif`（自定义 outbox 源码经顶层 CMake 注入 mqtt 组件） |

## 支持目标芯片

所有示例的 `README.md` 顶部均标注支持目标（截至当前仓库）：
`ESP32 | ESP32-C2 | ESP32-C3 | ESP32-C5 | ESP32-C6 | ESP32-C61 | ESP32-H2 | ESP32-P4 | ESP32-S2 | ESP32-S3`

## 运行方式

```bash
idf.py set-target esp32
idf.py menuconfig      # 配 Example Connection Configuration（Wi-Fi）与 Broker URL
idf.py build
idf.py -p PORT flash monitor
```

> 公共 broker（如 mqtt.eclipseprojects.io / test.mosquitto.org）由社区维护，可能不稳定；`CONFIG_BROKER_URL_FROM_STDIN` 时启动时从 stdin 读 URI（用于测试）。
