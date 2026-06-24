# ESP-Claw 仓库真实示例与参考点索引

> 全部路径来自 `espressif/esp-claw` 真实仓库（相对仓库根）。用作 Recipe 引用与代码范本。

## 1. 应用入口与装配（edge_agent）

| 路径 | 说明 |
|---|---|
| `application/edge_agent/main/main.c` | `app_main`：NVS → config → board_manager → FATFS → claw_paths → wifi → http_server → `app_claw_start` |
| `application/edge_agent/main/app_fs.c` / `app_fs.h` | FATFS 挂载与 storage/system 根路径 |
| `application/edge_agent/main/Kconfig.projbuild` | App Config（Wi-Fi / HTTP allowlist 默认） |
| `application/edge_agent/sdkconfig.defaults` | 应用层默认 sdkconfig |
| `application/edge_agent/partitions_8MB.csv` / `partitions_16MB.csv` / `partitions_32MB.csv` | 分区表 |
| `components/common/app_claw/app_claw.c` | Agent 栈装配：session/router/scheduler/memory/skill/cap/core 启动顺序 |
| `components/common/app_claw/app_capabilities.c` | 各 `cap_*_register_group` 注册点 |
| `components/common/app_claw/app_lua_modules.c` | 各 `lua_module_*_register` / `lua_driver_*_register` 注册点（须先于 `cap_lua_register_group`） |
| `components/common/app_claw/app_claw_cli.c` | Console REPL 注册（`skill` 命令默认未在此启用） |
| `components/common/app_claw/Kconfig` | `APP_CLAW_CAP_*` / `APP_CLAW_LUA_*` / `APP_CLAW_MEMORY_MODE_*` |

## 2. 内置 Skill 与脚本（FATFS 种子，DATA 根）

| 路径 | 说明 |
|---|---|
| `application/edge_agent/fatfs_image/storage/skills/light_switch/SKILL.md` | 灯控 Skill（LED Strip / GPIO Light，含 JSON Schema 与 Recommended Flow） |
| `application/edge_agent/fatfs_image/storage/skills/light_switch/scripts/led_strip_switch.lua` | WS2812 等灯带控制范本（`arg_schema` + `led_strip` + `xpcall/cleanup`） |
| `application/edge_agent/fatfs_image/storage/skills/light_switch/scripts/gpio_light_switch.lua` | GPIO 灯控制 |
| `application/edge_agent/fatfs_image/storage/skills/scheduled_task/SKILL.md` | 定时任务 Skill（`wake_agent`/`send_message`/`run_script`） |
| `application/edge_agent/fatfs_image/storage/skills/scheduled_task/scripts/add_scheduled_task.lua` | `capability.call` 调 `add_router_rule`+`scheduler_add` 并带回滚的范本 |
| `application/edge_agent/fatfs_image/storage/skills/plan_mode/SKILL.md` | Plan 模式 Skill |
| `application/edge_agent/fatfs_image/system/.recovery/router_rules/router_rules.json` | 启动默认自动化规则（startup / im_session_command / im_llm_command / im_any_message_working_reply / im_any_message_agent / agent_stage_im_notify / agent_out_message_send_message） |
| `application/edge_agent/fatfs_image/system/.recovery/scheduler/schedules.json` | 启动默认调度种子 |
| `application/edge_agent/fatfs_image/system/.recovery/memory/{user,soul,identity}.md` | 记忆 profile 种子 |

## 3. 能力组件（`components/claw_capabilities/`）—— 实现参考

