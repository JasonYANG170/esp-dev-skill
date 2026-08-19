# Virtqueue 单向队列（链路层）

> **适用摘要**: 直接使用 ESP-AMP Virtqueue（Packed Virtqueue，单生产者-单消费者无锁环形缓冲）实现单向核间数据传输。是 RPMsg 的底层基础。适合需要精细控制缓冲生命周期、或不需要 RPMsg 端点复用的场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/virtqueue.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "virtqueue"
- "单向核间数据流"
- "lock-free queue"
- "subcore 发送数据到 maincore"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/virtqueue/` |
| 角色 | 一侧 master core（生产者），另一侧 remote core（消费者）；maincore/subcore 都可担任任一角色 |

## 分步说明

### 1. maincore 初始化（作为 remote core 接收）

```c
#include "esp_amp.h"

esp_amp_queue_t queue;

/* maincore 作为 remote core（消费者）：is_master=false，cb_func 为接收回调 */
/* queue_len 必须为 2 的幂；queue_item_size 为单条最大字节数 */
assert(esp_amp_queue_main_init(&queue, 8 /*len*/, 128 /*item_size*/,
        recv_cb /*callback*/, NULL /*priv*/, false /*is_master*/,
        SYS_INFO_RESERVED_ID_VQUEUE /*sysinfo_id*/) == ESP_OK);

/* 若用中断接收：使能 software interrupt handler（必须 remote core 调用） */
assert(esp_amp_queue_intr_enable(&queue, SW_INTR_RESERVED_ID_RPMSG) == ESP_OK);
```

### 2. subcore 初始化（作为 master core 发送）

```c
#include "esp_amp.h"

esp_amp_queue_t queue;

int notify_func(void *arg)
{
    (void)arg;
    /* master core 发送成功后通知对端：触发对端的 sw_intr_id */
    esp_amp_sw_intr_trigger(SW_INTR_RESERVED_ID_RPMSG);  /* 必须与 remote 的 intr_enable id 一致 */
    return 0;
}

int main(void)
{
    assert(esp_amp_init() == 0);
    /* subcore 作为 master core（生产者）：is_master=true，cb_func 为 notify 函数 */
    assert(esp_amp_queue_sub_init(&queue, notify_func, NULL, true /*is_master*/,
            SYS_INFO_RESERVED_ID_VQUEUE) == ESP_OK);
    /* ... */
}
```

### 3. master core：申请缓冲 → 填数据 → 发送

```c
/* master core only */
void *buf;
if (esp_amp_queue_alloc_try(&queue, &buf, 64) == ESP_OK) {
    memcpy(buf, payload, 64);
    /* send 成功后自动调用 notify_func 触发对端中断；此后 buf 不可再访问 */
    if (esp_amp_queue_send_try(&queue, buf, 64) != ESP_OK) {
        /* 发送失败（size 过大等）；注意 buf 状态 */
    }
}
```

### 4. remote core：接收 → 处理 → 释放

```c
/* remote core only（可在 callback 或主动 poll 中调用） */
void *buf; uint16_t size;
if (esp_amp_queue_recv_try(&queue, &buf, &size) == ESP_OK) {
    process(buf, size);
    esp_amp_queue_free_try(&queue, buf);   /* 释放后 buf 不可再访问 */
}
```

### 5. callback vs notify 语义

初始化时 `cb_func` 同时承担两个角色，由 `is_master` 决定：
- `is_master=true`：`cb_func` = **notify 函数**，`esp_amp_queue_send_try` 成功后自动调用（用于触发对端中断）。
- `is_master=false`：`cb_func` = **callback 函数**，收到新数据时自动调用（可在 ISR 上下文，取决于是否 `intr_enable`）。

### 6. 中断 vs 轮询

- remote core 调用 `esp_amp_queue_intr_enable(&queue, sw_intr_id)`：收到数据时在 ISR 调 callback。
- 不调用 `intr_enable`：需在主循环主动 `esp_amp_queue_recv_try` 轮询。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `queue_len` 报错 ESP_ERR_INVALID_ARG | 非 2 的幂 | queue_len 必须为 2 的幂（内部会向上取整，但仍建议直接传 2 的幂） |
| 缓冲永久不可用 | alloc 不 send、或 recv 不 free | master: alloc→send 成对；remote: recv→free 成对 |
| 用非 alloc 的指针调 send/free | 未定义行为 | send 的 buffer 必须来自 alloc_try；free 的必须来自 recv_try |
| master 调 recv / remote 调 alloc | 角色错 | master: alloc/send；remote: recv/free |
| notify 中断不触发对端 | sw_intr_id 两端不一致 | master notify_func 触发的 id 必须等于 remote `intr_enable` 的 id |
| 并发访问破坏一致性 | 非单生产者-单消费者 | 每侧只有一个上下文访问队列；需要多生产者请用 RPMsg |

## 参考

- `examples/virtqueue/` — subcore(master)→maincore(remote) 完整示例
- `espressif-repos/esp-amp/docs/queue.md`
- `espressif-repos/esp-amp/components/esp_amp/include/esp_amp_queue.h`
