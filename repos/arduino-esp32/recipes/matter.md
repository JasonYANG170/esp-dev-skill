# Matter 智能家居（Wi-Fi / Thread，commissioning，on-off/light/color/sensor/按钮）

> **适用摘要**: 使用 `Matter` 库（基于 ESP Matter SDK）创建 Apple HomeKit/Google Home/Amazon Alexa 兼容的 Matter 设备，注册 `MatterEndPoint`（灯/传感器/插座/开关/窗帘），含 QR 码配网、事件回调、解绑出厂。⚠️ 必须选 `Huge APP (3 MB No OTA / 1 MB SPIFFS)` 分区方案并擦除全 flash 后上传。

> Evidence: `repos/arduino-esp32/resources/`, source/examples in `repos/arduino-esp32/`, and this recipe path `repos/arduino-esp32/recipes/matter.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Matter 设备 / 智能家居 / HomeKit / Google Home"
- "Matter on-off light / color light"
- "Matter temperature sensor / contact sensor"
- "Matter 配网 / QR 码 / 解绑"
- "Matter over Wi-Fi / Thread"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/Matter/examples/MatterMinimum/`、`MatterOnOffLight/`、`MatterColorLight/`、`MatterTemperatureSensor/`、`MatterContactSensor/`、`MatterSmartButton/` |
| 分区方案 | Arduino IDE: Tools → Partition Scheme → **"Huge APP (3 MB No OTA / 1 MB SPIFFS)"**（Matter 栈占 ~3MB） |
| 擦除 | Tools → **"Erase All Flash Before Sketch Upload: Enabled"**，清除 NVS 中残留 Wi-Fi/Matter 凭据（或 `esptool.py erase_flash`） |
| 芯片 | Matter over Wi-Fi：ESP32/S3/C5/C6；Matter over Thread：C6/C5/H2（预编译库不含 Thread，需 ESP-IDF 组件方式重建） |
| Wi-Fi | 若 `CONFIG_ENABLE_CHIPOBLE` 未开（非 BLE 配网），代码中需 `WiFi.begin(ssid, pass)` 连 2.4GHz |

## 分步说明

### 启动顺序（取自 MatterMinimum / MatterOnOffLight 示例）

1. `pinMode` + 初始外设状态 → 2.（非 BLE 配网时）`WiFi.begin` 等 `WL_CONNECTED` → 3. 实例化端点 + `endpoint.begin()` + `endpoint.onChange(cb)` → 4. `Matter.begin()`（最后一步）→ 5. 若 `!Matter.isDeviceCommissioned()` 打印配网码/QR URL，`loop()` 等待配网完成。

### 最小 On/Off 灯（取自 MatterMinimum 示例）

```cpp
#include <Arduino.h>
#include <Matter.h>
#if !CONFIG_ENABLE_CHIPOBLE
#include <WiFi.h>      // BLE 配网开启时不需要
#endif

MatterOnOffLight OnOffLight;          // 单例端点

#if !CONFIG_ENABLE_CHIPOBLE
const char *ssid = "your-ssid", *password = "your-password";
#endif
const uint8_t ledPin = LED_BUILTIN, buttonPin = BOOT_PIN;

// Matter 控制器下发状态回调（必须返回 true 表示成功）
bool onOffLightCallback(bool state) {
  digitalWrite(ledPin, state ? HIGH : LOW);
  return true;
}

void setup() {
  Serial.begin(115200);
  pinMode(buttonPin, INPUT_PULLUP);
  pinMode(ledPin, OUTPUT);

#if !CONFIG_ENABLE_CHIPOBLE
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
#endif

  OnOffLight.begin();                 // 初始化端点
  OnOffLight.onChange(onOffLightCallback);
  Matter.begin();                     // 必须在所有 endpoint.begin() 之后

  if (!Matter.isDeviceCommissioned()) {
    Serial.printf("Manual pairing code: %s\n", Matter.getManualPairingCode().c_str());
    Serial.printf("QR code URL: %s\n", Matter.getOnboardingQRCodeUrl().c_str());
  }
}

void loop() {
  // BOOT 键长按 >5s 解绑出厂
  static uint32_t t0 = 0; static bool pressed = false;
  if (digitalRead(buttonPin) == LOW && !pressed) { t0 = millis(); pressed = true; }
  if (digitalRead(buttonPin) == HIGH) pressed = false;
  if (pressed && millis() - t0 > 5000) {
    Matter.decommission();
    t0 = millis();   // 避免重复触发
  }
  delay(500);
}
```

### 状态持久化 + 双向控制（取自 MatterOnOffLight 示例）

```cpp
#include <Preferences.h>
Preferences pref;
const char *key = "OnOff";

bool setLightOnOff(bool state) {
  digitalWrite(ledPin, state ? HIGH : LOW);
  pref.putBool(key, state);       // 掉电保存
  return true;
}

void setup() {
  pinMode(ledPin, OUTPUT);
  pref.begin("MatterPrefs", false);
  bool lastState = pref.getBool(key, true);   // 默认开
  OnOffLight.begin(lastState);                 // 传入初始状态
  OnOffLight.onChange(setLightOnOff);
  Matter.begin();

  if (Matter.isDeviceCommissioned()) {
    OnOffLight.updateAccessory();              // 同步初始状态到 Matter 网络
  }
}
void loop() {
  // 按键短按切换：OnOffLight.toggle();
  // 长按 5s 解绑：OnOffLight.setOnOff(false); Matter.decommission();
}
```

