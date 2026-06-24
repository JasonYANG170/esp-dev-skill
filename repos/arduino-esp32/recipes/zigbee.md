# Zigbee 3.0 设备（Coordinator / Router / End Device，on-off/light/sensor/网关）

> **适用摘要**: 使�� `Zigbee` 库（基于 ESP-ZIGBEE-SDK）创建 Zigbee 终端设备/路由/协调器，注册 `ZigbeeEP` 端点（灯/开关/传感器/网关），含组网、绑定、睡眠终端与 RCP 网关。⚠️ 仅 ESP32-C6/C5/H2 有原生 802.15.4 无线电；其它 SoC 需外接 RCP。

## 触发意图

- "Zigbee 设备 / 智能家居"
- "Zigbee 协调器 / 路由 / 终端"
- "Zigbee 灯 / 开关 / 传感器"
- "Zigbee 网络 / 入网 / 扫描"
- "Zigbee 绑定 / sleepy 终端 / 网关"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/Zigbee/examples/Zigbee_On_Off_Light/`、`Zigbee_On_Off_Switch/`、`Zigbee_Temp_Hum_Sensor_Sleepy/`、`Zigbee_Gateway/`、`Zigbee_Scan_Networks/` |
| 芯片 | 原生 802.15.4：ESP32-C6/C5/H2；其它（ESP32/S3/C3）需外接 Zigbee RCP 模组（见 `Zigbee_Gateway`） |
| 工具菜单 | Arduino IDE: Tools → Zigbee mode 必须匹配角色；Partition Scheme 选匹配的 `Zigbee xMB with spiffs` |
| 角色↔模式 | `ZIGBEE_COORDINATOR`/`ZIGBEE_ROUTER` → mode `Zigbee ZCZR`；`ZIGBEE_END_DEVICE` → mode `Zigbee ED` |

## 分步说明

### 库结构与启动顺序

`Zigbee`（单例 `ZigbeeCore`）管网络；`ZigbeeEP` 是所有端点基类；具体类如 `ZigbeeLight`/`ZigbeeSwitch`/`ZigbeeTempSensor`/`ZigbeeGateway` 继承自 `ZigbeeEP`。统一流程：

1. 实例化端点 → 2. `setManufacturerAndModel()` / `setPowerSource()` 等可选配置 → 3. `Zigbee.addEndpoint(&ep)` → 4. `Zigbee.begin(role)` → 5. 等 `Zigbee.connected()` → 6. `loop()` 处理业务 / `factoryReset()`。

### 角色 / 模式 / 分区表

```cpp
// 角色（zigbee_role_t）
ZIGBEE_COORDINATOR = 0;   // 建网、管理，存储网络信息
ZIGBEE_ROUTER      = 1;   // 入网、中继、常供电（灯/开关）
ZIGBEE_END_DEVICE  = 2;   // 入网、可睡眠（电池传感器）

// begin 重载
bool Zigbee.begin(zigbee_role_t role = ZIGBEE_END_DEVICE, bool erase_nvs = false);
bool Zigbee.begin(esp_zb_cfg_t *role_cfg, bool erase_nvs = false);  // 自定义配置
```

> ⚠️ 角色与 Tools 菜单绑定：示例用 `#ifndef ZIGBEE_MODE_ZCZR #error ...` / `#ifndef ZIGBEE_MODE_ED #error ...` 在编译期校验。选错会编译失败或运行异常。

### End Device：On/Off 灯（取自 Zigbee_On_Off_Light 示例）

```cpp
#include <Arduino.h>
#ifndef ZIGBEE_MODE_ED
#error "Zigbee end device mode is not selected in Tools->Zigbee mode"
#endif
#include "Zigbee.h"

#define ZIGBEE_LIGHT_ENDPOINT 10
ZigbeeLight zbLight = ZigbeeLight(ZIGBEE_LIGHT_ENDPOINT);

void setLED(bool value) { digitalWrite(RGB_BUILTIN, value); }

void setup() {
  Serial.begin(115200);
  pinMode(RGB_BUILTIN, OUTPUT); digitalWrite(RGB_BUILTIN, LOW);
  pinMode(BOOT_PIN, INPUT_PULLUP);   // 用于恢复出厂

  zbLight.setManufacturerAndModel("Espressif", "ZBLightBulb");
  zbLight.onLightChange(setLED);     // 远程开关回调

  Zigbee.addEndpoint(&zbLight);
  if (!Zigbee.begin()) { Serial.println("Zigbee failed!"); ESP.restart(); }
  while (!Zigbee.connected()) { Serial.print("."); delay(100); }
}
void loop() {
  // BOOT 键按住 >3s 恢复出厂；短按切换灯
  if (digitalRead(BOOT_PIN) == LOW) {
    delay(100); uint32_t t = millis();
    while (digitalRead(BOOT_PIN) == LOW) {
      if (millis() - t > 3000) { Zigbee.factoryReset(); }   // 默认 restart=true
      delay(50);
    }
    zbLight.setLight(!zbLight.getLightState());
  }
  delay(100);
}
```

