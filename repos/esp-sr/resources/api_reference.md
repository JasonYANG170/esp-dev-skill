# ESP-SR API Quick Reference

> 所有签名、结构体、枚举均来自仓库 `include/<target>/`、`src/include/`、`esp-tts/esp_tts_chinese/include/` 的真实头文件。target 由 CMake 自动选择（如 `include/esp32s3/`）；下文示例路径以 esp32s3 为例，其他芯片同名同签名。
> AFE/WakeNet/MultiNet/VADNet 的 handle 是**函数指针结构体**（iface），通过 `->成员` 调用。

## 1. AFE（Audio Front-End）— 主入口

头文件：`esp_afe_sr_iface.h`、`esp_afe_config.h`、`esp_afe_sr_models.h`

### 类型

```c
typedef struct esp_afe_sr_data_t esp_afe_sr_data_t;   // 不透明实例

typedef struct afe_fetch_result_t {
    int16_t *data;            // 输出目标通道音频（单声道）
    int data_size;            // 字节
    int16_t *vad_cache;       // VAD 缓存（vad_cache_size>0 时有效）
    int vad_cache_size;
    float data_volume;        // 输入音量 dB（agc 前计算）
    wakenet_state_t wakeup_state;
    int wake_word_index;      // 命中唤醒词索引，从 1 起
    int wakenet_model_index;  // 多 wakenet 时哪个模型命中，从 1 起
    vad_state_t vad_state;
    int trigger_channel_id;   // 输出通道索引
    int wake_word_length;     // 唤醒词长度（样本数）
    int ret_value;            // fetch 返回状态
    int16_t *raw_data;        // 多通道原始输出
    int raw_data_channels;
    float ringbuff_free_pct;  // ringbuf 空闲比例，>0.5 表示繁忙
    void *reserved;
} afe_fetch_result_t;

typedef struct {
    esp_afe_sr_data_t *afe_data;
    const esp_afe_sr_iface_t *afe_handle;
    TaskHandle_t feed_task;
    TaskHandle_t fetch_task;
} afe_task_into_t;
```

### esp_afe_sr_iface_t 成员（函数指针）

```c
typedef struct {
    esp_afe_sr_data_t *(*create_from_config)(afe_config_t *afe_config);
    int  (*feed)(esp_afe_sr_data_t *afe, const int16_t *in);
    afe_fetch_result_t *(*fetch)(esp_afe_sr_data_t *afe);                          // 超时 2000ms
    afe_fetch_result_t *(*fetch_with_delay)(esp_afe_sr_data_t *afe, TickType_t t); // 自定义超时
    int  (*reset_buffer)(esp_afe_sr_data_t *afe);
    int  (*get_feed_chunksize)(esp_afe_sr_data_t *afe);
    int  (*get_fetch_chunksize)(esp_afe_sr_data_t *afe);
    int  (*get_channel_num)(esp_afe_sr_data_t *afe);       // = get_feed_channel_num
    int  (*get_feed_channel_num)(esp_afe_sr_data_t *afe);
    int  (*get_fetch_channel_num)(esp_afe_sr_data_t *afe);
    int  (*get_samp_rate)(esp_afe_sr_data_t *afe);
    int  (*set_wakenet_threshold)(esp_afe_sr_data_t *afe, int index, float threshold); // index=1/2, 阈值0.4~0.9999
    int  (*reset_wakenet_threshold)(esp_afe_sr_data_t *afe, int index);
    int  (*disable_wakenet)(esp_afe_sr_data_t *afe);
    int  (*enable_wakenet)(esp_afe_sr_data_t *afe);
    int  (*disable_aec)(esp_afe_sr_data_t *afe);
    int  (*enable_aec)(esp_afe_sr_data_t *afe);
    int  (*disable_se)(esp_afe_sr_data_t *afe);
    int  (*enable_se)(esp_afe_sr_data_t *afe);
    int  (*disable_vad)(esp_afe_sr_data_t *afe);
    int  (*enable_vad)(esp_afe_sr_data_t *afe);
    int  (*reset_vad)(esp_afe_sr_data_t *afe);
    int  (*disable_ns)(esp_afe_sr_data_t *afe);
    int  (*enable_ns)(esp_afe_sr_data_t *afe);
    int  (*disable_agc)(esp_afe_sr_data_t *afe);
    int  (*enable_agc)(esp_afe_sr_data_t *afe);
    int  (*add_wakenet_model)(esp_afe_sr_data_t *afe, const char *model_name);
    void (*print_pipeline)(esp_afe_sr_data_t *afe);
    void (*destroy)(esp_afe_sr_data_t *afe);
} esp_afe_sr_iface_t;

// 取 handle
const esp_afe_sr_iface_t *esp_afe_handle_from_config(const afe_config_t *config);
```

