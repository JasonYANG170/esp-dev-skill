# ESP-ADF API Quick Reference

> 所有签名均来自仓库 `components/*/include/*.h` 与 `docs/en/api-reference/`。stream/codec 的 `*_init` 与 `*_CFG_DEFAULT` 宏来自对应头文件；编解码器头位于 `components/esp-adf-libs/include`（需递归克隆子模块）。

## Framework

### audio_pipeline（`audio_pipeline.h`）

```c
typedef struct audio_pipeline *audio_pipeline_handle_t;

typedef struct audio_pipeline_cfg {
    int rb_size;        // 环形缓冲大小
} audio_pipeline_cfg_t;

#define DEFAULT_PIPELINE_RINGBUF_SIZE    (8*1024)
#define DEFAULT_AUDIO_PIPELINE_CONFIG() { .rb_size = DEFAULT_PIPELINE_RINGBUF_SIZE }

audio_pipeline_handle_t audio_pipeline_init(audio_pipeline_cfg_t *config);
esp_err_t audio_pipeline_deinit(audio_pipeline_handle_t pipeline);

esp_err_t audio_pipeline_register(audio_pipeline_handle_t pipeline, audio_element_handle_t el, const char *name);
esp_err_t audio_pipeline_unregister(audio_pipeline_handle_t pipeline, audio_element_handle_t el);
esp_err_t audio_pipeline_link(audio_pipeline_handle_t pipeline, const char *link_tag[], int link_num);
esp_err_t audio_pipeline_unlink(audio_pipeline_handle_t pipeline);

esp_err_t audio_pipeline_run(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_terminate(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_terminate_with_ticks(audio_pipeline_handle_t pipeline, TickType_t ticks_to_wait);
esp_err_t audio_pipeline_resume(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_pause(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_stop(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_wait_for_stop(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_wait_for_stop_with_ticks(audio_pipeline_handle_t pipeline, TickType_t ticks_to_wait);

esp_err_t audio_pipeline_set_listener(audio_pipeline_handle_t pipeline, audio_event_iface_handle_t evt);
esp_err_t audio_pipeline_remove_listener(audio_pipeline_handle_t pipeline);
audio_event_iface_handle_t audio_pipeline_get_event_iface(audio_pipeline_handle_t pipeline);

audio_element_handle_t audio_pipeline_get_el_by_tag(audio_pipeline_handle_t pipeline, const char *tag);

esp_err_t audio_pipeline_reset_items_state(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_reset_ringbuffer(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_reset_elements(audio_pipeline_handle_t pipeline);
esp_err_t audio_pipeline_change_state(audio_pipeline_handle_t pipeline, audio_element_state_t new_state);
esp_err_t audio_pipeline_relink(audio_pipeline_handle_t pipeline, const char *link_tag[], int link_num);
esp_err_t audio_pipeline_breakup_elements(audio_pipeline_handle_t pipeline, audio_element_handle_t kept_ctx_el);
```

### audio_element（`audio_element.h`）

