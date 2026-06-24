# ESP-IoT-Solution 示例索引

> 取自仓库 `examples/`（master 分支）。每个示例路径相对 `examples/`。拷贝到独立目录后，记得删除 `main/idf_component.yml` 中的 `override_path`，改用 Component Registry 拉取依赖。

## get-started（入门 / 低功耗）

| 示例 | 说明 |
|---|---|
| `get-started/blink` | 基础 LED 闪烁入门 |
| `get-started/button_power_save` | `button` + light sleep 低功耗唤醒（GPIO 唤醒） |
| `get-started/knob_power_save` | `knob` + light sleep 低功耗唤醒 |

## 总线 / 基础驱动（无独立示例，组件 README 含用法）

| 组件 | 示例来源 |
|---|---|
| `i2c_bus` | 组件 `components/i2c_bus/README.md` 内含完整读写示例 |
| `spi_bus` | 组件 `components/spi_bus/README.md` |

## sensors（传感器）

| 示例 | 说明 |
|---|---|
| `sensors/sensor_hub_monitor` | `sensor_hub` 监控：创建多个传感器、注册事件、打印数据 |
| `sensors/sensor_control_led` | `sensor_hub` + SHT3x：温度阈值控制板载 LED |
| `sensors/power_measure` | 功率计量（BL0937/BL0942/INA236） |
| `sensors/ntc_temperature_sensor` | NTC 热敏电阻温度采集 |

## 输入设备（button / knob / keyboard / touch）

| 示例 | 说明 |
|---|---|
| `touch/touch_button` | 触摸按键（iot_button 集成） |
| `touch/touch_button_sensor` | `touch_button_sensor` 多通道触摸检测（含 P4 多频采样） |
| `touch/touch_proximity` | 触摸接近感应 |
| `touch/touch_slider_sensor` | 触摸滑条 |
| `keyboard/*` | 矩阵键盘（components/main/hid_device） |

## 指示灯（led_indicator）

| 示例 | 说明 |
|---|---|
| `indicator/gpio` | GPIO 单色 LED 指示灯 |
| `indicator/ledc` | LEDC PWM LED（亮度/呼吸） |
| `indicator/rgb` | RGB LED 指示灯 |
| `indicator/ws2812_strips` | WS2812 灯带（addressable strips） |
| `indicator/components` | indicator 共用组件 |

## display（LCD / GUI）

| 示例 | 说明 |
|---|---|
| `display/lcd/qspi_without_ram` | QSPI LCD（无片上 RAM） |
| `display/lcd/qspi_with_ram` | QSPI LCD（带片上 RAM） |
| `display/lcd/rgb_avoid_tearing` | RGB LCD 防撕裂 |
| `display/lcd/rgb_lcd_8bit` | 8 位 RGB LCD |
| `display/lcd/mipi_dsi_avoid_tearing` | MIPI-DSI 防撕裂 |
| `display/lcd/lcd_with_te` | 带 Tearing Effect 信号的 LCD |
| `display/lcd/lcd_pattern_test` | LCD 花屏/测试图 |
| `display/lcd/lcd_layer_blending` | LCD 图层混合 |
| `display/lcd/hdmi_video_renderer` | HDMI 视频渲染 |
| `display/gui` | LVGL GUI 示例 |

## usb（Host / Device / OTG）

| 示例 | 说明 |
|---|---|
| `usb/host/usb_camera_lcd_display` | UVC 摄像头帧上 LCD 显示 |
| `usb/host/usb_camera_mic_spk` | UVC + UAC（摄像头 + 麦克风 + 扬声器） |
| `usb/host/usb_audio_player` | UAC 音频播放（扬声器） |
| `usb/host/usb_hub_dual_camera` | USB Hub 双摄像头 |
| `usb/host/usb_cdc_basic` | USB Host CDC-ACM 串口基本收发 |
| `usb/host/usb_cdc_4g_module` | USB CDC 接 4G 模组 |
| `usb/host/usb_ecm_4g_module` | USB ECM 接 4G 模组 |
| `usb/host/usb_rndis_4g_module` | USB RNDIS 接 4G 模组 |
| `usb/host/usb_host_msc_example` | USB Host MSC（U 盘） |
| `usb/host/usb_msc_ota` | USB MSC OTA 升级 |
| `usb/device` | USB Device（UAC/UVC/UF2 等） |
| `usb/otg` | USB OTG |

## bluetooth（BLE）