返回值约定：disable/enable 系列返回 `-1` 失败 / `0` 已禁用 / `1` 已启用；set/reset_threshold 返回 `-1` 失败 / `1` 成功。

## 2. AFE 配置 — afe_config_t

头文件：`esp_afe_config.h`

```c
typedef enum { AFE_MODE_LOW_COST = 0, AFE_MODE_HIGH_PERF = 1 } afe_mode_t;
typedef enum { AFE_TYPE_SR = 0, AFE_TYPE_VC = 1, AFE_TYPE_VC_8K = 2, AFE_TYPE_FD = 3 } afe_type_t;
typedef enum {
    AFE_MEMORY_ALLOC_MORE_INTERNAL = 1,
    AFE_MEMORY_ALLOC_INTERNAL_PSRAM_BALANCE = 2,
    AFE_MEMORY_ALLOC_MORE_PSRAM = 3
} afe_memory_alloc_mode_t;
typedef enum { AFE_NS_MODE_WEBRTC = 0, AFE_NS_MODE_NET = 1 } afe_ns_mode_t;
typedef enum { AFE_AGC_MODE_WEBRTC = 0, AFE_AGC_MODE_WAKENET = 1 } afe_agc_mode_t;
typedef enum {
    AFE_MN_PEAK_AGC_MODE_1 = -9, AFE_MN_PEAK_AGC_MODE_2 = -6,
    AFE_MN_PEAK_AGC_MODE_3 = -3, AFE_MN_PEAK_NO_AGC = 0
} afe_mn_peak_agc_mode_t;

typedef struct {
    int total_ch_num; int mic_num; uint8_t *mic_ids;
    int ref_num; uint8_t *ref_ids; int sample_rate;
} afe_pcm_config_t;

#define AFE_MAX_WAKEWORD_NUM 3

typedef struct {
    bool aec_init; aec_mode_t aec_mode; int aec_filter_length; aec_nlp_level_t aec_nlp_level;
    bool se_init;
    bool ns_init; char *ns_model_name; afe_ns_mode_t afe_ns_mode;
    bool vad_init; vad_mode_t vad_mode; char *vad_model_name;
    int vad_min_speech_ms; int vad_min_noise_ms; int vad_delay_ms;
    bool vad_mute_playback; bool vad_enable_channel_trigger;
    bool wakenet_init; char *wakenet_model_name; char *wakenet_model_name_2; det_mode_t wakenet_mode;
    bool agc_init; afe_agc_mode_t agc_mode; int agc_compression_gain_db; int agc_target_level_dbfs;
    afe_pcm_config_t pcm_config; afe_mode_t afe_mode; afe_type_t afe_type;
    int afe_perferred_core; int afe_perferred_priority; int afe_ringbuf_size;
    afe_memory_alloc_mode_t memory_alloc_mode; float afe_linear_gain;
    bool debug_init; bool fixed_first_channel; bool fixed_output_channel; bool output_playback_channel;
} afe_config_t;

afe_config_t *afe_config_init(const char *input_format, srmodel_list_t *models,
                              afe_type_t type, afe_mode_t mode);
afe_config_t *afe_config_check(afe_config_t *afe_config);   // 自动修正冲突
bool afe_parse_input_format(const char *input_format, afe_pcm_config_t *pcm_config);
void afe_parse_input(int16_t *data, int frame_size, int16_t *mic_data, int16_t *ref_data, afe_pcm_config_t *pcm_config);
void afe_parse_data(int16_t *data, int frame_size, int channel_num, int16_t *out_data);   // 交错→连续
void afe_format_data(int16_t *data, int frame_size, int channel_num, int16_t *out_data);   // 连续→交错
int16_t *afe_adjust_gain(int16_t *data, int frame_size, float factor);   // 原地改
afe_config_t *afe_config_copy(afe_config_t *dst, const afe_config_t *src);
void afe_config_print(const afe_config_t *afe_config);
afe_config_t *afe_config_alloc();
void afe_config_free(afe_config_t *afe_config);
```

