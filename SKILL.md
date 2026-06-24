---
name: esp-dev-skill
description: >-
  Unified AI Skill for Espressif ESP32 / ESP8266 firmware & software development across the
  entire ESP ecosystem. Bundles 41 grounded sub-skills (one per ESP framework/SDK repo):
  ESP-IDF, Arduino-ESP32, ESP8266 RTOS SDK, ESP-ADF/GMF, ESP-Brookesia, ESP-Claw,
  Matter/Zigbee/Thread/connectedhomeip, ESP-NOW, ESP-Hosted, NimBLE, MQTT, Modbus,
  ESP-USB/TinyUSB, ESP-AT, ESP-DL/DSP/SR/Skainet/Detection/Vision, mbedTLS/TF-PSA-Crypto,
  and more. Use for any task targeting an Espressif SoC.
  Trigger words: "ESP32", "ESP8266", "ESP-IDF", "esp-idf", "Espressif", "乐鑫", "安信可",
  "Arduino ESP", "Matter", "Zigbee", "Thread", "BLE", "NimBLE", "Wi-Fi", "ESP-NOW", "MQTT",
  "OTA", "FreeRTOS", "嵌入式 ESP", "IoT", "idf.py"
tags:
  - embedded
  - espressif
  - esp32
  - esp8266
  - esp-idf
  - arduino
  - firmware
  - IoT
  - bluetooth
  - wifi
  - matter
  - zigbee
license: Apache-2.0
compatibility: >-
  Targets ESP32, ESP32-S2/S3, ESP32-C2/C3/C5/C6/C61, ESP32-H2/H4/H21, ESP32-P4, ESP8266.
  Build via ESP-IDF (idf.py / CMake + Ninja), Arduino-ESP32, or ESP8266 RTOS SDK toolchains.
metadata:
  author: Community
  version: "1.0.0"
---

# esp-dev-skill

Espressif 全生态统一开发技能。把 **41 个** 独立的 ESP 框架/SDK 子技能（每个对应一个真实仓库，内容全部基于该仓库的真实文档与源码）聚合到 `repos/<repo>/` 下。本 `SKILL.md` 是**路由入口**：帮你定位到正确的子技能；每个子技能有自己的 `repos/<repo>/SKILL.md`（核心原则、recipe 索引、❌/✅ 陷阱、API 参考）。

- **41 个仓库 · 554 个 recipe · 全部基于真实 docs/源码接地**
- 跨仓库按主题查找：`resources/recipe_index.md`（全部 recipe 总表）
- 仓库一览：`resources/repo_index.md`；术语：`resources/glossary.md`

## 如何使用本技能

1. **定位需求** —— 你要用哪个框架/SDK/功能？（不确定就用下面的路由表）
2. **进入子技能** —— 打开 `repos/<repo>/SKILL.md` 看核心原则与 recipe 索引
3. **读 recipe** —— 按 `repos/<repo>/recipes/*.md` 的场景一步步做（含真实代码、常见错误、参考示例）
4. **查 API/配置** —— `repos/<repo>/resources/{api_reference,config_reference,pitfalls,example_list}.md`
5. **跨仓库找主题** —— `resources/recipe_index.md`

## ESP 开发生态（分层）

**SoC**：ESP32、ESP32-S2/S3、ESP32-C2/C3/C5/C6/C61、ESP32-H2/H4/H21、ESP32-P4、ESP8266

**软件栈分层**（从底到顶）：
- **核心框架** —— ESP-IDF（官方地基）/ Arduino-ESP32 / ESP8266 RTOS SDK
- **领域框架** —— ADF(音频) · GMF(多媒体) · WDF/Wasmachine(WASM) · Brookesia(UI) · IoT-Solution(组件) · Vision/Who(视觉) · Insights(诊断) · Claw(AI agent) · AMP(多核) · LowCode-Matter · Agents-Firmware
- **协议/连接** —— Matter · Zigbee · Thread · connectedhomeip · ESP-NOW · Hosted · NimBLE(BLE) · MQTT · Modbus · USB · TinyUSB · AT · Protocols
- **AI/语音/视觉/DSP** —— Skainet · SR · DL · DSP · Detection · Video-Components · RainMaker(云)
- **安全/加密** —— mbedTLS · TF-PSA-Crypto · Secure-Cert-Mgr
- **其他组件库** —— BIST(自测) · Serial-Flasher(烧录)

## 路由表 —— 要做某件事，用哪个仓库


### 核心框架

_顶层开发平台，应用直接构建其上。先在这里选一个。_

