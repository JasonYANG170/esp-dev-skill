# ESP-Skainet / esp-sr API 速查

> 全部签名取自 `espressif-repos/esp-sr/include/esp32s3/` 与 `espressif-repos/esp-skainet/components/` 真实头文件。
> 仅收录可验证的函数/结构/枚举/宏。芯片相关 API 以 ESP32-S3 路径为准（其它目标在 `include/<target>/` 下同名）。

## 1. 模型加载（model_path.h）

```c
// espressif-repos/esp-sr/src/include/model_path.h
typedef struct {
    char **model_name;
    char **model_info;
    esp_partition_t *partition;
    void *mmap_handle;
    int num;
    srmodel_data_t **model_data;
} srmodel_list_t;

srmodel_list_t *esp_srmodel_init(const char *partition_label);
void            esp_srmodel_deinit(srmodel_list_t *models);
void            esp_srmodel_deinit_with_refcount(srmodel_list_t *models);

// keyword1=ESP_WN_PREFIX/ESP_MN_PREFIX, keyword2=ESP_MN_CHINESE/ESP_MN_ENGLISH/"alexa" 等
char *esp_srmodel_filter(srmodel_list_t *models, const char *keyword1, const char *keyword2);
int   esp_srmodel_exists(srmodel_list_t *models, char *model_name);
char *esp_srmodel_get_wake_words(srmodel_list_t *models, char *model_name);
srmodel_list_t *get_static_srmodels(void);
```

## 2. WakeNet（esp_wn_iface.h / esp_wn_models.h）

```c
// esp_wn_models.h
#define ESP_WN_PREFIX "wn"
const esp_wn_iface_t *esp_wn_handle_from_name(const char *model_name);
char *esp_wn_wakeword_from_name(const char *model_name);

// esp_wn_iface.h —— 接口结构体方法（通过 handle-> 调用）
typedef model_iface_data_t* (*esp_wn_iface_op_create_t)(const void *model_name, det_mode_t det_mode);
typedef int                 (*esp_wn_iface_op_get_samp_chunksize_t)(model_iface_data_t *model);
typedef int                 (*esp_wn_iface_op_get_channel_num_t)(model_iface_data_t *model);
typedef int                 (*esp_wn_iface_op_get_samp_rate_t)(model_iface_data_t *model);
typedef int                 (*esp_wn_iface_op_get_word_num_t)(model_iface_data_t *model);
typedef char*               (*esp_wn_iface_op_get_word_name_t)(model_iface_data_t *model, int word_index);
typedef int                 (*esp_wn_iface_op_set_det_threshold_t)(model_iface_data_t *model, float det_threshold, int word_index);
typedef int                 (*esp_wn_iface_op_reset_det_threshold_t)(model_iface_data_t *model);
typedef float               (*esp_wn_iface_op_get_det_threshold_t)(model_iface_data_t *model, int word_index);
typedef wakenet_state_t     (*esp_wn_iface_op_detect_t)(model_iface_data_t *model, int16_t *samples);
typedef void                (*esp_wn_iface_op_clean_t)(model_iface_data_t *model);
typedef void                (*esp_wn_iface_op_destroy_t)(model_iface_data_t *model);
// 还有 get_start_point / get_triggered_channel / get_vol_gain / detect_mfcc / get_mfcc_data

typedef struct {
    esp_wn_iface_op_create_t             create;
    esp_wn_iface_op_get_start_point_t    get_start_point;
    esp_wn_iface_op_get_samp_chunksize_t get_samp_chunksize;
    esp_wn_iface_op_get_channel_num_t    get_channel_num;
    esp_wn_iface_op_get_samp_rate_t      get_samp_rate;
    esp_wn_iface_op_get_word_num_t       get_word_num;
    esp_wn_iface_op_get_word_name_t      get_word_name;
    esp_wn_iface_op_set_det_threshold_t  set_det_threshold;
    esp_wn_iface_op_reset_det_threshold_t reset_det_threshold;
    esp_wn_iface_op_get_det_threshold_t  get_det_threshold;
    esp_wn_iface_op_get_triggered_channel_t get_triggered_channel;
    esp_wn_iface_op_get_vol_gain_t       get_vol_gain;
    esp_wn_iface_op_detect_t             detect;
    esp_wn_iface_op_detect_mfcc_t        detect_mfcc;
    esp_wn_iface_op_get_mfcc_data_t      get_mfcc_data;
    esp_wn_iface_op_clean_t              clean;
    esp_wn_iface_op_destroy_t            destroy;
} esp_wn_iface_t;
```

