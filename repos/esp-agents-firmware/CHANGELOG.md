# Changelog

本文件记录 `esp-agents-firmware-skill` 的版本变更。

格式参考 <https://keepachangelog.com/zh-CN/1.1.0/>，版本号遵循语义化版本（SemVer）。

## [1.1.0] - 2026-06-22

补齐审计确认的两个文档化但未覆盖的场景：Matter Controller 服务初始化层、无显示屏设备适配。

### 新增

- **recipes/matter_controller_service_init.md** — 覆盖 `esp.service.matter-controller`（MatterCTL）RainMaker 服务的服务层（与既有 `matter_controller_control.md` 的工具层互补）：
  - 6 个服务参数（BaseURL/UserToken/RMakerGroupID/MatterNodeID/MTCtlCMD/MTCtlStatus）与读写/持久化标志。
  - MTCtlStatus 7 位状态位图（`matter_controller_status_t`）。
  - 手机 App 触发的初始化时序（setparams → UpdateNOC → UpdateDeviceList），来自 SPEC.md。
  - 固件侧 `matter_controller_start()` 初始化链：RainMaker node + MatterController device、`CONFIG_OPENTHREAD_BORDER_ROUTER` 下的 `esp_rmaker_thread_br_enable`、`matter_controller_enable(vendor_id, callback)`、`esp_matter::start`、`matter_controller_client::init(0,0,5580)`、`init_device_manager`、`IP_EVENT_STA_GOT_IP` → `matter_controller_handle_update`+`matter_controller_update_device_list`。
  - 5 种 callback 分派（AUTHORIZE/QUERY_MATTER_FABRIC_ID/SETUP_CONTROLLER/UPDATE_CONTROLLER_NOC/UPDATE_DEVICE）。
  - 来源示例：`examples/matter_controller/`（SPEC.md / setup_guide.md / README.md / app_controller.cpp / app_controller.h / app_matter_controller.h / app_matter_controller_callback.{h,cpp} / matter_controller_std.h / app_matter_device_manager.h）。
- **recipes/headless_no_display.md** — 覆盖 `docs/example_customisation.md` "Devices with no display" 一节：
  - 注释 `app_main.c` 的 `app_display_init()`（无屏时 `init_display()` 失败卡住）。
  - 在 `app_text_message_callback` / `app_state_changed_callback` 里去掉 `app_display_set_text` / `app_display_system_state_changed`，保留串口打印。
  - `set_emotion` 工具处理建议（跳过 `app_display_set_emotion` 或不注册）。
  - 无屏配网二维码改由产品包装提供（`setup_guide.md` 原文），开发期可用串口 `set-wifi`/`set-token`/`set-agent`。
  - 来源：`examples/voice_chat/main/app_main.c`、`examples/matter_controller/main/app_main.c`、`examples/common/app_common/{include/app_display.h,src/app_display.c}`、`docs/example_customisation.md`、`examples/matter_controller/setup_guide.md`。

### 变更

- **SKILL.md**：版本 `1.0.0`→`1.1.0`；在"项目与构建"表加 `headless_no_display.md`；在"Matter 控制器"表加 `matter_controller_service_init.md`（并标注既有 `matter_controller_control.md` 为"工具层"）。
- **resources/example_list.md**：补充 matter_controller 组件下 `SPEC.md`、`app_matter_controller.h`、`app_matter_controller_callback.{h,cpp}`、`matter_controller_std.h`、`app_matter_device_manager.h` 路径。
- **resources/api_reference.md**：新增 10.3 节"Matter Controller 服务规格"（6 参数表 + MTCtlStatus 位说明 + `matter_controller_service_create` + callback 分派），原 10.3 顺延为 10.4。

### Grounding

- 新增内容均源自仓库 `examples/matter_controller/` 的 SPEC.md / 头文件 / app_controller.cpp 与 `docs/example_customisation.md`、`examples/common/app_common/` 源码，可逐项查证。无屏板对 `set_emotion` 工具的处理为基于源码的最小改动建议（仓库未提供无屏专用分支），已在 recipe 中标注。

## [1.0.0] - 2026-06-18

首个正式版本。基于 `esp-agents-firmware` 仓库的真实文档与源码构建。

### 新增

- **SKILL.md**：12 条核心原则、When to Use、10 个 recipe 索引、板级支持表、Agent 事件参考表、设备状态机、13 条带正确/错误代码对照的关键陷阱、执行流程表、失败策略表。
- **AGENTS.md**：项目上下文、文件命名/包含模式、标准工程结构、标准 `app_main` 模板、本地工具 handler 模板、构建流程、组件依赖版本、codegen 清单、Do Not Modify 注记。
- **recipes/**（10 个场景）：
  - `build_and_flash.md` — 选板、构建、烧录、监控
  - `add_custom_board.md` — 自定义板级配置
  - `custom_deployment.md` — 切换自定义 ESP Private Agents 部署
  - `agent_init_and_events.md` — Agent 初始化与事件处理
  - `device_setup_provisioning.md` — 配网与首次设置
  - `local_tool_register.md` — 注册与实现本地工具
  - `builtin_tools.md` — 内置工具（set_reminder/get_local_time/set_volume/set_emotion）
  - `matter_controller_control.md` — Matter 设备列表与控制
  - `audio_pipeline.md` — 音频管线配置
  - `console_commands.md` — 串口 console 命令
- **resources/**：
  - `api_reference.md` — 10 个模块的真实 API 速查（esp_agent / app_agent / setup / console / app_audio / app_device / app_display / app_common_tools / touch / Matter）
  - `config_reference.md` — 全部真实 Kconfig 与板级宏、组件依赖版本
  - `pitfalls.md` — 15 条陷阱展开版
  - `example_list.md` — 仓库真实目录与一句话说明
- **README.md**：中文介绍、功能特性、安装方式、目录结构。
- **CHANGELOG.md**：本文件。

### Grounding

- 所有函数名、结构体、枚举、宏、配置项、文件路径、代码片段均源自 `esp-agents-firmware` 仓库的 `components/`、`examples/`、`docs/` 与头文件，可逐项查证。
- 未臆造任何 API；仓库文档较薄处（如部分组件内部实现）以可查证的头文件签名与示例源码为准，不补造内容。