### Coordinator：On/Off 开关（取自 Zigbee_On_Off_Switch 示例）

```cpp
#ifndef ZIGBEE_MODE_ZCZR
#error "Zigbee coordinator mode is not selected in Tools->Zigbee mode"
#endif
#include "Zigbee.h"

#define SWITCH_ENDPOINT_NUMBER 5
ZigbeeSwitch zbSwitch = ZigbeeSwitch(SWITCH_ENDPOINT_NUMBER);

void setup() {
  Serial.begin(115200);
  zbSwitch.setManufacturerAndModel("Espressif", "ZigbeeSwitch");
  zbSwitch.allowMultipleBinding(true);          // 允许多灯绑定
  zbSwitch.onLightStateChange([](bool s){ Serial.printf("Light=%d\n", s); });

  Zigbee.addEndpoint(&zbSwitch);
  Zigbee.setRebootOpenNetwork(180);             // 重启后开网 180s 接纳入网请求
  if (!Zigbee.begin(ZIGBEE_COORDINATOR)) { ESP.restart(); }

  while (!zbSwitch.bound()) { delay(500); }     // 等待灯绑定过来
  std::list<zb_device_params_t *> lights = zbSwitch.getBoundDevices();
  for (auto &d : lights) {
    char *mfr = zbSwitch.readManufacturer(d->endpoint, d->short_addr, d->ieee_addr);
    char *mdl = zbSwitch.readModel(d->endpoint, d->short_addr, d->ieee_addr);
  }
}
void loop() {
  // 按键 → zbSwitch.lightToggle();
}
```

### Sleepy End Device：温湿度传感器（取自 Sleepy 示例）

```cpp
#ifndef ZIGBEE_MODE_ED
#error "Zigbee end device mode is not selected in Tools->Zigbee mode"
#endif
#include "Zigbee.h"
#include <esp_sleep.h>

#define TEMP_SENSOR_ENDPOINT_NUMBER 10
ZigbeeTempSensor zbTemp(TEMP_SENSOR_ENDPOINT_NUMBER);

void setup() {
  Serial.begin(115200);
  esp_sleep_enable_timer_wakeup(55ULL * 1000000ULL);    // 睡 55s + 连接 5s ≈ 每分钟上报

  zbTemp.setManufacturerAndModel("Espressif", "SleepyZigbeeTempSensor");
  zbTemp.setMinMaxValue(10, 50);
  zbTemp.setDefaultValue(10.0);
  zbTemp.setTolerance(1);                                // °C，最小 0.01
  zbTemp.setPowerSource(ZB_POWER_SOURCE_BATTERY, 100, 35);  // 电池 100% / 3.5V
  zbTemp.addHumiditySensor(0, 100, 1, 0.0);              // 加湿度簇（min/max/tol/default）

  Zigbee.onGlobalDefaultResponse([](zb_cmd_type_t cmd, esp_zb_zcl_status_t st,
                                    uint8_t ep, uint16_t cluster){
    if (cmd == ZB_CMD_REPORT_ATTRIBUTE && ep == TEMP_SENSOR_ENDPOINT_NUMBER) {
      // ESP_ZB_ZCL_STATUS_SUCCESS / _FAIL / _TIMEOUT
    }
  });

  Zigbee.addEndpoint(&zbTemp);

  esp_zb_cfg_t cfg = ZIGBEE_DEFAULT_ED_CONFIG();
  cfg.nwk_cfg.zed_cfg.keep_alive = 10000;    // 10s keep alive，避免与上报冲突
  Zigbee.setTimeout(10000);                  // 缩短入网超时省电（默认 30s）
  if (!Zigbee.begin(&cfg, false)) ESP.restart();
  while (!Zigbee.connected()) { delay(100); }

  xTaskCreate([](void *){
    while (true) {
      zbTemp.setTemperature(temperatureRead());
      zbTemp.setHumidity(temperatureRead());     // 演示
      zbTemp.report();                            // 上报温/湿度
      // 等 onGlobalDefaultResponse 确认成功后 esp_deep_sleep_start()
      esp_deep_sleep_start();
    }
  }, "sensor", 3072, NULL, 10, NULL);
}
```

### Coordinator/Router：RCP 网关（取自 Zigbee_Gateway 示例）

```cpp
#ifndef ZIGBEE_MODE_ZCZR
#error "Zigbee coordinator mode is not selected in Tools->Zigbee mode"
#endif
#include "Zigbee.h"
#include <WiFi.h>
#include <esp_sntp.h>

ZigbeeGateway zbGateway(1);    // endpoint 1

void setup() {
  Serial.begin(115200);
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) { delay(500); }

  zbGateway.setManufacturerAndModel("Espressif", "ZigbeeGateway");
  zbGateway.addTimeCluster(timeinfo, gmtOffset_sec);   // 时间同步簇
  Zigbee.addEndpoint(&zbGateway);
  Zigbee.setRebootOpenNetwork(180);

  // 外接 RCP（如 ESP32-H2）通过 UART 通信
  esp_zb_radio_config_t radio = ZIGBEE_DEFAULT_UART_RCP_RADIO_CONFIG();
  radio.radio_uart_config.port = UART_NUM_1;
  radio.radio_uart_config.rx_pin = (gpio_num_t)4;
  radio.radio_uart_config.tx_pin = (gpio_num_t)5;
  Zigbee.setRadioConfig(radio);

  if (!Zigbee.begin(ZIGBEE_COORDINATOR)) ESP.restart();
  configTime(3600, 3600, "pool.ntp.org", "time.nist.gov");
}
```