枚举：
```c
typedef enum { WAKENET_NO_DETECT=0, WAKENET_CHANNEL_VERIFIED=-1, WAKENET_DETECTED=1 } wakenet_state_t;
typedef enum {
    DET_MODE_90=0, DET_MODE_95=1,
    DET_MODE_2CH_90=2, DET_MODE_2CH_95=3,
    DET_MODE_3CH_90=4, DET_MODE_3CH_95=5,
    DET_MODE_90_COPY_PARAMS=6,
} det_mode_t;
```

## 3. MultiNet（esp_mn_iface.h / esp_mn_models.h / esp_mn_speech_commands.h）

```c
// esp_mn_models.h
#define ESP_MN_PREFIX "mn"
#define ESP_MN_ENGLISH "en"
#define ESP_MN_CHINESE "cn"
esp_mn_iface_t *esp_mn_handle_from_name(char *model_name);
char *esp_mn_language_from_name(char *model_name);

// esp_mn_iface.h 关键宏
#define ESP_MN_RESULT_MAX_NUM 5
#define ESP_MN_MAX_PHRASE_NUM 400
#define ESP_MN_MAX_PHRASE_LEN 63
#define ESP_MN_MIN_PHRASE_LEN 2

typedef enum { ESP_MN_STATE_DETECTING=0, ESP_MN_STATE_DETECTED=1, ESP_MN_STATE_TIMEOUT=2 } esp_mn_state_t;
typedef enum { ESP_MN_LOAD_FROM_PSRAM=0, ESP_MN_LOAD_FROM_PSRAM_FLASH=1, ESP_MN_LOAD_FROM_FLASH=2 } esp_mn_loader_mode_t;
typedef enum { ESP_MN_GREEDY_SEARCH=0, ESP_MN_BEAM_SEARCH=1, ESP_MN_BEAM_SEARCH_WITH_FST=2 } esp_mn_search_method_t;

typedef struct {
    esp_mn_state_t state;
    int num;
    int   command_id[ESP_MN_RESULT_MAX_NUM];
    int   phrase_id[ESP_MN_RESULT_MAX_NUM];
    float prob[ESP_MN_RESULT_MAX_NUM];
    char  string[256];
    char  raw_string[256];
} esp_mn_results_t;

typedef struct { char *string; char *phonemes; int16_t command_id; float threshold; int16_t *wave; } esp_mn_phrase_t;
typedef struct { int16_t num; esp_mn_phrase_t **phrases; } esp_mn_error_t;

// 接口方法（handle-> 调用）
typedef model_iface_data_t* (*esp_mn_iface_op_create_t)(const char *model_name, int duration);
typedef int   (*esp_mn_iface_op_get_samp_chunksize_t)(model_iface_data_t *model);
typedef int   (*esp_mn_iface_op_set_det_threshold_t)(model_iface_data_t *model, float det_threshold);
typedef esp_mn_state_t (*esp_mn_iface_op_detect_t)(model_iface_data_t *model, int16_t *samples);
typedef esp_mn_results_t* (*esp_mn_iface_op_get_results_t)(model_iface_data_t *model);
typedef void  (*esp_mn_iface_op_clean_t)(model_iface_data_t *model_data);
typedef void  (*esp_mn_iface_op_destroy_t)(model_iface_data_t *model);
typedef esp_mn_error_t* (*esp_wn_iface_op_set_speech_commands)(model_iface_data_t *model_data, esp_mn_node_t *mn_command_root);
typedef model_iface_data_t* (*esp_mn_iface_op_switch_loader_mode_t)(model_iface_data_t *model, esp_mn_loader_mode_t mode);
typedef void  (*esp_mn_iface_op_print_active_speech_commands)(model_iface_data_t *model_data);
typedef int   (*esp_mn_iface_op_check_speech_command)(model_iface_data_t *model_data, const char *str);
```

