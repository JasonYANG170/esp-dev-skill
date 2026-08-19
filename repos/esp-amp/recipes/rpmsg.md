# RPMsg 双向通信（传输层）

> **适用摘要**: 使用 ESP-AMP RPMsg（Remote Processor Messaging）实现双向端到端通信，基于一对 Virtqueue（TX/RX），支持在单个设备上创建多个端点复用底层队列。支持零拷贝发送。这是最常用的核间数据交换方式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/rpmsg.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "RPMsg"
- "双向核间通信"
- "多端点消息"
- "零拷贝发送"
- "maincore subcore 互发消息"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/rpmsg_send_recv/` |
| notify/poll | 两端参数互补：一端 `poll=false` 则对端 `notify=true` |

## 分步说明

### 1. maincore 初始化（中断模式 + 创建端点）

```c
#include "esp_amp.h"

esp_amp_rpmsg_dev_t rpmsg_dev;
esp_amp_rpmsg_ept_t rpmsg_ept[3];

/* IRAM_ATTR 回调（中断模式下在 ISR 上下文调用） */
int IRAM_ATTR ept_cb(void *msg_data, uint16_t data_len, uint16_t src_addr, void *rx_cb_data)
{
    QueueHandle_t q = (QueueHandle_t)rx_cb_data;
    BaseType_t woken = pdFALSE;
    /* 把消息指针投递到任务处理 */
    if (xQueueSendFromISR(q, &msg_data, &woken) != pdTRUE) {
        esp_amp_rpmsg_destroy(&rpmsg_dev, msg_data);   /* 队列满则销毁避免泄漏 */
    }
    portYIELD_FROM_ISR(woken);
    return 0;
}

void app_main(void)
{
    assert(esp_amp_init() == 0);

    /* notify=true, poll=false → 中断模式；poll=false 必须随后 intr_enable */
    assert(esp_amp_rpmsg_main_init(&rpmsg_dev, 32 /*queue_len*/, 128 /*item_size*/,
            true /*notify*/, false /*poll*/) == 0);

    QueueHandle_t q = xQueueCreate(32, sizeof(void*));
    assert(esp_amp_rpmsg_create_endpoint(&rpmsg_dev, 0 /*ept_addr*/, ept_cb, q, &rpmsg_ept[0]) != NULL);

    esp_amp_rpmsg_intr_enable(&rpmsg_dev);   // poll=false 必须调用

    /* 加载启动 + 握手 ... */
}
```

### 2. maincore 轮询模式（可选，poll=true 时）

```c
/* notify=true, poll=true → 轮询模式；回调在任务上下文调用，无需 intr_enable */
assert(esp_amp_rpmsg_main_init(&rpmsg_dev, 32, 128, true, true) == 0);
/* 主循环中主动 poll */
while (1) {
    while (esp_amp_rpmsg_poll(&rpmsg_dev) == 0);   // 0=还有消息，继续
    vTaskDelay(pdMS_TO_TICKS(1000));
}
```

### 3. subcore 初始化（与对端参数互补）

```c
#include "esp_amp.h"

esp_amp_rpmsg_dev_t rpmsg_dev;
esp_amp_rpmsg_ept_t rpmsg_ept[3];

int ept0_cb(void *msg_data, uint16_t data_len, uint16_t src_addr, void *rx_cb_data)
{
    printf("SUB: [EPT%d]: %s\r\n", (int)src_addr, (char*)msg_data);
    esp_amp_rpmsg_destroy(&rpmsg_dev, msg_data);   // 接收端必须 destroy
    return 0;
}

