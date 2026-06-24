# 编写 Event Router 自动化规则

> **适用摘要**: 用 `router_rules.json` 把事件（IM 消息、启动、按钮、定时、附件）路由到 `call_cap` / `run_agent` / `run_script` / `send_message` / `emit_event` / `drop`，并打通 Agent 回复到 IM 的 `out_message` 回路。

## 触发意图

- "加自动化规则"
- "router_rules.json 怎么写"
- "匹配 IM 消息跑脚本"
- "Agent 回复发到飞书"
- "命令前缀匹配 /xx"

## 前置条件

| 条件 | 要求 |
|---|---|
| 文件位置 | `/fatfs/router_rules/router_rules.json`（edge_agent 启动默认加载） |
| 控制台 | `auto`（edge_agent）或 `event_router`（cap_router_mgr）命令 |
| 参考 | `application/edge_agent/fatfs_image/system/.recovery/router_rules/router_rules.json` |

## 分步说明

### 1. 规则骨架与动作类型

每条规则：`id` / `enabled` / `consume_on_match` / `ack` / `match` / `actions[]`。规则数组顺序即求值顺序。动作类型：

| Action | 用途 |
|---|---|
| `call_cap` | 不经 LLM 直接调能力 |
| `run_agent` | 异步提交 `claw_core`，回复以 `out_message` 事件回来 |
| `run_script` | 跑 Lua（不经 LLM） |
| `send_message` | 经 outbound binding 发 IM |
| `emit_event` | 向路由器发新事件 |
| `drop` | 丢弃事件 |

> `run_script` / `send_message` / `emit_event` 的参数放在 `actions[*].input` 下（如 `{"type":"run_script","input":{"path":"demo.lua"}}`），**不要**放 action 顶层。

### 2. 匹配字段

`match` 常用字段：`source_cap` / `event_type` / `event_key` / `content_type` / `text`。
- `text` 默认精确匹配；要命令式用 `"text_match_rule":"prefix"`，路由左裁剪文本、要求命令后有词边界，剩余参数暴露为 `{{match.remainder}}`。
- 模板变量：`{{match.text}}` `{{match.rule}}` `{{match.remainder}}`；事件字段：`{{event.source_channel}}` `{{event.chat_id}}` `{{event.text}}` `{{event.event_type}}`。

### 3. 命令前缀 + 跑脚本

```json
{
  "id": "run_script_command",
  "enabled": true,
  "consume_on_match": true,
  "match": {
    "event_type": "message",
    "event_key": "text",
    "content_type": "text",
    "text": "/run",
    "text_match_rule": "prefix"
  },
  "actions": [
    {
      "type": "run_script",
      "input": {
        "path": "/fatfs/skills/runner/scripts/run.lua",
        "args": { "command": "{{match.remainder}}" }
      }
    }
  ]
}
```

### 4. IM 消息默认路由到 Agent

```json
{
  "id": "im_any_message_agent",
  "enabled": true,
  "consume_on_match": true,
  "ack": "{{event.source_channel}} routed to agent",
  "match": { "event_type": "message", "event_key": "text", "content_type": "text" },
  "actions": [
    { "type": "run_agent",
      "input": { "target_channel": "{{event.source_channel}}", "session_policy": "chat" } }
  ]
}
```

### 5. Agent 回复发回 IM（关键回路）

`run_agent` 是异步的，回复以 `out_message` 事件回来。必须配一条匹配它的 `send_message` 规则，否则 Agent 答了但 IM 收不到：

```json
{
  "id": "agent_out_message_send_message",
  "enabled": true,
  "consume_on_match": true,
  "match": { "source_cap": "claw_core", "event_type": "out_message", "content_type": "text" },
  "actions": [
    { "type": "send_message",
      "input": {
        "channel": "{{event.source_channel}}",
        "chat_id": "{{event.chat_id}}",
        "message": "{{event.text}}" } }
  ]
}
```
> `channel` / `chat_id` 通常来自触发 `run_agent` 时传入的 `target_channel` / `target_chat_id`。`agent_stage` 事件同理（受 `CLAW_CORE_STAGE_VERBOSITY` 控制），可路由到 IM 推工具调用进度。

### 6. 用 Console 改完即生效

```sh
auto add_rule '<上面 JSON>'      # 追加
auto update_rule '<JSON>'        # 按 id 替换
auto delete_rule <id>
auto reload                      # 重新加载文件（手编文件后必须）
auto last                        # 看上次匹配：matched / route(0=PASS,1=CONSUMED) / first_rule_id / last_error
```
> 手编 `router_rules.json`（Web 文件管理器）后必须 `auto reload`。

### 7. 注入事件用于联调

```sh
auto emit_message qq_gateway qq 123456 "hello world"
auto emit_trigger tester trigger smoke_test '{"ok":true}'
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Agent 有回复但 IM 收不到 | 没配 `out_message` → `send_message` 规则 | 加第 5 步那条规则 |
| `run_script` 不执行 | `path` 放在 action 顶层 | 放进 `input.path`；skill 自带脚本用 `{CUR_SKILL_DIR}/scripts/...` |
| 命令匹配不到 | `text` 默认精确匹配 | 命令式用 `"text_match_rule":"prefix"` |
| 改了文件没生效 | 未 `auto reload` | Console 跑 `auto reload` |
| 自环 / 死循环 | 路由器自身发出的事件被自己规则接住 | 路由器对自身 `source_cap=claw_event_router` 的事件会跳过防自环；避免规则间互相 `emit_event` 循环 |
| 多规则都命中导致重复回复 | `consume_on_match` 未设 | 需独占的规则设 `consume_on_match:true` |

## 参考

- `application/edge_agent/fatfs_image/system/.recovery/router_rules/router_rules.json`（含 startup / im_session_command / im_llm_command / im_any_message_working_reply / im_any_message_agent / agent_stage_im_notify / agent_out_message_send_message）
- `components/claw_modules/claw_event_router/include/claw_event.h`（`claw_event_t` 字段）
- `docs/src/content/docs/en/reference-project/dataflow-and-automation.mdx`（完整匹配状态机 + JSON Schema）
- `docs/src/content/docs/en/reference-project/console-usage.mdx`
- `docs/src/assets/router_rules.schema.json`
