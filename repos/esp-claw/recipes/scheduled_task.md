# 定时任务（scheduler + router rule）

> **适用摘要**: 用 `cap_scheduler` 按时发布 `schedule` 事件，再用 Event Router 规则决定触发后做什么（唤醒 Agent / 发固定 IM / 跑 Lua）。支持 `cron` / `interval` / `once`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-claw/resources/`, source/examples in `repos/esp-claw/`, and this recipe path `repos/esp-claw/recipes/scheduled_task.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "每天 8 点提醒我"
- "每 5 分钟跑一次脚本"
- "定时唤醒 Agent"
- "scheduler_add / schedules.json"
- "cron / interval / once"

## 前置条件

| 条件 | 要求 |
|---|---|
| 子系统 | `APP_CLAW_CAP_SCHEDULER` 启用（默认 y，会 select `APP_CLAW_CAP_EVENT_ROUTER`） |
| 时间同步 | `cron` / `once` 依赖 SNTP；首次同步成功会 `cap_scheduler_handle_time_sync` 重基准 |
| 参考 | `docs/.../reference-cap/cap-scheduler.mdx`、`scheduled_task` Skill |

## 分步说明

### 1. 两步法（调度与执行分离）

`cap_scheduler` 只按时发事件；行为由路由规则决定。新增定时任务通常：
1. 加一条 schedule（`scheduler_add` 或 `schedules.json`）
2. 加一条匹配它的 router rule（`add_router_rule` 或 `router_rules.json`）

典型匹配：schedule 设 `event_type:"schedule"` + `event_key:<id>`，规则用 `match.event_type` + `match.event_key` 接。

### 2. schedules.json 条目示例

```json
{
  "id": "morning_reminder",
  "kind": "cron",
  "cron_expr": "0 7 * * *",
  "enabled": true,
  "event_type": "schedule",
  "event_key": "reminder",
  "text": "早上好，记得喝水。"
}
```
- `kind`：`once` / `interval` / `cron`（cron 为标准 5 段：`minute hour mday month wday`，无秒）
- 状态文件 `schedules.json.state` 自动派生（`last_fire_ms` / `run_count`），重启不丢/不重
- `cron` 匹配基于设备本地时间 `localtime_r`

### 3. 配套 router rule（发到 IM）

```json
{
  "id": "handle_morning_reminder",
  "enabled": true,
  "match": { "event_type": "schedule", "event_key": "reminder" },
  "actions": [
    { "type": "send_message",
      "input": { "channel": "telegram", "chat_id": "YOUR_CHAT_ID", "message": "{{event.text}}" } }
  ]
}
```

### 4. 用 Console 操作

```sh
scheduler --list
scheduler --add --json '{"id":"morning_reminder","kind":"cron","cron_expr":"0 7 * * *","event_type":"schedule","event_key":"reminder","text":"早"}'
scheduler --enable <id>        # 或 --disable / --pause / --resume
scheduler --trigger <id>       # 立即触发一次（不影响后续计划）
scheduler --reload             # 从盘重载
```

### 5. 用 LLM 工具 / scheduled_task Skill

`cap_scheduler` 工具（默认对 LLM 可见）：`scheduler_list` / `scheduler_get` / `scheduler_add` / `scheduler_update` / `scheduler_remove` / `scheduler_enable` / `scheduler_disable` / `scheduler_pause` / `scheduler_resume` / `scheduler_trigger_now` / `scheduler_reload`。
- `scheduler_add` 入参 `schedule_json` 是**字符串**（转义的 JSON 串），不是嵌套对象：`{"schedule_json":"<JSON string>"}`
- `scheduler_*` 的 `id` 入参：`{"id":"..."}`

推荐用内置 `scheduled_task` Skill（一条命令搞定 schedule + rule）：它跑 `{CUR_SKILL_DIR}/scripts/add_scheduled_task.lua`，支持 `mode`: `wake_agent`（复用 `im_any_message_agent`，只建 schedule）/ `send_message` / `run_script`（建 rule + schedule，失败回滚 rule）。

唤醒 Agent 示例（每天 17:09）：
```json
{"path":"{CUR_SKILL_DIR}/scripts/add_scheduled_task.lua",
 "args":{"task_id":"weather_outfit_reminder","kind":"cron","cron_expr":"9 17 * * *",
         "mode":"wake_agent","text":"查今天天气并告诉我穿什么。",
         "chat_channel":"feishu","chat_id":"ou_xxx","trigger_count":0},
 "timeout_ms":60000}
```

### 6. C 初始化（应用层，参考）

```c
ESP_RETURN_ON_ERROR(cap_scheduler_init(&(cap_scheduler_config_t) {
    .schedules_path = "/fatfs/scheduler/schedules.json",
    .tick_ms        = 1000,
    .max_items      = 32,
    .task_stack_size= 6144,
    .task_priority  = 5,
    .task_core      = tskNO_AFFINITY,
    .publish_event  = claw_event_router_publish,
    .persist_after_fire = true,
}), TAG, "init scheduler");
// cap_scheduler_start() 在 claw_event_router_start() 之后调用
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 定时到了没反应 | 只加了 schedule，没加匹配的 router rule | 加第 3 步规则；或用 `scheduled_task` Skill |
| cron / once 不触发 | 系统时间未同步（早于 2024-01-01） | 等 SNTP 同步；`get_current_time` 会先尝试同步 |
| interval 不触发 | `interval_ms <= 0` | interval 必须 > 0；不依赖外部时间，上电即计时 |
| 重启后重复触发 | 误以为无状态 | 有 `.state` 文件记录 `last_fire_ms`/`run_count`，正常不会重 |
| `scheduler_add` 报 JSON 错 | 传了嵌套对象 | `schedule_json` 要是转义字符串 |
| 触发次数不符 | `max_runs` / `trigger_count` 混淆 | `trigger_count` 优先；`0` = 无限 |

## 参考

- `application/edge_agent/fatfs_image/system/.recovery/scheduler/schedules.json`
- `application/edge_agent/fatfs_image/storage/skills/scheduled_task/SKILL.md` 与 `scripts/add_scheduled_task.lua`
- `components/claw_capabilities/cap_scheduler/`（`src/cap_scheduler.c`、`include/cap_scheduler.h`）
- `docs/src/content/docs/en/reference-cap/cap-scheduler.mdx`
- `docs/src/content/docs/en/reference-project/dataflow-and-automation.mdx`（Scheduled Dispatch 段）
