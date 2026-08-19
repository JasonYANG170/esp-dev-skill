---
name: arduino-esp32-skill
description: >-
  AI Skill for developing firmware with the arduino-esp32 Arduino core (setup()/loop() Arduino API over ESP-IDF).
  Used when users need to create, modify, or debug ESP32 / ESP32-S / ESP32-C / ESP32-H / ESP32-P sketches,
  covering GPIO, Serial, I2C, SPI, LEDC/PWM, ADC, Timer, Wi-Fi (STA/AP), BLE (GATT/beacon), ESP-NOW, Zigbee,
  Matter, WebServer, OTA, Preferences/NVS, FreeRTOS tasks, deep sleep, partition tables and the v3.x Peripheral Manager API.
  Trigger words: "ESP32", "arduino-esp32", "Arduino Core ESP32", "ESP32-C3", "ESP32-S3", "乐鑫", "安信可", "安信可ESP32",
  "Wi-Fi STA", "SoftAP", "BLE", "GATT", "iBeacon", "Eddystone", "Zigbee", "Matter", "HomeKit", "WebServer",
  "ArduinoOTA", "LEDC", "ESP-NOW", "Preferences", "NVS", "乐鑫 Arduino", "ESP-IDF component"
license: LGPL-2.1
metadata:
  author: Community
  version: "1.1.0"
---

# arduino-esp32-skill

面向 arduino-esp32（Arduino core for ESP32）固件开发的 AI Skill。提供基于真实仓库文档（`docs/en/api/*.rst`、`libraries/`、`cores/esp32/`）的场景化 recipe、API 速查、配置参考与常见陷阱。覆盖 ESP32 / S2 / S3 / C3 / C5 / C6 / H2 / P8 系列芯片。

## Core Principles

1. **绝不臆造 API** — 所有��数名 / 结构体 / 宏 / Kconfig 必须可在 `resources/` 或仓库 `docs/en/api/` 中查到；查不到即视为不存在，绝不杜撰。
2. **v3.x Peripheral Manager** — 自 3.0 起 LEDC 通道自动绑定到引脚，旧 API `ledcSetup` / `ledcAttachPin` 已删除；使用 `ledcAttach(pin, freq, resolution)`。
3. **Network 类替代 WiFiClient/Server** — 3.x 引入统一 `Network` 库，TCP 服务端推荐 `NetworkServer` + `server.accept()`（`WiFiServer` 的 `available()` 已弃用）。
4. **setup()/loop() 在 loopTask 中运行** — Arduino 入口由 `cores/esp32/main.cpp` 的 FreeRTOS `loopTask` 调用，默认栈 8192 字节，`loop()` 不可阻塞太久否则触发任务看门狗。
5. **事件回调在独立线程触发** — `WiFi.onEvent()` 等回调运行于独立 FreeRTOS 任务，访问共享变量需加锁，且不可在回调里调用 `WiFi.onEvent/removeEvent`。
6. **NVS（Preferences）替代 EEPROM** — Preferences 基于片上 NVS 分区，namespace 与 key 各限 15 字符；适合存大量小键值，大块数据请用 LittleFS/FFat/SPIFFS。
7. **setHostname 必须在 WiFi 启动前** — 调用 `WiFi.begin/softAP/mode/run` 之后再改 hostname 无效；需先 `WiFi.mode(WIFI_MODE_NULL)` 复位。
8. **LP UART 引脚固定** — ESP32-C5/C6/P4 的低功耗 UART 引脚不可用 `setPins()` 改；HP UART 可任意 GPIO matrix 映射。
9. **analogRead 分辨率默认 12 位** — `analogReadResolution()` 范围 1–16；ESP32-S2 rev v0.0 默认 13 位（ADC-112 errata）。
10. **自定义分区表放在 sketch 同目录** — 文件名 `partitions.csv`，构建系统自动拾取；OTA 需 ≥2 个 `ota_x` 分区 + 8KB `otadata`。
11. **ESP-NOW 需先 `WiFi.mode()` 再 `ESP_NOW.begin()`** — ESP-NOW 依赖 Wi-Fi 已初始化并固定 channel。
12. **C2 / C61 需以 ESP-IDF 组件方式构建** — 不在 boards manager 预编译库覆盖范围，需 `idf_component.yml` 集成或 lib_builder 重建。
13. **BLE 协议栈按 SoC 不同** — ESP32 默认 Bluedroid（含 BT Classic，4.0 起移除）；C3/C5/C6/H2/S3 默认 NimBLE（仅 BLE，更省内存）。用 `BLEDevice::getBLEStackString()` 判断；NimBLE 权限走特征值属性（`PROPERTY_READ_AUTHEN`），Bluedroid 走 `setAccessPermissions(ESP_GATT_PERM_*)`。
14. **Zigbee 仅 C6/C5/H2 原生支持** — 其它 SoC 无 802.15.4 无线电，需外接 RCP 模组（`ZigbeeGateway` + `setRadioConfig`）。角色与 Tools 菜单强绑定：Coordinator/Router → `Zigbee ZCZR` mode；End Device → `Zigbee ED` mode，且需匹配的 `Zigbee xMB with spiffs` 分区方案；选错会 `#error` 编译失败。
15. **Matter 必须用 Huge APP 分区 + 擦全 flash** — 选 `Huge APP (3 MB No OTA / 1 MB SPIFFS)`（Matter 栈占 ~3MB）；上传前 `Erase All Flash Before Sketch Upload: Enabled` 清除 NVS 残留凭据，否则配网失败。Matter over Thread 在 C6/C5 需 ESP-IDF 组件方式重建预编译库。