`input_format`: `M`=麦克风，`R`=播放参考，`N`=未用。例：`MMNR`、`MR`、`M`。

## 3. 模型管理 — model_path.h

```c
#define SRMODEL_STRING_LENGTH 32
#define MODEL_NAME_MAX_LENGTH 64
#define ESP_WN_PREFIX  "wn"
#define ESP_MN_PREFIX  "mn"
#define ESP_MN_ENGLISH "en"
#define ESP_MN_CHINESE "cn"

typedef struct {
    int num; char **files; char **data; int *sizes;
} srmodel_data_t;

typedef struct {
    char **model_name; char **model_info;
    esp_partition_t *partition; void *mmap_handle;
    int num; srmodel_data_t **model_data;
} srmodel_list_t;

srmodel_list_t *esp_srmodel_init(const char *partition_label);   // 参数固定为 "model"
void esp_srmodel_deinit(srmodel_list_t *models);
void esp_srmodel_deinit_with_refcount(srmodel_list_t *models);   // 引用计数版本
char *esp_srmodel_filter(srmodel_list_t *models, const char *keyword1, const char *keyword2);
int   esp_srmodel_exists(srmodel_list_t *models, char *model_name);
char *esp_srmodel_get_wake_words(srmodel_list_t *models, char *model_name);
char *get_model_base_path(void);
srmodel_list_t *get_static_srmodels(void);
srmodel_list_t *srmodel_load(const void *root);
srmodel_list_t *srmodel_host_init(const char *filename);    // host 测试
srmodel_list_t *srmodel_spiffs_init(const esp_partition_t *part);   // SPIFFS
```

## 4. WakeNet — esp_wn_iface.h / esp_wn_models.h

```c
typedef enum {
    WAKENET_NO_DETECT = 0, WAKENET_CHANNEL_VERIFIED = -1, WAKENET_DETECTED = 1
} wakenet_state_t;

typedef enum {
    DET_MODE_90 = 0, DET_MODE_95 = 1,
    DET_MODE_2CH_90 = 2, DET_MODE_2CH_95 = 3,
    DET_MODE_3CH_90 = 4, DET_MODE_3CH_95 = 5,
    DET_MODE_90_COPY_PARAMS = 6
} det_mode_t;

typedef struct {
    esp_wn_iface_op_create_t create;            // create(model_name, det_mode)
    esp_wn_iface_op_get_start_point_t get_start_point;
    esp_wn_iface_op_get_samp_chunksize_t get_samp_chunksize;
    esp_wn_iface_op_get_channel_num_t get_channel_num;
    esp_wn_iface_op_get_samp_rate_t get_samp_rate;
    esp_wn_iface_op_get_word_num_t get_word_num;
    esp_wn_iface_op_get_word_name_t get_word_name;       // index 从 1 起
    esp_wn_iface_op_set_det_threshold_t set_det_threshold;   // 阈值 0.4~0.9999, word_index 从 1
    esp_wn_iface_op_reset_det_threshold_t reset_det_threshold;
    esp_wn_iface_op_get_det_threshold_t get_det_threshold;
    esp_wn_iface_op_get_triggered_channel_t get_triggered_channel;
    esp_wn_iface_op_get_vol_gain_t get_vol_gain;
    esp_wn_iface_op_detect_t detect;             // 返回唤醒词 index，0=未唤醒
    esp_wn_iface_op_detect_mfcc_t detect_mfcc;
    esp_wn_iface_op_get_mfcc_data_t get_mfcc_data;
    esp_wn_iface_op_clean_t clean;
    esp_wn_iface_op_destroy_t destroy;
} esp_wn_iface_t;

const esp_wn_iface_t *esp_wn_handle_from_name(const char *model_name);
char *esp_wn_wakeword_from_name(const char *model_name);
```

## 5. MultiNet — esp_mn_iface.h / esp_mn_models.h / esp_mn_speech_commands.h

