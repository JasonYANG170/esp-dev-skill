# Recipe 总索引（全 41 仓库）

> 跨仓库按主题查找 recipe 的总表。每个 recipe 链接到 `repos/<repo>/recipes/<file>`。


## 核心框架


### `esp-idf` (21 recipes)

| recipe | 摘要 |
|---|---|
| **BLE 主机（Bluedroid GATT Client）**<br>`repos/esp-idf/recipes/ble_central.md` | 用 Bluedroid 实现 BLE GATT Client：扫描（`esp_ble_gap_set_scan_params` + `esp_ble_gap_start_scanning`）、连接（`esp_ble_gattc_open` / `esp_ble_gattc_enh_open`）、服务/特征值发现（`esp_ble_gattc_search_service` + `esp_ble_gattc_get_char_by_uuid`）、读写（`esp_ble_gattc_read_char` / `esp_ble_gattc_write_char`）、订阅 notify。适配自 `examples/bluetooth/bluedroid/ble/gatt_client`。 |
| **ESP-BLE-MESH 节点（Generic OnOff Server）**<br>`repos/esp-idf/recipes/ble_mesh.md` | 用 ESP-BLE-MESH 协议栈实现一个 Mesh 节点（Generic OnOff Server 模型）：配置 Composition Data（含 Config Server + Generic OnOff Server 模型）、`esp_ble_mesh_init` 启动、开启 PB-ADV/PB-GATT 配网承载（Provisioning）、`esp_ble_mesh_node_prov_enable` 进入待配网状态、处理 Generic Server 的 GET/SET 消息、`esp_ble_mesh_model_publish` 发布状态。适配自 `examples/bluetooth/esp_ble_mesh/onoff_models/onoff_server`。 |
| **BLE 外设（Bluedroid GATT Server）**<br>`repos/esp-idf/recipes/ble_peripheral.md` | 用 Bluedroid 协议栈实现 BLE GATT Server：控制器/Host 初始化固定顺序、设置广播（`esp_ble_gap_config_adv_data` / raw）、用属性表注册自定义服务与特征值（`esp_ble_gatts_create_attr_tab`）、处理 `ESP_GATTS_WRITE_EVT`、发送 notification/indication（`esp_ble_gatts_send_indicate`）。适配自 `examples/bluetooth/bluedroid/ble/gatt_server_service_table`。 |
| **经典蓝牙 A2DP 音频接收（Sink）+ AVRCP**<br>`repos/esp-idf/recipes/classic_bt_a2dp.md` | 用 Bluedroid 实现 A2DP Sink（Advanced Audio Distribution Profile，接收手机推送的音频流）与 AVRCP CT（音量/播放控制）：`esp_a2d_sink_init`、`esp_a2d_register_callback`、`esp_a2d_sink_register_audio_data_callback` 收 PCM 数据、`esp_a2d_sink_connect` 主动连接、`esp_avrc_ct_init` 控制器。**仅 esp32 支持，需 `CONFIG_BT_A2DP_ENABLE`。** 适配自 `examples/bluetooth/bluedroid/classic_bt/a2dp_sink_stream`。 |
| **经典蓝牙 SPP（串口透传）**<br>`repos/esp-idf/recipes/classic_bt_spp.md` | 用 Bluedroid 实现经典蓝牙 SPP（Serial Port Profile，基于 RFCOMM）：`esp_spp_enhanced_init` + `esp_spp_cfg_t` 选 CB/VFS 模式、`esp_spp_start_srv` 开服务、`esp_spp_write` 发数据、事件回调处理。**经典蓝牙需 `CONFIG_BT_CLASSIC_ENABLED=y` 且仅 esp32（双模）支持**，并需释放 BLE 控制器内存。适配自 `examples/bluetooth/bluedroid/classic_bt/bt_spp_acceptor`。 |
| **深度睡眠与唤醒源**<br>`repos/esp-idf/recipes/deep_sleep.md` | 进入深度睡眠并配置定时器 / GPIO（ext0/ext1）唤醒源，启动后用 `esp_sleep_get_wakeup_causes` 判断唤醒来源（适配自 system/deep_sleep）。 |
| **以太网（内部 MAC + PHY）**<br>`repos/esp-idf/recipes/ethernet.md` | 用内部以太网 MAC（EMAC）+ 外部 PHY 跑有线网络：`esp_eth_mac_new_esp32` 建 MAC、`esp_eth_phy_new_generic` 建 PHY（通用 802.3 驱动）、`esp_eth_driver_install` 装驱动、`esp_eth_new_netif_glue` 挂到 netif、`esp_eth_start` 启动、事件处理（`ETHERNET_EVENT_*` / `IP_EVENT_ETH_GOT_IP`）。适配自 `examples/ethernet/basic`。 |
| **FreeRTOS 任务与队列**<br>`repos/esp-idf/recipes/freertos_task.md` | 创建 FreeRTOS 任务、用队列在任务/ISR 间传递数据，含任务通知与延时用法。 |
| **GPIO 输入输出与中断**<br>`repos/esp-idf/recipes/gpio_control.md` | 配置 GPIO 为输入/输出，读取/设置电平，安装 ISR 服务并处理下降沿等中断。 |
| **GPTimer 通用定时器**<br>`repos/esp-idf/recipes/gptimer.md` | 用新 HAL 的 GPTimer（`gptimer_new_timer`）创建高分辨率定时器、配置告警回调，实现周期任务（适配自 gptimer 示例）。 |
| **I2C 主机（新总线 API）**<br>`repos/esp-idf/recipes/i2c_master.md` | 使用 ESP-IDF v5.x 新总线式句柄 API（`i2c_new_master_bus` + `i2c_master_bus_add_device`）创建 I2C 主机并读写传感器寄存器。 |
| **LEDC PWM 输出与调光**<br>`repos/esp-idf/recipes/ledc_pwm.md` | 用 LEDC 外设输出 PWM、改变频率与占空比，含基础初始化与运行时调光（适配自 ledc_basic）。 |
| **新建 ESP-IDF 工程**<br>`repos/esp-idf/recipes/new_project.md` | 从零创建一个可编译/烧录/监控的 ESP-IDF 工程，含目录结构、顶层与组件 CMakeLists、目标设置与构建流程。 |
| **NVS 键值存储**<br>`repos/esp-idf/recipes/nvs_storage.md` | 用 NVS（Non-Volatile Storage）持久化整数/字符串/blob，含 `nvs_open`/`nvs_set_*`/`nvs_get_*`/`nvs_commit`（适配自 storage/nvs/nvs_rw_value）。 |
| **OTA 固件升级**<br>`repos/esp-idf/recipes/ota_update.md` | 通过 HTTPS 下载新固件并切换分区启动，覆盖高层 `esp_https_ota`（推荐）与底层 `esp_ota_begin/write/end` 两种用法（适配自 simple_ota_example���。 |
| **自定义分区表与分区查询**<br>`repos/esp-idf/recipes/partition_table.md` | 用 CSV 自定义 Flash 分区表，运行时用 `esp_partition_find_first` / `esp_partition_find` 查询分区（适配自 storage/partition_api/partition_find）。 |
| **Wi-Fi SoftAP 热点**<br>`repos/esp-idf/recipes/softap.md` | 以 SoftAP 模式创建 Wi-Fi 热点，配置 SSID/密码/认证/最大连接数，处理 STA 接入事件（适配自 getting_started/softAP）。 |
| **SPI 主机**<br>`repos/esp-idf/recipes/spi_master.md` | 初始化 SPI 主机总线（`spi_bus_initialize`）、添加设备（`spi_bus_add_device`）、阻塞/排队传输。 |
| **UART 通信**<br>`repos/esp-idf/recipes/uart_comm.md` | 配置 UART 端口、安装驱动、阻塞/事件驱动收发，含引脚设置。 |
| **Wi-Fi Mesh（ESP-WIFI-MESH）**<br>`repos/esp-idf/recipes/wifi_mesh.md` | 用 ESP-WIFI-MESH 协议栈组建自组织 Wi-Fi 网状网络：`esp_mesh_init`、`esp_mesh_set_config`（mesh ID / 路由器 / SoftAP 凭据）、`esp_mesh_start`、收发（`esp_mesh_send` / `esp_mesh_recv`）、路由表查询、事件处理（`MESH_EVENT_*`）。适配自 `examples/mesh/internal_communication`。 |
| **Wi-Fi STA 连接**<br>`repos/esp-idf/recipes/wifi_sta.md` | 以 Station 模式连接 AP，含事件循环注册、`esp_wifi_start`、连接成功判定（`IP_EVENT_STA_GOT_IP`）与重连（适配自 getting_started/station）。 |

### `arduino-esp32` (17 recipes)

| recipe | 摘要 |
|---|---|
| **ADC 模拟量读取**<br>`repos/arduino-esp32/recipes/adc_reading.md` | 使用 `analogRead` / `analogReadMilliVolts` 单次转换，以及 `analogContinuous` 连续模式对多通道后台采样并回调。含衰减（attenuation）与电压范围选择。 |
| **BLE 蓝牙低功耗（扫描 / 服务端 / 客户端 / UART / Notify / Beacon）**<br>`repos/arduino-esp32/recipes/ble.md` | 使用 `BLEDevice` 库实现 BLE 扫描、GATT Server/Client、UART 透传、Notify 通知、iBeacon/Eddystone 广播。⚠️ 3.x 中 ESP32 仍用 Bluedroid，其余 SoC（C3/C5/C6/H2/S3）默认走 NimBLE；4.0 起 Bluedroid 将被移除。 |
| **深睡眠与唤醒 (Deep Sleep)**<br>`repos/arduino-esp32/recipes/deep_sleep.md` | 使用 `esp_deep_sleep_*` 与定时器 / 外部(GPIO) / 触摸唤醒源让 ESP32 进入低功耗深睡眠，并在唤醒后重启运行。 |
| **ESP-NOW 低延迟通信**<br>`repos/arduino-esp32/recipes/espnow.md` | 使用 `ESP32_NOW` 库（`ESP_NOW` 类 + `ESP_NOW_Peer` 子类）在 ESP32 设备间做广播 / 点对点通信，无需 AP。⚠️ 必须先 `WiFi.mode()` 再 `ESP_NOW.begin()`。 |
| **FreeRTOS 多任务**<br>`repos/arduino-esp32/recipes/freertos_tasks.md` | 在 Arduino 环境中用原生 FreeRTOS API（`xTaskCreate`/`xTaskCreatePinnedToCore`、队列、信号量、互斥）实现并发任务，避免阻塞 `loop()`。 |
| **GPIO 输出输入与中断**<br>`repos/arduino-esp32/recipes/gpio_blink_interrupt.md` | 使用 Arduino 标准 `pinMode`/`digitalWrite`/`digitalRead` 控制引脚，并用 `attachInterrupt` 响应外部中断（按键、脉冲等）。 |
| **硬件定时器 (Hardware Timer)**<br>`repos/arduino-esp32/recipes/hardware_timer.md` | 使用 `timerBegin` / `timerAlarm` 配置 64 位硬件定时器并产生周期中断。ESP32/S2/S3 各 4 个，C3/C6/H2 各 2 个。 |
| **I2C (Wire) 主机与从机**<br>`repos/arduino-esp32/recipes/i2c_master_slave.md` | 使用 `Wire` 库进行 I2C 主机读写和从机应答，含 `setPins`、`beginTransmission`、`requestFrom`、`onReceive`/`onRequest` 回调。 |
| **LEDC (PWM) 输出与调光**<br>`repos/arduino-esp32/recipes/ledc_pwm.md` | 使用 v3.x LEDC Peripheral Manager API（`ledcAttach`/`ledcWrite`）输出 PWM、呼吸灯、播放音符（`ledcWriteNote`）。⚠️ 2.x 的 `ledcSetup`/`ledcAttachPin` 已删除。 |
| **Matter 智能家居（Wi-Fi / Thread，commissioning，on-off/light/color/sensor/按钮）**<br>`repos/arduino-esp32/recipes/matter.md` | 使用 `Matter` 库（基于 ESP Matter SDK）创建 Apple HomeKit/Google Home/Amazon Alexa 兼容的 Matter 设备，注册 `MatterEndPoint`（灯/传感器/插座/开关/窗帘），含 QR 码配网、事件回调、解绑出厂。⚠️ 必须选 `Huge APP (3 MB No OTA / 1 MB SPIFFS)` 分区方案并擦除全 flash 后上传。 |
| **OTA 固件升级（ArduinoOTA / Update / HTTPUpdate / WebUpdater / Signed OTA）**<br>`repos/arduino-esp32/recipes/ota_update.md` | 使用 `ArduinoOTA`（IDE/网络推送）、`Update`（底层流式写入）、`httpUpdate`（HTTP(S) 拉取）、`OTAWebUpdater`（浏览器上传）实现固件升级，含分区表要求与签名验证流程。 |
| **Preferences (NVS 持久化存储)**<br>`repos/arduino-esp32/recipes/preferences_nvs.md` | 使用 ESP32 专属 `Preferences` 库（基于片上 NVS）保存配置/计数器等小数据，含 namespace、各类型读写、只读模式。建议替代 Arduino EEPROM。 |
| **Serial (UART) 串口通信**<br>`repos/arduino-esp32/recipes/serial_uart.md` | 配置 ESP32 硬件 UART（`Serial`/`Serial1`/`Serial2`），包括自定义引脚、波特率、RX 缓冲区、`onReceive` 回调与 RS485 半双工模式。 |
| **SPI 主机通信**<br>`repos/arduino-esp32/recipes/spi_master.md` | 使用 `SPI` 库在多总线上以主机方式与外设（传感器、SD、显示屏等）通信，含 `SPIClass` 自定义引脚与多总线复用。 |
| **WebServer HTTP 服务（路由 / 认证 / 文件上传 / 中间件 / OTA 上传）**<br>`repos/arduino-esp32/recipes/webserver.md` | 使用 `WebServer` 库（基于 3.x `Network` 库）实现 HTTP 路由、Basic/Digest 认证、文件上传、`serveStatic`、流式响应，并与 `Update` 联动做网页 OTA。单客户端同步模型。 |
| **Wi-Fi STA / SoftAP / 扫描 / 事件**<br>`repos/arduino-esp32/recipes/wifi_sta_ap.md` | 用 `WiFi` 库以 Station 模式连接 AP、SoftAP 开热点、扫描周边网络、`onEvent` 接收连接事件，含 `setHostname` 时机与重连。 |
| **Zigbee 3.0 设备（Coordinator / Router / End Device，on-off/light/sensor/网关）**<br>`repos/arduino-esp32/recipes/zigbee.md` | 使�� `Zigbee` 库（基于 ESP-ZIGBEE-SDK）创建 Zigbee 终端设备/路由/协调器，注册 `ZigbeeEP` 端点（灯/开关/传感器/网关），含组网、绑定、睡眠终端与 RCP 网关。⚠️ 仅 ESP32-C6/C5/H2 有原生 802.15.4 无线电；其它 SoC 需外接 RCP。 |

### `ESP8266_RTOS_SDK` (18 recipes)

| recipe | 摘要 |
|---|---|
| **构建与烧录（make / cmake 双轨）**<br>`repos/ESP8266_RTOS_SDK/recipes/build_and_flash.md` | 用 make 或 cmake/`idf.py` 配置（menuconfig）、构建、烧录、擦除、监视 ESP8266_RTOS_SDK 项目。 |
| **深度睡眠与 WiFi 省电（Deep sleep & power save）**<br>`repos/ESP8266_RTOS_SDK/recipes/deep_sleep_power_save.md` | ESP8266 的两类省电：①WiFi modem sleep（`esp_wifi_set_ps`，STA 连上 AP 后周期性关 RF，保持连接，三种模式 NONE/MIN/MAX）；②deep sleep（`esp_deep_sleep`，关 CPU+RF，定时器或 RST 唤醒，唤醒等同重启）。前者在连接态省电，后者用于周期性采集场景。 |
| **ESPNOW 设备间通信**<br>`repos/ESP8266_RTOS_SDK/recipes/espnow.md` | 用 ESPNOW 在 ESP8266 之间做免连接的低延迟通信，支持广播/单播、PMK/LMK 加密、对端列表管理；回调在 WiFi 任务中触发，应用通过队列转交任务处理。 |
| **GPIO 输入输出与中断**<br>`repos/ESP8266_RTOS_SDK/recipes/gpio.md` | 用 `gpio_config` 统一配置 ESP8266 GPIO 输入/输出/上下拉与中断类型，安装 per-pin ISR 服务、通过队列把中断转交任务处理。 |
| **第一个项目 hello_world**<br>`repos/ESP8266_RTOS_SDK/recipes/hello_world_project.md` | 从零创建第一个 ESP8266_RTOS_SDK（esp-idf style）项目，完成配置、构建、烧录与串口监视，验证工具链与环境就绪。 |
| **固件 OTA 升级**<br>`repos/ESP8266_RTOS_SDK/recipes/http_ota.md` | 两种真实 OTA 路线 —— (A) native OTA：用 socket 从 HTTP 服务器拉取镜像，经 `esp_ota_begin/write/end/set_boot_partition` 写入备用 OTA 分区并重启；(B) esp_https_ota 简易接口：一行调用基于 HTTPS 拉取升级。 |
| **HTTP GET 请求（POSIX socket）**<br>`repos/ESP8266_RTOS_SDK/recipes/http_request.md` | 在已联网的 ESP8266 上，用 BSD socket（`getaddrinfo`/`socket`/`connect`/`write`/`read`）发起 HTTP GET 请求并打印响应。适用于明文 HTTP；HTTPS 见 `examples/protocols/https_mbedtls` 或 `esp_http_client`。 |
| **HTTP 服务器 esp_http_server（URI handler）**<br>`repos/ESP8266_RTOS_SDK/recipes/http_server.md` | 用 `esp_http_server` 组件在 ESP8266 上跑 RESTful/控制平面 HTTP 服务，注册 URI handler（`HTTP_GET`/`HTTP_POST`/`HTTP_PUT`）处理 `/hello`、`/echo` 等，支持读请求头/查询串、`httpd_resp_send`/`httpd_resp_send_chunk` 响应、运行期动态注册/注销 handler。是 `http_request.md`（出站请求）的入站对应。 |
| **I2C 主机读写**<br>`repos/ESP8266_RTOS_SDK/recipes/i2c.md` | 用 I2C 主机驱动（仅 `I2C_NUM_0`）通过命令链（command link）对从设备发起 start/write/read/stop 序列，以 MPU6050 为实例演示寄存器写与读。 |
| **MQTT 客户端（TCP/SSL/双向认证/PSK/WS/WSS）**<br>`repos/ESP8266_RTOS_SDK/recipes/mqtt.md` | 在已联网的 ESP8266 上用 esp-mqtt 组件连接 MQTT broker，支持 6 种传输（`mqtt://`、`mqtts://`、双向认证、PSK、`ws://`、`wss://`），基于事件回调处理连接/订阅/发布/数据；客户端在独立内部任务中运行，应用只通过 `esp_mqtt_client_register_event` 注册回调。 |
| **PWM 多通道输出**<br>`repos/ESP8266_RTOS_SDK/recipes/pwm.md` | 用软件 PWM 驱动（`pwm_init`）配置多通道（最多 8 路）的周期、占空比、相位，运行时调整并调用 `pwm_start` 生效，支持停机与通道反相。 |
| **SmartConfig 一键配网**<br>`repos/ESP8266_RTOS_SDK/recipes/smartconfig.md` | 用 SmartConfig（Esptouch / AirKiss / Esptouch v2）让 ESP8266 通过手机 App 获取 SSID 与密码并连上 AP；基于 `SC_EVENT` 事件流处理。 |
| **SNTP 网络对时与系统时间**<br>`repos/ESP8266_RTOS_SDK/recipes/sntp_time.md` | 联网后用 LwIP SNTP 模块从 NTP 服务器（如 `pool.ntp.org`）同步系统时间，通过 `time()`/`localtime_r()` 读取，用 POSIX `setenv("TZ",...)` + `tzset()` 设置时区。时间同步是 TLS 证书校验、日志时间戳、定时任务的前置需求。 |
| **BSD Socket：TCP/UDP 服务端客户端与组播**<br>`repos/ESP8266_RTOS_SDK/recipes/sockets.md` | 用 lwIP BSD socket API 实现 TCP 服务端（`socket`→`bind`→`listen`→`accept`→`recv/send`）、TCP 客户端（`connect`）、UDP 收发与 IPv4 组播（`setsockopt IP_ADD_MEMBERSHIP/IP_MULTICAST_TTL`）。本 recipe 补充 `http_request.md` 没覆盖的服务端/组播场景。 |
| **SPIFFS 文件系统读写**<br>`repos/ESP8266_RTOS_SDK/recipes/spiffs.md` | 在 ESP8266 上挂载 SPIFFS 文件系统，用 POSIX/C 标准库函数（fopen/fprintf/fgets/rename/unlink/stat）进行文件读写，并查询分区容量。 |
| **UART 收发与事件队列**<br>`repos/ESP8266_RTOS_SDK/recipes/uart.md` | 用 UART 驱动（`uart_param_config` + `uart_driver_install`）配置串口，通过事件队列处理接收/溢出/错误，用 `uart_write_bytes` / `uart_read_bytes` 收发数据。 |
| **WiFi SoftAP 热点**<br>`repos/ESP8266_RTOS_SDK/recipes/wifi_softap.md` | 把 ESP8266 配为 SoftAP 开放热点，监听 station 的连接/离开事件，设置最大连接数与认证模式。 |
| **WiFi Station 连接 AP**<br>`repos/ESP8266_RTOS_SDK/recipes/wifi_station.md` | 把 ESP8266 配为 Station 连接到指定 AP，基于 esp_event 处理连接/断开/拿 IP，带最大重试次数，用事件组阻塞等待联网结果。 |

## 领域框架


### `esp-adf` (15 recipes)

| recipe | 摘要 |
|---|---|
| **音频处理元素：EQ / Downmix / Sonic / Resample / ALC**<br>`repos/esp-adf/recipes/audio_processing.md` | 在 pipeline 中插入音频处理 element 实现均衡器（EQ）、混音（Downmix）、变速变调（Sonic）、重采样（Resample Filter）、自动电平控制（ALC）。这些 element 通常位于解码器与输出 stream 之间。 |
| **蓝牙服务：A2DP Sink / Source 与 HFP**<br>`repos/esp-adf/recipes/bt_a2dp_hfp.md` | 用 `bluetooth_service` 创建蓝牙音频流元素与外设，接入 pipeline 实现蓝牙音箱（A2DP Sink）、蓝牙音源（A2DP Source）或免提通话（HFP）。`bluetooth_service_create_stream()` 返回可直接 link 的 audio element。 |
| **CLI 命令行交互**<br>`repos/esp-adf/recipes/cli_console.md` | 用 ESP-IDF 的 `esp_console`（线性命令行）在 ESP-ADF 工程里注册命令，配合串口做交互调试与控制（如手动切歌、调音量、查状态）。ESP-ADF 自带 `cli` 示例演示命令注册与执行。 |
| **云端 TTS 语音合成：百度 / AWS Polly 在线 TTS**<br>`repos/esp-adf/recipes/cloud_tts.md` | 调云端 TTS 接口把文本转成 MP3，再走 `http_stream → mp3_decoder → i2s_stream` 播放。百度用 API Key/Secret Key 换 access_token 后 POST 表单（`http://tsn.baidu.com/text2audio`）；AWS Polly 需 SNTP 校时 + AWS4-HMAC-SHA256 签名（`https://polly.<region>.amazonaws.com/v1/speech`）。两者都用 `http_stream` 的 `event_handle` 回调在 `HTTP_STREAM_PRE_REQUEST` 阶段注入鉴权头/请求体。数据流：`[cloud TTS] → http_stream(reader) → mp3_decoder → i2s_stream(writer) → codec`。 |
| **显示服务：LED 灯效模式(display_service)与 LED 驱动**<br>`repos/esp-adf/recipes/display_service.md` | 用 `display_service_set_pattern(handle, pattern, value)` 按功能语义（WiFi 连接中 / 蓝牙已连 / 唤醒 / 播放中 / 录音中 / 音量 / 低电等）点亮板载 LED，底层驱动由 `audio_board_led_init()` 根据 `CONFIG_*_BOARD` 自动选：PWM 单 LED（`led_indicator`）、AW2013、IS31x、WS2812。`display_pattern_t` 枚举定义了所有可用灯效，驱动里实现不支持的 pattern 会打 `LED_INDI: The led mode is invalid` 警告并安全跳过。 |
| **OTA 固件升级服务**<br>`repos/esp-adf/recipes/ota_service.md` | 用 `ota_service` 组件从网络（HTTP）下载新固件并写入 OTA 分区。服务内置版本比较、分区管理、流式写入与错误码（`OTA_SERV_ERR_REASON_*`）。 |
| **外设集（Wi-Fi / SD / 触摸 / 按键）与事件集成**<br>`repos/esp-adf/recipes/peripherals_services.md` | ESP-ADF 用 `esp_peripherals` 统一管理外设（Wi-Fi、SD 卡、触摸、ADC 按键、GPIO 按键等），所有外设事件汇入同一个 `audio_event_iface`，与 pipeline 事件一起处理。本 recipe 覆盖外设初始化与事件分发。 |
| **Stream 速查与组合**<br>`repos/esp-adf/recipes/pipeline_streams.md` | 汇总 ESP-ADF 各类 stream 的初始化宏、类型与典型用法，方便按数据源/输出口快速选型。所有 stream ���是 `audio_element_handle_t`，用统一的 register/link 模式接入 pipeline。 |
| **通过 HTTP/HTTPS 流式播放 MP3**<br>`repos/esp-adf/recipes/play_http_mp3.md` | 用 http_stream 从网络读 MP3，经 mp3_decoder 解码，i2s_stream 输出到 codec。包含 Wi-Fi 连接（periph_wifi）与事件监听。数据流：HTTP server → http_stream → mp3_decoder → i2s_stream → codec。 |
| **从 Flash 内存播放 MP3**<br>`repos/esp-adf/recipes/play_mp3_flash.md` | 将 MP3 数据以二进制嵌入 Flash，通过自定义 read 回调喂给 mp3_decoder，再经 i2s_stream 写到 codec 芯片播放。是 ESP-ADF 最经典的入门 pipeline（Flash → mp3_decoder → i2s）。 |
| **从 SD 卡（FatFs）播放音乐**<br>`repos/esp-adf/recipes/play_sdcard_fatfs.md` | 用 fatfs_stream 作为 reader 从 SD 卡读音频文件，接解码器与 i2s 输出。SD 卡通过 `audio_board_sdcard_init` 挂载。数据流：SD 卡 → fatfs_stream(reader) → decoder → i2s_stream(writer) → codec。 |
| **播放列表管理：SD 卡扫描(sdcard_scan) + 多存储后端 playlist**<br>`repos/esp-adf/recipes/playlist.md` | 用 `sdcard_scan()` 扫描 SD 卡音频文件并经回调存入 playlist，再用 `playlist_operator_handle_t` 句柄做 next / prev / choose(id) / current，由 `playlist_handle_t` 管理多个列表。四种存储后端：SD 卡（`sdcard_list`）、DRAM（`dram_list`）、NVS Flash（`flash_list`）、DATA_UNDEFINED 分区（`partition_list`）。播放时用 `audio_element_set_uri()` 把列表给出的 URL 喂给 fatfs_stream 并 reset pipeline 切歌。 |
| **录音并编码 WAV/AMR 写入 SD 卡**<br>`repos/esp-adf/recipes/record_to_sdcard.md` | 用 i2s_stream(reader) 从 codec 麦克风取音，接编码器（wav_encoder / amrwb_encoder / amrnb_encoder），再经 fatfs_stream(writer) 写入 SD 卡。数据流：mic → i2s(reader) → encoder → fatfs(writer) → SD。 |
| **语音识别与录音器：WakeNet 唤醒 + MultiNet 命令词 + VAD + audio_recorder**<br>`repos/esp-adf/recipes/speech_recognition.md` | 用 `audio_recorder` 高层录音器把 AFE（AEC/AGC/NS）、WakeNet 唤醒词、MultiNet 命令词识别、VAD 与编码整合进一个事件驱动的 recorder 管线，通过 `rec_engine_cb` 回调收 `AUDIO_REC_WAKEUP_START` / `AUDIO_REC_VAD_START` / `AUDIO_REC_COMMAND_DECT` 等事件。数据流：mic → i2s(reader) → rsp_filter(16kHz) → raw_stream(reader) → AFE → WakeNet → MultiNet → (可选 encoder) → `audio_recorder_data_read`。底层 VAD（`esp_vad`）也可独立用于纯「是否有人说话」检测。 |
| **VoIP / SIP 网络电话：SIP/RTP 通话与 AEC 回声消除**<br>`repos/esp-adf/recipes/voip_sip.md` | 用 `av_stream_init` 建一条带 AEC 的双向音频管线（mic→算法流→G.711 编码→RTP；RTP→G.711 解码→算法流→喇叭），再用 SIP 服务（基于 `esp_rtc`）注册到 PBX（FreeSWITCH/Asterisk），实现接/打电话。`wifi_service` 管联网，`input_key_service` 把按键映射到 call/answer/hangup/volume。数据流参考 `examples/protocols/voip`。 |