```c
typedef struct audio_element *audio_element_handle_t;

typedef enum {
    AEL_IO_OK, AEL_IO_FAIL, AEL_IO_DONE = -2, AEL_IO_ABORT = -3,
    AEL_IO_TIMEOUT = -4, AEL_PROCESS_FAIL = -5,
} audio_element_err_t;

typedef enum {
    AEL_STATE_NONE=0, AEL_STATE_INIT, AEL_STATE_INITIALIZING,
    AEL_STATE_RUNNING, AEL_STATE_PAUSED, AEL_STATE_STOPPED,
    AEL_STATE_FINISHED, AEL_STATE_ERROR
} audio_element_state_t;

typedef enum {
    AEL_MSG_CMD_NONE=0, AEL_MSG_CMD_FINISH=2, AEL_MSG_CMD_STOP=3,
    AEL_MSG_CMD_PAUSE=4, AEL_MSG_CMD_RESUME=5, AEL_MSG_CMD_DESTROY=6,
    AEL_MSG_CMD_REPORT_STATUS=8, AEL_MSG_CMD_REPORT_MUSIC_INFO=9,
    AEL_MSG_CMD_REPORT_CODEC_FMT=10, AEL_MSG_CMD_REPORT_POSITION=11,
} audio_element_msg_cmd_t;

typedef enum {
    AEL_STATUS_NONE=0, AEL_STATUS_ERROR_OPEN, AEL_STATUS_ERROR_INPUT,
    AEL_STATUS_ERROR_PROCESS, AEL_STATUS_ERROR_OUTPUT, AEL_STATUS_ERROR_CLOSE,
    AEL_STATUS_ERROR_TIMEOUT, AEL_STATUS_ERROR_UNKNOWN,
    AEL_STATUS_INPUT_DONE, AEL_STATUS_INPUT_BUFFERING,
    AEL_STATUS_OUTPUT_DONE, AEL_STATUS_OUTPUT_BUFFERING,
    AEL_STATUS_STATE_RUNNING, AEL_STATUS_STATE_PAUSED,
    AEL_STATUS_STATE_STOPPED, AEL_STATUS_STATE_FINISHED,
    AEL_STATUS_MOUNTED, AEL_STATUS_UNMOUNTED,
} audio_element_status_t;

typedef struct {
    int sample_rates, channels, bits, bps;
    int64_t byte_pos, total_bytes;
    int duration;
    char *uri;
    esp_codec_type_t codec_fmt;
    audio_element_reserve_data_t reserve_data;
} audio_element_info_t;

#define AUDIO_ELEMENT_INFO_DEFAULT() { .sample_rates=44100, .channels=2, .bits=16, \
    .bps=0, .byte_pos=0, .total_bytes=0, .duration=0, .uri=NULL, \
    .codec_fmt=ESP_CODEC_TYPE_UNKNOW }

typedef struct {
    el_io_func  open; ctrl_func seek; process_func process;
    el_io_func  close; el_io_func destroy;
    stream_func read; stream_func write;
    int buffer_len, task_stack, task_prio, task_core, out_rb_size;
    void *data; const char *tag; bool stack_in_ext;
    int multi_in_rb_num, multi_out_rb_num;
} audio_element_cfg_t;

#define DEFAULT_ELEMENT_RINGBUF_SIZE   (8*1024)
#define DEFAULT_ELEMENT_BUFFER_LENGTH  (4*1024)
#define DEFAULT_ELEMENT_STACK_SIZE     (2*1024)
#define DEFAULT_ELEMENT_TASK_PRIO      (5)
#define DEFAULT_ELEMENT_TASK_CORE      (0)
#define DEFAULT_AUDIO_ELEMENT_CONFIG() { .buffer_len=DEFAULT_ELEMENT_BUFFER_LENGTH, \
    .task_stack=DEFAULT_ELEMENT_STACK_SIZE, .task_prio=DEFAULT_ELEMENT_TASK_PRIO, \
    .task_core=DEFAULT_ELEMENT_TASK_CORE, .multi_in_rb_num=0, .multi_out_rb_num=0 }

audio_element_handle_t audio_element_init(audio_element_cfg_t *config);
esp_err_t audio_element_deinit(audio_element_handle_t el);

esp_err_t audio_element_set_read_cb(audio_element_handle_t el, stream_func fn, void *context);
esp_err_t audio_element_set_write_cb(audio_element_handle_t el, stream_func fn, void *context);

esp_err_t audio_element_set_uri(audio_element_handle_t el, const char *uri);
char *audio_element_get_uri(audio_element_handle_t el);
esp_err_t audio_element_setinfo(audio_element_handle_t el, audio_element_info_t *info);
esp_err_t audio_element_getinfo(audio_element_handle_t el, audio_element_info_t *info);
esp_err_t audio_element_set_music_info(audio_element_handle_t el, int sample_rates, int channels, int bits);

audio_element_state_t audio_element_get_state(audio_element_handle_t el);
esp_err_t audio_element_run(audio_element_handle_t el);
esp_err_t audio_element_stop(audio_element_handle_t el);
esp_err_t audio_element_wait_for_stop(audio_element_handle_t el);
esp_err_t audio_element_terminate(audio_element_handle_t el);
esp_err_t audio_element_pause(audio_element_handle_t el);
esp_err_t audio_element_resume(audio_element_handle_t el, float wait_for_rb_threshold, TickType_t timeout);

audio_element_err_t audio_element_input(audio_element_handle_t el, char *buffer, int wanted_size);
audio_element_err_t audio_element_output(audio_element_handle_t el, char *buffer, int write_size);

esp_err_t audio_element_set_input_ringbuf(audio_element_handle_t el, ringbuf_handle_t rb);
esp_err_t audio_element_set_output_ringbuf(audio_element_handle_t el, ringbuf_handle_t rb);
ringbuf_handle_t audio_element_get_input_ringbuf(audio_element_handle_t el);
ringbuf_handle_t audio_element_get_output_ringbuf(audio_element_handle_t el);
```