## When to Use

**适用：**
- 为 ESP32 全系列创建 / 修改 / 调试 Arduino `.ino` sketch
- 外设驱动：GPIO、Serial(UART)、I2C、SPI、LEDC/PWM、ADC、Hardware Timer、Touch、RMT、TWAI
- 网络与无线：Wi-Fi STA/AP、BLE（GATT server/client、beacon）、ESP-NOW、Zigbee、Matter、WebServer、HTTPClient、mDNS、OTA
- 存储与持久化：Preferences/NVS、LittleFS、FFat、SPIFFS、SD/SD_MMC
- 系统：FreeRTOS 多任务、深睡眠与唤醒、分区表、Event 回调

**不适用：**
- 纯 ESP-IDF C 项目（应使用 ESP-IDF skill，非 Arduino API）
- Arduino AVR / STM32 / ESP8266 等其他平台
- PCB 硬件设计与原理图
- 上层应用框架（PlatformIO 配置、UI LVGL 业务逻辑等，超出本 skill 范围）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应 recipe**，其中含完整调用链、分步代码与常见错误。

### 外设驱动

| recipe | 场景 |
|---|---|
| `recipes/gpio_blink_interrupt.md` | GPIO 输入输出、按键、外部中断（attachInterrupt） |
| `recipes/serial_uart.md` | Serial(UART) 初始化、收发、自定义引脚、onReceive 回调、RS485 |
| `recipes/i2c_master_slave.md` | Wire 主机/从机通信、setPins、requestFrom、onReceive/onRequest |
| `recipes/spi_master.md` | SPI 主机多总线、SPIClass 自定义引脚 |
| `recipes/ledc_pwm.md` | LEDC PWM 输出、调光、ledcAttach/ledcWrite、音符 ledcWriteNote |
| `recipes/adc_reading.md` | analogRead / analogReadMilliVolts / 连续模式 analogContinuous |
| `recipes/hardware_timer.md` | 硬件定时器 timerBegin / timerAlarm 中断 |

### 网络 / 无线

| recipe | 场景 |
|---|---|
| `recipes/wifi_sta_ap.md` | Wi-Fi STA 连接、SoftAP、扫描、事件回调 |
| `recipes/ble.md` | BLE 扫描、GATT Server/Client、UART、Notify、iBeacon/Eddystone、配对/绑定 |
| `recipes/espnow.md` | ESP-NOW 广播 / 点对点、ESP_NOW_Peer 子类化 |
| `recipes/zigbee.md` | Zigbee 3.0 Coordinator/Router/End Device、灯/开关/传感器/网关、绑定、sleepy |
| `recipes/matter.md` | Matter over Wi-Fi/Thread、灯/传感器/按钮端点、QR 配网、解绑 |
| `recipes/webserver.md` | WebServer 路由、Basic/Digest 认证、文件上传、serveStatic、网页 OTA |
| `recipes/ota_update.md` | ArduinoOTA / Update / HTTPUpdate / OTAWebUpdater / 签名 OTA |
| `recipes/preferences_nvs.md` | Preferences 读写 NVS、namespace、各数据类型 |

