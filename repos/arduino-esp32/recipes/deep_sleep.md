# 深睡眠与唤醒 (Deep Sleep)

> **适用摘要**: 使用 `esp_deep_sleep_*` 与定时器 / 外部(GPIO) / 触摸唤醒源让 ESP32 进入低功耗深睡眠，并在唤醒后重启运行。

## 触发意图

- "低功耗 / 省电"
- "深睡眠定时唤醒"
- "GPIO 唤醒 / 触摸唤醒"
- "RTC 内存保持"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ESP32/examples/DeepSleep/TimerWakeUp/TimerWakeUp.ino`、`ExternalWakeUp/ExternalWakeUp.ino`、`TouchWakeUp/TouchWakeUp.ino` |
| 头文件 | `<esp_sleep.h>`（ESP-IDF，Arduino 可直接用） |
| 注意 | 深睡眠唤醒等同于复位，从 `setup()` 重新开始 |

## 分步说明

### 定时器唤醒

```cpp
#include <esp_sleep.h>
#define US_MIN(u) ((uint64_t)(u) * 1000ULL)
#define US_SEC(s) ((uint64_t)(s) * 1000000ULL)

void setup() {
    Serial.begin(115200);
    delay(100);
    Serial.println("Going to sleep for 10s");
    esp_sleep_enable_timer_wakeup(US_SEC(10));   // 微秒
    esp_deep_sleep_start();                       // 不返回；唤醒=复位
}
void loop() {}
```

### 外部唤醒（GPIO ext0/ext1）

```cpp
#define WAKE_PIN GPIO_NUM_13

// ext0：单个 RTC GPIO 高/低电平唤醒（仅 ESP32）
esp_sleep_enable_ext0_wakeup(WAKE_PIN, 0);       // 0=低电平唤醒

// ext1：多个 RTC GPIO 组合唤醒
// esp_sleep_enable_ext1_wakeup(BIT(GPIO_NUM_13) | BIT(GPIO_NUM_14),
//                              ESP_EXT1_WAKEUP_ANY_HIGH);

esp_deep_sleep_start();
```

> 仅 RTC GPIO 可做唤醒源（因 SoC 而异，查 datasheet）。ESP32-S3/C3 等 ext0 行为有差异，详见 ESP-IDF 文档。

### 触摸唤醒（仅 ESP32 / S2 / S3 支持 Touch）

```cpp
touchAttachInterrupt(T0, /*cb*/ nullptr, 40);   // 阈值 40
esp_sleep_enable_touchpad_wakeup();
esp_deep_sleep_start();
// 唤醒后：
touch_pad_t t = esp_sleep_get_touchpad_wakeup_status();
```

### 唤醒原因

```cpp
esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
switch (cause) {
    case ESP_SLEEP_WAKEUP_TIMER:    Serial.println("timer"); break;
    case ESP_SLEEP_WAKEUP_EXT0:     Serial.println("ext0");  break;
    case ESP_SLEEP_WAKEUP_EXT1:     Serial.println("ext1");  break;
    case ESP_SLEEP_WAKEUP_TOUCHPAD: Serial.println("touch"); break;
    case ESP_SLEEP_WAKEUP_UNDEFINED:Serial.println("reset/power-on"); break;
    default: break;
}
```

### RTC 内存保持（跨睡眠存活）

```cpp
RTC_DATA_ATTR int bootCount = 0;   // 存放在 RTC 内存，深睡眠不丢失（断电丢失）
void setup() { Serial.printf("boot %d\n", ++bootCount); }
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 不进入睡眠 | 未调用 `esp_deep_sleep_start()` 或前序 IO 阻塞 | 确认调用，且唤醒源已 enable |
| 唤醒不了 | 用了非 RTC GPIO / 引脚未接 | 改用 RTC GPIO；ext0 仅 ESP32 |
| 功耗仍高 | 外设未关 / 上拉漏电 | 必要时 `esp_bluedroid_disable`/`WiFi.mode(WIFI_OFF)`，断开外设电源 |
| `RTC_DATA_ATTR` 复位后丢 | 完全断电 | RTC 内存仅在深睡眠保持；断电即丢 |

## 参考

- `libraries/ESP32/examples/DeepSleep/TimerWakeUp/TimerWakeUp.ino`
- `libraries/ESP32/examples/DeepSleep/ExternalWakeUp/ExternalWakeUp.ino`
- `libraries/ESP32/examples/DeepSleep/TouchWakeUp/TouchWakeUp.ino`
- 仓库文档 `docs/en/api/deepsleep.rst`
