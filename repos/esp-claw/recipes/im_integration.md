# IM 平台端到端集成（事件接入 → 路由 → Agent → 回复 → 附件）

> **适用摘要**: 打通「IM 入站消息 → Event Router → claw_core → out_message 回送 IM」的完整闭环，理解 `cap_im_platform` 统一源组件、各平台 chat_id 格式与差异、附件异步落盘后的 `attachment_saved` 事件，以及 `router_rules.json` 里真实存在的 7 条 IM 相关规则。

## 触发意图

- "Telegram / 飞书 / QQ / 微信 收消息没反应"
- "Agent 回复发不到 IM"
- "cap_im_platform 怎么接入"
- "附件下载怎么处理 / attachment_saved"
- "chat_id 怎么填 / 各平台差异"
- "IM 事件源 → Agent 全链路"
- "去重缓存 / 长轮询 / WebSocket"

## 前置条件

| 条件 | 要求 |
|---|---|
| 能力组件 | `CONFIG_APP_CLAW_CAP_IM_<PLATFORM>=y`（`<PLATFORM>` = `TG` / `FEISHU` / `QQ` / `WECHAT` / `LOCAL`） |
| 凭据配置 | 对应 App ID / Secret / Token（运行时 NVS 优先，见 `recipes/configuration.md`）；微信走扫码登录 |
| Agent 核心 | `llm_api_key` / `llm_model` / `llm_backend_type` 三者非空（否则 `claw_core` 不启动，IM 入站只能跑本地规则，不能走 Agent） |
| 出站绑定 | `edge_agent` 启动时 `claw_event_router_register_outbound_binding` 已把 `qq/feishu/telegram/wechat/web` 绑到对应 send 工具 |
| 内置 Skill | `cap_im_platform`（cap_groups: `cap_im_feishu` / `cap_im_qq` / `cap_im_tg` / `cap_im_wechat`） |
| 参考示例 | `application/edge_agent/fatfs_image/system/.recovery/router_rules/router_rules.json`、`components/claw_capabilities/cap_im_platform/src/cap_im_tg.c` |

## 分步说明

### 1. cap_im_platform 的统一源 + 分平台运行时模型

`cap_im_platform` 是**一个 ESP-IDF 源组件**，但构建期依赖统一、运行时仍按平台拆分能力组（保留旧的 enable/disable / Skill 可见性行为）。各平台后端在各自 `.c` 文件里实现同一个三段式：

```text
1. 事件源：从 IM 平台收消息 → 归一化 → publish 到 claw_event_router
2. 可调用工具：暴露 *_send_message / *_send_image / *_send_file
3. 附件处理：媒体异步下载到 inbox → 发 attachment_saved 事件
```

| 运行时组 | 事件源 | 文本 | 图片 | 文件 |
|---|---|---|---|---|
| `cap_im_feishu` | `feishu_gateway` | `feishu_send_message` | `feishu_send_image` | `feishu_send_file` |
| `cap_im_qq` | `qq_gateway` | `qq_send_message` | `qq_send_image` | `qq_send_file` |
| `cap_im_tg` | `tg_gateway` | `tg_send_message` | `tg_send_image` | `tg_send_file` |
| `cap_im_wechat` | `wechat_gateway` | `wechat_send_message` | `wechat_send_image` | **不支持** |
| `cap_im_local` | `local_gateway` | `local_send_message` | — | — |

### 2. 平台差异（chat_id 格式与能力）

| 平台 | 入站模型 | chat 目标 | 备注 |
|---|---|---|---|
| 飞书 | WebSocket / Event API | `chat_id` 或 `ou_` 开头的 `open_id` | 文本优先 Markdown 交互卡片 + 纯文本兜底；caption 作为后续文本发 |
| QQ | QQ Bot WebSocket API | `c2c:<openid>` 或 `group:<group_openid>` | 文件下发依赖 QQ 平台支持；图/文件是不同工具调用 |
| Telegram | Bot API **长轮询**（20 s） | 数字 chat id（如 `123456789` 或 `-100...`） | 长文本自动分块；文件 multipart 流式上传（不整载入 RAM） |
| 微信 | ClawBot 轮询 API | 具体 room id 或 contact id | 仅文本/图片；非图片文件**不可**发；`wechat_send_*` 不回退到上下文，必须显式 `chat_id` |
| Web (local) | HTTP/WebSocket | `web` channel | Web 配置页内置聊天 UI，仅文本 |

