# Changelog

本文件记录 esp-claw-skill 的发布历史。格式参考 Keep a Changelog，版本号语义化。

## [1.1.0] - 2026-06-18

补齐 audit 确认的 3 个 Recipe 缺口：MCP 服务端部署、记忆系统使用、IM 平台端到端集成。新增内容全部来自 `espressif/esp-claw` 仓库的真实文档与源码。

### 新增 Recipes
- `recipes/mcp_server.md` — 把 ESP32 当 MCP 服务端：`application/mcp_server_point` 精简应用（仅 `cap_lua` + `cap_mcp_server`，SSE + mDNS），6 个 `lua.*` MCP 工具，`gen-bmgr-config` 构建流程。源码：`application/mcp_server_point/{main/main.c, components/mcp_server_point_tools/cap_mcp_lua.c, sdkconfig.defaults}`、`components/claw_capabilities/cap_mcp_server/include/cap_mcp_server.h`、`docs/.../reference-cap/cap-mcp.mdx`
- `recipes/memory_usage.md` — 长期记忆 + profile：`memory_store/recall/list/update/forget`（含真实 `input_schema_json`）、summary-tag 检索（无向量库）、自动抽取/去重、profile 三件套（`user.md`/`soul.md`/`identity.md`）、full/lightweight 模式。源码：`components/claw_modules/claw_memory/{include/claw_memory.h, src/claw_memory_cap.c, skills/{memory_ops,profile_memory_ops}/SKILL.md}`、`application/edge_agent/fatfs_image/system/.recovery/memory/*.md`、`docs/.../{reference-project/memory, reference-core/claw-memory}.mdx`
- `recipes/im_integration.md` — IM 端到端闭环：`cap_im_platform` 统一源组件、平台差异（chat_id 格式）、Telegram 长轮询 + FNV-1a 去重环 + 异步附件、`attachment_saved` 事件载荷、`router_rules.json` 真实 7 条 IM 规则。源码：`application/edge_agent/fatfs_image/system/.recovery/router_rules/router_rules.json`、`components/claw_capabilities/cap_im_platform/{src/cap_im_{tg,feishu,qq,wechat,attachment}.c, include/cap_im_*.h, skills/cap_im_platform/SKILL.md}`、`docs/.../reference-cap/cap-im-platform.mdx`

### 更新
- `SKILL.md`：版本 `1.0.0` → `1.1.0`；在 Scenario Quick Reference 三张分组表（项目与构建 / 自动化与编排 / 设备行为与扩展）各加一行新 Recipe 索引
- `resources/example_list.md`：扩写「其它应用」节（mcp_server_point 6 条路径）、能力组件节（cap_mcp_server header / IM 平台 header 与 Skill / IM setter）、框架核心节（claw_memory 6 条 header/src/skills/profile 种子）
- `resources/api_reference.md`：新增 3 节真实 C API 签名——`claw_memory_*`（含 `claw_memory_item_t`/`claw_memory_query_t`/Context Providers）、`cap_mcp_server_*`（`cap_mcp_server_tool_def_t` + init/add_tool/start/stop/deinit）、`cap_im_*` 凭据与附件 setter（含 `cap_im_tg_attachment_config_t`）
- `CHANGELOG.md`：本条目

### 接地说明
- 三篇 Recipe 引用的所有函数名、结构体、工具 id、Kconfig 符号、文件路径、JSON 片段均来自仓库真实文档与源码（含 `cap_mcp_lua.c::s_tool_defs[]`、`claw_memory_cap.c::s_memory_descriptors[]`、`router_rules.json` 7 条规则、`cap_im_tg.c::CAP_IM_TG_DEDUP_CACHE_SIZE=64`）；未在仓库中找到的内容一律未写入。

## [1.0.0] - 2026-06-18

首个发布。基于 `espressif/esp-claw` 仓库真实文档（`docs/`、`.agents/`）与源码（`application/edge_agent/`、`components/`）构建。

### 新增
- `SKILL.md`：12 条核心原则、When to Use、Recipe 索引、Capability/Lua 模块速查、FATFS 布局、Event Router 状态机、15 条关键陷阱（含 WRONG/CORRECT 对照）、执行流程与失败策略表
- `AGENTS.md`：项目上下文、命名与目录约定、include/import 模式、boot/注册模式、构建工作流、代码生成 checklist、Do-Not-Modify 说明
- `recipes/`（8 篇）：
  - `build_and_flash.md` — ESP-IDF v5.5.4 + Board Manager 构建烧录
  - `board_adaptation.md` — 新开发板适配（YAML + setup_device.c + overlay）
  - `implement_capability.md` — 新增 `cap_*` 能力组
  - `lua_module.md` — 新增 Lua 模块（C 绑定 + README + test/lib + Skill）
  - `router_rules.md` — Event Router 规则与 `out_message` 回路
  - `scheduled_task.md` — 定时任务（cron/interval/once）
  - `write_skill.md` — 编写 ESP-Claw Skill（JSON frontmatter + `{CUR_SKILL_DIR}`）
  - `automation_lua.md` — Lua 驱动硬件 + storage/event_publisher/capability.call
  - `configuration.md` — LLM/IM/记忆配置与 NVS 优先级
- `resources/`：
  - `api_reference.md` — C（claw_cap/core/event_router）、能力工具 id、Lua 模块 API、Console 命令
  - `config_reference.md` — Kconfig 符号表 + NVS 运行时字段 + 优先级模型 + 路径布局
  - `pitfalls.md` — 36 条汇总陷阱（启动/路径/能力/Skill/自动化/Lua/配置）
  - `example_list.md` — 仓库真实示例、组件、板子、文档索引
- `README.md`：中文介绍、安装方式、目录结构、支持范围
- `CHANGELOG.md`：本文件

### 基线
- 目标：ESP32-S3 / ESP32-P4 / ESP32-C5 / ESP32-S31
- 构建：ESP-IDF v5.5.4 + `esp-bmgr-assist`
- 许可：Apache-2.0

### 接地说明
- 所有函数名、结构体、工具 id、Kconfig 符号、文件路径、代码片段均来自仓库真实文档与源码；未在仓库中找到的内容一律未写入
- Recipe 引用的示例路径（如 `fatfs_image/storage/skills/light_switch/scripts/led_strip_switch.lua`、`.recovery/router_rules/router_rules.json`）均为仓库中真实存在文件
