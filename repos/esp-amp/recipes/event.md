# Event 跨核同步

> **适用摘要**: 使用 ESP-AMP Event（基于 32-bit 原子整数的位图同步机制）实现轻量跨核事件通知。创建事件、绑定 FreeRTOS EventGroup、通知、等待/轮询。适合需要双向或多任务广播同步的场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/event.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "跨核事件通知"
- "maincore 等待 subcore 就绪"
- "Event 同步"
- "EVENT_SUBCORE_READY 握手"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/event/` |
| Kconfig | `CONFIG_ESP_AMP_EVENT_TABLE_LEN`（默认 8，可绑定 OS event handle 数） |

## 分步说明

### 1. 定义事件位与 SysInfo ID（common/event.h、common/sys_info.h）

```c
/* common/event.h */
#pragma once
#define EVENT_SUBCORE_READY   (1 << 0)   /* subcore 通知 maincore 就绪（保留事件内置） */

/* common/sys_info.h */
#pragma once
#define SYS_INFO_ID_MAINCORE_EVENT  0x0002   /* maincore → subcore 的事件 */
#define SYS_INFO_ID_SUBCORE_EVENT   0x0003   /* subcore → maincore 的事件 */
#define EVENT_MAINCORE_EVENT        (1 << 0)
#define EVENT_SUBCORE_EVENT_1       (1 << 1)
#define EVENT_SUBCORE_EVENT_2       (1 << 2)
```

> 注：`EVENT_SUBCORE_READY` 走 ESP-AMP 内置保留事件（`SYS_INFO_RESERVED_ID_EVENT_SUB`），用便捷宏 `esp_amp_event_notify/wait` 即可，无需 create。

### 2. maincore：创建事件 + 绑定 EventGroup（FreeRTOS）

```c
#include "esp_amp.h"
#include "freertos/event_groups.h"

EventGroupHandle_t sub_core_event_group;

void app_main(void)
{
    assert(esp_amp_init() == 0);

    /* 创建两个自定义事件（仅 maincore 能 create） */
    assert(esp_amp_event_create(SYS_INFO_ID_MAINCORE_EVENT) == 0);
    assert(esp_amp_event_create(SYS_INFO_ID_SUBCORE_EVENT) == 0);

    /* FreeRTOS 端必须绑定 EventGroup，否则 wait 永不唤醒 */
    sub_core_event_group = xEventGroupCreate();
    assert(esp_amp_event_bind_handle(SYS_INFO_ID_SUBCORE_EVENT, sub_core_event_group) == 0);

    /* 加载启动 subcore 后握手（用便捷宏走保留事件） */
    const esp_partition_t *p = esp_partition_find_first(ESP_PARTITION_TYPE_DATA, 0x40, NULL);
    ESP_ERROR_CHECK(esp_amp_load_sub_from_partition(p));
    ESP_ERROR_CHECK(esp_amp_start_subcore());
    assert((esp_amp_event_wait(EVENT_SUBCORE_READY, true, true, 10000)
            & EVENT_SUBCORE_READY) == EVENT_SUBCORE_READY);

    /* 之后既可用 xEventGroupWaitBits 也可用 esp_amp_event_wait_by_id 等待 */
}
```

### 3. maincore：通知 subcore

```c
/* bit_mask 仅用低 24 位（高 8 位 FreeRTOS 保留） */
esp_amp_event_notify_by_id(SYS_INFO_ID_MAINCORE_EVENT, EVENT_MAINCORE_EVENT);
```

### 4. subcore：通知 maincore + 轮询（bare-metal）

```c
#include "esp_amp.h"
#include "esp_amp_platform.h"

int main(void)
{
    assert(esp_amp_init() == 0);

    /* 通知就绪（便捷宏走 SYS_INFO_RESERVED_ID_EVENT_SUB） */
    esp_amp_event_notify(EVENT_SUBCORE_READY);

    for (;;) {
        /* 轮询 maincore 发来的事件（bare-metal 用 poll，clear_on_exit=true） */
        uint32_t ret = esp_amp_event_poll_by_id(
            SYS_INFO_ID_MAINCORE_EVENT, EVENT_MAINCORE_EVENT, true, true);
        if ((ret & EVENT_MAINCORE_EVENT) == EVENT_MAINCORE_EVENT) {
            printf("SUB: recv EVENT_MAINCORE_EVENT\r\n");
        }

        /* 通知 maincore（自定义事件） */
        esp_amp_event_notify_by_id(SYS_INFO_ID_SUBCORE_EVENT, EVENT_SUBCORE_EVENT_1);

        esp_amp_platform_delay_us(1000000);
    }
}
```

### 5. 关键 API 语义

| API | 作用 | 注意 |
|---|---|---|
| `esp_amp_event_create(id)` | 创建事件（仅 maincore） | subcore 启动前调用；不可销毁 |
| `esp_amp_event_bind_handle(id, eg)` | 绑定 FreeRTOS EventGroup（仅 FreeRTOS） | 不绑定则 wait 永不唤醒 |
| `esp_amp_event_notify_by_id(id, mask)` | 通知对端（位 OR） | mask 仅低 24 位 |
| `esp_amp_event_wait_by_id(id, mask, clear, all, timeout)` | 等待（FreeRTOS 阻塞 / bare-metal 忙等） | clear_on_exit=true 避免丢事件 |
| `esp_amp_event_poll_by_id(...)` | 轮询（=wait timeout=0，仅 bare-metal） | light sleep 下不推荐 |

### 6. 单向原则

单个 ESP-AMP event 只能单向通知。双向同步须建两个 event。禁止在同一核上对同一 event 同时 `notify()` 与 `clear()`——会丢失事件顺序信息。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| FreeRTOS 任务 wait 永不唤醒 | 未绑定 EventGroup | `esp_amp_event_bind_handle(id, eg)` 必须调用 |
| 事件丢失 | wait 与 clear 非原子 | wait/poll 设 `clear_on_exit=true` |
| 双向用同一 event 出错 | event 是单向的 | 双向同步创建两个 event |
| 高 8 位被截断 | FreeRTOS 保留高 8 位 | bit_mask 仅用低 24 位 |
| light sleep 不生效 | subcore 用 `esp_amp_event_poll()` 忙等阻止睡眠 | light sleep 下改用软件中断做 HP→LP 通知 |
| 同核 notify+clear 顺序混乱 | 设计禁止 | set 与 clear 必须分属不同核 |

## 参考

- `examples/event/maincore/main/app_main.c` — create/bind/notify/wait 完整流程
- `examples/event/subcore/main/main.c` — bare-metal poll/notify
- `examples/rpmsg_send_recv/common/event.h` — `EVENT_SUBCORE_READY` 定义
- `espressif-repos/esp-amp/docs/event.md`
- `espressif-repos/esp-amp/components/esp_amp/include/esp_amp_event.h`