### `esp-gmf` (16 recipes)

| recipe | 摘要 |
|---|---|
| **运行时调整音频效果（EQ/ALC/Sonic/Fade/DRC/MBC）**<br>`repos/esp-gmf/recipes/audio_effects.md` | 在播放流水线运行过程中，通过 element 命名 setter 或运行时方法（AMETHOD）实时调整均衡、自动增益、变速变调、淡入淡出、动态范围控制等效果。 |
| **自定义音频 Element 模板**<br>`repos/esp-gmf/recipes/custom_element.md` | 实现一个自定义 audio element（含 open/process/close 生命周期、输入输出端口属性、acquire/release 数据协议），并注册进 pool 参与 pipeline。 |
| **多流混音渲染（esp_audio_render）**<br>`repos/esp-gmf/recipes/esp_audio_render.md` | 用 `esp_audio_render` 把多路 PCM 输入（如「音乐 + TTS + 提示音」或 4 轨钢琴合成）混音为统一输出格式后由 writer 回调送出（I2S/BT sink/网络皆可）。每路可独立挂 per-stream 处理链（ALC/EQ/Sonic/Fade），混音后还可挂 post-mix 处理（ALC/limiter）。区别于单流的 `simple_player.md`/`esp_player.md`，本包面向「多轨合成 + 统一输出」。 |
| **蓝牙音频（经典 BT A2DP/HFP + LE Audio）**<br>`repos/esp-gmf/recipes/esp_bt_audio.md` | 用 `esp_bt_audio` 统一管理经典蓝牙（A2DP Sink/Source 扬声器耳机、HFP HF/AG 免提通话、AVRCP、PBAP 通讯录）与 LE Audio（TMAP/BAP 单播与广播、VCP/MCP/CSIP 等）。一次 `esp_bt_audio_init` 自动初始化对应协议栈，profile 差异收敛到单一事件回调；音频数据可经 `esp_bt_audio_stream_*` 直读直写，或用 `esp_gmf_io_bt` 接入 GMF 流水线解码/编码后送喇叭/麦克风。 |
| **用 esp_capture 做高级音视频采集**<br>`repos/esp-gmf/recipes/esp_capture.md` | 用高级包 `esp_capture` 按「source → path → sink」模型从麦克风/摄像头采集音视频，自动按 source/target 格式协商编码与转码链，支持 AEC 采集、多 sink 并行（一路流式拉帧 + 一路 MP4 本地存储）、overlay 文字叠加、单帧抓拍与自定义处理流水线。与 `pipeline_record.md`（GMF-Core 手写的 `codec_dev→aud_enc→io_file` 单链）不同，`esp_capture` 封装了整套协商与多路分发逻辑。 |
| **用 esp_player 做音视频同步播放**<br>`repos/esp-gmf/recipes/esp_player.md` | 用 `esp_player` 单实例串联解封装、解码、渲染，支持本地文件、HTTP(S)、HLS、外部帧（fill/block）输入，含 A/V 同步、seek、变速、多轨道选择。 |
| **视频合成与显示叠加（esp_video_render）**<br>`repos/esp-gmf/recipes/esp_video_render.md` | 用 `esp_video_render` 把一路或多路视频（H.264/MJPEG 解码输入）与 UI 叠加（overlay→container→widget 层级）合成到统一显示后端（直接 LCD / LVGL 集成 / framebuffer）。每路 stream 独立控制位置/裁剪/旋转/显隐/z-order/alpha；dirty-region 部分刷新减少重绘；支持同步或异步渲染、手动 compose 模式。覆盖视频播放器、智能屏、机器人双眼、视频门铃、摄像头预览等场景。 |
| **AI 语音前端（gmf_ai_audio + esp-sr）**<br>`repos/esp-gmf/recipes/gmf_ai_audio.md` | 用 `gmf_ai_audio` 把 `esp-sr`（AEC/NS/VAD/唤醒词 WakeNet/命令词 MultiNet/DOA）封装成 6 个 GMF element：`ai_afe`（全功能统一入口，含 feed/fetch 双任务、wakeup+VAD 状态机、命令词与手动唤醒）、`ai_aec`/`ai_wn`/`ai_ns`/`ai_vad`/`ai_doa`（单能力 element，可直接接入 GMF pipeline）。配套 `esp_gmf_afe_manager` 可脱离 pipeline 独立使用。 |
| **无缝循环播放**<br>`repos/esp-gmf/recipes/loop_play.md` | 用 `esp_gmf_task_set_strategy_func` 注册策略函数，在播放结束（FINISH/ABORT）时返回 `GMF_TASK_STRATEGY_ACTION_RESET` 自动续播下一曲，配合 IO reload 实现「不重建 pipeline 的无缝切歌」。 |
| **用 aud_muxer 封装容器录制**<br>`repos/esp-gmf/recipes/pipeline_muxer.md` | 在录音/转码流水线末尾加 `aud_muxer`，把编码后的音频封装为 TS/MP4/FLV/WAV/CAF/OGG/AVI 容器，支持流式输出或分片文件写入。 |
| **基于流水线播放 Flash 内嵌音乐**<br>`repos/esp-gmf/recipes/pipeline_play_embed.md` | 用 GMF-Core 的 pool/pipeline 构建「embed_flash → aud_dec → 音频效果链 → codec_dev」最小播放流水线，覆盖 pool 注册、pipeline 构建、task 绑定与事件等待。是理解 ESP-GMF 工作流的入门模板。 |
| **播放 HTTP/HTTPS 网络音乐**<br>`repos/esp-gmf/recipes/pipeline_play_http.md` | 构建「io_http → aud_dec → 效果链 → io_codec_dev」流水线播放网络音频，处理 HTTPS 的 TLS 握手栈、异步 IO 预取与控制超时。 |
| **录音并编码到 SD 卡**<br>`repos/esp-gmf/recipes/pipeline_record.md` | 构建「io_codec_dev → aud_enc → io_file」录音流水线，从麦克风采集 PCM 并编码为 AAC/AMR/OPUS 等格式写入 SD 卡，含编码器 reconfig、主动上报格式信息与录音栈配置。 |
| **运行时方法（AMETHOD）解耦接口与实现**<br>`repos/esp-gmf/recipes/runtime_methods.md` | 用 `AMETHOD(MODULE, METHOD)` 拼装方法名字符串、经 `esp_gmf_element_exe_method` 调用 element，使应用层只依赖方法名而非具体 element 类型，便于在 pool 中替换实现（如 aud_rate_cvt ↔ aud_asrc）。 |
| **用 esp_audio_simple_player 快速播放**<br>`repos/esp-gmf/recipes/simple_player.md` | 用高级包 `esp_audio_simple_player` 以 URI 驱动播放音频（file/http/embed/raw），自动按 scheme 选 IO、按扩展名选解码器，支持同步/异步、事件回调与运行时 pause/stop。 |
| **状态机、ERROR 恢复与控制接口**<br>`repos/esp-gmf/recipes/state_error_recovery.md` | 理解 pipeline/task 状态机、各控制 API 的有效状态、ERROR 后的恢复步骤，以及 stop 超时、pause/resume、seek 的正确用法。 |

### `esp-wdf` (13 recipes)

| recipe | 摘要 |
|---|---|
| **WAMR App Framework 定时器应用**<br>`repos/esp-wdf/recipes/app_framework_timer.md` | 使用 ESP-WDF 的 WAMR App Framework，以 `on_init()`/`on_destroy()` 为入口，通过 `api_timer_create` 创建周期定时器并在回调中执行逻辑。 |
| **云服务与配网（HTTP / MQTT / RainMaker / Wi-Fi Provisioning）**<br>`repos/esp-wdf/recipes/cloud_protocols.md` | WASM 应用通过 ESP-WDF 扩展适配层使用 ESP-IDF 风格 API：ESP HTTP Client 发起请求、ESP-MQTT 收发消息、ESP-RainMaker 上报设备、Wi-Fi Provisioning 配网。各模块由对应 Kconfig 开关启用（默认均为 `y`）。 |
| **应用间事件发布/订阅**<br>`repos/esp-wdf/recipes/event_pub_sub.md` | 用 `api_publish_event` / `api_subscribe_event` 在 WASM 应用之间通过事件 URL 异步通信，负载使用 `attr_container_t` 结构化容器。 |
| **GPIO 外设（VFS + ioctl）**<br>`repos/esp-wdf/recipes/gpio_vfs.md` | WASM 应用通过设备节点 `/dev/gpio/<pin>` 访问 GPIO：`open` 拿到 fd，`ioctl(GPIOCSCFG)` 配置上下拉，`write` 翻转电平，`close` 释放。 |
| **I2C 传感器读取（BH1750）**<br>`repos/esp-wdf/recipes/i2c_sensor.md` | WASM 应用通过 `/dev/i2c/0` 设备节点作 I2C 主机，`ioctl(I2CIOCSCFG)` 配置，用 `I2CIOCRDWR`（两步）或 `I2CIOCEXCHANGE`（一步收发）读取 BH1750 光照传感器。 |
| **I2C + GPIO 中断驱动的电容触摸屏读取（TT21100）**<br>`repos/esp-wdf/recipes/i2c_touch_gpio_irq.md` | WASM 应用同时打开 I2C 设备 `/dev/i2c/0` 与 GPIO 设备 `/dev/gpio/<ready_pin>`，轮询 READY 引脚电平，当 READY 拉低时用两步 I2C 读取（先读 2 字节长度字，再读该长度字节）解析 TT21100 触点/按键报告。与 `i2c_sensor.md`（BH1750，单外设、定长读取）不同，本配方处理“GPIO 数据就绪门控 + 变长结构化多字节报告”这一真实 HMI 输入模式。 |
| **LVGL 图形界面（WASM 适配）**<br>`repos/esp-wdf/recipes/lvgl_gui.md` | WASM 应用用 ESP-WDF 提供的 LVGL WASM 适配 API 构建界面——用 `lvgl_init`/`lvgl_lock`/`lvgl_unlock` 异步初始化，用 `lv_obj_get_data` 等访问器替代直接解引用结构指针。 |
| **POSIX 多线程与同步**<br>`repos/esp-wdf/recipes/multithread.md` | WASM 应用使用 `pthread_create/join` 创建线程，配合 `pthread_mutex_t` 互斥锁与 `pthread_cond_t` 条件变量做同步。 |
| **创建并编译一个 WASM 应用**<br>`repos/esp-wdf/recipes/new_wasm_app.md` | 从零创建 ESP-WDF WebAssembly 应用工程，配置 `sdkconfig.defaults`、`main/CMakeLists.txt`，编译出 `.wasm`/`.aot` 固件，并通过 `host_tool.py` 安装运行。 |
| **应用间请求/响应**<br>`repos/esp-wdf/recipes/request_response.md` | 用 `api_register_resource_handler` 注册资源处理者，用 `init_request` + `api_send_request` 异步发起请求并接收响应，构造 `response_t` 回送结果。 |
| **BSD Socket 网络通信**<br>`repos/esp-wdf/recipes/sockets_network.md` | WASM 应用用标准 BSD socket API（`socket/connect/send/recv` 等）实现 TCP 客户端/服务端；在 `__wasi__` 环境下需包含 `wasi_socket_ext.h`，并配置特殊 sdkconfig.defaults。 |
| **SPI 主机收发与 LEDC 占空比**<br>`repos/esp-wdf/recipes/spi_ledc.md` | WASM 应用通过 `/dev/spi/2` 作 SPI 主机收发（`SPIIOCSCFG` + `SPIIOCEXCHANGE`），或通过 `/dev/ledc/0` 控制 LEDC 通道占空比（`LEDCIOCSCFG` + `LEDCIOCSSETDUTY`）。 |
| **UART 输出与文件系统读写**<br>`repos/esp-wdf/recipes/uart_filesystem.md` | WASM 应用通过 `/dev/uart/0`（或 `/dev/usbserjtag`）`write` 串口数据；通过 `/storage/<file>` 路径用 `open/write/read/lseek/close` 读写 VFS 文件。 |

### `esp-wasmachine` (13 recipes)

| recipe | 摘要 |
|---|---|
| **App Manager 与 host_tool 远程管理**<br>`repos/esp-wasmachine/recipes/app_manager.md` | 启用 WAMR App Manager + TCP 服务，用 shell 的 `install`/`uninstall`/`query` 或 Linux 上的 `host_tool` 远程安装、卸载、查询常驻 WASM applet。 |
| **编译、烧录与监视**<br>`repos/esp-wasmachine/recipes/build_flash.md` | 完成 ESP-WASMachine 固件的 `set-target`、`build`、`storage-flash`、`flash monitor` 全流程，并验证启动日志。 |
| **data_sequence：VM 与 WASM 应用的参数序列化**<br>`repos/esp-wasmachine/recipes/data_sequence.md` | 使用 `wasmachine_data_sequence` 组件（`data_seq`）在固件/VM 与 WASM 应用之间序列化传递参数，是 native 模块 `ioctl`/属性容器的底层机制。 |
| **Extended VFS（UART / GPIO / I2C / SPI / LEDC）**<br>`repos/esp-wasmachine/recipes/ext_vfs.md` | 启用 Extended VFS，让 WASM 应用通过 `/dev/uart/x` 与 `ioctl` 访问 UART、GPIO、I2C、SPI、LEDC 外设。 |
| **littleFS 文件系统与 WASM 应用打包**<br>`repos/esp-wasmachine/recipes/filesystem.md` | 把 `.wasm` 文件放进 `main/fs_image/`，编译生成并烧录 `storage` 分区的 littleFS 镜像，让 `iwasm`/`install` 能读到。 |
| **WASM Native HTTP 与 MQTT**<br>`repos/esp-wasmachine/recipes/native_http_mqtt.md` | 启用并使用 ESP-WASMachine 暴露给 WASM 应用的 HTTP client 与 MQTT native API（基于 `esp_http_client` 与 `esp-mqtt`，通过函数 ID 派发）。 |
| **WASM Native libc（文件 I/O 与基础 POSIX）**<br>`repos/esp-wasmachine/recipes/native_libc.md` | 启用并理解 WASM libc native API（`open`/`read`/`write`/`ioctl`/`close`/`sleep`/`time`/`rand` 等），WASM 应用通过它访问 VFS 文件与外设。 |
| **WASM Native LVGL 与 BSP 显示**<br>`repos/esp-wasmachine/recipes/native_lvgl.md` | 在 ESP32-S3-BOX 或 ESP32-P4-Function-EV-Board 上启用 LVGL native，注册 BSP 显示回调，让 WASM 应用调用 LVGL 绘制 UI。 |
| **WASM Native RainMaker 与 Wi-Fi 配网**<br>`repos/esp-wasmachine/recipes/native_rainmaker_prov.md` | 启用 RainMaker 与 Wi-Fi provisioning native API（基于 ESP RainMaker 与 `wifi_provisioning`），供 WASM 应用集成云与 SoftAP/BLE 配网。 |
| **新建 WASMachine 固件项目**<br>`repos/esp-wasmachine/recipes/new_project.md` | 从 `examples/wasmachine` 拷贝并配置一个新的 ESP-WASMachine（WebAssembly 虚拟机）固件工程，确定目标芯片、分区表与启用的组件。 |
| **用 iwasm 运行 WASM 应用**<br>`repos/esp-wasmachine/recipes/run_iwasm.md` | 在 `WASMachine>` 控制台用 `iwasm` 从文件系统加载并一次性运行 WASM 应用，设置栈/堆与 WASI 环境变量、目录、地址池。 |
| **Shell sta 命令：连接 AP**<br>`repos/esp-wasmachine/recipes/shell_wifi.md` | 用 `WASMachine> sta` 命令让设备以 Station 模式连接 AP，为远程 App 管理（`host_tool`）与联网 native（HTTP/MQTT/RainMaker）准备网络。 |
| **选择目标板与 sdkconfig.defaults**<br>`repos/esp-wasmachine/recipes/target_board.md` | 为 ESP32 / ESP32-S3（含 BOX）/ ESP32-C6 / ESP32-P4 选择正确的 `set-target` 与板级 `sdkconfig.defaults` 覆盖。 |

### `esp-brookesia` (12 recipes)

| recipe | 摘要 |
|---|---|
| **AI 语音助手（AgentManager + 多 Agent + MCP）**<br>`repos/esp-brookesia/recipes/agent_chatbot.md` | 基于 `brookesia_agent_manager` 构建完整 AI 语音助手——初始化 AgentManager、配置 XiaoZhi/Coze/OpenAI 多 Agent、通过 MCP 工具让 LLM 调用设备能力、用状态机（Activate→Start→Sleep→WakeUp→Stop）驱动生命周期、配合 Audio AFE 做语音唤醒与 Emote 做表情反馈。 |
| **Audio 服务：播放、控制、编解码与 AFE**<br>`repos/esp-brookesia/recipes/audio_service.md` | 使用 `brookesia_service_audio` 播放本地/网络音频、多 URL 队列、播放控制（暂停/恢复/停止）、编解码（PCM/OPUS/G711A）回环，以及 AFE（VAD + 唤醒词）。Audio 服务依赖 HAL 设备，需先初始化存储与音频设备。 |
| **Custom 服务：动态注册函数与事件**<br>`repos/esp-brookesia/recipes/custom_service.md` | 使用 `brookesia_service_custom` 提供的 `CustomService`，在运行期动态注册自定义 function 与 event，把 LED/PWM/传感器等轻量逻辑封装为统一可本地/远程调用的服务能力，无需单独开发 Brookesia 组件。 |
| **Device 服务：能力查询与外设控制**<br>`repos/esp-brookesia/recipes/device_service.md` | 使用 `brookesia_service_device` 查询板级能力（capabilities）、读取板信息、控制显示背光与音频播放器（音量/静音）、查询存储文件系统与电池、订阅状态变化事件。Device 服务依赖 HAL，且常与 NVS 服务一起 bind（用于 `ResetData` 持久化）。 |
| **Emote 表情模块：表情、动画与事件消息**<br>`repos/esp-brookesia/recipes/expression_emote.md` | 使用 `brookesia_expression_emote`（Helper 类 `service::helper::ExpressionEmote`）为 AI 交互提供拟人化视觉反馈——加载表情/动画资源、设置 emoji、显示/隐藏事件消息文本、显示二维码、插入动画。Emote 通常与 Agent 状态事件联动。 |
| **HAL Boards 选板与设备初始化**<br>`repos/esp-brookesia/recipes/hal_boards.md` | 通过 `brookesia_hal_boards` 选择开发板、用 `brookesia_hal_adaptor` 初始化 HAL 设备，并按接口名查询硬件能力（音频/显示/触摸/存储/电源/背光）。依赖外设的服务（Audio/Device/Agent）必须先完成 HAL 初始化。 |
| **NVS 服务：键值存储**<br>`repos/esp-brookesia/recipes/nvs_service.md` | 使用 `brookesia_service_nvs` 做基于命名空间的键值存储。提供两套 API：类型安全的 `save_key_value`/`get_key_value`（推荐）与通用 JSON 的 `Set`/`Get`/`List`/`Erase`（细粒度控制）。 |
| **项目搭建与第一次服务调用**<br>`repos/esp-brookesia/recipes/project_setup.md` | 新建 ESP-Brookesia 项目：声明组件依赖、初始化 ServiceManager、绑定服务、完成第一次服务调用。这是使用任何 Brookesia 服务（Wi-Fi / NVS / Audio 等）的统一前置步骤。 |
| **服务控制台：CLI 调试、跨设备 RPC 与运行期性能剖析**<br>`repos/esp-brookesia/recipes/service_console_rpc_debug.md` | 使用 ESP-Brookesia 自带的 `examples/service/console` 串口交互 CLI，在运行期驱动任意服务/Agent（`svc_call`/`svc_subscribe` 等）、跨设备经 Wi-Fi 进行 RPC 远程调用与事件订阅（`svc_rpc_server`/`svc_rpc_call`/`svc_rpc_subscribe`），并用 `debug_mem`/`debug_thread`/`debug_time_report` 三类剖析器诊断内存、CPU 与耗时。本配方不写应用层 C++，而是教你编译烧录该控制台示例、用命令在线编排设备——这是框架对外宣称的 "Agent CLI" 与 "RPC-based remote communication" 能力的唯一官方入口。 |
| **服务框架通用范式：调用与订阅**<br>`repos/esp-brookesia/recipes/service_framework_basics.md` | 掌握 ESP-Brookesia 服务框架的统一调用范式——ServiceManager 启动、bind、同步/异步函数调用、事件订阅与 EventMonitor 阻塞等待。这是使用所有具体服务（Wi-Fi/NVS/Audio 等）的共同基础。 |
| **SNTP 服务：网络时间同步**<br>`repos/esp-brookesia/recipes/sntp_service.md` | 使用 `brookesia_service_sntp` 配置 NTP 服务器与时区，查询同步状态、服务器列表与时区。SNTP 可选依赖 NVS 做持久化，网络可用后自动同步。 |
| **Wi-Fi 服务：扫描、连接与 SoftAP 配网**<br>`repos/esp-brookesia/recipes/wifi_service.md` | 使用 `brookesia_service_wifi` 完成 AP 扫描、STA 连接/断开、SoftAP 配网、状态/历史查询与事件订阅。Wi-Fi 服务为纯芯片工程，无需 HAL 设备初始化。 |

### `esp-iot-solution` (18 recipes)

