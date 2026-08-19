# 配置设备（LLM / IM / 记忆 / NVS 优先级）

> **适用摘要**: 配置 ESP-Claw `edge_agent` 的运行时项——LLM 后端、IM 平台凭据、记忆模式、HTTP/Web Search——理解「menuconfig 默认 vs NVS 覆盖」的优先级，并通过 Web / 串口生效。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-claw/resources/`, source/examples in `repos/esp-claw/`, and this recipe path `repos/esp-claw/recipes/configuration.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "配置 LLM / 模型 / API key"
- "配 Telegram / 飞书 / QQ / 微信"
- "menuconfig 改了不生效"
- "恢复出厂 / 清 NVS"
- "Web 配置页"

## 前置条件

| 条件 | 要求 |
|---|---|
| NVS 命名空间 | `app`（`edge_agent` 存配置于此） |
| 读写逻辑 | `application/edge_agent/components/app_config/app_config.c` |
| 参考 | `docs/.../reference-project/configuration.mdx`、`tutorial/web-config.mdx` |

## 分步说明

### 1. 理解优先级（关键）

```
生效运行时值 = NVS 有值 ? NVS 值 : menuconfig 编译时默认
```
- 首次从 Web / 串口保存后，NVS 接管；menuconfig 默认值不再生效
- 「恢复出厂」= 清除相关 NVS 键
- LLM 相关字段在 `main.c::main_copy_claw_to_app_config` / `app_config_to_claw` 间同步：`llm_api_key` / `llm_backend_type` / `llm_model` / `llm_base_url` / `llm_auth_type` / `llm_timeout_ms` / `llm_max_tokens` / `llm_supports_tools` / `llm_supports_vision` / `llm_image_remote_url_only` 等

### 2. 让 claw_core 启动的三个必填项

`llm_api_key`、`llm_model`、`llm_backend_type` 三者**任一为空**，`app_claw_start` 跳过 `claw_core_init/start` 并 WARN。后果：
- `ask` / 默认 Agent 路由 / 图像检查 等核心功能不可用
- 事件路由 / 自动化 / 本地能力 / Console REPL 仍可用
- 配齐三项后**重启设备**才生效

支持的后端：OpenAI 风格 API（GPT 系列）、Anthropic 风格 API（Claude 系列）、阿里云百炼（Qwen）、DeepSeek，以及自定义 endpoint。README 推荐 `gpt-5.4` / `qwen3.6-plus` / `claude4.6-sonnet` / `deepseek-v4-pro` 或同等工具调用能力的模型（自编程依赖强工具调用）。

### 3. 通过 Web 配置（推荐）

设备启动后开 SoftAP（SSID 形如 `esp-claw-<MAC后3字节>`，前缀由 `CONFIG_APP_WIFI_AP_SSID_PREFIX` 控制，默认 `esp-claw`）。连上后会被 captive DNS 重定向到配置页；也可直连 `http://<AP_IP>/`。在网页改 Wi-Fi / LLM / IM / 时区并保存（写 NVS）。改 LLM 后设备会 `app_claw_update_config` 热更运行时配置。

### 4. 通过串口（CLI）

部分项可在 Console 改；Wi-Fi 也可用 `register_wifi_command()` 注册的 wifi 命令。LLM/IM 凭据类敏感字段优先走 Web。

### 5. IM 平台凭据

`cap_im_platform` 在 `APP_CLAW_CAP_IM_<PLATFORM>` 启用时注册对应运行时组：
| 运行时组 | 事件源 | 文本 | 图片 | 文件 |
|---|---|---|---|---|
| `cap_im_feishu` | `feishu_gateway` | `feishu_send_message` | `feishu_send_image` | `feishu_send_file` |
| `cap_im_qq` | `qq_gateway` | `qq_send_message` | `qq_send_image` | `qq_send_file` |
| `cap_im_tg` | `tg_gateway` | `tg_send_message` | `tg_send_image` | `tg_send_file` |
| `cap_im_wechat` | `wechat_gateway` | `wechat_send_message` | `wechat_send_image` | 不支持 |
| `cap_im_local` | `local_gateway` | `local_send_message` | — | — |

