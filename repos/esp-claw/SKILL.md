---
name: esp-claw-skill
description: >-
  AI Skill for ESP-Claw (Espressif "Chat Coding" edge-agent firmware framework on ESP32).
  Used when users need to build, configure, extend, or debug ESP-Claw firmware and its
  on-device Agent stack — capabilities, skills, Lua modules, event-router automation,
  scheduling, MCP, memory, and board adaptation. Targets the edge_agent application
  built with ESP-IDF v5.5.4 + ESP Board Manager.
  Trigger words: "ESP-Claw", "esp-claw", "Edge Agent", "edge_agent", "Chat Coding",
  "聊天造物", "Agent 框架", "乐鑫 Agent", "ESP32 Agent", "claw_core", "claw_event_router",
  "cap_lua", "claw_skill", "lua_module", "Board Manager", "idf.py bmgr"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-claw-skill

面向乐鑫 **ESP-Claw**（物联网设备「Chat Coding 聊天造物」式 AI Agent 固件框架）的开发技能包。提供场景化 Recipe、真实 API 参考、配置项速查、常见陷阱与执行流程，全部内容均来自 `espressif/esp-claw` 仓库的真实文档与源码。ESP-Claw 用 C 语言在 ESP32 系列芯片上本地完成「感知—决策—执行」闭环：IM 聊天 / 定时 / 事件 触发 Agent Loop，由 `claw_core` 调用云端 LLM、通过 `claw_cap` 执行 Capability 工具、用 Lua 脚本驱动硬件、用 `claw_event_router` 做声明式自动化。

## Core Principles

1. **以 Event 为中心** — ESP-Claw 是事件驱动的。IM 网关、定时器、Lua 脚本、启动钩子、CLI 都会发布 `claw_event_t` 事件进入 `claw_event_router`；规则决定 `call_cap` / `run_agent` / `run_script` / `send_message` / `emit_event` / `drop`。
2. **Core/Capability/Event 三层职责** — `claw_core` 只跑 LLM 循环（组上下文→调模型→执行工具→持久化→回响应）；`claw_cap` 是工具注册与分发层；Capability 既能发事件也能响应动作。不要把业务塞进 core 循环。
3. **Skill 是渐进式上下文，不是常驻提示词** — Skill 通过 `activate_skill` 以「工具返回值」形式注入会话历史，从而保持系统提示词稳定、最大化 LLM API 缓存命中率。不要把 Skill 文档写进系统提示。
4. **路径分层：SYSTEM 只读、DATA 可写** — `CLAW_PATH_SYSTEM`（`/system`）只读，存放固件内置 Skill / Lua 库 / `.recovery` 种子；`CLAW_PATH_DATA`（`/fatfs` 或 SD 卡挂载点）可写，存放 router_rules、scheduler、sessions、memory、用户 Skill。可复用代码里禁止硬编码 `/fatfs`，C 用 `claw_paths_join()`，Lua 用 `storage.get_root_dir()` + `storage.join_path()`。
5. **Capability 通过 Kconfig 启用** — 每个 `cap_*` / `lua_module_*` / `lua_driver_*` 都受 `APP_CLAW_CAP_*` / `APP_CLAW_LUA_*` Kconfig 控制；运行时缺失某工具，多半是相应 Kconfig 未启用或当前芯片 SoC 不支持（如 PCNT/RMT/TOUCH）。
6. **Lua 注册必须在 `cap_lua_register_group` 之前** — `cap_lua_register_module()` 仅在 `cap_lua_group_init` 前有效，之后 `s_module_registration_locked = true`，再注册返回 `ESP_ERR_INVALID_STATE`。所有 `lua_module_*_register()` 都要在 app 初始化里、`cap_lua_register_group("/fatfs/scripts")` 之前调用。
7. **配置以 NVS 为准** — 运行时生效值 = NVS 有值用 NVS，否则用 menuconfig 编译时默认。改了 menuconfig 不一定生效（已被 NVS 覆盖）；恢复出厂需清除对应 NVS 键。`llm_api_key` / `llm_model` / `llm_backend_type` 三者任一为空，`claw_core` 不会启动，`ask` 等核心功能不可用。
8. **out_message / agent_stage 是异步** — `run_agent` 动作异步提交到 `claw_core`，响应以 `out_message` 事件回到路由器；要发到 IM 必须配一条匹配 `source_cap=claw_core, event_type=out_message` 的规则。多轮工具进度用 `agent_stage` 事件（受 `CLAW_CORE_STAGE_VERBOSITY` 控制）。
9. **调度与执行分离** — `cap_scheduler` 只按时发布 `schedule` 事件；具体行为由 Event Router 规则决定。新增定时任务通常两步：`scheduler_add` 一条 + `add_router_rule` 一条（cron/once 在 SNTP 同步后才触发，interval 不依赖外部时间）。
10. **Skill 文档写给 LLM** — 激活后整篇 `SKILL.md` 注入对话；要含场景框架、调用规则、可直接复制的 JSON/Lua 示例、错误剧本。`lua_module_*` Skill 要给完整 API 表（参数/返回/约束），而非散文。
11. **`{CUR_SKILL_DIR}` 而非硬编码路径** — Skill 内引用自带脚本/资源必须用 `{CUR_SKILL_DIR}/scripts/...`，它会展开成设备文件系统中该 Skill 的绝对目录（内置 Skill 在 `/system/skills/<id>`，用户 Skill 在 DATA 根 `skills/<id>`）。文件工具路径必须是绝对路径，禁止 `..`。
12. **构建走 ESP Board Manager** — `idf.py bmgr -c ./boards -b <board>` 自动按板子选芯片 target，不需要 `idf.py set-target`；改板子后重新 `bmgr`。Board 适配放在 `application/edge_agent/boards/<vendor>/<board>/`。

