# esp-agents-firmware API 速查

> 全部签名取自仓库真实头文件。模块按组件/封装分组。代码片段为示意，实际以仓库源码为准。

## 1. esp_agent（底层 Agent 通信）

头文件：`components/agent/include/esp_agent.h`（聚合 `esp_agent_core.h` + `esp_agent_events.h` + `esp_agent_tools.h` + `esp_agent_messages.h`）。依赖 `espressif/esp_websocket_client`。

### 1.1 类型与配置（esp_agent_core.h）

```c
typedef void *esp_agent_handle_t;

typedef enum {
    ESP_AGENT_CONVERSATION_TEXT,
    ESP_AGENT_CONVERSATION_SPEECH,
    ESP_AGENT_CONVERSATION_TYPE_MAX,
} esp_agent_conversation_type_t;

typedef enum {
    ESP_AGENT_CONVERSATION_AUDIO_FORMAT_PCM,
    ESP_AGENT_CONVERSATION_AUDIO_FORMAT_OPUS,
    ESP_AGENT_CONVERSATION_AUDIO_FORMAT_MAX,
} esp_agent_conversation_audio_format_t;

typedef struct {
    esp_agent_conversation_audio_format_t format;
    uint16_t sample_rate;       /* Hz, e.g. 8000/16000 */
    uint8_t frame_duration;     /* ms, e.g. 20/40/60 */
} esp_agent_audio_config_t;

typedef struct {
    const char *agent_id;
    const char *refresh_token;
    esp_agent_conversation_type_t conversation_type;
    esp_agent_audio_config_t *upload_audio_config;
    esp_agent_audio_config_t *download_audio_config;
} esp_agent_config_t;

#define ESP_AGENT_API_ENDPOINT CONFIG_ESP_AGENT_API_ENDPOINT
```

### 1.2 生命周期（esp_agent_core.h）

```c
esp_agent_handle_t esp_agent_init(const esp_agent_config_t *config);
void               esp_agent_deinit(esp_agent_handle_t handle);
esp_err_t          esp_agent_start(esp_agent_handle_t handle, const char *conversation_id);
esp_err_t          esp_agent_stop(esp_agent_handle_t handle);
esp_err_t          esp_agent_new_conversation(esp_agent_handle_t handle);  /* 须先 start */
esp_err_t          esp_agent_set_agent_id(esp_agent_handle_t handle, const char *agent_id);
esp_err_t          esp_agent_set_refresh_token(esp_agent_handle_t handle, const char *refresh_token);
```

> `esp_agent_init` 会 strdup/memcpy config，调用者可释放。`agent_id`/`refresh_token` 可后置 set，但 `esp_agent_start` 前必须都设。

### 1.3 事件（esp_agent_events.h）

```c
typedef enum {
    ESP_AGENT_EVENT_INIT, ESP_AGENT_EVENT_DEINIT,
    ESP_AGENT_EVENT_START, ESP_AGENT_EVENT_STOP,
    ESP_AGENT_EVENT_ERROR,
    ESP_AGENT_EVENT_CONNECTED, ESP_AGENT_EVENT_DISCONNECTED,
    ESP_AGENT_EVENT_SPEECH_START, ESP_AGENT_EVENT_SPEECH_END,
    ESP_AGENT_EVENT_DATA_TYPE_TEXT, ESP_AGENT_EVENT_DATA_TYPE_THINKING, ESP_AGENT_EVENT_DATA_TYPE_SPEECH,
    ESP_AGENT_EVENT_DATA_TYPE_MAX,
} esp_agent_event_t;

typedef enum { ESP_AGENT_AUDIO_CONVERSATION_ERROR, ESP_AGENT_ERROR_MAX } esp_agent_error_t;
typedef enum { ESP_AGENT_MESSAGE_ROLE_USER, ESP_AGENT_MESSAGE_ROLE_ASSISTANT, ESP_AGENT_MESSAGE_ROLE_MAX } esp_agent_message_role_t;
typedef enum {
    ESP_AGENT_MESSAGE_GENERATION_STAGE_SPECULATIVE,
    ESP_AGENT_MESSAGE_GENERATION_STAGE_FINAL,
    ESP_AGENT_MESSAGE_GENERATION_STAGE_UNKNOWN,
} esp_agent_message_generation_stage_t;

typedef union {
    struct { const char *text; esp_agent_message_role_t role; esp_agent_message_generation_stage_t generation_stage; } text;
    struct { const uint8_t *data; const size_t len; } speech;
    struct { const char *conversation_id; } start;
    struct { const char *thought; } thinking;
    struct { esp_agent_error_t error; } error;
} esp_agent_message_data_t;

esp_err_t esp_agent_register_event_handler(esp_agent_handle_t handle, esp_agent_event_t event,
                                           esp_event_handler_t handler, void *user_data,
                                           esp_event_handler_instance_t *handler_instance);
esp_err_t esp_agent_unregister_event_handler(esp_agent_handle_t handle,
                                             esp_event_handler_instance_t *handler_instance,
                                             esp_agent_event_t event);
```

