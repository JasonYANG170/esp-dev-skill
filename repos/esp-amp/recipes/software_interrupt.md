# 软件中断（核间中断）

> **适用摘要**: 使用 ESP-AMP 软件中断（基于 PMU 或 INTMTX 的核间中断）实现异步核间通知。注册处理函数、触发对端核、多 handler 复用同一中断源。这是 Queue/RPMsg 通知机制的基础。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/software_interrupt.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "核间中断"
- "软件中断通知"
- "IPC trigger"
- "register sw interrupt handler"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/software_interrupt/` |
| Kconfig | `CONFIG_ESP_AMP_SW_INTR_HANDLER_TABLE_LEN`（默认 8，最多可注册 handler 数） |

## 分步说明

### 1. 中断源 ID（共 32 个，高段保留）

```c
/* 来自 esp_amp_sw_intr.h：SW_INTR_ID_0 ~ SW_INTR_ID_15 用户可用；
   SW_INTR_RESERVED_ID_16 ~ SW_INTR_RESERVED_ID_EVENT 为 ESP-AMP 内部保留
   （RPMSG/EVENT/SYS_SVC/PANIC 等） */
```

### 2. maincore 注册/注销处理函数

```c
#include "esp_amp.h"

static const DRAM_ATTR char TAG[] = "sw";

/* maincore handler 必须 IRAM_ATTR；返回值表示是否需要上下文切换 */
static IRAM_ATTR int sw_intr_id0_handler_1(void *arg)
{
    (void)arg;
    ESP_DRAM_LOGI(TAG, "sw_intr_id0_handler_1() called");
    BaseType_t need_yield = pdFALSE;
    /* 例：xEventGroupSetBitsFromISR(eg, BIT, &need_yield); */
    return (need_yield == pdTRUE);   // 1=唤醒高优先级任务，公共处理函数据此 portYIELD_from_ISR
}

/* 注册：同一中断源可注册多个 handler */
assert(esp_amp_sw_intr_add_handler(SW_INTR_ID_0, sw_intr_id0_handler_1, NULL) == 0);
assert(esp_amp_sw_intr_add_handler(SW_INTR_ID_0, sw_intr_id0_handler_2, NULL) == 0);

/* 注销 */
esp_amp_sw_intr_delete_handler(SW_INTR_ID_0, sw_intr_id0_handler_1);
```

### 3. subcore 注册处理函数（bare-metal）

```c
#include "esp_amp.h"

/* subcore handler 无需 IRAM_ATTR（整段固件已在内部 RAM）；返回值被忽略 */
static int sub_handler(void *arg)
{
    (void)arg;
    printf("SUB: sw intr received\r\n");
    return 0;
}

int main(void)
{
    assert(esp_amp_init() == 0);
    assert(esp_amp_sw_intr_add_handler(SW_INTR_ID_0, sub_handler, NULL) == 0);
    esp_amp_event_notify(EVENT_SUBCORE_READY);

    for (;;) {
        /* bare-metal 主循环 */
    }
}
```

### 4. 触发对端核的中断

```c
/* 在 maincore 触发 subcore 的 ID_0 */
esp_amp_sw_intr_trigger(SW_INTR_ID_0);

/* 在 subcore 触发 maincore 的 ID_1 */
esp_amp_sw_intr_trigger(SW_INTR_ID_1);
```

### 5. 工作机制

- 单条中断线由所有软件中断源共享；公共处理函数遍历 handler 表，逐个调用匹配 pending 源的 handler，全部调用后清除 pending 位。
- handler 表越大，ISR 耗时越长；用 `CONFIG_ESP_AMP_SW_INTR_HANDLER_TABLE_LEN` 控制。
- 一个 handler 可注册到多个源；一个源可挂多个 handler。

### 6. 调试

```c
esp_amp_sw_intr_handler_dump();   // 打印 handler 表
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| maincore handler 未执行 / cache miss | 缺 `IRAM_ATTR` | maincore 中断函数加 `IRAM_ATTR`，TAG 用 `DRAM_ATTR`，日志用 `ESP_DRAM_LOGx` |
| `add_handler` 返回非 0 | handler 表满 | 增大 `CONFIG_ESP_AMP_SW_INTR_HANDLER_TABLE_LEN` |
| 误用保留 ID | 使用 `SW_INTR_RESERVED_ID_*` | 仅用 `SW_INTR_ID_0`~`SW_INTR_ID_15` |
| subcore handler 返回值不生效 | subcore 返回值被设计忽略 | maincore 才依赖返回值做上下文切换 |
| light sleep 时 LP subcore 中断响应慢 | ISR 过长 | 尽量缩短 LP subcore ISR 以省电 |

## 参考

- `examples/software_interrupt/maincore/main/app_main.c` — 注册多个 handler、触发
- `examples/software_interrupt/subcore/main/main.c`
- `espressif-repos/esp-amp/docs/software_interrupt.md`
- `espressif-repos/esp-amp/components/esp_amp/include/esp_amp_sw_intr.h`