### 系统

| recipe | 场景 |
|---|---|
| `recipes/freertos_tasks.md` | FreeRTOS 多任务、xTaskCreate、互斥/信号量 |
| `recipes/deep_sleep.md` | 深睡眠 + 定时器/外部/触摸唤醒、低功耗 |

---

## 芯片与外设支持矩阵（节选自 `docs/en/libraries.rst`）

| Feature | ESP32 | C3 | C5 | C6 | H2 | P4 | S2 | S3 |
|---|:--:|:--:|:--:|:--:|:--:|:--:|:--:|:--:|
| ADC | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| BT Classic | ✅ | — | — | — | — | — | — | — |
| BLE | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ |
| DAC | ✅ | — | — | — | — | ✅ | ✅ | — |
| ESP-NOW | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ | ✅ |
| Ethernet | ✅ | — | — | — | — | ✅ | — | — |
| GPIO / I2C / I2S / LEDC | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Touch | ✅ | — | — | — | — | — | ✅ | ✅ |
| Matter(Wi-Fi) | ✅ | no | ✅ | ✅ | — | — | ✅ | ✅ |
| Wi-Fi | ✅ | ✅ | ✅ | ✅ | — | — | ✅ | ✅ |

> ✅ = Arduino Core 支持��— = 该 SoC 无此硬件；no = 硬件存在但 Arduino Core 暂不支持。完整矩阵见 `docs/en/libraries.rst`。C2 / C61 需以 ESP-IDF 组件方式构建。

## Serial(UART) 数量矩阵（节选自 `docs/en/api/serial.rst`）

| SoC | HP UART | LP UART |
|---|:--:|:--:|
| ESP32 | 3 | 0 |
| ESP32-S2 | 2 | 0 |
| ESP32-S3 | 3 | 0 |
| ESP32-C3 | 2 | 0 |
| ESP32-C5 | 2 | 1 |
| ESP32-C6 | 2 | 1 |
| ESP32-H2 | 2 | 0 |
| ESP32-P4 | 5 | 1 |

`Serial0`(=`Serial`) 为 UART0；`Serial1`/`Serial2` 等按序枚举；LP UART 引脚固定，不可 `setPins()`。

---

## Critical Pitfalls (Must Read)

以下为最常见错误，违反任意一条都会导致固件无法正常工作。

### 1. v3.x LEDC API 已变（旧 ledcSetup/ledcAttachPin 删除）

```cpp
// ❌ WRONG — 2.x 写法，3.x 编译失败
ledcSetup(0, 5000, 12);
ledcAttachPin(5, 0);
ledcWrite(0, 2048);

// ✅ CORRECT — 3.x Peripheral Manager，通道自动绑定到引脚
ledcAttach(5, 5000, 12);   // pin, freq, resolution
ledcWrite(5, 2048);        // 以 pin 为索引
```

### 2. v3.x 服务端应使用 NetworkServer + accept()

```cpp
// ❌ WRONG — WiFiServer.available() 已弃用
WiFiServer server(80);
WiFiClient client = server.available();

// ✅ CORRECT — Network 库
#include <Network.h>
NetworkServer server(80);
NetworkClient client = server.accept();
```

### 3. setHostname 必须在 WiFi 启动之前

```cpp
// ❌ WRONG — begin 之后再改 hostname 无效
WiFi.begin(ssid, password);
WiFi.setHostname("myesp32");

// ✅ CORRECT
WiFi.setHostname("myesp32");   // 必须先于 mode/begin
WiFi.mode(WIFI_STA);
WiFi.begin(ssid, password);
// 若运行中修改：WiFi.mode(WIFI_MODE_NULL) -> setHostname -> 重新启动
```

### 4. onEvent 回调运行在独立线程，不可直接访问共享状态

```cpp
// ❌ WRONG — 回调里直接读写全局且调用非线程安全 API
volatile bool connected = false;
WiFi.onEvent([](arduino_event_id_t id){
    if (id == ARDUINO_EVENT_WIFI_STA_CONNECTED) {
        connected = true;                 // 数据竞争
        WiFi.onEvent(...);                // 非线程安全，禁止
    }
});

// ✅ CORRECT — 用互斥/原子或仅做线程安全操作（Serial.print 线程安全）
portMUX_TYPE mux = portMUX_INITIALIZER_UNLOCKED;
WiFi.onEvent([](arduino_event_id_t id){
    if (id == ARDUINO_EVENT_WIFI_STA_GOT_IP) {
        portENTER_CRITICAL(&mux);
        connected = true;
        portEXIT_CRITICAL(&mux);
    }
});
```