## When to Use

**适用：**
- 为 ESP-Claw `edge_agent` 适配新开发板（`boards/<vendor>/<board>/` 下的 board_info / board_devices / board_peripherals YAML + setup_device.c）
- 新增 / 调试 Capability（`components/claw_capabilities/cap_*`），编写 `claw_cap_descriptor_t` / `claw_cap_group_t` 并在 `app_capabilities.c` 注册
- 新增 / 调试 Lua 模块或驱动（`components/lua_modules/lua_module_*` / `lua_driver_*`），在 `app_lua_modules.c` 注册
- 编写或修改 ESP-Claw Skill（`skills/<id>/SKILL.md` + JSON frontmatter + 可选 `scripts/` `references/` `assets/`）
- 编写 / 调试 Event Router 自动化规则（`router_rules.json`）与定时任务（`schedules.json`）
- 调试记忆系统（`claw_memory` full / lightweight 模式、`memory_records.jsonl`、`user.md`/`soul.md`/`identity.md`）
- 配置 LLM 后端、IM 平台（QQ/飞书/Telegram/微信）、MCP、Web Search
- 排查 Agent 不响应、Skill 不激活、Lua 注册失败、路径越界等运行时问题

**不适用：**
- 通用 ESP-IDF / FreeRTOS / 驱动开发问题（与 ESP-Claw 框架无关）
- 其它 Agent 框架（OpenClaw 原版、MimiClaw 等）—— 仅 ESP-Claw 的 C 实现适用
- 纯云端 LLM 应用（无设备端）—— 本技能聚焦边缘设备固件
- PCB 硬件设计、原理图

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应 Recipe**——它包含完整调用链、分步说明、常见错误与真实代码。

### 项目与构建

| recipe | scenario |
|---|---|
| `recipes/build_and_flash.md` | 从源码构建 edge_agent：选板、menuconfig、build/flash/monitor |
| `recipes/board_adaptation.md` | 适配新开发板：board YAML + setup_device.c + FATFS overlay |
| `recipes/mcp_server.md` | 把设备当 MCP 服务端（`mcp_server_point` 精简应用，仅 `cap_lua` + `cap_mcp_server` + SSE/mDNS） |

### Capability 开发

| recipe | scenario |
|---|---|
| `recipes/implement_capability.md` | 新增一个 `cap_*` 能力组：descriptor / group / execute / 注册 |
| `recipes/lua_module.md` | 新增 Lua 模块（C 绑定 + README.md + test/ + lib/ + Skill） |

### 自动化与编排

| recipe | scenario |
|---|---|
| `recipes/router_rules.md` | 编写 router_rules.json：match / actions / 模板变量 / out_message 回路 |
| `recipes/scheduled_task.md` | 定时任务：scheduler_add + 配套 router rule（cron / interval / once） |
| `recipes/im_integration.md` | IM 端到端闭环：入站 → 路由 → Agent → out_message 回送 / 附件 attachment_saved / 各平台 chat_id 差异 |