> Feishu / QQ / Telegram 的 `*_send_message` 在 `chat_id` 缺省时会**回退到当前调用上下文**（`ctx->chat_id`）；微信不会，必须显式传。

### 3. 入站 → publish_message（以 Telegram 为参考实现）

`cap_im_tg` 在 group `start` 钩子里起两个 FreeRTOS 任务。`tg_poll_task` 长轮询 `getUpdates`，每条文本 update 调：

```c
// components/claw_capabilities/cap_im_platform/src/cap_im_tg.c
claw_event_router_publish_message(
    "tg_gateway",   // source_cap
    "telegram",     // source_channel（绑定到 tg_send_*）
    chat_id,        // 平台 chat id
    text,           // 正文
    sender_id,
    message_id
);
```

网络抖动可能重放同一条 update，后端用 **FNV-1a 64 位哈希环**去重（`cap_im_tg.c`，`CAP_IM_TG_DEDUP_CACHE_SIZE = 64`）：

```c
static bool cap_im_tg_dedup_check_and_record(const char *update_key) {
    uint64_t key = cap_im_tg_fnv1a64(update_key);
    for (size_t i = 0; i < CAP_IM_TG_DEDUP_CACHE_SIZE; i++) {
        if (s_tg.seen_update_keys[i] == key) return true;  // 已处理
    }
    s_tg.seen_update_keys[s_tg.seen_update_idx] = key;
    s_tg.seen_update_idx = (s_tg.seen_update_idx + 1) % CAP_IM_TG_DEDUP_CACHE_SIZE;
    return false;
}
```

### 4. 附件异步落盘 + attachment_saved

媒体下载慢，Telegram/QQ/飞书都走异步：poll 把 `attachment_job` 入队 → 独立任务调 `getFile`、流式写入 FATFS → 完成后发 `attachment_saved` 事件，载荷含本地路径、MIME、大小、平台元数据：

```json
// attachment_saved 的 payload_json（真实字段）
{
  "platform": "telegram",
  "attachment_kind": "photo",
  "saved_path": "/fatfs/inbox/telegram/-123456/789/photo.jpg",
  "saved_dir":  "/fatfs/inbox/telegram/-123456/789",
  "saved_name": "photo.jpg",
  "mime": "image/jpeg",
  "caption": "Look at this",
  "platform_file_id": "AgACAgIAAxkBAAI...",
  "size_bytes": 45231,
  "saved_at_ms": 1714000000000
}
```

附件根目录与大小上限在 app 启动时通过各平台 setter 配置（真实 API）：

```c
// 头文件：components/claw_capabilities/cap_im_platform/include/cap_im_tg.h
typedef struct {
    const char *storage_root_dir;            // 默认 /fatfs/inbox
    size_t      max_inbound_file_bytes;      // 默认 2 * 1024 * 1024
    bool        enable_inbound_attachments;
} cap_im_tg_attachment_config_t;

esp_err_t cap_im_tg_set_token(const char *bot_token);
esp_err_t cap_im_tg_set_attachment_config(const cap_im_tg_attachment_config_t *config);
// 飞书/QQ 同名前缀：cap_im_feishu_set_attachment_config / cap_im_qq_set_attachment_config
```

### 5. 完整 IM 闭环规则（仓库真实 `router_rules.json`）

`application/edge_agent/fatfs_image/system/.recovery/router_rules/router_rules.json` 含 7 条 IM 相关规则，构成入站到回复的完整回路。规则数组**按顺序**求值：