### 5. setRxBufferSize 必须在 begin() 之前

```cpp
// ❌ WRONG — begin 之后调用无效
Serial1.begin(115200);
Serial1.setRxBufferSize(1024);

// ✅ CORRECT
Serial1.setRxBufferSize(1024);
Serial1.begin(115200);
```

### 6. I2C setPins 必须在 begin() 之前

```cpp
// ❌ WRONG
Wire.begin();
Wire.setPins(SDA_PIN, SCL_PIN);   // 已用默认引脚初始化，无效

// ✅ CORRECT — setPins 在 begin 前，或直接传参 begin(sda, scl, freq)
Wire.setPins(SDA_PIN, SCL_PIN);
Wire.begin();
// 等价：Wire.begin(SDA_PIN, SCL_PIN, 100000);
```

### 7. Preferences namespace/key 限 15 字符

```cpp
// ❌ WRONG — 超长静默失败
preferences.begin("my-very-long-namespace", false);
preferences.putUInt("startup_counter_value", 1);

// ✅ CORRECT — namespace 与 key 均 ≤ 15 字符
preferences.begin("my-app", false);
preferences.putUInt("counter", 1);
```

### 8. ESP-NOW 必须先 WiFi.mode 再 begin

```cpp
// ❌ WRONG — Wi-Fi 未初始化，ESP_NOW.begin() 失败
ESP_NOW.begin();

// ✅ CORRECT
WiFi.mode(WIFI_STA);
WiFi.setChannel(ESPNOW_WIFI_CHANNEL);
while (!WiFi.STA.started()) delay(100);
ESP_NOW.begin();
```

### 9. loop() 不可长时间阻塞（任务看门狗）

```cpp
// ❌ WRONG — 长时间 delay/忙等触发 Task Watchdog
void loop() {
    while (waitSensor()) { }   // 可能阻塞 > 几秒
    delay(5000);
}

// ✅ CORRECT — 让出 CPU 或拆分到独立任务
void loop() {
    if (!waitSensor()) { delay(10); return; }   // 非阻塞轮询
    // 或 xTaskCreate(...) 把阻塞工作放到独立任务
}
```

### 10. ADC 衰减与可测电压范围按芯片不同

```cpp
// ❌ WRONG — 不设衰减直接读 3.3V，超出范围饱和
int v = analogRead(34);

// ✅ CORRECT — 11dB 衰减可测到 ~3.1V（ESP32/S3），且用毫伏校准
analogSetAttenuation(ADC_ATTEN_DB_11);
uint32_t mv = analogReadMilliVolts(34);   // 校准后毫伏
```

### 11. 自定义分区表文件名与位置

```text
# ❌ WRONG — 放错目录 / 名字不对，不会被构建系统拾取
my-sketch/part-table.csv

# ✅ CORRECT — 文件必须命名为 partitions.csv 且与 .ino 同目录
my-sketch/my-sketch.ino
my-sketch/partitions.csv
```

### 12. Hardware Timer 中断需用 IRAM_ATTR（推荐）并避免长操作

```cpp
// ❌ WRONG — 在定时器中断里 Serial.println / delay
void IRAM_ATTR onTimer() {
    Serial.println("tick");   // 中断上下文，不安全
}

// ✅ CORRECT — 中断仅置标志，loop()/任务里处理
volatile bool tickFlag = false;
void IRAM_ATTR onTimer() { tickFlag = true; }
void loop() {
    if (tickFlag) { tickFlag = false; Serial.println("tick"); }
}
```

---

## Execution Workflow