### 1.4 消息收发（esp_agent_messages.h）

```c
esp_err_t esp_agent_speech_conversation_start(esp_agent_handle_t handle);
esp_err_t esp_agent_speech_conversation_end(esp_agent_handle_t handle);
esp_err_t esp_agent_send_speech(esp_agent_handle_t handle, const uint8_t *data, size_t len, TickType_t timeout);
esp_err_t esp_agent_send_text(esp_agent_handle_t handle, const char *text, TickType_t timeout);
```

### 1.5 本地工具（esp_agent_tools.h）

```c
typedef enum { ESP_AGENT_PARAM_TYPE_NUMBER, ESP_AGENT_PARAM_TYPE_STRING, ESP_AGENT_PARAM_TYPE_BOOL, ESP_AGENT_PARAM_TYPE_MAX } esp_agent_tool_param_type_t;
typedef union { double i; const char *s; bool b; } esp_agent_tool_param_value_t;
typedef struct { const char *name; esp_agent_tool_param_type_t type; esp_agent_tool_param_value_t value; } esp_agent_tool_param_t;

/* result 须 heap 分配，框架发完内部 free */
typedef esp_err_t (*esp_agent_tool_handler_t)(esp_agent_handle_t handle, const char *tool_name,
                                              esp_agent_tool_param_t params[], size_t num_params,
                                              void *user_data, char **result);

esp_err_t esp_agent_register_local_tool(esp_agent_handle_t handle, const char *name,
                                        esp_agent_tool_handler_t tool_handler, void *user_data);
esp_err_t esp_agent_unregister_local_tool(esp_agent_handle_t handle, const char *name);
```

## 2. app_agent（应用层封装）

头文件：`examples/common/app_common/include/app_agent.h`。封装 setup 联动 + 音频配置 + 事件分发。

```c
typedef enum {
    APP_AGENT_STATE_DISCONNECTED = 0,
    APP_AGENT_STATE_CONNECTING,
    APP_AGENT_STATE_CONNECTED,
    APP_AGENT_STATE_STARTED,
} app_agent_state_t;

typedef struct {
    esp_event_handler_t event_handler;   /* 必填，自定义 handler 须转发给 default */
} app_agent_config_t;

esp_err_t app_agent_init(app_agent_config_t *config);
esp_err_t app_agent_start(void);                        /* 内部 setup_rainmaker_init + agent_setup_start */
esp_err_t app_agent_connect(void);                      /* 内部 esp_agent_start(handle, NULL) */
esp_err_t app_agent_send_speech(uint8_t *audio_data, size_t audio_data_len);  /* 内部 1000ms 超时 */
bool     app_agent_is_active(void);                     /* state == STARTED */
app_agent_state_t app_agent_get_state(void);
esp_err_t app_agent_speech_conversation_start(void);
esp_err_t app_agent_speech_conversation_end(void);

void app_agent_default_event_handler(void *arg, esp_event_base_t event_base, int32_t event_id, void *event_data);

esp_err_t app_agent_register_tool(const char *name, esp_agent_tool_handler_t tool_handler, void *user_data);
esp_err_t app_agent_tool_unregister(const char *name);
```

## 3. agent_setup / RainMaker setup

头文件：`components/setup/include/agent_setup.h`、`setup/rainmaker.h`、`setup/console.h`。