- 启动时 app 按存在的凭据应用 App ID/Secret/Token；配共享 inbox 路径（`/fatfs/inbox/`）与最大附件尺寸
- `edge_agent` 把 outbound 通道 `qq/feishu/telegram/wechat/web` 绑定到对应 send 工具
- 微信走扫码登录（`cap_im_wechat_qr_login_*` API，`CONFIG_APP_CLAW_CAP_IM_WECHAT` 时 Web 暴露登录状态接口）
- 平台差异：飞书目标用 `chat_id` 或 `ou_` 开头的 `open_id`；QQ 用 `c2c:<openid>` 或 `group:<group_openid>`；Telegram 用数字 chat id（如 `-100...`）

### 6. 记忆模式（编译期）

`(Top)` → `App Claw Config` → `App Claw memory management mode`（依赖 `APP_CLAW_CAP_MEMORY`）：
- **Structured memory management**（full，默认）：结构化记录 + summary-tag 目录；`memory_recall` 按需召回；LLM 可见组加 `claw_memory`；注入 summary-tag 目录而非整篇 `MEMORY.md`
- **Lightweight memory**：跳过结构化抽取，直接注入 `MEMORY.md` 文本；适合内存/上下文受限场景

> full 模式下 `MEMORY.md` **不是**检索源真相，检索/更新基于 `memory_records.jsonl` + `memory_index.json`。

### 7. 编译期 vs 运行时

| 类型 | 例子 | 怎么改 |
|---|---|---|
| 运行时（NVS 覆盖） | Wi-Fi、LLM、IM 凭据、时区、search allowlist | Web / 串口保存 |
| 纯编译期（重编生效） | `APP_CLAW_MEMORY_MODE`、`APP_CLAW_ENABLE_EMOTE`、各 `APP_CLAW_CAP_*` / `APP_CLAW_LUA_*`、`CLAW_CORE_STAGE_VERBOSITY` | `idf.py menuconfig` → build → flash |

### 8. agent_stage 进度推送

`(Top)` → `Component config` → `ESP-Claw Core` → `Agent stage notification verbosity`：
- `Simple`（默认）：不向路由层发 `agent_stage`
- `Verbose`：含工具调用轮的 `agent_stage` 事件发给路由器，配合 `agent_stage_im_notify` 规则推到 IM

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ask` 不可用 / core 不启动 | LLM 三项之一为空 | Web 填齐 `api_key`/`model`/`backend_type` 保存，重启 |
| 改 menuconfig 默认值不生效 | 被 NVS 覆盖 | 清 NVS 键或 Web 重写保存 |
| SoftAP 配置页打不开 | captive DNS 起不来（日志 WARN） | 直连 `http://<AP_IP>/`；或换端口/固件 |
| IM 收不到回复 | 凭据未配 / 无 `out_message` 规则 | 配齐凭据；见 `recipes/router_rules.md` 第 5 步 |
| 微信不在线 | 未扫码登录 | Web 触发扫码，查 `cap_im_wechat_qr_login_status` |
| 记忆「不记得」 | 用 lightweight 模式或 `MEMORY.md` 当真相 | full 模式检索看 `memory_records.jsonl`/`memory_index.json` |
| 导出配置泄密 | 含 token/密钥 | 不要公开分享导出配置或 NVS dump |

## 参考

- `application/edge_agent/components/app_config/app_config.c`
- `application/edge_agent/main/main.c`（`main_copy_claw_to_app_config` / `main_save_claw_config`）
- `components/common/app_claw/Kconfig`（`APP_CLAW_MEMORY_MODE_*`、`APP_CLAW_CAP_IM_*`）
- `application/edge_agent/main/Kconfig.projbuild`（`APP_WIFI_*`、`APP_SEARCH_HTTP_ALLOWLIST`）
- `docs/src/content/docs/en/reference-project/configuration.mdx`、`tutorial/web-config.mdx`
- `docs/src/content/docs/en/reference-cap/cap-im-platform.mdx`
