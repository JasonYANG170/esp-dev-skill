# 设备配网与首次设置

> **适用摘要**: 设备首次配网（Wi-Fi、agent_id、refresh_token）、Agent 启动条件、恢复出厂，覆盖 voice_chat 与 matter_controller 两种 App 路径。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-agents-firmware/resources/`, source/examples in `repos/esp-agents-firmware/`, and this recipe path `repos/esp-agents-firmware/recipes/device_setup_provisioning.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "配网"
- "首次设置设备"
- "set-wifi / set-token / set-agent"
- "RainMaker 配网"
- "恢复出厂 / factory reset"

## 前置条件

| 条件 | 要求 |
|---|---|
| 固件已烧录 | 见 `recipes/build_and_flash.md` |
| 手机 App | voice_chat：ESP RainMaker Home App；matter_controller：ESP RainMaker App（后续也将支持 Home App） |
| Agent 信息 | agent_id（从 ESP Private Agents Dashboard 获取）+ refresh token（用户网站 Profile → Copy Refresh Token） |

## 分步说明

### 1. 理解 Agent 启动的三前置条件

来自 `components/setup/include/agent_setup.h`：仅当以下**全部**满足时 `agent_setup` 才发 `AGENT_SETUP_EVENT_START`，应用据此在任务里启动 Agent：

1. 网络连通（`AGENT_SETUP_EVENT_NETWORK_CONNECTED`）
2. agent_id 已设（`AGENT_SETUP_EVENT_AGENT_ID_UPDATE` 会单独触发，但不保证其他条件已满足）
3. refresh_token 已设

> 含义：只配网不设 agent_id，Agent 永远不启动。

### 2. 方式 A：手机 App 配网（推荐）

**voice_chat** — 用 ESP RainMaker Home App（iOS / Android）：
1. App 授权蓝牙/定位
2. 设备上电后屏幕显示二维码
3. App 右上 "+" → 扫码 → 选 Wi-Fi → 输密码 → 完成

**matter_controller** — 用 ESP RainMaker App：
1. 扫设备二维码
2. 选 Wi-Fi → 选组（Home）→ 重新登录完成 Controller Configuration
3. 在设备页设 agent-id、agent name、Thread border、Update Device List

> 配网后还需设 agent_id（见步骤 4），否则 Agent 不启动。

### 3. 方式 B：串口命令配网

命令定义于 `components/setup/src/setup_console.c`（`set-token` / `set-agent`）与 `agent_setup.c`（`set-wifi`）：

```bash
set-wifi <ssid> <passphrase>     # 来自 register_set_wifi_cli_handler
set-token <refresh_token>        # 来自 setup_console.c
set-agent <agent_id>             # 来自 setup_console.c
```

这三个命令都经 `agent_console_register_command`（`components/agent_console`）注册，由 `setup_console_register_commands()` 在启动时挂上。

### 4. 设置 agent_id

四种方式：

| 方式 | 操作 | 说明 |
|---|---|---|
| 编译期默认 | menuconfig `CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` | 来自 `components/setup/Kconfig.projbuild`，默认空 |
| 串口 | `set-agent <id>` | 调 `agent_setup_set_agent_id()`，写 NVS 并发 `AGENT_SETUP_EVENT_AGENT_ID_UPDATE` |
| RainMaker Home App | 扫 Share Agent 二维码 | App 自动下发到选中设备 |
| RainMaker 参数 | `CONFIG_AGENT_SETUP_CREATE_RAINMAKER_DEVICE=y` 时 App 内可改 | 见 `setup/Kconfig.projbuild` |

### 5. API 层操作（编程式）

来自 `components/setup/include/agent_setup.h`：

```c
agent_setup_init();                              /* 启动前先 init，从 NVS 读 agent_id/token */
agent_setup_start();                             /* 启动配网/连云 */

char *id = agent_setup_get_agent_id();           /* 内部分配缓冲，勿 free */
char *tk = agent_setup_get_refresh_token();      /* 同上 */

agent_setup_set_agent_id("my-agent-id");         /* 写 NVS + 触发 AGENT_ID_UPDATE */
agent_setup_set_refresh_token("xxxxx");

agent_setup_factory_reset();                     /* 恢复出厂 */
```

RainMaker 侧 API（来自 `components/setup/include/setup/rainmaker.h`）：

```c
setup_rainmaker_init(BOARD_DEVICE_MANUAL_URL);   /* 创建 RainMaker 设备，manual URL 显示在 App */
setup_rainmaker_factory_reset();

/* 音量参数回调（agent 可经 set_volume 工具联动） */
setup_rainmaker_register_volume_callbacks(get_cb, set_cb);
setup_rainmaker_update_volume(75);
```

### 6. 恢复出厂

任选其一：
- 物理方式：长按屏幕顶部 10 秒（屏幕提示后松手）
- App 方式：RainMaker (Home) App → 设备设置 → Factory Reset
- 串口：调用 `agent_setup_factory_reset()`（若暴露命令）或重刷固件

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 配网后 Agent 不启动 | 缺 agent_id 或 token | `set-agent` / `set-token` 或设默认 |
| 扫码连不上 | App 权限未开 | 授权蓝牙 + 定位 |
| `set-wifi` 连不上 | SSID/密码错或 2.4G/5G 问题 | 确认 SSID/密码；ESP32 仅 2.4G |
| matter_controller 看不到设备 | 用错 App | 用 ESP RainMaker App（非 Home App），后续会合并支持 |
| 设备名想改 | 默认 "Espressif AI Agent" | menuconfig `CONFIG_AGENT_SETUP_RAINMAKER_DEVICE_NAME` |

## 参考

- `examples/voice_chat/setup_guide.md` — voice_chat 完整配网与使用
- `examples/matter_controller/setup_guide.md` — matter_controller 配网与 Thread Border Router
- `components/setup/include/agent_setup.h` — setup API
- `components/setup/include/setup/rainmaker.h` — RainMaker setup API
- `components/setup/src/setup_console.c` — `set-token` / `set-agent` 命令
- `components/setup/src/agent_setup.c` — `set-wifi` 命令与 NVS 存储
- `docs/agent_customisation.md` — 更新设备上的 Agent
