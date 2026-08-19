# 节点间时间同步（无需联网）

> **适用摘要**: initiator 广播权威时间，responder 接收并调整本地时间；适合从 deep sleep 唤醒的节点同步时间（基于 `espnow_time.h`）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-now/resources/`, source/examples in `repos/esp-now/`, and this recipe path `repos/esp-now/recipes/time_sync.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-NOW 时间同步"
- "节点对时"
- "espnow_time"
- "deep sleep 后同步时间"
- "无网时间同步"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `espnow_time.h`（节点间同步）, `espnow_utils.h`（SNTP `espnow_timesync_*`） |
| 参考示例 | `examples/solution/main/app_main.c`（`CONFIG_APP_ESPNOW_TIMESYNC` 分支） |

## 分步说明

### 1. Initiator：启动并周期广播权威时间

```c
#include "espnow_time.h"

// 配置：sync_interval_ms=0 表示仅按需广播
espnow_time_initiator_config_t icfg = ESPNOW_TIME_INITIATOR_CONFIG_DEFAULT();
// 周期广播可设: icfg.sync_interval_ms = 60 * 1000;  // 每 60s 广播一次
esp_err_t ret = espnow_time_initiator_start(&icfg);
// 注: initiator 从不调整自身时间
// 按需立即广播:
espnow_time_initiator_broadcast();

// 停止
espnow_time_initiator_stop();
```

### 2. Responder：接收并调整本地时间

```c
espnow_time_responder_config_t rcfg = ESPNOW_TIME_RESPONDER_CONFIG_DEFAULT();
// rcfg.max_drift_ms = 100;  // 漂移超过此值才调整（默认 100ms）
espnow_time_responder_start(&rcfg);

// deep sleep 唤醒后主动请求同步
espnow_time_responder_request();
```

### 3. 监听同步事件

```c
// 事件 ID（来自 espnow_time.h）
// ESP_EVENT_ESPNOW_TIMESYNC_STARTED   时间同步模块已启动
// ESP_EVENT_ESPNOW_TIMESYNC_STOPPED   已停止
// ESP_EVENT_ESPNOW_TIMESYNC_SYNCED    已与 initiator 同步
// ESP_EVENT_ESPNOW_TIMESYNC_TIMEOUT   请求超时
esp_event_handler_register(ESP_EVENT_ESPNOW, ESP_EVENT_ANY_ID,
                           timesync_event_cb, NULL);

static void timesync_event_cb(void *a, esp_event_base_t base, int32_t id, void *data)
{
    if (id == ESP_EVENT_ESPNOW_TIMESYNC_SYNCED) {
        espnow_timesync_event_t *e = data;
        ESP_LOGI(TAG, "synced from " MACSTR " drift=%" PRId32 "ms time_us=%lld",
                 MAC2STR(e->src_addr), e->drift_ms, e->synced_time_us);
    }
}
```

### 4. 事件数据与包结构

```c
// 同步事件附带数据
typedef struct {
    uint8_t src_addr[6];
    int32_t drift_ms;
    int64_t synced_time_us;
} espnow_timesync_event_t;

// 时间同步包（内部）
typedef struct {
    uint8_t version;
    uint8_t type;            // broadcast 或 request
    int64_t timestamp_us;    // 发送方启动后微秒
    int64_t utc_time_us;     // 发送方 UTC 微秒(0 表示不可用)
} __attribute__((packed)) espnow_time_packet_t;
```

### 5. 关联：基于 SNTP 的时间同步（来自 espnow_utils.h）

```c
// 这是另一套：联网后通过 SNTP 同步系统时间
espnow_timesync_start();             // 初始化 SNTP
bool ok = espnow_timesync_check();   // 是否已对时（相对 2020-01-01）
espnow_timesync_wait(10 * 1000);     // 阻塞等待对时(参数为 wait_ms)
```

> 区分：`espnow_time.h` 是 **节点间**（initiator↔responder，无需联网）；`espnow_utils.h` 的 `espnow_timesync_*` 是 **联网 SNTP**。两者不要混用。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| responder 时间不变 | 未 `espnow_time_responder_start` 或未收到广播 | initiator 先 `initiator_start` 并广播 |
| 唤醒后时间不对 | 唤醒后未 request | deep sleep 唤醒后调 `espnow_time_responder_request` |
| 频繁调整抖动 | `max_drift_ms` 太小 | 调大（默认 100ms） |
| 与 SNTP 混淆 | 用错 API | 节点间用 `espnow_time.h`，联网用 `espnow_timesync_*` |
| TIMEOUT | initiator 未运行 | 确认 initiator `initiator_start` 成功（非 `ESP_ERR_INVALID_STATE`） |

## 参考

- `src/time/include/espnow_time.h` — 节点间时间同步全部 API
- `src/utils/include/espnow_utils.h` — SNTP 对时 `espnow_timesync_*`
- `examples/solution/main/app_main.c` — `CONFIG_APP_ESPNOW_TIMESYNC` 分支