```c
/* agent_setup.h */
ESP_EVENT_DECLARE_BASE(AGENT_SETUP_EVENT);
typedef enum {
    AGENT_SETUP_EVENT_START,                /* 网络+agent_id+token 全满足时触发 */
    AGENT_SETUP_EVENT_NETWORK_CONNECTED,
    AGENT_SETUP_EVENT_AGENT_ID_UPDATE,      /* 单独触发，不保证其他条件已满足 */
} agent_setup_event_t;

esp_err_t agent_setup_init(void);
esp_err_t agent_setup_start(void);
char*    agent_setup_get_agent_id(void);       /* 内部缓冲，勿 free */
char*    agent_setup_get_refresh_token(void);  /* 同上 */
esp_err_t agent_setup_set_agent_id(const char *agent_id);
esp_err_t agent_setup_set_refresh_token(const char *refresh_token);
esp_err_t agent_setup_factory_reset(void);

/* setup/rainmaker.h */
esp_err_t setup_rainmaker_init(const char *board_device_manual_url);
esp_err_t setup_rainmaker_factory_reset(void);
esp_err_t setup_rainmaker_register_volume_callbacks(esp_err_t (*get_cb)(uint8_t *volume),
                                                    esp_err_t (*set_cb)(uint8_t volume));
esp_err_t setup_rainmaker_update_volume(uint8_t volume);

/* setup/console.h */
esp_err_t setup_console_register_commands(void);   /* 注册 set-token / set-agent */
```

## 4. agent_console（串口命令）

头文件：`components/agent_console/include/agent_console.h`。

```c
esp_err_t agent_console_init(void);
esp_err_t agent_console_register_default_commands(void);   /* cpu-dump/mem-dump/reboot/reset-to-factory 等 */
esp_err_t agent_console_register_command(const esp_console_cmd_t *cmd);
```

> `set-wifi` 由 `components/setup/src/agent_setup.c` 的 `register_set_wifi_cli_handler()` 注册。

## 5. app_audio（音频管线）

头文件：`examples/common/app_common/include/app_audio.h`。底层依赖 `audio_recorder.h` / `audio_playback.h`（来自 `components/audio`）。

```c
typedef enum { MICROPHONE_STATE_STOP, MICROPHONE_STATE_START, MICROPHONE_STATE_PAUSE, MICROPHONE_STATE_MAX } app_audio_microphone_state_t;

esp_err_t app_audio_init(void);
esp_err_t app_audio_start(void);
esp_err_t app_audio_set_playback_volume(uint8_t volume);          /* 写 NVS */
esp_err_t app_audio_play_speech(uint8_t *data, size_t data_len);
esp_err_t app_audio_microphone_set_state(app_audio_microphone_state_t state);
esp_err_t app_audio_speaker_start(void);
esp_err_t app_audio_speaker_stop(void);
esp_err_t app_audio_speaker_download_complete(void);
esp_err_t app_audio_play_media_sync(const char *media_url, const uint8_t *data, size_t data_len);
esp_err_t app_audio_play_media_async(const char *media_url, const uint8_t *data, size_t data_len);
esp_err_t app_audio_trigger_sleep(void);
esp_err_t app_audio_set_awake(bool awake);
```

## 6. app_device（状态机 / 事件队列）

头文件：`examples/common/app_common/include/app_device.h`。

```c
typedef enum {
    DEVICE_EVENT_SYSTEM_INITIALIZED, DEVICE_EVENT_SPEECH_START, DEVICE_EVENT_SPEECH_END,
    DEVICE_EVENT_SPEECH_PLAYBACK_COMPLETE, DEVICE_EVENT_WAKEUP, DEVICE_EVENT_SLEEP,
    DEVICE_EVENT_INTERRUPT, DEVICE_EVENT_FACTORY_RESET, DEVICE_EVENT_AGENT_STATE_CHANGED,
    DEVICE_EVENT_REMINDER, DEVICE_EVENT_REMINDER_COMPLETE,
    DEVICE_EVENT_SET_USER_TEXT, DEVICE_EVENT_SET_ASSISTANT_TEXT, DEVICE_EVENT_MAX,
} app_device_event_t;

typedef union { const char *text; } device_event_data_t;

typedef enum { APP_DEVICE_TEXT_TYPE_USER, APP_DEVICE_TEXT_TYPE_ASSISTANT, APP_DEVICE_TEXT_TYPE_SYSTEM } app_device_text_type_t;
typedef enum { APP_DEVICE_SYSTEM_STATE_SLEEP, APP_DEVICE_SYSTEM_STATE_ACTIVE, APP_DEVICE_SYSTEM_STATE_LISTENING } app_device_system_state_t;

typedef struct {
    esp_err_t (*set_text_cb)(app_device_text_type_t text_type, const char *text, void *priv_data);
    esp_err_t (*system_state_changed_cb)(app_device_system_state_t new_state, void *priv_data);
    void *priv_data;
} app_device_config_t;

esp_err_t app_device_event_enqueue(app_device_event_t event, device_event_data_t *data);              /* 任务上下文，阻塞 portMAX_DELAY */
esp_err_t app_device_event_enqueue_from_isr(app_device_event_t event, device_event_data_t *data);     /* ISR，非阻塞 */
esp_err_t app_device_init(app_device_config_t *config);
```

