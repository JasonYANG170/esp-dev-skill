# AGENTS.md — Supplementary Agent Guide

> 核心原则、Recipe 索引、陷阱、执行流程都在 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未覆盖的工程约定与工具指引，不重复内容。

## Project Context

- **Language**: C（ESP-IDF C-style OO：`typedef struct xxx_t *xxx_handle_t`，opaque handle）+ Lua（设备端可编程脚本）+ 少量前端（TS/React，`http_server/frontend_source`）
- **Target**: ESP32-S3 / ESP32-P4 / ESP32-C5 / ESP32-S31，多板由 ESP Board Manager 管理
- **Toolchain / Build**: ESP-IDF **v5.5.4** + `esp-bmgr-assist`（`pip install esp-bmgr-assist`）+ Board Manager
- **App entry**: `application/edge_agent/main/main.c`（`app_main`）
- **Agent wiring**: `components/common/app_claw/app_claw.c`（`app_claw_start`）

## File Naming & Layout

### 顶层结构
```
esp-claw/
├── application/edge_agent/        # 主应用（main / boards / components / fatfs_image / partitions_*.csv）
├── components/
│   ├── claw_modules/              # 框架核心：claw_core / claw_cap / claw_event_router / claw_memory / claw_skill / claw_paths
│   ├── claw_capabilities/         # cap_* 能力（cap_lua / cap_files / cap_scheduler / cap_im_platform / cap_mcp_* / ...）
│   ├── lua_modules/               # lua_driver_* / lua_module_*（每个含 README.md + 可选 src/ test/ lib/ skills/）
│   └── common/                    # app_claw（应用层装配：注册能力与 Lua 模块）、skill_builder
├── docs/                          # Astro + Starlight 文档站（docs/src/content/docs/{en,zh-cn}/...）
└── .agents/                       # design.md / docs.md / gotchas.md / spec/{claw-skill-spec,lua-module-spec}.md
```

### 能力 / Lua 模块命名
- Capability 组件：`components/claw_capabilities/cap_<name>/`（`include/cap_<name>.h`、`src/cap_<name>.c`、可选 `skills/cap_<name>/SKILL.md`）
- Lua 驱动：`components/lua_modules/lua_driver_<name>/`；Lua 模块：`components/lua_modules/lua_module_<name>/`
- 每个 Lua 组件目录名必须 `lua_module_xx` / `lua_driver_xx`；`README.md` 是 agent 面向的 API 文档（构建期同步进 docs）
- Skill 目录：`skills/<skill_id>/SKILL.md`（`<skill_id>` 全局唯一，且 `name` 字段必须等于目录名）

### Board 适配
```
application/edge_agent/boards/<vendor>/<board>/
├── board_info.yaml            # 板子元信息（芯片、名称）
├── board_devices.yaml         # 设备清单（占用 IO）
├── board_peripherals.yaml     # 外设总线配置
├── sdkconfig.defaults.board   # 板级默认 sdkconfig
├── setup_device.c             # 板级初始化（被 board_manager 调用）
├── components/                # 可选板级本地组件
└── fatfs_image/               # 可选，构建期 overlay 到 SYSTEM 镜像（不进 DATA）
```

## Include / Import 约定

### C（新增 Capability）
```c
#include "cap_<name>.h"        // 仅暴露 esp_err_t cap_<name>_register_group(void);
#include "claw_cap.h"          // claw_cap_descriptor_t / claw_cap_group_t / claw_cap_register_group
#include "claw_event_router.h" // claw_event_router_publish / publish_message（事件源）
#include "cJSON.h"             // 解析 input_json
```

### C（应用层装配）
```c
// components/common/app_claw/app_capabilities.c —— 注册各 cap_*_register_group()
// components/common/app_claw/app_lua_modules.c   —— 注册各 lua_module_*_register() / lua_driver_*_register()
// 注意：Lua 注册必须在 cap_lua_register_group() 之前
```

### Lua（设备脚本）
```lua
local gpio   = require("gpio")          -- lua_driver_gpio
local delay  = require("delay")         -- lua_module_delay
local storage= require("storage")       -- 解析可写根，禁止硬编码 /fatfs
local json   = require("json")
local capability = require("capability")-- Lua 侧调 cap_* 工具
local ep     = require("event_publisher")
local arg_schema = require("arg_schema")-- 参数归一化（见 lua_module_system）
```