| recipe | 摘要 |
|---|---|
| **BLE 连接管理 esp_ble_conn_mgr**<br>`repos/esp-iot-solution/recipes/ble_conn_mgr.md` | 使用 `ble_conn_mgr` 组件的简化 API（`esp_ble_conn_init` / `esp_ble_conn_start`）搭建 BLE 外设/中心角色，覆盖周期广播（periodic advertising）、周期同步（periodic sync）、SPP 串口透传、L2CAP CoC 信道。组件基于 NimBLE，用 `esp_ble_conn_config_t` 配设备名/广播数据/扩展广播/周期广播，事件经 `esp_event` 投递到 `BLE_CONN_MGR_EVENTS`。 |
| **BLE GATT Profiles 与服务（OTA 固件升级、HTP 健康体温）**<br>`repos/esp-iot-solution/recipes/ble_profiles_ota.md` | 在 `esp_ble_conn_mgr` 之上使用 `ble_profiles` 组件的��准 GATT 服务与 profile：`esp_ble_ota_raw` 提供基于扇区 CRC 校验的 BLE OTA 固件升级（服务 UUID `0x8018`）；`esp_ble_htp` 提供健康体温计 profile（服务 UUID `0x1809`）。profile 内部注册服务和特征值，应用只需 init + 注册回调。 |
| **BTHome 协议（Home Assistant 集成）**<br>`repos/esp-iot-solution/recipes/bthome.md` | 使用 `bthome_v2` 组件实现 BTHome V2 协议，对接 Home Assistant。支持加密/非加密、传感器/二元传感器/事件数据上报与解析：广播端用 `bthome_make_adv_data` + `bthome_payload_adv_add_*` 拼广播包；接收端用 `bthome_create` + `bthome_set_encrypt_key` + `bthome_parse_adv_data` 解出 `bthome_reports_t`。配合 `ble_hci` 组件做底层 BLE 收发。 |
| **GPIO 按键**<br>`repos/esp-iot-solution/recipes/button_gpio.md` | 使用 `button` 组件创建 GPIO 按键、注册各类事件回调（按下、释放、单击、双击、长按、多击），并动态修改长短按阈值。button 组件 v3.x 采用 `iot_button_new_gpio_device` 工厂函数 + `button_handle_t` 句柄模式。 |
| **按键低功耗与 Light Sleep**<br>`repos/esp-iot-solution/recipes/button_power_save.md` | 使用 `button` 组件的 power-save 模式，配合 ESP-IDF PM 框架与 light sleep，实现按键唤醒。创建设备时 `enable_power_save=true`，并注册 `iot_button_register_power_save_cb`，在所有按键空闲时进入 light sleep。 |
| **FOC 无刷电机控制 esp_simplefoc**<br>`repos/esp-iot-solution/recipes/foc_motor.md` | 使用 `esp_simplefoc` 组件（基于 Arduino-FOC，适配 ESP 的 LEDC/MCPWM）驱动三相无刷电机（BLDC），覆盖开环速度（`velocity_openloop`）与闭环速度（`velocity` + 角度传感器 AS5600/MT6701/AS5048A + PID）。入口仍是 ESP-IDF 的 `app_main`，但电机代码是 C++（`BLDCMotor` / `BLDCDriver3PWM` / `AS5600`）。 |
| **I2C 总线通信**<br>`repos/esp-iot-solution/recipes/i2c_bus.md` | 使用 `i2c_bus` 组件初始化 I2C 主机总线、创建总线上的设��句柄、进行字节/多字节/位级寄存器读写与总线扫描。`i2c_bus` 在 ESP-IDF >= 5.3 默认走新 `i2c_master` 驱动，旧版本走 `driver/i2c.h`，由 `CONFIG_I2C_BUS_BACKWARD_CONFIG` 控制。 |
| **旋转编码器旋钮（knob）**<br>`repos/esp-iot-solution/recipes/knob.md` | 使用 `knob` 组件接入 AB 相旋转编码器，创建旋钮句柄、注册左右旋/上下限/归零事件回调、读取累计计数值。组件基于 GPIO 中断实现，支持 power-save。 |
| **GPIO LED 指示灯**<br>`repos/esp-iot-solution/recipes/led_indicator_gpio.md` | 使用 `led_indicator` 组件的 GPIO 后端创建指示灯，定义 `blink_step_t` 灯效表（双闪、三闪、慢闪、快闪），按优先级启停灯效。GPIO 后端只支持开关（LED_DUTY_1_BIT），不支持亮度调节。 |
| **LEDC PWM LED 指示灯**<br>`repos/esp-iot-solution/recipes/led_indicator_ledc.md` | 使用 `led_indicator` 组件的 LEDC 后端创建支持亮度调节的单色 LED 指示灯，实现呼吸灯、亮度过渡（25%/75%）等灯效。LEDC 后端支持 `LED_BLINK_BREATHE` / `LED_BLINK_BRIGHTNESS` 动作。 |
| **RGB LED 与 WS2812 灯带（HSV/RGB 彩色灯效）**<br>`repos/esp-iot-solution/recipes/led_indicator_strips.md` | 使用 `led_indicator` 的 RGB 后端（三通道 PWM LED）与 Strips 后端（WS2812 等 RMT/SPI 寻址灯带）做彩色灯效：`LED_BLINK_RGB` / `LED_BLINK_HSV` 设颜色，`LED_BLINK_RGB_RING` / `LED_BLINK_HSV_RING` 做颜色渐变，`SET_IHSV` / `SET_IRGB` / `INSERT_INDEX` 控制灯带单颗或全部灯。颜色宏 `SET_RGB(r,g,b)`、`SET_HSV(h,s,v)` 在 `led_convert.h`。 |
| **功率计量（BL0937 / BL0942 / INA236）**<br>`repos/esp-iot-solution/recipes/power_measure.md` | 使用 `power_measure` 组件通过工厂模式创建功率计量设备（BL0937 脉冲型、BL0942 UART/SPI 型、INA236 I2C 型），读取电压、电流、有功功率、功率因数、电能。三种芯片各有独立 config 结构体与 `power_measure_new_xxx_device` 工厂函数。 |
| **sensor_hub 传感器集线器**<br>`repos/esp-iot-solution/recipes/sensor_hub.md` | 使用 `sensor_hub` 组件统一管理传感器：创建传感器实例、按类型注册事件回调、启动后周期采集并接收 `SENSOR_XXX_DATA_READY` 事件。sensor_hub 通过链接脚本机制自动加载被加入工程的传感器驱动组件。 |
| **LEDC 舵机角度控制（iot_servo）**<br>`repos/esp-iot-solution/recipes/servo.md` | 使用 `iot_servo` 组件基于 ESP-IDF LEDC 生成 PWM 控制舵机（如 SG90/MG996R），初始化通道、写入目标角度并读取当前角度。组件用 `servo_config_t` 配置最大角度、脉宽范围（典型 500~2500µs）与 PWM 频率（典型 50Hz）。 |
| **SPI 总线通信**<br>`repos/esp-iot-solution/recipes/spi_bus.md` | 使用 `spi_bus` 组件初始化 SPI 主机总线、在总线上添加设备、进行单字节/多字节/16 位/32 位传输。该组件封装了 ESP-IDF `spi_master`，提供更简洁的句柄式接口。 |
| **触摸按键（touch_button_sensor / touch_button）**<br>`repos/esp-iot-solution/recipes/touch_button.md` | 使用 ESP-IoT-Solution 的触摸按键组件在 ESP32 / ESP32-S2 / ESP32-S3（ESP32-P4 支持多频采样）上实现电容触摸检测。本 recipe 以仓库公开文档化的 `touch_button_sensor` 组件为主：配置通道列表与阈值、创建实例、注册触摸状态回调、周期性 `touch_button_sensor_handle_events` 处理事件。另提供与 `iot_button` 框架集成的 `iot_button_new_touch_button_device` 接口。注意：ESP32/S2/S3 的触摸抗干扰能力有限，仅供测试/演示，不建议量产通过 EMS 测试；本组件需 IDF >= v5.3。 |
| **USB UVC/UAC 流（usb_stream）**<br>`repos/esp-iot-solution/recipes/usb_stream.md` | 使用 `usb_stream` 组件在 ESP32-S2 / ESP32-S3 上以 USB Host 方式接入 UVC 摄像头与 UAC 音频设备，配置 UVC 帧回调、UAC 麦克风/扬声器流，并对流进行暂停/恢复与音量/静音控制。`usb_stream` 仅支持 ESP32-S2/ESP32-S3，且 UVC 摄像头须兼容 USB1.1 全速 + MJPEG 输出，UAC 须兼容 UAC1.0。 |
| **USB 主机 CDC 串口（iot_usbh_cdc）**<br>`repos/esp-iot-solution/recipes/usbh_cdc.md` | 使用 `iot_usbh_cdc` 组件在 ESP32-S2/ESP32-S3/ESP32-P4 等 USB Host 上接入 USB CDC-ACM 串口设备（如 4G 模组、USB 转串口），安装驱动、打开端口、收发数据并处理设备连接/断开事件。驱动提供带环形缓冲与不带缓冲两种模式，以及自定义控制传输接口。 |

### `esp-vision` (18 recipes)

| recipe | 摘要 |
|---|---|
| **用 asyncio 组织多协程视觉应用**<br>`repos/esp-vision/recipes/asyncio_pipeline.md` | 用 MicroPython `asyncio` 把视觉流水线（采集/推理/预览）与遥测、健康监控拆成协作调度的协程，通过复制后的标量状态与锁避免并发访问摄像头/模型/帧缓冲。 |
| **摄像头方向与状态诊断**<br>`repos/esp-vision/recipes/camera_orientation.md` | 用水平镜像 / 垂直翻转校正传感器安装方向，并用 `sensor.status()` 诊断分辨率、像素格式、sensor ID 与就绪状态。 |
| **采集并保存图像**<br>`repos/esp-vision/recipes/camera_snapshot_save.md` | 采集一帧画面并以多种格式（jpg / bmp / ppm）保存到 SD 卡，用 `os.stat` 校验文件大小。 |
| **云端 AI 视觉推理（OpenAI 兼容 vision API）**<br>`repos/esp-vision/recipes/cloud_ai_vision.md` | 采集一帧，JPEG 编码后 base64，通过 HTTPS POST 到 OpenAI 兼容 vision API（如 `gpt-4o-mini`），打印返回的图像描述。补足端侧 ESP-DL 之外的"云大脑"路径。 |
| **颜色色块追踪**<br>`repos/esp-vision/recipes/color_blob_tracking.md` | 用 LAB 阈值配合 `find_blobs` 做 8 连通色块检测，绘制外接框与十字，并用 `pixels_threshold` / `area_threshold` 过滤噪声。 |
| **客制化 ESP-VISION 固件**<br>`repos/esp-vision/recipes/customize_firmware.md` | 在不同配置层移除或增加能力：Python 模块（`micropython.cmake`）、图像算法（`boards/<BOARD>/imlib_config.h`）、标准 MicroPython 功能（`mpconfigboard.h`）、冻结 Python（`manifest.py`）、板级服务与可选组件（`board.cmake`）。 |
| **帧差法运动检测（内存 ImageIO 流）**<br>`repos/esp-vision/recipes/frame_differencing.md` | 用 `image.Image.difference()` 做背景差分检测运动，配合 `binary()` 阈值化与形态学清理得到运动掩码；演示内存 `image.ImageIO` 流做短帧历史缓存。对应 SKILL.md 避坑 #2（`snapshot()` 返回可复用缓冲，跨帧使用必须 `.copy()`）。 |
| **H.264 录制裸码流（仅 ESP32-P4）**<br>`repos/esp-vision/recipes/h264_record.md` | 用 `h264.H264Encoder` 把采集帧逐帧编码为 Annex-B H.264 NAL 单元并写入 SD 卡。`h264` 模块仅在 ESP32-P4 构建中可用。 |
| **首个摄像头脚本**<br>`repos/esp-vision/recipes/hello_camera.md` | 复位摄像头、选择像素格式与分辨率、等待自动曝光/白平衡稳定后连续采集，并通过 `img.flush()` 在主机预览。 |
| **图像分类（ImageNetCls）**<br>`repos/esp-vision/recipes/image_classification.md` | 用 `espdl.ImageNetCls` 对图像做分类，返回最多 `topk` 个 `(label, score)`；以文件图像分类为例，必要时先 `to_rgb565(copy=True)` 转换格式。 |
| **图像绘图原语总览**<br>`repos/esp-vision/recipes/image_drawing.md` | 汇总 `image.Image` 的 `draw_*` 方法（line / rectangle / circle / ellipse / cross / arrow / string / image），覆盖颜色与线宽约定、RGB565 与 GRAYSCALE 的颜色传参差异，以及 `fill` 实心填充。所有方法**原地修改**并返回 `self`，可链式调用。 |
| **图像滤波与二值化**<br>`repos/esp-vision/recipes/image_filters.md` | 用邻域/秩/保边滤波降噪或增强边缘，配合 `binary` 做阈值分割与形态学清理。 |
| **板载 LCD 显示**<br>`repos/esp-vision/recipes/lcd_display.md` | 创建 `display.Display()`（即 `ESP32Display`）并把采集帧用 `write()` 送到屏幕，支持 `fit` 缩放、显式 `x_scale`/`y_scale`、`roi` 裁剪与背光调节。显示对象只创建一次并复用。 |
| **目标检测（ESPDet / YOLO11）**<br>`repos/esp-vision/recipes/object_detection.md` | 用 ESP-DL 的 `ESPDet` / `YOLO11` 封装类在采集帧上做目标检测，绘制检测框，并在运行时调整 `score` / `nms` 阈值。模型只加载一次、跨帧复用。 |
| **姿态估计（YOLO11nPose）**<br>`repos/esp-vision/recipes/pose_estimation.md` | 用 `espdl.YOLO11nPose` 做 COCO 17 关键点姿态估计，绘制检测框、骨架连线与关节点。缺失/低置信度关键点返回为 `(0, 0)`，绘图前需跳过。 |
| **二维码 / 条码 / AprilTag 检测**<br>`repos/esp-vision/recipes/qrcode_barcode_apriltag.md` | 用 `find_qrcodes` / `find_barcodes`（P4 ZXing）/ `find_apriltags` 检测并解码标记，绘制框与角点。灰度输入通常可降低处理开销。 |
| **RTSP 推流（仅 ESP32-P4）**<br>`repos/esp-vision/recipes/rtsp_stream.md` | 起以太网（P4-Function-EV-Board 的 IP101 RMII PHY），用 `h264.H264Encoder` 编码、`rtsp.RTSPServer` 通过 RTSP 推送 H.264，VLC/ffplay 在 `rtsp://<board-ip>:8554/` 观看。`rtsp` 仅 ESP32-P4 构建。 |
| **Wi-Fi MJPEG 视频流推送**<br>`repos/esp-vision/recipes/wifi_mjpeg_stream.md` | 把采集帧逐帧 JPEG 编码，用板载 HTTP 服务器以 `multipart/x-mixed-replace` 推 MJPEG 流，浏览器打开 `http://<board-ip>/` 即可观看。这是无 H.264 硬件（ESP32-S3 等）时唯一的实时视频推流路径。 |

### `esp-who` (9 recipes)

| recipe | 摘要 |
|---|---|
| **摄像头选型与初始化**<br>`repos/esp-who/recipes/camera_selection.md` | 根据 SoC 选择正确的 `WhoCam` 子类（S3 的 `WhoS3Cam` / P4 的 `WhoP4Cam` / USB 的 `WhoUVCCam`），正确传像素格式、帧尺寸、翻转方向与 fb_count。 |
| **定制检测应用（自定义回调与绘制）**<br>`repos/esp-who/recipes/custom_detect_app.md` | 通过继承 `WhoDetectAppLCD` / `WhoDetectAppTerm`，override `detect_result_cb` / `lcd_disp_cb` / `cleanup`，实现自定义的检测结果处理（上报、存盘、自定义画框）。 |
| **自定义帧采集流水线**<br>`repos/esp-who/recipes/frame_cap_pipeline.md` | 理解并定制 ESP-WHO 的摄像头→处理节点链。讲解 `WhoFrameCap` + `WhoFetchNode`/`WhoDecodeNode`/`WhoPPAResizeNode` 的组装方式，以及 `fb_count`/`ringbuf_len` 的取值规则。 |
| **人脸识别（检测 + 注册 + 识别 + 删除）**<br>`repos/esp-who/recipes/human_face_recognition.md` | 使用 `WhoRecognitionAppLCD`（或 `WhoRecognitionAppTerm`）跑完整人脸识别流程：实时检测人脸、按键注册新面孔、识别已注册人脸、删除最后一条特征。 |
| **目标检测（LCD 显示）**<br>`repos/esp-who/recipes/object_detect_lcd.md` | 在带 LCD 的开发板上跑目标检测（人脸/行人/猫/狗），实时画框并显示。基于 `WhoDetectAppLCD`。 |
| **目标检测（串口输出，无 LCD）**<br>`repos/esp-who/recipes/object_detect_term.md` | 没有 LCD 或不想用图形库时，用 `WhoDetectAppTerm` 把每帧检测结果（box / score / keypoint）打印到串口。对应 `*_noglib` BSP。 |
| **构建与烧录 ESP-WHO 示例工程**<br>`repos/esp-who/recipes/project_setup.md` | 从零搭建 ESP-WHO 任一示例（human_face_recognition / object_detect / qrcode_recognition），配置 ESP-IDF 环境、选定 BSP 与目标芯片、编译烧录并查看串口。 |
| **二维码识别**<br>`repos/esp-who/recipes/qrcode_recognition.md` | 使用 `WhoQRCodeAppLCD`（或 `WhoQRCodeAppTerm`）实时识别摄像头画面中的二维码，把解码文本显示到 LCD / 打印到串口。底层是 `quirc` 库。 |
| **任务生命周期控制与自定义任务**<br>`repos/esp-who/recipes/task_lifecycle.md` | 演示 `WhoTask` 的 `run`/`pause`/`resume`/`stop`（同步与异步）用法、事件位机制，以及如何继承 `WhoTask` 写自定义节点/任务挂到流水线或 App 上。 |

### `esp-insights` (10 recipes)

| recipe | 摘要 |
|---|---|
| **Core Dump 捕获与上报**<br>`repos/esp-insights/recipes/core_dump.md` | 让设备在崩溃时把 core dump 写入 flash，并在下次启动时把“摘要”（PC、异常 cause/vaddr、通用寄存器、backtrace）上报到云端。涉及 Kconfig、分区表与 ELF 固件包上传。 |
| **自定义 Metrics（指标）**<br>`repos/esp-insights/recipes/custom_metrics.md` | 注册并上报自定义 metrics（如室温、CPU 负载），用于在仪表盘绘制随时间变化的曲线。覆盖 metadata 1.0 与 2.0 两套 API，以及各数据类型的上报函数。 |
| **自定义 Transport（复用已有 TLS/MQTT 连接）**<br>`repos/esp-insights/recipes/custom_transport.md` | 当默认 HTTPS/MQTT transport 不满足需求（例如想复用应用已有的 TLS/MQTT 连接以节省一次握手内存），通过 `esp_insights_transport_register()` + `esp_insights_enable()` 注入自定义 transport 回调。 |
| **自定义 Variables（变量）**<br>`repos/esp-insights/recipes/custom_variables.md` | 注册并上报自定义 variables（如当前关联的 station 数、���备状态机当前态），区别于 metrics：variables 强调“当前值”而非“随时间序列”。覆盖 1.0 与 2.0 API 及各数据类型。 |
| **数据存储调优（RTC / RAM）与低内存事件**<br>`repos/esp-insights/recipes/data_store_tuning.md` | 调整 ESP-Insights 诊断数据存储（默认 RTC memory）的容量与水位线，监听低内存事件以应对 RTC 满载丢日志。 |
| **HTTPS 传输快速接入**<br>`repos/esp-insights/recipes/https_quickstart.md` | 用最少的代码把 ESP-Insights 通过 HTTPS 接入 ESP Insights 云端，采集错误/告警/事件日志。这是大多数新项目的默认起点。 |
| **日志采集与自定义事件**<br>`repos/esp-insights/recipes/log_capture.md` | 配置 ESP-Insights 采集 error/warning 日志与自定义事件，包括 `log_type` 位掩码、按 tag 调级别、`ESP_DIAG_EVENT` 宏，以及“日志不显示”的诊断思路。 |
| **MQTT(TLS) 传输与 RainMaker Claiming**<br>`repos/esp-insights/recipes/mqtt_transport.md` | 把 ESP-Insights 切换为 MQTT(TLS) 传输，通过 ESP RainMaker Claiming 获取证书并复用 RainMaker 的 MQTT 连接上报诊断数据，节省独立 TLS 会话的内存。 |
| **运行时控制：上报开关、手动发送、Command-Response**<br>`repos/esp-insights/recipes/runtime_control.md` | 在运行时控制 ESP-Insights 的上报行为（暂停/恢复/立即发送），以及启用 RainMaker MQTT 节点的 command-response 远程控制能力。 |
| **系统级 Heap / Wi-Fi Metrics 与 Network Variables**<br>`repos/esp-insights/recipes/system_metrics.md` | 启用 ESP-Insights 内置的系统 metrics（free/最大块/历史最小空闲堆，内部 RAM 与 PSRAM；Wi-Fi RSSI 与历史最小 RSSI）与 network variables，并按需主动 dump。 |

### `esp-claw` (12 recipes)

| recipe | 摘要 |
|---|---|
| **用 Lua 驱动硬件并发布事件**<br>`repos/esp-claw/recipes/automation_lua.md` | 在设备上写 Lua 脚本控制 GPIO / 显示 / LED strip，用 `storage` 安全读写文件，用 `event_publisher` 把结果发回 IM/路由器，用 `capability.call` 在脚本里调 `cap_*` 工具。 |
| **适配新开发板**<br>`repos/esp-claw/recipes/board_adaptation.md` | 为 ESP-Claw `edge_agent` 新增一块开发板：编写 board YAML、`setup_device.c`、板级 sdkconfig 默认与可选 FATFS overlay，并用 `idf.py bmgr` 选中构建。 |
| **从源码构建并烧录 edge_agent**<br>`repos/esp-claw/recipes/build_and_flash.md` | 在本地用 ESP-IDF v5.5.4 + ESP Board Manager 编译 ESP-Claw `edge_agent`，选择开发板、调整 menuconfig、烧录并进入串口 Console。 |
| **配置设备（LLM / IM / 记忆 / NVS 优先级）**<br>`repos/esp-claw/recipes/configuration.md` | 配置 ESP-Claw `edge_agent` 的运行时项——LLM 后端、IM 平台凭据、记忆模式、HTTP/Web Search——理解「menuconfig 默认 vs NVS 覆盖」的优先级，并通过 Web / 串口生效。 |
| **IM 平台端到端集成（事件接入 → 路由 → Agent → 回复 → 附件）**<br>`repos/esp-claw/recipes/im_integration.md` | 打通「IM 入站消息 → Event Router → claw_core → out_message 回送 IM」的完整闭环，理解 `cap_im_platform` 统一源组件、各平台 chat_id 格式与差异、附件异步落盘后的 `attachment_saved` 事件，以及 `router_rules.json` 里真实存在的 7 条 IM 相关规则。 |
| **新增一个 Capability 能力组**<br>`repos/esp-claw/recipes/implement_capability.md` | 从零写一个 `cap_*` 能力组件——定义 descriptor / group、实现 `execute`、在 app 注册、可选附带 Skill，最终成为 LLM/Console/自动化可调用的工具。 |
| **新增 Lua 模块（C 绑定 + 文档 + Skill）**<br>`repos/esp-claw/recipes/lua_module.md` | 创建一个 `lua_module_*` / `lua_driver_*` 组件，把硬件或服务以 Lua API 暴露给设备脚本，并在 app 里注册（必须在 `cap_lua_register_group` 之前）。 |
| **设备作为 MCP 服务端（mcp_server_point 精简应用）**<br>`repos/esp-claw/recipes/mcp_server.md` | 用仓库内置的 `application/mcp_server_point` 把 ESP32 变成一台 **MCP 服务端**，对外（桌面 IDE / 其它 ESP-Claw 设备）通过 SSE + mDNS 暴露 `lua.run_script` / `lua.run_script_async` / async 作业管理等工具，是唯一不依赖完整 `edge_agent` 栈的 ESP-Claw 运行形态——无 `claw_core`、无 `event_router`、无 IM、无 CLI、无记忆，仅 `cap_lua` + `cap_mcp_server`。 |
| **记忆系统使用（长期记忆 + profile）**<br>`repos/esp-claw/recipes/memory_usage.md` | 用 ESP-Claw 的「长期记忆」工具（`memory_store` / `recall` / `list` / `update` / `forget`）和可编辑 profile 三件套（`user.md` / `soul.md` / `identity.md`）让 Agent 跨会话记住事实与人格；理解 full / lightweight 两种模式、summary-tag 轻量检索（无向量库）、自动抽取/去重，以及「`MEMORY.md` 不是检索真相」这一关键陷阱。 |
| **编写 Event Router 自动化规则**<br>`repos/esp-claw/recipes/router_rules.md` | 用 `router_rules.json` 把事件（IM 消息、启动、按钮、定时、附件）路由到 `call_cap` / `run_agent` / `run_script` / `send_message` / `emit_event` / `drop`，并打通 Agent 回复到 IM 的 `out_message` 回路。 |
| **定时任务（scheduler + router rule）**<br>`repos/esp-claw/recipes/scheduled_task.md` | 用 `cap_scheduler` 按时发布 `schedule` 事件，再用 Event Router 规则决定触发后做什么（唤醒 Agent / 发固定 IM / 跑 Lua）。支持 `cron` / `interval` / `once`。 |
| **编写 ESP-Claw Skill**<br>`repos/esp-claw/recipes/write_skill.md` | 按 ESP-Claw Skill 规范创建 `skills/<skill_id>/SKILL.md`（JSON frontmatter + 正文 + 可选 scripts/references/assets），让 LLM 按需激活并获得工具与使用指南。 |

### `esp-amp` (11 recipes)

| recipe | 摘要 |
|---|---|
| **Event 跨核同步**<br>`repos/esp-amp/recipes/event.md` | 使用 ESP-AMP Event（基于 32-bit 原子整数的位图同步机制）实现轻量跨核事件通知。创建事件、绑定 FreeRTOS EventGroup、通知、等待/轮询。适合需要双向或多任务广播同步的场景。 |
| **subcore 生命周期管理**<br>`repos/esp-amp/recipes/lifecycle.md` | 在 maincore 管理 subcore 的完整生命周期：加载固件、启动、停止、检测 panic、自定义 panic 处理、路由 subcore printf 到 maincore 控制台。 |
| **LP subcore 自动 Light Sleep 低功耗**<br>`repos/esp-amp/recipes/light_sleep.md` | 在 ESP32-C5 / ESP32-C6（LP subcore）上启用 HP maincore 自动 light sleep，LP subcore 与 RTC RAM 在睡眠期间保持供电，仅在访问 HP RAM/外设时唤醒 maincore。适合电池供电的低功耗场景。**仅 LP subcore 支持，ESP32-P4 不支持。** |
| **RPC 框架（应用层）**<br>`repos/esp-amp/recipes/rpc.md` | 使用 ESP-AMP RPC（基于 RPMsg 的简单远程过程调用框架）在一核定义 RPC 服务、另一核调用。支持阻塞/非阻塞、有/无响应命令。适合需要"调用对端函数并取回结果"的场景。 |
| **RPMsg 双向通信（传输层）**<br>`repos/esp-amp/recipes/rpmsg.md` | 使用 ESP-AMP RPMsg（Remote Processor Messaging）实现双向端到端通信，基于一对 Virtqueue（TX/RX），支持在单个设备上创建多个端点复用底层队列。支持零拷贝发送。这是最常用的核间数据交换方式。 |
| **Separate Build：分别构建两侧固件**<br>`repos/esp-amp/recipes/separate_build.md` | 使用 ESP-AMP separate build 模式，maincore 与 subcore 各自独立工程、独立构建。适合 subcore 固件需独立开发或由第三方提供的场景。subcore 固件必须手动 esptool 烧入 flash 分区。 |
| **共享内存与 SysInfo**<br>`repos/esp-amp/recipes/shared_memory.md` | 通过 SysInfo 在 maincore 分配共享内存块并分配 ID，subcore 按 ID 查询获取，实现跨核数据共享。这是所有 ESP-AMP IPC 组件的基础。 |
| **软件中断（核间中断）**<br>`repos/esp-amp/recipes/software_interrupt.md` | 使用 ESP-AMP 软件中断（基于 PMU 或 INTMTX 的核间中断）实现异步核间通知。注册处理函数、触发对端核、多 handler 复用同一中断源。这是 Queue/RPMsg 通知机制的基础。 |
| **subcore 外设驱动开发**<br>`repos/esp-amp/recipes/subcore_peripheral.md` | 在 subcore 上开发外设驱动。HP subcore 使用 ESP-IDF hal 组件的 `_ll.h` 低层驱动；LP subcore 使用 IDF ulp 组件已实现的 LP 外设驱动。强调避免双核并发访问同一外设。 |
| **Unified Build：单命令构建 maincore + subcore**<br>`repos/esp-amp/recipes/unified_build.md` | 使用 ESP-AMP unified build 模式，通过一条 `idf.py build` 同时构建 maincore 与 subcore 固件，支持将 subcore 固件嵌入 maincore 或烧入 flash 分区。适合 ESP-AMP 入门与协作开发场景。 |
| **Virtqueue 单向队列（链路层）**<br>`repos/esp-amp/recipes/virtqueue.md` | 直接使用 ESP-AMP Virtqueue（Packed Virtqueue，单生产者-单消费者无锁环形缓冲）实现单向核间数据传输。是 RPMsg 的底层基础。适合需要精细控制缓冲生命周期、或不需要 RPMsg 端点复用的场景。 |

### `esp-lowcode-matter` (16 recipes)

