# FreeRTOS 多任务

> **适用摘要**: 在 Arduino 环境中用原生 FreeRTOS API（`xTaskCreate`/`xTaskCreatePinnedToCore`、队列、信号量、互斥）实现并发任务，避免阻塞 `loop()`。

> Evidence: `repos/arduino-esp32/resources/`, source/examples in `repos/arduino-esp32/`, and this recipe path `repos/arduino-esp32/recipes/freertos_tasks.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "多任务 / 并发"
- "后台任务 / 双核"
- "任务间通信 / 队列 / 信号量"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ESP32/examples/FreeRTOS/BasicMultiThreading/`、`Queue/`、`Semaphore/`、`Mutex/` |
| 头文件 | `<freertos/FreeRTOS.h>` `<freertos/task.h>` `<freertos/queue.h>` `<freertos/semphr.h>` |
| 核数 | ESP32/S3 双核可同核/异核；其余单核（`CONFIG_FREERTOS_UNICORE`） |

## 分步说明

### 创建任务

```cpp
#include <freertos/FreeRTOS.h>
#include <freertos/task.h>

void taskLed(void *arg) {
    for (;;) {
        digitalWrite(2, !digitalRead(2));
        vTaskDelay(pdMS_TO_TICKS(500));     // 非阻塞延时
    }
}

void setup() {
    pinMode(2, OUTPUT);
    xTaskCreatePinnedToCore(taskLed, "led", 2048, NULL, 1, NULL, 0);  // 绑核 0
}
void loop() { vTaskDelay(pdMS_TO_TICKS(1000)); }
```

> `xTaskCreate`：栈、优先级由 FreeRTOS 管理，核由调度器选。
> `xTaskCreatePinnedToCore(fn, name, stack, arg, prio, handle, core)`：`core` 0/1（单核 SoC 必须为 0）。
> 任务函数必须为无限循环，结束后调用 `vTaskDelete(NULL)`。

### 队列（任务间传数据）

```cpp
#include <freertos/queue.h>
QueueHandle_t q;

void producer(void *a) {
    int v = 0;
    for (;;) { xQueueSend(q, &v, portMAX_DELAY); v++; vTaskDelay(pdMS_TO_TICKS(200)); }
}
void consumer(void *a) {
    int v;
    for (;;) { if (xQueueReceive(q, &v, portMAX_DELAY)) Serial.println(v); }
}
void setup() {
    Serial.begin(115200);
    q = xQueueCreate(8, sizeof(int));
    xTaskCreate(producer, "prod", 2048, NULL, 1, NULL);
    xTaskCreate(consumer, "cons", 2048, NULL, 1, NULL);
}
void loop() {}
```

### 互斥（保护共享资源）

```cpp
SemaphoreHandle_t mux;
void safePrint(const char *s) {
    xSemaphoreTake(mux, portMAX_DELAY);
    Serial.print(s);
    xSemaphoreGive(mux);
}
void setup() {
    Serial.begin(115200);
    mux = xSemaphoreCreateMutex();
    // ...
}
```

> 也可用二值/计数信号量做同步（`xSemaphoreCreateBinary` / `xSemaphoreCreateCounting`）。

### 在任务里调 Arduino API

`Serial`、`WiFi`、`Preferences`、`Wire` 等可在任意任务中调用；但跨任务共享状态需加锁，事件回调（`WiFi.onEvent` 等）在独立线程。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 任务栈溢出崩溃 | 栈太小 | 增大 `stack`（如 4096） |
| 看门狗复位 | 任务 `while(1)` 无 `vTaskDelay` | 让出 CPU：`vTaskDelay`/`vTaskDelayUntil` |
| 双核数据竞争 | 多核任务共享变量 | 用互斥/原子；IDF 默认 `CONFIG_FREERTOS_TASK_CREATE_ALLOW_EXT_MEM` 谨慎 |
| `xTaskCreatePinnedToCore` 在单核 SoC 失败 | `core` 非 0 | 单核 SoC 用 `xTaskCreate` 或 `core=0` |

## 参考

- `libraries/ESP32/examples/FreeRTOS/BasicMultiThreading/`
- `libraries/ESP32/examples/FreeRTOS/Queue/`
- `libraries/ESP32/examples/FreeRTOS/Semaphore/`
- `libraries/ESP32/examples/FreeRTOS/Mutex/`
