# 初始化 Agent 与处理事件

> **适用摘要**: 初始化 Agent、注册事件回调、处理文本/语音/思考/错误事件，理解 `app_agent_*` 封装与底层 `esp_agent_*` 的关系。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-agents-firmware/resources/`, source/examples in `repos/esp-agents-firmware/`, and this recipe path `repos/esp-agents-firmware/recipes/agent_init_and_events.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "初始化 Agent"
- "处理 Agent 事件"
- "注册事件回调"
- "接收语音/文本"
- "esp_agent_register_event_handler"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考源码 | `examples/common/app_common/src/app_agent.c`、`examples/voice_chat/main/app_main.c` |
| 组件 | `components/agent`（含 `esp_agent.h` 聚合头） |

## 分步说明

### 1. 推荐路径：用 app_agent 封装

仓库示例统一用 `app_agent_init/start/connect`，它内部已封装 setup 联动、音频配置与事件分发。`app_main.c` 标准模式：

```c
#include <app_agent.h>
#include <app_device.h>

void app_event_handler(void *arg, esp_event_base_t event_base, int32_t event_id, void *event_data)
{
    /* 必须转发给默认 handler，否则状态机/音频联动失效 */
    app_agent_default_event_handler(arg, event_base, event_id, event_data);
}

void app_main(void)
{
    /* ... 前置初始化（见 SKILL.md 执行流程）... */

    app_agent_config_t agent_config = {
        .event_handler = app_event_handler,
    };
    app_agent_init(&agent_config);   /* 内部调 agent_setup_init + esp_agent_init + 注册事件 */
    app_tools_register();
    app_agent_start();               /* 内部调 setup_rainmaker_init + agent_setup_start */
}
```

`app_agent_init`（来自 `app_agent.c`）内部：
1. `agent_setup_init()` 读 NVS 中的 agent_id / refresh_token
2. 用 `CONFIG_AUDIO_UPLOAD/DOWNLOAD_SAMPLE_RATE`、`FRAME_DURATION_MS` 构造上下行 `esp_agent_audio_config_t`（格式均 `ESP_AGENT_CONVERSATION_AUDIO_FORMAT_OPUS`）
3. `esp_agent_init(&agent_config)`（`conversation_type = ESP_AGENT_CONVERSATION_SPEECH`）拿到 `agent_handle`
4. `esp_agent_register_event_handler(handle, ESP_EVENT_ANY_ID, handler, ...)` 注册应用事件

### 2. 默认事件处理器已做的事

`app_agent_default_event_handler`（来自 `app_agent.c`）已自动处理以下事件，**自定义 handler 必须转发给它**：

| 事件 | 默认行为 |
|---|---|
| `ESP_AGENT_EVENT_CONNECTED` | 状态置 `APP_AGENT_STATE_CONNECTED` |
| `ESP_AGENT_EVENT_DISCONNECTED` | 置 `DISCONNECTED` + 投递 `DEVICE_EVENT_SLEEP` |
| `ESP_AGENT_EVENT_SPEECH_START` | 投递 `DEVICE_EVENT_SPEECH_START` |
| `ESP_AGENT_EVENT_SPEECH_END` | 投递 `DEVICE_EVENT_SPEECH_END` |
| `ESP_AGENT_EVENT_DATA_TYPE_TEXT` | 按 `role` 投递 `SET_USER_TEXT`/`SET_ASSISTANT_TEXT`（仅非 FINAL 阶段） |
| `ESP_AGENT_EVENT_DATA_TYPE_SPEECH` | `app_audio_play_speech(data, len)` 喂播放器 |
| `ESP_AGENT_EVENT_DATA_TYPE_THINKING` | 打印思考文本（灰色） |
| `ESP_AGENT_EVENT_ERROR` | 若是 `ESP_AGENT_AUDIO_CONVERSATION_ERROR` 则 `esp_agent_stop` |
| `ESP_AGENT_EVENT_START` | 状态置 `APP_AGENT_STATE_STARTED` |

### 3. 直接路径：裸 esp_agent API

若不使用 app 封装，需自行初始化、配音频、注册事件、并在 setup 就绪后启动。关键 API（来自 `components/agent/include/`）：