### Matter 核心类 API（取自 matter.rst）

```cpp
// 单例 Matter（singleton）
Matter.begin();
bool Matter.isDeviceCommissioned();
bool Matter.isWiFiConnected();       // Wi-Fi 连接状态
bool Matter.isThreadConnected();     // Thread 连接状态
bool Matter.isDeviceConnected();     // 整体连接状态
bool Matter.isWiFiStationEnabled();
bool Matter.isWiFiAccessPointEnabled();
bool Matter.isThreadEnabled();
bool Matter.isBLECommissioningEnabled();
void Matter.decommission();          // 出厂复位（擦 Matter 凭据）
String Matter.getManualPairingCode();
String Matter.getOnboardingQRCodeUrl();

// 事件回调
Matter.onEvent([](matterEvent_t event, const chip::DeviceLayer::ChipDeviceEvent *data){
  switch (event) {
    case MATTER_COMMISSIONING_COMPLETE:        Serial.println("Commissioned!"); break;
    case MATTER_WIFI_CONNECTIVITY_CHANGE:      Serial.println("WiFi changed"); break;
    // ... 其它事件
  }
});
```

### 可用端点类（节选自 matter.rst）

| 类别 | 类 |
|---|---|
| 灯 | `MatterOnOffLight` `MatterDimmableLight` `MatterColorTemperatureLight` `MatterColorLight`（HSV）`MatterEnhancedColorLight` |
| 传感器 | `MatterTemperatureSensor` `MatterHumiditySensor` `MatterPressureSensor` `MatterContactSensor` `MatterWaterLeakDetector` `MatterOccupancySensor` `MatterLightSensor` |
| 控制 | `MatterFan` `MatterThermostat` `MatterOnOffPlugin`（插座）`MatterDimmablePlugin` `MatterGenericSwitch`（智能按钮）`MatterWindowCovering`（窗帘） |

端点通用：`ep.begin()` / `ep.begin(initialState)` / `ep.onChange(cb)` / `ep.getOnOff()` / `ep.setOnOff(bool)` / `ep.toggle()` / `ep.updateAccessory()`（同步本地状态到 Matter 网络）。

### MatterEndPoint 基类（matter_ep.rst）

每个端点有唯一 endpoint ID；支持 Identify Cluster（识别闪烁）、属性读写、第二网络接口。`onChange` 回调签名因端点而异（灯 `bool(bool)`、传感器上报用 `set` 类方法、按钮通过 `MatterGenericSwitch` 发事件）。

> ⚠️ 配网失败最常见原因：未擦除全 flash → NVS 残留旧 Wi-Fi/Matter 凭据 → 配网中途冲突。务必 `Erase All Flash Before Sketch Upload: Enabled` 或 `esptool.py --port <PORT> erase_flash`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译/链接失败（空间不足） | 默认分区太小 | 选 `Huge APP (3 MB No OTA / 1 MB SPIFFS)` |
| 配网失败 / App 找不到 | NVS 残留旧凭据 / 未擦 flash | `Erase All Flash Before Sketch Upload: Enabled` 或 `esptool.py erase_flash` |
| 配网后无响应 | Wi-Fi 未连 / 非 2.4GHz | 代码中 `WiFi.begin`（非 BLE 配网时）；确认 2.4GHz |
| `Matter.begin()` 崩溃 | endpoint 未先 `begin()` | 先 `ep.begin()` 再 `Matter.begin()` |
| 回调不触发 | 注册顺序错 / 端点未初始化 | 配网前注册；`ep.begin()` 成功后再 `onChange` |
| Thread 设备连不上 | 预编译库不含 Thread | 用 ESP-IDF 组件方式重建（见 `idf_component_examples/Arduino_ESP_Matter_over_OpenThread/`） |
| C5 编译失败 | 预编译库仅 Thread | Matter over Wi-Fi 需 ESP-IDF 组件方式并禁用 Thread |
| OTA 不可用 | Huge APP 无 OTA 分区 | Matter 当前不支持 A/B OTA；如需 OTA 需自定义分区 |

## 参考

- `libraries/Matter/examples/MatterMinimum/`、`MatterOnOffLight/`
- `libraries/Matter/examples/MatterColorLight/`、`MatterDimmableLight/`、`MatterEnhancedColorLight/`
- `libraries/Matter/examples/MatterTemperatureSensor/`、`MatterContactSensor/`、`MatterSmartButton/`
- `libraries/Matter/examples/MatterEvents/`（事件监控）、`MatterStatus/`（连接状态）、`MatterCommissionTest/`
- `idf_component_examples/Arduino_ESP_Matter_over_OpenThread/`（Matter over Thread）
- 仓库文档 `docs/en/matter/matter.rst`、`matter_ep.rst`
- `docs/en/libraries.rst`（Matter 支持矩阵：Wi-Fi 与 Thread 分列）
