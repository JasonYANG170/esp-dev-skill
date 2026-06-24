# ESP-IDF 示例工程索引（真实路径）

> 全部为 ESP-IDF 仓库 `examples/` 下真实存在的目录（已校验）。路径相对 `D:/esp-skill/espressif-repos/esp-idf/examples/`。

## 起步

| 路径 | 说明 |
|---|---|
| `get-started/hello_world` | 最小工程，打印芯片信息并倒计时重启（含 `MINIMAL_BUILD`） |
| `get-started/blink` | GPIO / 可寻址 LED（led_strip RMT/SPI 后端）闪烁 |

## Wi-Fi

| 路径 | 说明 |
|---|---|
| `wifi/getting_started/station` | STA 模式连接 AP（含 WPA3 选项、事件组等待） |
| `wifi/getting_started/softAP` | SoftAP 热点（含 STA 接入事件） |
| `wifi/scan` | 扫描周围 AP |
| `wifi/espnow` | ESPNOW 点对点通信 |
| `wifi/power_save` | Wi-Fi 省电模式（minimum modem sleep） |
| `wifi/roaming` | 漫游（BSSID 切换） |

## 外设

| 路径 | 说明 |
|---|---|
| `peripherals/gpio/generic_gpio` | GPIO 输入/输出/中断（ISR 服务 + 队列） |
| `peripherals/uart/uart_echo` | UART 回环收发 |
| `peripherals/uart/uart_async_rxtxtasks` | UART 异步收发任务 + 事件队列 |
| `peripherals/i2c/i2c_basic` | 新总线式 I2C 主机初始化/读写/deinit |
| `peripherals/i2c/i2c_eeprom` | I2C EEPROM 读写（含分页） |
| `peripherals/spi_master/hd_eeprom` | SPI 主机半双工 EEPROM |
| `peripherals/spi_master/lcd` | SPI LCD（DMA + 全屏刷新） |
| `peripherals/ledc/ledc_basic` | LEDC 基础 PWM + 运行时调占空比 |
| `peripherals/ledc/ledc_fade` | LEDC 硬件渐变 |
| `peripherals/timer_group/gptimer` | GPTimer 周期告警、auto-reload、读写计数 |
| `peripherals/timer_group/gptimer_capture_hc_sr04` | GPTimer 捕获模式（HC-SR04 测距） |
| `peripherals/adc/oneshot_read` | ADC oneshot 单次采样 |
| `peripherals/mcpwm` | 电机控制 PWM（MCPWM） |
| `peripherals/rmt` | RMT（红外/WS2812/自定义协议） |
| `peripherals/touch_sensor` | 触摸传感器 |
| `peripherals/i2s` | I2S 音频/数据 |
| `peripherals/lcd` | 并行/SPI LCD 驱动 |

## 存储

| 路径 | 说明 |
|---|---|
| `storage/nvs/nvs_rw_value` | NVS 整数读写（含升级擦除） |
| `storage/nvs/nvs_rw_blob` | NVS blob（结构体）读写 |
| `storage/nvs/nvs_iteration` | 遍历 NVS 命名空间键 |
| `storage/partition_api/partition_find` | `esp_partition_find_first` / 迭代分区 |
| `storage/partition_api/partition_ops` | 分区 read/write/erase |
| `storage/partition_api/partition_mmap` | 分区内存映射读 |
| `storage/spiffs` | SPIFFS 文件系统 |
| `storage/fatfs` | FATFS（SD 卡/内部） |
| `storage/wear_levelling` | wear_levelling 均衡磨损层 |

## 系统

| 路径 | 说明 |
|---|---|
| `system/deep_sleep` | 深度睡眠 + 定时器/ext0/ext1/GPIO 唤醒源 |
| `system/freertos/real_time_stats` | FreeRTOS 任务运行时统计 |
| `system/app_trace` | 应用跟踪（JTAG/SEGGER SystemView） |

## OTA

| 路径 | 说明 |
|---|---|
| `system/ota/simple_ota_example` | HTTPS OTA（高层 `esp_https_ota`） |
| `system/ota/native_ota_example` | 底层 `esp_ota_begin/write/end` 流程 |
| `system/ota/advanced_https_ota` | 带进度/状态的高级 OTA |

## 协议