### 网络扫描（取自 Zigbee_Scan_Networks 示例）

```cpp
Zigbee.scanNetworks();                       // 默认全信道、scan_duration=5
// int16_t scanComplete(): -2=失败/-1=进行中/0=无网/>0=找到数
zigbee_scan_result_t *r = Zigbee.getScanResult();
// 字段：short_pan_id / logic_channel / permit_joining / router_capacity /
//       end_device_capacity / extended_pan_id[8]
Zigbee.scanDelete();                         // 释放结果内存
```

### ZigbeeEP 通用 API（取自 zigbee_ep.rst）

```cpp
ep.setManufacturerAndModel("Mfr", "Model");  // 各 ≤32 字符
ep.setVersion(1); ep.setHardwareVersion(1);  // 必须在 begin() 前
ep.setPowerSource(ZB_POWER_SOURCE_MAINS);    // 或 _BATTERY/_BATTERY_BACKUP
ep.setBatteryPercentage(80); ep.reportBatteryPercentage();
ep.addTimeCluster(tm_time, gmt_offset_sec);  // 时间同步
ep.addOTAClient(file_ver, dl_ver, hw_ver);   // OTA 客户端
ep.bound();                                  // 是否有绑定设备
ep.getBoundDevices();                        // std::list<zb_device_params_t*>
ep.allowMultipleBinding(true); ep.setManualBinding(false);
ep.clearBoundDevices();
ep.onIdentify([](uint16_t time){});
ep.onDefaultResponse([](zb_cmd_type_t, esp_zb_zcl_status_t){});
```

### ZigbeeCore 通用 API

```cpp
Zigbee.factoryReset(true);                   // 清网络配置并重启
Zigbee.setDebugMode(true);                   // 调试输出（建议同时选 *debug* 模式）
Zigbee.openNetwork(180); Zigbee.closeNetwork();
Zigbee.setPrimaryChannelMask(0x07FFF800);    // 信道 11-26
Zigbee.scanNetworks(ESP_ZB_TRANSCEIVER_ALL_CHANNELS_MASK, 5);
Zigbee.setTimeout(30000);                    // ms，默认 30s
Zigbee.setRxOnWhenIdle(false);               // 终端可睡眠
Zigbee.onGlobalDefaultResponse(cb);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译期 `#error Zigbee ... mode is not selected` | Tools → Zigbee mode 与角色不符 | ED 角色选 `Zigbee ED`；Coordinator/Router 选 `Zigbee ZCZR` |
| `begin()` 失败重启 | 分区方案不匹配 / NVS 脏 | 选对应 `Zigbee xMB with spiffs`；`Zigbee.begin(role, /*erase_nvs=*/true)` |
| 设备无法入网 | 协调器未开网 / 信道不匹配 | `setRebootOpenNetwork(180)` / `openNetwork(180)`；核对信道掩码 |
| Sleepy 设备掉线 | keep alive 过长 / 父节点休眠 | 缩短 keep_alive（10s）；保证父路由常供电 |
| `bound()` 一直 false | 未绑定 / 缺 `allowMultipleBinding` | 调用 `allowMultipleBinding(true)`；终端支持对应簇 |
| ESP32/S3/C3 编译失败 | 无原生 802.15.4 无线电 | 用 RCP 模式（`ZigbeeGateway` + `setRadioConfig`）外接 H2/C6 |
| 上报后无确认 | 未注册 `onGlobalDefaultResponse` | 注册回调判断 `ESP_ZB_ZCL_STATUS_SUCCESS` 后再睡眠 |
| 配网缓存导致异常 | 旧网络信息残留 | `Zigbee.factoryReset()` 或 `begin(role, true)` 擦 NVS |

## 参考

- `libraries/Zigbee/examples/Zigbee_On_Off_Light/`、`Zigbee_On_Off_Switch/`
- `libraries/Zigbee/examples/Zigbee_Temp_Hum_Sensor_Sleepy/`、`Zigbee_Gateway/`、`Zigbee_Scan_Networks/`
- `libraries/Zigbee/examples/Zigbee_Dimmable_Light/`、`Zigbee_Color_Dimmer_Light/`、`Zigbee_Window_Covering/`
- 仓库文档 `docs/en/zigbee/zigbee.rst`、`zigbee_core.rst`、`zigbee_ep.rst`
- `docs/en/libraries.rst`（芯片支持矩阵：Zigbee 仅 C6/C5/H2 原生）