## Canonical Init / Boot 模式

`edge_agent` 启动顺序（`main.c::app_main` → `app_claw.c::app_claw_start`）要点：

1. `nvs_flash_init` → `app_config_init` / `app_config_load`（Wi-Fi/LLM/IM/时区等）
2. `esp_board_manager_init`（装配板级设备句柄）→ 可选 `app_claw_ui_start`（emote，受 `CONFIG_APP_CLAW_ENABLE_EMOTE`）
3. `app_fs_init` 挂载 FATFS → `claw_paths_set(CLAW_PATH_DATA/SYSTEM, ...)`（之后所有路径用 `claw_paths`）
4. `wifi_manager_init/start` + `http_server_init/start` + `captive_dns_start`
5. `app_claw_start(config)` 内部依次：
   - `cap_session_mgr_set_session_root_dir` → `claw_event_router_init`（加载 router_rules.json）
   - `cap_scheduler_init`（不立即 start）
   - `claw_memory_init`（sessions / memory 根）
   - `claw_skill_init`（skills 根）
   - `claw_cap_init` + 各 `cap_*_register_group` → `claw_cap_set_llm_visible_groups` → `claw_cap_start_all`
   - `claw_event_router_register_outbound_binding`（qq/feishu/telegram/wechat/web → 各 send 工具）
   - **仅当** `llm_api_key`/`llm_model`/`llm_backend_type` 都非空：`claw_core_init` + 注册 context provider（profile / long_term / session_history / skills_list / tools）+ `claw_core_start`
   - `claw_event_router_start` → `cap_scheduler_start` → `cap_system_time_sync_service_start`（SNTP，首次同步 `cap_scheduler_handle_time_sync` 重基准）→ `app_claw_cli_start`（Console REPL）→ 发布 `startup`/`boot_completed` 事件

### 标准 Capability 注册模式
```c
esp_err_t cap_my_feature_register_group(void)
{
    if (claw_cap_group_exists(s_my_group.group_id)) return ESP_OK;  // 幂等
    return claw_cap_register_group(&s_my_group);
}
```

### execute 签名（固定）
```c
static esp_err_t my_execute(const char *input_json,
                            const claw_cap_call_context_t *ctx,  // 带 session_id/chat_id/source_channel/caller
                            char *output, size_t output_size);   // output 通常 4–8KB，勿溢出
```
- 成功 `ESP_OK`；人/模型可读文本写进 `output`；错误前缀 `"Error: ..."`
- 在 `execute` 返回前释放临时分配；大载荷分块/落盘/返回路径让上层再读

## Build Workflow

```bash
# 0. 导出 ESP-IDF v5.5.4 环境
. $IDF_PATH/export.sh
pip install esp-bmgr-assist          # 一次性

# 1. 取源码并进应用目录
git clone https://github.com/espressif/esp-claw.git
cd esp-claw/application/edge_agent

# 2. 选板（自动选芯片 target，无需 set-target）
idf.py bmgr -c ./boards -l                # 列出支持的板子
idf.py bmgr -c ./boards -b esp32_S3_DevKitC_1

# 3. 可选 menuconfig：(Top)→App Claw Config、Component config→ESP-Claw Core→stage verbosity
idf.py menuconfig

# 4. 构建烧录
idf.py build
idf.py flash monitor
```

文档站（改 docs 时）：`cd docs && pnpm install && pnpm build`（或 `pnpm dev`）。
Web 配置前端（改 http_server 前端时）：`cd application/edge_agent/components/http_server/frontend_source && pnpm build && pnpm typecheck`。

## 代码生成 Checklist（新增 Capability / Lua 模块）

