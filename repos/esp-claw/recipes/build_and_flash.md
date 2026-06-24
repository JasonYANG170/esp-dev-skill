# 从源码构建并烧录 edge_agent

> **适用摘要**: 在本地用 ESP-IDF v5.5.4 + ESP Board Manager 编译 ESP-Claw `edge_agent`，选择开发板、调整 menuconfig、烧录并进入串口 Console。

## 触发意图

- "编译 ESP-Claw"
- "build esp-claw from source"
- "idf.py bmgr 选板子"
- "烧录 edge_agent"
- "flash monitor 看启动日志"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.5.4 已安装并 `. $IDF_PATH/export.sh` |
| 工具 | `pip install esp-bmgr-assist` |
| 源码 | `git clone https://github.com/espressif/esp-claw.git` |
| 参考 | `application/edge_agent/README.md`、`docs/.../reference-project/build-from-source.mdx` |

## 分步说明

### 1. 进应用目录并选板

```bash
cd esp-claw/application/edge_agent

# 列出支持的板子
idf.py bmgr -c ./boards -l

# 选一块（自动按板子选芯片 target，无需 idf.py set-target）
idf.py bmgr -c ./boards -b esp32_S3_DevKitC_1
```

> 板子定义在 `application/edge_agent/boards/<vendor>/<board>/`。改板子只需重跑 `bmgr`。

### 2. 可选：调整 menuconfig

```bash
idf.py menuconfig
```

常用入口：
- `(Top)` → `App Claw Config`
  - `App Claw memory management mode`：`Structured memory management`（full，默认）/ `Lightweight memory`
  - `Enable App Claw emote`（依赖 `ESP_BOARD_DEV_DISPLAY_LCD_SUPPORT`）
  - `App Claw Capabilities` / `App Claw Lua Modules`：按需裁剪 `APP_CLAW_CAP_*` / `APP_CLAW_LUA_*`
- `(Top)` → `Component config` → `ESP-Claw Core` → `Agent stage notification verbosity`：`Simple`（默认）/ `Verbose`（让路由收到 `agent_stage` 事件）
- `(Top)` → `Component config` → `ESP System Settings` → `Channel for console output`：对齐板子硬件

> 运行时可改的字段（Wi-Fi/LLM/IM 等）以 NVS 为准，menuconfig 默认值可能被覆盖——见 `recipes/.../configuration` 与 `resources/config_reference.md`。

### 3. 构建与烧录

```bash
idf.py build
idf.py flash monitor
```

### 4. 启动后用串口 Console 验证

```sh
help                       # 看全部命令
cap list                   # 列出已注册能力
cap call get_current_time '{}'
auto rules                 # 当前 router_rules.json
scheduler --list
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `idf.py set-target` 后板子外设不对 | 跳过了 Board Manager | 用 `idf.py bmgr -c ./boards -b <board>`，它会自动选 target |
| `ask` 无响应 | `llm_api_key`/`llm_model`/`llm_backend_type` 未配齐，`claw_core` 不启动 | Web 配置页填齐三项并保存（写 NVS），重启设备 |
| `bmgr` 命令找不到 | 未装 `esp-bmgr-assist` | `pip install esp-bmgr-assist` 后重新 export IDF |
| 串口有日志但打不了字 | 端口选错（UART vs USB-Serial JTAG） | 换另一个 USB 口；或编 `Serial JTAG` 固件并改 `Channel for console output` |
| 改了 menuconfig 默认值不生效 | 该字段已被 NVS 覆盖 | 清除对应 NVS 键或用 Web 重写保存 |

## 参考

- `application/edge_agent/README.md`
- `application/edge_agent/sdkconfig.defaults`、`main/Kconfig.projbuild`
- `components/common/app_claw/Kconfig`（能力 / Lua 模块开关）
- `docs/src/content/docs/en/reference-project/build-from-source.mdx`
- `docs/src/content/docs/en/reference-project/configuration.mdx`