| Step | 名称 | 说明 |
|------|------|------|
| 1 | Plan | 明确目标芯片、所需外设/网络能力、是否 OTA/低功耗 |
| 2 | Recipe | 在 `recipes/` 找匹配场景，按其调用链与代码实现 |
| 3 | Query | recipe 未覆盖的 API，查 `resources/api_reference.md` |
| 4 | Validate | 核对函数签名、include、引脚、分区表、SoC 支持 |
| 5 | Confirm | 向用户说明：include 列表、setup() 初始化顺序、loop() 行为 |
| 6 | Execute | **新项目**：复制最接近的 `libraries/*/examples/` 示例改造；**现有项目**：原地编辑 |
| 7 | Check | 校验：v3.x API（无 2.x 旧 API）、回调线程安全、阻塞时长、分区表 |
| 8 | Build | Arduino IDE / `arduino-cli compile --board <fqbn>` 或 idf.py（组件方式） |
| 9 | Flash | USB（esptool）/ OTA（ArduinoOTA / HTTPUpdate） |

### Step 6 Detail — 项目创建策略

**目标目录尚无项目（首次创建）：**

1. 根据需求在仓库选最接近的示例：
   - Wi-Fi STA → `libraries/WiFi/examples/WiFiClient/WiFiClient.ino`
   - SoftAP → `libraries/WiFi/examples/WiFiAccessPoint/WiFiAccessPoint.ino`
   - BLE → `libraries/BLE/examples/Server/`、`Client/`、`Scan/`、`UART/`、`Notify/`、`iBeacon/`
   - Zigbee → `libraries/Zigbee/examples/Zigbee_On_Off_Light/`、`Zigbee_On_Off_Switch/`、`Zigbee_Temp_Hum_Sensor_Sleepy/`
   - Matter → `libraries/Matter/examples/MatterMinimum/`、`MatterOnOffLight/`
   - WebServer → `libraries/WebServer/examples/HelloServer/`、`HttpBasicAuth/`、`FSBrowser/`
   - OTA → `libraries/ArduinoOTA/examples/BasicOTA/`、`libraries/Update/examples/OTAWebUpdater/`、`HTTPS_OTA_Update/`
   - GPIO/中断 → `libraries/ESP32/examples/GPIO/GPIOInterrupt/`
   - LEDC → `libraries/ESP32/examples/AnalogOut/LEDCSoftwareFade/`
   - ADC → `libraries/ESP32/examples/AnalogRead/`
   - Timer → `libraries/ESP32/examples/Timer/RepeatTimer/`
   - Preferences → `libraries/Preferences/examples/StartCounter/`
   - ESP-NOW → `libraries/ESP_NOW/examples/ESP_NOW_Broadcast_Master/`
   - FreeRTOS → `libraries/ESP32/examples/FreeRTOS/BasicMultiThreading/`
   - DeepSleep → `libraries/ESP32/examples/DeepSleep/TimerWakeUp/`、`ExternalWakeUp/`
   - I2C → `libraries/Wire/examples/WireMaster/`、`WireSlave/`
   - SPI → `libraries/SPI/examples/SPI_Multiple_Buses/`
2. 复制示例目录到用户工程目录，保留结构。
3. 在副本上按需求改造（改名、调引脚、调分区表）。
4. 说明复制来源与原因。

**目标目录已有项目：** 原地编辑，勿整体覆盖。

---

## Failure Strategies

| 情况 | 动作 |
|---|---|
| API 在 resources/docs 查不到 | 立即停止，告知用户该 API 不存在于 arduino-esp32 |
| 编译报旧 API（ledcSetup 等） | 迁移到 v3.x API，参考 `resources/pitfalls.md` 与迁移指南 |
| 引脚冲突/无响应 | 核对 SoC 受限引脚（输入仅、禁止 GPIO），改用 GPIO matrix 可映射引脚 |
| WiFi 连不上 | 检查 `setMinSecurity`、密码长度、信道；`WiFi.setAutoReconnect(true)` |
| Preferences 写入失败 | namespace/key ≤15 字符；确认存在 `nvs` 分区（默认分区表已含） |
| OTA 空间不足 | 切换带 OTA 的分区表，或 `Update` 前确认 `partition` 大小 |
| 看门狗复位 | 检查 loop() 阻塞 / 中断处理耗时；必要时 `disableCore0WDT()` 并评估 |

## References

- 场景 recipe → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置 / 分区 / Kconfig → `resources/config_reference.md`
- 常见陷阱汇总 → `resources/pitfalls.md`
- 仓库示例索引 → `resources/example_list.md`
- 仓库原始文档 → 仓库 `docs/en/api/*.rst`、`docs/en/tutorials/*.rst`、`docs/en/migration_guides/2.x_to_3.0.rst`
