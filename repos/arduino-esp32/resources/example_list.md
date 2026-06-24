# arduino-esp32 仓库示例索引

> 所有路径相对于仓库根 `D:/esp-skill/espressif-repos/arduino-esp32/`。仅列举真实存在的示例（取自 `libraries/*/examples/` 与 `libraries/ESP32/examples/`）。

## 核心 ESP32 库示例（`libraries/ESP32/examples/`）

| 路径 | 说明 |
|---|---|
| `libraries/ESP32/examples/GPIO/GPIOInterrupt/GPIOInterrupt.ino` | GPIO 外部中断 |
| `libraries/ESP32/examples/AnalogRead/AnalogRead.ino` | ADC 单次读取（analogRead） |
| `libraries/ESP32/examples/AnalogReadContinuous/AnalogReadContinuous.ino` | ADC 连续模式 |
| `libraries/ESP32/examples/AnalogOut/LEDCSoftwareFade/LEDCSoftwareFade.ino` | LEDC 软件呼吸灯 |
| `libraries/ESP32/examples/AnalogOut/LEDCFade/LEDCFade.ino` | LEDC 硬件渐变 |
| `libraries/ESP32/examples/AnalogOut/ledcWrite_RGB/ledcWrite_RGB.ino` | LEDC 驱动 RGB LED |
| `libraries/ESP32/examples/Timer/RepeatTimer/RepeatTimer.ino` | 硬件定时器周期中断 |
| `libraries/ESP32/examples/Timer/WatchdogTimer/WatchdogTimer.ino` | 看门狗定时器 |
| `libraries/ESP32/examples/Serial/OnReceive_Demo/OnReceive_Demo.ino` | Serial onReceive 回调 |
| `libraries/ESP32/examples/Serial/RS485_Echo_Demo/RS485_Echo_Demo.ino` | RS485 半双工 |
| `libraries/ESP32/examples/Serial/HardwareFlowControl_Demo/HardwareFlowControl_Demo.ino` | UART 硬件流控 |
| `libraries/ESP32/examples/Serial/BaudRateDetect_Demo/BaudRateDetect_Demo.ino` | 自动波特率检测 |
| `libraries/ESP32/examples/Serial/IrdaMode_DualUART_Demo/IrdaMode_DualUART_Demo.ino` | IrDA 模式 |
| `libraries/ESP32/examples/DeepSleep/TimerWakeUp/TimerWakeUp.ino` | 深睡眠定时唤醒 |
| `libraries/ESP32/examples/DeepSleep/ExternalWakeUp/ExternalWakeUp.ino` | 深睡眠外部唤醒 |
| `libraries/ESP32/examples/DeepSleep/TouchWakeUp/TouchWakeUp.ino` | 深睡眠触摸唤醒 |
| `libraries/ESP32/examples/FreeRTOS/BasicMultiThreading/` | FreeRTOS 多任务 |
| `libraries/ESP32/examples/FreeRTOS/Queue/` | FreeRTOS 队列 |
| `libraries/ESP32/examples/FreeRTOS/Semaphore/` | FreeRTOS 信号量 |
| `libraries/ESP32/examples/FreeRTOS/Mutex/` | FreeRTOS 互斥 |
| `libraries/ESP32/examples/Touch/TouchRead/` | 触摸读取 |
| `libraries/ESP32/examples/Touch/TouchButton/` | 触摸按键 |
| `libraries/ESP32/examples/Touch/TouchInterrupt/` | 触摸中断 |
| `libraries/ESP32/examples/RMT/RMTWrite_RGB_LED/` | RMT 驱动 WS2812 RGB |
| `libraries/ESP32/examples/RMT/RMTLoopback/` | RMT 回环 |
| `libraries/ESP32/examples/RMT/RMTCallback/` | RMT 回调 |
| `libraries/ESP32/examples/TWAI/TWAIreceive/`、`TWAItransmit/` | TWAI(CAN) 收发 |
| `libraries/ESP32/examples/Camera/CameraWebServer/` | 摄像头 Web 服务（S3 等） |
| `libraries/ESP32/examples/Time/SimpleTime/` | NTP 对时 |
| `libraries/ESP32/examples/ChipID/`、`MacAddress/`、`ResetReason/` | 芯片 ID / MAC / 复位原因 |

## Wi-Fi 库（`libraries/WiFi/examples/`）

