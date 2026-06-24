# ESP RainMaker 官方示例索引

> 全部路径来自 `esp-rainmaker/examples/` 真实目录。每个示例目录下均有 `main/app_main.c` 与 `README.md`。

| 示例路径 | 说明 |
|---|---|
| `examples/switch/` | 标准 Switch 节点入门示例。包含 NVS、配网、node_init、设备创建、write 回调、OTA、调度、场景、Insights 全流程。最常作为模板。 |
| `examples/led_light/` | LED 灯具示例。演示多参数（Power/Brightness/Hue/Saturation）、bulk_write_cb、system service、Connectivity/Groups 服务、Command-Response 自定义命令、direct MQTT 发布。 |
| `examples/fan/` | 风扇示例。标准 Fan 设备 + Speed 参数。 |
| `examples/temperature_sensor/` | 温度传感器示例。temp_sensor 设备 + 温度主动上报。 |
| `examples/multi_device/` | 多设备节点。单节点挂 Switch + Light + Fan + Temperature Sensor，共享 write 回调按设备名分发，演示设备属性（Serial Number/MAC）。 |
| `examples/gpio/` | GPIO 控制示例，演示将 GPIO 状态映射为 RainMaker 参数。 |
| `examples/ethernet_switch/` | 以太网 Switch 示例。以太网为主网络，`CONFIG_ESP_RMAKER_NO_CLAIM=y`，可选 `EXAMPLE_ENABLE_WIFI` 双网络（先连上的胜出）。PHY/RMII GPIO 在 `components/app_ethernet/Kconfig.projbuild`；on-network chal_resp 在 `main/app_on_network_test.c`（支持独立 HTTP 或复用 local control 两种实现，互斥）。 |
| `examples/homekit_switch/` | RainMaker + HomeKit 集成（依赖 esp-homekit-sdk）。 |
| `examples/thread_br/` | Thread 边界路由器示例。`esp_rmaker_thread_br_enable(platform_config)` 启用 BR 服务，让 RainMaker-over-Thread 设备经 NAT64 上云。硬件需 ESP Thread Border Router Board（ESP32-S3 主控 + ESP32-H2 RCP，UART 互联）。配网后用 `esp-rainmaker-cli` 写 `TBRService.ThreadCmd=1` 或 `ActiveDataset` 起 Thread 网络。构建需 `OPENTHREAD_BORDER_ROUTER=y`、`LWIP_IPV6_NUM_ADDRESSES`（IDF 5.3.1+ 要求 12）、mbedTLS EC-JPAKE/DTLS/CMAC。 |
| `examples/zigbee_gateway/` | Zigbee 网关示例。运行时把加入的 Zigbee 终端动态映射为 RainMaker 设备（`simple_desc_cb` 按 HA device id 分派，加完调 `esp_rmaker_report_node_details`）。两种加设备方式：预共享 key `ZigBeeAlliance09`（`Add_zigbee_device` 布尔开关，180s 入网窗口）或 install code（`CONFIG_ZIGBEE_INSTALLCODE_ENABLED=y`，`Add_zigbee_device` 字符串 + `ESP_RMAKER_UI_QR_SCAN`）。硬件：ESP32 主控（`ZB_ZCZR` 协调器）+ ESP32-H2 RCP。子设备映射：On/Off Light→lightbulb，IAS Zone→contact-sensor。 |
| `examples/camera/` | 摄像头示例（WebRTC/KVS）。含 `standalone` 与 `split_mode` 两种；`components/rmaker_camera/` 提供 camera 组件。 |
| `examples/rainmaker_controller/` | 控制器节点。`esp_rmaker_controller_enable` + `CONFIG_ENABLE_RM_USER_HELPER_API=y`，把 ESP 设备变成 RainMaker 控制器，UART console 提供 CLI 控制其它节点：`getnodes`/`getnodedetails`/`getparams`/`setparams`/`getnodeconfig`/`getnodestatus`/`getschedules`/`setschedule`/`removenode`/`getheapstatus`。User API 用 `esp_rmaker_auth_service` 的 refresh token 登录（无需硬编码凭据）。 |
| `examples/matter/` | Matter + RainMaker 集成示例（需 esp-matter，ESP-IDF v5.4.1）。仅 iOS App v3.0.0+。 |

## 示例共用组件（`examples/common/`）

| 组件路径 | 说明 |
|---|---|
| `examples/common/rmaker_app_network/` | 网络/配网封装（Wi-Fi + Thread）。提供 `app_network_init/start/set_custom_mfg_data`，含 `APP_NETWORK_EVENT` 事件。 |
| `examples/common/rmaker_app_insights/` | ESP Insights 集成（需 `CONFIG_ESP_INSIGHTS_ENABLED=y`）。 |
| `examples/common/rmaker_app_reset/` | 按键复位（factory/wifi reset）封装。 |
| `examples/common/rmaker_user_api/` | RainMaker User API 封装：core 层 `app_rmaker_user_api.h`（`app_rmaker_user_api_generic` 任意端点）+ helper 层 `app_rmaker_user_helper_api.h`（需 `CONFIG_ENABLE_RM_USER_HELPER_API=y`，提供 `get_nodes_list`/`get_node_params`/`set_node_params`/`get_node_config`/`get_node_connection_status`/`set_node_mapping` 等）。控制器示例基于此组件。 |
| `examples/common/rmaker_matter_controller/` | Matter controller 集成组件。 |
| `examples/common/gpio_button/` | GPIO 按键驱动（`iot_button`）。 |
| `examples/common/ledc_driver/` | LEDC PWM 驱动（灯具调光用）。 |

## 组件位置

- RainMaker Agent 组件：`components/esp_rainmaker/`
  - 公开头文件：`components/esp_rainmaker/include/*.h`
  - Kconfig：`components/esp_rainmaker/Kconfig.projbuild`
  - 服务器证书：`components/esp_rainmaker/server_certs/`（claiming/mqtt/ota）
  - 组件清单：`components/esp_rainmaker/idf_component.yml`（version 1.15.0，依赖 IDF >=5.1）
- 主机工具：`tools/`（Python CLI 等，非固件范围）