```c
#include <esp_agent.h>

esp_agent_audio_config_t up = {
    .format = ESP_AGENT_CONVERSATION_AUDIO_FORMAT_OPUS,
    .sample_rate = CONFIG_AUDIO_UPLOAD_SAMPLE_RATE,
    .frame_duration = CONFIG_AUDIO_UPLOAD_FRAME_DURATION_MS,
};
esp_agent_audio_config_t down = {
    .format = ESP_AGENT_CONVERSATION_AUDIO_FORMAT_OPUS,
    .sample_rate = CONFIG_AUDIO_DOWNLOAD_SAMPLE_RATE,
    .frame_duration = CONFIG_AUDIO_DOWNLOAD_FRAME_DURATION_MS,
};
esp_agent_config_t cfg = {
    .conversation_type = ESP_AGENT_CONVERSATION_SPEECH,
    .upload_audio_config = &up,
    .download_audio_config = &down,
    /* agent_id / refresh_token 可后置 set */
};

esp_agent_handle_t h = esp_agent_init(&cfg);
esp_agent_register_event_handler(h, ESP_EVENT_ANY_ID, my_handler, NULL, &inst);

/* 待网络 + agent_id + token 就绪后 */
esp_agent_set_agent_id(h, agent_id);
esp_agent_set_refresh_token(h, token);
esp_agent_start(h, NULL);          /* NULL = 新会话；传 conversation_id 可恢复 */
```

### 4. 解析事件数据

事件 `event_data` 类型为 `esp_agent_message_data_t`（联合体），按事件取字段：

```c
void my_handler(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    esp_agent_message_data_t *md = (esp_agent_message_data_t *)data;
    switch (id) {
    case ESP_AGENT_EVENT_DATA_TYPE_TEXT:
        /* md->text.text / md->text.role / md->text.generation_stage */
        break;
    case ESP_AGENT_EVENT_DATA_TYPE_SPEECH:
        /* md->speech.data / md->speech.len —— 喂音频播放器 */
        break;
    case ESP_AGENT_EVENT_DATA_TYPE_THINKING:
        /* md->thinking.thought */
        break;
    case ESP_AGENT_EVENT_START:
        /* md->start.conversation_id —— 可保存用于恢复 */
        break;
    case ESP_AGENT_EVENT_ERROR:
        /* md->error.error ∈ {ESP_AGENT_AUDIO_CONVERSATION_ERROR} */
        break;
    default: break;
    }
}
```

### 5. Agent 状态查询与语音发送

```c
/* 仅在 STARTED 后才允许发语音 */
if (app_agent_get_state() == APP_AGENT_STATE_STARTED) {
    app_agent_send_speech(buf, len);   /* 内部 pdMS_TO_TICKS(1000) 超时 */
}

/* 或底层（需自管句柄） */
esp_agent_send_speech(h, buf, len, pdMS_TO_TICKS(1000));
esp_agent_send_text(h, "hello", pdMS_TO_TICKS(1000));
```

`app_agent_state_t`：`DISCONNECTED(0)` / `CONNECTING` / `CONNECTED` / `STARTED`。

### 6. 开新会话 / 停止

```c
esp_agent_new_conversation(h);   /* 断开并清 conversation_id 后重连，服务端分配新 id（须先 start） */
esp_agent_stop(h);
esp_agent_deinit(h);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 自定义 handler 不触发联动 | 没转发给 `app_agent_default_event_handler` | 在自定义 handler 里调用默认 handler |
| `esp_agent_start` 返回错误 | 三前置条件未满足 | 网络通 + agent_id + token 全部就绪后再 start |
| 语音发不出 | state 未到 STARTED | 用 `app_agent_get_state()` 判断后再发 |
| 重复初始化 | 多次 `app_agent_init` | `app_agent_init` 内部有 `initialized` 标志，重复调返回 `INVALID_STATE` |
| `agent_handle` 为 NULL | `esp_agent_init` 失败 | 检查 config 字段与内存 |

## 参考

- `components/agent/include/esp_agent_core.h` — `esp_agent_config_t`、`esp_agent_init/start/stop`
- `components/agent/include/esp_agent_events.h` — 事件枚举与 `esp_agent_message_data_t`
- `components/agent/include/esp_agent_messages.h` — `esp_agent_send_speech/send_text`
- `components/agent/include/esp_agent_tools.h` — 工具相关（见 `recipes/local_tool_register.md`）
- `examples/common/app_common/src/app_agent.c` — app 封装与默认事件处理
- `examples/voice_chat/main/app_main.c` — 标准 app_main 模式
