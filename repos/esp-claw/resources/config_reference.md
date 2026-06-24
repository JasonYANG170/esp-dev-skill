# ESP-Claw 配置项速查

> 全部 Kconfig 符号与运行时字段来自真实源码：`components/common/app_claw/Kconfig`、`components/claw_modules/claw_core/Kconfig`、`application/edge_agent/main/Kconfig.projbuild`、`app_config.c`、`main.c`。

## 1. 优先级模型（必读）

```
生效运行时值 = NVS 有值 ? NVS 值 : menuconfig 编译时默认
```
- 配置存于 NVS 命名空间 `app`（`application/edge_agent/components/app_config/app_config.c`）
- 首次从 Web/串口保存后 NVS 接管；恢复出厂 = 清对应 NVS 键
- 启动：`app_config_load` 先装 menuconfig 默认，再用 NVS 键覆盖

## 2. 编译期 Kconfig（改后需 `idf.py build && flash`）

### App Claw Config（`components/common/app_claw/Kconfig`）

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `APP_CLAW_ENABLE_CLI` | bool | y | 启用 app_claw 管理的串口 REPL |
| `APP_CLAW_ENABLE_EMOTE` | bool | y | 启用龙虾表情 UI（依赖 `ESP_BOARD_DEV_DISPLAY_LCD_SUPPORT`） |
| `APP_CLAW_MEMORY_MODE_FULL` | choice | 选 full | 结构化记忆（full）：summary-tag 目录 + `memory_recall` |
| `APP_CLAW_MEMORY_MODE_LIGHTWEIGHT` | choice | | 轻量：直接注入 `MEMORY.md`，无结构化抽取 |
| `APP_CLAW_CAP_FILES` | bool | y | 文件能力 |
| `APP_CLAW_CAP_MEMORY` | bool | y | `claw_memory` 子系统（`APP_CLAW_MEMORY_MODE` 依赖它） |
| `APP_CLAW_CAP_EVENT_ROUTER` | bool | y | 事件路由器 |
| `APP_CLAW_CAP_CORE` | bool | y | `claw_core`（LLM 凭据齐时启动） |
| `APP_CLAW_CAP_IM_QQ` | bool | y | QQ（select EVENT_ROUTER） |
| `APP_CLAW_CAP_IM_FEISHU` | bool | y | 飞书 |
| `APP_CLAW_CAP_IM_TG` | bool | y | Telegram |
| `APP_CLAW_CAP_IM_WECHAT` | bool | y | 微信 |
| `APP_CLAW_CAP_IM_LOCAL` | bool | y | Web 内置 IM（select SESSION_MGR + EVENT_ROUTER） |
| `APP_CLAW_CAP_SCHEDULER` | bool | y | 调度器 |
| `APP_CLAW_CAP_LUA` | bool | y | Lua 能力 |
| `APP_CLAW_CAP_MCP_CLIENT` | bool | y | MCP 客户端 |
| `APP_CLAW_CAP_MCP_SERVER` | bool | y | MCP 服务端 |
| `APP_CLAW_CAP_SKILL_MGR` | bool | y | Skill 管理 |
| `APP_CLAW_CAP_SYSTEM` | bool | y | 系统信息/时间/重启 |
| `APP_CLAW_CAP_LLM_INSPECT` | bool | y | LLM 检查（嵌套推理/图像） |
| `APP_CLAW_CAP_HTTP_REQUEST` | bool | y | HTTP 请求（受 allowlist 守护） |
| `APP_CLAW_CAP_WEB_SEARCH` | bool | y | Web 搜索 |
| `APP_CLAW_CAP_ROUTER_MGR` | bool | y | 路由规则管理（select EVENT_ROUTER） |
| `APP_CLAW_CAP_SESSION_MGR` | bool | y | 会话管理 |
| `APP_CLAW_CAP_AGENT_MGR` | bool | n（SESSION_MGR&&CORE 时默认 y） | root-only 子 Agent 管理 |

### App Claw Lua Modules（同 Kconfig，节选）

| 符号 | 默认 | 说明 |
|---|---|---|
| `APP_CLAW_LUA_DRIVER_ADC/GPIO/I2C/MCPWM` | y | 基础驱动 |
| `APP_CLAW_LUA_DRIVER_PCNT` | y if `SOC_PCNT_SUPPORTED` | 脉冲计数/编码器 |
| `APP_CLAW_LUA_DRIVER_RMT` | y if `SOC_RMT_SUPPORTED` | RMT（WS2812 等） |
| `APP_CLAW_LUA_DRIVER_TOUCH` | y if `SOC_TOUCH_SENSOR_SUPPORTED` | 触摸 |
| `APP_CLAW_LUA_DRIVER_UART` | y | UART |
| `APP_CLAW_LUA_MODULE_AUDIO` | y if `ESP_BOARD_DEV_AUDIO_CODEC_SUPPORT` | 音频 |
| `APP_CLAW_LUA_MODULE_CAMERA` | y if `ESP_BOARD_DEV_CAMERA_SUPPORT` | 摄像头 |
| `APP_CLAW_LUA_MODULE_LCD_TOUCH` | y if `ESP_BOARD_DEV_LCD_TOUCH_SUPPORT && ...I2C_SUPPORT` | LCD 触摸 |
| `APP_CLAW_LUA_MODULE_VISION` | y（依赖 CAMERA_SUPPORT） | 视觉（motion_detect 等） |
| `APP_CLAW_LUA_MODULE_BLE` | n | 通用 BLE |
| `APP_CLAW_LUA_MODULE_BLE_HID` | n（select `BT_NIMBLE_HID_SERVICE`） | BLE HID |
| `APP_CLAW_LUA_MODULE_BOARD_MANAGER/BUTTON/CAPABILITY/DELAY/DISPLAY/EVENT_PUBLISHER/HTTP_SERVER/JSON/THREAD/IMAGE/LED_STRIP/LVGL/STORAGE/SYSTEM` | y | 高层模块 |
| `APP_CLAW_LUA_MODULE_ENVIRONMENTAL_SENSOR/FUEL_GAUGE/IMU/IR/KNOB/LCD/MAGNETOMETER/SCI` | n | 按需 |