### 设备行为与扩展

| recipe | scenario |
|---|---|
| `recipes/write_skill.md` | 编写 ESP-Claw Skill：JSON frontmatter / cap_groups / `{CUR_SKILL_DIR}` / Skill-Lua 脚本 |
| `recipes/automation_lua.md` | 用 Lua 驱动硬件 + 发布事件：gpio/display/storage/event_publisher/capability.call |
| `recipes/memory_usage.md` | 记忆系统：`memory_store/recall/list/update/forget` + profile 三件套（user/soul/identity.md）+ full/lightweight + 自动抽取 |

---

## Capability 与 Lua 模块速查

ESP-Claw 默认对 LLM 可见的能力组（`claw_cap_set_llm_visible_groups`）：`cap_files`、`cap_scheduler`、`cap_lua`、`cap_skill`、`cap_llm_inspect`、`cap_http_request`、`cap_web_search`、`cap_router_mgr`；full 记忆模式再加 `claw_memory`。其余能力（IM、MCP、memory_ops 等）通常需激活对应 Skill 才对模型可见。

### 核心 Capability 工具（节选，真实工具 id）

| 能力组 | 工具 id | 作用 |
|---|---|---|
| `cap_files` | `read_file` / `write_file` / `edit_file` / `delete_file` / `copy_file` / `move_file` / `list_dir` | 受管 FATFS 树（默认根 `/fatfs`）下的文件操作，禁 `..`，读上限 32KB |
| `cap_lua` | `lua_run_script` / `lua_run_script_async` / `lua_list_async_jobs` / `lua_get_async_job` / `lua_stop_async_job` / `lua_stop_all_async_jobs` | 同步/异步跑 Lua，受管根默认 `/fatfs/scripts` |
| `cap_scheduler` | `scheduler_list` / `scheduler_get` / `scheduler_add` / `scheduler_update` / `scheduler_remove` / `scheduler_enable` / `scheduler_disable` / `scheduler_pause` / `scheduler_resume` / `scheduler_trigger_now` / `scheduler_reload` | 时间触发，发布 `schedule` 事件 |
| `cap_router_mgr` | `list_router_rules` / `get_router_rule` / `add_router_rule` / `update_router_rule` / `delete_router_rule` / `reload_router_rules` | 管理 router_rules.json |
| `cap_skill` | `list_skill` / `register_skill` / `unregister_skill` / `activate_skill` | Skill 管理；`activate_skill` 返回完整文档并打开 `cap_groups` |
| `cap_system` | `get_system_info` / `get_current_time` / `restart_device` | 系统信息/时间/重启 |
| `cap_mcp_client` | `mcp_list_tools` / `mcp_call_tool` / `mcp_discover` | 远程 MCP 工具发现与调用 |
| `cap_im_tg/qq/feishu/wechat` | `<platform>_gateway` / `<platform>_send_message` / `<platform>_send_image` / `<platform>_send_file` | IM 事件源 + 发送工具（wechat 不支持 send_file） |

### 内置 Lua 模块（`lua_driver_*` / `lua_module_*`，真实 require 名）

`adc` `gpio` `i2c` `ssd1306` `lib_si12t_touch` `mcpwm` `pcnt` `touch` `uart` · `audio` `ble_hid` `board_manager` `button` `capability` `camera` `delay` `dht` `display` `environmental_sensor` `event_publisher` `http_server` `image` `json` `lib_fuel_gauge` `imu` `ir` `knob` `lcd` `lcd_touch` `led_strip` `lvgl` `magnetometer` `motion_detect` `sci` `storage` `system` `thread` `arg_schema`

> 每个模块受 `APP_CLAW_LUA_DRIVER_*` / `APP_CLAW_LUA_MODULE_*` Kconfig 控制；芯片 SoC 不支持时默认关闭（如无 PCNT/RMT/TOUCH）。

---

## Runtime FATFS 布局（DATA 根，默认 `/fatfs`）

```
fatfs/
├── sessions/                 # 会话历史（按 session_id 隔离）
├── memory/
│   ├── MEMORY.md             # 可读记忆视图（full 模式只读，非检索源）
│   ├── memory_records.jsonl  # 结构化记忆条目
│   ├── memory_index.json     # summary-tag / 关键词索引
│   ├── memory_digest.log     # 记忆操作摘要
│   ├── user.md / soul.md / identity.md   # 可编辑 profile
├── skills/<skill_id>/SKILL.md   # 用户/运行时 Skill
├── scripts/                  # cap_lua 受管根
├── router_rules/router_rules.json
├── scheduler/schedules.json (.state)
└── inbox/                    # IM 附件落地
```

