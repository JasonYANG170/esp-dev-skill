# 运行时控制：上报开关、手动发送、Command-Response

> **适用摘要**: 在运行时控制 ESP-Insights 的上报行为（暂停/恢复/立即发送），以及启用 RainMaker MQTT 节点的 command-response 远程控制能力。

## 触发意图

- "运行时暂停 insights 上报"
- "立即发送诊断数据"
- "esp_insights_send_data"
- "command response 远程控制"
- "dashboard 远程重启设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| 运行 | 已 `esp_insights_init(&config)` 或 `esp_insights_enable(&config)` |
| command-response | `CONFIG_ESP_INSIGHTS_ENABLED=y` + `CONFIG_ESP_INSIGHTS_TRANSPORT_MQTT=y` + `CONFIG_ESP_INSIGHTS_CMD_RESP_ENABLED=y`；RainMaker claim 完成 |
| 参考 | `components/esp_insights/include/esp_insights.h`、`FEATURES.md` “Command Response” 段 |

## 分步说明

### 1. 查询/控制上报状态（来自 `esp_insights.h`）

```c
#include "esp_insights.h"

if (esp_insights_is_reporting_enabled()) {
    /* 当前正在上报 */
}

/* 暂停上报（注意：meta/boot 消息仍会发，因为对云端关键） */
esp_insights_reporting_disable();

/* 恢复上报 */
esp_insights_reporting_enable();
```

### 2. 立即手动发送（来自 `esp_insights.h`）

```c
/* 异步：读取 buffer 并尝试发送，可能需要时间才完成 */
esp_err_t ret = esp_insights_send_data();
```
适用：监听到 `*_LOW_MEM` 事件后立刻腾空、或业务关键节点主动 flush。

### 3. 彻底启停 Insights（来自 `esp_insights.h`）

```c
/* 关闭上报并断开 transport */
esp_insights_disable();    /* 不注销自定义 transport */

/* 重新启用（自定义 transport 场景配合 transport_register） */
esp_insights_enable(&config);

/* 完全销毁 */
esp_insights_deinit();
```

### 4. 监听 transport 事件（来自 `esp_insights.h`）

```c
#include "esp_event.h"
#include "esp_insights.h"

static void on_xport(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    esp_insights_transport_event_data_t *ed = data;
    switch (id) {
    case INSIGHTS_EVENT_TRANSPORT_SEND_SUCCESS:
        /* ed->msg_id 成功 */
        break;
    case INSIGHTS_EVENT_TRANSPORT_SEND_FAILED:
        /* ed->msg_id 失败，可重试或记日志 */
        break;
    case INSIGHTS_EVENT_TRANSPORT_RECV:
        /* ed->data / ed->data_len 收到云端下发 */
        break;
    }
}

void register_xport_events(void)
{
    esp_event_handler_register(INSIGHTS_EVENT, ESP_EVENT_ANY_ID, on_xport, NULL);
}
```

### 5. Command-Response（仅 RainMaker MQTT，来自 FEATURES.md + Kconfig）

启用后，仪表盘 Node 的 Settings 标签出现交互选项，可远程：重启设备、启停诊断采集、粒度启停 metrics/variables。

```c
#include "esp_insights.h"

/* 当通过 esp_insights_init() 启动时，内部已自动 enable cmd-resp。
 * 若只用 esp_insights_enable()（如 RainMaker app_insights），需自行调用： */
esp_insights_cmd_resp_enable();
```
前提 Kconfig（来自 `components/esp_insights/Kconfig`）：
```
CONFIG_ESP_INSIGHTS_ENABLED=y
CONFIG_ESP_INSIGHTS_TRANSPORT_MQTT=y
CONFIG_ESP_INSIGHTS_CMD_RESP_ENABLED=y
```
> HTTPS 节点不可用（该选项 `depends on ... && TRANSPORT_MQTT`）。

### 6. 获取 Node ID（来自 `esp_insights.h`）

```c
const char *node_id = esp_insights_get_node_id();  /* NULL 结尾字符串；未设则用 MAC */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `reporting_disable` 后仍有数据上报 | meta/boot 消息关键，仍发 | 要完全停用 `esp_insights_disable()` |
| `send_data` 没立刻到云 | 异步发送 + 动态间隔 | 接受延迟；或检查 transport SEND_FAILED 事件 |
| cmd-resp 在 dashboard 不出现 | HTTPS 或未 claim 或未启 Kconfig | 切 MQTT + claim + `CMD_RESP_ENABLED=y` |
| 只调 `enable()` 后 cmd-resp 不工作 | `enable` 不自动 enable cmd-resp | 显式 `esp_insights_cmd_resp_enable()` |
| `INSIGHTS_EVENT` 未定义 | 未 include `esp_insights.h` | 该 base 由头文件声明 |

## 参考

- `components/esp_insights/include/esp_insights.h` — `reporting_enable/disable`、`send_data`、`enable/disable/deinit`、`cmd_resp_enable`、`get_node_id`、transport 事件枚举
- `components/esp_insights/Kconfig` — `ESP_INSIGHTS_CMD_RESP_ENABLED` 依赖
- `FEATURES.md` “Command Response” 段 — dashboard 远程控制能力
- `CHANGELOG.md` 2024-07 条目 — command-response 框架（含 reboot 命令）
