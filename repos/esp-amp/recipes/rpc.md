# RPC 框架（应用层）

> **适用摘要**: 使用 ESP-AMP RPC（基于 RPMsg 的简单远程过程调用框架）在一核定义 RPC 服务、另一核调用。支持阻塞/非阻塞、有/无响应命令。适合需要"调用对端函数并取回结果"的场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-amp/resources/`, source/examples in `repos/esp-amp/`, and this recipe path `repos/esp-amp/recipes/rpc.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "RPC"
- "跨核函数调用"
- "远程过程调用"
- "maincore 调用 subcore 的函数"
- "RPC 超时与 abort"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/rpc/maincore_client_subcore_server/`、`examples/rpc/subcore_client_maincore_server/` |
| 基础 | 已完成 RPMsg 初始化（RPC 内部使用 RPMsg 端点） |
| client/server ID | 两侧 `common/rpc_service.h` 中统一定义 |

## 分步说明

### 1. 定义命令 ID 与序列化结构（common/rpc_service.h）

```c
#pragma once
#include <stdint.h>

#define RPC_DEMO_CLIENT  0x0001
#define RPC_DEMO_SERVER  0x1001
#define RPC_SRV_NUM      2

#define RPC_CMD_ID_ADD     1
#define RPC_CMD_ID_PRINTF  2

typedef struct { int a; int b; } add_params_in_t;
typedef struct { int ret; } add_params_out_t;
typedef struct { int len; char buf[100]; } printf_params_in_t;

int rpc_cmd_handler_add(esp_amp_rpc_cmd_t *cmd);      // 需 include esp_amp_rpc.h
void rpc_cmd_handler_printf(esp_amp_rpc_cmd_t *cmd);
```

### 2. server 端（subcore，bare-metal 轮询）

```c
#include "esp_amp.h"
#include "esp_amp_platform.h"
#include "rpc_service.h"

static esp_amp_rpmsg_dev_t rpmsg_dev;
static esp_amp_rpc_server_stg_t rpc_server_stg;
static uint8_t req_buf[128];
static uint8_t resp_buf[128];
static uint8_t srv_tbl_stg[sizeof(esp_amp_rpc_service_t) * RPC_SRV_NUM];

/* 命令处理函数原型：void (esp_amp_rpc_cmd_t *cmd) */
void rpc_cmd_handler_add(esp_amp_rpc_cmd_t *cmd)
{
    add_params_in_t *in = (add_params_in_t *)cmd->req_data;
    int c = in->a + in->b;
    add_params_out_t *out = (add_params_out_t *)cmd->resp_data;
    out->ret = c;
    cmd->resp_len = sizeof(add_params_out_t);
    cmd->status = ESP_AMP_RPC_STATUS_OK;
}

int main(void)
{
    assert(esp_amp_init() == 0);
    assert(esp_amp_rpmsg_sub_init(&rpmsg_dev, true /*notify*/, true /*poll*/) == 0);
    esp_amp_event_notify(EVENT_SUBCORE_READY);

    esp_amp_rpc_server_cfg_t cfg = {
        .rpmsg_dev = &rpmsg_dev,
        .server_id = RPC_DEMO_SERVER,
        .stg = &rpc_server_stg,
        .req_buf_len = sizeof(req_buf),
        .resp_buf_len = sizeof(resp_buf),
        .req_buf = req_buf,
        .resp_buf = resp_buf,
        .srv_tbl_len = RPC_SRV_NUM,
        .srv_tbl_stg = srv_tbl_stg,
    };
    esp_amp_rpc_server_t server = esp_amp_rpc_server_init(&cfg);
    assert(server != NULL);

    assert(esp_amp_rpc_server_add_service(server, RPC_CMD_ID_ADD, rpc_cmd_handler_add) == 0);

    /* bare-metal：主循环 poll RPMsg，server 内部处理命令并自动回结果 */
    for (;;) {
        while (esp_amp_rpmsg_poll(&rpmsg_dev) == 0);
        esp_amp_platform_delay_us(1000);
    }
}
```

### 3. client 端（maincore，FreeRTOS 中断模式）

```c
#include "esp_amp.h"
#include "freertos/task.h"
#include "rpc_service.h"

static esp_amp_rpmsg_dev_t rpmsg_dev;
static esp_amp_rpc_client_stg_t rpc_client_stg;

/* 响应到达时回调（在 ISR），唤醒阻塞任务 */
static void cmd_add_cb(esp_amp_rpc_client_t client, esp_amp_rpc_cmd_t *cmd, void *arg)
{
    TaskHandle_t task = (TaskHandle_t)arg;
    BaseType_t need_yield = false;
    vTaskNotifyGiveFromISR(task, &need_yield);
    portYIELD_FROM_ISR(need_yield);
}

