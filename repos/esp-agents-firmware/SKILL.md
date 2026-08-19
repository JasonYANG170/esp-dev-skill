---
name: esp-agents-firmware-skill
description: >-
  AI Skill for Espressif ESP Private Agents device-side firmware development. Used when users need to
  build, customise, or debug firmware that talks to the ESP Private Agents AI platform, including
  voice chat agents, Matter Controller / Thread Border Router agents, local tool development,
  board customisation, device setup (Wi-Fi provisioning + agent ID/token), and audio pipeline
  tuning. Targets ESP32-S3 based boards (ESP-BOX-3, ESP-VoCat, M5Stack CoreS3) using ESP-IDF.
  Trigger words: "ESP Private Agents", "esp-agents-firmware", "ESP Agent", "voice chat agent",
  "Matter Controller", "Thread Border Router", "ESP-BOX-3", "ESP-VoCat", "M5Stack CoreS3",
  "AI Agent firmware", "ESP 智能体", "ESP 语音助手", "嘉立创", "智能体固件", "语音对话", "Matter 控制"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-agents-firmware-skill

面向 `esp-agents-firmware` 仓库的 AI Skill。该仓库是 ESP Private Agents（<https://agents.espressif.com>）AI 智能体平台的**设备端固件 SDK 与示例**，提供与云端 Agent 通过 WebSocket 通信、本地工具调用、语音录制/播放、显示、以及 RainMaker 配网的完整能力。本 Skill 提供场景化 recipes、真实 API 速查、配置参考与常见陷阱，帮助 AI Agent 正确开发基于该 SDK 的固件。

## Core Principles

1. **绝不臆造 API** — 所有函数名、结构体、宏、配置项必须可在 `resources/` 或仓库源码中查证；查不到即视为不存在，直接省略。
2. **设备端 Agent 通信走 WebSocket** — `components/agent` 通过 `esp_websocket_client` 与 ESP Private Agents 后端通信，所有交互以 `esp_agent_event_t` 事件回调形式上报，应用层只处理事件，不直接读 socket。
3. **`app_agent_*` 是应用层封装** — `examples/common/app_common` 中的 `app_agent_init/start/connect` 等封装了裸 `esp_agent_*` 与 setup/音频联动；新示例应优先复用 `app_*` 封装而非直接调底层。
4. **初始化顺序固定** — `app_main` 必须按：event loop + NVS → `agent_console_init` → `esp_board_manager_init` → `app_display_init` → `app_audio_init` → `app_device_init` → `app_agent_init` → `app_tools_register` → `app_agent_start` 的顺序执行（Matter 示例最后再调 `matter_controller_start_task`）。顺序错乱会导致句柄为空、事件丢失或硬件未就绪。
5. **Agent 启动依赖三个前置条件** — 网络连通 + agent_id 已设 + refresh_token 已设，三者全部满足后 `agent_setup` 才会抛出 `AGENT_SETUP_EVENT_START`，应用据此在任务中 `esp_agent_start()`。在未就绪时调用 `esp_agent_start()` 会失败。
6. **本地工具通过 `esp_agent_register_local_tool` 注册** — 工具签名固定为 `esp_agent_tool_handler_t`，参数仅支持 NUMBER/STRING/BOOL 三种类型；`*result` 必须 **heap 分配**（`strdup`/`malloc`），框架在发完响应后内部 `free`，栈指针或字面量会引发崩溃。
7. **工具必须固件与 Agent 配置双向对应** — 仅在固件里注册工具不够，还必须在 ESP Private Agents Dashboard 的 `agent_config.json`（`toolConfiguration.tools`）里声明同名工具及 `inputSchema`；服务端不知道的工具永远不会被调用。
8. **音频配置在初始化时一次性设定** — 上行/下行音频的 `format`、`sample_rate`、`frame_duration` 由 `app_agent_init` 内部从 `CONFIG_AUDIO_*` 宏写入 `esp_agent_config_t`，默认上行 OPUS 8kHz/20ms、下行 OPUS 16kHz/60ms。改这些值要同步修改 Kconfig 与云端 Agent。
9. **配网三选一** — 设备首次配网可经 ESP RainMaker Home App（BLE Provisioning，默认）、串口 `set-wifi <ssid> <passphrase>`、或预置 token/agent_id。`set-token` / `set-agent` 串口命令用于手动配置自定义部署。
10. **板级差异在 `board_defs.h`** — LCD 镜像、背光类型、电容触摸通道、`BOARD_DEVICE_MANUAL_URL`、指示灯设备名等都在各板的 `examples/common/boards/<board>/board_defs.h` 定义；新板需建子目录并经 `idf.py select-board` 选择。
11. **状态机驱动一切交互** — `app_device` 维护 `SLEEP/ACTIVE/LISTENING` 系统状态与 `DEVICE_EVENT_*` 事件队列；唤醒词 "Hi, ESP" 或点按屏幕触发 `DEVICE_EVENT_WAKEUP`，15 秒无活动进入 `SLEEP`。自定义交互应通过 `app_device_event_enqueue` 投递事件，而不是绕过状态机直接驱动外设。
12. **Matter 控制器是纯 client-only** — `CONFIG_ESP_CLIENT_ONLY_MATTER_CONTROLLER=y` 要求关闭 Matter server/commissioner、Wi-Fi/Thread commissioning driver 与 CHIPOBLE；它依赖 RainMaker REST API，且只能控制 OnOff/LevelControl/ColorControl 三个 cluster。

