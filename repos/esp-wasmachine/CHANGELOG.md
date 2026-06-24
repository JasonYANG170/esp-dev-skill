# Changelog

## [1.0.0] - 2026-06-18

Initial release of `esp-wasmachine-skill`.

### Added
- `SKILL.md` — 12 条核心原则、何时使用、Kconfig/组件对照表、12 条 WRONG/CORRECT 关键陷阱、执行流程与失败策略。
- `AGENTS.md` — 工程上下文、仓库布局、include 模式、`app_main` 标准模式、新增 native 模块范式、构建流程、codegen 清单、Do-Not-Modify 说明。
- `recipes/` — 12 个场景菜谱：new_project、build_flash、target_board、run_iwasm、app_manager、filesystem、native_libc、native_http_mqtt、native_lvgl、native_rainmaker_prov、ext_vfs、shell_wifi、data_sequence。
- `resources/api_reference.md` — 固件侧 API（`wm_wamr_*` / `wm_shell_*` / `wm_ext_wasm_vfs_*`）与全部 WASM native import（libc/libm/HTTP/MQTT/LVGL/RainMaker/Wi-Fi provisioning）及 WAMR 签名。
- `resources/config_reference.md` — 全部 `WASMACHINE_*` Kconfig 选项、默认值、依赖关系，及各目标 `sdkconfig.defaults` 与分区表对照。
- `resources/pitfalls.md` — 汇总陷阱（文件系统、init 顺序、Kconfig 依赖、运行/安装、VFS、LVGL、内存）。
- `resources/example_list.md` — `examples/wasmachine/` 真实文件索引 + 各组件 README 路径。
- `resources/lifecycle.md` — `app_main` 启动顺序与 `iwasm` / App Manager 两种执行模型。
- `README.md`（中文）、`CHANGELOG.md`。

### Grounding
- 所有 API 名、Kconfig 符号、命令选项、文件路径均取自 `esp-wasmachine` 仓库 `components/` 与 `examples/wasmachine/` 的源码、头文件与 `Kconfig.wasmachine`。
- 仓库内无 Sphinx 文档树（`docs/` 仅含一张架构图），故依据 README + 源码而非臆造。
