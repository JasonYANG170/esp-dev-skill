# ESP_HOSTED 事件、心跳与传输故障恢复

> **适用摘要**: 订阅 `ESP_HOSTED_EVENT` 事件循环，感知协处理器 INIT、传输 UP/DOWN/失败，并用心跳（heartbeat）做活体检测；在传输故障或意外重启时自动 deinit → 重新 init/connect 恢复链路。这是生产级 ESP-Hosted 应用的必备模式。

## 触发意图

- "ESP-Hosted 传输断线重连"
- "host 监测 slave 心跳"
- "ESP_HOSTED_EVENT_TRANSPORT_UP"
- "协处理器崩溃恢复"
- "esp_hosted_configure_heartbeat"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include "esp_hosted.h"`（含 `esp_hosted_event.h`、`esp_hosted_misc.h`） |
| 参考例程 | `examples/host_hosted_events/main/main.c` |

## 分步说明

### 1. 事件 base 与事件 ID（来自 `host/esp_hosted_event.h`）

```c
ESP_EVENT_DECLARE_BASE(ESP_HOSTED_EVENT);

enum {
    ESP_HOSTED_EVENT_CP_INIT = 0,        // 协处理器已启动（带 reset reason）
    ESP_HOSTED_EVENT_CP_HEARTBEAT,       // 心跳
    ESP_HOSTED_EVENT_TRANSPORT_FAILURE,  // 传输故障
    ESP_HOSTED_EVENT_TRANSPORT_UP,       // 传输就绪
    ESP_HOSTED_EVENT_TRANSPORT_DOWN,     // 传输掉线
    ESP_HOSTED_EVENT_MEM_MONITOR,        // 内存监控
};

// CP_INIT 事件参数
typedef struct { esp_reset_reason_t reason; } esp_hosted_event_init_t;
// HEARTBEAT 事件参数
typedef struct { uint32_t heartbeat; } esp_hosted_event_heartbeat_t;
```

### 2. 注册事件处理 + 用信号量等待就绪

```c
#include "freertos/semphr.h"
#include "esp_event.h"
#include "esp_hosted.h"

static SemaphoreHandle_t sem_hosted_is_up;

static void esp_hosted_event_handler(void *arg, esp_event_base_t base,
                                     int32_t id, void *data)
{
    if (base != ESP_HOSTED_EVENT) return;

    switch (id) {
    case ESP_HOSTED_EVENT_CP_INIT: {
        esp_hosted_event_init_t *e = (esp_hosted_event_init_t *)data;
        ESP_LOGI(TAG, "CP INIT, reset_reason=%d", e->reason);
        // 首次 CP_INIT 正常；运行期再次 CP_INIT 说明 slave 重启 -> 触发恢复
        break;
    }
    case ESP_HOSTED_EVENT_TRANSPORT_UP:
        ESP_LOGI(TAG, "Transport UP");
        xSemaphoreGive(sem_hosted_is_up);   // 放行主流程
        break;
    case ESP_HOSTED_EVENT_TRANSPORT_DOWN:
        ESP_LOGW(TAG, "Transport DOWN");
        break;
    case ESP_HOSTED_EVENT_TRANSPORT_FAILURE:
        ESP_LOGE(TAG, "Transport FAILURE -> 触发恢复");
        // 置位事件组让恢复线程去 deinit + 重新 init
        break;
    case ESP_HOSTED_EVENT_CP_HEARTBEAT: {
        esp_hosted_event_heartbeat_t *e = (esp_hosted_event_heartbeat_t *)data;
        ESP_LOGI(TAG, "heartbeat %" PRIu32, e->heartbeat);
        break;
    }
    default:
        break;
    }
}