```json
[
  // (a) 命令前缀：/session 切会话（先 call_cap 再 send_message 回执）
  {
    "id": "im_session_command",
    "consume_on_match": true,
    "match": {"event_type":"message","event_key":"text","content_type":"text",
              "text":"/session","text_match_rule":"prefix"},
    "actions": [
      {"type":"call_cap","cap":"session_command","input":{"command":"{{match.remainder}}"}},
      {"type":"send_message","input":{"channel":"{{event.source_channel}}","chat_id":"{{event.chat_id}}"}}
    ]
  },
  // (b) 命令前缀：/llm 改 LLM 配置
  { "id":"im_llm_command", /* 同上结构，cap:"llm_config_command", text:"/llm" */ },
  // (c) 收到任何 IM 文本先回一句「正在处理」（consume_on_match:false，不拦截后续）
  {
    "id":"im_any_message_working_reply",
    "consume_on_match": false,
    "match": {"event_type":"message","event_key":"text","content_type":"text"},
    "actions":[{"type":"send_message","input":{
        "channel":"{{event.source_channel}}","chat_id":"{{event.chat_id}}",
        "message":"🦞 ESP-Claw is snapping on it..."}}]
  },
  // (d) 附件落盘后回执
  {
    "id":"im_attachment_saved_reply",
    "match":{"event_type":"attachment_saved"},
    "actions":[{"type":"send_message","input":{
        "channel":"{{event.source_channel}}","chat_id":"{{event.chat_id}}",
        "message":"File received from {{event.source_channel}}"}}]
  },
  // (e) 把 IM 文本路由给 Agent（异步，回复以 out_message 事件回来）
  {
    "id":"im_any_message_agent",
    "consume_on_match": true,
    "match":{"event_type":"message","event_key":"text","content_type":"text"},
    "actions":[{"type":"run_agent","input":{
        "target_channel":"{{event.source_channel}}","session_policy":"chat"}}]
  },
  // (f) Agent 工具调用进度推回 IM（受 CLAW_CORE_STAGE_VERBOSITY 控制）
  {
    "id":"agent_stage_im_notify",
    "consume_on_match": true,
    "match":{"source_cap":"claw_core","event_type":"agent_stage","content_type":"text"},
    "actions":[{"type":"send_message","input":{
        "channel":"{{event.source_channel}}","chat_id":"{{event.chat_id}}",
        "message":"{{event.text}}"}}]
  },
  // (g) Agent 最终回复推回 IM（关键回路，缺这条 Agent 答了 IM 收不到）
  {
    "id":"agent_out_message_send_message",
    "consume_on_match": true,
    "match":{"source_cap":"claw_core","event_type":"out_message","content_type":"text"},
    "actions":[{"type":"send_message","input":{
        "channel":"{{event.source_channel}}","chat_id":"{{event.chat_id}}",
        "message":"{{event.text}}"}}]
  }
]
```

> `run_agent` 是异步的：它把请求提交给 `claw_core`，回复以 `out_message` 事件**回到路由器**，不会自动发到 IM。第 (g) 条规则就是把这个事件转回 IM 的桥。`channel`/`chat_id` 通常来自触发 `run_agent` 时传入的 `target_channel` / `target_chat_id`。

### 6. 让 LLM 能发额外消息/图片/文件

默认情况下 Agent 的回复走上面的 `out_message` 回路自动发。**额外**主动发文本/图片/文件时，让 LLM 激活 `cap_im_platform` Skill（声明 4 个 IM 组），调对应 send 工具：

```json
// 给当前 Telegram 回一句文本（chat_id 缺省回退到调用上下文）
// tg_send_message
{ "message": "The task has been completed." }

// 发本地图片到显式 Telegram chat
// tg_send_image
{ "chat_id":"-1001234567890", "path":"/fatfs/inbox/capture.jpg", "caption":"Here is the image." }

// 微信：必须显式 chat_id，且不能发非图片文件
// wechat_send_message
{ "chat_id":"room123", "message":"device online" }
```

