# FreeRTOS 任务与队列

> **适用摘要**: 创建 FreeRTOS 任务、用队列在任务/ISR 间传递数据，含任务通知与延时用法。

## 触发意图

- "创建任务"
- "FreeRTOS 多任务"
- "任务间通信"
- "队列发送接收"
- "vTaskDelay 延时"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `freertos/FreeRTOS.h`、`freertos/task.h`、`freertos/queue.h` |
| 参考 | `examples/get-started/hello_world`（`vTaskDelay`） |

## 分步说明

### 创建任务

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

static const char *TAG = "task";

static void sensor_task(void *arg)
{
    while (1) {
        ESP_LOGI(TAG, "tick");
        vTaskDelay(pdMS_TO_TICKS(1000));   // 1s
    }
}

void app_main(void)
{
    /* 栈深单位是字（word），不是字节；esp32 上 1 word = 4 byte */
    xTaskCreate(sensor_task, "sensor", 4096, NULL, 5, NULL);
}
```

> `app_main` 自身运行在 `main` 任务中，优先级 1；它返回后该任务会被删除，故长驻逻辑须自建任务或 `while(1)`。

### 队列（任务间通信）

```c
#include "freertos/queue.h"

typedef struct {
    int id;
    int value;
} msg_t;

static QueueHandle_t s_queue;

static void producer(void *arg)
{
    msg_t m = {0};
    while (1) {
        m.id++;
        m.value = m.id * 10;
        xQueueSend(s_queue, &m, portMAX_DELAY);   // 满则阻塞等待
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

static void consumer(void *arg)
{
    msg_t m;
    while (1) {
        if (xQueueReceive(s_queue, &m, portMAX_DELAY) == pdTRUE) {
            ESP_LOGI("consumer", "id=%d value=%d", m.id, m.value);
        }
    }
}

void app_main(void)
{
    s_queue = xQueueCreate(10, sizeof(msg_t));
    xTaskCreate(producer, "producer", 3072, NULL, 5, NULL);
    xTaskCreate(consumer, "consumer", 3072, NULL, 4, NULL);
}
```

### ISR 内向队列发数据

```c
static void IRAM_ATTR gpio_isr(void *arg)
{
    BaseType_t hpw = pdFALSE;
    msg_t m = { .id = 99, .value = -1 };
    xQueueSendFromISR(s_queue, &m, &hpw);
    portYIELD_FROM_ISR(hpw);   /* 必要时触发上下文切换 */
}
```

### 任务通知（轻量同步，无需创建队列）

```c
/* 等待方 */
ulTaskNotifyTake(pdTRUE, portMAX_DELAY);   /* 清零计数，阻塞 */

/* 通知方（任务或 ISR） */
xTaskNotifyGive(xHandle);            /* 任务内 */
vTaskNotifyGiveFromISR(xHandle, &hpw); /* ISR 内 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 任务创建后立即"消失" | 任务函数 return 退出 | 任务内必须 `while(1)`，或显式 `vTaskDelete(NULL)` |
| 栈溢出 / crash | 栈深过小 | 增大 `xTaskCreate` 的 `usStackDepth`（单位 word） |
| ISR 调用 `xQueueSend` 崩溃 | 用了非 ISR 版 | ISR 内用 `xQueueSendFromISR` + `portYIELD_FROM_ISR` |
| `vTaskDelay(1000)` 延时不对 | 误把 ms 当 ticks | 用 `pdMS_TO_TICKS(1000)` 换算 |
| 优先级反转/饿死 | 高优先级任务不让出 | 高优先级任务要有阻塞点（`vTaskDelay`/`xQueueReceive(portMAX_DELAY)`） |

## 参考

- `examples/get-started/hello_world/main/hello_world_main.c` — `vTaskDelay` 用法
- ESP-IDF `components/freertos/FreeRTOS-Kernel/include/freertos/task.h`、`queue.h`
