# Changelog

## 1.1.0 — 2026-06-22

### 新增
- 5 个场景 recipe，填补审计确认的高价值缺口：
  - `recipes/ble.md` — BLE 扫描 / GATT Server / Client / UART / Notify / iBeacon/Eddystone / 配对绑定
    （来源：`libraries/BLE/examples/Scan`、`Server`、`Client`、`UART`、`Notify`、`iBeacon`、`Beacon_Scanner`、`EddystoneURL/TLM_Beacon`、`Server_secure_static_passkey`、`docs/en/api/ble.rst`、`libraries/BLE/README.md`）
  - `recipes/ota_update.md` — ArduinoOTA / Update / HTTPUpdate / OTAWebUpdater / 签名 OTA
    （来源：`libraries/ArduinoOTA/examples/BasicOTA`、`SignedOTA`、`libraries/Update/examples/HTTPS_OTA_Update`、`OTAWebUpdater`、`Signed_OTA_Update`、`libraries/HTTPUpdate/src/HTTPUpdate.h`、`libraries/Update/src/Update.h`、`docs/en/ota_web_update.rst`）
  - `recipes/zigbee.md` — Zigbee Coordinator/Router/End Device、灯/开关/sleepy 传感器/RCP 网关、绑定
    （来源：`libraries/Zigbee/examples/Zigbee_On_Off_Light`、`Zigbee_On_Off_Switch`、`Zigbee_Temp_Hum_Sensor_Sleepy`、`Zigbee_Gateway`、`Zigbee_Scan_Networks`、`docs/en/zigbee/*.rst`）
  - `recipes/matter.md` — Matter over Wi-Fi/Thread、灯/传感器/按钮端点、QR 配网、解绑
    （来源：`libraries/Matter/examples/MatterMinimum`、`MatterOnOffLight`、`MatterColorLight`、`MatterTemperatureSensor`、`MatterContactSensor`、`MatterSmartButton`、`docs/en/matter/*.rst`）
  - `recipes/webserver.md` — WebServer 路由 / Basic/Digest 认证 / 文件上传 / serveStatic / 中间件 / 网页 OTA
    （来源：`libraries/WebServer/examples/HelloServer`、`AdvancedWebServer`、`HttpBasicAuth`、`FSBrowser`、`WebUpdate`、`UploadHugeFile`、`libraries/WebServer/src/WebServer.h`）
- SKILL.md 新增 3 条核心原则（#13 BLE 协议栈差异、#14 Zigbee 芯片/角色/模式绑定、#15 Matter 分区与擦 flash）；
  Scenario Quick Reference 表新增上述 5 个 recipe；When to Use 与 Step 6 示例选择清单补全 BLE/Zigbee/Matter/WebServer/OTA。
- `resources/api_reference.md` 新增 BLE / WebServer / Update+ArduinoOTA+HTTPUpdate / Zigbee / Matter 五节 API 速查（签名均取自头文件与 rst 文档）。
- `resources/example_list.md` 扩充 BLE 示例（13 条）与 OTA 示例（7 条），新增 Zigbee 与 Matter 章节。
- tags 与 description/Trigger words 补充 ble/zigbee/matter/ota/webserver/HomeKit/GATT/iBeacon 等。

## 1.0.0 — 2026-06-18

首发版本。

### 新增
- 基于 arduino-esp32 仓库真实文档与源码构建（`docs/en/api/*.rst`、`docs/en/tutorials/*.rst`、`docs/en/migration_guides/2.x_to_3.0.rst`、`libraries/*/examples/`、`cores/esp32/main.cpp`）
- 13 个场景 recipe：GPIO/中断、Serial(UART)、I2C、SPI、LEDC/PWM、ADC、硬件定时器、Wi-Fi STA/AP、Preferences/NVS、ESP-NOW、FreeRTOS 多任务、深睡眠
- 资源文档：`api_reference.md`、`config_reference.md`、`pitfalls.md`、`example_list.md`
- SKILL.md 含 12 条原则、芯片/外设支持矩阵、12 条关键陷阱（WRONG/CORRECT 对照）、执行流程
- AGENTS.md 含工具链（Arduino CLI / IDE / ESP-IDF 组件）、代码生成清单
- 紧跟 v3.x：Peripheral Manager（LEDC）、统一 Network 库（NetworkServer.accept）