> `/system`（只读）下有 `/system/skills/`（固件内置 Skill）、`/system/scripts/builtin/lib`（内置 Lua 库）、`/system/.recovery/`（缺失时回拷到 DATA 的种子）。

---

## Event Router 规则匹配状态机

```
事件入队 → 是 claw_event_router 自身发出的？ ─是→ 跳过(防自环)
                                  └否→ 构建 ctx(事件/规则/变量/last)
       → 还有规则？ ─是→ 规则启用 && 所有 match 字段命中？
                          ├命中→ 顺序执行 actions → consume_on_match=true? ─是→ 停止(消费)
                          └未命中→ 继续下一条
       → 规则走完且无匹配 → 默认路由到 Agent? && event_type="message"? ─是→ run_agent
                                                              └否→ 结束
```

匹配模板变量：`{{match.text}}` `{{match.rule}}` `{{match.remainder}}`（`text_match_rule:"prefix"` 时为命令后剩余文本）；事件字段：`{{event.source_channel}}` `{{event.chat_id}}` `{{event.text}}` `{{event.event_type}}`。

---

## Critical Pitfalls (Must Read)

下列是最高频的错误，违反任何一条都会导致固件/Agent 行为异常。

### 1. Lua 模块注册顺序错

```c
// ❌ WRONG — 在 cap_lua_register_group 之后才注册，返回 ESP_ERR_INVALID_STATE
cap_lua_register_group("/fatfs/scripts");
lua_module_display_register();

// ✅ CORRECT — 所有模块在 register_group 之前注册
lua_module_display_register();
lua_driver_gpio_register();
lua_module_delay_register();
cap_lua_register_group("/fatfs/scripts");  // 之后锁定
```

### 2. 硬编码 `/fatfs` 写路径

```lua
-- ❌ WRONG — 假设可写根一定是 /fatfs（SD 卡场景会错）
local f = io.open("/fatfs/logs/today.txt", "w")

-- ✅ CORRECT — 通过 storage 模块解析可写根
local storage = require("storage")
local root = storage.get_root_dir()
local path = storage.join_path(root, "logs", "today.txt")
storage.mkdir(storage.join_path(root, "logs"))
storage.write_file(path, "hello")
```

### 3. Skill 内用相对路径或硬编码 Skill 路径

```json
// ❌ WRONG — 写死 /fatfs/skills/<id>，换板子/换存储就失效
{"path":"/fatfs/skills/light_switch/scripts/x.lua"}

// ✅ CORRECT — 用 {CUR_SKILL_DIR} 占位符（仅 SKILL.md 正文有效，frontmatter 无效）
{"path":"{CUR_SKILL_DIR}/scripts/x.lua"}
```

### 4. router rule 的 run_script / send_message 参数放错层级

```json
// ❌ WRONG — path 放在 action 顶层
{"type":"run_script","path":"demo.lua"}

// ✅ CORRECT — 放进 actions[*].input
{"type":"run_script","input":{"path":"demo.lua","args":{"k":"v"}}}
```

### 5. 期望 run_agent 同步返回，却没配 out_message 规则

```json
// ❌ WRONG — run_agent 异步，Agent 回复不会自动发到 IM
{"id":"route_to_agent","match":{"event_type":"message"},"actions":[{"type":"run_agent"}]}

// ✅ CORRECT — 另加一条匹配 out_message 的规则回送 IM
{
  "id":"agent_out_message_send_message",
  "consume_on_match": true,
  "match":{"source_cap":"claw_core","event_type":"out_message","content_type":"text"},
  "actions":[{"type":"send_message","input":{
      "channel":"{{event.source_channel}}",
      "chat_id":"{{event.chat_id}}",
      "message":"{{event.text}}"}}]
}
```

### 6. 改 menuconfig 后以为一定生效

```text
# ❌ WRONG — llm_api_key 等运行时字段已被 NVS 覆盖，改 menuconfig 无效
(Top) → App Claw Config 改默认值后直接 flash

# ✅ CORRECT — 运行时值 = NVS 有则用 NVS，否则用编译默认
# 要恢复出厂：清除对应 NVS 键，或通过 Web 配置重写后保存
```