| 路径 | 说明 |
|---|---|
| `libraries/WiFi/examples/WiFiClient/WiFiClient.ino` | STA 连接 + TCP 客户端 |
| `libraries/WiFi/examples/WiFiClientBasic/` | 基础 STA 连接 |
| `libraries/WiFi/examples/WiFiClientEvents/WiFiClientEvents.ino` | Wi-Fi 事件回调 |
| `libraries/WiFi/examples/WiFiClientStaticIP/` | 静态 IP |
| `libraries/WiFi/examples/WiFiClientEnterprise/` | WPA2-Enterprise |
| `libraries/WiFi/examples/WiFiAccessPoint/WiFiAccessPoint.ino` | SoftAP + Web |
| `libraries/WiFi/examples/SimpleWiFiServer/SimpleWiFiServer.ino` | 简单 TCP 服务端 |
| `libraries/WiFi/examples/WiFiScan/WiFiScan.ino` | 同步扫描 |
| `libraries/WiFi/examples/WiFiScanAsync/` | 异步扫描 |
| `libraries/WiFi/examples/WiFiMulti/`、`WiFiMultiAdvanced/`、`WiFiMultiEnterprise/` | 多 AP |
| `libraries/WiFi/examples/WiFiSmartConfig/` | SmartConfig |
| `libraries/WiFi/examples/WPS/` | WPS |
| `libraries/WiFi/examples/WiFiIPv6/` | IPv6 |
| `libraries/WiFi/examples/FTM/` | Wi-Fi FTM 测距（S2/C3） |
| `libraries/WiFi/examples/WiFiTelnetToSerial/` | Telnet 转 Serial |
| `libraries/WiFi/examples/WiFiExtender/` | STA→AP 中继 |

## 网络 / 应用库

| 路径 | 说明 |
|---|---|
| `libraries/WebServer/examples/HelloServer/` | 最小 HTTP 服务 |
| `libraries/WebServer/examples/AdvancedWebServer/` | 高级 HTTP 服务 |
| `libraries/WebServer/examples/HttpBasicAuth/` 等 | HTTP 认证 |
| `libraries/WebServer/examples/FSBrowser/` | 文件系统浏览 |
| `libraries/HTTPClient/examples/BasicHttpClient/` | HTTP GET |
| `libraries/HTTPClient/examples/BasicHttpsClient/` | HTTPS |
| `libraries/ESPmDNS/examples/mDNS_Web_Server/` | mDNS |
| `libraries/ArduinoOTA/examples/BasicOTA/` | 基础 OTA（IDE/espota 推送） |
| `libraries/ArduinoOTA/examples/SignedOTA/` | 签名 OTA（RSA/ECDSA 验签） |
| `libraries/Update/examples/HTTPS_OTA_Update/` | HTTPS OTA 下载 |
| `libraries/Update/examples/OTAWebUpdater/` | 浏览器上传 OTA |
| `libraries/Update/examples/Signed_OTA_Update/` | 签名 OTA（Update 库层） |
| `libraries/Update/examples/AWS_S3_OTA_Update/`、`SD_Update/` | S3 / SD 卡 OTA |
| `libraries/HTTPUpdateServer/examples/WebUpdater/` | HTTPUpdateServer 网页 OTA |
| `libraries/WiFiProv/examples/WiFiProv/` | Wi-Fi 配网（SoftAPProvisioning / BLE Provisioning） |

## 存储 / 文件系统

| 路径 | 说明 |
|---|---|
| `libraries/Preferences/examples/StartCounter/StartCounter.ino` | Preferences/NVS 计数器 |
| `libraries/EEPROM/examples/eeprom_write/`、`eeprom_class/`、`eeprom_extra/` | EEPROM 兼容 |
| `libraries/LittleFS/examples/LITTLEFS_test/`、`LITTLEFS_time/` | LittleFS |
| `libraries/SD/examples/SD_Test/`、`SD_time/` | SD 卡（SPI） |
| `libraries/FFat/examples/FFat_Test/`、`FFat_time/` | FAT 文件系统（wear leveling） |
| `libraries/SPIFFS/examples/SPIFFS_Test/`、`SPIFFS_time/` | SPIFFS |

## 通信外设

| 路径 | 说明 |
|---|---|
| `libraries/Wire/examples/WireMaster/WireMaster.ino` | I2C 主机 |
| `libraries/Wire/examples/WireSlave/WireSlave.ino` | I2C 从机 |
| `libraries/SPI/examples/SPI_Multiple_Buses/SPI_Multiple_Buses.ino` | SPI 多总线 |
| `libraries/ESP_NOW/examples/ESP_NOW_Broadcast_Master/`、`ESP_NOW_Broadcast_Slave/` | ESP-NOW 广播 |
| `libraries/ESP_NOW/examples/ESP_NOW_Serial/` | ESP-NOW 串口透传 |

## 蓝牙