| 路径 | 说明 |
|---|---|
| `cap_lua/src/cap_lua.c`、`include/cap_lua.h` | Lua 受管根、同步/异步作业、模块注册锁 |
| `cap_files/src/cap_files.c`、`include/cap_files.h` | 文件 IO + 路径沙箱（禁 `..`）参考 |
| `cap_scheduler/src/cap_scheduler.c`、`include/cap_scheduler.h` | 时间触发；`cap_scheduler_config_t` 字段参考 |
| `cap_router_mgr/` | router_rules 管理（`event_router` Console 命令） |
| `cap_skill_mgr/src/cap_skill_mgr.c` | `cap_skill` 工具实现（`activate_skill` 注入文档） |
| `cap_system/src/cap_system.c` | 纯查询型能力参考（`get_system_info`/`get_current_time`/`restart_device`） |
| `cap_im_platform/src/cap_im_tg.c` | 事件源型能力参考（长轮询 + FNV-1a 去重环 `CAP_IM_TG_DEDUP_CACHE_SIZE=64` + 异步附件队列 + multipart 流式 send） |
| `cap_im_platform/src/cap_im_{feishu,qq,wechat}.c`、`cap_im_attachment.c` | 各 IM 后端 + 共享附件 |
| `cap_im_platform/include/cap_im_{tg,feishu,qq,wechat}.h` | `*_set_token` / `*_set_attachment_config`（`storage_root_dir`/`max_inbound_file_bytes`/`enable_inbound_attachments`，默认 2 MB） |
| `cap_im_platform/skills/cap_im_platform/SKILL.md` | IM 渠道选择规则 + send 工具用法（声明 4 个 IM 组） |
| `cap_mcp_client/`、`cap_mcp_server/include/cap_mcp_server.h` | MCP：client 发现调用；server 是 HYBRID 把设备工具对外暴露（`cap_mcp_server_tool_def_t` + init/add_tool/start/stop/deinit，SSE + mDNS） |
| `cap_http_request/`、`cap_web_search/`、`cap_llm_inspect/` | HTTP / 搜索 / 嵌套推理 |

## 4. 框架核心（`components/claw_modules/`）

| 路径 | 说明 |
|---|---|
| `claw_core/include/claw_core.h` | Agent 运行时：`claw_core_request_t`/`_response_t`/`_config_t`、context provider、取消、完成观察者 |
| `claw_cap/include/claw_cap.h` | `claw_cap_descriptor_t`/`claw_cap_group_t`/`claw_cap_call_context_t` |
| `claw_event_router/include/claw_event.h` | `claw_event_t`、publish/publish_message/outbound binding/cancel/purge |
| `claw_memory/` | full/lightweight 记忆、session history、profile provider；skills: `memory_ops`、`profile_memory_ops` |
| `claw_memory/include/claw_memory.h` | `claw_memory_config_t` / `claw_memory_item_t`（`id[40]`/`source[16]`/`content[256]`/`tags[96]`/`keywords[128]`）/ `claw_memory_query_t` + C API（store/recall/update/forget/list） |
| `claw_memory/src/claw_memory_cap.c` | `s_memory_descriptors[]`：`memory_store`/`recall`/`list`/`update`/`forget` 真实 `input_schema_json` |
| `claw_memory/skills/memory_ops/SKILL.md` | `memory_*` 工具用法与硬/软规则（cap_groups: `claw_memory`） |
| `claw_memory/skills/profile_memory_ops/SKILL.md` | profile 三件套用法（cap_groups: `cap_files`） |
| `application/edge_agent/fatfs_image/system/.recovery/memory/{user,soul,identity}.md` | 可编辑 profile 种子文件（user 画像 / soul 灵魂 / identity 身份卡） |
| `claw_skill/` | Skill 目录扫描、按 session 持久化激活、文档读取、catalog provider |
| `claw_paths/` | `CLAW_PATH_SYSTEM`/`CLAW_PATH_DATA`、`claw_paths_set`/`claw_paths_join` |

## 5. Lua 模块 / 驱动（`components/lua_modules/`）

每个组件含 `README.md`（agent 面向 API 文档，构建期同步进 docs）；下表为已确认文档可读的代表：

| 组件 | README 说明要点 |
|---|---|
| `lua_driver_gpio/README.md` | `gpio.set_direction/set_level/get_level`，mode 枚举 |
| `lua_module_delay/README.md` | `delay.delay_ms/delay_us`，整数约束 |
| `lua_module_storage/README.md` | `get_root_dir/join_path/exists/stat/mkdir/write_file/read_file/listdir/remove/rename/get_free_space` |
| `lua_module_json/README.md` | `json.encode/decode`，数组 vs 对象规则 |
| `lua_module_event_publisher/README.md` | `publish_message`（字符串优先）/ `publish_trigger` / `publish`，点号调用 |
| `lua_driver_i2c/lib/ssd1306.md`、`lib_si12t_touch.md` | 纯 Lua I2C OLED / Si12T 触摸 |
| `lua_driver_rmt/lib/ir_driver.md` | 红外驱动 |
| `lua_module_fuel_gauge/lib/lib_fuel_gauge.md` | BQ27220/MAX17048 电量计 |
| `lua_module_ble/lib/`、`lua_module_ble_hid/lib/ble_hid_actions.md` | BLE / BLE HID |