```c
#define ESP_MN_RESULT_MAX_NUM 5
#define ESP_MN_MAX_PHRASE_NUM 400
#define ESP_MN_MAX_PHRASE_LEN 63
#define ESP_MN_MIN_PHRASE_LEN 2

typedef enum { ESP_MN_STATE_DETECTING = 0, ESP_MN_STATE_DETECTED = 1, ESP_MN_STATE_TIMEOUT = 2 } esp_mn_state_t;
typedef enum { ESP_MN_LOAD_FROM_PSRAM = 0, ESP_MN_LOAD_FROM_PSRAM_FLASH = 1, ESP_MN_LOAD_FROM_FLASH = 2 } esp_mn_loader_mode_t;
typedef enum { ESP_MN_GREEDY_SEARCH = 0, ESP_MN_BEAM_SEARCH = 1, ESP_MN_BEAM_SEARCH_WITH_FST = 2 } esp_mn_search_method_t;
typedef enum { CHINESE_ID = 1, ENGLISH_ID = 2 } language_id_t;

typedef struct {
    esp_mn_state_t state;
    int num;                                            // <=5
    int command_id[ESP_MN_RESULT_MAX_NUM];
    int phrase_id[ESP_MN_RESULT_MAX_NUM];
    float prob[ESP_MN_RESULT_MAX_NUM];
    char string[256];                                   // 带命令图
    char raw_string[256];
} esp_mn_results_t;

typedef struct {
    char *string; char *phonemes; int16_t command_id; float threshold; int16_t *wave;
} esp_mn_phrase_t;

typedef struct {
    int16_t num; esp_mn_phrase_t **phrases;
} esp_mn_error_t;

typedef struct {
    esp_mn_iface_op_create_t create;            // create(model_name, duration_ms)
    esp_mn_iface_op_get_samp_rate_t get_samp_rate;
    esp_mn_iface_op_get_samp_chunksize_t get_samp_chunksize;
    esp_mn_iface_op_get_samp_chunknum_t get_samp_chunknum;
    esp_mn_iface_op_set_det_threshold_t set_det_threshold;    // 阈值 0.0~0.9999
    esp_mn_iface_op_get_language_t get_language;             // ESP_MN_CHINESE / ESP_MN_ENGLISH
    esp_mn_iface_op_detect_t detect;
    esp_mn_iface_op_destroy_t destroy;
    esp_mn_iface_op_get_results_t get_results;
    esp_mn_iface_op_open_log_t open_log;
    esp_mn_iface_op_clean_t clean;
    esp_wn_iface_op_set_speech_commands set_speech_commands;
    esp_mn_iface_op_switch_loader_mode_t switch_loader_mode;       // mn6+
    esp_mn_iface_op_print_active_speech_commands print_active_speech_commands;
    esp_mn_iface_op_check_speech_command check_speech_command;
} esp_mn_iface_t;

esp_mn_iface_t *esp_mn_handle_from_name(char *model_name);
char *esp_mn_language_from_name(char *model_name);
```

### 命令词管理（esp_mn_speech_commands.h）

```c
esp_err_t esp_mn_commands_alloc(const esp_mn_iface_t *multinet, model_iface_data_t *model_data);
esp_err_t esp_mn_commands_free(void);
esp_err_t esp_mn_commands_add(int command_id, const char *string);
esp_err_t esp_mn_commands_phoneme_add(int command_id, const char *string, const char *phonemes);
esp_err_t esp_mn_commands_modify(const char *old_string, const char *new_string);
esp_err_t esp_mn_commands_remove(const char *string);
esp_err_t esp_mn_commands_clear(void);
char *esp_mn_commands_get_string(int command_id);
esp_mn_phrase_t *esp_mn_commands_get_from_index(int index);      // index 从 0
esp_mn_phrase_t *esp_mn_commands_get_from_string(const char *string);
esp_mn_error_t *esp_mn_commands_update();                         // 必须调，NULL=成功
esp_mn_phrase_t *esp_mn_phrase_alloc(int command_id, const char *string);
void esp_mn_phrase_free(esp_mn_phrase_t *phrase);
esp_mn_node_t *esp_mn_node_alloc(esp_mn_phrase_t *phrase);
void esp_mn_node_free(esp_mn_node_t *node);
void esp_mn_commands_print(void);
void esp_mn_active_commands_print(void);
```

