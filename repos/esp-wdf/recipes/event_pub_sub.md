# 应用间事件发布/订阅

> **适用摘要**：用 `api_publish_event` / `api_subscribe_event` 在 WASM 应用之间通过事件 URL 异步通信，负载使用 `attr_container_t` 结构化容器。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wdf/resources/`, source/examples in `repos/esp-wdf/`, and this recipe path `repos/esp-wdf/recipes/event_pub_sub.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "应用间通信"
- "发布订阅事件"
- "event publisher subscriber"
- "WASM app 互相通知"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_WAMR_APP_FRAMEWORK=y` |
| 参考 | `examples/simple/event_publisher/`、`examples/simple/event_subscriber/` |

## 分步说明

### 1. 发布者（取自 examples/simple/event_publisher）

```c
#include "wasm_app.h"
#include "wa-inc/request.h"
#include "wa-inc/timer_wasm_app.h"

int num = 0;

void
publish_overheat_event()
{
    attr_container_t *event;

    event = attr_container_create("event");
    /* 注意：首参是 &event（二级指针），容器可能被重建 */
    attr_container_set_string(&event, "warning", "temperature is over high");

    /* 发布事件：url、格式 FMT_ATTR_CONTAINER、负载、负载长度 */
    api_publish_event("alert/overheat", FMT_ATTR_CONTAINER, event,
                      attr_container_get_serialize_length(event));

    attr_container_destroy(event);
}

/* 定时器回调中周期发布 */
void
timer1_update(user_timer_t timer)
{
    publish_overheat_event();
}

void
start_timer()
{
    user_timer_t timer;
    timer = api_timer_create(1000, true, false, timer1_update);
    api_timer_restart(timer, 1000);
}

void
on_init()
{
    start_timer();
}

void
on_destroy() {}
```

### 2. 订阅者（取自 examples/simple/event_subscriber）

```c
#include "wasm_app.h"
#include "wa-inc/request.h"

void
over_heat_event_handler(request_t *request)
{
    printf("### user over heat event handler called\n");

    /* 事件 payload 通过 request 传入；校验格式后转为 attr_container 打印 */
    if (request->payload != NULL && request->fmt == FMT_ATTR_CONTAINER)
        attr_container_dump((attr_container_t *)request->payload);
}

void
on_init()
{
    /* 订阅事件：匹配 url "alert/overheat"，收到时回调 handler */
    api_subscribe_event("alert/overheat", over_heat_event_handler);
}

void
on_destroy() {}
```

### 3. 关键点

- 事件 URL（如 `"alert/overheat"`）在发布端与订阅端必须完全一致。
- 负载格式统一用 `FMT_ATTR_CONTAINER`（=99）；订阅端在回调里通过 `request->fmt` 校验后用 `(attr_container_t *)request->payload` 取值。
- 发布端用 `attr_container_get_serialize_length(event)` 得到负载字节长度传入 `api_publish_event`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 订阅者收不到事件 | 发布/订阅 URL 不一致 | 两端 URL 完全相同（含大小写） |
| 负载乱码/越界 | 直接 `request->payload` 当原始缓冲读 | 先判 `request->fmt == FMT_ATTR_CONTAINER`，再按容器 API 读 |
| `set_string` 编译报错 | 首参传了一级指针 | 写 `attr_container_set_string(&event, ...)`（二级指针） |
| 发布端内存泄漏 | 用完未销毁 | 发布后 `attr_container_destroy(event)` |

## 参考

- `examples/simple/event_publisher/main/event_publisher.c`
- `examples/simple/event_subscriber/main/event_subscriber.c`
- `components/wamr/app-framework/include/wa-inc/request.h`
- `components/wamr/app-framework/include/bi-inc/attr_container.h`
- `resources/api_reference.md` —— 第 1.3、1.5、二节