> 其余模块（`display`/`lcd`/`lvgl`/`audio`/`camera`/`imu`/`button`/`knob`/`ir`/`http_server`/`thread`/`system`/`image`/`dht`/`environmental_sensor`/`magnetometer`/`lcd_touch`/`sci`/`led_strip`/`mcpwm`/`pcnt`/`uart`/`adc`/`touch`/`call_capability`）API 见各自 `README.md` 与 `lib/*.md`。

## 6. 板子定义（`application/edge_agent/boards/`）

| vendor / board | 说明 |
|---|---|
| `espressif/esp32_S3_DevKitC_1` | ESP32-S3 DevKitC（含 breadboard 变体） |
| `espressif/esp32_S3_DevKitC_1_breadboard`、`..._N32R16V_breadboard` | 面包板变体 |
| `espressif/esp_box_3` | ESP-BOX 3 |
| `espressif/esp_sparkbot` | 含 README + board_devices.yaml + setup_device.c 的完整范本 |
| `espressif/esp32_p4_eye`、`esp32_p4_function_ev` | ESP32-P4 |
| `espressif/esp32_s31_korvo1` | ESP32-S31 |
| `m5stack/m5stack_cores3` | M5Stack CoreS3（含板级 components） |
| `lilygo/lilygo_t_display_p4_v1` | 含 README / README_CN |
| `waveshare/waveshare_ESP32_S3_RLCD_4_2` | Waveshare |
| `movecall/movecall_{moji,moji2,moji_esp32s3,cuican_esp32s3}` | movecall 系列 |
| `dfrobot/`、`rockbase-iot/nm_cyd_c5`、`Nologo.Tech/xingzhi_395`、`community/` | 其它 |

> 板子目录均含 `board_info.yaml` / `board_devices.yaml` / `board_peripherals.yaml` / `sdkconfig.defaults.board` / `setup_device.c`，可选 `components/` 与 `fatfs_image/`（overlay 进 SYSTEM）。

## 7. 其它应用

| 路径 | 说明 |
|---|---|
| `application/mcp_server_point/README.md` | MCP server 精简应用（仅 `cap_lua` + `cap_mcp_server`，SSE + mDNS） |
| `application/mcp_server_point/main/main.c` | `app_main` 真实装配：NVS→FATFS→Wi-Fi→`app_claw_start`→`cap_mcp_server` 三段式（init/tools_init/start） |
| `application/mcp_server_point/components/app_config/app_config.c` | `enabled_cap_groups` 写死 `"cap_lua"` |
| `application/mcp_server_point/components/mcp_server_point_tools/cap_mcp_lua.c` | 6 个 `lua.*` MCP 工具回调与 `s_tool_defs[]`（run_script / async / list / get / stop / stop_all） |
| `application/mcp_server_point/components/mcp_server_point_tools/cap_mcp_lua.h` | `cap_mcp_lua_tools_init()` 调用契约 |
| `application/mcp_server_point/sdkconfig.defaults` | lean Kconfig 默认（关 CORE/EVENT_ROUTER/MEMORY/IM/CLI 等） |
| `application/third_party/buddy_pet/` | 第三方「宠物」应用（含 match_watch、pet_buddy、skill_builder、自有 lua_modules） |

## 8. 规范与设计文档（`.agents/`、`docs/`）

| 路径 | 说明 |
|---|---|
| `.agents/design.md` | 架构约束（core loop 小、capability/skill 扩展、FS 分层、板子 overlay） |
| `.agents/gotchas.md` | 能力/Lua 选择是配置驱动；Lua driver README 是 agent 文档 |
| `.agents/docs.md` | 文档工作流指引 |
| `.agents/spec/claw-skill-spec.md` | Skill 目录 / SKILL.md / `{CUR_SKILL_DIR}` / 构建同步 / 命名冲突 |
| `.agents/spec/lua-module-spec.md` | Lua 模块规范 |
| `docs/src/content/docs/{en,zh-cn}/tutorial/` | get-started / first-interactions / web-config / skills-lab / bom / assemble / faq / supported-list |
| `docs/src/content/docs/{en,zh-cn}/reference-project/` | boot-and-runtime / dataflow-and-automation / configuration / console-usage / core-cap-event / lua / memory / skills / skills-and-capability / build-from-source |
| `docs/src/content/docs/{en,zh-cn}/reference-core/` | claw-core / claw-cap / claw-event-router / claw-memory / claw-skill / claw-ramfs |
| `docs/src/content/docs/{en,zh-cn}/reference-cap/` | 每个 cap_* 的工具表与实现 + implement-capability + lua-modules |
| `docs/src/assets/router_rules.schema.json` | router_rules.json JSON Schema |