- [ ] 组件目录命名规范（`cap_<name>` / `lua_module_<name>` / `lua_driver_<name>`）
- [ ] `CMakeLists.txt` 用 `idf_component_register(...)`，C 模块列出 SRCS/INCLUDE_DIRS/REQUIRES（至少 `claw_cap cJSON`）
- [ ] 公开头文件只暴露 `*_register_group()` / `*_register()`，struct 私有于 `.c`
- [ ] `execute` 用 `cJSON_Parse` 校验入参，错误写进 `output` 前缀 `Error:`，返回非 `ESP_OK`
- [ ] `claw_cap_descriptor_t` 的 `id` 全局唯一；`input_schema_json` 是合法 JSON Schema
- [ ] 在 `app_capabilities.c`（或 `app_lua_modules.c`）注册；Lua 注册先于 `cap_lua_register_group`
- [ ] 新增 Kconfig 项放进 `components/common/app_claw/Kconfig`（`APP_CLAW_CAP_*` / `APP_CLAW_LUA_*`），默认值与 SoC 依赖写对（如 `default y if SOC_RMT_SUPPORTED`）
- [ ] 配套 Skill 文档放 `skills/<id>/SKILL.md`（JSON frontmatter，`metadata.cap_groups` / `manage_mode`/ `category`）
- [ ] Lua 模块有 `README.md`（API 表 + 可运行示例）；`lib/*.lua` 必须有同名 `lib/*.md`
- [ ] 不硬编码 `/fatfs`：C 用 `claw_paths_*`，Lua 用 `storage.*`
- [ ] `idf.py build` 通过；改了前端跑 `pnpm build && pnpm typecheck`

## Runtime Path Rules（必读）

- `CLAW_PATH_SYSTEM`（`/system`）只读：内置 Skill、内置 Lua 库、`.recovery` 种子、板子 `fatfs_image/` overlay
- `CLAW_PATH_DATA`（`/fatfs` 或 SD 卡挂载点）可写：router_rules、scheduler、sessions、memory、用户 Skill、scripts、inbox
- C：`claw_paths_join(CLAW_PATH_DATA, ...)`；Lua：`storage.get_root_dir()` + `storage.join_path(...)`
- Skill 内引用自带文件用 `{CUR_SKILL_DIR}/...`（仅正文，frontmatter 不展开）
- 文件工具（`read_file` 等）路径必须绝对、禁 `..`、禁反斜杠

## Console / 调试入口（真实命令）

```sh
# 会话与问答
session [demo]            # 切换/查看会话
ask <text>                # 多轮（写历史）
ask_once <text>           # 单轮（不写历史）

# 能力
cap list
cap call <id> '<json>'    # 例：cap call get_current_time '{}'
cap groups / cap enable|disable|unload <group_id>
cap load qq               # 动态注册 cap_im_qq

# 自动化（auto 来自 edge_agent；event_router 来自 cap_router_mgr，等价）
auto reload / auto rules / auto rule <id>
auto add_rule|update_rule|delete_rule '<json>'
auto emit_message <source_cap> <channel> <chat_id> <text...>
auto emit_trigger <source_cap> <event_type> <event_key> '<payload_json>'
auto last                 # 上次匹配统计

# 调度
scheduler --list|--reload|--add --json '<json>'|--enable|--disable|--pause|--resume|--trigger <id>

# 其它已注册：help / time / web_search / mcp_client / mcp_server / llm_inspect / event_router
```
> `skill` 串口命令默认未注册（`edge_agent` 未调 `register_cap_skill()`）；用 `cap call activate_skill '{"skill_id":"xxx"}'` 替代，或在 `app_claw_cli.c` 里启用后重编。

## Do Not Modify

- 仓库源码 `components/`、`application/edge_agent/` 内的现有文件 —— 如需改行为走 PR，不要在本地静默改框架
- `resources/` —— 本 Skill 的 API 文档源，勿改
- `SKILL.md` frontmatter —— Skill 元数据

## Specs & 进一步阅读（仓库内）

- `.agents/design.md` — 架构约束（保持 core loop 小、通过 capability/skill 扩展、文件系统分层）
- `.agents/gotchas.md` — 常见 gotcha（能力/Lua 选择是配置驱动；Lua driver README 是 agent 面向文档）
- `.agents/spec/claw-skill-spec.md` — Skill 目录结构 / SKILL.md 规则 / `{CUR_SKILL_DIR}` / 构建同步
- `.agents/spec/lua-module-spec.md` — Lua 模块规范
- `docs/src/content/docs/{en,zh-cn}/reference-core/` — claw_core / claw_cap / claw_event_router / claw_memory / claw_skill / claw_ramfs
- `docs/src/content/docs/{en,zh-cn}/reference-cap/` — 每个 cap_* 的工具表与实现说明