| 仓库 | 用途 | recipes | 入口 |
|---|---|---|---|
| `esp-idf` | Espressif 官方 IoT 开发框架（ESP32 全系列 SoC，Wi-Fi/BLE/外设/OTA/分区/FreeRTOS/idf.py） | 21 | `repos/esp-idf/SKILL.md` |
| `arduino-esp32` | ESP32 的 Arduino core（setup()/loop()，Arduino API over ESP-IDF） | 17 | `repos/arduino-esp32/SKILL.md` |
| `ESP8266_RTOS_SDK` | ESP8266 的 FreeRTOS SDK，esp-idf 风格 API | 18 | `repos/ESP8266_RTOS_SDK/SKILL.md` |

### 领域框架

_构建于核心框架之上的垂直/领域开发框架。_

| 仓库 | 用途 | recipes | 入口 |
|---|---|---|---|
| `esp-adf` | 音频/多媒体高级开发框架（播放/录音/流媒体/管道） | 15 | `repos/esp-adf/SKILL.md` |
| `esp-gmf` | 通用多媒体框架（pipeline/element/音频处理） | 16 | `repos/esp-gmf/SKILL.md` |
| `esp-wdf` | WASM 开发框架 | 13 | `repos/esp-wdf/SKILL.md` |
| `esp-wasmachine` | WebAssembly 虚拟机开发框架，跑 WASM 应用 | 13 | `repos/esp-wasmachine/SKILL.md` |
| `esp-brookesia` | HMI 人机交互 UI 框架（LVGL 资源/屏幕方案） | 12 | `repos/esp-brookesia/SKILL.md` |
| `esp-iot-solution` | IoT 组件库与解决方案（驱动/BSP/USB/显示） | 18 | `repos/esp-iot-solution/SKILL.md` |
| `esp-vision` | 边缘 AI / 计算机视觉框架 | 18 | `repos/esp-vision/SKILL.md` |
| `esp-who` | 人脸检测/识别框架 | 9 | `repos/esp-who/SKILL.md` |
| `esp-insights` | 远程诊断 / 可观测性框架（日志/指标上报） | 10 | `repos/esp-insights/SKILL.md` |
| `esp-claw` | IoT 设备 'Chat Coding' AI agent 框架 | 12 | `repos/esp-claw/SKILL.md` |
| `esp-amp` | 异构多核（AMP）框架，主核+子核/核间通信 | 11 | `repos/esp-amp/SKILL.md` |
| `esp-lowcode-matter` | 低代码 Matter 设备构建框架 | 16 | `repos/esp-lowcode-matter/SKILL.md` |
| `esp-agents-firmware` | ESP Private Agents 平台的设备端 AI agent 固件 SDK | 12 | `repos/esp-agents-firmware/SKILL.md` |

### 协议/连接 SDK

_无线/有线连接与协议栈（Matter/Zigbee/Thread/BLE/Wi-Fi/MQTT/USB…）。_

| 仓库 | 用途 | recipes | 入口 |
|---|---|---|---|
| `esp-matter` | Matter SDK（配网/cluster/fabric） | 17 | `repos/esp-matter/SKILL.md` |
| `esp-zigbee-sdk` | Zigbee SDK（ZC/ZR/ZED，cluster/endpoint） | 12 | `repos/esp-zigbee-sdk/SKILL.md` |
| `esp-thread-br` | Thread 边界路由器 SDK | 14 | `repos/esp-thread-br/SKILL.md` |
| `connectedhomeip` | Matter/CHIP 上游参考实现（CSA），Espressif fork | 14 | `repos/connectedhomeip/SKILL.md` |
| `esp-now` | ESP-NOW 无连接 Wi-Fi 协议 | 12 | `repos/esp-now/SKILL.md` |
| `esp-hosted-mcu` | ESP 作为通信协处理器的 Hosted SDK（SDIO/SPI） | 16 | `repos/esp-hosted-mcu/SKILL.md` |
| `esp-nimble` | BLE 协议栈 NimBLE（GAP/GATT/安全） | 13 | `repos/esp-nimble/SKILL.md` |
| `esp-mqtt` | MQTT 客户端组件 | 12 | `repos/esp-mqtt/SKILL.md` |
| `esp-modbus` | Modbus 协议库（RTU + TCP） | 10 | `repos/esp-modbus/SKILL.md` |
| `esp-usb` | USB host/device 类驱动（CDC/MSC/HID） | 13 | `repos/esp-usb/SKILL.md` |
| `tinyusb` | TinyUSB 设备栈（Espressif fork） | 14 | `repos/tinyusb/SKILL.md` |
| `esp-at` | AT 指令固件平台（WiFi/BT 经 AT 控制） | 8 | `repos/esp-at/SKILL.md` |
| `esp-protocols` | ESP-IDF 网络协议组件集合（esp_modbus/esp_hosted 等） | 11 | `repos/esp-protocols/SKILL.md` |

### AI/语音/视觉/DSP

_端侧 AI、语音、视觉、信号处理库与 SDK。_