## When to Use

**Applicable:**
- 基于 `esp-agents-firmware` 创建/修改 voice_chat 或 matter_controller 示例固件
- 开发新的**本地工具**（local tool）并在固件 + Agent 配置中注册
- 适配新硬件板（在 `examples/common/boards/` 下新增板级配置）
- 接入自定义 ESP Private Agents 部署（修改 API endpoint、设置 token/agent_id）
- 调试 Agent 连接、语音收发、事件回调、状态机不工作等问题
- 配置音频上下行采样率/帧长、AEC、播放音量
- Matter 设备列表获取与控制（OnOff / LevelControl / ColorControl）
- 添加串口 console 命令、配置 RainMaker 参数（如音量）

**Not applicable:**
- 在 ESP Private Agents **云端 Dashboard** 创建/编排 Agent 的操作（那是平台侧，非本仓库）
- 非 ESP-IDF / 非该 SDK 架构的通用 ESP32 开发
- 训练或微调 LLM / 语音模型（设备端只消费模型，不训练）
- PCB 原理图设计与硬件布线
- 其他厂商的 AI 硬件平台

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下方某个场景时，**先读对应 recipe** — 它包含完整调用链、分步说明与常见错误。

### 项目与构建

| recipe | scenario |
|---|---|
| `recipes/build_and_flash.md` | 选择板子、构建、烧录、查看串口日志（voice_chat / matter_controller 通用） |
| `recipes/add_custom_board.md` | 在 `examples/common/boards/` 新增自定义板级配置 |
| `recipes/custom_deployment.md` | 切换到自定义 ESP Private Agents 部署（API endpoint + token） |
| `recipes/headless_no_display.md` | 无显示屏板适配：注释 `app_display_init` / `app_display_*` 调用、配网二维码改由包装提供 |

### Agent 通信与事件

| recipe | scenario |
|---|---|
| `recipes/agent_init_and_events.md` | 初始化 Agent、注册事件回调、处理文本/语音/思考/错误事件 |
| `recipes/device_setup_provisioning.md` | 设备首次配网（Wi-Fi、agent_id、refresh_token）与重置 |

### 本地工具开发

| recipe | scenario |
|---|---|
| `recipes/local_tool_register.md` | 注册/注销本地工具，实现 `esp_agent_tool_handler_t` |
| `recipes/builtin_tools.md` | 使用内置工具：set_reminder / get_local_time / set_volume / set_emotion |

### Matter 控制器