/* 阻塞式 RPC 调用（带超时） */
static int rpc_cmd_add(esp_amp_rpc_client_t client, int a, int b, int *ret, uint32_t timeout_ms)
{
    add_params_in_t in = { .a = a, .b = b };
    add_params_out_t out;

    esp_amp_rpc_cmd_t cmd = {
        .cmd_id = RPC_CMD_ID_ADD,
        .req_len = sizeof(in),
        .resp_len = sizeof(out),        // resp_data 缓冲容量，必须用户分配
        .req_data = (uint8_t *)&in,
        .resp_data = (uint8_t *)&out,
        .cb = cmd_add_cb,
        .cb_arg = xTaskGetCurrentTaskHandle(),
    };

    int err = esp_amp_rpc_client_execute_cmd(client, &cmd);
    if (err == ESP_AMP_RPC_OK) {
        if (ulTaskNotifyTake(true, pdMS_TO_TICKS(timeout_ms)) == 0) {
            /* 超时必须 abort，防止迟到的响应改写已释放的栈上 cmd 结构 */
            esp_amp_rpc_client_abort_cmd(client, &cmd);
            err = ESP_AMP_RPC_ERR_TIMEOUT;
        } else if (cmd.status == ESP_AMP_RPC_STATUS_OK) {
            *ret = out.ret;
        } else {
            err = ESP_AMP_RPC_FAIL;
        }
    }
    return err;
}

static void client_task(void *args)
{
    esp_amp_rpc_client_cfg_t cfg = {
        .client_id = RPC_DEMO_CLIENT,
        .server_id = RPC_DEMO_SERVER,
        .rpmsg_dev = &rpmsg_dev,
        .stg = &rpc_client_stg,
    };
    esp_amp_rpc_client_t client = esp_amp_rpc_client_init(&cfg);
    assert(client != NULL);

    int ret;
    int err = rpc_cmd_add(client, 3, 4, &ret, 1000);
    printf("add result=%d err=%d\n", ret, err);
    vTaskDelete(NULL);
}

void app_main(void)
{
    assert(esp_amp_init() == 0);
    assert(esp_amp_rpmsg_main_init(&rpmsg_dev, 8, 128, false /*notify*/, false /*poll*/) == 0);
    esp_amp_rpmsg_intr_enable(&rpmsg_dev);

    /* 加载启动 + 握手 ... */

    xTaskCreate(client_task, "c", 2048, NULL, tskIDLE_PRIORITY + 1, NULL);
}
```

### 4. 非阻塞无响应命令（client）

```c
static void rpc_cmd_printf(esp_amp_rpc_client_t client, const char *fmt, ...)
{
    uint8_t buf[100] = {0};
    va_list ap; va_start(ap, fmt);
    int len = vsnprintf((char *)buf, sizeof(buf), fmt, ap);
    va_end(ap);

    printf_params_in_t in;
    memcpy(&in.len, &len, sizeof(int));
    memcpy(&in.buf, buf, len);

    esp_amp_rpc_cmd_t cmd = {
        .cmd_id = RPC_CMD_ID_PRINTF,
        .req_len = sizeof(in),
        .resp_len = 0,
        .req_data = (uint8_t *)&in,
        .resp_data = NULL,
        .cb = NULL,                 // 非阻塞
    };
    esp_amp_rpc_client_execute_cmd(client, &cmd);   // 立即返回
}
```

### 5. FreeRTOS server 端处理命令（非 ISR 上下文）

```c
/* FreeRTOS server 用 esp_amp_rpc_server_run 在任务上下文处理排队命令 */
esp_amp_rpc_server_run(server, pdMS_TO_TICKS(100));
/* 命令队列满时 server 返回 ESP_AMP_RPC_STATUS_SERVER_BUSY */
```

### 6. 命令状态码

| 值 | 宏 | 含义 |
|---|---|---|
| 0x0000 | `ESP_AMP_RPC_STATUS_OK` | 成功 |
| 0x0001 | `ESP_AMP_RPC_STATUS_SERVER_BUSY` | server 忙，命令丢弃 |
| 0x0002 | `ESP_AMP_RPC_STATUS_INVALID_CMD` | 找不到 cmd_id |
| 0x0003 | `ESP_AMP_RPC_STATUS_EXEC_FAILED` | server 执行出错 |
| 0x0004 | `ESP_AMP_RPC_STATUS_PENDING` | 仍在 pending（阻塞时即超时） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 超时后栈损坏 | 未 abort，迟到响应改写已释放的栈上 cmd | 超时必须 `esp_amp_rpc_client_abort_cmd(client, &cmd)` |
| resp_data 被截断 | resp_len 设小了 | resp_len 设为响应缓冲实际容量；超长部分被截断 |
| client 多任务并发崩溃 | client 实例非线程安全 | 同一 client 仅一个任务使用，或外部加锁 |
| `add_service` 返回 ERR_EXIST | cmd_id 已注册 | 先 `del_service` 再注册；一个 cmd_id 仅一个 handler |
| server 不处理命令 | bare-metal 未 poll RPMsg / FreeRTOS 未 run | bare-metal 主循环 `while(esp_amp_rpmsg_poll()==0)`；FreeRTOS 调 `esp_amp_rpc_server_run` |
| 仅 1 个 pending | 设计：同时只允许 1 个 pending 请求 | 需要响应才发下一个，或非阻塞不等待 |

## 参考

- `examples/rpc/maincore_client_subcore_server/maincore/main/app_main.c` — FreeRTOS client（阻塞+超时+abort）
- `examples/rpc/maincore_client_subcore_server/subcore/main/main.c` — bare-metal server
- `examples/rpc/subcore_client_maincore_server/` — 反向：subcore client / maincore server
- `espressif-repos/esp-amp/docs/rpc.md`
- `espressif-repos/esp-amp/components/esp_amp/include/esp_amp_rpc.h`