选渠道规则（来自内置 `cap_im_platform` Skill）：优先用当前请求的 `source_cap`，不要跨渠道乱发；除非用户明确指定，不要臆造目标 chat。图片走 image 工具（`.jpg/.jpeg/.png/.gif/.webp`），其它走 file 工具。`path` 必须是**本地真实路径**，不要传远程 URL。

### 7. 调试与联调

```sh
# 注入一条假 IM 消息测路由（不依赖真实平台）
auto emit_message tg_gateway telegram -1001234567890 "hello world"

# 看上次匹配结果：matched / route(0=PASS,1=CONSUMED) / first_rule_id / last_error
auto last

# 手编 router_rules.json 后必须 reload
auto reload
```

排查顺序：①凭据配齐 + core 启动？②`auto last` 是否命中预期规则？③`out_message` → `send_message` 规则在不在？④出站绑定 (`qq/feishu/telegram/wechat/web`) 是否对？⑤微信是否扫码登录了？

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| IM 收消息但 Agent 不回 | 缺第 (g) 条 `out_message → send_message` 规则 / `claw_core` 没启动 | 加那条规则；配齐 LLM 三项重启 |
| Agent 答了但 IM 收不到 | 同上：`run_agent` 异步，回复走 `out_message` 事件，不会自动发 IM | 必须配 `agent_out_message_send_message` 规则 |
| 同一消息被处理两次 | 平台重放 + 去重环未覆盖 | 正常情况下后端 FNV-1a 环已去重；若发生在自定义网关，自行加去重 |
| `wechat_send_*` 不回执 | 微信 send 不回退到上下文 | 必须显式传 `chat_id`（room 或 contact id） |
| 飞书目标不对 | `chat_id` 与 `open_id` 混用 | 群用 `chat_id`；单聊用户用 `ou_` 开头 `open_id` |
| QQ 目标不对 | 传了原始 openid | c2c 用 `c2c:<openid>`；群用 `group:<group_openid>` |
| 附件下不下来 | `enable_inbound_attachments=false` 或超 `max_inbound_file_bytes`（默认 2 MB） | 用 `cap_im_<plat>_set_attachment_config` 开启并调上限 |
| 附件下完没后续 | 没有 `attachment_saved` 匹配规则 | 加一条 match `event_type=attachment_saved` 的规则，链 `cap_llm_inspect`/文件操作 |
| 跨渠道乱发 | LLM 没用 `source_cap` 选渠道 | 激活 `cap_im_platform` Skill；遵循「优先当前 source_cap」规则 |
| 给 send 工具传远程 URL | 工具只接本地 path | 先下载到 `/fatfs/...` 再发 |

## 参考项目

- `docs/src/content/docs/en/reference-cap/cap-im-platform.mdx` — 统一 IM 组件、平台差异、Telegram 参考后端、d2 流程图
- `docs/src/content/docs/en/tutorial/web-config.mdx` — Web 配置 IM 凭据
- `application/edge_agent/fatfs_image/system/.recovery/router_rules/router_rules.json` — 真实 7 条 IM 闭环规则（session/llm 命令、working_reply、attachment_saved_reply、route_to_agent、agent_stage_im_notify、agent_out_message_send_message）
- `components/claw_capabilities/cap_im_platform/src/cap_im_tg.c` — 长轮询 + FNV-1a 去重环 + 异步附件队列 + multipart 流式上传
- `components/claw_capabilities/cap_im_platform/src/cap_im_{feishu,qq,wechat}.c` — 其它后端
- `components/claw_capabilities/cap_im_platform/src/cap_im_attachment.c` — 共享附件路径助手
- `components/claw_capabilities/cap_im_platform/include/cap_im_tg.h`（及 feishu/qq/wechat）— `*_set_token` / `*_set_attachment_config` API
- `components/claw_capabilities/cap_im_platform/skills/cap_im_platform/SKILL.md` — 渠道选择规则与 send 工具用法
- `recipes/router_rules.md` — Event Router 规则通用写法与本闭环的基础
- `recipes/configuration.md` 第 5 步 — IM 平台凭据配置