| recipe | scenario |
|---|---|
| `recipes/matter_controller_control.md` | Matter 设备列表获取与 OnOff/LevelControl/ColorControl 控制（工具层） |
| `recipes/matter_controller_service_init.md` | `esp.service.matter-controller` 服务初始化：6 参数 / MTCtlStatus 位图 / NOC 签发 / UpdateDeviceList / `matter_controller_enable` + Thread BR + `matter_controller_client` |

### 音频与外设

| recipe | scenario |
|---|---|
| `recipes/audio_pipeline.md` | 音频管线（录音/播放）初始化、音量、上下行采样率配置 |
| `recipes/console_commands.md` | 注册自定义串口命令与使用默认命令（set-token/set-agent/set-wifi） |

---

## Supported Boards（板级支持）

| Board 标识 | 硬件 | 用途 | 是否支持 Thread Border Router |
|---|---|---|---|
| `esp_vocat_board_v1_2` | ESP-VoCat Core Board v1.2 | 语音 Agent 通用 | 否 |
| `esp_box_3` | ESP-BOX-3 | 语音 Agent 通用 | 否 |
| `m5stack_cores3` | M5Stack CoreS3 | 语音 Agent / Matter 控制器 | 否 |
| `m5stack_cores3_h2_gateway` | M5Stack CoreS3 + M5Stack Module Gateway H2 | Matter 控制器 + Thread Border Router | 是 |

> 选择板子：`idf.py select-board --board <board>`。板子定义见 `examples/common/boards/<board>/`。

## Agent 事件参考

Agent 对上层以事件回调形式上报。事件 ID 与 `event_data`（`esp_agent_message_data_t`）对应关系如下：

| 事件 (`esp_agent_event_t`) | `event_data` 有效字段 | 含义 |
|---|---|---|
| `ESP_AGENT_EVENT_CONNECTED` | — | WebSocket 已连接，等待开始对话 |
| `ESP_AGENT_EVENT_DISCONNECTED` | — | 连接断开，应进入 SLEEP |
| `ESP_AGENT_EVENT_START` | `start.conversation_id` | 对话握手完成，Agent 已启动 |
| `ESP_AGENT_EVENT_STOP` | — | Agent 已停止 |
| `ESP_AGENT_EVENT_SPEECH_START` | — | 服务端开始下发语音 |
| `ESP_AGENT_EVENT_SPEECH_END` | — | 服务端语音下发结束 |
| `ESP_AGENT_EVENT_DATA_TYPE_TEXT` | `text.text` / `text.role` / `text.generation_stage` | 文本消息（用户或助手，推测态/终态） |
| `ESP_AGENT_EVENT_DATA_TYPE_THINKING` | `thinking.thought` | 助手思考过程（推理文本） |
| `ESP_AGENT_EVENT_DATA_TYPE_SPEECH` | `speech.data` / `speech.len` | 下行语音数据帧（喂给播放器） |
| `ESP_AGENT_EVENT_ERROR` | `error.error`（`ESP_AGENT_AUDIO_CONVERSATION_ERROR`） | 音频会话错误 |
| `ESP_AGENT_EVENT_INIT` / `_DEINIT` | — | 生命周期事件 |

## Device 状态机

`app_device` 维护系统状态与事件队列，驱动唤醒/聆听/播报/休眠流转：

```
                 DEVICE_EVENT_WAKEUP
   SLEEP  ───────────────────────────►  ACTIVE(LISTENING)
     ▲                                       │
     │ DEVICE_EVENT_SLEEP                    │ DEVICE_EVENT_SPEECH_START
     │ (15s 无活动 / 断连)                    ▼
     └──────────────────────────  播报中 (SPEECH_PLAYBACK_COMPLETE)
```

关键事件（`app_device_event_t`）：`SYSTEM_INITIALIZED`、`WAKEUP`、`SLEEP`、`SPEECH_START`、`SPEECH_END`、`SPEECH_PLAYBACK_COMPLETE`、`INTERRUPT`、`FACTORY_RESET`、`REMINDER`、`AGENT_STATE_CHANGED`、`SET_USER_TEXT`、`SET_ASSISTANT_TEXT`。

