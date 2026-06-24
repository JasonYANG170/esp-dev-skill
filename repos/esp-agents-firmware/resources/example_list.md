# 示例与组件清单

> 仓库实际目录的真实路径与一句话说明。新增示例请以这些为模板。

## 顶层示例（examples/）

| 路径 | 说明 |
|---|---|
| `examples/voice_chat/` | 通用语音对话 Agent 固件（默认 Agent: `friend`），支持 set_reminder / get_local_time / set_volume / set_emotion |
| `examples/matter_controller/` | Matter 控制器 + Thread Border Router 固件（默认 Agent: `matter_controller`），在通用工具外加 get_device_list / control_device |

### voice_chat 内部

| 路径 | 说明 |
|---|---|
| `examples/voice_chat/main/app_main.c` | 标准 app_main 初始化与启动 |
| `examples/voice_chat/main/app_tools.c` | 注册公共工具 + `set_emotion` handler |
| `examples/voice_chat/main/app_tools.h` | `esp_err_t app_tools_register(void);` |
| `examples/voice_chat/main/idf_component.yml` | idf ≥ 5.5 + 依赖 |
| `examples/voice_chat/agent_config.json` | `friend` Agent 配置（不入固件，用于 Dashboard 建模） |
| `examples/voice_chat/setup_guide.md` | voice_chat 配网与使用指南 |
| `examples/voice_chat/README.md` | voice_chat 示例说明 |

### matter_controller 内部

| 路径 | 说明 |
|---|---|
| `examples/matter_controller/main/app_main.c` | app_main + `matter_controller_start_task()` |
| `examples/matter_controller/main/app_tools.c` | 公共工具 + `get_device_list` / `control_device` / `set_emotion` |
| `examples/matter_controller/main/matter/app_controller.h` | `matter_controller_start_task` / `get_device_list` / `control_device` |
| `examples/matter_controller/components/matter_controller/` | Matter 控制器组件 |
| `examples/matter_controller/components/matter_controller/rmaker_controller_service/` | RainMaker 控制器服务（app_matter_controller.h / matter_controller_std.h / app_matter_device_manager.h） |
| `examples/matter_controller/components/matter_controller/rmaker_controller_service/SPEC.md` | `esp.service.matter-controller` 服务规格（6 参数 / MTCtlCMD 枚举 / MTCtlStatus 7 位位图 / 初始化时序） |
| `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_controller.h` | `matter_controller_status_t` / `matter_controller_handle_t` / callback 枚举 / `enable`·`handle_update`·`report_status`·`update_device_list` |
| `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_controller_callback.h` | `app_matter_controller_callback` 声明 |
| `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_controller_callback.cpp` | 5 种 callback 实现（authorize / fetch_fabric_id / setup_controller / update_noc / update_device_list） |
| `examples/matter_controller/components/matter_controller/rmaker_controller_service/matter_controller_std.h` | RainMaker 参数宏与 `matter_controller_service_create` |
| `examples/matter_controller/components/matter_controller/rmaker_controller_service/app_matter_device_manager.h` | `update_device_list` / `fetch_device_list` / `init_device_manager` |
| `examples/matter_controller/components/matter_controller/controller_rest_apis/` | RainMaker REST API 封装（controller_rest_apis.h / matter_device.h） |
| `examples/matter_controller/components/matter_controller/op_creds_issuer/` | Matter 凭证签发 |
| `examples/matter_controller/agent_config.json` | `matter_controller` Agent 配置（含 control_device schema） |
| `examples/matter_controller/setup_guide.md` | matter_controller 配网与 Thread Border Router 指南 |
| `examples/matter_controller/README.md` | matter_controller 示例说明 |

## 公共组件（examples/common/）