| 路径 | 说明 |
|---|---|
| `libraries/BluetoothSerial/examples/SerialToSerialBT/` | 经典蓝牙串口 |
| `libraries/BluetoothSerial/examples/SerialToSerialBT_SSP/` | SSP 配对 |
| `libraries/BluetoothSerial/examples/bt_classic_device_discovery/` | 经典蓝牙发现 |
| `libraries/BLE/examples/Scan/Scan.ino` | BLE 扫描 |
| `libraries/BLE/examples/Server/Server.ino` | GATT Server（读写） |
| `libraries/BLE/examples/Client/Client.ino` | GATT Client（连接+读+订阅） |
| `libraries/BLE/examples/UART/UART.ino` | BLE UART 透传（Nordic UART Service） |
| `libraries/BLE/examples/Notify/Notify.ino` | Notify 通知（CCCD 2902 + 2901） |
| `libraries/BLE/examples/iBeacon/iBeacon.ino` | iBeacon 广播 |
| `libraries/BLE/examples/Beacon_Scanner/Beacon_Scanner.ino` | iBeacon/Eddystone ��标扫描 |
| `libraries/BLE/examples/EddystoneURL_Beacon/`、`EddystoneTLM_Beacon/` | Eddystone URL/TLM 信标 |
| `libraries/BLE/examples/Server_secure_static_passkey/`、`Client_secure_static_passkey/` | 安全配对（静态密钥） |
| `libraries/BLE/examples/Server_multiconnect/`、`Client_multiconnect/`、`Client_Server/` | 多连接 / 双角色 |
| `libraries/BLE/examples/BLE5_extended_scan/`、`BLE5_multi_advertising/`、`BLE5_periodic_advertising/`、`BLE5_periodic_sync/` | BLE 5 扩展广播/周期广播 |

## Zigbee（仅 ESP32-C6/C5/H2 原生，其它 SoC 需 RCP）

| 路径 | 说明 |
|---|---|
| `libraries/Zigbee/examples/Zigbee_On_Off_Light/Zigbee_On_Off_Light.ino` | End Device：on/off 灯 |
| `libraries/Zigbee/examples/Zigbee_On_Off_Switch/Zigbee_On_Off_Switch.ino` | Coordinator：on/off 开关 |
| `libraries/Zigbee/examples/Zigbee_Temp_Hum_Sensor_Sleepy/Zigbee_Temp_Hum_Sensor_Sleepy.ino` | Sleepy End Device：温湿度传感器 |
| `libraries/Zigbee/examples/Zigbee_Gateway/Zigbee_Gateway.ino` | Coordinator：RCP 网关 + NTP 时间同步 |
| `libraries/Zigbee/examples/Zigbee_Scan_Networks/Zigbee_Scan_Networks.ino` | Zigbee 网络扫描 |
| `libraries/Zigbee/examples/Zigbee_Dimmable_Light/`、`Zigbee_Color_Dimmer_Light/` | 可调光/调色灯 |
| `libraries/Zigbee/examples/Zigbee_Window_Covering/` | 窗帘电机 |
| `libraries/Zigbee/examples/Zigbee_OTA/`（若存在） | Zigbee OTA 升级 |

## Matter（Wi-Fi / Thread，需 Huge APP 分区）

| 路径 | 说明 |
|---|---|
| `libraries/Matter/examples/MatterMinimum/MatterMinimum.ino` | 最小 Matter 设备（on/off 灯） |
| `libraries/Matter/examples/MatterOnOffLight/MatterOnOffLight.ino` | On/Off 灯 + 状态持久化 + 按键控制 |
| `libraries/Matter/examples/MatterColorLight/`、`MatterDimmableLight/`、`MatterEnhancedColorLight/` | 彩色/可调光/增强灯 |
| `libraries/Matter/examples/MatterTemperatureSensor/`、`MatterContactSensor/`、`MatterHumiditySensor/` | 各类传感器 |
| `libraries/Matter/examples/MatterSmartButton/` | 智能按钮（Generic Switch） |
| `libraries/Matter/examples/MatterEvents/`、`MatterStatus/`、`MatterCommissionTest/` | 事件监控 / 状态 / 配网测试 |
| `idf_component_examples/Arduino_ESP_Matter_over_OpenThread/` | Matter over Thread（IDF 组件方式） |

## 定时器 / 系统

| 路径 | 说明 |
|---|---|
| `libraries/Ticker/examples/Blinker/`、`TickerBasic/`、`TickerParameter/` | Ticker 周期回调 |

## ESP-IDF 组件示例（`idf_component_examples/`）

| 路径 | 说明 |
|---|---|
| `idf_component_examples/hello_world/` | Arduino 作为 IDF 组件的最小工程 |
| `idf_component_examples/hw_cdc_hello_world/` | 硬件 CDC |
| `idf_component_examples/Arduino_ESP_Matter_over_OpenThread/` | Matter over Thread |
