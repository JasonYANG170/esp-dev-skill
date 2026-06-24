# esp-amp-skill

ESP-AMP 非对称多核（Asymmetric Multiprocessing）固件开发 AI Skill。基于 Espressif 官方 [esp-amp](https://github.com/espressif/esp-amp) 仓库的真实文档与源码构建，帮助 AI agent 正确开发 ESP32-C5 / ESP32-C6 / ESP32-P4 上的 maincore + subcore（LP/HP core）双核固件，覆盖核间 IPC、构建系统、生命周期管理与低功耗优化。

## 功能特性

- **场景化 recipes（11 个）**：unified/separate build、subcore 生命周期、共享内存、软件中断、Event、Virtqueue、RPMsg、RPC、light sleep、subcore 外设。
- **API/配置速查**：按组件分组的真实函数签名（来自 `include/esp_amp_*.h`），真实 Kconfig 符号（来自 `Kconfig`）。
- **陷阱清单**：37 条 WRONG/CORRECT 对照，覆盖初始化、Event、RPMsg、Virtqueue、构建、运行时、light sleep、外设。
- **真实 example 索引**：仓库 `examples/` 下全部 9 个 example 的路径、描述与支持目标。
- **完全 grounding**：所有 API 名、结构体、宏、Kconfig 符号、文件路径、代码片段均来自真实仓库文档与源码，未发现则省略，绝不臆造。

## 支持范围

| SoC | IDF 版本 | maincore | subcore |
|---|---|---|---|
| ESP32-C5 | v5.5+ | HP core | LP core（bare-metal） |
| ESP32-C6 | v5.3.1+ | HP core | LP core（bare-metal） |
| ESP32-P4 | v5.3.1+ | HP core | HP core（bare-metal，LP 暂不支持） |

覆盖组件：Shared Memory / SysInfo、Software Interrupt、Event、Virtqueue（Queue）、RPMsg、RPC、System（生命周期/panic/printf 路由）、Port/Env 层、自动 Light Sleep、subcore 外设。

## 安装

将本 skill 克隆/复制到 Claude Code 的 skills 目录：

- **项目级**：复制到 `<project>/.claude/skills/esp-amp-skill/`
- **用户级**：复制到 `~/.claude/skills/esp-amp-skill/`

```shell
# 项目级（与 esp-amp 仓库同目录使用）
mkdir -p .claude/skills
cp -r esp-amp-skill .claude/skills/

# 用户级
mkdir -p ~/.claude/skills
cp -r esp-amp-skill ~/.claude/skills/
```

确保 `esp-amp` 仓库可访问（skill 引用路径为 `espressif-repos/esp-amp/`，可按需调整 `SKILL.md` 与 recipes 中的路径）。

## 目录结构

```
esp-amp-skill/
├── SKILL.md                    # Skill 入口：原则、陷阱、工作流、recipe 索引
├── AGENTS.md                   # 工程约定：命名、结构、构建流程、清单
├── recipes/                    # 11 个场景化 recipe
│   ├── unified_build.md
│   ├── separate_build.md
│   ├── lifecycle.md
│   ├── shared_memory.md
│   ├── software_interrupt.md
│   ├── event.md
│   ├── virtqueue.md
│   ├── rpmsg.md
│   ├── rpc.md
│   ├── light_sleep.md
│   └── subcore_peripheral.md
├── resources/                  # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   ├── example_list.md
│   └── architecture.md
├── README.md
└── CHANGELOG.md
```

## 使用方式

Claude Code 启动后，当用户请求涉及 ESP-AMP / 主核副核 / 跨核通信 / RPMsg / RPC 等 trigger 词时，自动加载本 skill。agent 会：

1. 阅读 `SKILL.md` 获取原则与陷阱；
2. 按 user 意图匹配 `recipes/` 中的场景；
3. 需要时查 `resources/` 的 API/配置/陷阱速查；
4. 以 `examples/` 中最接近的工程为模板生成代码。

## 许可证

Apache-2.0（与 esp-amp 仓库一致）。
