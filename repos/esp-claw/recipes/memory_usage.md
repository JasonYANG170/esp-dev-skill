# 记忆系统使用（长期记忆 + profile）

> **适用摘要**: 用 ESP-Claw 的「长期记忆」工具（`memory_store` / `recall` / `list` / `update` / `forget`）和可编辑 profile 三件套（`user.md` / `soul.md` / `identity.md`）让 Agent 跨会话记住事实与人格；理解 full / lightweight 两种模式、summary-tag 轻量检索（无向量库）、自动抽取/去重，以及「`MEMORY.md` 不是检索真相」这一关键陷阱。

## 触发意图

- "记住 / 别忘了 / 保存这条事实"
- "你记得我什么 / 还记得吗"
- "更新 / 删除一条记忆"
- "改 Agent 性格 / 人设 / 身份"
- "改用户偏好 / 默认行为"
- "full vs lightweight 模式"
- "MEMORY.md 改了不生效"

## 前置条件

| 条件 | 要求 |
|---|---|
| 记忆能力 | `CONFIG_APP_CLAW_CAP_MEMORY=y`（`edge_agent` 默认开；`mcp_server_point` 默认关） |
| 记忆模式 | `(Top) → App Claw Config → App Claw memory management mode`：`Structured memory management`（full，默认）/ `Lightweight memory` |
| 记忆目录 | `CLAW_PATH_DATA` 下 `/memory/`（`edge_agent` 即 `/fatfs/memory/`） |
| 关键文件 | `memory_records.jsonl`（结构化条目）、`memory_index.json`（summary-tag/关键词索引）、`MEMORY.md`（人类可读视图，full 模式只读）、`memory_digest.log`（操作摘要）、`user.md` / `soul.md` / `identity.md`（可编辑 profile） |
| 内置 Skill | `memory_ops`（cap_groups: `claw_memory`，管 `memory_*` 工具）、`profile_memory_ops`（cap_groups: `cap_files`，管 profile 三件套） |
| 参考文档 | `docs/.../reference-project/memory.mdx`、`reference-core/claw-memory.mdx` |

## 分步说明

### 1. 两种记忆的分工（关键概念）

| 记忆类型 | 作用域 | 解决什么 | 实现位置 |
|---|---|---|---|
| Session history | 单个 Session ID 内 | 同一会话的轮次连贯（线程隔离，不同 IM 用户/频道各一线程） | `/fatfs/sessions/<session_id>/` |
| Long-term memory | **跨所有 Session 共享** | 跨会话的偏好、设备摘要、跨任务约定 | `/fatfs/memory/` |

Session history 由 `claw_memory_session_history_provider` 注入；超限的旧轮次**先丢**，把持久事实挤进长期记忆。Long-term memory 由 `claw_memory_long_term_provider`（full）/ `_lightweight_provider`（lightweight）注入。

### 2. full vs lightweight 模式

| 模式 | 行为 | LLM 可见组 | 注入内容 |
|---|---|---|---|
| Structured（full，默认） | 结构化记录 + summary-tag 目录，按需 `memory_recall` | 多加 `claw_memory` | **仅**注入 summary-tag 目录，不注入整篇 `MEMORY.md` |
| Lightweight | 跳过结构化抽取 | **不**加 `claw_memory` | 直接注入 `MEMORY.md` 文本 |

> full 模式下 `MEMORY.md` **不是**检索真相，检索/更新基于 `memory_records.jsonl` + `memory_index.json`。把它当真相源会让记忆「不记得」。

### 3. 长期记忆五件套工具（真实 id + 真实 schema）

来自 `components/claw_modules/claw_memory/src/claw_memory_cap.c` 的 `s_memory_descriptors[]`（`cap_groups: claw_memory`，仅 full 模式对 LLM 可见）：

| 工具 id | 必填 | 入参 schema（真实 `input_schema_json`） | 用途 |
|---|---|---|---|
| `memory_store` | `content` | `{memory_id?, content, tags?, keywords?}` | 写一条长期记忆（建议写归一化事实，不要写原话） |
| `memory_recall` | `summary_labels`（来自注入目录的精确标签） | `{summary_labels: string[], limit?}` | 按标签取详情；**不能**塞自然语言问题 |
| `memory_list` | — | `{}` | 列出全部已存记忆 |
| `memory_update` | `memory_id` | `{memory_id, content?, tags?, keywords?}` | 改一条（先 recall/list 拿到 id） |
| `memory_forget` | `memory_id` | `{memory_id}` | 删一条 |

每条记忆条目（`claw_memory_item_t` 真实字段）：`id[40]` `source[16]`（如 `manual` / `auto_llm`）`content[256]` `tags[96]`（逗号分隔，建议 1-3 个）`keywords[128]` `created_at`/`updated_at` `access_count` `deleted`。

### 4. 显式存一条记忆

当用户**明确**说「记住/保存/别忘了」时才显式调 `memory_store`（内置 `memory_ops` Skill 的 Hard Rule 1/17）：

```json
// 用户：「记住我是嵌入式工程师」
// memory_store
{
  "content": "The user's profession is embedded systems engineer.",
  "tags": "profession",
  "keywords": "engineer,embedded,profession"
}
```

- `content` 写**归一化事实**（英文陈述句最佳），不要塞原话；不要超 256 字符
- `tags` 用**稳定可复用**的主题（`daily_routine` / `dietary_preferences` / `commute` / `hydration_habits`），不要用一次性细节词（`nap` / `8_glasses_of_water`）
- 自我介绍、随口偏好**不要**显式存——交给自动抽取在回复后静默处理

### 5. 召回（recall 的两段式）

