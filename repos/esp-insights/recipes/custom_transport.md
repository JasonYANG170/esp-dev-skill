# 自定义 Transport（复用已有 TLS/MQTT 连接）

> **适用摘要**: 当默认 HTTPS/MQTT transport 不满足需求（例如想复用应用已有的 TLS/MQTT 连接以节省一次握手内存），通过 `esp_insights_transport_register()` + `esp_insights_enable()` 注入自定义 transport 回调。

## 触发意图

- "自定义 insights transport"
- "复用已有 MQTT 连接上报诊断"
- "esp_insights_transport_register"
- "节省 TLS 内存 insights"
- "自定义上报通道"

## 前置条件

| 条件 | 要求 |
|---|---|
| 理解 | 默认 `esp_insights_init()` 会内部注册 transport；自定义时必须改用 `esp_insights_enable()` |
| 回调 | 已实现 init/deinit/connect/disconnect/data_send 中需要的回调 |
| 参考 | RainMaker `examples/common/app_insights/app_insights.c`（仓库外，但 README 与 FEATURES.md 指明此模式） |

## 分步说明

### 1. 填充 transport 配置（来自 `esp_insights.h` 真实结构体）

```c
#include "esp_insights.h"
#include "esp_diagnostics.h"

/* 回调原型由 esp_insights.h 定义：
 *   esp_insights_transport_init_t       -> esp_err_t (*)(void *userdata)
 *   esp_insights_transport_deinit_t     -> void (*)(void)
 *   esp_insights_transport_connect_t    -> esp_err_t (*)(void)
 *   esp_insights_transport_disconnect_t -> void (*)(void)
 *   esp_insights_transport_data_send_t  -> int   (*)(void *data, size_t len)
 */
static esp_err_t my_xport_init(void *ud)      { /* 建立底层资源 */ return ESP_OK; }
static void      my_xport_deinit(void)        { /* 释放 */ }
static esp_err_t my_xport_connect(void)       { /* 复用已有连接则直接返回 OK */ return ESP_OK; }
static void      my_xport_disconnect(void)    { /* 通常不动复用连接 */ }
static int       my_xport_data_send(void *data, size_t len)
{
    /* 把 CBOR 数据塞进你已有的 MQTT/TLS 通道。
     * 返回值约定：失败=-1；成功且同步发送=0；
     *           成功且异步发送=正整数 msg_id（随后须发 INSIGHTS_EVENT_TRANSPORT_SEND_SUCCESS/FAILED） */
    int msg_id = my_mqtt_publish("insights/topic", data, len);
    return msg_id > 0 ? msg_id : (msg_id == 0 ? 0 : -1);
}
```

### 2. 注册 + enable（关键：不要用 init）

```c
void app_main(void)
{
    /* ... NVS / netif / event loop / 联网 / 时间同步 ... */

    esp_insights_transport_config_t tcfg = {
        .callbacks = {
            .init       = my_xport_init,
            .deinit     = my_xport_deinit,
            .connect    = my_xport_connect,
            .disconnect = my_xport_disconnect,
            .data_send  = my_xport_data_send,
        },
        .userdata = NULL,
    };
    ESP_ERROR_CHECK(esp_insights_transport_register(&tcfg));  /* 覆盖默认 transport */

    esp_insights_config_t icfg = {
        .log_type = ESP_DIAG_LOG_TYPE_ERROR | ESP_DIAG_LOG_TYPE_WARNING | ESP_DIAG_LOG_TYPE_EVENT,
        /* 注意：自定义 transport 下 auth_key 由你的回调自行处理，框架不再用 */
    };
    ESP_ERROR_CHECK(esp_insights_enable(&icfg));   /* 不再内部注册 transport */
}
```

### 3. 异步发送须发 transport 事件（来自 `esp_insights.h` 事件枚举）

若 `data_send` 返回正 msg_id（异步），需在发送结果已知时通过默认事件循环发出：

```c
#include "esp_event.h"

extern esp_event_loop_handle_t /* 或用 esp_event_post 到默认循环 */;

/* 发送成功 */
esp_insights_transport_event_data_t ed = { .msg_id = msg_id, .data = NULL, .data_len = 0 };
esp_event_post(INSIGHTS_EVENT, INSIGHTS_EVENT_TRANSPORT_SEND_SUCCESS, &ed, sizeof(ed), portMAX_DELAY);
/* 失败则用 INSIGHTS_EVENT_TRANSPORT_RECV 表示收数据 */
```
> `INSIGHTS_EVENT` 在 `esp_insights.h` 中以 `ESP_EVENT_DECLARE_BASE(INSIGHTS_EVENT);` 声明。

### 4. 关闭顺序（来自头文件注释）

```c
esp_insights_disable();              /* 仅停上报，不注销 transport */
esp_insights_transport_unregister(); /* 真正移除 transport 回调 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 二次注册 transport / 报已存在 | 同时调了 `esp_insights_init()` 与 `transport_register` | 自定义场景一律用 `enable()`，不要 `init()` |
| 同步发送却返回了 msg_id 导致框架等异步事件 | 返回值约定误用 | 同步成功返回 `0`；仅异步才返回正 msg_id 并补发事件 |
| `INSIGHTS_EVENT` 未定义 | 未 include `esp_insights.h` | 该 base 在头文件 `ESP_EVENT_DECLARE_BASE` 声明 |
| 内存未省 | 仍新建了独立 TLS 会话 | 复用已有 MQTT/TLS 句柄，`connect` 直接返回 OK |

## 参考

- `components/esp_insights/include/esp_insights.h` — `esp_insights_transport_register/enable/disable`、`esp_insights_transport_config_t`、`esp_insights_transport_event_data_t`、`INSIGHTS_EVENT_TRANSPORT_*`
- `README.md`（仓库）Behind the Scenes 段 — transport 选择说明
- `FEATURES.md` “Transport Sharing” 段 — RainMaker 复用模式说明
- RainMaker `examples/common/app_insights/app_insights.c`（仓库外，被 README/FEATURES 引用）— 完整复用样例