### audio_event_iface（`audio_event_iface.h`）

```c
typedef struct {
    int cmd;
    void *data; int data_len;
    void *source; int source_type;
    bool need_free_data;
} audio_event_iface_msg_t;

typedef struct audio_event_iface *audio_event_iface_handle_t;

#define DEFAULT_AUDIO_EVENT_IFACE_SIZE  (5)
#define AUDIO_EVENT_IFACE_DEFAULT_CFG() { \
    .internal_queue_size=DEFAULT_AUDIO_EVENT_IFACE_SIZE, \
    .external_queue_size=DEFAULT_AUDIO_EVENT_IFACE_SIZE, \
    .queue_set_size=DEFAULT_AUDIO_EVENT_IFACE_SIZE, \
    .on_cmd=NULL, .context=NULL, .wait_time=portMAX_DELAY, .type=0 }

audio_event_iface_handle_t audio_event_iface_init(audio_event_iface_cfg_t *config);
esp_err_t audio_event_iface_destroy(audio_event_iface_handle_t evt);
esp_err_t audio_event_iface_set_listener(audio_event_iface_handle_t evt, audio_event_iface_handle_t listener);
esp_err_t audio_event_iface_remove_listener(audio_event_iface_handle_t evt, audio_event_iface_handle_t listener);
esp_err_t audio_event_iface_listen(audio_event_iface_handle_t evt, audio_event_iface_msg_t *msg, TickType_t wait_time);
```

### audio_common（`audio_common.h`）

```c
typedef enum {
    AUDIO_STREAM_NONE=0, AUDIO_STREAM_READER, AUDIO_STREAM_WRITER
} audio_stream_type_t;

typedef enum {
    AUDIO_CODEC_TYPE_NONE=0, AUDIO_CODEC_TYPE_DECODER, AUDIO_CODEC_TYPE_ENCODER
} audio_codec_type_t;
```

## Streams（`components/audio_stream/include/`）

```c
// i2s_stream.h
#define I2S_STREAM_TASK_STACK        (3584)
#define I2S_STREAM_BUF_SIZE          (3600)
#define I2S_STREAM_TASK_PRIO         (23)
#define I2S_STREAM_RINGBUFFER_SIZE   (8*1024)
#define I2S_STREAM_CFG_DEFAULT()  I2S_STREAM_CFG_DEFAULT_WITH_PARA(I2S_NUM_0, 44100, I2S_BITS_PER_SAMPLE_16BIT, AUDIO_STREAM_WRITER)
audio_element_handle_t i2s_stream_init(i2s_stream_cfg_t *config);
esp_err_t i2s_stream_set_clk(audio_element_handle_t i2s_stream, int rate, int bits, int ch);
// i2s_channel_type_t: I2S_CHANNEL_TYPE_RIGHT_LEFT / ALL_RIGHT / ALL_LEFT / ONLY_RIGHT / ONLY_LEFT

// http_stream.h
#define HTTP_STREAM_CFG_DEFAULT()  { ... }
audio_element_handle_t http_stream_init(http_stream_cfg_t *config);
esp_err_t http_stream_next_track(audio_element_handle_t el);
esp_err_t http_stream_restart(audio_element_handle_t el);
esp_err_t http_stream_fetch_again(audio_element_handle_t el);
esp_err_t http_stream_set_server_cert(audio_element_handle_t el, const char *cert);

// fatfs_stream.h
#define FATFS_STREAM_CFG_DEFAULT() { ... }
audio_element_handle_t fatfs_stream_init(fatfs_stream_cfg_t *config);

// spiffs_stream.h
#define SPIFFS_STREAM_CFG_DEFAULT() { ... }
audio_element_handle_t spiffs_stream_init(spiffs_stream_cfg_t *config);

// raw_stream.h
#define RAW_STREAM_CFG_DEFAULT() { ... }
audio_element_handle_t raw_stream_init(raw_stream_cfg_t *cfg);

// tone_stream.h / embed_flash_stream.h / tts_stream.h / pwm_stream.h / tcp_client_stream.h / algorithm_stream.h / aec_stream.h
//   每个均有 <name>_init(<name>_cfg_t*) 与 <NAME>_CFG_DEFAULT() 宏，type 字段同上
```