ESP-Claw **不**用向量库。它用「summary labels」做轻量检索：每次请求把当前记忆的标签目录注入系统上下文，LLM 先挑标签、再用 `memory_recall` 拿详情：

```json
// 用户：「我每天几点起床？」
// 假设注入的目录里有 daily_routine 标签
// memory_recall
{
  "summary_labels": ["daily_routine"],
  "limit": 5
}
```

`summary_labels` 必须是目录里的**精确**值；传非白名单值会报 `invalid_summary_labels` 并把当前目录回传给你（见 `cap_memory_recall_execute` 的校验）。不要在 `summary_labels` 里塞问题文本。

### 6. 列出 / 更新 / 删除

```json
// 列出全部（用户问「你记得我什么」前先看）
// memory_list  →  {}
```

```json
// 更新：先 list/recall 拿到 memory_id
// memory_update
{ "memory_id": "01J...", "content": "The user's profession is senior embedded engineer." }
```

```json
// 删除：同样先拿 memory_id
// memory_forget
{ "memory_id": "01J..." }
```

> 不要在没拿到 `memory_id` 前 update/forget；不要对 `memory_records.jsonl`/`memory_index.json`/`MEMORY.md` 直接 read/write（Hard Rule 15）。

### 7. 自动抽取与去重（full 模式）

full 模式在每次 `claw_core` 请求开始时，从最近用户消息里尝试抽取持久事实，写入结构化记忆并做写入前后的去重/替换（`claw_memory_extract.c` + `on_request_start` 回调，`enable_async_extract_stage_note=true`）。所以**普通偏好不要主动 `memory_store`**——让自动抽取静默处理，主回复保持自然（Hard Rule 17/20）。

### 8. 可编辑 profile 三件套

由 `profile_memory_ops` Skill 管理（cap_groups: `cap_files`，经 `read_file` / `edit_file` 改）：

| 文件 | 用途 | 建议内容 |
|---|---|---|
| `memory/user.md` | 用户画像 | 称呼、偏好语言、常用术语、长期约定 |
| `memory/soul.md` | Agent 灵魂与价值观 | 核心风格、操作原则、优先级 |
| `memory/identity.md` | Agent 身份卡 | 名字、角色、能力边界、强项 |

由 `claw_memory_profile_provider` 作为系统上下文注入。改人格流程（`profile_memory_ops` 典型流）：

```text
1. 先 read_file memory/soul.md
2. edit_file 做精确行级改（edit_file 只替换首处，要批量替换用 read+write 整体覆盖）
3. 保持简洁、结构化、无冗余
4. 保存后立即用新人设回复
```

profile 文件是**持久提示文档**，不是普通记忆条目——只放**显式、持久**的人设/身份/风格/默认用户画像变更；普通事实走长期记忆。

> 改 profile 用绝对路径 `/fatfs/memory/soul.md`（DATA 根）或经 `storage.get_root_dir()` 拼接，**不要**硬编码别的根。

### 9. 调试与排查

- `memory_list` 看 `memory_records.jsonl` 实际内容
- 看 `memory_index.json` 确认 summary-tag 目录是否如期
- `memory_digest.log` 是操作摘要日志
- 用 Console：`cap call memory_list '{}'`、`cap call memory_recall '{"summary_labels":["daily_routine"]}'`

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| full 模式「不记得」 | 把 `MEMORY.md` 当检索真相 | 检索看 `memory_records.jsonl`/`memory_index.json`；`MEMORY.md` 是只读同步视图 |
| `memory_recall` 报 invalid_summary_labels | 传了非白名单/自然语言标签 | 用回传的目录里的精确值；目录在每次请求的系统上下文里 |
| `memory_store` 写原话 | content 写了用户原句 | 写归一化事实陈述句（如 `The user ...`） |
| 标签越写越散 | 用一次性细节词当标签 | 用稳定主题（`daily_routine`、`dietary_preferences`） |
| 改 profile 后下次不生效 | 用了 `write_file` 覆盖破坏结构 / 改了错文件 | profile 改 `/fatfs/memory/{user,soul,identity}.md`；`profile_memory_ops` 要求先 read 再 edit |
| 把「随口喜欢苹果」也 `memory_store` | 没让自动抽取干活 | 非显式「记住」请求交给自动抽取（Hard Rule 17） |
| lightweight 模式找不到 `memory_*` 工具 | lightweight 不注册 `claw_memory` 组 | lightweight 只注入 `MEMORY.md`；要工具改 full 模式重编 |
| `update`/`forget` 报 memory_id required | 没 recall/list 拿 id | 先 `memory_recall` 或 `memory_list` 取 `memory_id` |
| 直接 read/write `memory_records.jsonl` | 违反 Hard Rule 15 | 一切走 `memory_*` 工具，别碰底层文件 |

## 参考项目

- `docs/src/content/docs/en/reference-project/memory.mdx` — 两类记忆、full/lightweight、summary-tag 检索、自动抽取/去重
- `docs/src/content/docs/en/reference-core/claw-memory.mdx` — `claw_memory_config_t` 字段、Context Providers、文件布局
- `components/claw_modules/claw_memory/include/claw_memory.h` — `claw_memory_item_t` / `claw_memory_query_t` / C API
- `components/claw_modules/claw_memory/src/claw_memory_cap.c` — `s_memory_descriptors[]` 工具 schema 与 execute 实现
- `components/claw_modules/claw_memory/skills/memory_ops/SKILL.md` — `memory_*` 工具用法与硬/软规则
- `components/claw_modules/claw_memory/skills/profile_memory_ops/SKILL.md` — profile 三件套用法
- `application/edge_agent/fatfs_image/system/.recovery/memory/{user,soul,identity}.md` — profile 种子文件
- `recipes/configuration.md` 第 6 步 — 记忆模式编译期切换