### ESP-Claw Core（`components/claw_modules/claw_core/Kconfig`）

| 符号 | 选项 | 默认 | 说明 |
|---|---|---|---|
| `CLAW_CORE_STAGE_VERBOSITY` | `CLAW_CORE_STAGE_VERBOSITY_SIMPLE` / `..._VERBOSE` | Simple | Simple 仅工具名；Verbose 含参数预览+轮次，向路由发 `agent_stage` 事件 |

### App Config（`application/edge_agent/main/Kconfig.projbuild`）

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `APP_WIFI_SSID` / `APP_WIFI_PASSWORD` | string | "" | 默认 Wi-Fi（被 NVS 覆盖） |
| `APP_WIFI_AP_SSID_PREFIX` | string | `esp-claw` | 配置 AP SSID 前缀，全 SSID = `<prefix>-<MAC[3..5]>` |
| `APP_WIFI_AP_CHANNEL` | int 1..13 | 1 | 配置 AP 信道 |
| `APP_WIFI_AP_MAX_CONN` | int 1..10 | 4 | 配置 AP 最大接入数 |
| `APP_WIFI_MAX_RETRY` | int 1..20 | 5 | STA 连续失败几次后回落纯 AP |
| `APP_SEARCH_HTTP_ALLOWLIST` | string | `*` | HTTP 请求 allowlist；`*` 全放行，或逗号分隔域名/IP（支持 `*.example.org`） |

## 3. 运行时字段（NVS 覆盖，经 Web/串口保存）

> 这些字段在 NVS 命名空间 `app`；改后 `app_claw_update_config` 热更运行时（LLM 类）。

**LLM**（任一为空则 `claw_core` 不启动）：`llm_api_key`、`llm_backend_type`、`llm_model`、`llm_base_url`、`llm_auth_type`、`llm_timeout_ms`、`llm_max_tokens`、`llm_max_tokens_field`、`llm_supports_tools`、`llm_supports_vision`、`llm_default_image_max_bytes`、`llm_image_remote_url_only`

**Wi-Fi**：`wifi_ssid`、`wifi_password`、`ap_ssid`、`ap_password`、`ap_behavior`

**IM**：各平台 App ID / Secret / Token（QQ / 飞书 / Telegram / 微信）

**其它**：时区（`TZ`，默认 `CST-8`）、`search_http_allowlist`、Web Search key（Brave/Tavily）

## 4. 文件系统布局（运行时路径）

| 路径 | 角色 | 可写 |
|---|---|---|
| `/system` (`CLAW_PATH_SYSTEM`) | 内置 Skill、内置 Lua 库（`/system/scripts/builtin/lib`）、`.recovery` 种子、板子 overlay | 只读 |
| `/fatfs` 或 SD 挂载点 (`CLAW_PATH_DATA`) | `sessions/` `memory/` `skills/`(用户) `scripts/` `router_rules/` `scheduler/` `inbox/` | 可写 |

> 可复用代码禁硬编码 `/fatfs`：C 用 `claw_paths_join()`，Lua 用 `storage.get_root_dir()`。

## 5. 受管根与边界

| 能力 | 受管根 | 文件类型 | 备注 |
|---|---|---|---|
| `cap_files` | `/fatfs`（可 `cap_files_set_base_dir` 改） | 任意文本 | 读上限 32KB；禁 `..`；`write_file` 自动建父目录 |
| `cap_lua` | `/fatfs/scripts`（默认，可 `cap_lua_set_base_dir`） | `.lua` | 也支持 `{CUR_SKILL_DIR}/scripts/...` |
| `cap_skill` | `/fatfs/skills`（用户）+ `/system/skills`（内置） | `SKILL.md` | 冲突 DATA 优先 |

## 参考来源

- `components/common/app_claw/Kconfig`
- `components/claw_modules/claw_core/Kconfig`
- `application/edge_agent/main/Kconfig.projbuild`
- `application/edge_agent/components/app_config/app_config.c`
- `application/edge_agent/main/main.c`
- `application/edge_agent/sdkconfig.defaults`、`sdkconfig.defaults.esp32p4`
- `docs/src/content/docs/en/reference-project/configuration.mdx`