| recipe | 摘要 |
|---|---|
| **按键组件（button_driver）**<br>`repos/esp-lowcode-matter/recipes/button_driver.md` | 用 `button_driver_create` 创建按键，`button_driver_register_cb` 注册单击/长按回调，实现单击切换设备状态、长按触发工厂复位。 |
| **创建与定制新产品**<br>`repos/esp-lowcode-matter/recipes/create_product.md` | 基于模板或最接近的现有产品创建一个新 LowCode 产品，正确组织目录结构、声明组件依赖。 |
| **LP Core 调试（panic 定位与日志）**<br>`repos/esp-lowcode-matter/recipes/debugging.md` | 识别 LP Core 的 Breakpoint panic（空指针）与 Illegal Instruction panic（缓冲/栈溢出），用 `addr2line` + MEPC 定位代码行，依靠 `printf` 日志与编码实践排查。 |
| **系统事件处理与事件上报**<br>`repos/esp-lowcode-matter/recipes/event_handling.md` | 在 `app_driver_event_handler()` 里 switch 处理 `LOW_CODE_EVENT_*` 系统事件（配网/网络/OTA/就绪/识别），并用 `low_code_event_to_system()` 主动上报事件（如工厂复位）。 |
| **特性（Feature）的接收与上报**<br>`repos/esp-lowcode-matter/recipes/feature_update.md` | 实现 `feature_update_from_system()` 接收系统下发的特性更新，并用 `low_code_feature_update_to_system()` 把设备状态（按键/传感器）主动上报，支持 `feature_id` 与 matter 低层标识两种路由方式。 |
| **本地终端环境搭建与首次烧录**<br>`repos/esp-lowcode-matter/recipes/getting_started.md` | 在本地终端用 ESP-IDF v5.3 + ESP-AMP + esp-lowcode-matter 搭建开发环境，完成 Prepare Device、Upload Configuration、Upload Code 全流程，把一个示例产品跑起来。 |
| **LP Core GPIO（system_* API）与 Arduino 映射**<br>`repos/esp-lowcode-matter/recipes/gpio_system.md` | 在 LP Core 上用 `system_set_pin_mode` / `system_digital_write` / `system_digital_read` 操作 GPIO，并提供 Arduino→LowCode 函数映射。 |
| **LD2420 雷达占用传感器（UART）**<br>`repos/esp-lowcode-matter/recipes/ld2420_occupancy.md` | 用 `occupancy_sensor_ld2420_init` 初始化 LD2420，进入 normal/report 模式，周期读取占用状态与距离，以 `LOW_CODE_FEATURE_ID_OCCUPANCY_SENSOR_VALUE` 上报。 |
| **灯光组件（light_driver：LED/PWM 与 WS2812）**<br>`repos/esp-lowcode-matter/recipes/light_driver.md` | 用 `light_driver_init` 配置 LED(PWM) 或 WS2812，设置通断/亮度/色温/色调/饱和度，并用 blink/breathe 特效做配网指示。 |
| **产品配置与数据模型定制**<br>`repos/esp-lowcode-matter/recipes/product_configuration.md` | 编辑 `product_info.json`（vendor/product/chip/connection 等）与 `data_model.zap`（endpoint/cluster/attribute），改后重跑 Upload Configuration 生成并烧录 `data_model.bin`。 |
| **继电器组件与智能插座（单/双通道）**<br>`repos/esp-lowcode-matter/recipes/relay_socket.md` | 用 `relay_driver_init` / `relay_driver_set_power` 控制继电器，实现单通道（`products/socket`）与多 endpoint 双通道（`products/socket_2_channel`）智能插座。 |
| **setup/loop 编程骨架**<br>`repos/esp-lowcode-matter/recipes/setup_loop_model.md` | 理解并实现 LowCode 在 LP Core 上的 `system_setup()` → `setup()` → `while{system_loop();loop();}` 骨架与回调注册。 |
| **SHT30 温度传感器（I2C）周期上报**<br>`repos/esp-lowcode-matter/recipes/sht30_sensor.md` | 用 I2C 初始化 + `temperature_sensor_sht30_init` + `temperature_sensor_sht30_get_celsius` 周期读取温度，并通过 `LOW_CODE_FEATURE_ID_TEMPERATURE_SENSOR_VALUE` 上报到 Matter。 |
| **SSD1306/SSD1315 OLED 显示（I2C）**<br>`repos/esp-lowcode-matter/recipes/ssd1306_display.md` | 用 `display_ssd1306_i2c_create` 创建 OLED 句柄，用 `display_ssd1306_draw_string` / `display_ssd1306_refresh_gram` / `display_ssd1306_clear_screen` 显示文本，结合事件回调显示配网状态。 |
| **system_timer 软件定时器（周期上报模式）**<br>`repos/esp-lowcode-matter/recipes/system_timer.md` | 用 `system_timer_create` / `system_timer_start` 创建周期或单次定时器，在回调里周期读取并上报特性（传感器读数的标准模式）。 |
| **温控器产品（单 endpoint 多特性：温度 + 制冷/制热设定点）**<br>`repos/esp-lowcode-matter/recipes/thermostat.md` | 基于 `products/thermostat` 实现温控器（MA-thermostat，device_type_id 769）——在**单个 endpoint** 上同时处理三个 feature_id：温度（`LOW_CODE_FEATURE_ID_TEMPERATURE`）、制冷设定点（`LOW_CODE_FEATURE_ID_COOLING_SETPOINT`）、制热设定点（`LOW_CODE_FEATURE_ID_HEATING_SETPOINT`），均使用带符号 `int16_t`（°C×100）。 |

### `esp-agents-firmware` (12 recipes)

| recipe | 摘要 |
|---|---|
| **添加自定义板级配置**<br>`repos/esp-agents-firmware/recipes/add_custom_board.md` | 在 `examples/common/boards/` 下新增一块自定义板的配置，使其能被 `idf.py select-board` 识别并参与构建。 |
| **初始化 Agent 与处理事件**<br>`repos/esp-agents-firmware/recipes/agent_init_and_events.md` | 初始化 Agent、注册事件回调、处理文本/语音/思考/错误事件，理解 `app_agent_*` 封装与底层 `esp_agent_*` 的关系。 |
| **音频管线（录音 / 播放）**<br>`repos/esp-agents-firmware/recipes/audio_pipeline.md` | 理解 `components/audio` 与 `app_audio` 封装，配置上下行采样率/帧长/AEC/音量，把 Agent 下行语音送入扬声器。 |
| **构建与烧录示例固件**<br>`repos/esp-agents-firmware/recipes/build_and_flash.md` | 基于 `esp-agents-firmware` 构建并烧录 voice_chat 或 matter_controller 示例到支持的板子，查看串口日志。 |
| **使用内置工具**<br>`repos/esp-agents-firmware/recipes/builtin_tools.md` | 使用仓库自带的内置本地工具 `set_reminder` / `get_local_time` / `set_volume` / `set_emotion`，理解其参数、行为与注册方式。 |
| **串口 Console 命令**<br>`repos/esp-agents-firmware/recipes/console_commands.md` | 使用 `agent_console` 组件注册与管理串口命令，复用默认命令（set-token/set-agent/set-wifi/cpu-dump/mem-dump/reboot/reset-to-factory 等）。 |
| **切换到自定义 ESP Private Agents 部署**<br>`repos/esp-agents-firmware/recipes/custom_deployment.md` | 把固件从公共部署（`api.agents.espressif.com`）切换到自建 AWS 部署，并配置 token 与 agent_id。 |
| **设备配网与首次设置**<br>`repos/esp-agents-firmware/recipes/device_setup_provisioning.md` | 设备首次配网（Wi-Fi、agent_id、refresh_token）、Agent 启动条件、恢复出厂，覆盖 voice_chat 与 matter_controller 两种 App 路径。 |
| **无显示屏设备适配**<br>`repos/esp-agents-firmware/recipes/headless_no_display.md` | 在没有 LCD 的板子（如裸 ESP-VoCat 不带屏）上运行 voice_chat / matter_controller 固件，按 `docs/example_customisation.md` 注释掉 `app_display_init()` 与设备回调里的 `app_display_*` 调用，配网二维码改由产品包装提供。 |
| **注册与实现本地工具（Local Tool）**<br>`repos/esp-agents-firmware/recipes/local_tool_register.md` | 通过 `esp_agent_register_local_tool` 注册设备端本地工具，实现 `esp_agent_tool_handler_t` 回调，并同步 Agent 配置。 |
| **Matter 设备控制（controller）**<br>`repos/esp-agents-firmware/recipes/matter_controller_control.md` | 在 matter_controller 示例中获取 Matter 设备列表、控制 OnOff / LevelControl / ColorControl cluster，以及 Thread Border Router 的板子选择与配置。 |
| **Matter Controller 服务初始化与 RainMaker 联动**<br>`repos/esp-agents-firmware/recipes/matter_controller_service_init.md` | `esp.service.matter-controller`（MatterCTL）RainMaker 服务的 6 个参数、7 位 MTCtlStatus 状态位图、控制器 NOC 签发与设备列表更新的手机 App 触发流程，以及固件侧 `matter_controller_enable` / `IP_EVENT_STA_GOT_IP` / Thread Border Router / `matter_controller_client` 的初始化链。 |

## 协议/连接 SDK


### `esp-matter` (17 recipes)

| recipe | 摘要 |
|---|---|
| **Bridge 设备（Zigbee / BLE Mesh / ESP-NOW 桥接）**<br>`repos/esp-matter/recipes/bridge_zigbee.md` | 用 Aggregator + 动态 bridged_node endpoint 把非 Matter 设备（Zigbee、BLE Mesh、ESP-NOW、RainMaker）桥接进 Matter 网络。说明 aggregator 创建、动态 endpoint 恢复、以及 `app_bridge_initialize` 回调机制。 |
| **用 chip-tool 入网与控制**<br>`repos/esp-matter/recipes/commissioning_chiptool.md` | 用 host 端 chip-tool 作为 commissioner，通过 BLE-Wi-Fi 或 BLE-Thread 把 Matter 设备加入 fabric，并用 cluster 命令读写属性。 |
| **设备端 Matter Controller / Commissioner（ESP32 做控制器）**<br>`repos/esp-matter/recipes/controller_ondevice.md` | 把 ESP32-S3 本身做成 Matter 控制器 / commissioner，在固件里发起 pairing、`invoke-cmd`、`read-attr` / `read-event`、`write-attr`、`subs-attr` / `subs-event` 与 group-settings 操作。与 `commissioning_chiptool.md`（host 端 chip-tool 做 commissioner）互补：本 recipe 适用于"没有外部 chip-tool、由 ESP32 直接入网并控制其它 Matter 设备"的场景。 |
| **自定义 Cluster（厂商扩展）**<br>`repos/esp-matter/recipes/custom_cluster.md` | 在 Matter 设备上加一个厂商自定义 cluster —— 编写 cluster XML 模板、用 `zap_regen_all.py` 生成 app-common 代码、实现属性/命令回调、用 esp-matter 低层 API 把它挂到 endpoint。 |
| **设备控制台（matter shell）**<br>`repos/esp-matter/recipes/device_console.md` | 使用 esp-matter 设备端 `matter` shell 命令进行 BLE/Wi-Fi 控制、属性读写、factory reset、桥接设备增删。需 `CONFIG_ENABLE_CHIP_SHELL=y`（示例默认开启）。 |
| **数据模型：Node / Endpoint / Cluster / Attribute / Command**<br>`repos/esp-matter/recipes/device_data_model.md` | 用 esp-matter 的命名空间 API 从零搭出一个设备数据模型 —— 创建 node、标准 device type endpoint、为 endpoint 追加 cluster、为 cluster 追加 attribute 与 command，并把 `priv_data` 传给回调。 |
| **门锁与窗帘（Door Lock / Window Covering）**<br>`repos/esp-matter/recipes/door_lock_window_covering.md` | 创建 Door Lock 与 Window Covering 两种 device type endpoint，重点说明 Window Covering `config_t` 构造函数如何指定 `EndProductType`，以及 Door Lock 常用属性。 |
| **工厂分区与认证凭据（esp-matter-mfg-tool）**<br>`repos/esp-matter/recipes/factory_data_attestation.md` | 用 `esp-matter-mfg-tool` 生成含 VID/PID/CD/DAC/passcode/discriminator 的工厂分区二进制，烧录到 fctry 分区，并在 menuconfig 选择对应的 Factory/Secure Cert Provider，让设备用真实（或测试）凭据入网。 |
| **暖通 / 家电设备类型（Thermostat / Refrigerator / Room AC）**<br>`repos/esp-matter/recipes/hvac_appliances.md` | 用 esp-matter 标准 device type 创建暖通与白电 endpoint —— Thermostat（带 heating/cooling feature conformance）、Refrigerator + TemperatureControlledCabinet（父子 endpoint）、Room Air Conditioner（OnOff + Thermostat 组合）。涉及 cluster 级 feature flag 设置、父子 endpoint 关联、以及需要 delegate 的 cluster（TemperatureControl）。 |
| **ICD 间歇连接设备（Short / Long Idle Time）**<br>`repos/esp-matter/recipes/icd_device.md` | 用 `CONFIG_ENABLE_ICD_SERVER=y` 把电池供电 / sleepy 的 ESP32-H2 / ESP32-C6 做成 Matter ICD（Intermittently Connected Device），按 SIT（Short Idle Time）或 LIT（Long Idle Time）配置 polling / idle / active 参数，并配合 power management、IEEE 802.15.4 sleep、tickless idle 真正进入低功耗。 |
| **灯具设备（On/Off / Dimmable / Color Temperature / Extended Color）**<br>`repos/esp-matter/recipes/lighting.md` | 用 esp-matter 标准 device type 创建各类灯具 endpoint，绑定 LED 驱动，在 `app_attribute_update_cb` 里做 Matter 单位到 LED 单位的���映射，并用 `attribute::update()` 让按键反向写回数据模型。 |
| **Matter OTA Requestor（含加密 OTA）**<br>`repos/esp-matter/recipes/matter_ota.md` | 启用 Matter OTA Requestor，让设备能向 OTA Provider 拉取并验证 Matter OTA 镜像；进一步启用加密 OTA，用 RSA-3072 私钥解密应用镜像。 |
| **RAM 与 Flash 优化**<br>`repos/esp-matter/recipes/optimizations.md` | Matter 固件体积大、内存吃紧（尤其 esp32c2 / esp32h2）时，按收益从高到低逐项打开 Kconfig 优化项，并用 measured before/after 表预估节省量。所有数字取自 `docs/en/optimizations.rst`（基于 esp32c3 / esp32h2 + light 示例）。 |
| **传感器设备（温度 / 湿度 / 占用 / 接触）**<br>`repos/esp-matter/recipes/sensors.md` | 在一个 Matter 节点上创建多个传感器 endpoint（温度、湿度、占用），从传感器驱动异步拿到数据后，用 `chip::DeviceLayer::SystemLayer().ScheduleLambda(...)` 切到 Matter 线程，再 `attribute::update()` 上报。 |
| **环境搭建与首次构建**<br>`repos/esp-matter/recipes/setup_and_build.md` | 克隆 esp-matter 及 connectedhomeip 子模块，配置 ESP-IDF，设置目标芯片，构建并烧录 `light` 示例，确认设备能启动并打印 commissioning 广播日志。 |
| **开关设备与 Binding（On/Off Light Switch）**<br>`repos/esp-matter/recipes/switches_binding.md` | 创建一个 On/Off Light Switch（client 端 OnOff），用 Binding cluster 把它绑定到远端灯，绑定后通过按键发送 OnOff 命令控制远端灯，并可选订阅灯的状态以同步本机指示灯。 |
| **Thread Border Router（ESP32-S3 + ESP32-H2）**<br>`repos/esp-matter/recipes/thread_border_router.md` | 用 ESP Thread Border Router 板（ESP32-S3 主控 + ESP32-H2 作 15.4 RCP）搭一个 Matter Thread Border Router：烧 RCP 固件到 H2、烧 BR 固件到 S3、commission BR 后用 ThreadBorderRouterManagement cluster 配置 Thread 网络，再 commission Thread 终端设备入网。 |

### `esp-zigbee-sdk` (12 recipes)

| recipe | 摘要 |
|---|---|
| **协调器（ZC）on/off 灯：网络形成与开放入网**<br>`repos/esp-zigbee-sdk/recipes/coordinator_light.md` | 实现一个 Zigbee Coordinator（ZC）角色的 HA on/off 灯，完成网络形成（FORMATION）、开放网络（open network）与入网引导（STEERING），并通过 ZCL Core Action 接收 on/off 属性变化驱动 LED。 |
| **自定义 Cluster（custom cluster）**<br>`repos/esp-zigbee-sdk/recipes/custom_cluster.md` | 讲解如何创建厂商/私有 cluster——注册自定义命令处理回调（`ezb_zcl_custom_cluster_handlers_t`）、定义私有 cluster ID / 命令 ID / 属性、发送与接收自定义命令。基于 `examples/customized_devices/` 的 data stream（producer/consumer）模式。 |
| **深度休眠终端（Deep Sleep End Device）**<br>`repos/esp-zigbee-sdk/recipes/deep_sleep_end_device.md` | 实现 Zigbee End Device 的 deep sleep 最低功耗模式：每次唤醒都从 reset 重启、重新初始化协议栈并 rejoin 网络，用 RTC timer + EXT1（BOOT 按键）双唤醒源，`RTC_DATA_ATTR` 跨睡眠保存时间戳。与 light sleep（Zigbee 任务常驻、keep-alive 自动维持）是两套完全不同的流程。 |
| **Zigbee OTA 升级（ota_server 下发 / ota_client 刷写）**<br>`repos/esp-zigbee-sdk/recipes/ota_upgrade.md` | 讲解 Zigbee OTA 升级流程——ota_server 作为升级文件提供方（ZC 侧），ota_client 作为接收方接收分块镜像并刷写到 OTA 分区，支持整包与 delta OTA。 |
| **路由器/终端（ZR/ZED）开关：入网 + ZDO 发现与绑定**<br>`repos/esp-zigbee-sdk/recipes/router_switch.md` | 实现一个 HA on/off switch（ZR 或 ZED），通过 STEERING 加入协调器网络，用 ZDO `Match_Desc_req` 发现远端灯，再用 `Bind_req` 绑定，绑定后用 `ezb_zcl_on_off_toggle_cmd_req` 一键控制（无需指定目的地址）。 |
| **休眠终端（Sleepy End Device）：Light Sleep**<br>`repos/esp-zigbee-sdk/recipes/sleepy_end_device.md` | 实现 Zigbee End Device 的 light sleep 低功耗模式：关闭 RxOnWhenIdle、配置 `esp_pm`、用 EXT1 唤醒，按键触发 ZCL 命令并保持与父节点的 keep-alive。 |
| **Touchlink Commissioning（initiator 与 target）**<br>`repos/esp-zigbee-sdk/recipes/touchlink.md` | 讲解 BDB Touchlink commissioning——initiator 主动扫描并拉起 target 入网，target 等待被发起。两者均通过 `ezb_bdb_start_top_level_commissioning` 配合对应模式与信号完成。 |
| **ZCL 属性写本地 + 主动上报（Report Attribute）**<br>`repos/esp-zigbee-sdk/recipes/zcl_attribute_report.md` | 讲解两类属性操作——服务端用 `ezb_zcl_set_attr_value` 更新本地属性值（如温度传感器刷新读数），以及已绑定的客户端用 `ezb_zcl_report_attr_cmd_req` 主动上报当前属性给绑定目的端。 |
| **发送 ZCL 命令（on/off、level、color、通用 read/write/report）**<br>`repos/esp-zigbee-sdk/recipes/zcl_command_send.md` | 汇总常用 ZCL 命令的发送方法——on/off toggle/on/off、level move-to-level、color move-to-hue-and-saturation，以及通用 read/write/config-report 命令的请求结构与调用。 |
| **接收属性写入：ZCL Core Action 回调**<br>`repos/esp-zigbee-sdk/recipes/zcl_core_action.md` | 讲解服务端如何接收并处理来自远端的属性写入/命令——通过 `ezb_zcl_core_action_handler_register` 注册回调，在 `EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID` 分支里取 `ezb_zcl_set_attr_value_message_t` 驱动外设（如点灯、关窗帘）。 |
| **构建 ZHA 数据模型：device / endpoint / cluster / Basic 属性**<br>`repos/esp-zigbee-sdk/recipes/zha_device_model.md` | 讲解 ESP Zigbee SDK v2.x 的 ZHA 数据模型构建流程——device descriptor → endpoint descriptor（由 `ezb_zha_create_*` 一次性创建含 cluster）→ 补充 Basic 厂商/型号属性 → 注册到协议栈。这是所有 ZHA 设备共用的骨架。 |
| **Zigbee 网关与 RCP（Radio Co-Processor）**<br>`repos/esp-zigbee-sdk/recipes/zigbee_gateway_rcp.md` | 在没有 802.15.4 radio 的主芯片（ESP32-C3/S3/P4）上，通过 UART 连接一块 ESP32-H2/C6（烧录 ot_rcp 固件作为 Radio Co-Processor）构建 Zigbee 网关，可选 Wi-Fi/以太网回程、软件共存与 RCP 自动升级。 |

### `esp-thread-br` (14 recipes)

| recipe | 摘要 |
|---|---|
| **自动起网模式 (AUTO_START)**<br>`repos/esp-thread-br/recipes/auto_start_mode.md` | 启用 `OPENTHREAD_BR_AUTO_START`，设备开机自动连 Wi-Fi、生成 Thread dataset、成为 Leader，并提供 Ethernet backbone 的唯一可行路径。 |
| **双向 IPv6 连通**<br>`repos/esp-thread-br/recipes/bidirectional_ipv6.md` | 让 Wi-Fi/Ethernet 主机与 Thread 设备互相用全局 IPv6 地址 ping 通；包含 Linux 主机 RA 接收配置。 |
| **板型与通信接口配置**<br>`repos/esp-thread-br/recipes/board_and_interface.md` | 选择板型 (DEV_KIT / STANDALONE / M5STACK_CORES3)、配置 UART/SPI 通信接口与 RCP Reset/Boot 引脚、Standalone 模组接线。 |
| **构建与烧录 Thread Border Router**<br>`repos/esp-thread-br/recipes/build_and_run.md` | 从零拉取 esp-thread-br、构建 RCP 镜像、配置并烧录 `basic_thread_border_router`，使设备连上 Wi-Fi 并形成 Thread 网络。 |
| **Credential Sharing / ePSKc Commissioner**<br>`repos/esp-thread-br/recipes/credential_sharing.md` | 演示 Thread 1.4 Credential Sharing：在 BR 上生成临时密钥 ePSKc，通告 `meshcop-e` 服务，使一个 Thread Commissioner 经 DTLS 安全会话从 BR 检索或配置 Thread 网络凭据（Network Key / PSKd）。 |
| **DHCPv6 Prefix Delegation (PD) 客户端**<br>`repos/esp-thread-br/recipes/dhcpv6_pd.md` | 让 BR 作为 DHCPv6 PD 客户端，从网络中的 DHCPv6 服务器（如 Kea）申请一段 IPv6 前缀，并下发给 Thread 设备，使其获得可全局路由的 IPv6 地址。 |
| **HTTPS OTA 升级 Border Router**<br>`repos/esp-thread-br/recipes/http_ota.md` | 用本地 openssl HTTPS 服务器下发 `ota_with_rcp_image`，触发 BR 自身 OTA（必要时连带 RCP）。 |
| **组播转发 (Multicast Forwarding)**<br>`repos/esp-thread-br/recipes/multicast_forwarding.md` | 让 Wi-Fi/Ethernet 与 Thread 网络中处于同一组播组的设备相互可达（ICMP/UDP）。 |
| **NAT64 / DNS64 访问 IPv4 互联网**<br>`repos/esp-thread-br/recipes/nat64.md` | 让 Thread 设备经 BR 的 NAT64 访问 IPv4 互联网（如 `curl http://www.espressif.com`）。 |
| **RCP 更新机制**<br>`repos/esp-thread-br/recipes/rcp_update.md` | 让主控 SoC 自动/手动更新 RCP（ESP32-H2/C6）固件；理解序列号、verified flag 与自动回滚。 |
| **RF External Coexistence（Wi-Fi ↔ 802.15.4）**<br>`repos/esp-thread-br/recipes/rf_coexistence.md` | 在 ESP32-S3（Wi-Fi）与 ESP32-H2（802.15.4 RCP）双芯片 BR 上启用外部 RF 共存，用 3 线或 4 线握手信号降低同频段干扰；重点说明何时有用、3/4 线差异、以及两端 Kconfig 必须同时开启。 |
| **服务发现 (SRP + mDNS)**<br>`repos/esp-thread-br/recipes/service_discovery.md` | Thread 设备经 SRP 注册服务，BR 通过 mDNS 转发到 Wi-Fi；反之 Wi-Fi mDNS 服务也能被 Thread 设备经 DNS 解析。 |
| **TREL (Thread Radio Encapsulation Link)**<br>`repos/esp-thread-br/recipes/trel.md` | 让具备 Wi-Fi 但无 802.15.4 的设备（如 ESP32-S3）通过 TREL 经 Wi-Fi 直接参与 Thread 网络，与 Thread CLI 设备互通。 |
| **Web GUI 与 REST API（含 Home Assistant）**<br>`repos/esp-thread-br/recipes/web_gui.md` | 启用 BR Web Server，通过浏览器图形界面发现/组网/查状态，访问 REST API，并接入 Home Assistant。 |

### `connectedhomeip` (14 recipes)

