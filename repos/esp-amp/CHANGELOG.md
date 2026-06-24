# Changelog

所有重要变更记录于此文件。格式基于 [Keep a Changelog](https://keepachangelog.com/)，版本号遵循 [Semantic Versioning](https://semver.org/)。

## [1.0.0] - 2026-06-18

### Added

- 初始发布 esp-amp-skill，针对 Espressif esp-AMP 非对称多核框架。
- 新增 `SKILL.md`：12 条核心原则、SoC 支持表、分层架构表、printf/panic 工作流表、13 条 Critical Pitfalls（WRONG/CORRECT 对照）、执行工作流、失败策略表。
- 新增 `AGENTS.md`：工程约定（命名、include 模式、unified/separate 工程结构、canonical maincore/subcore 入口模式、ISR 安全日志约定、构建流程、CMake 关键片段、双核代码生成清单、Do Not Modify 说明）。
- 新增 10 个场景化 recipe：`unified_build`、`separate_build`、`lifecycle`、`shared_memory`、`software_interrupt`、`event`、`virtqueue`、`rpmsg`、`rpc`、`light_sleep`、`subcore_peripheral`。每个含适用摘要、触发意图、前置条件表、分步真实代码、常见错误表、参考 example 路径。
- 新增 `resources/api_reference.md`：来自 `include/esp_amp_*.h` 与 `port/include/*.h`、`system/include/esp_amp_system.h` 的真实函数签名（SysInfo/SwIntr/Event/Queue/RPMsg/RPC/System/Port/Env 分组）。
- 新增 `resources/config_reference.md`：来自 `components/esp_amp/Kconfig` 的真实配置项（总开关、内存管理、工具链、表大小、System、断言级别、unified build 自定义选项、内存放置宏）。
- 新增 `resources/pitfalls.md`：37 条分类陷阱（初始化/Event/RPMsg/Virtqueue/SwIntr/构建/运行时/Light Sleep/外设）。
- 新增 `resources/example_list.md`：仓库 `examples/` 下全部 9 个 example 路径、描述、支持目标与复制模板建议。
- 新增 `resources/architecture.md`：分层架构图、SysInfo/Virtqueue/RPMsg/Event 数据流、subcore 生命周期状态流、printf 路由决策流、各目标内存布局、与 OpenAMP 区别。
- 新增 `README.md`（中文）与 `CHANGELOG.md`。

### Grounding

- 所有 API、结构体、宏、Kconfig 符号、文件路径、代码片段均来自 `espressif-repos/esp-amp/` 的 `docs/`、`include/`、`Kconfig` 与 `examples/`。
- 目标芯片支持、IDF 版本要求来自 `README.md` 与各 example `README.md`。
- 未在仓库中找到的内容一律省略，未做任何臆造。