### 从 sdkconfig 加载（esp_process_sdkconfig.h）

```c
void check_chip_config(void);
esp_mn_error_t *esp_mn_commands_update_from_sdkconfig(const esp_mn_iface_t *multinet, model_iface_data_t *model_data);
```

## 6. VADNet / WebRTC VAD

### VADNet（神经网络）— esp_vadn_iface.h / esp_vadn_models.h

```c
// 前缀
#define ESP_VADN_PREFIX  (在 esp_vadn_models.h 中定义)

typedef struct {
    esp_vadn_iface_op_create_t create;   // create(model_name, vad_mode, channel_num, min_speech_ms, min_noise_ms)
    esp_vadn_iface_op_get_samp_chunksize_t get_samp_chunksize;
    esp_vadn_iface_op_get_channel_num_t get_channel_num;
    esp_vadn_iface_op_get_samp_rate_t get_samp_rate;
    esp_vadn_iface_op_set_det_threshold_t set_det_threshold;   // 0.5~0.9999
    esp_vadn_iface_op_get_det_threshold_t get_det_threshold;
    esp_vadn_iface_op_get_triggered_channel_t get_triggered_channel;
    esp_vadn_iface_op_detect_t detect;                          // 返回 vad_state_t
    esp_vadn_iface_op_detect_mfcc_t detect_mfcc;
    esp_vadn_iface_op_get_mfcc_data_t get_mfcc_data;
    esp_vadn_iface_op_clean_t clean;
    esp_vadn_iface_op_destroy_t destroy;
} esp_vadn_iface_t;

const esp_vadn_iface_t *esp_vadn_handle_from_name(const char *model_name);
```

### WebRTC VAD — esp_vad.h

```c
#define SAMPLE_RATE_HZ 16000      // 支持 32000/16000/8000
#define VAD_FRAME_LENGTH_MS 30    // 支持 10/20/30

typedef enum { VAD_MODE_0=0, VAD_MODE_1, VAD_MODE_2, VAD_MODE_3, VAD_MODE_4 } vad_mode_t;
typedef enum { VAD_SILENCE = 0, VAD_SPEECH = 1 } vad_state_t;

vad_handle_t vad_create(vad_mode_t vad_mode);
vad_handle_t vad_create_with_param(vad_mode_t vad_mode, int sample_rate, int one_frame_ms, int min_speech_ms, int min_noise_ms);
vad_state_t vad_process(vad_handle_t handle, int16_t *data, int sample_rate_hz, int one_frame_ms);
vad_state_t vad_process_with_trigger(vad_handle_t handle, int16_t *data);
void vad_reset_trigger(vad_handle_t handle);
void vad_destroy(vad_handle_t inst);

// trigger 辅助
vad_trigger_t *vad_trigger_alloc(int min_speech_len, int min_noise_len);
void vad_trigger_free(vad_trigger_t *trigger);
void vad_trigger_reset(vad_trigger_t *trigger);
vad_state_t vad_trigger_detect(vad_trigger_t *trigger, vad_state_t state);
```

## 7. AEC — esp_aec.h / esp_afe_aec.h

```c
typedef enum {
    AEC_MODE_SR_LOW_COST = 0, AEC_MODE_SR_HIGH_PERF = 1,
    AEC_MODE_VOIP_LOW_COST = 3, AEC_MODE_VOIP_HIGH_PERF = 4,
    AEC_MODE_FD_LOW_COST = 5,  AEC_MODE_FD_HIGH_PERF = 6
} aec_mode_t;

typedef struct {
    int mic_num; int ref_num; int out_num; int filter_length;
    int sample_rate; uint32_t caps; aec_mode_t mode; aec_nlp_level_t nlp_level;
} aec_config_t;

typedef struct { void *aec_handle; int frame_size; aec_config_t config; } aec_handle_t;

aec_handle_t *aec_create(int sample_rate, int filter_length, int channel_num, aec_mode_t mode);
aec_handle_t *aec_create_from_config(aec_config_t *config);
void aec_process(const aec_handle_t *handle, int16_t *indata, int16_t *refdata, int16_t *outdata);
void aec_linear_process(const aec_handle_t *handle, int16_t *indata, int16_t *refdata, int16_t *outdata);
int  aec_nlp_process(const aec_handle_t *handle, int16_t *outdata);   // 返回输出样本数
aec_nlp_level_t aec_set_nlp_level(const aec_handle_t *handle, aec_nlp_level_t level);
int  aec_get_chunksize(const aec_handle_t *handle);
char *aec_get_mode_string(aec_mode_t aec_mode);
char *aec_get_nlp_string(aec_nlp_level_t nlp_level);
char *aec_get_config_string(const aec_handle_t *handle);
void aec_destroy(aec_handle_t *handle);
```