### 7. claw_core 不启动却调 ask

```text
# ❌ WRONG — llm_api_key / llm_model / llm_backend_type 任一为空，core 不启动
ask 你好          # 失败/无响应

# ✅ CORRECT — 三项都配齐后重启设备；core 启动日志可见
# （路由/自动化/本地能力/Console REPL 仍可用，仅 ask/默认 Agent 路由/图像检查不可用）
```

### 8. 把 Skill 文档写进系统提示词

```text
# ❌ WRONG — 自行维护「已激活 Skill 文档」注入系统提示，每次激活/取消都打破 LLM 缓存前缀

# ✅ CORRECT — 依赖 activate_skill 把文档作为工具返回值注入会话历史；
# 系统提示只放稳定的 Skill 目录（id + summary），命中率高
```

### 9. 定时任务只加 scheduler 不加 router rule

```json
// ❌ WRONG — scheduler 只发 schedule 事件，没人接，什么都不会发生
{"id":"remind","kind":"cron","cron_expr":"0 7 * * *","event_type":"schedule","event_key":"reminder"}

// ✅ CORRECT — 配套一条 router rule
// schedules.json: 上面那条
// router_rules.json:
{"id":"handle_reminder","match":{"event_type":"schedule","event_key":"reminder"},
 "actions":[{"type":"send_message","input":{"channel":"telegram","chat_id":"123","message":"{{event.text}}"}}]}
```

### 10. event_publisher 用冒号调用或残缺 table

```lua
-- ❌ WRONG — 冒号语法会把 self 当第一个参数；残缺 table 缺 source_cap 会报错
ep:publish_message("hi")
ep.publish_message({ channel = "qq", text = "hi" })  -- 缺 source_cap

-- ✅ CORRECT — 点号调用；回调里优先用字符串形式
local ep = require("event_publisher")
ep.publish_message("Button pressed!")                 -- 最简形式
-- 或完整 table：
ep.publish_message({ source_cap="lua_script", channel=args.channel, chat_id=args.chat_id, text="hi" })
```

### 11. edit_file 期望全局替换

```text
# ❌ WRONG — 以为 edit_file 替换全部匹配
edit_file 把所有 "foo" 改成 "bar"

# ✅ CORRECT — edit_file 只替换第一处（strstr + 单次回写）
# 要批量替换：读取 → 在 Lua/LLM 侧处理 → write_file 整体覆盖
```

### 12. 在 cap_lua register_group 之后想动态加模块

```c
// ❌ WRONG — 运行时 cap_lua_register_module 返回 ESP_ERR_INVALID_STATE
// （模块集在启动时冻结）

// ✅ CORRECT — 模块在编译期决定；运行期扩展用 Lua 脚本 + capability.call 调已有 cap_*
```

### 13. SKILL.md frontmatter 不是合法 JSON 或 name 与目录不符

```text
# ❌ WRONG — 用 YAML 风格 / name 与目录名不一致 / 缺 H1
---
name: lightSwitch      # 目录是 light_switch → 构建失败
---

# ✅ CORRECT — JSON 对象，name == skills/<id>/ 目录名，恰好一个 H1
---
{
  "name": "light_switch",
  "description": "Turn board light on/off, set LED strip color/brightness. Requires board_hardware_info skill.",
  "metadata": { "cap_groups": ["cap_lua"], "manage_mode": "readonly" }
}
---
# Light Switch
```

### 14. 忘了 Board Manager 选板，直接 set-target

```bash
# ❌ WRONG — 跳过 bmgr，芯片 target / 外设配置不对
idf.py set-target esp32s3

# ✅ CORRECT — bmgr 按板子自动选 target
idf.py bmgr -c ./boards -b esp32_S3_DevKitC_1
idf.py build
```

### 15. 长 Lua 循环不让出 CPU

