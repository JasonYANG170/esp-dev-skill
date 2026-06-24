# 音频管线（录音 / 播放）

> **适用摘要**: 理解 `components/audio` 与 `app_audio` 封装，配置上下行采样率/帧长/AEC/音量，把 Agent 下行语音送入扬声器。

## 触发意图

- "音频管线"
- "录音 / 播放配置"
- "上下行采样率"
- "AEC 回声消除"
- "set_playback_volume"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考头 | `examples/common/app_common/include/app_audio.h` |
| 组件 | `components/audio`（依赖 esp_codec_dev / esp-sr / gmf_audio 等，见 `idf_component.yml`） |
| 板支持 | 经 `esp_board_manager_init()` 初始化 codec/I2S 等外设 |

## 分步说明

### 1. 初始化与启动

`app_main` 中按顺序（见 `recipes/agent_init_and_events.md`）：

```c
esp_board_manager_init();   /* 板级外设：i2c/i2s/spi，含音频 codec */
app_audio_init();           /* 初始化 recorder + playback 管线 */
/* Agent 启动后由 app_agent_start_task 内部 app_audio_start() */
```

`app_audio_init()`（来自 `examples/common/app_common/include/app_audio.h`）初始化录音与播放管线；`app_audio_start()` 在 Agent 启动任务里被调用。

### 2. 上下行音频配置（Kconfig）

`app_agent_init` 内部从 Kconfig 构造上下行 `esp_agent_audio_config_t`，格式均为 `ESP_AGENT_CONVERSATION_AUDIO_FORMAT_OPUS`（来自 `app_agent.c`）：

```c
esp_agent_audio_config_t up = {
    .format = ESP_AGENT_CONVERSATION_AUDIO_FORMAT_OPUS,
    .sample_rate = CONFIG_AUDIO_UPLOAD_SAMPLE_RATE,       /* 默认 8000 */
    .frame_duration = CONFIG_AUDIO_UPLOAD_FRAME_DURATION_MS, /* 默认 20 */
};
esp_agent_audio_config_t down = {
    .format = ESP_AGENT_CONVERSATION_AUDIO_FORMAT_OPUS,
    .sample_rate = CONFIG_AUDIO_DOWNLOAD_SAMPLE_RATE,     /* 默认 16000 */
    .frame_duration = CONFIG_AUDIO_DOWNLOAD_FRAME_DURATION_MS, /* 默认 60 */
};
```

相关 Kconfig（来自 `examples/common/app_common/Kconfig`）：

| 配置项 | 默认 | 含义 |
|---|---|---|
| `CONFIG_AUDIO_UPLOAD_SAMPLE_RATE` | 8000 | 上行（录音→云）采样率 Hz |
| `CONFIG_AUDIO_DOWNLOAD_SAMPLE_RATE` | 16000 | 下行（云→播放）采样率 Hz |
| `CONFIG_AUDIO_UPLOAD_FRAME_DURATION_MS` | 20 | 上行帧时长 ms |
| `CONFIG_AUDIO_DOWNLOAD_FRAME_DURATION_MS` | 60 | 下行帧时长 ms |
| `CONFIG_APP_AUDIO_DEFAULT_PLAYBACK_VOLUME` | 75 | 默认播放音量 0–100 |

修改：

```bash
idf.py menuconfig
# → App Common Config → 各项
```

> 改采样率/帧长需与云端 Agent 模型期望一致，否则语音异常。

### 3. AEC（回声消除）

`components/audio/Kconfig` 提供：

| 配置项 | 默认 | 含义 |
|---|---|---|
| `CONFIG_ENABLE_AEC` | n | 启用 AEC（须板硬件支持） |

```bash
idf.py menuconfig
# → Audio Config → Enable AEC (Acoustic Echo Cancellation)
```

> 仅当板支持 AEC 硬件加速时开启，否则无效或编译错。

### 4. 播放下行语音

默认事件处理器在收到 `ESP_AGENT_EVENT_DATA_TYPE_SPEECH` 时自动喂播放器（来自 `app_agent.c`）：

```c
case ESP_AGENT_EVENT_DATA_TYPE_SPEECH:
    app_audio_play_speech((uint8_t *)data->speech.data, data->speech.len);
    break;
```

相关播放控制 API（`app_audio.h`）：

```c
app_audio_play_speech(data, len);          /* 播一段语音帧 */
app_audio_speaker_start();
app_audio_speaker_stop();
app_audio_speaker_download_complete();      /* 通知一整段下载完毕 */
app_audio_play_media_sync(url, data, len);  /* 同步播放媒体 */
app_audio_play_media_async(url, data, len); /* 异步播放媒体 */
```

### 5. 麦克风状态控制

```c
typedef enum {
    MICROPHONE_STATE_STOP,     /* 默认 */
    MICROPHONE_STATE_START,
    MICROPHONE_STATE_PAUSE,
} app_audio_microphone_state_t;

app_audio_microphone_set_state(MICROPHONE_STATE_START);
```

唤醒（`DEVICE_EVENT_WAKEUP`）后由状态机驱动麦克风进入录音，休眠（`DEVICE_EVENT_SLEEP`）时停止，避免断连时仍上行数据。

### 6. 设置播放音量

```c
app_audio_set_playback_volume(80);   /* 0–100，内部写 NVS 持久化 */
```

`set_volume` 工具（见 `recipes/builtin_tools.md`）即调此函数。RainMaker 侧可经 `setup_rainmaker_update_volume()` / `setup_rainmaker_register_volume_callbacks()` 与 App 同步。

### 7. 休眠/唤醒联动

```c
app_audio_trigger_sleep();
app_audio_set_awake(true);   /* true=唤醒态 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 录音无声音 | 麦克风状态未 START | 由状态机驱动；确认唤醒事件已触发 |
| 语音卡顿 | 采样率/帧长与云端不符 | 对齐 `CONFIG_AUDIO_UPLOAD/DOWNLOAD_*` 与 Agent 模型 |
| 音量改了重启丢失 | 没走 `app_audio_set_playback_volume` | 该函数写 NVS；勿直接调底层 codec |
| AEC 开启编译错 | 板不支持 | 仅支持 AEC 的板开 `CONFIG_ENABLE_AEC` |
| 播放有回声 | 未开 AEC | 板支持时开 AEC，或降低音量/调整麦克位置 |

## 参考

- `examples/common/app_common/include/app_audio.h` — app_audio 全部 API
- `examples/common/app_common/src/app_agent.c` — 下行语音事件 → `app_audio_play_speech`
- `examples/common/app_common/Kconfig` — 音频/音量 Kconfig
- `components/audio/Kconfig` — `CONFIG_ENABLE_AEC`
- `components/audio/idf_component.yml` — 音频组件依赖（esp_codec_dev / esp-sr / gmf_*）
- `components/audio/README.md` — 音频管线组件说明
