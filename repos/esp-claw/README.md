# esp-claw-skill

面向乐鑫 **ESP-Claw**（物联网设备「Chat Coding 聊天造物」式 AI Agent 固件框架）的 Claude Code / Agent 技能包。全部内容（API、配置项、示例路径、代码片段）均来自 `espressif/esp-claw` 仓库的真实文档与源码，不臆造任何接口。

ESP-Claw 用 C 语言在 ESP32 系列芯片上本地完成「感知—决策—执行」闭环：IM 聊天 / 定时 / 事件 触发 Agent Loop，由 `claw_core` 调用云端 LLM、通过 `claw_cap` 执行 Capability 工具、用 Lua 脚本驱动硬件、用 `claw_event_router` 做声明式自动化。

## 特性

- **场景化 Recipe（8 篇）**：构建烧录、板子适配、Capability 开发、Lua 模块开发、router_rules、定时任务、Skill 编写、Lua 自动化、设备配置
- **真实 API 速查**：`claw_cap` / `claw_core` / `claw_event_router` C 结构体与 API、全部能力工具 id、Lua 模块签名
- **配置项速查**：`APP_CLAW_CAP_*` / `APP_CLAW_LUA_*` / `CLAW_CORE_STAGE_VERBOSITY` 等 Kconfig，NVS 运行时字段与优先级模型
- **陷阱清单**：启动条件、路径分层、Lua 注册锁、Skill 规范、异步回路等 36 条
- **执行流程**：从意图到落地（固件改动 vs 运行时改动）的标准步骤

## 安装

### 方式一：克隆到项目级 skills 目录
```bash
git clone <本仓库> .claude/skills/esp-claw-skill
```

### 方式二：放到用户级 skills 目录
- macOS / Linux：`~/.claude/skills/esp-claw-skill/`
- Windows：`%USERPROFILE%\.claude\skills\esp-claw-skill\`

把整个 `esp-claw-skill/` 目录（含 `SKILL.md`、`AGENTS.md`、`recipes/`、`resources/`）放进去即可。Claude Code 启动时会按 `SKILL.md` frontmatter 的 `description` 与触发词自动加载。

## 目录结构

```
esp-claw-skill/
├── SKILL.md                 # 主技能文件（原则 / 何时用 / Recipe 索引 / 陷阱 / 流程）
├── AGENTS.md                # 工程约定补充（命名 / include / boot 模式 / 构建 / checklist）
├── recipes/                 # 8 篇场景 Recipe（含真实代码与常见错误表）
│   ├── build_and_flash.md
│   ├── board_adaptation.md
│   ├── implement_capability.md
│   ├── lua_module.md
│   ├── router_rules.md
│   ├── scheduled_task.md
│   ├── write_skill.md
│   └── automation_lua.md
├── resources/
│   ├── api_reference.md     # C / Lua / Console API 速查
│   ├── config_reference.md  # Kconfig + NVS 运行时字段
│   ├── pitfalls.md          # 36 条汇总陷阱
│   └── example_list.md      # 仓库真实示例与参考点索引
├── README.md                # 本文件
└── CHANGELOG.md
```

## 支持范围

- **目标芯片**：ESP32-S3 / ESP32-P4 / ESP32-C5 / ESP32-S31
- **构建**：ESP-IDF v5.5.4 + ESP Board Manager（`esp-bmgr-assist`）
- **主应用**：`application/edge_agent`
- **覆盖**：Capability（`cap_*`）、Lua 模块/驱动（`lua_module_*`/`lua_driver_*`）、Skill、Event Router、Scheduler、Memory、MCP、IM、Board 适配

## 上游

- 仓库：https://github.com/espressif/esp-claw
- 在线文档：https://esp-claw.com/en/tutorial/
- 许可：Apache-2.0（与上游一致）
