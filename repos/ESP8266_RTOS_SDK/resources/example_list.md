# ESP8266_RTOS_SDK 示例索引

> 仓库真实路径（相对 `examples/`）。每个示例都是独立工程，可整目录拷出再改；`IDF_PATH` 是与 SDK 的唯一连接。绝大多数示例同时含 `Makefile` 与 `CMakeLists.txt`。

## get-started（入门）

| 路径 | 说明 |
|---|---|
| `examples/get-started/hello_world` | 最小工程：`app_main` 打印芯片信��、`spi_flash_get_chip_size`、倒计时 `esp_restart`；新建工程的首选模板 |

## wifi/getting_started（WiFi 入门）

| 路径 | 说明 |
|---|---|
| `examples/wifi/getting_started/station` | Station 连 AP：`tcpip_adapter_init` + `esp_event_loop_create_default` + 事件回调 + 重连 + `xEventGroupWaitBits` 等 IP；`Kconfig.projbuild` 提供 SSID/密码/重试 |
| `examples/wifi/getting_started/softAP` | SoftAP 开放/WPA2 热点：处理 `WIFI_EVENT_AP_STACONNECTED` / `_STADISCONNECTED`，`Kconfig.projbuild` 提供 SSID/密码/最大接入数 |

## wifi（高级 WiFi）

| 路径 | 说明 |
|---|---|
| `examples/wifi/espnow` | ESPNOW 广播/单播：WiFi 初始化 → `esp_now_init` + 收发回调 + peer 管理 + PMK/LMK 加密 + CRC 校验 |
| `examples/wifi/smart_config` | 一键配网：Esptouch / AirKiss / Esptouch v2，`SC_STATUS_*` 状态机拿 SSID/密码 |
| `examples/wifi/sniffer` | Promiscuous 模式抓包：`esp_wifi_set_promiscuous_rx_cb`，解析 `wifi_promiscuous_pkt_t` |
| `examples/wifi/iperf` | iperf 带宽测试（STA / AP 两端） |
| `examples/wifi/power_save` | 省电模式：`esp_wifi_set_ps` + light sleep + GPIO 唤醒 |
| `examples/wifi/roaming` | 漫游：RSSI 监测与切换 AP |
| `examples/wifi/wpa2_enterprise` | WPA2 企业级（EAP）认证连接 |
| `examples/wifi/wps` | WPS 配网（PBC / PIN） |

## protocols（网络协议）

| 路径 | 说明 |
|---|---|
| `examples/protocols/http_request` | 直接用 POSIX socket（`getaddrinfo`/`socket`/`connect`/`read`/`write`）发 HTTP GET，含超时 `setsockopt` |
| `examples/protocols/esp_http_client` | `esp_http_client` 组件：GET/POST、事件回调、流式读写 |
| `examples/protocols/https_request` | HTTPS GET（mbedTLS） |
| `examples/protocols/https_mbedtls` | mbedTLS 客户端基础 |
| `examples/protocols/sockets/tcp_client` | TCP 客户端（`example_connect` 连网后 socket 通信） |
| `examples/protocols/sockets/tcp_server` | TCP 服务端 |
| `examples/protocols/sockets/udp_client` | UDP 客户端 |
| `examples/protocols/sockets/udp_server` | UDP 服务端 |
| `examples/protocols/sockets/udp_multicast` | UDP 组播 |
| `examples/protocols/mqtt/tcp` | MQTT over TCP |
| `examples/protocols/mqtt/ssl` | MQTT over TLS |
| `examples/protocols/mqtt/ssl_mutual_auth` | MQTT 双向认证 |
| `examples/protocols/mqtt/ssl_psk` | MQTT PSK |
| `examples/protocols/mqtt/ws` | MQTT over WebSocket |
| `examples/protocols/mqtt/wss` | MQTT over WebSocket Secure |
| `examples/protocols/http_server/simple` | 极简 HTTP 服务端 |
| `examples/protocols/http_server/persistent_sockets` | 持久连接 HTTP 服务端 |
| `examples/protocols/http_server/advanced_tests` | HTTP 服务端高级测试 |
| `examples/protocols/sntp` | SNTP 对时，`time()` 获取 UTC |
| `examples/protocols/mdns` | mDNS 广播/发现 |
| `examples/protocols/coap_client` | CoAP 客户端 |
| `examples/protocols/coap_server` | CoAP 服务端 |
| `examples/protocols/icmp_echo` | ICMP ping |
| `examples/protocols/openssl_client` / `openssl_server` / `openssl_demo` | OpenSSL（esp-wolfssl）示例 |

