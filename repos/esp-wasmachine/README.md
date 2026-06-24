# esp-wasmachine-skill

针对 Espressif **ESP-WASMachine**（基于 wasm-micro-runtime / WAMR 的 WebAssembly 虚拟机固件开发框架）的 AI Skill。本 Skill 让 AI 代理在 ESP32 / ESP32-S3 / ESP32-C6 / ESP32-P4 上正确地创建、配置、编译、烧录与调试 ESP-WASMachine 固件工程——所有 API、Kconfig 选项、命令、文件路径均严格取自 `esp-wasmachine` 仓库真实源码，绝不臆造。

## 特性

- **场景化 recipes**：覆盖新建工程、编译烧录、选板、`iwasm` 运行、App Manager + `host_tool` 远程管理、文件系统打包、libc/HTTP/MQTT/LVGL/RainMaker/Wi-Fi 配网 native、Extended VFS、`sta` 配网、`data_seq` 序列化等真实用法。
- **真实 API 参考**：`resources/api_reference.md` 列出每个注册到 `"env"` 模块的 native import（libc/libm/HTTP/MQTT/LVGL/RainMaker/Wi-Fi 配网）及其 WAMR 签名，以及固件侧 `wm_wamr_init`、`wm_wamr_app_mgr_init`、`wm_shell_init` 等调用。
- **完整 Kconfig 地图**：`resources/config_reference.md` 收录全部 `WASMACHINE_*` 选项、默认值与 `depends on` 关系，外加各目标的 `sdkconfig.defaults` 与分区表对照。
- **关键陷阱清单**：`SKILL.md` 的 12 条 WRONG/CORRECT 对照，外加 `resources/pitfalls.md` 的汇总。
- **运行时生命周期**：`resources/lifecycle.md` 画出 `app_main` 启动顺序与两种执行模型（`iwasm` 一次性 vs App Manager 常驻 applet）。

## 安装

将本 Skill 目录放入 Claude Code 的 skills 目录（项目级或用户级）：

- 项目级：`<repo>/.claude/skills/esp-wasmachine-skill/`
- 用户级：`~/.claude/skills/esp-wasmachine-skill/`

```bash
git clone <本仓库> /path/to/esp-wasmachine-skill
# 项目级
mkdir -p .claude/skills && cp -r /path/to/esp-wasmachine-skill .claude/skills/
```

加载后，匹配触发词（如 "ESP-WASMachine"、"WASMachine"、"WAMR"、"iwasm"、"wasm 虚拟机" 等）即自动启用。

## 支持范围

- **目标芯片**：ESP32、ESP32-S3（含 S3-BOX）、ESP32-C6、ESP32-P4
- **工具链**：ESP-IDF v5.1.x–v5.5.x/master；wasm-micro-runtime 2.x（托管依赖）
- **组件覆盖**：`wasmachine_core`、`wasmachine_ext_wasm_native`、`wasmachine_ext_wasm_native_rainmaker`、`wasmachine_ext_wasm_vfs`、`wasmachine_shell`、`wasmachine_data_sequence`
- **参考工程**：`examples/wasmachine/`（仓库唯一参考固件）

## 目录结构

```
esp-wasmachine-skill/
├── SKILL.md            # 主入口：原则、何时使用、陷阱、执行流程
├── AGENTS.md           # 约定、include 模式、构建流程、codegen 清单
├── README.md           # 本文件
├── CHANGELOG.md
├── recipes/            # 12 个场景菜谱
│   ├── new_project.md
│   ├── build_flash.md
│   ├── target_board.md
│   ├── run_iwasm.md
│   ├── app_manager.md
│   ├── filesystem.md
│   ├── native_libc.md
│   ├── native_http_mqtt.md
│   ├── native_lvgl.md
│   ├── native_rainmaker_prov.md
│   ├── ext_vfs.md
│   ├── shell_wifi.md
│   └── data_sequence.md
└── resources/
    ├── api_reference.md      # native + host API（按模块）
    ├── config_reference.md   # 全部 Kconfig 选项 + sdkconfig.defaults
    ├── pitfalls.md           # 陷阱汇总
    ├── example_list.md       # examples/ 真实路径索引
    └── lifecycle.md          # 启动顺序 + 两种执行模型
```

## 许可

本 Skill 文档本身按其仓库许可使用；引用的 ESP-WASMachine 源码许可为 Apache-2.0（见 `SKILL.md` frontmatter）。
