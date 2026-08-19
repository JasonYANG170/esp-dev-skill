# 使用内置工具

> **适用摘要**: 使用仓库自带的内置本地工具 `set_reminder` / `get_local_time` / `set_volume` / `set_emotion`，理解其参数、行为与注册方式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-agents-firmware/resources/`, source/examples in `repos/esp-agents-firmware/`, and this recipe path `repos/esp-agents-firmware/recipes/builtin_tools.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "内置工具有哪些"
- "set_reminder 用法"
- "set_volume / 调音量"
- "set_emotion / 表情"
- "get_local_time"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考源码 | `examples/common/app_common/src/app_common_tools.c`、`examples/*/main/app_tools.c` |
| 已 init | `app_agent_init` 完成 |
| emote 分区 | `set_emotion` 需 `CONFIG_APP_EMOTE_PARTITION_LABEL`（默认 "anim_icon"）对应分区 |

## 分步说明

### 1. 注册内置工具

`examples/voice_chat/main/app_tools.c` 的 `app_tools_register()` 真实写法：

```c
#include "app_common_tools.h"   /* 来自 examples/common/app_common */
#include "app_agent.h"

esp_err_t app_tools_register(void)
{
    /* 公共内置工具 */
    app_agent_register_tool(TOOL_NAME_SET_REMINDER,   app_common_tools_set_reminder_handler,   NULL);
    app_agent_register_tool(TOOL_NAME_GET_LOCAL_TIME, app_common_tools_get_local_time_handler, NULL);
    app_agent_register_tool(TOOL_NAME_SET_VOLUME,     app_common_tools_set_volume_handler,     NULL);

    /* voice_chat 专属 */
    app_agent_register_tool("set_emotion", app_tools_set_emotion_handler, NULL);
    return ESP_OK;
}
```

工具名宏来自 `app_common_tools.h`：

```c
#define TOOL_NAME_SET_REMINDER    "set_reminder"
#define TOOL_NAME_GET_LOCAL_TIME  "get_local_time"
#define TOOL_NAME_SET_VOLUME      "set_volume"
```

`set_emotion` 字面量名 `"set_emotion"`（在各示例 `app_tools.c` 内）。

### 2. set_reminder — 设提醒

handler：`app_common_tools_set_reminder_handler`（来自 `app_common_tools.c`）

| 参数 | 类型 | 说明 |
|---|---|---|
| `task` | STRING | 提醒内容 |
| `timeout` | NUMBER | 多少秒后提醒 |

行为：用 `esp_timer_create` + `esp_timer_start_once(timeout * 1000000ULL)` 设单次定时器；到点回调把 task 经 `app_device_event_enqueue(DEVICE_EVENT_REMINDER, ...)` 投递给状态机，由设备播放提示音并显示。

`agent_config.json` 对应声明（已含 required `["task","timeout"]`）。

### 3. get_local_time — 取本地时间

handler：`app_common_tools_get_local_time_handler`

无参数。用 `time(NULL)` + `localtime` + `strftime("%Y-%m-%d %H:%M:%S")` 生成字符串返回。

### 4. set_volume — 设音量

handler：`app_common_tools_set_volume_handler`

| 参数 | 类型 | 说明 |
|---|---|---|
| `volume` | NUMBER | 0–100 |

行为：调 `app_audio_set_playback_volume(volume)`，内部写 NVS 持久化。RainMaker 侧可通过 `setup_rainmaker_update_volume()` / `setup_rainmaker_register_volume_callbacks()` 与 App 同步。

### 5. set_emotion — 设表情

handler：各示例 `app_tools_set_emotion_handler`

| 参数 | 类型 | 说明 |
|---|---|---|
| `emotion_name` | STRING | 表情名 |

有效表情名（来自 `app_display.h`，需与 emote 分区资源一致）：

```c
DISP_EMOTE_NEUTRAL   "neutral"
DISP_EMOTE_HAPPY     "happy"
DISP_EMOTE_SAD       "sad"
DISP_EMOTE_CRYING    "crying"
DISP_EMOTE_ANGRY     "angry"
DISP_EMOTE_SLEEPY    "sleepy"
DISP_EMOTE_CONFUSED  "confused"
DISP_EMOTE_SHOCKED   "shocked"
DISP_EMOTE_WINKING   "winking"
DISP_EMOTE_IDLE      "idle"
```

行为：先 `app_display_is_emotion_valid(emotion)` 校验，再 `app_display_set_emotion(emotion)`。无效返回 `ESP_ERR_INVALID_ARG`。

### 6. agent_config.json 中的声明

四个工具在 `examples/voice_chat/agent_config.json`（与 matter_controller 的 `agent_config.json`）中均有完整 `inputSchema` 声明，新增/改动工具时以之为模板。

## 各示例工具集对照

| 工具 | voice_chat | matter_controller |
|---|---|---|
| `set_reminder` | ✓ | ✓ |
| `get_local_time` | ✓ | ✓ |
| `set_volume` | ✓ | ✓ |
| `set_emotion` | ✓ | ✓ |
| `get_device_list` | — | ✓ |
| `control_device` | — | ✓ |
| 远程 `esp_rainmaker` MCP | ✓（remote_fetch） | — |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| set_emotion 无效表情 | LLM 传了不在列表的名字 | handler 内 `app_display_is_emotion_valid` 校验已兜底，返回错误字符串 |
| 提醒不触发 | task/timeout 缺失或 timeout ≤ 0 | handler 已校验，确认 LLM 传齐 required |
| 音量改了不持久 | 没走 `app_audio_set_playback_volume` | 该函数内部写 NVS；勿绕过 |
| 时间不对 | 设备未同步时间 | 确认已连网并经 SNTP 同步（IDF 默认） |
| 表情资源缺失 | emote 分区未烧录 | 确认 `CONFIG_APP_EMOTE_PARTITION_LABEL` 分区存在 |

## 参考

- `examples/common/app_common/include/app_common_tools.h` — 工具名宏与 handler 原型
- `examples/common/app_common/src/app_common_tools.c` — reminder/time/volume 实现
- `examples/voice_chat/main/app_tools.c` — set_emotion 实现 + 注册汇总
- `examples/matter_controller/main/app_tools.c` — Matter 工具 + 公共工具注册
- `examples/common/app_common/include/app_display.h` — 表情宏 `DISP_EMOTE_*`
- `examples/common/app_common/include/app_audio.h` — `app_audio_set_playback_volume`
- `examples/voice_chat/agent_config.json` — 工具 JSON 声明模板