NLP 等级（`esp_aec_nlp.h`）：`AEC_NLP_LEVEL_NORMAL` / `AEC_NLP_LEVEL_AGGR`（默认）/ `AEC_NLP_LEVEL_VERYAGGR`。

### AFE AEC 封装（带 input_format）

```c
typedef struct {
    aec_handle_t *handle; aec_mode_t mode; afe_pcm_config_t pcm_config;
    int frame_size; int16_t *data;
} afe_aec_handle_t;

afe_aec_handle_t *afe_aec_create(const char *input_format, int filter_length, afe_type_t type, afe_mode_t mode);
size_t afe_aec_process(afe_aec_handle_t *handle, const int16_t *indata, int16_t *outdata);  // 返回 outdata 字节
int afe_aec_get_chunksize(afe_aec_handle_t *handle);
void afe_aec_destroy(afe_aec_handle_t *handle);
```

> `afe_aec_create` 当前仅支持 1 麦 + 1 参考。

## 8. DOA（声源定位）— esp_doa.h / esp_afe_doa.h

### 独立 SRP-PHAT（左右声道分开）— esp_doa.h

```c
typedef struct doa_handle_t doa_handle_t;

doa_handle_t *esp_doa_create(int fs, float resolution, float d_mics, int input_timedate_samples);
// 推荐：fs=16000, resolution=20, d_mics=0.06, samples=1024
float esp_doa_process(doa_handle_t *doa, int16_t *left, int16_t *right);   // 返回 0~180 度
void esp_doa_destroy(doa_handle_t *doa);
```

### AFE-aware DOA（按 input_format 自动解交错）— esp_afe_doa.h

> 仅 ESP32-S3 / ESP32-S31 / ESP32-P4 提供（ESP32 只有独立 `esp_doa.h`）。

```c
// 公开结构体（非不透明指针），成员可直接读
typedef struct {
    doa_handle_t *doa_handle;   // 内部 SRP-PHAT 实例
    afe_pcm_config_t pcm_config; // 由 input_format 解析得到（total_ch_num/mic_ids/ref_ids/sample_rate）
    int16_t *leftdata;          // 内部抽取出的左声道缓冲
    int16_t *rightdata;         // 内部抽取出的右声道缓冲
    int frame_size;             // = input_timedate_samples（单声道每帧样本数）
} afe_doa_handle_t;

// input_format 同 AFE：M=麦，N=未用，R=参考；需至少 2 个 M
afe_doa_handle_t *afe_doa_create(const char *input_format, int fs,
                                 float resolution, float d_mics, int input_timedate_samples);
float afe_doa_process(afe_doa_handle_t *handle, const int16_t *indata);   // indata 为交错多通道；返回 0~180 度
void afe_doa_destroy(afe_doa_handle_t *handle);
```

## 9. 中文 TTS — esp_tts.h（esp-tts 子树）

```c
typedef struct esp_tts_voice_t esp_tts_voice_t;
typedef struct esp_tts_handle_t esp_tts_handle_t;

esp_tts_voice_t *esp_tts_voice_set_init(const void *template_set, int16_t *voice_data);
esp_tts_handle_t *esp_tts_create(esp_tts_voice_t *voice);
int   esp_tts_parse_chinese(esp_tts_handle_t *tts, char *text);    // 1=成功解析
short *esp_tts_stream_play(esp_tts_handle_t *tts, int *len, int speed);  // 流式合成
void  esp_tts_voice_set_free(esp_tts_voice_t *voice);
void  esp_tts_destroy(esp_tts_handle_t *tts);
```

声音集：`esp_tts_voice_xiaole.h`（xiaole）、`esp_tts_voice_template.h`（模板）。默认输出 16k/16bit/单声道。
