# 陷阱汇总（Pitfalls）

> SKILL.md "Critical Pitfalls" 的展开版，附定位与最小修复。

## 1. app_main 初始化顺序错乱

**症状**：`agent_handle` 为 NULL、事件不触发、外设未就绪。

**根因**：`app_tools_register` 依赖 `app_agent_init` 内部的 `agent_handle`；`app_agent_start` 依赖 board_manager / audio / device 已就绪。

**修复**：严格按 `examples/voice_chat/main/app_main.c` 顺序：event loop → NVS → `agent_console_init` → `esp_board_manager_init` → `app_display_init` → `app_audio_init` → `app_device_init` → `app_agent_init` → `app_tools_register` → `app_agent_start`（Matter 再 `matter_controller_start_task`）。

## 2. 工具 result 用栈/字面量

**症状**：工具调用后设备崩溃（use-after-free / 栈失效）。

**根因**：`esp_agent_tools.h` 注释明确 `*result` 必须 heap 分配，框架发完响应后内部 `free`。

**修复**：一律 `*result = strdup("...")` 或 `malloc` + `snprintf`。

## 3. 工具参数类型假设错误

**症状**：参数取不到值。

**根因**：框架仅支持 NUMBER/STRING/BOOL（`esp_agent_tool_param_type_t`），无数组/对象。

**修复**：遍历 `params[]`，按 `name` 完全匹配 + `type` 判断取 `.value.i/.s/.b`。

## 4. 固件注册了工具但 Agent 不调用

**症状**：handler 永远不进。

**根因**：`agent_config.json` 未声明同名 tool 或 inputSchema 不符。

**修复**：在 `examples/<ex>/agent_config.json` 的 `toolConfiguration.tools[]` 加同名 `{ "type":"local", "name":"<name>", "inputSchema":{...} }`，重建 Agent 并重新下发 agent_id。

## 5. 在 app_main 直接调 esp_agent_start

**症状**：start 返回错误，Agent 不连。

**根因**：三前置条件（网络 + agent_id + token）未满足；`agent_setup` 仅在全部满足时发 `AGENT_SETUP_EVENT_START`。

**修复**：交给 `app_agent.c` 的 `agent_setup_event_handler` 在 START 事件里 `xTaskCreate(app_agent_start_task,...)` 启动。

## 6. 默认 agent_id 为空导致不连接

**症状**：配网完成但仍卡在未启动。

**根因**：`CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` 默认空，`agent_setup_get_agent_id()` 返回 NULL → 启动任务直接 return。

**修复**：编译期 `CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` 设值，或运行时 `set-agent` / RainMaker App 扫码下发。

## 7. 自定义部署未改 endpoint

**症状**：token 有效但连公共云失败。

**根因**：`CONFIG_ESP_AGENT_API_ENDPOINT` 仍为默认 `api.agents.espressif.com`。

**修复**：`idf.py menuconfig` → ESP Agent Config → 改为自建 URL，重新 build flash。

## 8. 未到 STARTED 就发语音

**症状**：`app_agent_send_speech` 返回 `ESP_ERR_INVALID_STATE`。

**根因**：内部判断 `state != APP_AGENT_STATE_STARTED` 即拒绝。

**修复**：先 `app_agent_get_state()` 判断，或在 `ESP_AGENT_EVENT_START` 之后再驱动麦克风。

## 9. Matter client-only 与 server/commissioner 冲突

**症状**：`CONFIG_ESP_CLIENT_ONLY_MATTER_CONTROLLER` 不可见或编译错。

**根因**：Kconfig 互斥约束未满足。

**修复**：关闭 `ESP_MATTER_ENABLE_MATTER_SERVER`、`ESP_MATTER_COMMISSIONER_ENABLE`、`WIFI_NETWORK_COMMISSIONING_DRIVER`、`THREAD_NETWORK_COMMISSIONING_DRIVER`、`ENABLE_CHIPOBLE` 后再开 client-only。

## 10. control_device cluster/command 不符 schema

**症状**：Matter 设备无响应或报错。

**根因**：cluster_id/command_id/args 不符 `agent_config.json` 文档。

**修复**：OnOff(6) 仅 TurnOff=0/TurnOn=1/Toggle=2；LevelControl(8) MoveToLevel=0；ColorControl(768) MoveToHue=0/MoveToSaturation=3。args 只改变量字段，固定字段保持 `0`。

## 11. 板级宏缺失导致外设异常

**症状**：电容触摸/背光/指示灯不工作。

**根因**：`board_defs.h` 未定义对应宏。

**修复**：参考 `examples/common/boards/esp_vocat_board_v1_2/board_defs.h`，按需定义 `CAPACITIVE_TOUCH_SUPPORTED`/`CAPACITIVE_TOUCH_CHANNEL_GPIO`/`LEDC_BACKLIGHT_SUPPORTED`/`INDICATOR_DEVICE_NAME` 等，代码侧用 `#if defined(...)` 保护。

## 12. 自定义 event handler 未转发

**症状**：自定义 handler 生效但状态机/音频联动失效（不休眠、不播放、不切状态）。

**根因**：自定义 handler 没调 `app_agent_default_event_handler`。

**修复**：在自定义 handler 内首先调用 `app_agent_default_event_handler(arg, event_base, event_id, event_data)`，再做自定义逻辑。

## 13. 无屏板仍调 app_display

**症状**：无 LCD 板初始化失败或卡死。

**根因**：`app_display_init` 找不到面板。

**修复**：按 `docs/example_customisation.md`，在 `app_main.c` 注释 `app_display_init()` 及所有 `app_display_*` 调用；device callbacks 中也去掉 display 调用。完整分步见 `recipes/headless_no_display.md`。

## 14. 音频参数与云端不匹配

**症状**：语音卡顿、变调、听不清。

**根因**：`CONFIG_AUDIO_UPLOAD/DOWNLOAD_SAMPLE_RATE`/`FRAME_DURATION_MS` 与云端 Agent 模型期望不符。

**修复**：对齐上下行采样率（默认上 8000 / 下 16000）与帧长（默认 20 / 60 ms）；改后两端一致。

## 15. AGENT_SETUP_EVENT_AGENT_ID_UPDATE 误用

**症状**：在 ID_UPDATE 事件里直接启动 Agent，但 token/网络未就绪导致失败。

**根因**：`agent_setup.h` 明确 ID_UPDATE 不保证其他条件已满足，仅 `START` 保证。

**修复**：只在 `AGENT_SETUP_EVENT_START` 里启动；ID_UPDATE 仅用于刷新已运行 Agent 的 agent_id（见 `app_agent.c` 的 `agent_setup_event_handler`）。