void app_main(void)
{
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_event_handler_instance_t inst;
    ESP_ERROR_CHECK(esp_event_handler_instance_register(
        ESP_HOSTED_EVENT, ESP_EVENT_ANY_ID, esp_hosted_event_handler, NULL, &inst));

    sem_hosted_is_up = xSemaphoreCreateBinary();

    esp_hosted_init();
    esp_hosted_connect_to_slave();

    xSemaphoreTake(sem_hosted_is_up, portMAX_DELAY);  // 等 TRANSPORT_UP
    // 继续 Wi-Fi/BT 初始化...
}
```

### 3. 启用心跳（活体检测）

```c
// 在 TRANSPORT_UP 之后调用
// 最小 1 秒，最大 24 小时
esp_err_t r = esp_hosted_configure_heartbeat(true, 5);  // 每 5 秒一次
if (r != ESP_OK) {
    ESP_LOGE(TAG, "configure heartbeat failed: %s", esp_err_to_name(r));
}
```

例程还演示了用 `esp_timer` 给心跳加超时监视：在收到 `ESP_HOSTED_EVENT_CP_HEARTBEAT` 时 `esp_timer_restart`，超时则认定 slave 卡死（见 `host_hosted_events` 例程的 `HEARTBEAT_TIMEOUT_SEC` 配置与 `my_timer_cb`）。

### 4. 传输故障恢复循环

`host_hosted_events` 例程的 `app_main` 用一个 while 循环封装恢复：

```c
while (true) {
    init_cp_error_detection();
    app_esp_hosted_init();                         // esp_hosted_init + connect_to_slave
    xSemaphoreTake(sem_hosted_is_up, portMAX_DELAY);

    bool ok = app_esp_hosted_verify_up();          // e.g. 查 fw 版本/能力
    if (ok) {
        esp_hosted_configure_heartbeat(true, HEARTBEAT_INTERVAL_SEC);
        example_wifi_init_sta();
        // 阻塞等待 ESP_HOSTED_RESET_BIT（由 TRANSPORT_FAILURE / 意外 CP_INIT / 心跳超时置位）
        xEventGroupWaitBits(s_esp_hosted_event_group, ESP_HOSTED_RESET_BIT,
                            pdTRUE, pdTRUE, portMAX_DELAY);
    }
    // 告知数据线程中止，deinit Wi-Fi/netif
    resetting_esp_hosted_transport = true;
    app_do_cp_recovery();                          // esp_hosted_deinit() + 重新 init/connect
    resetting_esp_hosted_transport = false;
    deinit_cp_error_detection();
}
```

### 5. 复位策略（Kconfig）

`Component config -> ESP-Hosted config -> Common Slave Reset Strategy`：
- `ESP_HOSTED_SLAVE_RESET_ON_EVERY_HOST_BOOTUP`（默认，最稳，每次 host 启动都复位 slave）
- `ESP_HOSTED_SLAVE_RESET_ONLY_IF_NECESSARY`

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 主流程在 TRANSPORT_UP 之前跑飞 | 没等信号量 | 用信号量在 UP 之前阻塞 |
| 心跳超时但未恢复 | 未监视或未在回调里置恢复标志 | 用 esp_timer 超时 + 事件组触发 deinit/重连 |
| 反复 CP_INIT | slave 不停重启 | 检查 slave 供电、Reset 接线、是否 OTA 失败 |
| 恢复后 Wi-Fi 不工作 | 恢复时未清理旧 netif/event handler | 例程在恢复前 `example_wifi_deinit_sta()` / 关 netif |
| 心跳间隔非法 | < 1s 或 > 24h | `esp_hosted_configure_heartbeat` 返回 `ESP_ERR_INVALID_ARG` |

## 参考

- `host/esp_hosted_event.h`（事件 ID 与参数结构）
- `host/esp_hosted_misc.h`（`esp_hosted_configure_heartbeat`、`esp_hosted_set_mem_monitor`）
- `examples/host_hosted_events/main/main.c`（完整恢复循环）
- `examples/host_hosted_events/main/station_example.c`（Wi-Fi deinit/reinit 配合恢复）
- `docs/migration_guide.md`（v2.12.4 自定义回调签名变更）