```lua
-- ❌ WRONG — 死循环忙等，触发任务看门狗
while true do
  if gpio.get_level(0) == 0 then break end
end

-- ✅ CORRECT — 轮询循环里显式 delay 让出 CPU（runtime 有指令钩子+墙钟超时保护，但仍建议协作式 yield）
local delay = require("delay")
while true do
  if gpio.get_level(0) == 0 then break end
  delay.delay_ms(20)
end
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 弄清目标：是改固件（Capability/Lua 模块/板子）还是改运行时（Skill/规则/调度/记忆）？是否需联网 LLM？ |
| 2 | Recipe | 匹配 `recipes/` 场景，按其调用链走；没有匹配再查 `resources/` |
| 3 | Query | API/配置不在 Recipe 覆盖范围时，查 `resources/api_reference.md`、`config_reference.md`、`example_list.md` |
| 4 | Validate | 核对所有：函数签名、头文件、Kconfig 符号、frontmatter、路径分层、注册顺序 |
| 5 | Confirm | 向用户给出实现计划：改哪些文件、注册点、menuconfig 项、router rule、Skill 结构 |
| 6 | Execute | 改源码或改设备上 FATFS 文件；固件改动需 `idf.py build && flash monitor`；运行时改动（规则/脚本/Skill）改文件后 `auto reload` |
| 7 | Check | 验证：注册顺序、路径用 `{CUR_SKILL_DIR}`/`storage`、out_message 回路、Kconfig 启用、core 启动条件 |
| 8 | Run | `idf.py monitor` 看启动日志；Console `cap list` / `auto rules` / `scheduler --list` 验证；IM 端发消息测端到端 |
| 9 | Debug | Console REPL（`ask` / `cap call` / `auto last`）+ 日志（`claw_core`/`claw_event_router` tag）；FAQ 见 docs tutorial/faq |

### Step 6 Detail — 改动落点策略

**改固件（C 代码）：**
- 新 Capability → `components/claw_capabilities/cap_<name>/` + 在 `components/common/app_claw/app_capabilities.c` 注册 + Kconfig
- 新 Lua 模块 → `components/lua_modules/lua_module_<name>/` + 在 `components/common/app_claw/app_lua_modules.c` 注册 + Kconfig
- 新板子 → `application/edge_agent/boards/<vendor>/<board>/`（`board_info.yaml`/`board_devices.yaml`/`board_peripherals.yaml`/`setup_device.c`/`sdkconfig.defaults.board`，可选 `fatfs_image/` overlay）

**改运行时（设备上 FATFS，无需重编）：**
- 自动化规则 → `/fatfs/router_rules/router_rules.json`（Console `auto add_rule` 或 Web 文件管理器，改完 `auto reload`）
- 定时任务 → `scheduler --add --json '<json>'` 或 `schedules.json`，配套加 router rule
- 用户 Skill → `/fatfs/skills/<id>/SKILL.md`（或 `register_skill` / Skills Lab 下载）
- Lua 脚本 → `/fatfs/scripts/*.lua`（`write_file` 写，`lua_run_script` 跑）
- 记忆 profile → `/fatfs/memory/{user,soul,identity}.md`

---

## Failure Strategies

| Situation | Action |
|---|---|
| API / 工具 id 在 resources 里查不到 | 停下，告知用户该工具不存在；勿臆造 |
| `ask` 无响应 | 检查 `llm_api_key`/`llm_model`/`llm_backend_type` 是否齐；core 是否启动（日志 WARN） |
| IM 收消息无回复 | 查 `auto last` 是否匹配；是否有 `out_message` → `send_message` 规则；IM 凭据是否配 |
| Skill 激活但工具不可见 | 确认 `metadata.cap_groups` 写对 group id；激活是按 `session_id` 隔离，`--session` 要对齐 |
| Lua 模块 `require` 失败 | 对应 `APP_CLAW_LUA_*` Kconfig 是否启用；是否在 `register_group` 前注册 |
| 定时任务不触发 | cron/once 需 SNTP 同步成功；interval 不依赖外部时间；查 `schedules.json.state` |
| 路径越界报错 | 文件工具禁 `..`；Skill 内用 `{CUR_SKILL_DIR}`；可写路径用 `storage.get_root_dir()` |
| 改 menuconfig 不生效 | 该字段可能被 NVS 覆盖；清除 NVS 键或用 Web 重写保存 |

## References

- 场景 Recipe → `recipes/` 目录
- Capability / Lua / 核心 API 速查 → `resources/api_reference.md`
- Kconfig / 配置项速查 → `resources/config_reference.md`
- 仓库内真实示例与板子索引 → `resources/example_list.md`
- 汇总陷阱 → `resources/pitfalls.md`
- ESP-Claw 在线文档：https://esp-claw.com/en/tutorial/
- 仓库：https://github.com/espressif/esp-claw