| recipe | 摘要 |
|---|---|
| **构建 ESP32/ESP32-C3 CHIP 示例**<br>`repos/connectedhomeip/recipes/build_esp32_example.md` | 使用 ESP-IDF v4.3 构建、烧录并监视一个 CHIP（Matter）设备示例（以 lock-app / all-clusters-app 为例），覆盖环境准备、目标设置、menuconfig、烧录与日志确认。 |
| **设备端 CHIP Shell 调试（CLI bring-up 与功能测试）**<br>`repos/connectedhomeip/recipes/chip_shell_debug.md` | 启用并使用设备端 CHIP Shell（`chip::LaunchShell()`），通过串口交互式检查设备配置（vendorid / productid / discriminator / pincode / fabricid）、验证配网凭据、跑功能测试。这是仓库文档化的主要交互式 bring-up 工具，shell/esp32 是专用示例。当无法用 chip-tool 配网时，shell 是定位"凭据对不对 / 栈启没起"的第一手段。 |
| **使用 chip-tool CLI 配对并下发 ZCL 命令**<br>`repos/connectedhomeip/recipes/chip_tool_cli.md` | 用 CHIP 自带的命令行客户端 `chip-tool` 配对设备（BLE/Bypass）并列出/调用支持的集群命令。适合快速功能验证与命令速查。 |
| **通过 BLE 完成 Rendezvous 配网**<br>`repos/connectedhomeip/recipes/commissioning_ble.md` | 设备烧录后，在默认 BLE Rendezvous 模式下，使用 chip-tool 或 python controller 完成 PASE 配对、下发 Wi-Fi 凭据、关闭 BLE 并解析 mDNS，使设备加入网络。这是示例的默认配网路径。 |
| **通过 Wi-Fi / Bypass 模式配网**<br>`repos/connectedhomeip/recipes/commissioning_wifi_bypass.md` | 当不使用 BLE 时，将设备配网模式切换为 Wi-Fi 或 Bypass：在 menuconfig 设置 SSID/密码与 Rendezvous 模式，使设备开机即尝试联网（Bypass 跳过安全配对，用于调试）。 |
| **实现自定义属性回调并响应集群写入**<br>`repos/connectedhomeip/recipes/custom_attribute_callback.md` | 扩展 `CHIPDeviceManagerCallbacks::PostAttributeChangeCallback`，使其能处理多个集群/属性；并演示如何用 `VerifyOrExit` 安全过滤、用 `EmberAfStatus` 回写服务端属性。 |
| **处理 CHIP 设备事件（网络 / IP / 会话）**<br>`repos/connectedhomeip/recipes/device_event_handling.md` | 在 `CHIPDeviceManagerCallbacks::DeviceEventCallback` 中处理 `ChipDeviceEvent`：网络连接变化、IP 地址变化（需重启 mDNS）、安全会话建立等关键事件，确保设备配网后可被 commissioner 发现与控制。 |
| **门锁集群服务端集成（DoorLock server：LockState / PIN / RFID）**<br>`repos/connectedhomeip/recipes/door_lock_cluster.md` | 集成真正的 `DoorLock` 集群（`ZCL_DOOR_LOCK_CLUSTER_ID = 0x0101`）服务端，而不是把锁逻辑挂到 OnOff→GPIO。涵盖 `LockState` 属性写入（`EMBER_ZCL_DOOR_LOCK_STATE_LOCKED/UNLOCKED`）、`emberAfPluginDoorLockServerActivateDoorLockCallback` 动作回调、PIN/RFID 校验（`ApplyPin/ApplyRfid`）、用户表与日志。这是真实 Matter 门锁设备类型的入口；lock-app/esp32 里的 OnOff→继电器映射只是简化版。 |
| **调光 / Level Control 集群服务端（current-level 写入与硬件同步）**<br>`repos/connectedhomeip/recipes/level_control_cluster.md` | 集成 `Level Control` 集群（`ZCL_LEVEL_CONTROL_CLUSTER_ID = 0x0008`）服务端，实现亮度/级别控制。涵盖 `CurrentLevel` 属性初始化（`ZCL_CURRENT_LEVEL_ATTRIBUTE_ID = 0x0000`，`uint8_t` 0–255）、启动时硬件状态同步钩子 `emberAfPluginLevelControlClusterServerPostInitCallback`，以及 chip-tool 的完整调光命令集。本仓库无 `lighting-app/esp32`，因此 `all-clusters-app` 是唯一的 ESP32 参考实现。 |
| **创建自定义 ESP32 Matter 设备示例**<br>`repos/connectedhomeip/recipes/new_esp32_example.md` | 以 `examples/lock-app/esp32` 为模板，派生一个自定义 CHIP 设备应用：复制目录、调整 `app_main` 初始化顺序、注册回调、设置 GPIO 与 Kconfig，最小改动获得可编译可配网的固件。 |
| **将 OnOff 集群属性绑定到 GPIO（LED / 继电器）**<br>`repos/connectedhomeip/recipes/onoff_cluster_hardware.md` | 把 Matter OnOff 集群（`ZCL_ON_OFF_CLUSTER_ID`）的属性写入事件映射到物理 GPIO（LED 或继电器），并在本地状态变化时回写服务端属性。这是 lock-app 与 all-clusters-app 的核心模式。 |
| **Pigweed Echo RPC 远程调试通道（UART RPC console）**<br>`repos/connectedhomeip/recipes/pigweed_rpc_debug.md` | 构建并使用 `pigweed-app/esp32` 的 Echo RPC 服务端，通过 UART 在主机用 `pw_hdlc.rpc_console` 远程调用设备上的 `EchoService.Echo(msg=...)`。这是仓库文档化的主机 ↔ 设备 RPC 通道，能力超出普通串口日志（可双向触发设备动作、结构化往返）。当需要比 `ESP_LOGI` 更强的交互式调试时使用。 |
| **使用 python controller 配网与集群控制**<br>`repos/connectedhomeip/recipes/python_controller.md` | 用 CHIP 自带的 python `chip-device-ctrl` 完成 BLE 扫描、配对、网络下发、DNS-SD 解析及 ZCL 集群命令下发。适合脚本化测试与自动化配网验证。 |
| **上报温度传感器集群测量值（TemperatureMeasurement server）**<br>`repos/connectedhomeip/recipes/temperature_measurement_cluster.md` | 把一个传感器读数（温度）发布到 Matter `TemperatureMeasurement` 集群（`ZCL_TEMP_MEASUREMENT_CLUSTER_ID = 0x0402`）。核心调用是 `emberAfTemperatureMeasurementClusterSetMeasuredValueCallback(endpoint, int16_t)`，其中 `int16_t` 值为摄氏度 × 100。这是 sensor 类设备类型的规范上报模式（与 actuator-only 的 OnOff → GPIO 模式相对）。 |

### `esp-now` (12 recipes)

| recipe | 摘要 |
|---|---|
| **硬币电池低功耗开关（Light Sleep + 控制）**<br>`repos/esp-now/recipes/coin_cell_switch.md` | 在硬币电池供电的开关上，利用 light sleep 给电容充电、power lock 保持供电、状态持久化，并按需发送控制/绑定/解绑帧（参考 `examples/coin_cell_demo/switch`）。 |
| **设备控制：Initiator（开关/传感器）侧**<br>`repos/esp-now/recipes/control_initiator.md` | 在 initiator 设备上通过按键触发绑定、解绑与控制数据发送，控制 responder（灯/插座）动作（参考 `examples/control`）。 |
| **设备控制：Responder（灯/插座）侧**<br>`repos/esp-now/recipes/control_responder.md` | 在 responder 设备上进入绑定窗口、注册控制数据回调、维护绑定列表（持久化到 NVS），并依据控制数据执行动作（参考 `examples/control` 与 `examples/coin_cell_demo/bulb`）。 |
| **ESP-NOW 入门：广播发送与回调接收**<br>`repos/esp-now/recipes/get_started_send_recv.md` | 在 ESP-IDF 工程中集成 ESP-NOW 组件，完成 storage/Wi-Fi/espnow 初始化，实现广播发送用户数据并通过回调接收（参考 `examples/get-started`）。 |
| **批量固件 OTA 升级**<br>`repos/esp-now/recipes/ota_batch_upgrade.md` | initiator 从 HTTP 下载固件并写入本地升级分区，再扫描 responder、分发包并支持断点续传；responder 启动升级接收并写 flash（参考 `examples/ota`）。 |
| **ESP-NOW Wi-Fi 配网（Provisioning）**<br>`repos/esp-now/recipes/provisioning_wifi.md` | 已联网的 responder 广播配网 beacon 并在回调校验 initiator 后下发 Wi-Fi 配置；未联网的 initiator 扫描 beacon、发送身份请求并应用收到的 SSID/密码（参考 `examples/provisioning`）。 |
| **安全握手与加解密收发**<br>`repos/esp-now/recipes/security_handshake.md` | initiator 扫描 responder、用 ECDH+PoP 完成握手并分发 app key，双方基于 AES-CCM 加解密 ESP-NOW 用户数据（参考 `examples/security`）。 |
| **多特性集成固件（综合方案）**<br>`repos/esp-now/recipes/solution_integrated_firmware.md` | 用条件编译在单个二进制中同时集成 Wi-Fi 配网（BLE/SoftAP）、ESP-NOW 配网、设备控制、无线调试、批量 OTA、安全握手与时间同步；通过 `CONFIG_APP_ESPNOW_INITIATOR`/`RESPONDER` 切换角色，单按键复用配网/绑定/控制/复位，共享 LED 表达状态（参考 `examples/solution`）。 |
| **存储 / 内存 / 重启工具（espnow_storage / espnow_mem / espnow_utils）**<br>`repos/esp-now/recipes/storage_utils.md` | 使用组件封装的 NVS 存储接口、带调试记录的内存宏、以及重启计数/异常判定等工具函数（基于 `src/utils/include/` 三个头文件）。 |
| **节点间时间同步（无需联网）**<br>`repos/esp-now/recipes/time_sync.md` | initiator 广播权威时间，responder 接收并调整本地时间；适合从 deep sleep 唤醒的节点同步时间（基于 `espnow_time.h`）。 |
| **单播（peer）与分组控制**<br>`repos/esp-now/recipes/unicast_and_group.md` | 使用 `espnow_add_peer` 实现单播定向发送，使用 `espnow_add_group` / `espnow_set_group` 动态管理设备分组与组播（基于 `espnow.h` 真实 API）。 |
| **无线调试：日志、Console 与命令**<br>`repos/esp-now/recipes/wireless_debug.md` | 通过 ESP-NOW 远程抓取设备日志（按等级 UART/flash/ESPNOW/custom 分流）、下发调试命令、运行 console（参考 `examples/wireless_debug` 与 debug 模块头文件）。 |

### `esp-hosted-mcu` (16 recipes)

| recipe | 摘要 |
|---|---|
| **SDIO（1-Bit / 4-Bit）通信链路搭建**<br>`repos/esp-hosted-mcu/recipes/bringup_sdio.md` | 使用 SDIO 作为 Host 与 Co-processor 之间的高性能传输介质。SDIO 4-Bit 是 ESP-Hosted 吞吐最高的传输方式（shield-box 实测 UDP ~79.5 / TCP ~53.4 Mbits/s），但信号完整性要求严格，必须用 PCB 并加外部上拉。 |
| **SPI 全双工（Full-Duplex）通信链路搭建**<br>`repos/esp-hosted-mcu/recipes/bringup_spi_fd.md` | 使用 SPI 全双工作为 Host 与 Co-processor 之间的传输介质，完成协处理器（slave）固件烧录、主机（host）工程配置、引脚连接与链路验证。这是最易上手、可用跳线评估的传输方式。 |
| **UART 传输链路搭建（Wi-Fi + 蓝牙）**<br>`repos/esp-hosted-mcu/recipes/bringup_uart.md` | 使用 UART 作为 Host 与 Co-processor 之间的传输介质。UART 只需 2 根数据线（TX/RX）+ Reset + GND，所有 ESP 芯片都支持，适合低吞吐场景。该模式下 Wi-Fi 与蓝牙以 Hosted HCI（复用模式）在同一根 UART 上传输。不要与"仅蓝牙的专用 HCI over UART"混淆。 |
| **协处理器蓝牙控制器初始化（v2.5.2+）**<br>`repos/esp-hosted-mcu/recipes/host_bt_controller.md` | 自 ESP-Hosted-MCU v2.5.2 起，协处理器上的蓝牙控制器默认关闭，以便在使能前设置 BT MAC。本配方演示如何在 host 上经 `esp_hosted_bt_controller_init/enable` 启用协处理器控制器、读写 BT MAC，以及与 NimBLE / BlueDroid host 栈的衔接（Hosted HCI 与标准 HCI 两种路径）。 |
| **ESP_HOSTED 事件、心跳与传输故障恢复**<br>`repos/esp-hosted-mcu/recipes/host_events_recovery.md` | 订阅 `ESP_HOSTED_EVENT` 事件循环，感知协处理器 INIT、传输 UP/DOWN/失败，并用心跳（heartbeat）做活体检测；在传输故障或意外重启时自动 deinit → 重新 init/connect 恢复链路。这是生产级 ESP-Hosted 应用的必备模式。 |
| **外部共存 EXT_COEX（从 host 配置协处理器 PTA）**<br>`repos/esp-hosted-mcu/recipes/host_ext_coex.md` | 通过 ESP-Hosted 链路远程配置协处理器（slave）的硬件 PTA（Packet Traffic Arbitrator），让 slave 的 Wi-Fi 与 host 上的另一颗外部射频（BLE/Zigbee/Thread 等）共享 2.4 GHz 频段。支持 1/2/3/4-wire 模式与 leader/follower 角色。host 侧平台无关，亦适用于非 ESP host（STM32/nRF）。 |
| **主机控制协处理器 GPIO（GPIO Expander）**<br>`repos/esp-hosted-mcu/recipes/host_gpio_expander.md` | 通过 ESP-Hosted 链路，从 host 远程配置与读写协处理器的 GPIO（输出电平、输入读取、开漏、上下拉、中断类型）。相当于把协处理器的 IO"扩展"给 host 使用。API 平台无关，亦可用于非 ESP host。 |
| **主机省电（Host Power Save / Deep Sleep）**<br>`repos/esp-hosted-mcu/recipes/host_power_save.md` | 让 host MCU 进入低功耗状态（当前支持 deep sleep），由协处理器在需要时通过 GPIO 唤醒 host，同时保持网络在线。需要 host 与 slave 双方在 menuconfig 开启"Allow host to power save"。 |
| **主机 Wi-Fi STA 应用（经 ESP-Hosted）**<br>`repos/esp-hosted-mcu/recipes/host_wifi_sta.md` | 在 host 上运行一个标准的 ESP-IDF Wi-Fi Station 应用，底层经 ESP-Hosted RPC 透明转发到协处理器。应用代码与原生 ESP-IDF Wi-Fi 几乎一致——区别只在初始化阶段需要先 `esp_hosted_init()` / `esp_hosted_connect_to_slave()` 并等待传输就绪。 |
| **Network Split（host 与 slave 共享一个 IP，按端口分流）**<br>`repos/esp-hosted-mcu/recipes/network_split.md` | 让 host 与协处理器共享同一个 IP 地址，slave 根据端口把入站流量路由到自身或 host 的 lwIP 栈。host 睡眠时 slave 仍可处理 MQTT/DNS 等选定业务，并在收到唤醒包时把 host 拉起。仅支持 C5/C6/S2/S3 协处理器。 |
| **OpenThread RCP / Zigbee（协处理器作 RCP）**<br>`repos/esp-hosted-mcu/recipes/openthread_zigbee_rcp.md` | 把 ESP 协处理器配置为 802.15.4 的 RCP（Radio Co-Processor），host 上运行 OpenThread Host 或 Zigbee Host。当前 OpenThread/Zigbee 数据通过**专用 UART** 通道在 host 与 RCP 间传输（与 ESP-Hosted 主传输分离）；Wi-Fi 仍走 ESP-Hosted 传输。可工作于基础模式或 Border Router / Gateway 模式。 |
| **Peer / Custom 数据传输（host ↔ 协处理器原始二进制通道）**<br>`repos/esp-hosted-mcu/recipes/peer_custom_data.md` | 在 host 与协处理器之间建立一条独立于 RPC 控制流量与网络/BT 数据之外的私有二进制通道。用 `msg_id` 区分不同业务，发送任意原始字节（最大 8166 字节/包），host 与 slave 两侧 API 同名。适用于 host ↔ co-processor 的应用级命令/传感数据透传。 |
| **协处理器 OTA（经 ESP-Hosted 传输链路）**<br>`repos/esp-hosted-mcu/recipes/slave_ota.md` | 在首次串口烧录协处理器固件后，后续的 slave 固件升级**复用现有的 ESP-Hosted 传输链路**（SDIO/SPI/UART）完成，无需额外硬件、ESP-Prog 或物理访问。host 通过 RPC 调用 `esp_hosted_slave_ota_begin/write/end/activate` 把固件分块写入协处理器并激活重启。 |
| **运行期配置传输介质（esp_hosted_*_set_config）**<br>`repos/esp-hosted-mcu/recipes/transport_config_code.md` | 不依赖 menuconfig 静态选择，而是在代码里通过 `esp_hosted_<transport>_set_config()` 自定义 SDIO / SPI / SPI-HD / UART 的引脚、时钟、队列大小等参数，再调用 `esp_hosted_init()`。适用于引脚重映射、运行期切换、或在非 ESP host 上移植。 |
| **Wi-Fi Easy Connect (DPP) 入网设备端（Enrollee）**<br>`repos/esp-hosted-mcu/recipes/wifi_dpp_enrollee.md` | 在 host 上以 DPP Responder-Enrollee 模式入网——host 生成并显示 QR 码，由支持 Wi-Fi Easy Connect（Initiator）的设备（如 Android 10+）扫码后把 SSID/密码安全下发，无需在 host 上预置明文密码。可作为无 UI 产品的 WPA-PSK 替代入网方式。 |
| **Wi-Fi iTWT（Individual Target Wake Time，Wi-Fi 6 省电）**<br>`repos/esp-hosted-mcu/recipes/wifi_itwt.md` | 在 host 上经 ESP-Hosted 协商 iTWT（802.11ax 个别目标唤醒时间），让 STA 与 AP 协商自己的唤醒/睡眠周期以降低 Wi-Fi 空口功耗。仅支持 Wi-Fi 6 协处理器（ESP32-C5 / C6），STA 模式，工作在 modem-sleep（默认）或 light-sleep（规划中）。 |

### `esp-nimble` (13 recipes)

| recipe | 摘要 |
|---|---|
| **设备地址配置**<br>`repos/esp-nimble/recipes/address_setup.md` | 为 NimBLE 设备配置本机 BLE 地址（public / 静态随机 / NRPA / RPA 隐私），并通过 `ble_hs_id_infer_auto` 推断 own_addr_type。 |
| **不可连接广播（Beacon）**<br>`repos/esp-nimble/recipes/beacon.md` | 实现不可连接的广播者（Broadcaster）：non-connectable advertising，常用于 Beacon / 厂商自定义数据广播，使用 NRPA 作为本机地址。 |
| **中心连接与 GATT 客户端**<br>`repos/esp-nimble/recipes/central_connect.md` | 实现中心角色：扫描到目标后取消扫描、`ble_gap_connect` 发起连接、进行服务发现、对特征执行读 / 写 / 订阅。 |
| **连接参数更新**<br>`repos/esp-nimble/recipes/conn_param_update.md` | 连接建立后协商更合适的连接参数：从机用 L2CAP Connection Parameter Update Procedure 请求、主机用 Link-Layer Connection Parameters Request Procedure 下发，以及处理对端的更新请求回调（accept / reject）与最终的 `BLE_GAP_EVENT_CONN_UPDATE` 结果。 |
| **扩展广播与周期广播**<br>`repos/esp-nimble/recipes/ext_adv.md` | 使用 NimBLE 扩展广播 API（`ble_gap_ext_adv_*`）配置多个 advertising instance，支持大广播数据、coded/2M PHY、周期广播（Periodic Advertising）。 |
| **自定义 GATT 服务**<br>`repos/esp-nimble/recipes/gatt_server.md` | 用 NimBLE 的静态服务表 `struct ble_gatt_svc_def` 定义自定义 GATT 服务与特征，实现 access_cb 读写回调，并通过 count + add 注册。 |
| **NimBLE Host 初始化**<br>`repos/esp-nimble/recipes/host_init.md` | 在 ESP-IDF 工程中正确初始化 NimBLE Host：注册回调、注册 GATT 服务、启动 host 任务，并确保所有 GAP 操作在 Host 同步后发起。 |
| **Bluetooth Mesh 节点**<br>`repos/esp-nimble/recipes/mesh_node.md` | 使用 NimBLE Mesh 子系统实现一个 Bluetooth Mesh 节点，注册 Generic OnOff / Health / Vendor 模型，完成初始化与代理广播。 |
| **服务端通知 / 指示**<br>`repos/esp-nimble/recipes/notify.md` | 在 GATT 服务端实现 NOTIFY / INDICATE 特征，通过 `ble_gatts_notify_custom` 发送通知数据，处理客户端的 CCCD 订阅事件。 |
| **可连接广播外设**<br>`repos/esp-nimble/recipes/peripheral_adv.md` | 实现一个传统可连接 BLE 外设：设置广播数据 / 扫描响应、启动非定向广播、处理连接与断开并在断开后恢复广播。 |
| **LE PHY 选择（1M / 2M / Coded 远距离）**<br>`repos/esp-nimble/recipes/phy_update.md` | 在已建立的 BLE 连接上切换 LE PHY——1 Mbps 兼容、2 Mbps 高吞吐、Coded PHY（S=2 / S=8）远距离；包含设置默认偏好、按连接设置偏好、读取当前 PHY，以及处理 `BLE_GAP_EVENT_PHY_UPDATE_COMPLETE` 结果事件。 |
| **扫描（Observer）**<br>`repos/esp-nimble/recipes/scanner.md` | 实现 BLE 扫描（passive / active），解析广播报告 `BLE_GAP_EVENT_DISC`，并解析 `ble_hs_adv_fields` 各字段。 |
| **SMP 配对与加密**<br>`repos/esp-nimble/recipes/security_pairing.md` | 实现 BLE 安全管理：响应 `BLE_GAP_EVENT_PASSKEY_ACTION`、通过 `ble_sm_inject_io` 提供输入、发起配对、读取加密状态。 |

### `esp-mqtt` (12 recipes)

| recipe | 摘要 |
|---|---|
| **自定义 Outbox 实现**<br>`repos/esp-mqtt/recipes/custom_outbox.md` | 通过 `CONFIG_MQTT_CUSTOM_OUTBOX` 替换默认 outbox 实现（如持久化到 NVM、用 C++ 内存资源等），对应 `examples/custom_outbox/`。 |
| **事件回调完整骨架**<br>`repos/esp-mqtt/recipes/event_handling.md` | 处理 ESP-MQTT 全部事件类型，包括 CONNECTED/DISCONNECTED/SUBSCRIBED/UNSUBSCRIBED/PUBLISHED/DATA/ERROR，并解析错误句柄。 |
| **遗嘱消息（LWT）**<br>`repos/esp-mqtt/recipes/last_will.md` | 配置 Last Will and Testament，使客户端异常断开时由 broker 代发通知消息。MQTT 3.1.1 与 5.0 均支持。 |
| **MQTT 5.0 协议**<br>`repos/esp-mqtt/recipes/mqtt5.md` | 使用 MQTT v5.0：设置协议版本、连接属性、用户属性、共享订阅、读取 reason code 与 CONNACK 服务端属性，对应 `examples/mqtt5/`。 |
| **QoS 1/2 与 Outbox 管理**<br>`repos/esp-mqtt/recipes/outbox_qos.md` | 理解 ESP-MQTT 的内存 outbox 机制，处理不稳定网络下的消息堆积、限流、重传与过期，避免 `-2` 与丢消息。 |
| **PSK 预共享密钥认证（mqtts://）**<br>`repos/esp-mqtt/recipes/psk_auth.md` | 在不支持证书或资源受限时，使用 TLS-PSK 预共享密钥认证 broker，对应 `examples/ssl_psk/`。PSK 仅在无其它校验方式时启用。 |
| **发布、订阅与 QoS**<br>`repos/esp-mqtt/recipes/publish_subscribe.md` | 使用 `esp_mqtt_client_publish` / `esp_mqtt_client_subscribe` / `unsubscribe`，理解三种 QoS、retain、多主题订阅与 publish/enqueue 返回值。 |
| **MQTT over TCP 最小连接**<br>`repos/esp-mqtt/recipes/tcp_connect.md` | 使用 ESP-MQTT 通过纯 TCP（`mqtt://`）连接 broker，完成 init/register/start 与事件回调的骨架，是所有其它传输方式的基础。 |
| **数字签名外设 TLS 认证（DS peripheral）**<br>`repos/esp-mqtt/recipes/tls_digital_signature.md` | 使用 ESP32-S2/S3/C3/C5/C6/H2/P4 内置的 Digital Signature（DS）硬件外设完成 mqtts:// 双向 TLS 认证，私钥永不出硬件；对应 `examples/ssl_ds/`。与 `tls_mutual_auth.md`（PEM 嵌入 cert + key）是两条不同的工作流。 |
| **双向 TLS 认证（mqtts://）**<br>`repos/esp-mqtt/recipes/tls_mutual_auth.md` | 使用客户端证书 + 私钥 + 服务端 CA 实现 mqtts:// 双向认证，对应 `examples/ssl_mutual_auth/`。 |
| **服务端证书单向校验（mqtts://）**<br>`repos/esp-mqtt/recipes/tls_server_cert.md` | 仅用服务端 CA 证书校验 broker 身份（不提供客户端证书），对应 `examples/ssl/`，是最常见的 TLS 用法。 |
| **WebSocket 与 WebSocket Secure（ws:// / wss://）**<br>`repos/esp-mqtt/recipes/ws_wss.md` | 通过 WebSocket 传输连接 MQTT broker，对应 `examples/ws/`（ws）与 `examples/wss/`（wss）。常用于穿越 80/443 防火墙或走 CDN/反代。 |

### `esp-modbus` (10 recipes)

| recipe | 摘要 |
|---|---|
| **自定义与覆盖功能码处理器**<br>`repos/esp-modbus/recipes/custom_handlers.md` | 用 `mbc_set_handler` / `mbc_get_handler` / `mbc_delete_handler` / `mbc_get_handler_count` 注册新的厂商自定义功能码（如 0x41），或覆盖标准功能码（如 0x04 读输入寄存器）。覆盖主站和从站两种用法，并附 FC 0x41 回显示例。 |
| **主站数据字典（Data Dictionary）**<br>`repos/esp-modbus/recipes/data_dictionary.md` | 为 Modbus 主站编写 `mb_parameter_descriptor_t` 数据字典表，把物理量（CID）映射到从站的 Modbus 寄存器。覆盖 CID 枚举、`STR`/`OPTS`/`HOLD_OFFSET` 宏、寄存器类型与数据类型的搭配、`param_offset` 的含义、自定义命令权限。 |
| **扩展数据类型与字节序转换**<br>`repos/esp-modbus/recipes/extended_types.md` | 启用 `CONFIG_FMB_EXT_TYPE_SUPPORT`，使用 `PARAM_TYPE_U32_ABCD` / `PARAM_TYPE_FLOAT_CDAB` / `PARAM_TYPE_DOUBLE_HGFEDCBA` 等扩展类型，以及 `mb_set_float_abcd` / `mb_get_uint32_dcba` 等字节序转换助手，正确处理第三方设备的 32/64 位值。 |
| **Modbus 串行主站（RTU / ASCII）**<br>`repos/esp-modbus/recipes/serial_master.md` | 在 ESP32 系列芯片上构建 Modbus 串行主站，覆盖 UART/RS485 初始化、数据字典（Data Dictionary）注册、按 CID 轮询从站参数、自定义命令、销毁。 |
| **Modbus 串行从站（RTU / ASCII）**<br>`repos/esp-modbus/recipes/serial_slave.md` | 在 ESP32 系列芯片上构建 Modbus 串行从站，覆盖 UART/RS485 初始化、寄存器区域描述符（Holding/Input/Coil/Discrete）、从站事件循环与销毁。适用于 RTU 与 ASCII 两种模式。 |
| **从站设备识别（Report Slave ID / FC 0x11 从站侧）**<br>`repos/esp-modbus/recipes/slave_device_id.md` | 在 ESP32 Modbus 从站上设置厂商自定义的设备识别信息（短 UID、运行状态字节、厂商扩展数据），供主站通过标准命令 0x11 Report Slave ID 读回。覆盖 `mbc_set_slave_id` / `mbc_get_slave_id` 的调用时机、`INIT_DEV_ID` 结构体的构造方式，以及配套的 Kconfig 三元组。 |
| **从站事件循环与参数访问通知**<br>`repos/esp-modbus/recipes/slave_events.md` | 在 Modbus 从站中使用 `mbc_slave_check_event` 阻塞等待主机访问，用 `mbc_slave_get_param_info` 取出被访问寄存器的详细信息（时间戳、偏移、类型、地址、大小），按事件掩码分类处理 Holding/Input/Coil/Discrete 访问。 |
| **从站寄存器区域映射**<br>`repos/esp-modbus/recipes/slave_register_areas.md` | 用 `mbc_slave_set_descriptor` 把用户存储结构映射到 Modbus 的 Holding / Input / Coil / Discrete 区域。覆盖 `mb_register_area_descriptor_t` 字段、`HOLD_OFFSET`/`INPUT_OFFSET` 宏、分段区域、字节 vs 位的区别、访问权限。 |
| **Modbus TCP 主站**<br>`repos/esp-modbus/recipes/tcp_master.md` | 在 ESP32 系列芯片（Wi-Fi 或以太网）上构建 Modbus TCP 主站，覆盖 netif 初始化、从站 IP 地址表（含 MDNS / 静态 IP / IPv6）、`mbc_master_create_tcp`、按 CID 轮询、销毁。 |
| **Modbus TCP 从站**<br>`repos/esp-modbus/recipes/tcp_slave.md` | 在 ESP32 系列芯片（Wi-Fi 或以太网）上构建 Modbus TCP 从站，覆盖 netif 初始化、`mbc_slave_create_tcp`、多连接、keep-alive、事件循环与销毁。 |