系统状态（`app_device_system_state_t`）：`APP_DEVICE_SYSTEM_STATE_SLEEP`、`_ACTIVE`、`_LISTENING`。

---

## Critical Pitfalls (Must Read)

下面是最高频的错误。违反任一条都会导致固件无法工作。

### 1. app_main 初始化顺序不能乱

```c
// ❌ WRONG — 在 app_agent_init 之前注册工具，agent_handle 还是 NULL
app_tools_register();
app_agent_init(&agent_config);

// ✅ CORRECT — 严格按仓库 app_main.c 的顺序
esp_event_loop_create_default();
nvs_flash_init();
agent_console_init();
esp_board_manager_init();
app_display_init();
app_audio_init();
app_device_init(&device_config);
app_agent_init(&agent_config);   // 此后才有 agent_handle
app_tools_register();            // 依赖 app_agent_init 内部句柄
app_agent_start();
```

### 2. 工具 result 必须堆分配

```c
// ❌ WRONG — 返回栈缓冲/字面量指针，框架 free 时崩溃
esp_err_t my_handler(..., char **result) {
    char buf[64];
    snprintf(buf, sizeof(buf), "ok");
    *result = buf;            // 栈地址，返回即失效
    return ESP_OK;
}

// ✅ CORRECT — 用 strdup/malloc 分配，框架发完会内部 free
esp_err_t my_handler(..., char **result) {
    *result = strdup("ok");   // heap 分配
    return ESP_OK;
}
```

### 3. 工具仅支持三种参数类型

```c
// ❌ WRONG — 期望拿到数组/对象参数（框架不会解析）
if (params[i].type == ???) { params[i].value.arr; }   // 不存��

// ✅ CORRECT — 只处理 NUMBER/STRING/BOOL
for (size_t i = 0; i < num_params; i++) {
    if (strcmp(params[i].name, "volume") == 0 &&
        params[i].type == ESP_AGENT_PARAM_TYPE_NUMBER) {
        volume = (int)params[i].value.i;
    } else if (strcmp(params[i].name, "emotion_name") == 0 &&
               params[i].type == ESP_AGENT_PARAM_TYPE_STRING) {
        emotion = params[i].value.s;
    }
}
```

### 4. 工具必须在固件 + Agent 配置两侧都声明

```c
// ❌ WRONG — 只在固件注册，Dashboard agent_config.json 没有同名 tool，永远不会被调用
app_agent_register_tool("my_tool", my_handler, NULL);

// ✅ CORRECT — 同时在 examples/<ex>/agent_config.json 的 toolConfiguration.tools[]
// 里声明 { "type":"local", "name":"my_tool", "inputSchema":{...} }
// 然后在固件注册同名 handler
app_agent_register_tool("my_tool", my_handler, NULL);
```

### 5. agent_start 必须等三个前置条件

```c
// ❌ WRONG — 在 app_main 里直接调 esp_agent_start，此时网络/token 可能未就绪
app_agent_init(&cfg);
esp_agent_start(handle, NULL);   // 失败：尚未配网

// ✅ CORRECT — 由 setup 在 AGENT_SETUP_EVENT_START 事件里启动
// app_agent.c 的 agent_setup_event_handler 在收到 START 时
// xTaskCreate(app_agent_start_task, ...) 在任务里读 agent_id/refresh_token
// 再调 esp_agent_set_agent_id / set_refresh_token / app_agent_connect()
```

### 6. 默认 agent_id 为空时设备无法自动连接

```bash
# ❌ WRONG — 只配网不设 agent_id，agent_setup_get_agent_id() 返回空，启动任务直接返回
set-wifi MySSID MyPass

# ✅ CORRECT — 配网后必须再设 agent_id（与 refresh token）
# 方式一：menuconfig 设 CONFIG_AGENT_SETUP_DEFAULT_AGENT_ID
# 方式二：串口 set-agent <agent_id> 与 set-token <refresh_token>
# 方式三：ESP RainMaker Home App 扫 Share Agent 二维码自动下发
set-agent <agent_id>
set-token <refresh_token>
```