## Codecs（`components/esp-adf-libs/include`，需递归克隆子模块）

> 编解码器以预编译库提供，头文件在 `esp-adf-libs`。文档：`docs/en/api-reference/codecs/`。

```c
// mp3_decoder.h
mp3_decoder_cfg_t mp3_cfg = DEFAULT_MP3_DECODER_CONFIG();
audio_element_handle_t mp3_decoder = mp3_decoder_init(&mp3_cfg);

// 其他解码器（同模式）：
//   aac_decoder_init / DEFAULT_AAC_DECODER_CONFIG
//   flac_decoder_init / DEFAULT_FLAC_DECODER_CONFIG
//   opus_decoder_init / DEFAULT_OPUS_DECODER_CONFIG
//   ogg_decoder_init / DEFAULT_OGG_DECODER_CONFIG
//   wav_decoder_init / DEFAULT_WAV_DECODER_CONFIG
//   amr_decoder_init ...

// 编码器：
//   wav_encoder_init / DEFAULT_WAV_ENCODER_CONFIG
//   amrnb_encoder_init / DEFAULT_AMRNB_ENCODER_CONFIG
//   amrwb_encoder_init / DEFAULT_AMRWB_ENCODER_CONFIG
```

## Peripherals（`components/esp_peripherals/include/`）

```c
// esp_peripherals.h
#define DEFAULT_ESP_PERIPH_SET_CONFIG() { ... }
esp_periph_set_handle_t esp_periph_set_init(esp_periph_config_t *config);
esp_err_t esp_periph_set_destroy(esp_periph_set_handle_t periph_set_handle);
esp_err_t esp_periph_start(esp_periph_set_handle_t set, esp_periph_handle_t periph);
esp_err_t esp_periph_stop(esp_periph_handle_t periph);
audio_event_iface_handle_t esp_periph_set_get_event_iface(esp_periph_set_handle_t periph_set_handle);

// periph_wifi.h
typedef enum { PERIPH_WIFI_UNCHANGE=0, PERIPH_WIFI_CONNECTING, PERIPH_WIFI_CONNECTED,
               PERIPH_WIFI_DISCONNECTED, PERIPH_WIFI_SETTING, PERIPH_WIFI_CONFIG_DONE,
               PERIPH_WIFI_CONFIG_ERROR, PERIPH_WIFI_ERROR } periph_wifi_state_t;
esp_periph_handle_t periph_wifi_init(periph_wifi_cfg_t *config);
esp_err_t periph_wifi_wait_for_connected(esp_periph_handle_t periph, TickType_t tick);

// periph_touch.h
// 事件命令：PERIPH_TOUCH_UNCHANGE=0, PERIPH_TOUCH_TAP, PERIPH_TOUCH_RELEASE,
//           PERIPH_TOUCH_LONG_TAP, PERIPH_TOUCH_LONG_RELEASE
esp_periph_handle_t periph_touch_init(periph_touch_cfg_t *config);

// periph_adc_button.h / periph_button.h / periph_sdcard.h / periph_spiffs.h / periph_console.h / periph_led.h / periph_is31fl3216.h
//   每个有 periph_<name>_init(<name>_cfg_t*) 与 PERIPH_<NAME>_CFG_DEFAULT() 宏
```

## Board（`components/audio_board/<board>/board.h`）

```c
typedef struct audio_board_handle *audio_board_handle_t;
audio_board_handle_t audio_board_init(void);                                  // 初始化 codec + 返回句柄
audio_board_handle_t audio_board_get_handle(void);
esp_err_t audio_board_key_init(esp_periph_set_handle_t set);                  // 注册板载按键
esp_err_t audio_board_sdcard_init(esp_periph_set_handle_t set, periph_sdcard_mode_t mode);  // 挂载 SD 卡
esp_err_t audio_board_deinit(audio_board_handle_t audio_board);
// 按键 ID 宏（板相关）：get_input_play_id / get_input_set_id / get_input_mode_id /
//                       get_input_volup_id / get_input_voldown_id / get_input_rec_id ...
// I2S 端口宏：CODEC_ADC_I2S_PORT / CODEC_DAC_I2S_PORT
```

## audio_hal（`components/audio_hal/include/audio_hal.h`）