命令管理 API（`esp_mn_speech_commands.h`，全局链表）：
```c
esp_err_t       esp_mn_commands_alloc(const esp_mn_iface_t *multinet, model_iface_data_t *model_data);
esp_err_t       esp_mn_commands_free(void);
esp_err_t       esp_mn_commands_add(int command_id, const char *string);
esp_err_t       esp_mn_commands_phoneme_add(int command_id, const char *string, const char *phonemes);
esp_err_t       esp_mn_commands_modify(const char *old_string, const char *new_string);
esp_err_t       esp_mn_commands_remove(const char *string);
esp_err_t       esp_mn_commands_clear(void);
char           *esp_mn_commands_get_string(int command_id);
esp_mn_phrase_t*esp_mn_commands_get_from_index(int index);
esp_mn_phrase_t*esp_mn_commands_get_from_string(const char *string);
esp_mn_error_t *esp_mn_commands_update(void);
esp_mn_phrase_t*esp_mn_phrase_alloc(int command_id, const char *string);

// esp_process_sdkconfig.h —— 从 Kconfig 批量导入
esp_mn_error_t *esp_mn_commands_update_from_sdkconfig(const esp_mn_iface_t *multinet, model_iface_data_t *model_data);
```

## 4. Audio Front-End（esp_afe_config.h / esp_afe_sr_iface.h）

```c
// esp_afe_config.h
afe_config_t *afe_config_init(const char *input_format, srmodel_list_t *models,
                              afe_type_t type, afe_mode_t mode);
afe_config_t *afe_config_check(afe_config_t *afe_config);
afe_config_t *afe_config_alloc(void);
afe_config_t *afe_config_copy(afe_config_t *dst, const afe_config_t *src);
void          afe_config_free(afe_config_t *afe_config);
void          afe_config_print(const afe_config_t *afe_config);
bool          afe_parse_input_format(const char *input_format, afe_pcm_config_t *pcm_config);
void          afe_parse_input(int16_t *data, int frame_size, int16_t *mic_data, int16_t *ref_data, afe_pcm_config_t *pcm_config);
int16_t      *afe_adjust_gain(int16_t *data, int frame_size, float factor);

// esp_afe_sr_iface.h —— 由 esp_afe_handle_from_config() 返回
const esp_afe_sr_iface_t *esp_afe_handle_from_config(afe_config_t *afe_config);
```

接口方法表（`esp_afe_sr_iface_t`，全部 `afe_handle->` 调用）：
```c
create_from_config / feed / fetch / fetch_with_delay / reset_buffer
get_feed_chunksize / get_fetch_chunksize
get_channel_num / get_feed_channel_num / get_fetch_channel_num / get_samp_rate
set_wakenet_threshold(index/*1|2*/, threshold/*0.4~0.9999*/)
reset_wakenet_threshold(index)
disable_wakenet / enable_wakenet
disable_aec / enable_aec
disable_se / enable_se
disable_vad / enable_vad / reset_vad
disable_ns / enable_ns
disable_agc / enable_agc
add_wakenet_model(model_name)
print_pipeline()
destroy()
```

fetch 结果结构：
```c
typedef struct afe_fetch_result_t {
    int16_t *data;            // 目标通道增强音频
    int data_size;            // 字节
    int16_t *vad_cache;       // VAD 缓存（vad_cache_size>0 时有效）
    int vad_cache_size;
    float data_volume;        // 音量 dB
    wakenet_state_t wakeup_state;
    int wake_word_index;      // 从 1 开始
    int wakenet_model_index;  // 多模型时区分，从 1 开始
    vad_state_t vad_state;    // VAD_SILENCE / VAD_SPEECH
    int trigger_channel_id;
    int wake_word_length;
    int ret_value;            // fetch 返回状态
    int16_t *raw_data;        // 多通道原始输出
    int raw_data_channels;
    float ringbuff_free_pct;
    void *reserved;
} afe_fetch_result_t;
```