int main(void)
{
    assert(esp_amp_init() == 0);

#ifdef CONFIG_EXAMPLE_RPMSG_ENABLE_INTERRUPT_ON_SUBCORE
    assert(esp_amp_rpmsg_sub_init(&rpmsg_dev, true, false) == 0);  // 中断
    assert(esp_amp_rpmsg_intr_enable(&rpmsg_dev) == 0);
#else
    assert(esp_amp_rpmsg_sub_init(&rpmsg_dev, true, true) == 0);   // 轮询
#endif

    esp_amp_rpmsg_create_endpoint(&rpmsg_dev, 0, ept0_cb, NULL, &rpmsg_ept[0]);

    esp_amp_event_notify(EVENT_SUBCORE_READY);

    for (;;) {
#if !CONFIG_EXAMPLE_RPMSG_ENABLE_INTERRUPT_ON_SUBCORE
        while (esp_amp_rpmsg_poll(&rpmsg_dev) == 0);   // 轮询模式才需要
#endif
        /* 发送 ... */
    }
}
```

### 4. 零拷贝发送（推荐）

```c
/* 方式 A：create_message + send_nocopy（零拷贝，必须成对） */
void *msg = esp_amp_rpmsg_create_message(&rpmsg_dev, 48, ESP_AMP_RPMSG_DATA_DEFAULT);
if (msg != NULL) {
    snprintf(msg, 48, "Normal Msg: %d", count++);
    assert(esp_amp_rpmsg_send_nocopy(&rpmsg_dev, &rpmsg_ept[0], 0 /*dst_addr*/, msg, 48) == 0);
}
```

### 5. 拷贝发送（单独使用）

```c
/* 方式 B：send（内部拷贝，不可与 create_message 混用） */
char buf[32] = "hello";
esp_amp_rpmsg_send(&rpmsg_dev, &rpmsg_ept[0], 0 /*dst_addr*/, buf, strlen(buf));
```

### 6. 多端点：一个 rpmsg_dev 上创建多个端点复用底层队列

```c
/* maincore 创建 3 个端点，地址 0/1/2；subcore 侧也创建对应端点 */
esp_amp_rpmsg_create_endpoint(&rpmsg_dev, 0, ept_cb, q0, &rpmsg_ept[0]);
esp_amp_rpmsg_create_endpoint(&rpmsg_dev, 1, ept_cb, q1, &rpmsg_ept[1]);
esp_amp_rpmsg_create_endpoint(&rpmsg_dev, 2, ept_cb, q2, &rpmsg_ept[2]);
/* 发送时指定 dst_addr 路由到对端对应端点 */
```

### 7. 多对 RPMsg 设备（可选，用 _by_id 指定不同 sysinfo_id）

```c
esp_amp_queue_t vq[2];
esp_amp_rpmsg_main_init_by_id(&dev2, vq, 8, 128, true, true, MY_SYSINFO_ID);
esp_amp_rpmsg_sub_init_by_id(&dev2, vq, true, true, MY_SYSINFO_ID);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `send()` 后缓冲泄漏 | send 与 create_message 混用 | send 单独用；零拷贝必须 create_message→send_nocopy 成对 |
| 接收端缓冲耗尽（create_message 返回 NULL） | 接收端未 destroy | 接收端用完消息必须 `esp_amp_rpmsg_destroy()` |
| 消息卡住不处理 | 两端都 poll=false 或都不 notify | 一端 poll=false 则对端 notify=true；poll=false 侧必须 `intr_enable` |
| ISR 内回调崩溃 | 中断模式回调在 ISR 执行 | 仅调用 ISR-safe API；用 `esp_amp_env_in_isr()` 判断上下文 |
| 数据超长被拒 | data_len > queue_item_size | 初始化时设大 `queue_item_size`；或拆包发送；用 `esp_amp_rpmsg_get_max_size()` 查询 |
| 回调中调 create_endpoint | 禁止在 ISR 调用 | create/delete/rebind/search endpoint 严禁中断上下文 |

## 参考

- `examples/rpmsg_send_recv/maincore/main/app_main.c` — 中断/轮询双模式、多端点
- `examples/rpmsg_send_recv/subcore/main/main.c` — subcore RPC 风格端点响应
- `espressif-repos/esp-amp/docs/rpmsg.md`
- `espressif-repos/esp-amp/components/esp_amp/include/esp_amp_rpmsg.h`