| 路径 | 说明 |
|---|---|
| `protocols/http_request` | HTTP GET/POST 请求 |
| `protocols/https_request` | HTTPS 请求（mbedTLS） |
| `protocols/mqtt` | MQTT 客户端（TCP/TLS/WebSocket） |

## 蓝牙

| 路径 | 说明 |
|---|---|
| `bluetooth/bluedroid/ble/gatt_server_service_table` | 属性表式 GATT Server（推荐，含 notify/indicate） |
| `bluetooth/bluedroid/ble/gatt_server` | 逐个 add_char 式 GATT Server（旧式） |
| `bluetooth/bluedroid/ble/gatt_client` | 单连接 GATT Client（扫描+发现+读写+notify） |
| `bluetooth/bluedroid/ble/gattc_multi_connect` | 多连接 GATT Client |
| `bluetooth/bluedroid/ble/gatt_security_client` / `gatt_security_server` | BLE 安全/加密连接 |
| `bluetooth/bluedroid/ble/ble_spp_server` / `ble_spp_client` | BLE SPP（BLE 上的串口透传） |
| `bluetooth/bluedroid/ble/ble_ibeacon` / `ble_eddystone_sender` / `ble_eddystone_receiver` | 信标（iBeacon / Eddystone） |
| `bluetooth/bluedroid/classic_bt/bt_spp_acceptor` / `bt_spp_initiator` | 经典蓝牙 SPP（CB 模式） |
| `bluetooth/bluedroid/classic_bt/bt_spp_vfs_acceptor` / `bt_spp_vfs_initiator` | 经典蓝牙 SPP（VFS 模式，高吞吐） |
| `bluetooth/bluedroid/classic_bt/a2dp_sink_stream` / `a2dp_sink_stream_aac` | A2DP Sink 音频接收 + I2S 输出 |
| `bluetooth/bluedroid/classic_bt/a2dp_source` / `a2dp_source_aac` | A2DP Source 音频发送 |
| `bluetooth/bluedroid/classic_bt/avrcp_absolute_volume` / `avrcp_ct_metadata` / `avrcp_ct_cover_art` | AVRCP 控制（音量/元数据/封面） |
| `bluetooth/bluedroid/classic_bt/hfp_ag` / `hfp_hf` | HFP 免提（AG/HF） |
| `bluetooth/bluedroid/classic_bt/bt_discovery` / `bt_l2cap_client` / `bt_l2cap_server` | 经典蓝牙发现与 L2CAP |
| `bluetooth/esp_ble_mesh/onoff_models/onoff_server` / `onoff_client` | BLE Mesh Generic OnOff 模型 |
| `bluetooth/esp_ble_mesh/provisioner` | BLE Mesh 配网器（Provisioner） |
| `bluetooth/esp_ble_mesh/fast_provisioning` | BLE Mesh 快速配网 |
| `bluetooth/esp_ble_mesh/sensor_models` / `vendor_models` | BLE Mesh 传感器 / 厂商模型 |
| `bluetooth/esp_ble_mesh/wifi_coexist` | BLE Mesh 与 Wi-Fi 共存 |
| `bluetooth/ble_get_started/bluedroid/Bluedroid_GATT_Server` / `Bluedroid_Beacon` / `Bluedroid_Connection` | Bluedroid 分步教学 |
| `bluetooth/ble_get_started/nimble/NimBLE_*` | NimBLE 对应教学版 |
| `bluetooth/nimble/BLE_ANCS` | NimBLE ANCS（Apple 通知中心） |

## 网状网络 / 以太网

| 路径 | 说明 |
|---|---|
| `mesh/internal_communication` | ESP-WIFI-MESH 节点间 P2P 收发（含 LED 控制） |
| `mesh/ip_internal_network` | ESP-WIFI-MESH 根节点 IP 上行（mesh 内部 IP 路由） |
| `mesh/manual_networking` | 手动选父节点组网 |
| `ethernet/basic` | 内部 MAC + 通用 PHY（RMII/RGMII，含 YT8531 寄存器配置示例） |
| `ethernet/iperf` | 以太网吞吐测试 |
| `ethernet/ptp` | IEEE 1588 PTP 精密时钟 |

> 完整列表见仓库 `examples/README.md` 与各子目录 `README.md`。复制样例到工程目录后再修改，避免直接在 ESP-IDF 仓库内编辑。
