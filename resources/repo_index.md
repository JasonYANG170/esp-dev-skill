# 仓库索引（41 个 ESP 框架/SDK）

| 仓库 | 分类 | 版本 | recipes | resources | 路径 | 说明 |
|---|---|---|---|---|---|---|
| `esp-idf` | 核心框架 | v1.1.0 | 21 | 4 | `repos/esp-idf/` | Espressif 官方 IoT 开发框架（ESP32 全系列 SoC，Wi-Fi/BLE/外设/OTA/分区/FreeRTOS/idf.py） |
| `arduino-esp32` | 核心框架 | v1.1.0 | 17 | 4 | `repos/arduino-esp32/` | ESP32 的 Arduino core（setup()/loop()，Arduino API over ESP-IDF） |
| `ESP8266_RTOS_SDK` | 核心框架 | v1.1.0 | 18 | 4 | `repos/ESP8266_RTOS_SDK/` | ESP8266 的 FreeRTOS SDK，esp-idf 风格 API |
| `esp-adf` | 领域框架 | v1.1.0 | 15 | 4 | `repos/esp-adf/` | 音频/多媒体高级开发框架（播放/录音/流媒体/管道） |
| `esp-gmf` | 领域框架 | v1.1.0 | 16 | 4 | `repos/esp-gmf/` | 通用多媒体框架（pipeline/element/音频处理） |
| `esp-wdf` | 领域框架 | v1.1.0 | 13 | 4 | `repos/esp-wdf/` | WASM 开发框架 |
| `esp-wasmachine` | 领域框架 | v1.0.0 | 13 | 5 | `repos/esp-wasmachine/` | WebAssembly 虚拟机开发框架，跑 WASM 应用 |
| `esp-brookesia` | 领域框架 | v1.1.0 | 12 | 4 | `repos/esp-brookesia/` | HMI 人机交互 UI 框架（LVGL 资源/屏幕方案） |
| `esp-iot-solution` | 领域框架 | v1.1.0 | 18 | 4 | `repos/esp-iot-solution/` | IoT 组件库与解决方案（驱动/BSP/USB/显示） |
| `esp-vision` | 领域框架 | v1.1.0 | 18 | 4 | `repos/esp-vision/` | 边缘 AI / 计算机视觉框架 |
| `esp-who` | 领域框架 | v1.0.0 | 9 | 5 | `repos/esp-who/` | 人脸检测/识别框架 |
| `esp-insights` | 领域框架 | v1.0.0 | 10 | 4 | `repos/esp-insights/` | 远程诊断 / 可观测性框架（日志/指标上报） |
| `esp-claw` | 领域框架 | v1.1.0 | 12 | 4 | `repos/esp-claw/` | IoT 设备 'Chat Coding' AI agent 框架 |
| `esp-amp` | 领域框架 | v1.0.0 | 11 | 5 | `repos/esp-amp/` | 异构多核（AMP）框架，主核+子核/核间通信 |
| `esp-lowcode-matter` | 领域框架 | v1.1.0 | 16 | 4 | `repos/esp-lowcode-matter/` | 低代码 Matter 设备构建框架 |
| `esp-agents-firmware` | 领域框架 | v1.1.0 | 12 | 4 | `repos/esp-agents-firmware/` | ESP Private Agents 平台的设备端 AI agent 固件 SDK |
| `esp-matter` | 协议/连接 SDK | v1.1.0 | 17 | 5 | `repos/esp-matter/` | Matter SDK（配网/cluster/fabric） |
| `esp-zigbee-sdk` | 协议/连接 SDK | v1.1.0 | 12 | 4 | `repos/esp-zigbee-sdk/` | Zigbee SDK（ZC/ZR/ZED，cluster/endpoint） |
| `esp-thread-br` | 协议/连接 SDK | v1.1.0 | 14 | 4 | `repos/esp-thread-br/` | Thread 边界路由器 SDK |
| `connectedhomeip` | 协议/连接 SDK | v1.1.0 | 14 | 4 | `repos/connectedhomeip/` | Matter/CHIP 上游参考实现（CSA），Espressif fork |
| `esp-now` | 协议/连接 SDK | v1.1.0 | 12 | 4 | `repos/esp-now/` | ESP-NOW 无连接 Wi-Fi 协议 |
| `esp-hosted-mcu` | 协议/连接 SDK | v1.1.0 | 16 | 5 | `repos/esp-hosted-mcu/` | ESP 作为通信协处理器的 Hosted SDK（SDIO/SPI） |
| `esp-nimble` | 协议/连接 SDK | v1.1.0 | 13 | 4 | `repos/esp-nimble/` | BLE 协议栈 NimBLE（GAP/GATT/安全） |
| `esp-mqtt` | 协议/连接 SDK | v1.1.0 | 12 | 4 | `repos/esp-mqtt/` | MQTT 客户端组件 |
| `esp-modbus` | 协议/连接 SDK | v1.1.0 | 10 | 4 | `repos/esp-modbus/` | Modbus 协议库（RTU + TCP） |
| `esp-usb` | 协议/连接 SDK | v1.1.0 | 13 | 4 | `repos/esp-usb/` | USB host/device 类驱动（CDC/MSC/HID） |
| `tinyusb` | 协议/连接 SDK | v1.1.0 | 14 | 4 | `repos/tinyusb/` | TinyUSB 设备栈（Espressif fork） |
| `esp-at` | 协议/连接 SDK | v1.0.0 | 8 | 4 | `repos/esp-at/` | AT 指令固件平台（WiFi/BT 经 AT 控制） |
| `esp-protocols` | 协议/连接 SDK | v1.1.0 | 11 | 4 | `repos/esp-protocols/` | ESP-IDF 网络协议组件集合（esp_modbus/esp_hosted 等） |
| `esp-skainet` | AI/语音/视觉/DSP | v1.1.0 | 13 | 4 | `repos/esp-skainet/` | 智能语音助手 SDK（wake word/命令词/SR） |
| `esp-sr` | AI/语音/视觉/DSP | v1.1.0 | 11 | 5 | `repos/esp-sr/` | 语音识别模型与库（WakeNet/MultiNet） |
| `esp-dl` | AI/语音/视觉/DSP | v1.1.0 | 16 | 4 | `repos/esp-dl/` | 深度学习库（模型部署/量化） |
| `esp-dsp` | AI/语音/视觉/DSP | v1.1.0 | 15 | 4 | `repos/esp-dsp/` | DSP 库（FFT/滤波/矩阵） |
| `esp-detection` | AI/语音/视觉/DSP | v1.0.0 | 9 | 5 | `repos/esp-detection/` | 轻量目标检测（基于 Ultralytics YOLOv11） |
| `esp-video-components` | AI/语音/视觉/DSP | v1.0.0 | 12 | 4 | `repos/esp-video-components/` | 摄像头/视频组件集合 |
| `esp-rainmaker` | AI/语音/视觉/DSP | v1.1.0 | 14 | 4 | `repos/esp-rainmaker/` | ESP RainMaker 云 IoT SDK / agent |
| `mbedtls` | 安全/加密 | v1.1.0 | 16 | 4 | `repos/mbedtls/` | mbedTLS SSL/TLS 库（Espressif fork） |
| `TF-PSA-Crypto` | 安全/加密 | v1.1.0 | 11 | 4 | `repos/TF-PSA-Crypto/` | PSA Cryptography API 参考实现 |
| `esp_secure_cert_mgr` | 安全/加密 | v1.0.0 | 10 | 4 | `repos/esp_secure_cert_mgr/` | 安全证书管理组件 |
| `esp-bist` | 其他组件库 | v1.1.0 | 12 | 4 | `repos/esp-bist/` | 内建自测（BIST）库，IEC 60730 安全关键 |
| `esp-serial-flasher` | 其他组件库 | v1.1.0 | 18 | 4 | `repos/esp-serial-flasher/` | 跨 MCU 烧录库（ESP Serial Flasher） |

**合计 41 个仓库，554 个 recipe。**