枚举：
```c
typedef enum { AFE_MODE_LOW_COST=0, AFE_MODE_HIGH_PERF=1 } afe_mode_t;
typedef enum { AFE_TYPE_SR=0, AFE_TYPE_VC=1, AFE_TYPE_VC_8K=2, AFE_TYPE_FD=3 } afe_type_t;
typedef enum {
    AFE_MEMORY_ALLOC_MORE_INTERNAL=1,
    AFE_MEMORY_ALLOC_INTERNAL_PSRAM_BALANCE=2,
    AFE_MEMORY_ALLOC_MORE_PSRAM=3
} afe_memory_alloc_mode_t;
typedef enum { AFE_NS_MODE_WEBRTC=0, AFE_NS_MODE_NET=1 } afe_ns_mode_t;
typedef enum { AFE_AGC_MODE_WEBRTC=0, AFE_AGC_MODE_WAKENET=1 } afe_agc_mode_t;
typedef enum {
    AFE_MN_PEAK_AGC_MODE_1=-9, AFE_MN_PEAK_AGC_MODE_2=-6,
    AFE_MN_PEAK_AGC_MODE_3=-3, AFE_MN_PEAK_NO_AGC=0
} afe_mn_peak_agc_mode_t;
```

## 5. 板级 / 播放 / SD（hardware_driver/include/esp_board_init.h）

```c
esp_err_t esp_board_init(uint32_t sample_rate, int channel_format, int bits_per_chan);
esp_err_t esp_sdcard_init(char *mount_point, size_t max_files);
esp_err_t esp_sdcard_deinit(char *mount_point);
esp_err_t esp_get_feed_data(bool is_get_raw_channel, int16_t *buffer, int buffer_len);
int       esp_get_feed_channel(void);
char     *esp_get_input_format(void);
esp_err_t esp_audio_play(const int16_t *data, int length, TickType_t ticks_to_wait);
esp_err_t esp_audio_set_play_vol(int volume);
esp_err_t esp_audio_get_play_vol(int *volume);
esp_err_t get_i2s_data(char *buffer, int buffer_len);
esp_err_t FatfsComboWrite(const void *buffer, int size, int count, FILE *stream);
```

## 6. DOA（esp_doa.h）

```c
typedef struct doa_handle_t doa_handle_t;
doa_handle_t *esp_doa_create(int fs, float resolution, float d_mics, int chunksize);
float         esp_doa_process(doa_handle_t *handle, int16_t *left, int16_t *right);
void          esp_doa_destroy(doa_handle_t *handle);
```

## 7. 中文 TTS（esp-tts/esp_tts_chinese/include/）

```c
// esp_tts.h
esp_tts_voice_t *esp_tts_voice_set_init(const esp_tts_voice_t *tv, int16_t *voicedata);
esp_tts_handle_t *esp_tts_create(esp_tts_voice_t *voice);
int   esp_tts_parse_chinese(esp_tts_handle_t *tts, char *str);

// esp_tts_player.h
short *esp_tts_stream_play(esp_tts_handle_t *tts, int *len, int speed);
void   esp_tts_stream_reset(esp_tts_handle_t *tts);
```

## 8. player 组件（components/player/esp_skainet_player.h）

```c
void *esp_skainet_player_create(int ringbuf_size, unsigned int core_num);
void  esp_skainet_player_play(void *handle, const char *path);
void  esp_skainet_player_pause(void *handle);
void  esp_skainet_player_continue(void *handle);
void  esp_skainet_player_exit(void *handle);
int   esp_skainet_player_get_state(void *handle);
void  esp_skainet_player_increase_vol(void *handle);
void  esp_skainet_player_decrease_vol(void *handle);
```

## 9. perf_tester 控制台命令（components/perf_tester）

通过串口控制台交互（来自 perf_tester README）：
```
help
config <fast|norm> <all|none|pink|pub> <all|none|0|5|10>   # 模式/噪声类型/SNR
start
```

测试工程额外注册的命令（`test/wakenet/main/wakenet_main.c`）：
```
rar   # 跑 /sdcard/{wn_name}.csv 唤醒率测试，输出 {wn_name}.log
far   # 跑 /sdcard/far_48h.csv 误唤醒测试（48 小时集）
```

### 9.1 perf_tester C API（components/perf_tester）