### 7. 切换自定义部署必须改 endpoint

```bash
# ❌ WRONG — 自建 AWS 部署但固件仍指向公共 api.agents.espressif.com
idf.py build flash monitor   # 连的是公共云，token 无效

# ✅ CORRECT — menuconfig 修改 ESP_AGENT_API_ENDPOINT
idf.py menuconfig
# → ESP Agent Config → ESP Private Agents API Endpoint = my.aws.endpoint
idf.py build flash monitor
# 然后串口设 set-token / set-agent
```

### 8. 语音发送前必须确认 Agent 已 STARTED

```c
// ❌ WRONG — 状态未到 STARTED 就发语音，app_agent_send_speech 返 INVALID_STATE
app_agent_send_speech(buf, len);   // state == DISCONNECTED

// ✅ CORRECT — 先判状态
if (app_agent_get_state() == APP_AGENT_STATE_STARTED) {
    app_agent_send_speech(buf, len);
}
// 或在收到 ESP_AGENT_EVENT_START 之后再驱动麦克风
```

### 9. Matter client-only 配置互斥

```
# ❌ WRONG — 同时启用 client-only controller 与 Matter server/commissioner/BLE
CONFIG_ESP_CLIENT_ONLY_MATTER_CONTROLLER=y
CONFIG_ESP_MATTER_ENABLE_MATTER_SERVER=y     # 冲突，选项不可见/编译错

# ✅ CORRECT — client-only 模式下必须全部关闭以下项：
# ESP_MATTER_ENABLE_MATTER_SERVER、ESP_MATTER_COMMISSIONER_ENABLE、
# WIFI_NETWORK_COMMISSIONING_DRIVER、THREAD_NETWORK_COMMISSIONING_DRIVER、ENABLE_CHIPOBLE
CONFIG_ESP_CLIENT_ONLY_MATTER_CONTROLLER=y
```

### 10. control_device 的 cluster/command 必须匹配 schema

```c
// ❌ WRONG — cluster_id/command_id 与 agent_config.json 文档不符
matter_controller_control_device(&r, node_id, 6, 99, "{}");  // OnOff 无 command_id=99

// ✅ CORRECT — OnOff(6): TurnOff=0/TurnOn=1/Toggle=2，args="{}"
matter_controller_control_device(&r, node_id, 6, 1, "{}");   // TurnOn
// LevelControl(8): MoveToLevel=0, args={"0:U8":<level>,"1:U16":0,"2:U8":0,"3:U8":0}
matter_controller_control_device(&r, node_id, 8, 0, "{\"0:U8\":200,\"1:U16\":0,\"2:U8\":0,\"3:U8\":0}");
```

### 11. board_defs.h 决定外设可用性

```c
// ❌ WRONG — 在不支持电容触摸的板上依赖 CAPACITIVE_TOUCH_SUPPORTED
app_capacitive_touch_init();   // 板子未定义该宏 → 行为未定义

// ✅ CORRECT — 用宏保护，参考 esp_vocat_board_v1_2/board_defs.h
#if defined(CAPACITIVE_TOUCH_SUPPORTED) && CAPACITIVE_TOUCH_SUPPORTED
    app_capacitive_touch_init();
#endif
```

### 12. event handler 转发不能省

```c
// ❌ WRONG — 自定义 event_handler 不转发，默认状态机/音频联动全部失效
void app_event_handler(void *a, esp_event_base_t b, int32_t id, void *d) {
    my_custom_logic(id);   // 忘记转发
}

// ✅ CORRECT — 自定义 handler 必须把事件交给默认 handler
void app_event_handler(void *a, esp_event_base_t b, int32_t id, void *d) {
    app_agent_default_event_handler(a, b, id, d);  // 必须转发
    my_custom_logic(id);
}
```

