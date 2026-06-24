# 调度与场景

> **适用摘要**: 启用 RainMaker 调度（Scheduling）与场景（Scenes）服务，理解触发源（`ESP_RMAKER_REQ_SRC_SCHEDULE` / `_SCENE_ACTIVATE`），并配置最大数量与日光（日出/日落）支持。

## 触发意图

- "RainMaker 定时调度"
- "esp_rmaker_schedule_enable"
- "场景 scenes"
- "日出日落定时"
- "schedule 触发源"

## 前置条件

| 条件 | 要求 |
|---|---|
| 节点 | 已 `esp_rmaker_node_init()` |
| 时区 | 建议先 `esp_rmaker_timezone_service_enable()`（调度依赖正确时间） |
| Kconfig | `CONFIG_ESP_RMAKER_SCHEDULING_MAX_SCHEDULES` / `_SCENES_MAX_SCENES` |

## 分步说明

### 1. 启用调度与场景（必须在 `esp_rmaker_start()` 之前）

```c
#include <esp_rmaker_schedule.h>
#include <esp_rmaker_scenes.h>

esp_rmaker_timezone_service_enable();  /* 调度依赖时区 */
esp_rmaker_schedule_enable();
esp_rmaker_scenes_enable();
```

> 调度依赖正确时间，建议同时启用时区服务并让用户在 App 里设时区（见 `recipes/services.md`）。

### 2. 调度触发后的请求源

当调度触发，write 回调的 `ctx->src` 会是 `ESP_RMAKER_REQ_SRC_SCHEDULE`：

```c
static esp_err_t write_cb(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
            const esp_rmaker_param_val_t val, void *priv_data, esp_rmaker_write_ctx_t *ctx)
{
    if (ctx) {
        ESP_LOGI(TAG, "Write via : %s", esp_rmaker_device_cb_src_to_str(ctx->src));
        /* 打印结果如 "Write via : Schedule" */
    }
    /* ... 处理参数 ... */
    esp_rmaker_param_update(param, val);
    return ESP_OK;
}
```

请求源枚举（`esp_rmaker_req_src_t`）：

| 枚举 | 含义 |
|---|---|
| `ESP_RMAKER_REQ_SRC_INIT` | 初始化序列（持久化参数恢复） |
| `ESP_RMAKER_REQ_SRC_CLOUD` | 云端下发 |
| `ESP_RMAKER_REQ_SRC_SCHEDULE` | 调度触发 |
| `ESP_RMAKER_REQ_SRC_SCENE_ACTIVATE` | 场景激活 |
| `ESP_RMAKER_REQ_SRC_SCENE_DEACTIVATE` | 场景反激活 |
| `ESP_RMAKER_REQ_SRC_LOCAL` | 本地控制 |
| `ESP_RMAKER_REQ_SRC_CMD_RESP` | 命令-响应框架 |
| `ESP_RMAKER_REQ_SRC_FIRMWARE` | 固件/console 命令 |
| `ESP_RMAKER_REQ_SRC_BLE_LOCAL` | BLE 本地控制 |

### 3. 场景反激活支持（可选）

默认场景只触发激活。若需反激活回调，启用：

```text
# sdkconfig.defaults
CONFIG_ESP_RMAKER_SCENES_DEACTIVATE_SUPPORT=y
```

> 反激活时回调 `ctx->src = ESP_RMAKER_REQ_SRC_SCENE_DEACTIVATE`，参数值与激活相同（core 不知反激活目标值，由应用判断）。

### 4. 标准调度/场景服务 helper（高级，自定义集成时）

```c
#include <esp_rmaker_standard_services.h>

/* 自定义调度服务（一般无需手写，esp_rmaker_schedule_enable 内部已建） */
esp_rmaker_device_t *sched_serv = esp_rmaker_create_schedule_service(
        "Schedule", write_cb, read_cb,
        CONFIG_ESP_RMAKER_SCHEDULING_MAX_SCHEDULES, NULL);

/* 自定义场景服务 */
esp_rmaker_device_t *scene_serv = esp_rmaker_create_scenes_service(
        "Scenes", write_cb, read_cb,
        CONFIG_ESP_RMAKER_SCENES_MAX_SCENES,
        false /* deactivation_support */, NULL);
```

标准参数 helper（用于自定义服务集成）：
```c
esp_rmaker_schedules_param_create(ESP_RMAKER_DEF_SCHEDULE_NAME, max_schedules);
esp_rmaker_scenes_param_create(ESP_RMAKER_DEF_SCENES_NAME, max_scenes);
```

### 5. 日光（日出/日落）调度

```text
# sdkconfig.defaults
CONFIG_ESP_SCHEDULE_ENABLE_DAYLIGHT=y
CONFIG_ESP_RMAKER_SCHEDULE_ENABLE_DAYLIGHT=y
```

> 启用后 JSON API 支持基于地理位置的日出/日落调度。依赖 `esp_schedule` 组件（`ESP_SCHEDULE_ENABLE_DAYLIGHT`）。

### 6. 关键 Kconfig

| 符号 | 默认 | 说明 |
|---|---|---|
| `CONFIG_ESP_RMAKER_SCHEDULING_MAX_SCHEDULES` | 10 | 最大调度数（1~50） |
| `CONFIG_ESP_RMAKER_SCENES_MAX_SCENES` | 10 | 最大场景数（1~50） |
| `CONFIG_ESP_RMAKER_SCENES_DEACTIVATE_SUPPORT` | n | 场景反激活回调 |
| `CONFIG_ESP_RMAKER_SCHEDULE_ENABLE_DAYLIGHT` | y | 日光调度（依赖 `ESP_SCHEDULE_ENABLE_DAYLIGHT`） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 调度时间不对 | 时区未设 | 先 `esp_rmaker_timezone_service_enable()` 并让 App 设时区 |
| 调度不触发 | enable 在 start 之后 | 所有 `*_enable` 必须在 `esp_rmaker_start()` 前 |
| 场景反激活无回调 | 未开反激活支持 | 设 `SCENES_DEACTIVATE_SUPPORT=y` |
| 调度数超限 | 超过 `MAX_SCHEDULES` | 调大配置（注意 JSON payload 也会变大） |
| 日光调度不可用 | 缺 `esp_schedule` 日光支持 | 启用 `ESP_SCHEDULE_ENABLE_DAYLIGHT` |

## 参考

- `components/esp_rainmaker/include/esp_rmaker_schedule.h`
- `components/esp_rainmaker/include/esp_rmaker_scenes.h`
- `components/esp_rainmaker/include/esp_rmaker_standard_services.h`
- `components/esp_rainmaker/Kconfig.projbuild` — Scheduling / Scenes 菜单
- 官方文档：https://rainmaker.espressif.com/docs/scheduling.html
