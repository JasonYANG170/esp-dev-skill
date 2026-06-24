# Changelog

本文件记录 ESP8266_RTOS_SDK-skill 的版本变更。格式参考 [Keep a Changelog](https://keepachangelog.com/)，版本号遵循 [Semantic Versioning](https://semver.org/)。

## [1.1.0] - 2026-06-18

补充确认的高价值网络/功耗场景缺口。

### Added

- **recipes/mqtt.md** — esp-mqtt 客户端，覆盖 TCP/SSL/双向认证/PSK/WS/WSS 六种传输与事件回调（订阅/发布/数据）。重新依据真实 esp-mqtt 头文件（`esp-mqtt/include/mqtt_client.h`，736 行，含完整 `esp_mqtt_client_config_t` 嵌套结构、`esp_mqtt_event_id_t`、`esp_mqtt_error_codes_t`、全部 API 签名）与 6 个 ESP8266 示例（`examples/protocols/mqtt/{tcp,ssl,ssl_mutual_auth,ssl_psk,ws,wss}`）、`components/mqtt/Kconfig` 校准：以头文件的嵌套字段（`broker.address.uri`、`broker.verification.certificate`、`credentials.authentication.certificate/key`、`broker.verification.psk_hint_key`）为权威写法，并标注示例中旧扁平简写（`.uri`/`.cert_pem`/`.client_cert_pem`/`.client_key_pem`/`.psk_hint_key`）的对应关系。
- **recipes/deep_sleep_power_save.md** — WiFi modem sleep（`esp_wifi_set_ps` NONE/MIN/MAX）、deep sleep（`esp_deep_sleep` 定时/RST 唤醒）、light sleep（GPIO 唤醒）与 `esp_pm_configure` 动态调频。来源：`examples/wifi/power_save`、`components/esp8266/include/esp_sleep.h`、`docs/en/api-reference/system/sleep_modes.rst`。
- **recipes/sntp_time.md** — LwIP SNTP 对时（`sntp_setoperatingmode`/`sntp_setservername`/`sntp_init`）、`time()`/`localtime_r()` 读取、POSIX `setenv("TZ")`+`tzset()` 时区。来源：`examples/protocols/sntp`（`main/sntp_example_main.c` + README）。
- **recipes/sockets.md** — BSD socket TCP 服务端（bind/listen/accept/echo）、TCP 客户端、UDP IPv4 组播（`IP_ADD_MEMBERSHIP`/`IP_MULTICAST_TTL` + `select`）。来源：`examples/protocols/sockets/{tcp_server,tcp_client,udp_client,udp_server,udp_multicast}`。
- **recipes/http_server.md** — `esp_http_server` URI handler（GET/POST/PUT、读请求头/查询串、`httpd_resp_send`/`send_chunk` 分块响应、运行期动态注册注销、网络事件启停）。来源：`examples/protocols/http_server/{simple,persistent_sockets}`、`components/esp_http_server/include/esp_http_server.h`。
- **SKILL.md**：新增 Core Principle #13（睡眠前先停 WiFi）；新增"功耗 / 睡眠"场景表与"网络 / 协议"表新增 5 个 recipe 行；版本 1.0.0 → 1.1.0。
- **resources/api_reference.md**：新增 HTTP Server、MQTT、SNTP 三个模块的真实函数签名/结构体/宏/错误码。
- **resources/example_list.md**：补全各 recipe 引用的示例路径（已有条目核对无误）。

### Grounding

所有新增函数名、结构体、枚举、宏、Kconfig 符号、文件路径、代码片段均取自仓库 `D:/esp-skill/espressif-repos/ESP8266_RTOS_SDK` 的真实头文件（`components/esp8266/include/esp_sleep.h`、`components/esp_http_server/include/esp_http_server.h`、`components/mqtt/Kconfig`）、示例源码（`examples/protocols/mqtt/*`、`examples/protocols/sntp`、`examples/protocols/sockets/*`、`examples/protocols/http_server/*`、`examples/wifi/power_save`）与文档（`examples/protocols/*/README.md`、`docs/en/api-reference/system/sleep_modes.rst`），无臆造内容。esp-mqtt 配置结构体与 API 签名依据真实头文件 `D:/esp-skill/espressif-repos/esp-mqtt/include/mqtt_client.h`（736 行，完整 `esp_mqtt_client_config_t` 嵌套定义 + 全部 `esp_mqtt_client_*` 签名 + `esp_mqtt_event_id_t` / `esp_mqtt_error_codes_t`）校准，已替换 1.0.0 中"子模块未检出、按示例推导"的临时说明。

## [1.0.0] - 2026-06-18

首个正式发布。

### Added

- **SKILL.md**：技能主入口，含 12 条核心原则、When to Use、13 个 recipe 场景速查、GPIO 速查、WiFi 模式/认证表、内置分区表（Single factory / Two OTA）、14 条 Critical Pitfalls（错误/正确对照代码）、执行工作流与失败策略。
- **AGENTS.md**：补充约定——项目上下文（C / ESP8266EX / xtensa-lx106-elf gcc8.4.0 / Make+CMake）、文件命名、include 模式、标准工程结构、`app_main`/初始化模板、FreeRTOS 任务与日志、错误处理、构建工作流（make / idf.py）、codegen checklist、Do Not Modify。
- **recipes/**：13 个场景 recipe，均基于仓库真实示例与头文件：
  - `hello_world_project.md`、`build_and_flash.md`（入门/构建）
  - `wifi_station.md`、`wifi_softap.md`、`smartconfig.md`、`espnow.md`（WiFi）
  - `http_request.md`、`http_ota.md`（网络/OTA）
  - `gpio.md`、`uart.md`、`pwm.md`、`i2c.md`（外设）
  - `spiffs.md`（存储）
- **resources/api_reference.md**：按模块（系统/FreeRTOS/NVS/事件/WiFi/TCPIP/ESPNOW/SmartConfig/GPIO/UART/I2C/SPI/PWM/ADC/hw_timer/LEDC/Sleep/SPI Flash/SPIFFS/OTA/HTTP Client/Socket）汇总仓库真实函数签名、结构体、枚举、宏。
- **resources/config_reference.md**：Kconfig/menuconfig 速查——工具链、Serial flasher、Partition Table、PHY、WiFi（`CONFIG_ESP8266_WIFI_*`）、Log、Example Configuration，附 `sdkconfig.defaults` 示例。
- **resources/pitfalls.md**：按 13 个模块（框架/版本、app_main/任务、NVS/WiFi、GPIO、UART、PWM、I2C、ADC、OTA/分区表、SPIFFS、Sleep、日志、构建）归类的扩展陷阱。
- **resources/example_list.md**：仓库 `examples/` 下全部真实示例路径（get-started、wifi、protocols、peripherals、storage、system、provisioning、common_components）与一句话描述。
- **README.md**：技能介绍、特性、适用范围、安装方式、目录结构、使用建议、License。
- **CHANGELOG.md**：本文件。

### Grounding

所有函数名、结构体、枚举、宏、Kconfig 符号、文件路径、代码片段均取自仓库 `D:/esp-skill/espressif-repos/ESP8266_RTOS_SDK` 的真实头文件（`components/esp8266/include/`、`components/*/include/`）、文档（`docs/en/`）与示例（`examples/`），无臆造内容。
