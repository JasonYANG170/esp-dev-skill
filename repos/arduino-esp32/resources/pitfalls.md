# arduino-esp32 常见陷阱汇总

> 取自 `docs/en/api/*.rst`、`docs/en/migration_guides/2.x_to_3.0.rst` 与库文档。每条配 WRONG / CORRECT。

## 1. v3.x LEDC 旧 API 已删（`ledcSetup`/`ledcAttachPin`）
- WRONG：`ledcSetup(0,5000,12); ledcAttachPin(5,0); ledcWrite(0,2048);`
- CORRECT：`ledcAttach(5,5000,12); ledcWrite(5,2048);`（以 pin 为索引，通道自动绑定）

## 2. v3.x 服务端用 NetworkServer.accept()
- WRONG：`WiFiServer.available()`
- CORRECT：`#include <Network.h>` + `NetworkServer server(80); server.accept();`

## 3. setHostname 必须在 WiFi 启动前
- WRONG：`WiFi.begin(...); WiFi.setHostname("x");`
- CORRECT：`WiFi.setHostname("x"); WiFi.mode(WIFI_STA); WiFi.begin(...);`
- 运行中改名：`WiFi.mode(WIFI_MODE_NULL)` → setHostname → 重启 WiFi

## 4. onEvent 回调在独立线程，非线程安全
- 回调内不可调用 `WiFi.onEvent/removeEvent`
- 访问共享变量需加锁；`Serial.print` 线程安全可用

## 5. setRxBufferSize / setPins / setHostname 必须在 begin 前
- Serial：`Serial1.setRxBufferSize(1024)` → `Serial1.begin(...)`
- I2C：`Wire.setPins(sda,scl)` → `Wire.begin()`
- Wi-Fi：见第 3 条

## 6. Preferences namespace/key ≤ 15 字符
- 超长静默返回 0/false
- 默认分区表含 `nvs` 分区；自定义分区表须自行加

## 7. ESP-NOW 必须先 WiFi.mode 再 begin
- WRONG：`ESP_NOW.begin(); WiFi.mode(STA);`
- CORRECT：`WiFi.mode(WIFI_STA); WiFi.setChannel(ch); while(!WiFi.STA.started()) delay(100); ESP_NOW.begin();`

## 8. loop() 不可长时间阻塞
- 长阻塞触发 loopTask 看门狗（Task WDT）
- 长任务用 `xTaskCreate` 独立任务；`loop()` 内用非阻塞状态机或 `delay` 短时让出

## 9. ADC 电压范围按衰减与芯片不同
- 11dB 衰减 ESP32/S3 才能测到 ~3.1V；其它 SoC 范围更小
- 用 `analogReadMilliVolts()` 而非手动换算（已校准）

## 10. LP UART 引脚固定（C5/C6/P4）
- 不可用 `setPins()` 改 LP UART 引脚
- 改用 HP UART

## 11. 自定义分区表命名与位置
- 文件必须叫 `partitions.csv` 且与 `.ino` 同目录才被自动拾取

## 12. 中断处理要短
- `attachInterrupt` ISR / Timer ISR 内禁 `Serial.print`、`delay`，只置 `volatile` 标志
- `IRAM_ATTR` 标注 ISR（推荐）

## 13. WiFiClient.flush() 行为变更（3.x）
- `flush()` 不再清接收缓冲；用 `clear()`

## 14. 受限引脚
- 输入-only 引脚、被 flash/PSRAM 占用引脚不可作输出，查 `variants/<board>/pins_arduino.h`

## 15. ESP32 仅 2.4 GHz Wi-Fi
- 连不上 5GHz；连接信号差用 `WiFi.setAutoReconnect(true)` + `setMinSecurity`