## 7. app_display（显示与表情）

头文件：`examples/common/app_common/include/app_display.h`。

```c
esp_err_t app_display_init(void);
esp_err_t app_display_set_text(app_device_text_type_t text_type, const char *text, void *arg);
esp_err_t app_display_system_state_changed(app_device_system_state_t new_state, void *arg);
bool      app_display_is_emotion_valid(const char *emotion);
esp_err_t app_display_set_emotion(const char *emotion);   /* 无效返 ESP_ERR_INVALID_ARG */

/* 表情字符串常量（须与 emote 分区资源一致） */
#define DISP_EMOTE_NEUTRAL "neutral"
#define DISP_EMOTE_HAPPY   "happy"
#define DISP_EMOTE_SAD     "sad"
#define DISP_EMOTE_CRYING  "crying"
#define DISP_EMOTE_ANGRY   "angry"
#define DISP_EMOTE_SLEEPY  "sleepy"
#define DISP_EMOTE_CONFUSED "confused"
#define DISP_EMOTE_SHOCKED "shocked"
#define DISP_EMOTE_WINKING "winking"
#define DISP_EMOTE_IDLE    "idle"
```

## 8. app_common_tools（内置工具 handler）

头文件：`examples/common/app_common/include/app_common_tools.h`。

```c
#define TOOL_NAME_SET_REMINDER    "set_reminder"
#define TOOL_NAME_GET_LOCAL_TIME  "get_local_time"
#define TOOL_NAME_SET_VOLUME      "set_volume"

esp_err_t app_common_tools_set_reminder_handler(esp_agent_handle_t handle, const char *tool_name,
                                                esp_agent_tool_param_t params[], size_t num_params,
                                                void *user_data, char **result);
esp_err_t app_common_tools_get_local_time_handler(...);   /* 同上签名 */
esp_err_t app_common_tools_set_volume_handler(...);       /* 同上签名 */
```

## 9. 触摸

头文件：`app_touch_press.h`、`app_capacitive_touch.h`。

```c
/* app_touch_press.h */
esp_err_t app_touch_press_init(void);
bool      app_touch_press_on_active(void);
bool      app_touch_press_on_inactive(void);
void      app_touch_press_deinit(void);

/* app_capacitive_touch.h（须板定义 CAPACITIVE_TOUCH_SUPPORTED） */
esp_err_t app_capacitive_touch_init(void);
```

## 10. Matter 控制器（仅 matter_controller 示例）

### 10.1 应用层入口

头文件：`examples/matter_controller/main/matter/app_controller.h`。

```c
esp_err_t matter_controller_start_task(void);                          /* 须在 app_agent_start 后 */
esp_err_t matter_controller_get_device_list(char **device_list_json);
esp_err_t matter_controller_control_device(char **result, uint64_t node_id,
                                           uint32_t cluster_id, uint32_t command_id,
                                           const char *command_params_json);
```

### 10.2 控制器服务组件

头文件：`examples/matter_controller/components/matter_controller/rmaker_controller_service/`。

```c
/* app_matter_controller.h */
typedef union {
    int raw;
    struct {
        unsigned base_url_set:1, user_token_set:1, access_token_set:1,
                 rmaker_group_id_set:1, matter_fabric_id_set:1, matter_node_id_set:1, matter_noc_installed:1;
    };
} matter_controller_status_t;

typedef struct {
    char *base_url, *user_token, *access_token, *rmaker_group_id;
    uint64_t matter_fabric_id, matter_node_id;
    bool matter_noc_installed;
    uint16_t matter_vendor_id;
    esp_rmaker_device_t *service;
} matter_controller_handle_t;

typedef enum {
    MATTER_CONTROLLER_CALLBACK_TYPE_AUTHORIZE = 1,
    MATTER_CONTROLLER_CALLBACK_TYPE_QUERY_MATTER_FABRIC_ID,
    MATTER_CONTROLLER_CALLBACK_TYPE_SETUP_CONTROLLER,
    MATTER_CONTROLLER_CALLBACK_TYPE_UPDATE_CONTROLLER_NOC,
    MATTER_CONTROLLER_CALLBACK_TYPE_UPDATE_DEVICE,
} matter_controller_callback_type_t;

typedef esp_err_t (*matter_controller_callback_t)(matter_controller_handle_t *handle, matter_controller_callback_type_t type);

esp_err_t matter_controller_enable(uint16_t matter_vendor_id, matter_controller_callback_t callback);
esp_err_t matter_controller_handle_update(void);
esp_err_t matter_controller_report_status(matter_controller_status_t status);
esp_err_t matter_controller_update_device_list(void);

/* app_matter_device_manager.h */
typedef void (*device_list_update_callback_t)(void);
esp_err_t update_device_list(matter_controller_handle_t *controller_handle);
matter_device_t *fetch_device_list(void);
esp_err_t init_device_manager(device_list_update_callback_t dev_list_update_cb);
```