## peripherals（外设）

| 路径 | 说明 |
|---|---|
| `examples/peripherals/gpio` | GPIO 输入/输出 + 双边沿中断：`gpio_config` + `gpio_install_isr_service` + `gpio_isr_handler_add` + ISR→队列→task |
| `examples/peripherals/uart_echo` | UART 回显：`uart_param_config` + `uart_driver_install` + `uart_read_bytes`/`uart_write_bytes` |
| `examples/peripherals/uart_events` | UART 事件队列模型：独立 task 处理 `UART_DATA`/`UART_FIFO_OVF`/`UART_BUFFER_FULL` |
| `examples/peripherals/uart_select` | UART + `select()` 多路复用 |
| `examples/peripherals/i2c` | I2C 主模式读写 MPU6050：命令链 `i2c_master_start/write_byte/read_byte/stop` |
| `examples/peripherals/spi_oled` | HSPI 驱动 OLED 屏 |
| `examples/peripherals/pwm` | 软件 PWM：4 通道（GPIO12~15）、占空比、相位、`pwm_start`/`pwm_stop` |
| `examples/peripherals/ledc` | LEDC（硬件 PWM/渐变）示例 |
| `examples/peripherals/hw_timer` | FRC1 硬件定时器：`hw_timer_init` + `hw_timer_alarm_us`（周期/单次） |
| `examples/peripherals/adc` | ADC 采样：`adc_init(ADC_READ_TOUT_MODE)` + `adc_read` / `adc_read_fast` |
| `examples/peripherals/i2s` | I2S 音频接口 |
| `examples/peripherals/ir_tx` | IR 红外发送 |
| `examples/peripherals/ir_rx` | IR 红外接收 |

## storage（存储）

| 路径 | 说明 |
|---|---|
| `examples/storage/spiffs` | SPIFFS 文件系统：自定义分区表含 `data, spiffs` → `esp_vfs_spiffs_register` → POSIX `fopen`/`fprintf`/`fgets`/`rename` → `esp_spiffs_info` 查容量 |

## system（系统）

| 路径 | 说明 |
|---|---|
| `examples/system/ota/simple_ota_example` | HTTPS 简易 OTA：`esp_https_ota(&config)` 成功后 `esp_restart`（需分区表 "Two OTA app" + `CONFIG_FIRMWARE_UPGRADE_URL`） |
| `examples/system/ota/native_ota/*` | 原生 OTA（仅 CMake）：手动 `esp_ota_begin/write/end/set_boot_partition`，含 1MB_flash / 2+MB_flash 多个变体（new_to_new_no_old / new_to_new_with_old 等） |
| `examples/system/console` | 命令行 console（linenoise + 命令注册） |
| `examples/system/factory-test` | 出厂测试固件 |

## provisioning（配网）

| 路径 | 说明 |
|---|---|
| `examples/provisioning/legacy/softap_prov` | SoftAP 配网（旧） |
| `examples/provisioning/legacy/custom_config` | 自定义配网协议（旧） |

## common_components（协议示例共用组件）

| 路径 | 说明 |
|---|---|
| `examples/common_components/protocol_examples_common` | 协议示例共用：`example_connect()` 统一连网（被 http_request/simple_ota 等调用）|

> 引用示例路径时请用仓库根相对路径（如 `examples/wifi/getting_started/station`）。