```c
typedef struct audio_hal *audio_hal_handle_t;
typedef enum { AUDIO_HAL_CODEC_MODE_UNKNOWN=0, AUDIO_HAL_CODEC_MODE_DECODE,
               AUDIO_HAL_CODEC_MODE_ENCODE, AUDIO_HAL_CODEC_MODE_LINE,
               AUDIO_HAL_CODEC_MODE_ADC_DAC } audio_hal_codec_mode_t;
typedef enum { AUDIO_HAL_CTRL_START=0, AUDIO_HAL_CTRL_STOP } audio_hal_ctrl_t;
audio_hal_handle_t audio_hal_init(audio_hal_codec_config_t *cfg, audio_hal_func_t *func);
esp_err_t audio_hal_ctrl_codec(audio_hal_handle_t audio_hal, audio_hal_codec_mode_t mode, audio_hal_ctrl_t ctrl);
esp_err_t audio_hal_codec_iface_config(audio_hal_handle_t audio_hal, audio_hal_codec_mode_t mode, audio_hal_codec_i2s_iface_t *iface);
esp_err_t audio_hal_set_mute(audio_hal_handle_t audio_hal, bool mute);
esp_err_t audio_hal_set_volume(audio_hal_handle_t audio_hal, int volume);   // 0..100
esp_err_t audio_hal_get_volume(audio_hal_handle_t audio_hal, int *volume);
esp_err_t audio_hal_enable_pa(audio_hal_handle_t audio_hal, bool enable);
```

## Services（`components/*/include/`）

```c
// bluetooth_service.h
audio_element_handle_t bluetooth_service_create_stream(void);
esp_periph_handle_t    bluetooth_service_create_periph(void);
esp_err_t              bluetooth_service_start(bluetooth_service_cfg_t *config);

// ota_service.h
#define OTA_SERVICE_DEFAULT_CONFIG() ...
// 错误码 ota_service_err_reason_t：见 recipes/ota_service.md 与头文件

// wifi_service.h / input_key_service.h / battery_service.h / coredump_upload_service.h / dueros_service.h
//   每个有 service_<name>_create(...) / 对应 cfg 与事件，详见 docs/en/api-reference/services/
```

### display_service（`components/display_service/include/display_service.h`）

```c
typedef enum {
    DISPLAY_PATTERN_UNKNOWN = 0, DISPLAY_PATTERN_WIFI_SETTING = 1,
    DISPLAY_PATTERN_WIFI_CONNECTTING = 2, DISPLAY_PATTERN_WIFI_CONNECTED = 3,
    DISPLAY_PATTERN_WIFI_DISCONNECTED = 4, DISPLAY_PATTERN_WIFI_SETTING_FINISHED = 5,
    DISPLAY_PATTERN_BT_CONNECTTING = 6, DISPLAY_PATTERN_BT_CONNECTED = 7, DISPLAY_PATTERN_BT_DISCONNECTED = 8,
    DISPLAY_PATTERN_RECORDING_START = 9, DISPLAY_PATTERN_RECORDING_STOP = 10,
    DISPLAY_PATTERN_RECOGNITION_START = 11, DISPLAY_PATTERN_RECOGNITION_STOP = 12,
    DISPLAY_PATTERN_WAKEUP_ON = 13, DISPLAY_PATTERN_WAKEUP_FINISHED = 14,
    DISPLAY_PATTERN_MUSIC_ON = 15, DISPLAY_PATTERN_MUSIC_FINISHED = 16,
    DISPLAY_PATTERN_VOLUME = 17, DISPLAY_PATTERN_MUTE_ON = 18, DISPLAY_PATTERN_MUTE_OFF = 19,
    DISPLAY_PATTERN_TURN_ON = 20, DISPLAY_PATTERN_TURN_OFF = 21,
    DISPLAY_PATTERN_BATTERY_LOW = 22, DISPLAY_PATTERN_BATTERY_CHARGING = 23, DISPLAY_PATTERN_BATTERY_FULL = 24,
    DISPLAY_PATTERN_POWERON_INIT = 25, DISPLAY_PATTERN_WIFI_NO_CFG = 26,
    DISPLAY_PATTERN_SPEECH_BEGIN = 27, DISPLAY_PATTERN_SPEECH_OVER = 28,
    DISPLAY_PATTERN_MAX,
} display_pattern_t;

typedef struct display_service_impl *display_service_handle_t;

typedef struct {
    periph_service_config_t based_cfg;   // 含 service_ioctl = <driver>_pattern 函数指针
    void                    *instance;   // 驱动实例（如 led_bar_ws2812_handle_t）
} display_service_config_t;

display_service_handle_t display_service_create(display_service_config_t *cfg);
esp_err_t display_service_set_pattern(void *handle, int disp_pattern, int value);  // value 用于 VOLUME 等
esp_err_t display_destroy(display_service_handle_t handle);
```