| 仓库 | 用途 | recipes | 入口 |
|---|---|---|---|
| `esp-skainet` | 智能语音助手 SDK（wake word/命令词/SR） | 13 | `repos/esp-skainet/SKILL.md` |
| `esp-sr` | 语音识别模型与库（WakeNet/MultiNet） | 11 | `repos/esp-sr/SKILL.md` |
| `esp-dl` | 深度学习库（模型部署/量化） | 16 | `repos/esp-dl/SKILL.md` |
| `esp-dsp` | DSP 库（FFT/滤波/矩阵） | 15 | `repos/esp-dsp/SKILL.md` |
| `esp-detection` | 轻量目标检测（基于 Ultralytics YOLOv11） | 9 | `repos/esp-detection/SKILL.md` |
| `esp-video-components` | 摄像头/视频组件集合 | 12 | `repos/esp-video-components/SKILL.md` |
| `esp-rainmaker` | ESP RainMaker 云 IoT SDK / agent | 14 | `repos/esp-rainmaker/SKILL.md` |

### 安全/加密

_TLS/PSA 加密与证书管理。_

| 仓库 | 用途 | recipes | 入口 |
|---|---|---|---|
| `mbedtls` | mbedTLS SSL/TLS 库（Espressif fork） | 16 | `repos/mbedtls/SKILL.md` |
| `TF-PSA-Crypto` | PSA Cryptography API 参考实现 | 11 | `repos/TF-PSA-Crypto/SKILL.md` |
| `esp_secure_cert_mgr` | 安全证书管理组件 | 10 | `repos/esp_secure_cert_mgr/SKILL.md` |

### 其他组件库

_烧录、自测等工具型组件库。_

| 仓库 | 用途 | recipes | 入口 |
|---|---|---|---|
| `esp-bist` | 内建自测（BIST）库，IEC 60730 安全关键 | 12 | `repos/esp-bist/SKILL.md` |
| `esp-serial-flasher` | 跨 MCU 烧录库（ESP Serial Flasher） | 18 | `repos/esp-serial-flasher/SKILL.md` |

## 跨仓库通用原则

1. **先选对层** —— 裸 IDF / Arduino / ESP8266 RTOS SDK 三选一，决定了项目结构、入口函数（`app_main` vs `setup()/loop()`）和构建系统。
2. **ESP-IDF 版本敏感** —— 蓝牙、Wi-Fi、peripheral API 在大版本间会变（如 v5.x 的 `esp_ble_gattc_enh_open`、`esp_spp_enhanced_init`）。写代码前先确认目标 IDF 版本。
3. **组件可来自 Component Registry** —— 很多仓库的组件也能用 `idf.py add-dependency` 从 ESP Component Registry 拉取，不必整仓引入。
4. **接地优先，拒绝臆造** —— 每个子技能的内容都来自真实仓库；若某子技能把某主题标为「留白/gap」，说明文档不足，**不要凭空补 API**，去对应仓库的 docs/examples 核实。
5. **协议栈多有关联** —— Matter 依赖 connectedhomeip；Zigbee/Thread 仅 ESP32-C/H 系列支持；ESP-NOW/NimBLE/MQTT 各有独立子技能但都跑在 IDF 之上。
6. **芯片能力差异** —— 经典蓝牙(A2DP/SPP)仅 ESP32(经典)支持；BLE Mesh/Thread/Zigbee 需 802.15.4/BLE 芯片（C/H 系列）。先确认目标芯片具备该能力。

## 执行流程

| 步骤 | 动作 |
|---|---|
| 1 定位 | 明确目标框架/SDK；不确定看路由表或 `resources/repo_index.md` |
| 2 子技能 | 读 `repos/<repo>/SKILL.md` 的核心原则与 recipe 索引 |
| 3 recipe | 找匹配场景的 `repos/<repo>/recipes/*.md`，按步骤实现 |
| 4 校验 | 核对包含、初始化顺序、配置项、引脚/芯片支持 |
| 5 确认 | 向用户给出实现计划（依赖、引脚、入口、构建命令） |
| 6 执行 | 新项目可参考子技能里引用的真实 example；现有项目就地编辑 |
| 7 构建/烧录 | `idf.py build flash monitor`（IDF）/ Arduino 上传 / AT 固件打包 |

## 失败策略

| 情况 | 处理 |
|---|---|
| 不确定用哪个仓库 | 查路由表 + `resources/repo_index.md` |
| API 在子技能里找不到 | 去对应仓库 `repos/<repo>/resources/api_reference.md` 或真实头文件核实 |
| 子技能标注某主题为 gap | 不要臆造；读上游仓库 docs/examples 后再写 |
| 跨仓库能力组合（如 Matter+Wi-Fi+OTA） | 分别参考各子技能，注意初始化顺序与内存 |

## 参考

- 子技能入口：`repos/<repo>/SKILL.md`（41 个）
- 全 recipe 总表：`resources/recipe_index.md`（554 个 recipe，按仓库分组）
- 仓库索引：`resources/repo_index.md`
- 术语表：`resources/glossary.md`
