# Changelog

本文件记录 esp-bist-skill 的版本变更。

格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

## [1.1.0] - 2026-06-18

补充三份审计确认的文档化+有示例覆盖、但此前无配方的话题。全部内容（函数/宏/Kconfig/路径/GDB 脚本）均取自上游仓库的真实文档与示例，未臆造。

### 新增

- **recipes/zephyr_integration.md**：Zephyr RTOS 在 ESP32-C6 HP+LP 双核的集成（`west build --sysbuild`、`Kconfig.sysbuild` 的 `REMOTE_BOARD` 自动绑定、`remote/boards/esp32c6_devkitc_lpcore.overlay` 的 mbox/SHM/LP UART、LP 核 `ulp_lp_core_intr_disable()/enable()` 包裹 `bist_cpu_regs_test` / `bist_cpu_csr_regs_test` / `bist_ram_test_march_x`、HP/LP mbox ping/pong）。
  - 源示例：`samples/zephyr/`（`src/main.c`、`remote/src/main.c`、`Kconfig.sysbuild`、`prj.conf`、`remote/prj.conf`、`remote/boards/esp32c6_devkitc_lpcore.overlay`、`README.md`）。
- **recipes/mcp_server_setup.md**：BIST MCP 服务器接入 AI 助手（Cursor / VS Code）。六个工具（`search_bist_docs` / `get_api_reference` / `get_architecture_info` / `search_kconfig_options` / `search_source_code` / `get_supported_socs`）、工作区/用户级/远程三种安装、`ingest.py` 五步确定性流水线、`mcp_data_drift` CI 强制、BM25 引擎。
  - 源文件：`docs/en/mcp_server.rst`、`mcp-server/{server.py,ingest.py,search.py,requirements.txt,mcp.json}`、`.cursor/mcp.json`、`.vscode/mcp.json`。
- **recipes/safety_validation_workflow.md**：IEC 60730 Class B 证据工作流。QEMU（`-icount 3`）+ GDB（`:1234`）故障注入的通用脚本骨架与每类测试的注入点（`tb testRegA_<name>`、`_bist_ram_test_start`、`bist_verify_pc_test`、`bist_test_wdt_timeout`）、pytest fixture（`conftest.py`）、JUnit XML 产出，并串起 `software_validation` → `test_traceability_matrix` → `coverage_analysis` → `tool_qualification` → `safety_case_summary` 五份证据文档。
  - 源文件：`docs/en/{software_validation,test_traceability_matrix,coverage_analysis,tool_qualification,safety_case_summary}.rst`、`conftest.py`、`tests/idf_targets.py`、`tests/{cpu_reg_test,ram_test,pc_test,wdt_test}/pytest_qemu_*.py`、`samples/standalone/gdbinit`。

### 变更

- **SKILL.md**：版本 `1.0.0` → `1.1.0`；Scenario Quick Reference 新增 "AI Assistant & Tooling" 与 "Safety Certification" 两个分组，并在 "Project Setup & Integration" 加入 `zephyr_integration.md`。
- **resources/example_list.md**：扩充 `samples/zephyr/` 描述；新增 "MCP Server" 与 "Validation Infrastructure" 两节；Docs 表补入 5 份认证/验证文档与 `mcp_server.rst`。
- **resources/api_reference.md**：External 节补入 `ulp_lp_core_interrupts.h`；新增 "Zephyr mbox API"（外部）、"MCP Server Tools"（Python）、"Validation Fixtures"（Python `conftest.py`）三节。

### 接地说明

三份新配方的每个函数名、Kconfig 符号、文件路径、GDB 脚本片段均来自上游仓库的 `samples/zephyr/`、`mcp-server/`、`docs/en/*.rst`、`conftest.py`、`tests/*/pytest_qemu_*.py`。外部（非 BIST 树）API（`ulp_lp_core_interrupts.h`、Zephyr `<zephyr/drivers/mbox.h>`）已在 api_reference.md 明确标注来源。

## [1.0.0] - 2026-06-18

首个发布版本。基于 esp-bist 上游仓库（v1.0.0，2026-01-23）的真实文档、头文件、Kconfig、示例与样例构建。

### 新增

- **SKILL.md**：核心规则（12 条）、When to Use、配方索引、芯片支持矩阵、Kconfig 速查表、错误码表、standalone 执行流状态机、12 条 WRONG/CORRECT 陷阱、执行流程表、失败策略表。
- **AGENTS.md**：项目上下文、文件命名与包含约定、标准工程结构、canonical `main()` 与 Unity 单测范式、构建流程、`bist.conf` 模式、代码生成清单、Do Not Modify 说明。
- **recipes/**（8 条配方）：
  - `project_setup.md`：工程搭建、CMake/MCUboot/QEMU。
  - `standalone_integration.md`：完整 IEC 60730 集成（post-boot + 运行时 + 窗口看门狗 + fail-safe）。
  - `cpu_register_csr_test.md`：CPU 通用寄存器 + CSR（trap/PMP/PMA/mexstatus）。
  - `pc_and_stack_test.md`：程序计数器（IRAM/Flash/RTC 放置）+ 栈溢出三件套。
  - `watchdog_windowed_test.md`：WDT 复位验证 + 窗口看门狗（下溢/溢出/连续）。
  - `ram_flash_test.md`：RAM March A/X + Flash CRC32（含后处理注入）。
  - `clock_test.md`：32 kHz 外部晶振（XT WDT）+ 40 MHz 主晶振漂移。
  - `gpio_test.md`：GPIO 输出/输入合理性 + 无效 GPIO 处理 + 各 SoC 引脚映射。
  - `metrics_collection.md`：RISC-V 性能计数器 CSR 指标采集。
- **resources/api_reference.md**：逐字摘自上游头文件的全部 `bist_*` 函数、错误码、WDT/GPIO/esp_timer 驱动封装、日志宏、性能指标宏。
- **resources/config_reference.md**：摘自 `src/bist/Kconfig` 的全部 `CONFIG_ESP_BIST_*` 符号、默认值、含义与 Zephyr 构建说明。
- **resources/pitfalls.md**：17 条来自真实头文件/示例/文档的易错点。
- **resources/example_list.md**：`samples/` 与 `tests/` 真实路径表，源码布局索引，文档索引。
- **README.md**：中文简介、功能特性、安装方式、目录结构、支持范围。
- **CHANGELOG.md**：本文件。

### 接地说明（Grounding）

所有函数名、结构体/typedef、宏、Kconfig 符号、文件路径、代码片段均来自上游仓库的 `src/bist/**/include/*.h`、`src/bist/Kconfig`、`samples/standalone/main.c`、`tests/*/main.c`、`tests/*/bist.conf`、`docs/en/*.rst` 与 `README.md`。未在仓库中找到的内容一律从略，不做臆造。
