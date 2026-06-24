# 硬件定时器 (Hardware Timer)

> **适用摘要**: 使用 `timerBegin` / `timerAlarm` 配置 64 位硬件定时器并产生周期中断。ESP32/S2/S3 各 4 个，C3/C6/H2 各 2 个。

## 触发意图

- "周期定时中断"
- "精确计时"
- "硬件定时器"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ESP32/examples/Timer/RepeatTimer/RepeatTimer.ino`、`WatchdogTimer/WatchdogTimer.ino` |
| 定时器数 | ESP32/S2/S3=4，C3/C6/H2=2（C3 为 54 位计数） |

## 分步说明

### 周期中断（1 Hz 闪烁）

```cpp
#define LED 2
hw_timer_t *timer = NULL;
volatile bool toggle = false;

void IRAM_ATTR onTimer() {
    toggle = !toggle;
    digitalWrite(LED, toggle);
}

void setup() {
    pinMode(LED, OUTPUT);
    timer = timerBegin(1000000);             // 计数频率 1 MHz → 每 tick 1 us
    timerAttachInterrupt(timer, &onTimer);
    timerAlarm(timer, 1000000, true, 0);     // 1,000,000 us = 1s，autoreload，0=无限
}
void loop() {}
```

> 3.x 中 `timerBegin(frequency)` 直接以 Hz 指定计数频率，自动启动。`timerAlarm(timer, alarm_value, autoreload, reload_count)`：`autoreload=true` 周期触发，`reload_count=0` 表示无限。

带参数中断：`timerAttachInterruptArg(timer, void(*fn)(void*), arg)`。

### 计时读取

```cpp
uint64_t us    = timerReadMicros(timer);
uint64_t ms    = timerReadMillis(timer);
double    sec  = timerReadSeconds(timer);
uint16_t  freq = timerGetFrequency(timer);
```

控制：`timerStart(timer)` / `timerStop(timer)` / `timerRestart(timer)` / `timerWrite(timer, val)` / `timerEnd(timer)`。
去中断：`timerDetachInterrupt(timer)`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 中断里崩溃/重入 | 中断里做长操作 | 只置 `volatile` 标志，处理放 `loop()` |
| 周期不准 | 频率与 alarm 单位混 | `timerBegin(f)` 设计数频率，`timerAlarm` 用对应计数单位 |
| `timerBegin` 返回 NULL | 无可用定时器 | 检查已用数量，是否超过 SoC 上限 |

## 参考

- `libraries/ESP32/examples/Timer/RepeatTimer/RepeatTimer.ino`
- `libraries/ESP32/examples/Timer/WatchdogTimer/WatchdogTimer.ino`
- 仓库文档 `docs/en/api/timer.rst`
