# ESP 术语表

| 缩写 | 全称 / 含义 |
|---|---|
| ESP-IDF | Espressif IoT Development Framework，ESP32 全系列官方开发框架 |
| SoC | System-on-Chip（ESP32/S3/C3/C6/H2/P4、ESP8266 等） |
| app_main | ESP-IDF/RTOS SDK 的用户入口函数（由系统任务调用） |
| idf.py | ESP-IDF 的构建/烧录/监控命令行工具（CMake + Ninja） |
| NVS | Non-Volatile Storage（键值对持久存储） |
| OTA | Over-The-Air 固件升级 |
| GATT/GAP | BLE 通用属性规范 / 通用访问规范 |
| NimBLE | 轻量 BLE 协议栈（ESP-IDF 默认 BLE 栈） |
| Bluedroid | ESP-IDF 另一套蓝牙栈（支持经典蓝牙 + BLE） |
| ESP-NOW | Espressif 无连接 Wi-Fi 通信协议 |
| A2DP/AVRCP/SPP | 经典蓝牙音频/远程控制/串口协议 |
| Matter | CSA 连接标准联盟的智能家居互联标准（原 Project CHIP） |
| Thread | 基于 6LoWPAN 的低功耗 mesh 网络协议（802.15.4） |
| Zigbee | 低功耗 mesh 无线协议（802.15.4） |
| connectedhomeip | Matter/CHIP 上游参考实现仓库 |
| Border Router | 边界路由器（Thread/Zigbee ↔ IP 网络） |
| mbedTLS | 轻量 SSL/TLS 库（ESP-IDF 默认 TLS 栈） |
| PSA Crypto | Platform Security Architecture 密码学 API |
| HAL | Hardware Abstraction Layer |
| BSP | Board Support Package |
| LVGL | Light and Versatile Graphics Library（嵌入式 GUI） |
| RISC-V / Xtensa | ESP 芯片的 CPU 架构（C/H 系多 RISC-V；原 ESP32/S3 为 Xtensa） |
| Component Registry | ESP 官方组件仓库（`idf.py add-dependency`） |
| AMP | Asymmetric Multiprocessing（异构多核） |
| WASM | WebAssembly（esp-wasmachine/wdf） |
| BIST | Built-In Self Test（芯片自测，IEC 60730） |
| Wi-Fi STA / AP | Station 模式 / SoftAP 模式 |
| FreeRTOS | ESP-IDF 使用的实时操作系统 |