### `esp-usb` (13 recipes)

| recipe | 摘要 |
|---|---|
| **USB 设备：CDC-ACM 串口**<br>`repos/esp-usb/recipes/device_cdc_serial.md` | 把 ESP 芯片做成一个 USB CDC-ACM 串口设备（虚拟串口），支持发送、接收、回调与双串口。 |
| **USB 设备：复合设备（CDC + MSC）**<br>`repos/esp-usb/recipes/device_composite.md` | 让 ESP 芯片同时作为 USB 串口和大容量存储设备（composite），使用接口关联描述符（IAD）。 |
| **USB 设备：控制台重定向与 VFS**<br>`repos/esp-usb/recipes/device_console_vfs.md` | 把标准输入/输出（`printf`/stdin）重定向到 USB CDC，或将 CDC 接口注册进 VFS 以便用 `fopen/fread/fwrite` 读写。 |
| **USB 设备：外部 PHY（ESP32-S3）**<br>`repos/esp-usb/recipes/device_external_phy.md` | 在 ESP32-S3 上使用外部 USB PHY（SP5301/TUSB1106/STUSB03E），让 USB-OTG 与 USB-Serial-JTAG 同时工作。 |
| **USB 设备：驱动安装/卸载与事件、VBUS 监测**<br>`repos/esp-usb/recipes/device_install_uninstall.md` | TinyUSB 设备驱动的安装与卸载生命周期、ATTACHED/DETACHED/SUSPEND/RESUME 事件回调、以及自供电设备的 VBUS 监测配置。 |
| **USB 设备：MSC 大容量存储**<br>`repos/esp-usb/recipes/device_msc_storage.md` | 把 ESP 芯片做成 USB 大容量存储（U 盘）设备，存储介质为 SPI-Flash 或 SD 卡，处理挂载/卸载事件。 |
| **USB 设备：NCM / RNDIS 以太网（Ethernet-over-USB）**<br>`repos/esp-usb/recipes/device_ncm_net.md` | 用 `tinyusb_net` 把 ESP32 暴露为一个 USB 网络接口（CDC-NCM 或 ECM/RNDIS），通过 `tinyusb_net_init` 注册收包/TX 释放回调，并用 `tinyusb_net_send_sync` / `tinyusb_net_send_async` 收发以太网帧。常用于 USB tethering、嵌入式 USB 以太网。 |
| **USB 主机：CDC-ACM 主机驱动**<br>`repos/esp-usb/recipes/host_cdc_acm.md` | 用 CDC-ACM 主机驱动与 USB 串口设备/调制解调器通信，包括 CP210x、FTDI、CH34x 等 vendor-specific 芯片。 |
| **USB 主机：HID 主机驱动（键盘/鼠标）**<br>`repos/esp-usb/recipes/host_hid.md` | 用 HID 主机驱动与 USB HID 设备（键盘、鼠标等）通信，处理驱动级与接口级事件、获取输入报告。 |
| **USB 主机：Host Library 基本用法**<br>`repos/esp-usb/recipes/host_library_basic.md` | USB Host Library 的安装、Daemon Task、client 注册、设备打开/接口 claim/裸传输，以及完整的卸载流程。 |
| **USB 主机：MSC 主机驱动（U 盘读写）**<br>`repos/esp-usb/recipes/host_msc.md` | 用 MSC 主机驱动读写 USB U 盘/移动存储，支持扇区读写、VFS 注册和设备信息查询。 |
| **USB 主机：UAC 主机驱动（USB 音频录放）**<br>`repos/esp-usb/recipes/host_uac.md` | 用 UAC 主机驱动连接 USB 音频设备（扬声器、麦克风、耳机），完成 12 步生命周期：安装 → 连接回调 → 打开 → 查格式（alt 参数）→ start/stop 流 → suspend/resume → 音量/静音控制 → RX/TX_DONE 回调 → close → uninstall。当前支持 UAC 1.0。 |
| **USB 主机：UVC 主机驱动（USB 摄像头视频流）**<br>`repos/esp-usb/recipes/host_uvc.md` | 用 UVC 主机驱动从 USB 摄像头采集视频流，处理设备连接回调、格式协商、帧回调（Frame Buffer），并在任务中取帧与归还帧。支持等时/批量传输、PSRAM 帧缓冲、多路流、运行时改格式。 |

### `tinyusb` (14 recipes)

| recipe | 摘要 |
|---|---|
| **Audio 设备类（UAC2 麦克风/扬声器/耳机）**<br>`repos/tinyusb/recipes/audio_uac2_device.md` | 用 TinyUSB Audio 类实现 USB 音频 2.0（UAC2）设备，包括麦克风（IN 端点）、扬声器（OUT 端点 + 反馈端点）、耳机（双向）、异步反馈端点（feedback endpoint）、采样率协商与音量/静音控制。 |
| **编译与烧录**<br>`repos/tinyusb/recipes/build_and_flash.md` | 用 CMake（首选）或 Make 构建 TinyUSB 示例/项目，包括拉取 MCU 依赖、选 `BOARD`、生成固件、jlink/openocd/uf2 烧录。 |
| **CDC 虚拟串口**<br>`repos/tinyusb/recipes/cdc_virtual_serial.md` | 用 TinyUSB CDC 类实现 USB 虚拟串口，包括配置使能、收发读写、DTR/RTS 线状态、行编码回调。 |
| **USB 描述符编写**<br>`repos/tinyusb/recipes/descriptors_config.md` | 编写 TinyUSB 设备侧描述符回调：设备/配置/字符串描述符，使用 `TUD_*_DESCRIPTOR` 宏组装配置描述符，正确分配接口号与端点号。 |
| **设备栈从零集成**<br>`repos/tinyusb/recipes/device_getting_started.md` | 在自定义固件中集成 TinyUSB 设备栈，完成 board_init → tusb_init → tud_task 主循环与必需的描述符回调，使设备可被主机枚举。 |
| **DFU 固件升级设备类（DFU 模式 + DFU Runtime）**<br>`repos/tinyusb/recipes/dfu_device.md` | 用 TinyUSB DFU 类实现 USB 固件升级，区分 **DFU Runtime**（应用运行时通过 DETACH 跳转到 bootloader）与 **DFU 模式**（ bootloader 内实际收发固件），含 download/upload/manifest 回调、多分区（alt）、bwPollTimeout 与 `tud_dfu_finish_flashing` 异步完成。 |
| **HID 设备（键盘/鼠标/手柄）**<br>`repos/tinyusb/recipes/hid_device.md` | 用 TinyUSB HID 类实现 USB HID 设备，包括 report descriptor、键盘/鼠标/游戏手柄 report 上报、多 report 链式续发与 `tud_hid_set_report_cb`。 |
| **USB 主机（CDC/MSC/HID）**<br>`repos/tinyusb/recipes/host_cdc_msc_hid.md` | 用 TinyUSB 主机栈枚举并通信外接 USB 设备，包括 `tuh_task` 主循环、挂载/卸载回调、CDC-ACM/HID/MSC host API 与异步传输回调。 |
| **MIDI 音乐设备接口类**<br>`repos/tinyusb/recipes/midi_device.md` | 用 TinyUSB MIDI 类实现 USB MIDI 设备（MIDI 输入/输出），包括 4 字节 USB-MIDI 事件包格式、cable number、Note On/Off 收发、`tud_midi_packet_read/write` 与流式 `tud_midi_stream_write`。 |
| **MSC 大容量存储（U 盘）**<br>`repos/tinyusb/recipes/msc_device.md` | 用 TinyUSB MSC 类实现 USB U 盘，包括 RAM/Flash 后端、必需的 SCSI 回调（read10/write10/capacity/inquiry）、多 LUN 与可写控制。 |
| **tusb_config.h 配置指南**<br>`repos/tinyusb/recipes/tusb_config_guide.md` | 系统说明 TinyUSB 核心配置宏（`CFG_TUSB_*` / `CFG_TUD_*` / `CFG_TUH_*`）：MCU、OS、类使能、缓冲区、端点 0、速度、内存对齐。 |
| **USB Type-C 与 Power Delivery（PD 3.0）**<br>`repos/tinyusb/recipes/typec_power_delivery.md` | 用 TinyUSB Type-C / Power Delivery 栈（`tuc_` API，`src/typec/`）实现 USB PD 3.0 sink/source，解析 Source Capabilities（PDO）、按电压/电流选 PDO 并发 Request（RDO）、响应 Accept/Reject/PS_READY。 |
| **USBTMC 测试测量设备类（SCPI/VISA 仪器）**<br>`repos/tinyusb/recipes/usbtmc_device.md` | 用 TinyUSB USBTMC 类把设备暴露成 USB 测试测量仪器（电源、万用表、示波器），让 pyvisa / NI-VISA / Linux usbtmc 通过 SCPI 命令读写。含 USB488 模式、status byte（STB/MAV/SRQ）、`*IDN?` 查询、Bulk-IN/OUT 消息收发与清除/中止回调。 |
| **Vendor 类与 WebUSB**<br>`repos/tinyusb/recipes/vendor_and_webusb.md` | 用 TinyUSB Vendor 类实现厂商自定义 USB 通信（含 WebUSB/WinUSB），包括缓冲模式与零缓冲直通模式、收发 API、WebUSB URL 描述符。 |

### `esp-at` (8 recipes)

| recipe | 摘要 |
|---|---|
| **添加用户自定义 AT 指令**<br>`repos/esp-at/recipes/add_custom_command.md` | 在不修改 esp-at 仓库源码的前提下，通过 `at_custom_cmd` 组件添加用户自定义 AT 指令，包括四种指令类型（Test/Query/Set/Execute）、参数解析、结果输出、可选参数、阻塞执行与接收端口原始数据。 |
| **通过 SPI 或 SDIO 承载 AT 指令**<br>`repos/esp-at/recipes/at_over_spi_sdio.md` | 不使用 UART，改为通过 SPI 或 SDIO 接口在 ESP-AT 设备与主机 MCU 之间传输 AT 指令与数据，适用于需要更高吞吐或主机 MCU 已占用 UART 的场景。 |
| **本地编译与烧录 ESP-AT 固件**<br>`repos/esp-at/recipes/build_and_flash.md` | 在本地克隆 esp-at 仓库，安装 ESP-IDF 环境，选择目标芯片/模块，配置功能，编译生成 `factory_XXX.bin` 并烧录到设备。 |
| **自定义 BLE GATT 服务**<br>`repos/esp-at/recipes/customize_ble_service.md` | 修改 ESP-AT 的 BLE GATT 服务定义文件 `gatts_data.csv`，自定义服务/特征/描述符的 UUID、权限与值，无需改源码。 |
| **自定义分区表 at_customize.csv**<br>`repos/esp-at/recipes/customize_partitions.md` | 修改二级分区表 `at_customize.csv`，新增/调整用户数据分区，生成并烧录 `at_customize.bin`，为 `AT+SYSFLASH`、`AT+FS`、SSL 服务端、BLE 服务端等功能提供存储。 |
| **实现 OTA 升级**<br>`repos/esp-at/recipes/ota_upgrade.md` | 选择并实现 ESP-AT 的三种 OTA 方案之一（`AT+USEROTA` 自有服务器、`AT+CIUPDATE` iot.espressif.cn、`AT+WEBSERVER` 浏览器/小程序），完成固件或用户分区升级。 |
| **用 at_override_module_config 覆盖模块配置**<br>`repos/esp-at/recipes/override_module_config.md` | 通过 `at_override_module_config` 外部目录覆盖默认模块配置（sdkconfig.defaults、补丁、分区表、工厂参数、ble_data 等），无需修改 esp-at 仓库源码，便于在自有 git 仓库轻量托管定制内容。 |
| **修改 AT 端口引脚**<br>`repos/esp-at/recipes/set_port_pin.md` | 修改 AT 命令端口（默认 UART1）与日志端口（默认 UART0）的 TX/RX/CTS/RTS 引脚，适配自定义硬件布局。 |

### `esp-protocols` (11 recipes)

| recipe | 摘要 |
|---|---|
| **eppp_link 双 MCU PPP 组网**<br>`repos/esp-protocols/recipes/eppp_link.md` | 用 eppp_link 在两个 MCU 之间通过 UART/SPI/SDIO/以太网建立 PPP 通道。典型用途是 WiFi 协处理器（通信协处理器跑 PPP server + NAT，主控跑 PPP client 拿到联网能力）。 |
| **esp_dns 安全 DNS（DoT / DoH / TCP）**<br>`repos/esp-protocols/recipes/esp_dns_secure.md` | 用 esp_dns 组件建立 DNS over TLS（DoT）、DNS over HTTPS（DoH）或 TCP DNS 解析，绕过明文 UDP DNS、提升隐私与可靠性。 |
| **mDNS 服务发布（Advertise）**<br>`repos/esp-protocols/recipes/mdns_advertise.md` | 初始化 mDNS，设置 hostname 与实例名，用 `mdns_service_add()` 发布服务（如 `_http._tcp`），配置 TXT 记录、子类型（subtype）与委托主机（delegated host）。 |
| **mDNS 服务查询（Query）**<br>`repos/esp-protocols/recipes/mdns_query.md` | 用 mDNS 查询局域网服务与主机，包括 PTR（服务）、SRV、TXT、A/AAAA 记录，遍历结果链表并释放。 |
| **WiFi AP ↔ PPPoS NAPT 网关**<br>`repos/esp-protocols/recipes/modem_ap_to_pppos_napt.md` | 把 ESP32 变成一个蜂窝路由器 —— 创建 WiFi soft-AP，用 lwip NAPT 将 AP 侧流量转发到蜂窝模组的 PPP 网络接口，使接入 AP 的客户端共享蜂窝上网。支持用标准 C-API `esp_modem_new`，或启用 `EXAMPLE_USE_MINIMAL_DCE` 走自定义 `NetDCE_Factory` + `NetModule`（仅实现建网所需的极简命令）。 |
| **CMUX 模式：数据通道上同时收发 AT**<br>`repos/esp-protocols/recipes/modem_cmux.md` | 用 CMUX（GSM 07.10 多路复用）在模组上建立两条虚拟通道，一条跑 PPP 数据，一条发 AT 命令，从而在拨号上网的同时查询信号、发短信等。 |
| **自定义蜂窝模组（Custom Module）**<br>`repos/esp-protocols/recipes/modem_custom_module.md` | 当官方内置模组枚举（SIM7600/SIM800/BG96 等）不满足时，通过继承 `GenericModule` 自定义模组类，添加私有 AT 命令，并用 `ESP_MODEM_DCE_CUSTOM` 创建 DCE。 |
| **蜂窝模组 PPPoS 拨号上网（UART）**<br>`repos/esp-protocols/recipes/modem_pppos_uart.md` | 通过 UART 连接蜂窝模组（如 SIM7600/SIM800/BG96），用 esp_modem 创建 PPP 网络接口并拨号上网，读取信号质量与 SIM 状态，切换 DATA 模式获取 IP。 |
| **ESP32 板载 Mosquitto Broker**<br>`repos/esp-protocols/recipes/mosquitto_broker.md` | 用 `mosquitto` 组件在 ESP32 上运行一个本地 MQTT broker —— 通过 `mosq_broker_run(&config)` 在调用线程阻塞运行；配置 `host`/`port`/`tls_cfg`（ESP-TLS 服务器配置）/`handle_connect_cb`（basic auth 校验）/`handle_message_cb`（消息回调）。可叠加本地 MQTT 客户端走 loopback 自测，或结合 serverless_mqtt 跨私有网络同步。 |
| **esp_mqtt_cxx C++ MQTT 客户端**<br>`repos/esp-protocols/recipes/mqtt_cxx_client.md` | 用 `esp_mqtt_cxx` 组件的 `idf::mqtt::Client` 封装类实现 MQTT 客户端 —— 继承 Client 重写 `on_connected`/`on_data` 等成员事件回调，通过 `BrokerConfiguration`/`ClientCredentials`/`Configuration` 三件套配置 broker、安全（明文/PEM/DER/PSK/Insecure/GlobalCAStore）与连接参数，使用 `subscribe`/`publish` 与 `Filter` 主题过滤器。支持 MQTT 3.11 与 TLS。 |
| **WebSocket 客户端**<br>`repos/esp-protocols/recipes/websocket_client.md` | 用 `esp_websocket_client` 建立 ws/wss 连接，处理连接/数据/错误事件，收发文本、二进制与分片帧，配置 TLS（证书包/双向认证）。 |

## AI/语音/视觉/DSP


### `esp-skainet` (13 recipes)

| recipe | 摘要 |
|---|---|
| **AFE 配置项调优**<br>`repos/esp-skainet/recipes/afe_config_tuning.md` | 理解并调整 `afe_config_t` 各字段，按场景开启/关闭 AEC、SE(BSS)、NS、VAD、AGC，选择 AFE type/mode 与内存分配策略。所有字段取自 `esp_afe_config.h`。 |
| **中文 TTS 语音合成**<br>`repos/esp-skainet/recipes/chinese_tts.md` | 用 esp-tts 把中文文本合成为 16k/16bit PCM 并通过 `esp_audio_play` 播放，支持从 UART 接收文本实时合成。发音集从 `voice_data` 分区 mmap 加载。 |
| **中文命令词识别**<br>`repos/esp-skainet/recipes/cn_speech_commands.md` | 在 ESP32-S3 上实现"唤醒 → 中文命令词识别"完整链路。WakeNet 唤醒后进入 MultiNet 命令模式，命中/超时后回到待唤醒，并从 sdkconfig 默认命令表导入。 |
| **自定义音频板移植**<br>`repos/esp-skainet/recipes/custom_board_porting.md` | 当 PCB 不在 `hardware_driver` 已支持板列表时，新建 `esp_custom_board.h`（引脚表）与 `bsp_board.c`（I2S/codec/feed 实现），让 `esp_board_init` / `esp_get_input_format` / `esp_get_feed_data` 跑在新硬件上。 |
| **运行时自定义命令词**<br>`repos/esp-skainet/recipes/customize_commands.md` | 不依赖 menuconfig 默认命令表，在运行时用 `esp_mn_commands_*` API 动态增删改命令词，并刷新 MultiNet 语言模型。适用于 mn6/mn7（推荐）。 |
| **深度降噪（AFE_TYPE_VC + NSNET2）**<br>`repos/esp-skainet/recipes/deep_noise_suppression.md` | 用语音通信型 AFE（`AFE_TYPE_VC`）+ 深度降噪模型做实时降噪，并把原始/降噪后 PCM 通过 ringbuf + SD 卡落盘用于评估对比。 |
| **双麦方向角估计（DOA）**<br>`repos/esp-skainet/recipes/direction_of_arrival.md` | 用 `esp_doa` 模块基于双麦克风做声源方向角（DOA）估计。需要关闭 AEC、按 input_format 中 `M` 的位置手动拆出左右声道，再喂给 `esp_doa_process`。 |
| **英文命令词识别**<br>`repos/esp-skainet/recipes/en_speech_commands.md` | 在 ESP32-S3 上实现"唤醒 → 英文命令词识别"。结构与中文版一致，区别仅在 MultiNet 模型过滤关键字（`ESP_MN_ENGLISH`）与 Kconfig 模型符号。注意：英文命令仅支持 ESP32-S3 系列（不支持 ESP32）。 |
| **性能基准测试与测试报告生成**<br>`repos/esp-skainet/recipes/perf_benchmarking.md` | 用 `perf_tester` 控制台与 `test/` 测试工程量化 WakeNet/MultiNet 的唤醒率（RAR）、误唤醒率（FAR）与 CPU/内存占用，并生成 pass/fail 报告。适用于产品发布前的精度验收与回归测试。 |
| **语音活动检测（VAD）**<br>`repos/esp-skainet/recipes/voice_activity_detection.md` | 用 AFE 内置 VAD 判断当前帧是语音还是噪声/静音，利用 `vad_cache` 避免首字被截断，并可把语音帧落盘到 SD 卡。 |
| **语音通信（Voice Communication）数据增强**<br>`repos/esp-skainet/recipes/voice_communication.md` | 用 `AFE_TYPE_VC` 对通信语音做实时增强（降噪/AGC），从 `fetch` 取增强后的单声道 PCM，用于上行通话或录制。结构与深度降噪几乎相同，重点在"取数据、不强求落盘"。 |
| **基于 AFE 的实时唤醒词检测**<br>`repos/esp-skainet/recipes/wake_word_afe.md` | 使用 Audio Front-End (AFE) 在 ESP32-S3 上实时检测唤醒词（WakeNet），包含 feed/detect 双任务、模型加载、唤醒阈值调节与多模型加载。 |
| **离线 PCM/WAV 唤醒词推理**<br>`repos/esp-skainet/recipes/wake_word_raw.md` | 不走 AFE，直接对内存中的 PCM/WAV 数据逐帧调用 WakeNet 检测。适用于离线批量评估唤醒模型、回放录音测试、单元测试。 |

### `esp-sr` (11 recipes)

| recipe | 摘要 |
|---|---|
| **AEC 回声消除**<br>`repos/esp-sr/recipes/aec_usage.md` | 使用 ESP-SR AEC 消除扬声器回声，覆盖三种集成方式（独立 `aec_create`、带 input_format 的 `afe_aec_create`、经 AFE pipeline）以及 SR/FD/VOIP 三类 mode 选型。 |
| **AFE 语音识别主线（WakeNet + 命令词）**<br>`repos/esp-sr/recipes/afe_sr_pipeline.md` | 用 ESP-SR 的 Audio Front-End（AFE）搭建离线语音识别主线：唤醒词触发后切换到命令词识别，包含 menuconfig 选模型、分区配置、AFE 初始化与 feed/fetch 双任务驱动。 |
| **AFE 语音通信主线（VC / FD）**<br>`repos/esp-sr/recipes/afe_vc_pipeline.md` | 用 ESP-SR AFE 搭建**语音通信 / 全双工**主线：选 `AFE_TYPE_VC` / `AFE_TYPE_VC_8K` / `AFE_TYPE_FD`，得到与 SR 不同的 pipeline（VC 走 AEC(VOIP)→NS(nsnet2)→VAD；FD 走 AEC(FD)→SE(BSS)→VAD→WakeNet），并把 `fetch` 出来的干净单声道音频送往网络传输或录音。与 SR 主线共用 feed/fetch 双任务骨架。 |
| **中文语音合成（esp-tts）**<br>`repos/esp-sr/recipes/chinese_tts.md` | 使用 ESP-SR 内置的中文 TTS 模块把 UTF-8 中文文本流式合成为 16k/16bit 语音。仅支持中文，需 `voice_data` 分区存放声音集。 |
| **自定义中英文命令词**<br>`repos/esp-sr/recipes/custom_commands.md` | 为 MultiNet5/6/7 自定义中文与英文命令词：通过 `commands_cn.txt`/`commands_en.txt` 文件、`esp_mn_commands_add` API、以及 `tool/multinet_g2p.py` Grapheme-to-Phoneme 工具。 |
| **DOA 声源定位**<br>`repos/esp-sr/recipes/doa_sound_localization.md` | 用 ESP-SR 的 SRP-PHAT 声源定位（DOA）从双麦克风估计语音方向角（0~180°）。覆盖独立 `esp_doa_*`（左右声道分开）与 AFE-aware 的 `afe_doa_*`（按 `input_format` 自动从交错多通道数据里抽取左右声道）两条路径。 |
| **从 ESP-SR V1.* 迁移到 V2.0**<br>`repos/esp-sr/recipes/migration_v1_v2.md` | 把基于 ESP-SR V1.*（`AFE_CONFIG_DEFAULT`、`ESP_AFE_SR_HANDLE` 等）的旧代码迁移到 V2.0 的新 API（`afe_config_init` + `esp_afe_handle_from_config`）。 |
| **模型选择、分区配置与烧录**<br>`repos/esp-sr/recipes/model_partition.md` | 通过 menuconfig 选择 ESP-SR 模型（NS / VAD / WakeNet / MultiNet），配置 `partitions.csv` 的 `model` 分区，生成并烧录 `srmodels.bin`。 |
| **MultiNet 命令词识别**<br>`repos/esp-sr/recipes/multinet_commands.md` | 加载 MultiNet 模型、通过 API 或 sdkconfig 增删改命令词、运行 detect/get_results，并区分单次与连续识别模式。命令词源数据来自 AFE fetch 的单声道 16k/16bit 音频。 |
| **VADNet 语音活动检测**<br>`repos/esp-sr/recipes/vadnet.md` | 使用 VADNet（神经网络 VAD，替代 WebRTC VAD）检测语音/噪声状态，处理 VAD cache 防止首字截断。可经 AFE pipeline 默认启用，或单独运行。 |
| **单独运行 WakeNet（不经过 AFE）**<br>`repos/esp-sr/recipes/wakenet_standalone.md` | 不通过 AFE pipeline，直接调用 WakeNet 模型进行唤醒词检测。适用于自建前端处理、单元测试或低延迟独立场景。生产场景推荐用 AFE，本 recipe 对应 `test_apps/esp-sr/main/test_wakenet.cpp`。 |

### `esp-dl` (16 recipes)