| 示例 | 说明 |
|---|---|
| `bluetooth/ble_conn_mgr/ble_periodic_adv` | `ble_conn_mgr` 周期广播 |
| `bluetooth/ble_conn_mgr/ble_periodic_sync` | `ble_conn_mgr` 周期同步 |
| `bluetooth/ble_conn_mgr/ble_spp` | `ble_conn_mgr` 串口协议（SPP，含 `spp_server` / `spp_client`） |
| `bluetooth/ble_l2cap_coc/l2cap_coc_central` | BLE L2CAP CoC 中心 |
| `bluetooth/ble_l2cap_coc/l2cap_coc_peripheral` | BLE L2CAP CoC 外设 |
| `bluetooth/ble_adv` | BLE 广播（含 BTHome） |
| `bluetooth/ble_adv/bthome/bulb` | BTHome 接收端：解析加密广播控制 WS2812 灯泡 |
| `bluetooth/ble_adv/bthome/dimmer` | BTHome 广播端：按键+旋钮发加密事件（Home Assistant dimmer） |
| `bluetooth/ble_ota` | BLE OTA 升级 |
| `bluetooth/ble_profiles/ble_ota` | `ble_ota_raw` profile（服务 UUID 0x8018，含 partitions.csv） |
| `bluetooth/ble_profiles/ble_htp` | 健康体温计 profile（HTP，服务 0x1809，含 console 命令） |
| `bluetooth/ble_profiles/ble_hrp` | 心率 profile（HRS/BAS） |
| `bluetooth/ble_profiles` | BLE 标准服务 profile 总目录（ANP/MIDI/OTP/HTP/HRP/OTA） |
| `bluetooth/ble_services` | BLE 服务（ans/bas/cts/dis/hrs/hts/ias/midi/ota/ots/tps/uds/wss） |
| `bluetooth/ble_remote_control` | BLE 遥控 |
| `bluetooth/ble_uart_service` | BLE 串口服务 |
| `bluetooth/ble_uart_vibe_indicator` | BLE UART + 振动指示 |

## motor（电机）

| 示例 | 说明 |
|---|---|
| `motor/servo_control` | LEDC 舵机角度控制 |
| `motor/drv10987` | DRV10987 无刷驱动 |
| `motor/foc_openloop_control` | FOC 开环控制（esp_simplefoc） |
| `motor/foc_velocity_control` | FOC 速度闭环 |
| `motor/foc_knob_example` | FOC 力矩旋钮 |
| `motor/bldc_fan_rainmaker` | 无感 BLDC 风扇 + RainMaker |

## 其它

| 示例 | 说明 |
|---|---|
| `lighting/lightbulb` | 灯泡驱动（lightbulb_driver） |
| `lighting/dali_basic` | DALI 调光基础 |
| `camera/basic` / `camera/video_lcd_display` / `camera/video_recorder` / `camera/pic_server` / `camera/video_stream_server` / `camera/test_framerate` | ESP 摄像头（ESP32-S3 等） |
| `ai/esp_dl` | ESP-DL AI 推理 |
| `ai/xiaozhi_chat` | 小智 AI 对话 |
| `audio/wav_player` | WAV 播放 |
| `ota/simple_ota_example` | 简单 OTA |
| `zero_cross_detection/main` | 过零检测 |
| `utilities/adc_tp_calibration` | ADC 两点校准 |
| `utilities/xz_decompress_file` | XZ 解压 |
| `vision/opencv` | OpenCV 视觉 |
| `weaver/imu_gesture` / `weaver/led_light` | esp_weaver（IMU 手势 / 灯） |
| `mcp/mcp_server` / `mcp/mcp_client` | MCP（Model Context Protocol） |
| `quickjs-ng/main` | QuickJS 嵌入式 JS |
| `gprof/gprof_simple` | gprof 性能分析 |
| `relinker/main` | 重链接 |
| `elf_loader/*` | ELF 动态加载（elf_console / elf_loader_example / build_elf_file_example / build_shared_library_example / elf_embed_example） |
| `extended_vfs/gpio` / `extended_vfs/i2c` / `extended_vfs/ledc` / `extended_vfs/spi` | extended_vfs 文件式外设访问 |
| `ulp/lp_cpu` | 低功耗协处理器（LP CPU） |
| `common_components/boards` | 板级抽象（`iot_board_*`）支持的开发板 |
| `common_components/camera` / `common_components/network_test` / `common_components/web_wifi` | 通用共用组件 |

## 组件自带示例（非顶层 examples/）

| 示例 | 说明 |
|---|---|
| `components/usb/usb_stream/examples/usb_stream_lcd_display` | usb_stream 摄像头帧上 LCD（含 JPEG 解码、ppbuffer） |
| `components/usb/usb_stream/examples/usb_stream_mic_spk` | usb_stream 麦克风 + 扬声器（含 HTTP 传输） |
