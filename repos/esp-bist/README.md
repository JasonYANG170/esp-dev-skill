# esp-bist-skill

面向 Espressif **ESP-BIST（Built-In Self Test，内置自测）** 库的 AI Skill。ESP-BIST 是一个 CMake 库，提供符合 **IEC 60730-1 Annex H（Class B）** 与 IEC 60335-1 Annex R 的硬件自测例程，覆盖 Espressif RISC-V SoC 的 CPU 寄存器、CSR、RAM（March A/X）、Flash（CRC32）、程序计数器、时钟源、栈溢出、看门狗与数字 I/O。

本 Skill 让 AI 代理在**完全基于真实仓库文档与代码**的前提下，正确地开发/集成/调试基于 ESP-BIST 的安全关键固件，绝不臆造 API。

## 功能特性

- **场景配方（recipes）**：8 个真实场景，从工程搭建到单项测试再到完整 IEC 60730 集成，每条配方都附带可直接复制改编的代码、前置条件表与常见错误表。
- **API 速查（resources/api_reference.md）**：逐字摘自 `src/bist/**/include/*.h` 与 `src/bist/drivers/include/*.h`，包含所有 `bist_*` 函数签名、错误码、驱动封装。
- **配置速查（resources/config_reference.md）**：摘自 `src/bist/Kconfig` 的全部 `CONFIG_ESP_BIST_*` 符号、默认值与含义。
- **陷阱汇总（resources/pitfalls.md）**：17 条来自真实头文件/示例/文档的易错点，每条都有 WRONG/CORRECT 对照。
- **示例索引（resources/example_list.md）**：仓库内 `samples/` 与 `tests/` 的真实路径表。
- **芯片支持矩阵**：ESP32-C3 / C5 / C6 / H2（C61 / H4 / P4 见 `docs/en/get_started.rst`），含 CSR 数量、XT WDT 支持等差异。

## 安装

将本 Skill 克隆/复制到 Claude Code 的 skills 目录（项目级或用户级）：

```bash
# 项目级（推荐）：放进项目根目录的 .claude/skills/
git clone <this-skill-repo> .claude/skills/esp-bist-skill

# 或用户级：所有项目共享
# Windows: %USERPROFILE%\.claude\skills\esp-bist-skill
# Linux/macOS: ~/.claude/skills/esp-bist-skill
```

复制后，Claude Code 会在你提到 BIST、IEC 60730、安全关键、自检等触发词时自动加载本 Skill。

## 目录结构

```
esp-bist-skill/
├── SKILL.md                     # 核心规则、配方索引、芯片/配置/错误码表、陷阱、执行流程
├── AGENTS.md                    # 补充约定：工程结构、构建流程、代码生成清单
├── recipes/                     # 8 个场景配方
│   ├── project_setup.md
│   ├── standalone_integration.md
│   ├── cpu_register_csr_test.md
│   ├── pc_and_stack_test.md
│   ├── watchdog_windowed_test.md
│   ├── ram_flash_test.md
│   ├── clock_test.md
│   ├── gpio_test.md
│   └── metrics_collection.md
├── resources/                   # 速查文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md                    # 本文件
└── CHANGELOG.md
```

## 支持范围

| 维度 | 范围 |
|---|---|
| 目标 SoC | ESP32-C3、ESP32-C5、ESP32-C6、ESP32-H2（C61/H4/P4 见上游文档） |
| 架构 | RISC-V（CSR/PMP/PMA 相关测试为 RISC-V 专有） |
| 构建系统 | CMake + Ninja，基于 ESP-IDF + MCUboot 2.2.0 |
| 工具链 | `riscv32-esp-elf-gcc` / `riscv32-esp-elf-gdb` |
| 标准 | IEC 60730-1 Annex H（Class B）、IEC 60335-1 Annex R |
| 许可证 | LGPL-3.0-or-later（与上游 `LICENSE` 一致） |

## 上游仓库

- https://github.com/espressif/esp-bist

## 许可证

本 Skill 文档按上游 ESP-BIST 的 LGPL-3.0-or-later 精神提供；引用的 API 名称、代码片段均来自上游仓库，版权归原作者 Espressif Systems (Shanghai) Co., Ltd. 所有。
