# GPIO 输出输入与中断

> **适用摘要**: 使用 Arduino 标准 `pinMode`/`digitalWrite`/`digitalRead` 控制引脚，并用 `attachInterrupt` 响应外部中断（按键、脉冲等）。
> Excludes: ESP-IDF-only projects and ESP8266 RTOS SDK projects.
> Arduino: Verify the selected Arduino-ESP32 core version and board variant before assigning pins.
> Evidence: `repos/arduino-esp32/resources/api_reference.md`, `libraries/ESP32/examples/GPIO/GPIOInterrupt/GPIOInterrupt.ino`, and board `variants/<board>/pins_arduino.h`.
> Validation: example-derived.

## 触发意图

- "点亮 LED / 闪烁"
- "读取按键 / 按钮状态"
- "GPIO 中断 / 外部中断"
- "attachInterrupt 用法"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ESP32/examples/GPIO/GPIOInterrupt/GPIOInterrupt.ino` |
| 受限引脚 | 部分引脚为输入-only / 被 flash-PSRAM 占用，需查开发板 `variants/<board>/pins_arduino.h` |

## 分步说明

### 基本输出（LED 闪烁）

```cpp
#define LED 2   // 多数 ESP32 开发板板载 LED 在 GPIO2

void setup() {
    pinMode(LED, OUTPUT);
}
void loop() {
    digitalWrite(LED, HIGH);
    delay(500);
    digitalWrite(LED, LOW);
    delay(500);
}
```

### 输入 + 内部上下拉

ESP32 内部上下拉电阻约 45 kΩ。配置为 `INPUT` 而不指定上下拉时为高阻。

```cpp
#define BUTTON 0
pinMode(BUTTON, INPUT_PULLUP);   // 启用内部上拉
bool pressed = (digitalRead(BUTTON) == LOW);
```

`pinMode` 支持的模式：`INPUT`、`OUTPUT`、`INPUT_PULLDOWN`、`INPUT_PULLUP`。

### 外部中断

中断模式：`DISABLED` / `RISING` / `FALLING` / `CHANGE` / `ONLOW` / `ONHIGH` / `ONLOW_WE` / `ONHIGH_WE`。

```cpp
#define BUTTON 0
volatile int interruptCount = 0;

void IRAM_ATTR isr() {           // 中断服务：极短，仅置标志
    interruptCount++;
}

void setup() {
    Serial.begin(115200);
    pinMode(BUTTON, INPUT_PULLUP);
    attachInterrupt(BUTTON, isr, FALLING);
}

void loop() {
    static int last = 0;
    if (interruptCount != last) {
        last = interruptCount;
        Serial.printf("Pressed %d times\n", interruptCount);
    }
}
```

带参数版本：`attachInterruptArg(pin, voidFuncPtrArg handler, void *arg, int mode)`。
分离中断：`detachInterrupt(pin)`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 引脚无响应 | 使用了输入-only 或被 flash 占用的引脚 | 查阅开发板引脚图，改用可任意配置的 GPIO |
| 中断里 `Serial.print` 崩溃 | 中断上下文不可做长操作 | 中断只置 `volatile` 标志，在 `loop()` 中输出 |
| 按键抖动多触发 | 机械抖动 | 软件/硬件去抖，或用 `ONLOW_WE` 等电平敏感模式 |
| 按键读电平浮动 | `INPUT` 未加上下拉 | 用 `INPUT_PULLUP` / `INPUT_PULLDOWN` |

## 参考

- `libraries/ESP32/examples/GPIO/GPIOInterrupt/GPIOInterrupt.ino`
- 仓库文档 `docs/en/api/gpio.rst`