> 板级高层入口：`audio_board_led_init()`（各板 `board.c` 实现，按 `CONFIG_*_BOARD` 自动选驱动）。完整用法见 `recipes/display_service.md`。

### LED 驱动后端（`components/display_service/led_bar/include/`）

```c
// led_bar_ws2812.h — NeoPixel 灯带
typedef struct led_bar_ws2812 *led_bar_ws2812_handle_t;
led_bar_ws2812_handle_t led_bar_ws2812_init(gpio_num_t gpio_num, int led_num);
esp_err_t led_bar_ws2812_pattern(void *handle, int pat, int value);

// led_bar_aw2013.h — 3 颗 RGB（LyraTD-MSC）
esp_periph_handle_t led_bar_aw2013_init(void);
esp_err_t led_bar_aw2013_pattern(void *handle, int pat, int value);

// led_bar_is31x.h — IS31x 灯条
esp_periph_handle_t led_bar_is31x_init(void);
esp_err_t led_bar_is31x_pattern(void *handle, int pat, int value);

// PWM 单 LED：用 esp_peripherals 的 periph_led（led_indicator 模式，LyraT V4.3 board.c 即此）
```

### playlist（`components/playlist/include/`）

```c
// playlist.h — 列表管理器（多列表）
typedef playlist_operator_t *playlist_operator_handle_t;
typedef struct playlist_handle *playlist_handle_t;

typedef struct {
    esp_err_t (*show)(void*);  esp_err_t (*save)(void*, const char *url);
    esp_err_t (*next)(void*, int step, char **url_buff);
    esp_err_t (*prev)(void*, int step, char **url_buff);
    esp_err_t (*reset)(void*); esp_err_t (*choose)(void*, int url_id, char **url_buff);
    esp_err_t (*current)(void*, char **url_buff); esp_err_t (*destroy)(void*);
    bool (*exist)(void*, const char *url);
    int (*get_url_num)(void*); int (*get_url_id)(void*);
    playlist_type_t type;  // PLAYLIST_SDCARD / PLAYLIST_FLASH / PLAYLIST_DRAM / PLAYLIST_PARTITION
    esp_err_t (*remove_by_url)(void*, const char *url);
    esp_err_t (*remove_by_id)(void*, uint16_t url_id);
} playlist_operation_t;

playlist_handle_t playlist_create(void);
esp_err_t playlist_add(playlist_handle_t handle, playlist_operator_handle_t list_handle, uint8_t list_id);
esp_err_t playlist_checkout_by_id(playlist_handle_t handle, uint8_t id);
int      playlist_get_current_list_url_num(playlist_handle_t handle);
esp_err_t playlist_get_current_list_url(playlist_handle_t handle, char **url_buff);
esp_err_t playlist_next(playlist_handle_t handle, int step, char **url_buff);
esp_err_t playlist_prev(playlist_handle_t handle, int step, char **url_buff);
esp_err_t playlist_choose(playlist_handle_t handle, int url_id, char **url_buff);

// sdcard_scan.h — 扫描 SD 卡文件，经回调存入列表
typedef void (*sdcard_scan_cb_t)(void *user_data, char *url);
esp_err_t sdcard_scan(sdcard_scan_cb_t cb, const char *path, int depth,
                      const char *file_extension[], int filter_num, void *user_data);

// 四种存储后端（各自 create 返回 playlist_operator_handle_t*）：
//   sdcard_list_create(&h)  / dram_list_create(&h)
//   flash_list_create(&h)   / partition_list_create(&h)   // partition 需加 subtype 0x06/0x07 两个分区
```

> 完整切歌模式与按键集成见 `recipes/playlist.md`。

## Speech Recognition / Recorder（`components/audio_recorder/include/` + `docs/en/api-reference/speech-recognition/`）

### audio_recorder（高层录音器，整合 AFE + WakeNet + MultiNet + VAD + 编码）