| 路径 | 说明 |
|---|---|
| `examples/common/app_common/` | 跨示例共享的应用封装组件 |
| `examples/common/app_common/include/app_agent.h` | Agent 应用层封装 API |
| `examples/common/app_common/include/app_audio.h` | 音频管线封装 API |
| `examples/common/app_common/include/app_device.h` | 设备状态机与事件队列 |
| `examples/common/app_common/include/app_display.h` | 显示与表情 API |
| `examples/common/app_common/include/app_common_tools.h` | 内置工具宏与 handler 原型 |
| `examples/common/app_common/include/app_touch_press.h` | 触摸按压检测 |
| `examples/common/app_common/include/app_capacitive_touch.h` | 电容触摸 |
| `examples/common/app_common/src/app_agent.c` | app_agent 实现 + 默认事件处理 |
| `examples/common/app_common/src/app_common_tools.c` | set_reminder / get_local_time / set_volume 实现 |
| `examples/common/app_common/src/app_audio.c` | 音频管线实现 |
| `examples/common/app_common/src/app_device.c` | 状态机实现 |
| `examples/common/app_common/src/app_display.c` | 显示实现 |
| `examples/common/app_common/Kconfig` | App Common 配置项 |
| `examples/common/app_common/idf_component.yml` | 依赖（esp_emote_expression / agent / audio / setup / boards / network_provisioning） |
| `examples/common/assets/audio/` | 音频资源 |
| `examples/common/boards/` | 板级配置目录 |

## 板级配置（examples/common/boards/）

| 路径 | 说明 |
|---|---|
| `examples/common/boards/esp_vocat_board_v1_2/` | ESP-VoCat Core Board v1.2（board_defs.h / sdkconfig.defaults / README.md） |
| `examples/common/boards/esp_box_3/` | ESP-BOX-3 |
| `examples/common/boards/m5stack_cores3/` | M5Stack CoreS3 |
| `examples/common/boards/m5stack_cores3_h2_gateway/` | M5Stack CoreS3 + H2 Gateway Module（Thread Border Router） |
| `examples/common/boards/README.md` | 板目录说明 |
| `examples/common/boards/idf_component.yml` | boards 组件清单 |

## 官方组件（components/）

| 路径 | 说明 |
|---|---|
| `components/agent/` | Agent 通信组件（WebSocket） |
| `components/agent/include/esp_agent.h` | 聚合头 |
| `components/agent/include/esp_agent_core.h` | 核心：init/start/stop/config |
| `components/agent/include/esp_agent_events.h` | 事件枚举与 message_data |
| `components/agent/include/esp_agent_messages.h` | send_speech / send_text |
| `components/agent/include/esp_agent_tools.h` | 本地工具 API |
| `components/agent/Kconfig.projbuild` | `CONFIG_ESP_AGENT_API_ENDPOINT` |
| `components/agent_console/` | 串口 console 助手组件 |
| `components/agent_console/include/agent_console.h` | console API |
| `components/audio/` | 音频管线（recorder + playback） |
| `components/audio/audio_playback/` | 播放子模块 |
| `components/audio/audio_recorder/` | 录原子模块 |
| `components/audio/Kconfig` | `CONFIG_ENABLE_AEC` |
| `components/setup/` | 设备配网与 setup 组件 |
| `components/setup/include/agent_setup.h` | setup API + 事件 |
| `components/setup/include/setup/console.h` | `setup_console_register_commands` |
| `components/setup/include/setup/rainmaker.h` | RainMaker setup API |
| `components/setup/src/setup_console.c` | set-token / set-agent 命令 |
| `components/setup/src/agent_setup.c` | set-wifi 命令 + NVS 存储 |
| `components/setup/src/setup_rainmaker.c` | RainMaker 集成 |

## 文档（docs/）

| 路径 | 说明 |
|---|---|
| `docs/agent_customisation.md` | 自定义 Agent（Dashboard 建模 + 本地工具 + 更新设备 Agent） |
| `docs/board_customisation.md` | 自定义板步骤 |
| `docs/deployment_customisation.md` | 自定义 ESP Private Agents 部署 |
| `docs/example_customisation.md` | 示例定制（本地工具 / 设备手册 URL / 默认 Agent ID / 无屏设备） |
