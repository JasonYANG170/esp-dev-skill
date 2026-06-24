# 构建与烧录示例固件

> **适用摘要**: 基于 `esp-agents-firmware` 构建并烧录 voice_chat 或 matter_controller 示例到支持的板子，查看串口日志。

## 触发意图

- "编译 voice_chat 示例"
- "烧录 matter_controller"
- "idf.py build flash monitor"
- "选择板子构建"
- "如何烧录 ESP Agent 固件"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.5.2+，须在 `release/5.5` 分支 |
| 示例目录 | `examples/voice_chat/` 或 `examples/matter_controller/` |
| 支持的板子 | `esp_vocat_board_v1_2` / `esp_box_3` / `m5stack_cores3` / `m5stack_cores3_h2_gateway` |

## 分步说明

### 1. 确认 IDF 版本

仓库 `examples/voice_chat/main/idf_component.yml` 要求 `idf: version '>=5.5'`。

```bash
idf.py --version
# 若低于 5.5.2，切换到 release/5.5 分支并 source 导出脚本
```

### 2. 选择板子

进入示例目录，用官方 `select-board` 命令选板（该命令会写入板级配置到工程）：

```bash
cd examples/voice_chat          # 或 examples/matter_controller
idf.py select-board --board esp_box_3
```

可选板标识（来自各示例 README.md）：

| 示例 | 支持的 board |
|---|---|
| `voice_chat` | `esp_vocat_board_v1_2`、`esp_box_3`、`m5stack_cores3` |
| `matter_controller` | `esp_vocat_board_v1_2`、`esp_box_3`、`m5stack_cores3`、`m5stack_cores3_h2_gateway` |

> Thread Border Router 需用 `m5stack_cores3_h2_gateway`（M5Stack CoreS3 + H2 Gateway Module）。

### 3. 构建、烧录、监控

```bash
idf.py build flash monitor
```

`monitor` 默认以 115200 波特率打开串口。看到 `app_agent: ESP Agent Started` 即表示 Agent 握手成功。

### 4. 首次配网（详见 `recipes/device_setup_provisioning.md`）

固件首次启动处于未配置状态。任选其一：
- ESP RainMaker Home App 扫描屏幕二维码（voice_chat）/ ESP RainMaker App（matter_controller）
- 串口：`set-wifi <ssid> <passphrase>` → `set-token <refresh_token>` → `set-agent <agent_id>`

### 5. 唤醒与对话

设备配网完成后，说 "Hi, ESP" 或点按屏幕唤醒，听到提示音后即可对话；15 秒无活动自动休眠。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `select-board` 命令不存在 | IDF 版本过低或未在示例目录 | 确认 IDF ≥ 5.5.2 并在 `examples/<ex>` 目录执行 |
| 编译报组件找不到 | `idf_component.yml` override 路径错 | 确认从仓库根拷贝了完整 `examples/common`、`components` |
| 烧录后屏幕不亮 | 板选错（如 H2 固件刷到无 H2 板） | 重新 `select-board` 选对应板 |
| `app_agent: Agent ID or refresh token not found` | 未设 agent_id/token | 串口 `set-agent`/`set-token` 或 menuconfig 设 `CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID` |
| 连接公共云 token 无效 | 用了自建部署但 endpoint 没改 | `idf.py menuconfig` → ESP Agent Config → 改 endpoint（见 `recipes/custom_deployment.md`） |

## 参考

- `examples/voice_chat/README.md` — voice_chat 运行步骤
- `examples/matter_controller/README.md` — matter_controller 运行步骤（含 Thread Border Router）
- `docs/board_customisation.md` — 选板与自定义板
- `recipes/device_setup_provisioning.md` — 配网细节