### 13. 无显示屏设备需注释显示调用

```c
// ❌ WRONG — 无屏板仍调用 app_display_init，初始化失败卡死
app_display_init();

// ✅ CORRECT — 按 docs/example_customisation.md 注释掉 display 相关
// 在 app_main.c 中注释 app_display_init() 及所有 app_display_* 调用
// 并在 device callbacks 里去掉 app_display_* 调用
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确目标：新建示例 / 加工具 / 适配板 / 改部署；定位对应 recipe |
| 2 | Recipe | 在 `recipes/` 找匹配场景，按其调用链与分步说明执行 |
| 3 | Query | recipe 未覆盖的 API/配置查 `resources/`（api_reference / config_reference） |
| 4 | Validate | 核对：初始化顺序、头文件包含、`agent_config.json` 工具声明、Kconfig 值、板级宏 |
| 5 | Confirm | 向用户呈现方案：包含哪些组件、初始化顺序、工具清单、配置项 |
| 6 | Execute | **新建示例**：拷贝最近示例（`examples/voice_chat` 或 `examples/matter_controller`）再改；**已有项目**：就地编辑 |
| 7 | Build | `idf.py select-board --board <board>` → `idf.py build flash monitor` |
| 8 | Provision | 配网：RainMaker Home App 或串口 `set-wifi`/`set-token`/`set-agent` |
| 9 | Verify | 串口日志确认 `Agent Started`、唤醒词 "Hi, ESP" 触发聆听、语音往返正常 |

### Step 6 Detail — 示例创建策略

**目标目录无现有项目（首次创建）：**

1. **选最近示例**作为起点：
   - 通用语音 Agent / 朋友型助手 → `examples/voice_chat/`
   - Matter 控制 / Thread Border Router → `examples/matter_controller/`
2. **整目录拷贝**到用户工程目录，保留 `main/`、`CMakeLists.txt`、`idf_component.yml` 结构。
3. **就地修改**：`app_main.c` 初始化逻辑、`app_tools.c` 工具集、`main/idf_component.yml` 依赖、`agent_config.json` 工具声明。
4. **说明改动**：哪些是拷贝来的、哪些被改、为什么。

**目标目录已有项目：** 就地编辑，除非用户要求否则不覆盖。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/` 查不到 | 停下，告知用户该 API 不存在，不要凭印象编造 |
| Agent 一直不 START | 检查三前置条件：Wi-Fi 已连、`agent_setup_get_agent_id()` 非空、refresh_token 非空 |
| 工具不被调用 | 检查 `agent_config.json` 是否声明同名 tool 与 inputSchema；参数类型是否仅 NUMBER/STRING/BOOL |
| 连接公共云 token 无效 | 确认是否用自定义部署，需改 `CONFIG_ESP_AGENT_API_ENDPOINT` |
| 语音卡顿/不收音 | 检查 `CONFIG_AUDIO_UPLOAD_SAMPLE_RATE`(默认 8000)/`FRAME_DURATION`(20)、AEC（`CONFIG_ENABLE_AEC`）是否与板子硬件匹配 |
| Matter 控制无效 | 确认 client-only 互斥项已关、`matter_controller_start_task` 已在 `app_agent_start` 之后调用、目标 cluster ∈ {OnOff, LevelControl, ColorControl} |
| 自定义板外设异常 | 检查 `board_defs.h` 是否定义所需宏（LCD_MIRROR_X_Y / CAPACITIVE_TOUCH_* / LEDC_BACKLIGHT_SUPPORTED 等） |

## References

- 场景化 recipes → `recipes/` 目录
- Agent / Setup / Audio / Display API 速查 → `resources/api_reference.md`
- Kconfig / menuconfig 配置参考 → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 示例与板级清单 → `resources/example_list.md`
- 仓库根 README → `espressif-repos/esp-agents-firmware/README.md`
- 仓库 docs → `espressif-repos/esp-agents-firmware/docs/{agent,board,deployment,example}_customisation.md`