### 10.3 RainMaker 服务规格（SPEC.md + matter_controller_std.h）

`esp.service.matter-controller`（别名 **MatterCTL**）暴露 6 参数（详见 `examples/matter_controller/components/matter_controller/rmaker_controller_service/SPEC.md`）：

| 参数名 | 宏 | 标志 |
|---|---|---|
| BaseURL | `ESP_RMAKER_PARAM_BASE_URL` | R W P |
| UserToken | `ESP_RMAKER_PARAM_USER_TOKEN` | W P（不可回读） |
| RMakerGroupID | `ESP_RMAKER_PARAM_RMAKER_GROUP_ID` | R W P |
| MatterNodeID | `ESP_RMAKER_PARAM_MATTER_NODE_ID` | R |
| MTCtlCMD | `ESP_RMAKER_PARAM_MATTER_CTL_CMD` | W（1=UpdateNOC, 2=UpdateDeviceList） |
| MTCtlStatus | `ESP_RMAKER_PARAM_MATTER_CTL_STATUS` | R P（7 位位图，见 10.2 `matter_controller_status_t`） |

`MTCtlStatus` 7 位（bit 0→6）：`base_url_set` / `user_token_set` / `access_token_set` / `rmaker_group_id_set` / `matter_fabric_id_set` / `matter_node_id_set` / `matter_noc_installed`。授权完成的标志是全 1。

服务创建（`matter_controller_std.h`）：

```c
esp_rmaker_device_t *matter_controller_service_create(const char *serv_name,
                                                      esp_rmaker_device_write_cb_t write_cb,
                                                      esp_rmaker_device_read_cb_t read_cb,
                                                      void *priv_data);
```

回调分派（`app_matter_controller_callback.cpp`）按 `matter_controller_callback_type_t`（见 10.2）：`AUTHORIZE`→`fetch_access_token`、`QUERY_MATTER_FABRIC_ID`→`fetch_matter_fabric_id`、`SETUP_CONTROLLER`→`fetch_fabric_ipk`+`matter_controller_client::setup_controller`、`UPDATE_CONTROLLER_NOC`、`UPDATE_DEVICE`→`update_device_list`。

### 10.4 RainMaker 参数宏（matter_controller_std.h）

```c
#define ESP_RMAKER_DEVICE_MATTER_CONTROLLER  "esp.device.matter-controller"
#define ESP_RMAKER_SERVICE_MATTER_CONTROLLER "esp.service.matter-controller"
#define ESP_RMAKER_DEF_BASE_URL_NAME         "BaseURL"
#define ESP_RMAKER_PARAM_BASE_URL            "esp.param.base-url"
#define ESP_RMAKER_DEF_USER_TOKEN_NAME       "UserToken"
#define ESP_RMAKER_PARAM_USER_TOKEN          "esp.param.user-token"
#define ESP_RMAKER_DEF_RMAKER_GROUP_ID_NAME  "RMakerGroupID"
#define ESP_RMAKER_PARAM_RMAKER_GROUP_ID     "esp.param.rmaker-group-id"
#define ESP_RMAKER_DEF_MATTER_NODE_ID_NAME   "MatterNodeID"
#define ESP_RMAKER_PARAM_MATTER_NODE_ID      "esp.param.matter-node-id"
#define ESP_RMAKER_DEF_MATTER_CTL_CMD_NAME   "MTCtlCMD"
#define ESP_RMAKER_PARAM_MATTER_CTL_CMD      "esp.param.matter-ctl-cmd"
#define ESP_RMAKER_DEF_MATTER_CTL_STATUS_NAME "MTCtlStatus"
#define ESP_RMAKER_PARAM_MATTER_CTL_STATUS   "esp.param.matter-ctl-status"
```
