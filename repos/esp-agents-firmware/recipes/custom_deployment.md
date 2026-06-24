# 切换到自定义 ESP Private Agents 部署

> **适用摘要**: 把固件从公共部署（`api.agents.espressif.com`）切换到自建 AWS 部署，并配置 token 与 agent_id。

## 触发意图

- "自定义部署"
- "自己的 AWS Agent 部署"
- "改 API endpoint"
- "自建 ESP Private Agents"
- "set-token / set-agent"

## 前置条件

| 条件 | 要求 |
|---|---|
| 自建部署 | 已部署 ESP Private Agents，拿到 API URL |
| Refresh token | 用户网站 → Profile → Copy Refresh Token |
| 示例已构建 | 见 `recipes/build_and_flash.md` |

## 分步说明

### 1. 修改 API endpoint

进入示例目录打开 menuconfig：

```bash
cd examples/voice_chat       # 或 matter_controller
idf.py menuconfig
```

导航到 `ESP Agent Config`（来自 `components/agent/Kconfig.projbuild`），修改：

```
ESP Private Agents API Endpoint (CONFIG_ESP_AGENT_API_ENDPOINT)
```

默认值 `api.agents.espressif.com` 改为自建部署 URL。该宏在 `esp_agent_core.h` 中被引用：

```c
#define ESP_AGENT_API_ENDPOINT CONFIG_ESP_AGENT_API_ENDPOINT
```

### 2. 重新构建并烧录

```bash
idf.py build flash monitor
```

### 3. 在设备上设置 token 与 agent_id

设备启动后在串口运行（命令来自 `components/setup/src/setup_console.c` 与 `agent_setup.c`）：

```bash
set-token <refresh_token>
set-agent <agent_id>
```

### 4. 配置 Wi-Fi

任选其一：

```bash
# 串口方式（命令来自 agent_setup.c 的 register_set_wifi_cli_handler）
set-wifi <ssid> <passphrase>
```

或用 ESP BLE Provisioning App（ESP RainMaker Home App）。

### 5. 验证

串口日志应出现 `app_agent: Agent Connected` → `app_agent: ESP Agent Started`，说明已连上自建部署。

## 默认 agent_id 的两种设置方式

| 方式 | 操作 | 来源 |
|---|---|---|
| 编译期默认 | menuconfig → `ESP Agent Setup Configuration` → `CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` 填值 | `components/setup/Kconfig.projbuild` |
| 运行时 | 串口 `set-agent <id>` 或 RainMaker Home App 扫 Share Agent 二维码 | `setup_console.c` / RainMaker |

> `CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` 默认为空字符串；为空时设备需运行时设置才能启动 Agent。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| token 无效 | 仍连公共云 | 确认 `CONFIG_ESP_AGENT_API_ENDPOINT` 已改并重新 build |
| Agent 不 START | agent_id 为空 | `set-agent` 或设 `CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` |
| `AGENT_SETUP_EVENT_START` 不触发 | 三前置条件未全满足 | Wi-Fi 已连 + agent_id 非空 + refresh_token 非空，缺一不可 |
| 设备名不对 | 想改 RainMaker 中的设备名 | menuconfig → `CONFIG_AGENT_SETUP_RAINMAKER_DEVICE_NAME`（默认 "Espressif AI Agent"） |

## 参考

- `docs/deployment_customisation.md` — 官方自定义部署步骤
- `components/agent/Kconfig.projbuild` — `CONFIG_ESP_AGENT_API_ENDPOINT` 定义
- `components/setup/Kconfig.projbuild` — `CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` / `_RAINMAKER_DEVICE_NAME`
- `components/setup/src/setup_console.c` — `set-token` / `set-agent` 命令实现
- `recipes/device_setup_provisioning.md` — 配网全流程
