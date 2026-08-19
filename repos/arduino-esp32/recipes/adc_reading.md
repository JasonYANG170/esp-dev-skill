# ADC 模拟量读取

> **适用摘要**: 使用 `analogRead` / `analogReadMilliVolts` 单次转换，以及 `analogContinuous` 连续模式对多通道后台采样并回调。含衰减（attenuation）与电压范围选择。

> Version: Arduino-ESP32 core version and selected board package.
> Evidence: `repos/arduino-esp32/resources/`, source/examples in `repos/arduino-esp32/`, and this recipe path `repos/arduino-esp32/recipes/adc_reading.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "读取模拟电压 / 传感器"
- "ADC 校准电压（毫伏）"
- "多通道连续采样"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ESP32/examples/AnalogRead/AnalogRead.ino`、`AnalogReadContinuous/AnalogReadContinuous.ino` |
| 引脚 | 仅 ADC 通道引脚可读，因 SoC 而异，查开发板引脚图 |
| 默认分辨率 | 12 位（0..4095）；ESP32-S2 rev v0.0 默认 13 位（ADC-112 errata） |

## 分步说明

### 单次读取

```cpp
void setup() {
    Serial.begin(115200);
    analogReadResolution(12);              // 1..16（ESP32 实改硬件 9..12）
    analogSetAttenuation(ADC_ATTEN_DB_11); // 全局衰减
}
void loop() {
    uint16_t raw = analogRead(34);
    uint32_t mv  = analogReadMilliVolts(34);   // 校准后毫伏
    Serial.printf("raw=%u  mv=%u\n", raw, mv);
    delay(500);
}
```

可测电压范围（节选自 `docs/en/api/adc.rst`，11dB 衰减）：

| SoC | ADC_ATTEN_DB_0 | ADC_ATTEN_DB_11 |
|---|---|---|
| ESP32 | 100~950 mV | 150~3100 mV |
| ESP32-S2 | 0~750 mV | 0~2500 mV |
| ESP32-C3 | 0~750 mV | 0~2500 mV |
| ESP32-S3 | 0~950 mV | 0~3100 mV |

按引脚单独设：`analogSetPinAttenuation(pin, ADC_ATTEN_DB_6)`。
ESP32 专用：`analogSetWidth(12)`（9..12，仅 ESP32 生效）。

### 连续模式（多通道 + 回调）

```cpp
const uint8_t pins[] = {34, 35};

void onConvDone(void *arg) {            // 可为 NULL
    // 转换完成回调
}

void setup() {
    Serial.begin(115200);
    analogContinuous(pins, 2, /*conv_per_pin*/4, /*sampling_hz*/20000, onConvDone);
    analogContinuousStart();
}
void loop() {
    adc_continuous_result_t *result = NULL;
    if (analogContinuousRead(&result, 100)) {
        for (int i = 0; i < 2; i++) {
            Serial.printf("pin=%u ch=%u raw=%d mv=%d\n",
                result[i].pin, result[i].channel,
                result[i].avg_read_raw, result[i].avg_read_mvolts);
        }
    }
    delay(500);
}
```

`adc_continuous_result_t` 字段：`pin`、`channel`、`avg_read_raw`、`avg_read_mvolts`。
连续模式 API：`analogContinuous` / `analogContinuousStart` / `analogContinuousStop` / `analogContinuousRead` / `analogContinuousDeinit` / `analogContinuousSetAtten` / `analogContinuousSetWidth`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 读值饱和在最大 | 超出当前衰减量程 | 提高衰减到 `ADC_ATTEN_DB_11` |
| 毫伏不准 | 未用校准函数 | 用 `analogReadMilliVolts` 而非 `raw*3300/4095` |
| `analogSetClockDiv`/`adcAttachPin` 编译失败 | 3.x 已删除 | 删除旧 API 调用 |
| 连续模式无回调 | 未 `analogContinuousStart` | 配置后必须 start |

## 参考

- `libraries/ESP32/examples/AnalogRead/AnalogRead.ino`
- `libraries/ESP32/examples/AnalogReadContinuous/AnalogReadContinuous.ino`
- 仓库文档 `docs/en/api/adc.rst`