| recipe | 摘要 |
|---|---|
| **用 AutoQuant 自动搜索最优量化配置**<br>`repos/esp-dl/recipes/auto_quant.md` | 当默认 8-bit 量化精度不足时，用 `espdl_auto_quantize_onnx` 在多种量化策略与参数间自动搜索，按评估指标保留 Top-K 候选 `.espdl`，减少人工调参。 |
| **从 FLASH 分区加载模型**<br>`repos/esp-dl/recipes/load_model_partition.md` | 把 `.espdl` 存到独立的 SPIFFS 数据分区，用 `MODEL_LOCATION_IN_FLASH_PARTITION` 加载。模型可独立于 app 更新，开发期可用 `idf.py app-flash` 跳过模型分区。 |
| **从 .rodata 加载模型**<br>`repos/esp-dl/recipes/load_model_rodata.md` | 把 `.espdl` 模型嵌入应用的 `.rodata` 段，用 `MODEL_LOCATION_IN_FLASH_RODATA` 加载。最简单的加载方式，缺点是改代码也会重新烧模型。 |
| **从 SD 卡加载模型**<br>`repos/esp-dl/recipes/load_model_sdcard.md` | 把 `.espdl` 放到 FAT32 SD 卡，用 `MODEL_LOCATION_IN_SDCARD` 加载。适合 Flash 紧张或需要频繁换模型的场景。 |
| **测试、剖析模型内存与延迟（test / profile）**<br>`repos/esp-dl/recipes/profile_test.md` | 用 `model->test()` 验证板端推理正确性，用 `profile_memory()` / `profile_module()` / `profile()` 打印内存占用和逐层延迟，定位精度与性能瓶颈。 |
| **项目集成：把 esp-dl 作为托管组件加入工程**<br>`repos/esp-dl/recipes/project_setup.md` | 通过 ESP-IDF Component Registry 把 `esp-dl` 加入新工程，配置 `idf_component.yml` 与 `CMakeLists.txt`，准备加载 `.espdl` 模型。 |
| **高级 PTQ：混合精度（坏层 int16）与逐层权重均衡**<br>`repos/esp-dl/recipes/quantize_advanced_ptq.md` | 当默认 8-bit per-tensor PTQ（尤其 ESP32-S3）掉点严重，又不想上 TQT/QAT 时，用两种**确定性、可定向**的 PTQ 技巧：① 把量化误差最大的几层派发到 int16（混合精度）；② 对带 ReLU/ReLU6 的模型做逐层权重均衡（layerwise equalization）。两者都不需要标签、不需要训练。 |
| **用 ESP-PPQ 把 ONNX 模型量化为 .espdl**<br>`repos/esp-dl/recipes/quantize_onnx.md` | 使用 `espdl_quantize_onnx` 把 ONNX 模型做训练后量化（PTQ），导出可在 ESP 芯片部署的 `.espdl` 模型。PyTorch/TensorFlow/Paddle 需先转 ONNX。 |
| **用 ESP-PPQ 把 PyTorch 模型量化为 .espdl**<br>`repos/esp-dl/recipes/quantize_torch.md` | 使用 `espdl_quantize_torch` 直接对 `torch.nn.Module` 做量化并导出 `.espdl`，无需先导 ONNX。支持普通模型和流式模型（`auto_streaming`）。 |
| **用 TQT（Trained Quantization Thresholds）训练量化阈值**<br>`repos/esp-dl/recipes/quantize_tqt.md` | 当默认 8-bit PTQ 精度不足时，启用 ESP-PPQ 的 TQT：在 log 域优化 `scale = 2^k` 并联合微调权重（**无需标签**，loss 是浮点输出与量化输出的 MSE），导出仍满足 ESP-DL 的 Power-of-2 约束。介于 PTQ 与 QAT 之间，是精度不足时最直接的下一步。 |
| **执行模型推理（输入量化、运行、输出反量化）**<br>`repos/esp-dl/recipes/run_inference.md` | 加载 `.espdl` 后，获取输入/输出 TensorBase，对 float 输入做量化，调用 `run()`，再把 int 输出反量化为 float。这是 ESP-DL 最核心的推理流程。 |
| **部署流式（流式/时序）模型**<br>`repos/esp-dl/recipes/streaming_model.md` | 把长时序输入（如音频）切分成 chunk，逐 chunk 喂给流式 `.espdl` 模型；模型内部自动维护跨 chunk 的状态（StreamingCache）。适合音频/语音等实时场景。 |
| **图像分类端到端（JPEG 解码 → 模型 → Top-K 后处理）**<br>`repos/esp-dl/recipes/vision_classification.md` | 使用 ESP-DL 跑一个 MobileNetV2/ImageNet 分类模型：软件解码 JPEG 得到 `img_t`，构造 `ImageNetCls`（内部 `ImageNetClsPostprocessor` 输出 Top-K 类别，可选 softmax），`run(img)` 得到 `(类别名, 分数)` 列表。**分类不输出框，与检测后处理完全不同**。 |
| **视觉目标检测端到端（JPEG 解码 → 模型 → 后处理）**<br>`repos/esp-dl/recipes/vision_detection.md` | 使用 ESP-DL 跑一个 YOLO11/COCO 检测模型：软件解码 JPEG 得到 `img_t`，构造检测器（`DetectWrapper` 子类），`run(img)` 得到检测结果（类别、分数、框），按需释放内存。 |
| **视觉姿态估计端到端（JPEG 解码 → 模型 → 关键点后处理）**<br>`repos/esp-dl/recipes/vision_pose.md` | 使用 ESP-DL 跑一个 YOLO11n-pose/COCO 姿态模型：软件解码 JPEG 得到 `img_t`，构造 `COCOPose`（内部用 `yolo11posePostProcessor`），`run(img)` 得到每个实例的边界框 + 17 个 COCO 关键点坐标。 |
| **实例分割端到端（JPEG 解码 → 模型 → mask 后处理）**<br>`repos/esp-dl/recipes/vision_segmentation.md` | 使用 ESP-DL 跑一个 YOLO11n-seg/COCO 分割模型：软件解码 JPEG 得到 `img_t`，构造 `COCOSeg`（内部 `yolo11segPostProcessor` 解析 score/box/mask_coeff + 32 维 proto，合成逐实例二值 mask），`run(img)` 得到每个实例的框 + 与框对齐的 mask 栅格，可按 alpha 混合可视化。 |

### `esp-dsp` (15 recipes)

| recipe | 摘要 |
|---|---|
| **实时音频流 IIR / 三缓冲音效处理**<br>`repos/esp-dsp/recipes/audio_iir_streaming.md` | 用三缓冲（triple buffer）+ `dsps_biquad_f32` 做实时音效链：从文件/I2S 读 int16 → 转 float → 串联 lowShelf（bass）与 highShelf（treble）biquad → `dsps_mulc_f32` 调音量 → 数字限幅器 → 回 int16 送 codec；运行时通过按钮重新生成 biquad 系数，而滤波器延迟线跨块保持。区别于一次性的 `iir_biquad.md` 静态测试。 |
| **实时音频流 FFT / 频谱可视化**<br>`repos/esp-dsp/recipes/audio_spectrum_streaming.md` | 用定点 `dsps_fft2r_init_sc16` / `dsps_fft2r_sc16` 对 I2S 麦克风的双声道音频块做实时流式 FFT，配 `dsps_wind_blackman_harris_f32` 加窗、`dsps_cplx2reC_sc16` 拆分两路实信号频谱、转 dB 与滑动平均，最后送显示任务。区别于一次性的 `fft_complex.md` / `fft_real.md` 测试信号链路。 |
| **2D 卷积 / 图像处理**<br>`repos/esp-dsp/recipes/conv2d_image.md` | 用 `dspi_conv_f32` 对 2D 图像（`image2d_t` 结构）做卷积，复现 Matlab `conv2(A,B,'same')`；通过 `stride_x`/`stride_y`/`step_x`/`step_y` 字段表达行宽与子采样，适用于相机/传感器阵列的边缘检测、模糊、锐化等核卷积。 |
| **DCT / DST 变换**<br>`repos/esp-dsp/recipes/dct.md` | 用 `dsps_dct_f32`（DCT-II）、`dsps_dct_inv_f32`（逆 DCT）、`dsps_dctiv_f32`（DCT-IV）、`dsps_dstiv_f32`（DST-IV）做离散余弦/正弦变换。这些函数基于 FFT，使用前需 init FFT 表。 |
| **复数 Radix-2 FFT 与功率谱**<br>`repos/esp-dsp/recipes/fft_complex.md` | 用 `dsps_fft2r_fc32` 对交错复数（Re,Im,Re,Im）信号做 radix-2 FFT，经 bit-reverse 与 `dsps_cplx2reC_fc32` 拆分，得到两个实信号的功率谱（dB）。 |
| **实信号 FFT（Radix-4）**<br>`repos/esp-dsp/recipes/fft_real.md` | 用 `dsps_fft4r_fc32`（radix-4）对实信号做 FFT，配合 `dsps_bit_rev4r_fc32` 与 `dsps_cplx2real_fc32` 把结果还原为实数频谱；适合单路实信号、点数为 4 的幂的场景。 |
| **FIR 滤波器（标准 / 抽取 / 多速率）**<br>`repos/esp-dsp/recipes/fir_filter.md` | 用 `dsps_fir_f32`（标准 FIR）、`dsps_fird_f32`（抽取 FIR）、`dsps_firmr_f32`（多速率 FIR）做滤波、降采样与任意速率变换。涵盖 `fir_f32_t` 初始化、分块复用与延迟线对齐。 |
| **IIR Biquad 滤波器**<br>`repos/esp-dsp/recipes/iir_biquad.md` | 用系数生成器 `dsps_biquad_gen_*_f32`（LPF/HPF/BPF/notch/shelf/peak/allpass）生成 biquad 系数，再用 `dsps_biquad_f32`（单声道）或 `dsps_biquad_sf32`（立体声）做 IIR 滤波。 |
| **13 态 IMU 扩展卡尔曼滤波（EKF）**<br>`repos/esp-dsp/recipes/kalman_ekf.md` | 用 C++ 类 `ekf_imu13states`（继承自 `ekf`）做 IMU 姿态估计：初始化 → 标定阶段 → 每周期 `Process(gyro, dt)` → `UpdateRefMeasurement(accel, magn, R)` 修正陀螺仪偏差与姿态四元数。配合 `dspm::Mat` 与 `ekf` 静态方法（`eul2rotm`/`rotm2quat`/`quat2eul`）。 |
| **矩阵运算（C 接口与 C++ dspm::Mat）**<br>`repos/esp-dsp/recipes/matrix.md` | 用 C 接口 `dspm_mult_f32` / `dspm_mult_ex_f32` / `dspm_add/sub/mulc` 做矩阵运算，或用 C++ `dspm::Mat` 类（运算符重载、`solve`/`roots`/`inverse`/`det`/`t`/`eye`/`ones`）做线性代数求解。 |
| **项目集成与 Kconfig 配置**<br>`repos/esp-dsp/recipes/project_setup.md` | 把 esp-dsp 作为 ESP-IDF 组件加入新项目或已有项目，选择优化级别与最大 FFT 长度，跑通最小可工作 `app_main`。 |
| **重采样器（多速率 FIR 与多项式）**<br>`repos/esp-dsp/recipes/resampler.md` | 用多速率重采样器 `dsps_resampler_mr_init/exec/free` 做基于多速率 FIR 的采样率变换，或用多项式（Farrow）重采样器 `dsps_resampler_ph_init/exec` 做轻量的相位可调重采样。支持 float 与 int16 定点。 |
| **信号生成与文本显示**<br>`repos/esp-dsp/recipes/signal_generation.md` | 用 esp-dsp 生成测试信号（正弦、delta、复数 LUT 信号），并通过 `dsps_view` / `dsps_view_spectrum` 在串口打印文本波形与频谱图。 |
| **向量数学与点积**<br>`repos/esp-dsp/recipes/vector_math.md` | 用 `dsps_add_f32`/`dsps_sub_f32`/`dsps_mul_f32`/`dsps_mulc_f32`/`dsps_sqrt_f32` 做逐元素向量运算（含 step 参数），用 `dsps_dotprod_f32` 做点积。涵盖 f32 / s16 / s8 数据类型。 |
| **窗函数生成与应用**<br>`repos/esp-dsp/recipes/windows.md` | 用 `dsps_wind_*_f32` 生成 Hann / Blackman / Blackman-Harris / Blackman-Nuttall / Nuttall / flat-top 窗，并将其与信号相乘（可用 `dsps_mul_f32` 完成基本运算版本）。 |

### `esp-detection` (9 recipes)

| recipe | 摘要 |
|---|---|
| **一站式流水线 espdet_run.py**<br>`repos/esp-detection/recipes/all_in_one_pipeline.md` | 用 `espdet_run.py` 一条命令串联 train → export → quantize → 生成芯片端 ESP-IDF 工程（自动 `git clone esp-dl`、复制模板、`rename_project` 替换占位符）。 |
| **准备 YOLO 格式数据集**<br>`repos/esp-detection/recipes/dataset_prepare.md` | 按 Ultralytics YOLO 检测格式组织数据，编写 `cfg/datasets/*.yaml`，并按需启用 negative sampling（负样本）或 weighted sampling（类别均衡）。 |
| **搭建训练与量化环境**<br>`repos/esp-detection/recipes/env_setup.md` | 在 PC 上搭建 esp-detection 的 Python 训练/导出/量化环境（conda + requirements + esp-ppq），并验证自定义模块可被加载。 |
| **在 PC 上评估量化后模型**<br>`repos/esp-detection/recipes/eval_quantized.md` | 用 `deploy/eval_quantized_model.py` 在 PC 上对量化后的 ppq 计算图做 mAP 评估，验证 INT8 量化带来的精度损失。 |
| **导出 ONNX（opset 13 + onnxsim）**<br>`repos/esp-detection/recipes/export_onnx.md` | 用 `deploy/export.py::Export()` 把训练好的 `.pt` 导出为 ONNX（opset 13、onnxsim 简化），输出固定 6 个张量，为后续 esp-ppq 量化做准备。 |
| **在 ESP32-P4/S3 上部署与运行推理**<br>`repos/esp-detection/recipes/firmware_deploy.md` | 在 `espdet_run.py` 生成的 ESP-IDF 工程中，配置模型加载位置（flash/partition/sdcard）、编译、烧录，并在串口观察检测结果。 |
| **量化为 INT8 .espdl（esp-ppq PTQ）**<br>`repos/esp-detection/recipes/quantize_espdl.md` | 用 `deploy/quantize.py::quant_espdet()` 基于 esp-ppq 把 ONNX 后训练量化（PTQ）为 INT8 ESP-DL `.espdl`，需提供校准数据集。 |
| **非方形分辨率 rect=True 训练**<br>`repos/esp-detection/recipes/train_rect.md` | 对非方形输入（如 160×288）启用 `rect=True` 训练，含可选的方形预训练阶段，以在不增加模型复杂度/推理时间的前提下提升精度与速度。 |
| **方形分辨率训练 espdet_pico**<br>`repos/esp-detection/recipes/train_square.md` | 用 `train.py::Train()` 在方形输入分辨率（如 224×224、416×416）下从零训练 espdet_pico 检测模型，并验证 mAP。 |

### `esp-video-components` (12 recipes)

| recipe | 摘要 |
|---|---|
| **V4L2 POSIX 采集流程**<br>`repos/esp-video-components/recipes/capture_stream.md` | 用标准 V4L2 POSIX API（open/ioctl/mmap）从 `/dev/videoN` 采集图像流，包含设置格式、申请/映射缓冲、启动流、出队入队循环、停止流的全过程。 |
| **自定义传感器格式与寄存器序列**<br>`repos/esp-video-components/recipes/custom_format.md` | 当内置的 sensor 默认格式不满足需求时，通过自定义寄存器初始化序列 + `esp_cam_sensor_format_t` 描述 + `VIDIOC_S_SENSOR_FMT` 命令，让 sensor 按非内置分辨率/格式/帧率工作。 |
| **DVP 并行接口摄像头**<br>`repos/esp-video-components/recipes/dvp_sensor.md` | 在 ESP32-P4 / ESP32-S3 上配置 DVP（Digital Video Port）并行接口摄像头（如 OV2640、GC0308、BF3901 等），完成引脚、XCLK、数据宽度配置与采集。DVP 需要 `SOC_LCDCAM_CAM_SUPPORTED`。 |
| **图像/视频存储（SD 卡 / SPI Flash / USB MSC）**<br>`repos/esp-video-components/recipes/image_storage.md` | 将摄像头采集的帧经 JPEG/H.264 编码后，存储到 SD 卡、SPI Flash，或通过 USB MSC 暴露给主机。`image_storage` 示例分 `sd_card` 与 `usb_msc` 两个子目录。 |
| **ISP Pipeline 自动图像处理**<br>`repos/esp-video-components/recipes/isp_pipeline.md` | 为输出 RAW 格式的传感器（SC2336、OV5640 RAW 等）启用 ISP Pipeline Controller，自动执行 AE（自动曝光）、AWB（自动白平衡）、AF（自动对焦，需电机）算法，获得正常颜色与亮度的图像。 |
| **JPEG / H.264 硬件编解码（M2M 设备）**<br>`repos/esp-video-components/recipes/jpeg_h264_codec.md` | 使用 esp_video 的硬件编解码设备（JPEG 编码 `/dev/video10`、JPEG 解码 `/dev/video12`、H.264 编码 `/dev/video11`）进行 memory-to-memory（M2M）压缩。典型用于存图、视频流、UVC gadget。 |
| **MIPI-CSI 摄像头采集（ESP32-P4）**<br>`repos/esp-video-components/recipes/mipi_csi.md` | 在 ESP32-P4 上配置 MIPI-CSI 接口摄像头（如 SC2336、OV5640、OV2640 等 MIPI 型号），完成初始化与 RAW/YUV/RGB 数据采集。MIPI-CSI 仅 ESP32-P4 支持。 |
| **本地 HTTP 视频服务器（抓拍与 MJPEG 流）**<br>`repos/esp-video-components/recipes/simple_video_server.md` | 在 ESP32 上建立多端口 HTTP 服务器，通过浏览器进行图像抓拍、MJPEG 视频流预览与相机参数配置。`simple_video_server` 示例同时支持多摄像头。 |
| **SPI 接口摄像头**<br>`repos/esp-video-components/recipes/spi_sensor.md` | 在 ESP32-C3/C5/C6/C61（仅 SPI 可用）以及 ESP32-P4/S3 上配置 SPI（或 Parallel IO）接口的低分辨率摄像头，含双 SPI 摄像头同时使用。SPI 是唯一全系列支持的接口。 |
| **USB UVC Host（接入 USB 摄像头）**<br>`repos/esp-video-components/recipes/usb_uvc_host.md` | 把 ESP32 作为 USB Host，接入标准 UVC（USB Video Class）USB 摄像头/webcam，通过 `/dev/video40`~`/dev/video49` 采集。需要 USB-OTG（ESP32-P4/S3/S31）。 |
| **USB UVC Gadget（ESP32 作为 USB 摄像头）**<br>`repos/esp-video-components/recipes/uvc_gadget.md` | 通过 USB 外设把 ESP32 实现成标准 UVC 摄像头设备，主机（PC/手机）免驱识别为 webcam。`uvc` 示例采集摄像头帧 → JPEG/H.264 编码 → UVC 上报。 |
| **初始化 esp_video 系统**<br>`repos/esp-video-components/recipes/video_init.md` | 配置并初始化 esp-video-components 视频子系统，包括 SCCB（I2C）、复位/掉电引脚、MIPI-CSI/DVP/SPI/USB 接口子设备注册，是所有采集操作的前置步骤。 |

### `esp-rainmaker` (14 recipes)

| recipe | 摘要 |
|---|---|
| **Claiming 与配网**<br>`repos/esp-rainmaker/recipes/claiming_and_provisioning.md` | 选择并配置 Claiming 类型（Self / Assisted / No Claim），完成 Wi-Fi/Thread 配网、Proof of Possession（PoP）以及用户-节点映射（含挑战-响应）。 |
| **控制器节点与 User Helper API**<br>`repos/esp-rainmaker/recipes/controller_node.md` | 把一个 ESP32 设备变成 RainMaker **控制器节点**，通过云端 User API（`CONFIG_ENABLE_RM_USER_HELPER_API`）枚举、查询、设置**其它**节点的参数 / 配置 / 调度 / 在线状态，并可移除用户-节点映射；支持子节点数据下行回调。 |
| **自定义设备与参数**<br>`repos/esp-rainmaker/recipes/custom_device.md` | 用 `esp_rmaker_device_create()` + `esp_rmaker_param_create()` 创建自定义设备，添加自定义参数、UI Type、数值范围（bounds）、有效字符串列表，并编写 write 回调。 |
| **以太网 / 双网络连接**<br>`repos/esp-rainmaker/recipes/ethernet_connectivity.md` | 以太网作为 RainMaker **主传输**（无 BT/SoftAP 配网），通过 **on-network challenge-response** 完成用户-节点映射；可选 `EXAMPLE_ENABLE_WIFI` 开启双网络（"先连上的胜出"）。覆盖 PHY GPIO 配置、`app_ethernet_init/start`、与 `app_network` 的并存。 |
| **从零创建 RainMaker 节点**<br>`repos/esp-rainmaker/recipes/getting_started.md` | 从零搭建一个 ESP RainMaker 节点工程，完成 NVS、网络初始化、节点创建、设备挂载、服务启用与 Agent 启动的完整流程。这是所有 RainMaker 应用的骨架。 |
| **本地控制（Local Control）**<br>`repos/esp-rainmaker/recipes/local_control.md` | 启用 RainMaker 本地控制服务，让用户在同一 Wi-Fi/Thread 网络内无需互联网即可控制节点，配置 PoP、安全等级（sec0/sec1/sec2），以及 chal_resp 端点。 |
| **MQTT 直发、预算与主题**<br>`repos/esp-rainmaker/recipes/mqtt_topics.md` | 使用 `esp_rmaker_mqtt_*` 与 `esp_rmaker_publish_direct()` 进行 MQTT 直发，理解 MQTT 预算（budgeting）与 Basic Ingest 主题机制以降低成本与避免丢消息。 |
| **多设备节点**<br>`repos/esp-rainmaker/recipes/multi_device.md` | 在单个节点上挂载多个设备（如 Switch + Light + Fan + Temperature Sensor），共享 write 回调并按设备名分发。参考 `examples/multi_device/`。 |
| **RainMaker OTA 固件升级**<br>`repos/esp-rainmaker/recipes/ota_update.md` | 启用 RainMaker OTA（推荐 `esp_rmaker_ota_enable_default()`），监听 OTA 事件，配置 OTA 协议（HTTPS/MQTT）与回滚诊断，以及手动 fetch。 |
| **调度与场景**<br>`repos/esp-rainmaker/recipes/scheduling_scenes.md` | 启用 RainMaker 调度（Scheduling）与场景（Scenes）服务，理解触发源（`ESP_RMAKER_REQ_SRC_SCHEDULE` / `_SCENE_ACTIVATE`），并配置最大数量与日光（日出/日落）支持。 |
| **标准服务（时区 / 系统 / 连接性 / 分组）**<br>`repos/esp-rainmaker/recipes/services.md` | 启用 RainMaker 标准服务：时区（timezone）、系统（reboot/factory-reset/wifi-reset）、连接性（Connectivity，含 MQTT LWT）、分组（Groups）。均须在 `esp_rmaker_start()` 之前调用。 |
| **标准设备与参数 Helper**<br>`repos/esp-rainmaker/recipes/standard_devices.md` | 使用 RainMaker 标准 helper API（`esp_rmaker_switch_device_create` / `lightbulb` / `fan` / `temp_sensor` 等）快速创建符合规范的设备，以及标准参数 helper（power/brightness/hue/saturation/temperature/speed 等）。 |
| **Thread 边界路由器服务**<br>`repos/esp-rainmaker/recipes/thread_border_router.md` | 用 `esp_rmaker_thread_br_enable(platform_config)` 把一个 RainMaker 节点变成 **Thread Border Router (TBR)**，让 RainMaker-over-Thread 设备经 BR 上的 **NAT64** 会话接入 RainMaker 云。覆盖 ESP32-S3(主) + ESP32-H2(RCP) 分体、RCP 自动更新、dataset/`ThreadCmd` 控制、LwIP IPv6 编译要求。 |
| **Zigbee 网关与子设备动态映射**<br>`repos/esp-rainmaker/recipes/zigbee_gateway.md` | 把一个 RainMaker 节点变成 **Zigbee 网关**，运行时把每个加入的 Zigbee 终端设备**动态映射**为一个 RainMaker 设备（数据驱动，非静态 `esp_rmaker_device_create`）。覆盖两种加设备方式（预共享 key `ZigBeeAlliance09` 与 install code JSON）、ESP32 + ESP32-H2 RCP 分体、`Add_zigbee_device` 参数、NVS 持久化与重启恢复。 |

## 安全/加密


### `mbedtls` (16 recipes)

