# AGENTS.md — 补充 Agent 指南

> 核心规则、recipe 索引、陷阱、执行流程均在 `SKILL.md`。
> 本文件只覆盖 `SKILL.md` 未提及的约定与工具链要点，不重复内容。

## 项目上下文

- **语言**：C / C++（`.ino` / `.cpp` / `.c` / `.h`）
- **目标**：Espressif ESP32 全系列 SoC（ESP32, ESP32-S2, ESP32-S3, ESP32-C3/C5/C6/C61, ESP32-H2, ESP32-P4）
- **底层**：Arduino API（`setup()`/`loop()`）构建于 ESP-IDF 之上（3.x core 基于 ESP-IDF >=5.3,<6.2）
- **工具链**：Arduino IDE / Arduino CLI（boards manager 预编译库），或 Arduino-as-ESP-IDF-component（`idf_component.yml`），或 PlatformIO（第三方）

## 代码生成约定

### 文件命名
- sketch 主文件：`<项目名>.ino`，与目录同名
- 库源码：`src/*.cpp` / `src/*.h`，库元数据 `library.properties`
- 头文件包含统一用 `<...>`（系统/库）或 `"..."`（本地）

### include 模式
```cpp
#include <Arduino.h>          // 核心（.ino 中通常隐式包含，多文件需显式）
#include <WiFi.h>             // Wi-Fi STA/AP/Scan
#include <WiFiAP.h>           // softAP 辅助
#include <Network.h>          // 3.x 统一网络：NetworkClient/Server/Udp
#include <ESP32_NOW.h>        // ESP-NOW（含 ESP_NOW / ESP_NOW_Peer）
#include <Wire.h>             // I2C
#include <SPI.h>              // SPI
#include <Preferences.h>      // NVS
#include <WebServer.h>        // 轻量 HTTP 服务
#include <HTTPClient.h>       // HTTP 客户端
#include <Update.h>           // OTA 升级
#include <esp_sleep.h>        // 深睡眠（ESP-IDF）
#include <freertos/FreeRTOS.h>// FreeRTOS 原生 API
```

### 标准 sketch 结构
```
MySketch/
├── MySketch.ino         // setup() + loop()
├── partitions.csv       // 可选：自定义分区表（与 .ino 同目录自动拾取）
├── src/                 // 可选：多文件 / 类
│   └── util.h
└── data/                // 可选：上传到 LittleFS/SPIFFS 的文件
```

### 标准 setup()/loop() 模板
```cpp
#include <Arduino.h>

void setup() {
    Serial.begin(115200);
    delay(150);                 // 给 Serial Monitor 时间打开
    // 初始化外设 / 网络 ...
}

void loop() {
    // 非阻塞；耗时操作交给独立 FreeRTOS 任务
}
```

> `setup()` / `loop()` 由 `cores/esp32/main.cpp` 的 FreeRTOS `loopTask` 调用，默认栈 8192 字节
> （可由 `SET_LOOP_STACK_SIZE` 或 `CONFIG_ARDUINO_LOOP_STACK_SIZE` 调整）。`loop()` 不可长时间阻塞。

### 中断回调模板
```cpp
// GPIO 中断
void IRAM_ATTR isrHandler() { /* 极短，仅置标志 */ }
attachInterrupt(BUTTON_PIN, isrHandler, FALLING);

// 串口接收回调（独立任务上下文，线程安全操作可用）
Serial1.onReceive([]() {
    while (Serial1.available()) { /* process */ }
});
```

### 调试输出约定
```cpp
// Arduino-ESP32 内置日志（级别由 menu / ARDUHAL_LOG_LEVEL 控制）
log_i("info: %d", value);
log_e("error: %s", msg);
// printf 到 Serial：Serial.printf("x=%d\n", x);  Serial 线程安全
```

## 构建工作流

### 方式 A：Arduino CLI
```bash
# 安装 core（示例版本，按实际取）
arduino-cli core install esp32:esp32@3.3.10
# 编译
arduino-cli compile --fqbn esp32:esp32:esp32 MySketch
# 烧录
arduino-cli upload -p COM5 --fqbn esp32:esp32:esp32 MySketch
```
板型号(FQBN)按芯片选：`esp32:esp32:esp32`、`esp32:esp32:esp32c3`、`esp32:esp32:esp32s3`、`esp32:esp32:esp32c6`、`esp32:esp32:esp32h2`、`esp32:esp32:esp32p4`、`esp32:esp32:esp32s2` 等。

### 方式 B：Arduino IDE
工具菜单选 Board / Port / Partition Scheme / Upload Speed / Flash Size；点 Upload。

### 方式 C：作为 ESP-IDF 组件（用于 C2/C61 或自定义 IDF 版本）
```bash
idf.py create-project-from-example "espressif/arduino-esp32:hello_world"
idf.py set-target esp32c2
idf.py build flash monitor
```
仓库根 `idf_component.yml` 声明依赖；示例见 `idf_component_examples/`。

## 代码生成清单

- [ ] 选对芯片 FQBN / Board，受限于 SoC 的外设（如 ESP32-S3 无 DAC、C3 无 Touch）
- [ ] 仅使用 v3.x API（无 `ledcSetup`/`ledcAttachPin`/`hallRead`/`analogSetClockDiv`/`adcAttachPin`）
- [ ] 网络服务端用 `NetworkServer` + `accept()`，而非 `WiFiServer.available()`
- [ ] `setPins()` / `setRxBufferSize()` / `setHostname()` 在对应 `begin()`/`mode()` 之前调用
- [ ] Preferences namespace、key 均 ≤ 15 字符，且存在 `nvs` 分区
- [ ] 回调（onEvent/onReceive/attachInterrupt）线程安全，不直接阻塞、不调用非线程安全 API
- [ ] `loop()` 非阻塞；长任务用 `xTaskCreate` 独立任务
- [ ] OTA / 大 sketch 选用合适分区表（含 `ota_0/ota_1/otadata`）
- [ ] 引脚避开受限引脚（输入-only、flash/PSRAM 占用、禁止 GPIO）

## Do Not Modify

- 仓库源码 `cores/`、`libraries/`、`variants/`、`tools/` — 只读参考；用户工程应复制示例改造，不直接改仓库
- `SKILL.md` frontmatter — Skill 元数据
- `resources/` — 文档来源，如需更新请基于仓库 `docs/en/` 与源码重新核对
