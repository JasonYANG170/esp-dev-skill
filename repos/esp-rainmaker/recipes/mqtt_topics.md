# MQTT 直发、预算与主题

> **适用摘要**: 使用 `esp_rmaker_mqtt_*` 与 `esp_rmaker_publish_direct()` 进行 MQTT 直发，理解 MQTT 预算（budgeting）与 Basic Ingest 主题机制以降低成本与避免丢消息。

## 触发意图

- "RainMaker MQTT 直发"
- "esp_rmaker_publish_direct"
- "MQTT 预算 budget"
- "Basic Ingest 主题"
- "MQTT 消息丢失"

## 前置条件

| 条件 | 要求 |
|---|---|
| Agent | 已 `esp_rmaker_start()` 并连上 MQTT |
| Kconfig | `CONFIG_ESP_RMAKER_MQTT_*`（见下） |
| 参考示例 | `examples/led_light/main/app_main.c`（`CONFIG_EXAMPLE_DIRECT_MQTT` 分支） |

## 分步说明

### 1. 直发字符串到 direct params topic

```c
#include <esp_rmaker_core.h>

/* 发布到 node/<node_id>/direct/params/local[/<group_id>] */
esp_rmaker_publish_direct("Direct msg: 1");
```

> App 可订阅 `node/+/direct/params/local/<pgrp_id>` 直接从 MQTT broker 收消息，绕过云端处理（见 `esp_rmaker_groups.h` 注释）。

### 2. 检查 MQTT 预算与连接状态

```c
#include <esp_rmaker_mqtt.h>

if (esp_rmaker_mqtt_is_budget_available()) {
    /* 预算充足，可安全发布 */
}

if (esp_rmaker_is_mqtt_connected()) {
    /* 已连上 MQTT broker */
}
```

> `esp_rmaker_mqtt_publish()` 内部已做预算检查；自定义发布场景需手动判断。

### 3. 底层 MQTT API（高级，自定义 topic）

```c
esp_err_t esp_rmaker_mqtt_publish(const char *topic, void *data, size_t data_len,
                                  uint8_t qos, int *msg_id);
esp_err_t esp_rmaker_mqtt_subscribe(const char *topic,
                                    esp_rmaker_mqtt_subscribe_cb_t cb,
                                    uint8_t qos, void *priv_data);
esp_err_t esp_rmaker_mqtt_unsubscribe(const char *topic);
```

> 这些 API 由 RainMaker core 内部使用；应用一般用 `esp_rmaker_publish_direct()` 与参数上报 API。

### 4. MQTT 预算（Budgeting）机制

默认 `CONFIG_ESP_RMAKER_MQTT_ENABLE_BUDGETING=y`。每发一条消息预算减 1，按 `BUDGET_REVIVE_PERIOD`（默认 5 秒）周期恢复 `BUDGET_REVIVE_COUNT`（默认 1）。预算耗尽则消息被丢弃。

| Kconfig | 默认 | 范围 | 说明 |
|---|---|---|---|
| `CONFIG_ESP_RMAKER_MQTT_ENABLE_BUDGETING` | y | — | 启用预算限流 |
| `CONFIG_ESP_RMAKER_MQTT_DEFAULT_BUDGET` | 100 | 64 ~ MAX | 初始预算 |
| `CONFIG_ESP_RMAKER_MQTT_MAX_BUDGET` | 1024 | 64 ~ 2048 | 预算上限 |
| `CONFIG_ESP_RMAKER_MQTT_BUDGET_REVIVE_PERIOD` | 5 | 5 ~ 600 | 恢复周期（秒） |
| `CONFIG_ESP_RMAKER_MQTT_BUDGET_REVIVE_COUNT` | 1 | 1 ~ 16 | 每周期恢复数 |

高频上报时调大 `DEFAULT_BUDGET`/`MAX_BUDGET`，或合并上报。

### 5. Basic Ingest 主题（降低成本）

```text
CONFIG_ESP_RMAKER_MQTT_USE_BASIC_INGEST_TOPICS=y   # 默认
```

启用后节点→云端通信使用 AWS Basic Ingest 主题，绕过 MQTT broker，降低消息成本。`esp_rmaker_create_mqtt_topic()` 会据此构造主题字符串。

### 6. 合并上报减少消息数

```c
/* 仅更新 core（不立即上报） */
esp_rmaker_param_update(p1, esp_rmaker_int(1));
esp_rmaker_param_update(p2, esp_rmaker_int(2));
/* 一次性上报所有已更新参数 */
esp_rmaker_report_updated_params();
```

### 7. MQTT 连接事件

```c
/* RMAKER_COMMON_EVENT base */
case RMAKER_MQTT_EVENT_CONNECTED:
    ESP_LOGI(TAG, "MQTT Connected.");
    break;
case RMAKER_MQTT_EVENT_DISCONNECTED:
    ESP_LOGI(TAG, "MQTT Disconnected.");
    break;
case RMAKER_MQTT_EVENT_PUBLISHED:
    ESP_LOGI(TAG, "MQTT Published. Msg id: %d.", *((int *)event_data));
    break;
```

### 8. MQTT host 配置（私有部署）

```text
# No Claim 场景需要手动指定 MQTT host
CONFIG_ESP_RMAKER_READ_MQTT_HOST_FROM_CONFIG=y
CONFIG_ESP_RMAKER_MQTT_HOST="mqtt.example.com"
# Self/Assisted Claim 时 host 从 claiming 服务返回值读取
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 直发消息丢失 | 预算耗尽 | 调大预算或合并上报 |
| App 收不到 direct 消息 | 未订阅对应 topic 或未设 group_id | App 订阅 `node/+/direct/params/local/<pgrp_id>` |
| 预算恢复太慢 | `BUDGET_REVIVE_PERIOD/COUNT` 太小 | 调大恢复频率/数量 |
| 自定义 topic 连不上 | host 错误 | 设 `MQTT_HOST` 或检查 claiming 返回值 |
| 连上立即断开 | LWT/重复连接 | 检查 `CONNECTIVITY_REPORT_DELAY` 缓解竞态 |

## 参考

- `components/esp_rainmaker/include/esp_rmaker_mqtt.h`
- `components/esp_rainmaker/include/esp_rmaker_groups.h` — direct topic 规则
- `components/esp_rainmaker/Kconfig.projbuild` — MQTT 相关配置
- `examples/led_light/main/app_main.c` — `CONFIG_EXAMPLE_DIRECT_MQTT` 分支