| recipe | 摘要 |
|---|---|
| **构建与配置（CMake + mbedtls_config.h）**<br>`repos/mbedtls/recipes/build_config.md` | 用 CMake 构建 Mbed TLS（4.x 仅支持 CMake），通过 `mbedtls_config.h` 与 PSA `crypto_config.h` 裁剪库，用 `scripts/config.py` 程序化修改配置，选用 `configs/` 预设。4.x 起不再支持 Make 与 Visual Studio 工程。 |
| **解析与校验 X.509 证书**<br>`repos/mbedtls/recipes/cert_parse_verify.md` | 从内存或文件加载 PEM/DER 证书，输出证书信息，并用受信任 CA 链校验端实体证书（含吊销列表 CRL）。适用于自建 PKI 校验、证书检查工具等。 |
| **签发 X.509 证书**<br>`repos/mbedtls/recipes/cert_signing.md` | 加载签发者（CA）私钥与证书、主体公钥或 CSR，设置版本/序列号/有效期/主题/扩展，签发 DER 或 PEM 证书。适用于自建 CA 签发端实体证书。 |
| **生成证书签名请求（CSR）**<br>`repos/mbedtls/recipes/csr_generation.md` | 加载私钥，设置主题名、密钥用途、签名算法，生成 DER 或 PEM 格式的 CSR（Certificate Signing Request）。适用于向 CA 申请证书。 |
| **DTLS 客户端**<br>`repos/mbedtls/recipes/dtls_client.md` | 使用 Mbed TLS 建立 DTLS（基于 UDP 的 TLS）客户端，完成握手并收发数据报。适用于物联网设备、低功耗 UDP 加密通信等场景。 |
| **DTLS 服务器（含 HelloVerify Cookie）**<br>`repos/mbedtls/recipes/dtls_server.md` | 使用 Mbed TLS 实现 DTLS 服务器，绑定 UDP 端口，启用 HelloVerifyRequest cookie 防 DoS、接受客户端、握手并回显数据。DTLS 服务器在 UDP 上易受放大攻击，必须启用 cookie 校验。 |
| **加密套件/协议裁剪与错误诊断**<br>`repos/mbedtls/recipes/hardening_error.md` | 收紧协议版本、加密套件、椭圆曲线、签名算法与证书 profile 以满足安全要求；用 `mbedtls_strerror` / `ssl_get_verify_result` 诊断错误。适用于生产环境加固与排障。 |
| **从 3.x 迁移到 4.x（PSA Crypto）**<br>`repos/mbedtls/recipes/psa_migration.md` | 将基于 Mbed TLS 3.x 的代码迁移到 4.x：移除手动 RNG（entropy/ctr_drbg）、删除 `mbedtls_ssl_conf_rng` 调用、改用 `psa_crypto_init()`、用 PSA 配置宏替换旧加密宏。这是 4.x 最大的破坏性变更。 |
| **PSK（预共享密钥）TLS/DTLS**<br>`repos/mbedtls/recipes/psk_tls.md` | 在 TLS/DTLS 中使用预共享密钥（PSK）做认证，无需证书。适用于资源受限、双方已共享密钥的嵌入式设备加密通信。 |
| **会话恢复与票据（Session Tickets）**<br>`repos/mbedtls/recipes/session_resumption.md` | 启用 TLS 会话恢复以加速重连：服务端用会话缓存（`ssl_cache`）或会话票据（`ssl_ticket`），客户端用 `get_session`/`set_session` 或序列化保存会话。会话恢复可避免完整握手，显著降低重连延迟与计算开销。 |
| **SMTP over TLS / STARTTLS 升级（ssl_mail_client）**<br>`repos/mbedtls/recipes/smtp_starttls_client.md` | 用同一底层 TCP 连接先跑明文 SMTP（读 banner、发 `EHLO`、发 `STARTTLS`），收到 `220` 后再把该 socket 就地升级为 TLS——即 STARTTLS 机会式加密模式。升级后在 TLS 通道内继续 SMTP 会话（`AUTH LOGIN` base64 鉴权、`MAIL FROM`/`RCPT TO`/`DATA`）。也可选用 mode=0 直接在连接时即上 TLS（SMTPS，端口 465）。该 STARTTLS 模式可迁移到 IMAP/POP3/LDAP/Postgres 等同类协议。 |
| **TLS 1.3 配置与密钥交换模式选择**<br>`repos/mbedtls/recipes/tls13_configuration.md` | 启用并裁剪 TLS 1.3——选择四种密钥交换模式（pure-PSK / pure-Ephemeral / PSK-Ephemeral 组合）、开关中间箱兼容模式、满足 TLS 1.3 的硬性前置依赖（`MBEDTLS_PSA_CRYPTO_C`、`MBEDTLS_SSL_KEEP_PEER_CERTIFICATE` 必须保持启用），以及运行时用 `mbedtls_ssl_conf_tls13_key_exchange_modes()` 收窄密钥交换集合。这些是独立于 TLS 1.2 配置的 TLS 1.3 专属配置面。 |
| **TLS 1.3 早期数据（0-RTT）收发与拒绝处理**<br>`repos/mbedtls/recipes/tls13_early_data.md` | 在 TLS 1.3 中收发早期数据（0-RTT data）。客户端用 `mbedtls_ssl_write_early_data()` 在握手首飞即发送应用数据、用 `mbedtls_ssl_get_early_data_status()` 判断服务端是否接受；服务端用 `mbedtls_ssl_conf_early_data()` 开启接收、在握手/read/write 返回 `MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA` 时用 `mbedtls_ssl_read_early_data()` 读取。早期数据 API 与普通 `ssl_read/write` 语义差异较大（独有的 `CANNOT_WRITE_EARLY_DATA` / `RECEIVED_EARLY_DATA` / `EARLY_DATA_STATUS_REJECTED` 分支），需单独处理。 |
| **TLS 客户端**<br>`repos/mbedtls/recipes/tls_client.md` | 使用 Mbed TLS 建立 TLS 1.2/1.3 客户端连接，完成握手、读写应用数据并校验服务器证书。适用于 HTTPS、MQTT over TLS 等需要加密通道的客户端场景。 |
| **多进程 fork() 并发 TLS 服务器**<br>`repos/mbedtls/recipes/tls_fork_server.md` | 用 POSIX `fork()` 实现"每客户端一进程"的并发 TLS 服务器。与 `tls_server.md` 的单连接模型、`ssl_pthread_server.c` 的每客户端一线程模型不同，进程级隔离让单个客户端的崩溃不会污染监听上下文或其他客户端。需要在 accept 后 `fork()`，父进程关 client_fd 继续监听，子进程关 listen_fd 专属服务该客户端。仅限 Unix/POSIX 环境（不支持 Windows）。 |
| **TLS 服务器**<br>`repos/mbedtls/recipes/tls_server.md` | 使用 Mbed TLS 实现 TLS 服务器（监听端口、装载服务器证书与私钥、接受客户端、握手并收发数据）。适用于 HTTPS 服务端、TLS 设备网关等场景。 |

### `TF-PSA-Crypto` (11 recipes)

| recipe | 摘要 |
|---|---|
| **AEAD（AES-GCM / AES-CCM / ChaCha20-Poly1305）**<br>`repos/TF-PSA-Crypto/recipes/aead.md` | 使用 PSA Crypto API 进行带认证的加密/解密(AEAD)。涵盖一次性 `psa_aead_encrypt`/`psa_aead_decrypt` 与分段 `psa_aead_*_setup/set_nonce/update_ad/update/finish/verify`，附加数据(AD)处理、短标签。 |
| **非对称签名与密钥协商（ECDSA / RSA / ECDH）**<br>`repos/TF-PSA-Crypto/recipes/asymmetric.md` | 使用 PSA Crypto API 进行非对称签名/验签（ECDSA、确定性 ECDSA、RSA-PSS/PKCS1v15）和密钥协商（ECDH/FFDH 原始协商），以及用 `psa_export_public_key` 导出公钥。 |
| **构建与配置（CMake / crypto_config.h）**<br>`repos/TF-PSA-Crypto/recipes/build_and_config.md` | 使用 CMake 构建 TF-PSA-Crypto（静态/共享库、子项目、`find_package`），通过 `include/psa/crypto_config.h` 或 `configs/` 预设选择启用的加密机制，以及用 `scripts/config.py` 编程式编辑配置。 |
| **对称加密（AES-CBC / CTR / ChaCha20）**<br>`repos/TF-PSA-Crypto/recipes/cipher.md` | 使用 PSA Crypto API 进行对称加密/解密。涵盖一次性 `psa_cipher_encrypt`/`psa_cipher_decrypt` 与分段 `psa_cipher_*_setup/generate_iv/set_iv/update/finish`，AES-CBC/PKCS7、CTR、ChaCha20 等模式。 |
| **哈希（SHA-256 / SHA-3 等）**<br>`repos/TF-PSA-Crypto/recipes/hashing.md` | 使用 PSA Crypto API 计算消息摘要。涵盖一次性(one-shot) `psa_hash_compute` 和分段(multi-part) `psa_hash_setup/update/finish`，以及 `psa_hash_clone` 克隆与 `psa_hash_verify` 比对。 |
| **库初始化与生命周期管理**<br>`repos/TF-PSA-Crypto/recipes/init_and_lifecycle.md` | 正确初始化 PSA Crypto 库、在程序结束时释放资源。这是使用任何 `psa_*` API 之前必须完成的第一步。 |
| **密钥派生（HKDF / PBKDF2 / TLS12-PRF）**<br>`repos/TF-PSA-Crypto/recipes/key_derivation.md` | 使用 PSA Crypto API 的密钥派生框架从主密钥派生新密钥或密钥材料。涵盖 HKDF、PBKDF2、TLS12-PRF 的输入步骤、`psa_key_derivation_output_key` 直接派生密钥对象、以及密钥阶梯(key ladder)模式。 |
| **密钥管理（生成 / 导入 / 导出 / 销毁）**<br>`repos/TF-PSA-Crypto/recipes/key_management.md` | 使用 `psa_key_attributes_t` 描述密钥，通过 `psa_generate_key` / `psa_import_key` 创建密钥，`psa_export_key` / `psa_export_public_key` 取出密钥，`psa_destroy_key` 释放。涵盖易失(volatile)与持久(persistent)两种生命周期。 |
| **MAC（HMAC / AES-CMAC）**<br>`repos/TF-PSA-Crypto/recipes/mac.md` | 使用 PSA Crypto API 计算消息认证码。涵盖 HMAC（任意哈希）与 AES-CMAC，一次性 `psa_mac_compute`/`psa_mac_verify` 与分段 `psa_mac_sign_setup/update/finish`，以及签名(sign)与验签(verify)的对称用法。 |
| **迁移旧 mbedtls_* 密码 API 到 PSA**<br>`repos/TF-PSA-Crypto/recipes/migration_to_psa.md` | 把基于 Mbed TLS 3.x `mbedtls_*`（AES/SHA/HMAC/CMAC/RSA/ECDSA/ECDH/HKDF/PBKDF2 等）的密码代码迁移到 TF-PSA-Crypto 的 `psa_*` API。涵盖 per-algorithm 旧→新函数对照、RNG 回调移除、配置文件拆分（`MBEDTLS_xxx_C` → `PSA_WANT_*` + `MBEDTLS_PSA_ACCEL_*`）、错误码合并、PK 模块变化与 PAKE 接口更新。 |
| **PSA 密码处理器驱动开发与 driver-only 构建**<br>`repos/TF-PSA-Crypto/recipes/psa_driver_development.md` | 为 TF-PSA-Crypto 编写并集成 PSA cryptoprocessor 驱动（transparent 加速器 / opaque 安全元件），以及配置 driver-only 构建（某机制只由驱动提供、移除内置实现以省代码体积）。涵盖 JSON 驱动描述文件、自动生成 vs 手动集成的入口点、`PSA_WANT_*` + `MBEDTLS_PSA_ACCEL_*` + `MBEDTLS_xxx_C` 三者关系，并以仓库自带的 ESP AES/SHA 硬件加速驱动 JSON 为实例。 |

### `esp_secure_cert_mgr` (10 recipes)

| recipe | 摘要 |
|---|---|
| **用 configure_esp_secure_cert.py 生成 / 签名 / 解析分区**<br>`repos/esp_secure_cert_mgr/recipes/generate_partition_csv.md` | 主机端使用 `tools/configure_esp_secure_cert.py` 工具，通过命令行参数或 CSV 配置文件生成 `esp_secure_cert.bin`，可选签名（Secure Boot V2）与解析已有镜像。这是出厂烧录与 QEMU 测试的标准流程。 |
| **在内存缓冲区生成分区镜像（Buffer 模式）**<br>`repos/esp_secure_cert_mgr/recipes/generate_partition_in_buffer.md` | 主机端工具或测试场景下，不直接写 flash，而是在 RAM 缓冲区中拼装出完整的 `esp_secure_cert` 分区镜像，再整体烧录或下发。使用 `ESP_SECURE_CERT_WRITE_MODE_BUFFER` 模式。 |
| **写入 HMAC 派生 ECDSA 私钥（私钥不落盘）**<br>`repos/esp_secure_cert_mgr/recipes/hmac_ecdsa_derivation.md` | 利用硬件 HMAC 外设 + PBKDF2-HMAC-SHA256 实时派生 ECDSA 私钥。分区里只存 salt，私钥永不在 flash 中。演示写入配置与（可选）一步生成并烧录 eFuse 的完整流程。 |
| **遍历 / 列出 / 通用查询 TLV 条目**<br>`repos/esp_secure_cert_mgr/recipes/iterate_tlv_entries.md` | 当分区中同类型有多条目、或需要读取自定义 TLV（`USER_DATA_1..5`）时，使用通用 TLV API 按 type+subtype 查询、用迭代器遍历全部条目、或调用 `esp_secure_cert_list_tlv_entries()` 打印清单。 |
| **对 esp_secure_cert 分区做 Fail-Safe OTA 升级**<br>`repos/esp_secure_cert_mgr/recipes/ota_update_partition.md` | 远程更新 `esp_secure_cert` 分区（轮换证书/密钥）。演示三种暂存策略（unallocated space / passive OTA / direct）、用 NVS 记录恢复点实现断电回滚，以及 `esp_secure_cert_tlv_set_partition()` 切换活动分区做校验。 |
| **读取设备证书 / CA 证书 / 私钥**<br>`repos/esp_secure_cert_mgr/recipes/read_certs_and_key.md` | 在固件中通过便捷 API 读取 `esp_secure_cert` 分区里的设备证书、CA 证书与（非 DS 场景下的）私钥，并正确释放内存。适用于 TLS 客户端加载凭据等场景。 |
| **使用 Digital Signature (DS) 外设获取私钥上下文**<br>`repos/esp_secure_cert_mgr/recipes/use_ds_peripheral.md` | 当设备预置时启用 RSA Digital Signature (DS) 外设（ESP32-S2/S3/C3/C5 等），私钥以密文形式存于 `esp_secure_cert` 分区。本配方演示如何在固件中获取 `esp_ds_data_ctx_t` 并喂给 TLS / mbedTLS / PSA 完成签名，同时校验密文有效性。 |
| **使用 ECDSA 外设（eFuse 私钥）签名**<br>`repos/esp_secure_cert_mgr/recipes/use_ecdsa_peripheral.md` | 当私钥类型为 `ESP_SECURE_CERT_ECDSA_PERIPHERAL_KEY`（私钥存于 eFuse block，由硬件 ECDSA 外设使用），演示如何判断类型、取 efuse block id，并用 mbedTLS/PSA 完成签名与验签。 |
| **启动期校验分区完整性与签名**<br>`repos/esp_secure_cert_mgr/recipes/verify_partition.md` | 设备启动时对 `esp_secure_cert` 分区做两层校验：完整性 TLV（SHA256，工具生成时自动追加）与签名块（Secure Boot V2，RSA/ECDSA）。校验应在解析分区内容之前进行。 |
| **运行时写入自定义 TLV 用户数据（Flash 模式）**<br>`repos/esp_secure_cert_mgr/recipes/write_user_data.md` | 在设备运行期向 `esp_secure_cert` 分区写入自定义 TLV（`ESP_SECURE_CERT_USER_DATA_1..5`）。演示 flash 模式的擦除、写入、读回校验完整流程，含写配置与备份/恢复策略。 |

## 其他组件库


### `esp-bist` (12 recipes)

| recipe | 摘要 |
|---|---|
| **时钟测试：32 kHz 外部晶振 + 40 MHz 主晶振**<br>`repos/esp-bist/recipes/clock_test.md` | 用 `bist_ext_crystal_fail_test()` 通过 XT WDT 监测外部 32.768 kHz 晶振（仅 `SOC_XT_WDT_SUPPORTED` 的 SoC，如 C3）；用 `bist_main_crystal_test()` 测 40 MHz 主晶振相对 32 kHz 参考的频率漂移（IEC 60730 组件 3）。 |
| **CPU 寄存器与 CSR 完整性测试**<br>`repos/esp-bist/recipes/cpu_register_csr_test.md` | 用 `bist_cpu_regs_test()` 校验 RISC-V 通用寄存器（X1–X31），用 `bist_cpu_csr_regs_test()` 校验 CSR（trap / PMP / PMA / mexstatus），并按 IEC 60730 组件 1.1 在运行时主循环中周期执行。 |
| **GPIO 数字 I/O 合理性测试**<br>`repos/esp-bist/recipes/gpio_test.md` | 用 `bist_gpio_output_test()` 验证引脚能被可靠拉高/拉低并回读；用 `bist_gpio_input_test()` 验证输入引脚能读到预期外部电平（IEC 60730 组件 7.1）。包含无效 GPIO 号与各 SoC 引脚映射。 |
| **BIST MCP 服务器：接入 AI 助手（Cursor / VS Code）**<br>`repos/esp-bist/recipes/mcp_server_setup.md` | 把 ESP-BIST 仓库自带的 MCP 服务器（`mcp-server/server.py`，FastMCP + BM25）接到 Cursor 或 VS Code，让 AI 助手通过 6 个工具（`search_bist_docs`、`get_api_reference`、`get_architecture_info`、`search_kconfig_options`、`search_source_code`、`get_supported_socs`）直接查询仓库的真实文档/头文件/Kconfig/源码，而不是凭记忆猜测；并用 `ingest.py` 在内容变更时重新生成 `data/*.json` 快照（CI 由 `mcp_data_drift` 强制）。 |
| **性能指标采集：RISC-V Performance Counter CSRs**<br>`repos/esp-bist/recipes/metrics_collection.md` | 用 `bist_metrics.h` 提供的 `BIST_METRICS_*` 宏，基于 RISC-V 性能计数器 CSR（`0x7e0` PCER / `0x7e1` PCMR / `0x7e2` PCCR）测量 BIST 测试的周期数、指令数、load/store、分支、hazard 等微架构事件。 |
| **程序计数器（PC）与栈溢出检测**<br>`repos/esp-bist/recipes/pc_and_stack_test.md` | 用 `bist_pc_test()` 覆盖 PC 寄存器位（IRAM/Flash/RTC 放置），用 `bist_cpu_stack_overflow_init/check/test` 做 0xDEADBEEF 哨兵式栈溢出检测（IEC 60730 组件 1.3 与 4.2）。 |
| **项目搭建：构建并运行第一个 BIST 应用**<br>`repos/esp-bist/recipes/project_setup.md` | 从零创建一个 ESP-BIST 应用工程，配置 CMake/Ninja 构建、MCUboot 引导、`bist.conf`，并在 QEMU 或真机上运行。 |
| **RAM March 测试与 Flash CRC32 完整性测试**<br>`repos/esp-bist/recipes/ram_flash_test.md` | 用 `bist_ram_test_march_a()` / `bist_ram_test_march_x()` 做非破坏式 RAM 完整性测试（IEC 60730 4.2）；用 `bist_flash_test()` 比对运行时 CRC32 与后处理注入的参考值（IEC 60730 4.1）。 |
| **安全认证证据工作流：QEMU GDB 故障注入 + 追溯矩阵 + 覆盖率 + 工具鉴定**<br>`repos/esp-bist/recipes/safety_validation_workflow.md` | 按 `docs/en/software_validation.rst` 的范式，用 QEMU（`qemu-system-riscv32 -icount 3`）+ GDB（`:1234`）对 BIST 测试做确定性故障注入，用 pytest 驱动断言 PASS/FAIL，产出 JUnit XML；再把结果串进 `test_traceability_matrix.rst` → `coverage_analysis.rst` → `tool_qualification.rst` → `safety_case_summary.rst` 的 IEC 60730 Class B 证据链。 |
| **完整 IEC 60730 集成：post-boot + 运行时自检 + 窗口看门狗**<br>`repos/esp-bist/recipes/standalone_integration.md` | 按 `samples/standalone/main.c` 的范式，把 BIST 库完整集成进一个安全关键应用：启动自检、运行时周期自检、窗口看门狗、fail-safe 退出。 |
| **看门狗测试：复位验证 + 窗口看门狗**<br>`repos/esp-bist/recipes/watchdog_windowed_test.md` | 用 `bist_wdt_test()` 通过双启动序列验证 MWDT 能触发复位（IEC 60730 6.3）；用 `wdt_init()` + `wdt_init_windowed()` + `wdt_feed()` 实现下溢/溢出双向监控，并按 `tests/windowed_wdt_test` 的三个用例验证。 |
| **Zephyr RTOS 集成：HP 主核 + LP 辅核，sysbuild + mbox 通信**<br>`repos/esp-bist/recipes/zephyr_integration.md` | 按 `samples/zephyr/` 的范式，用 `west build ... --sysbuild` 在 ESP32-C6 上同时构建 HP（主）核 Zephyr 应用与 LP（辅）核 BIST 自检固件，通过 mbox IPC 在两核之间收发 ping/pong，并在 LP 核上用 `ulp_lp_core_intr_disable()/enable()` 包裹 `bist_cpu_regs_test()` / `bist_cpu_csr_regs_test()` / `bist_ram_test_march_x()`。 |

### `esp-serial-flasher` (18 recipes)

| recipe | 摘要 |
|---|---|
| **用 flasher stub 连接（解锁高波特率 / deflate / 快速读）**<br>`repos/esp-serial-flasher/recipes/connect_with_stub.md` | 用 `esp_loader_connect_with_stub()` 连接目标，把 ROM bootloader 替换为功能更强的 `esp-flasher-stub`，从而支持更高波特率、>2MB flash、deflate 压缩写、快速 flash 读。仅 serial(SLIP) 接口支持。 |
| **为新主机平台实现自定义 port**<br>`repos/esp-serial-flasher/recipes/custom_port.md` | 当内置 port（ESP32/STM32/Zephyr/Pico/Linux）不覆盖你的主机时，实现自己的 `esp_loader_port_ops_t` vtable，把 ESP Serial Flasher 作为 external library（`PORT=USER_DEFINED`）集成。 |
| **deflate 压缩烧录（节省传输时间）**<br>`repos/esp-serial-flasher/recipes/deflate_flash.md` | 把目标固件预先 zlib 压缩，用 `esp_loader_flash_deflate_start/write/finish` 烧录压缩流，目标端 stub 解压写入，显著减少 UART 传输量。需 stub 连接，且 deflate 路径不内部做 MD5，需单独 verify。 |
| **整片擦除 / 区域擦除**<br>`repos/esp-serial-flasher/recipes/erase_flash.md` | 用 `esp_loader_flash_erase()` 擦除目标整片 flash，或用 `esp_loader_flash_erase_region()` 擦除指定 4KB 对齐区间。常用于烧录前清场、安全擦除。 |
| **快速重烧（已知 MD5 比对，跳过相同分区）**<br>`repos/esp-serial-flasher/recipes/fast_reflash_md5.md` | 用 `esp_loader_flash_verify_known_md5()` 比对目标的明文 MD5 与已知 MD5，只对不匹配的分区重新烧录，避免每次全量烧录。适合 OTA 旁路、量产复核、固件去重。 |
| **多分区烧录（手写 start/write/finish）**<br>`repos/esp-serial-flasher/recipes/flash_partitions.md` | 不依赖 `example_common` helper，手写 `esp_loader_flash_start` → 循环 `esp_loader_flash_write` → `esp_loader_flash_finish` 的标准烧录流程，明确控制 block_size、进度与 MD5 校验。 |
| **读取目标信息（MAC / flash 容量 / security info / 芯片型号）**<br>`repos/esp-serial-flasher/recipes/get_target_info.md` | 连接目标后读取芯片型号、MAC 地址、flash 容量、安全信息（secure boot / flash encryption / JTAG / USB 等）。常用于烧录前校验目标身份、安全状态盘点。 |
| **在 ESP-IDF 项目中集成（managed component）**<br>`repos/esp-serial-flasher/recipes/idf_component_setup.md` | 把 ESP Serial Flasher 作为 managed component 添加到 ESP-IDF 项目，配置 port 编译选项，设置 sdkconfig.defaults，并接入目标固件 bin2array 流程。 |
| **Linux 主机烧录（PC / 树莓派，DTR/RTS 或 libgpiod 复位）**<br>`repos/esp-serial-flasher/recipes/linux_host.md` | 在 Linux 主机（PC、树莓派 4/5、BeagleBone 等）上用 `linux_port` 经 `/dev/ttyUSB*` 或 `/dev/ttyACM*` 烧录 ESP 目标。复位/BOOT 可经 USB-UART 桥的 DTR/RTS 自动复位，或经 libgpiod 控制 GPIO。 |
| **通过 UART 把程序下载到 RAM 并运��**<br>`repos/esp-serial-flasher/recipes/load_ram_uart.md` | 用 `esp_loader_mem_start/write/finish` 把可执行镜像直接下载到目标 RAM 并跳转执行（不烧 flash）。常用于临时调试、RAM-only 测试程序。复用 `example_common.c` 的 `load_ram_binary()` helper。 |
| **Raspberry Pi Pico 主机烧录（Pico SDK / RP2040 / RP2350 ARM 与 RISC-V）**<br>`repos/esp-serial-flasher/recipes/pi_pico_host.md` | 用 Raspberry Pi Pico（RP2040）或 Pico 2（RP2350）作主机经 `uart1` 烧录 ESP 目标，走内置 `pi_pico_port`（`PORT=PI_PICO`）。Pico 2 的 RP2350 可选 ARM 或 RISC-V 核，由 `PICO_PLATFORM` 决定，两套交叉编译器**不可互换**。镜像经 `.uf2` 拖拽烧入 Pico。 |
| **从目标 flash 读取数据**<br>`repos/esp-serial-flasher/recipes/read_flash.md` | 用 `esp_loader_flash_read()` 从目标 flash 读取指定地址/长度的数据到主机缓冲区，并与写入数据比对校验。仅 serial(SLIP) 接口支持。 |
| **通过 SDIO 接口烧录（实验性）**<br>`repos/esp-serial-flasher/recipes/sdio_flash.md` | 用 ESP32 主机的 SDIO host（`esp32_sdio_port`）+ `esp_loader_init_sdio()` 烧录目标。SDIO 连接时自动上传 `esp-flasher-stub`，支持完整 stub 命令（含 deflate）。目前仅 ESP32-C5 / C6 作目标。SDIO 不支持改速率。 |
| **通过 SPI 接口下载 RAM（仅 RAM 下载）**<br>`repos/esp-serial-flasher/recipes/spi_load_ram.md` | 用 ESP32 主机的 SPI 外设（`esp32_spi_port`）+ `esp_loader_init_spi()` 把程序下载到目标 RAM 并运行。SPI 接口**只支持 RAM 下载**，不支持 flash 写/读/erase。支持标准 SPI 与 Quad-SPI。 |
| **STM32 主机烧录（STM32CubeMX / STM32 HAL 内置 port）**<br>`repos/esp-serial-flasher/recipes/stm32_host.md` | 用 STM32（任意 HAL 系列）作主机经 UART 烧录 ESP 目标，走内置 `stm32_port`（`PORT=STM32`）。与 `custom_port.md` 的 USER_DEFINED 不同：STM32 port 不实现 init 回调，而是要求外设由 CubeMX **预先生成并初始化**，调用者只填 `huart` 句柄和 BOOT/RESET 的 GPIO 端口/引脚。无现成工程，按 STM32CubeMX 流程生成。 |
| **UART 主机连接并烧录 ESP 目标（ROM bootloader）**<br>`repos/esp-serial-flasher/recipes/uart_connect_flash.md` | 在 ESP-IDF 主机（或任意 serial port）上，用 UART 接口把目标 ESP 芯片置入下载模式、连接（ROM bootloader，非 stub）、提速、多分区烧录并复位。这是最常用、最基础的烧录场景。 |
| **通过 USB CDC-ACM 主机端口烧录（USB Host）**<br>`repos/esp-serial-flasher/recipes/usb_cdc_acm.md` | 用 ESP32 主机的 USB OTG（USB Host）经 CDC-ACM 类烧录目标（目标的 USB Serial/JTAG 或 USB OTG 外设）。无需额外 TX/RX/BOOT 线，单根 USB 即可。port 可在断开后经 `esp_loader_init_serial()` 重连。 |
| **Zephyr 主机烧录（west module / device tree / prj.conf / esf shell）**<br>`repos/esp-serial-flasher/recipes/zephyr_host.md` | 把 `esp-serial-flasher` 作为 Zephyr **west module** 集成，用 device-tree 驱动的 `espressif,esp-loader` 节点描述 UART/复位/BOOT 引脚与波特率，应用代码经 `esp_loader_from_device()` 取 loader，可选启用交互式 `esf` shell。与其它 port 的核心差别：**不手填 port 结构体**——一切配置来自 DTS overlay，连接参数也由 driver 提供。 |


**合计 554 个 recipe。**