```c
// audio_recorder.h
typedef struct {
    enum audio_recorder_event_type_t {
        AUDIO_REC_WAKEUP_START = -100,   // event_data = recorder_sr_wakeup_result_t
        AUDIO_REC_WAKEUP_END,
        AUDIO_REC_VAD_START,
        AUDIO_REC_VAD_END,
        AUDIO_REC_COMMAND_DECT = 0       // type 本身即命令 id；event_data = recorder_sr_mn_result_t
    } type;
    void *event_data; size_t data_len;
} audio_rec_evt_t;

typedef esp_err_t (*rec_event_cb_t)(audio_rec_evt_t *event, void *user_data);
typedef int (*recorder_data_read_t)(void *buffer, int buf_sz, void *user_ctx, TickType_t ticks);  // 喂 AFE

typedef struct {
    int pinned_core, task_prio, task_size;
    rec_event_cb_t event_cb; void *user_data;
    recorder_data_read_t read;          // 关键：给 AFE 喂 raw 音频的回调
    void *sr_handle; recorder_sr_iface_t *sr_iface;   // 由 recorder_sr_create 填
    int wakeup_time, vad_start, vad_off, wakeup_end;  // 单位 ms
    void *encoder_handle; recorder_encoder_iface_t *encoder_iface;
} audio_rec_cfg_t;

#define AUDIO_RECORDER_DEFAULT_CFG() { .pinned_core=1, .task_prio=10, .task_size=4096, \
    .wakeup_time=10000, .vad_start=160, .vad_off=300, .wakeup_end=900, ... }

audio_rec_handle_t audio_recorder_create(audio_rec_cfg_t *cfg);
esp_err_t audio_recorder_trigger_start(audio_rec_handle_t handle);   // 强制开始（不依赖唤醒词）
esp_err_t audio_recorder_trigger_stop(audio_rec_handle_t handle);
esp_err_t audio_recorder_wakenet_enable(audio_rec_handle_t handle, bool enable);
esp_err_t audio_recorder_multinet_enable(audio_rec_handle_t handle, bool enable);
esp_err_t audio_recorder_vad_check_enable(audio_rec_handle_t handle, bool enable);
int      audio_recorder_data_read(audio_rec_handle_t handle, void *buffer, int length, TickType_t ticks);
esp_err_t audio_recorder_destroy(audio_rec_handle_t handle);
bool     audio_recorder_get_wakeup_state(audio_rec_handle_t handle);
```

### recorder_sr（`recorder_sr.h`）— AFE + WakeNet + MultiNet 配置

```c
// recorder_sr.h（CONFIG 由板宏决定，如 AUDIO_ADC_INPUT_CH_FORMAT）
recorder_sr_cfg_t DEFAULT_RECORDER_SR_CFG(const char *sr_input_fmt, const char *model_partition_label,
                                          afe_type_t afe_type, afe_mode_t afe_mode);
esp_err_t recorder_sr_create(recorder_sr_cfg_t *cfg, recorder_sr_iface_t **iface);
// 关键字段（afe_cfg 即 esp_afe_sr_iface_config_t，来自 esp-sr）：
//   .afe_cfg->wakenet_init / .vad_mode / .aec_init / .agc_mode / .memory_alloc_mode / .pcm_config
//   .multinet_init / .mn_language (ESP_MN_CHINESE / ESP_MN_ENGLISH)
esp_err_t recorder_sr_reset_speech_cmd(void *handle, const char *cmd_str, char *errinfo);
```

### esp_vad（`esp-sr/include/<chip>/esp_vad.h`，独立底层用法）

```c
typedef struct esp_vad *vad_handle_t;  // 来自 esp-sr 预编译库
typedef enum { VAD_SILENCE, VAD_SPEECH } vad_state_t;
typedef enum { VAD_MODE_0..VAD_MODE_4 } vad_mode_t;  // 越大越严格

vad_handle_t vad_create(vad_mode_t mode);
vad_state_t  vad_process(vad_handle_t vad, int16_t *data, int sample_rate, int length_ms);  // 8/16/32kHz, 10/20/30ms
void         vad_destroy(vad_handle_t vad);
```

> `esp_vad.h` / `esp_afe_sr_iface.h` / `esp_mn_models.h` 在 `esp-sr` 子模块内，需 `git clone --recursive`。完整事件流与模型分区见 `recipes/speech_recognition.md`。
