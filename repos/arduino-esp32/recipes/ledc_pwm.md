# LEDC (PWM) 输出与调光

> **适用摘要**: 使用 v3.x LEDC Peripheral Manager API（`ledcAttach`/`ledcWrite`）输出 PWM、呼吸灯、播放音符（`ledcWriteNote`）。⚠️ 2.x 的 `ledcSetup`/`ledcAttachPin` 已删除。

## 触发意图

- "输出 PWM / 调光"
- "LED 呼吸灯"
- "蜂鸣器播放音符"
- "RGB LED 控制"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ESP32/examples/AnalogOut/LEDCSoftwareFade/LEDCSoftwareFade.ino`、`ledcWrite_RGB/ledcWrite_RGB.ino`、`LEDCFade/LEDCFade.ino` |
| 通道数 | 因 SoC 而异：ESP32=16、S2=8、S3=8、C3=6、C6=6、H2=6 |
| 分辨率 | 1–14 位（ESP32 为 1–20 位） |

## 分步说明

### 基本 PWM（引脚自动绑定通道）

```cpp
#define LED_PIN 5
void setup() {
    ledcAttach(LED_PIN, 5000, 12);   // pin, freq(Hz), resolution(bits)
}
void loop() {
    for (int d = 0; d < 4096; d += 16) {
        ledcWrite(LED_PIN, d);       // 以 pin 为索引设置占空比
        delay(10);
    }
}
```

> 占空比范围由分辨率决定：12 位 → 0..4095。

### 硬件渐变 ledcFade

```cpp
ledcAttach(LED_PIN, 5000, 12);
ledcFade(LED_PIN, /*start*/0, /*target*/4095, /*time_ms*/1000);
// 或带中断：
// ledcFadeWithInterrupt(LED_PIN, 0, 4095, 1000, onFadeDone);
```

### 播放音符

```cpp
ledcAttach(BUZZER_PIN, 0, 8);                      // 频率先占位
ledcWriteNote(BUZZER_PIN, NOTE_A, 4);              // 第 4 八度的 A
// 音符枚举：NOTE_C NOTE_Cs NOTE_D NOTE_Eb NOTE_E NOTE_F NOTE_Fs NOTE_G NOTE_Gs NOTE_A NOTE_Bb NOTE_B
```

`ledcWriteTone(pin, freq)`：输出 50% 占空比方波，`freq=0` 时停止。

### 多引脚共用同一通道（同步输出）

```cpp
ledcAttachChannel(R1, 5000, 8, /*channel*/0);      // 首次设定 freq/resolution
ledcAttachChannel(G1, 5000, 8, 0);                 // 后续忽略 freq/resolution，共用 channel 0
ledcWriteChannel(0, 128);                          // 两个引脚同步
```

### 兼容 Arduino `analogWrite`

```cpp
analogWrite(LED_PIN, 128);                // 0..255
analogWriteFrequency(LED_PIN, 1000);
analogWriteResolution(LED_PIN, 10);
```

其它 API：`ledcRead(pin)`、`ledcReadFreq(pin)`、`ledcChangeFrequency(pin,freq,res)`、`ledcOutputInvert(pin,true)`、`ledcDetach(pin)`、`ledcSetClockSource(LEDC_APB_CLK)`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `ledcSetup` 未定义 | 用了 2.x 旧 API | 改用 `ledcAttach(pin, freq, resolution)` |
| `ledcAttach` 返回 false | 分辨率越界或无可用通道 | 检查分辨率范围；查 SoC 通道数 |
| `ledcWrite` 写到旧 channel 号 | 3.x 以 pin 为索引 | 用引脚号而非 channel 号 |
| 输出反相 | 默认高电平为占空 | 用 `ledcOutputInvert(pin, true)` |

## 参考

- `libraries/ESP32/examples/AnalogOut/LEDCSoftwareFade/LEDCSoftwareFade.ino`
- `libraries/ESP32/examples/AnalogOut/ledcWrite_RGB/ledcWrite_RGB.ino`
- `libraries/ESP32/examples/AnalogOut/LEDCFade/LEDCFade.ino`
- 仓库文档 `docs/en/api/ledc.rst`、`docs/en/migration_guides/2.x_to_3.0.rst`