```c
// perf_tester_cmd.h
typedef struct {
    char mode[32];    // "fast" / "norm"
    char noise[32];   // "all" / "none" / "pink" / "pub"
    char snr[32];     // "all" / "none" / "0" / "5" / "10"
    int  flag;        // update flag
} perf_tester_config_t;

void                   register_perf_tester_config_cmd(void);
void                   register_perf_tester_start_cmd(esp_console_cmd_func_t start_func);
perf_tester_config_t*  get_perf_tester_config(void);
bool                   check_noise(const char *filename, const char *noise);
bool                   check_snr(const char *filename, const char *snr);
```

```c
// wn_perf_tester.h
typedef enum {
    TESTER_PCM_3CH = 0,   // 3 通道 PCM [mic1, mic2, ref, ...]
    TESTER_WAV_3CH = 1,   // 3 通道 WAV（带 WAV 头，自动解码）
    TESTER_PCM_1CH = 2,   // 单通道 PCM
    TESTER_WAV_1CH = 3,   // 单通道 WAV
} tester_audio_t;

void* offline_wn_tester_start(const char *csv_file,
                              const char *log_file,
                              const esp_afe_sr_iface_t *afe_handle,  // NULL 时内部从 config 取
                              afe_config_t *afe_config,
                              int audio_type,                         // TESTER_WAV_3CH 等
                              perf_tester_config_t *config);
void  offline_wn_tester_stop(void *tester);
```

> 报告格式（`print_wn_report`）：依次打印 `Number of files: N`、`Tester PSRAM/SRAM`、`AFE CPU/PSRAM/SRAM`、每个文件的 `FileN, trigger/required/truth times`、`Total trigger/required/truth times`、`TEST DONE`。pytest 脚本据此断言 `trigger_times >= required_times`、`AFE PSRAM < 1120KB`、`AFE SRAM < 32KB`。

### 9.2 自定义板移植 API（hardware_driver/boards/include/bsp_board.h）

板级 API 声明（`bsp_*`），由 `esp_board_init.c` 转发为 `esp_*` 公共 API：

```c
// 板级初始化（I2S + I2C + codec + SD）
esp_err_t bsp_board_init(uint32_t sample_rate, int channel_format, int bits_per_chan);

// 取录音数据：is_get_raw_channel=false 时按 input_format 重排通道
esp_err_t bsp_get_feed_data(bool is_get_raw_channel, int16_t *buffer, int buffer_len);
int       bsp_get_feed_channel(void);             // 返回 codec 物理通道数
char*     bsp_get_input_format(void);             // 返回 "MR"/"MMR"/"RMNM"/"MN" 等

esp_err_t bsp_sdcard_init(char *mount_point, size_t max_files);
esp_err_t bsp_sdcard_deinit(char *mount_point);
esp_err_t bsp_audio_play(const int16_t *data, int length, TickType_t ticks_to_wait);
esp_err_t bsp_audio_set_play_vol(int volume);
esp_err_t bsp_audio_get_play_vol(int *volume);
```

选板 Kconfig（`components/hardware_driver/Kconfig.projbuild` 的 `choice AUDIO_BOARD`）：
```
CONFIG_ESP32_KORVO_V1_1_BOARD          / CONFIG_ESP32_S3_BOX_BOARD
CONFIG_ESP32_S3_KORVO_1_V4_0_BOARD     / CONFIG_ESP32_S3_BOX_3_BOARD
CONFIG_ESP32_S3_KORVO_2_V3_0_BOARD     / CONFIG_ESP32_S3_EYE_BOARD
CONFIG_ESP32_P4_FUNCTION_EV_BOARD
CONFIG_ESP_CUSTOM_BOARD                # → #include "esp_custom_board.h"（自定义板入口）
```

> 移植不变量：`bsp_get_feed_channel()` 返回值（物理通道数）、`bsp_get_input_format()` 字符串长度（重排后输出通道数）、`bsp_get_feed_data()` 的重排逻辑三者必须自洽，否则触发 `assert(nch == feed_channel)`。各板真实返回值：korvo-1=`"RMNM"`、esp32-korvo=`"MRNN"`、esp32p4-function-ev=`"MR"`、esp32s3-eye=`"MN"`。
