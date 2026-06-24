# ESP-Hosted-MCU 示例索引

> 所有路径均为 `esp-hosted-mcu` 仓库 `examples/` 下的真实目录。描述取自各例程 README 的标题行。

## 传输与基础设施

| 示例路径 | 一句话描述 |
|---|---|
| `examples/host_transport_config` | 运行期配置传输介质（SDIO/SPI/UART）的代码范式 |
| `examples/host_peer_data_transfer` | host ↔ 协处理器自定义二进制通道（`esp_hosted_send_custom_data`，msg_id 区分，echo 测试） |
| `examples/host_hosted_cp_meminfo` | 查询协处理器内存信息（memory info） |
| `examples/host_hosted_events` | ESP_HOSTED 事件、心跳与传输故障恢复（生产级恢复循环） |
| `examples/host_manage_copro_ext_coex` | 协处理器外部共存（EXT_COEX）配置 |
| `examples/host_gpio_expander` | host 远程控制协处理器 GPIO（GPIO expander） |
| `examples/host_bt_controller_mac_addr` | BT 控制器 MAC 读写与初始化（v2.5.2+ 流程） |
| `examples/host_sdcard_with_hosted` | SD 卡（SDMMC）与 ESP-Hosted 共存（ESP32-P4 / S3） |

## Wi-Fi

| 示例路径 | 一句话描述 |
|---|---|
| `examples/host_network_split__power_save` | Network Split + host deep sleep + iperf（含 SPI/SDIO/UART/HD 的 CI sdkconfig） |
| `examples/host_wifi_easy_connect_dpp_enrollee` | Wi-Fi Easy Connect（DPP）入网设备端（ESP32-P4） |
| `examples/host_wifi_itwt` | Wi-Fi iTWT（Wi-Fi 6 个别目标唤醒时间）例程（ESP32-P4） |

## 蓝牙（BlueDroid / NimBLE）

| 示例路径 | 一句话描述 |
|---|---|
| `examples/host_bluedroid_ble_compatibility_test` | BlueDroid BLE 兼容性测试（手机对接） |
| `examples/host_bluedroid_bt_hid_mouse_device` | Classic BT HID 鼠标设备 |
| `examples/host_bluedroid_host_only` | BlueDroid host-only（经 ESP-Hosted 作 HCI IO，改自 bt_discovery） |
| `examples/host_nimble_bleprph_host_only_uart_hci` | NimBLE BLE 外设，标准 HCI over UART |
| `examples/host_nimble_bleprph_host_only_vhci` | NimBLE BLE 外设，Hosted HCI（VHCI，无需额外 GPIO） |

## OpenThread / Zigbee

| 示例路径 | 一句话描述 |
|---|---|
| `examples/host_openthread_border_router` | OpenThread Border Router（ESP32-P4） |
| `examples/host_openthread_cli` | OpenThread 命令行（ESP32-P4） |
| `examples/host_zigbee_thermostat` | Zigbee 温控器（ESP32-P4） |

## OTA / 省电

| 示例路径 | 一句话描述 |
|---|---|
| `examples/host_performs_slave_ota` | 经 ESP-Hosted 传输链路做协处理器 OTA（三种方法、版本检查） |
| `examples/host_shuts_down_slave_to_power_save` | host 不使用时关闭协处理器以省电 |

## 协处理器（slave）工程

| 路径 | 一句话描述 |
|---|---|
| `slave/` | 协处理器固件工程，由 `idf.py create-project-from-example "espressif/esp_hosted:slave"` 生成，per-target `sdkconfig.defaults.esp32*` |

## 备注

- 例程的目标支持芯片见各自 README 顶部的 "Supported Hosts/Targets" 表。
- `slave/sdkconfig.ci.*` 提供 CI 用的传输变体参考（`sdio`/`spi`/`spi_hd`/`uart`/`openthread_rcp`/`dpp`/`wifi_enterprise`/`all_features`）。
- Wi-Fi 吞吐与性能参考见 `docs/performance_optimization.md`、`docs/shield-box-test-setup.md`。
