# ESP-GMF API 速查

> 所有签名取自仓库 `gmf_core/include/`、`elements/*/include/`、`packages/*/include/` 真实头文件与 `docs/`。未列出的 API 一律视为不存在。

## GMF-Core: Pool（esp_gmf_pool.h）

```c
esp_gmf_err_t esp_gmf_pool_init(esp_gmf_pool_handle_t *pool);
esp_gmf_err_t esp_gmf_pool_deinit(esp_gmf_pool_handle_t pool);

esp_gmf_err_t esp_gmf_pool_register_element(esp_gmf_pool_handle_t pool, esp_gmf_element_handle_t el, void *set_cfg_cb);
esp_gmf_err_t esp_gmf_pool_register_element_at_head(esp_gmf_pool_handle_t pool, esp_gmf_element_handle_t el, void *set_cfg_cb);  // 头插，可覆盖同名默认实现
esp_gmf_err_t esp_gmf_pool_register_io(esp_gmf_pool_handle_t pool, esp_gmf_io_handle_t io, void *set_cfg_cb);

// 按 in_io_tag + element 名数组 + out_io_tag 建 pipeline
esp_gmf_err_t esp_gmf_pool_new_pipeline(esp_gmf_pool_handle_t pool, const char *in_io_tag,
                                        const char **el_names, int num, const char *out_io_tag,
                                        esp_gmf_pipeline_handle_t *pipe);

// 按 URL 评分自动选 IO tag
esp_gmf_err_t esp_gmf_pool_get_io_tag_by_url(esp_gmf_pool_handle_t pool, const char *url,
                                             esp_gmf_io_dir_t dir, char **io_tag);

// 迭代遍历已注册模板（*it 初始 NULL，直到 NOT_FOUND）
esp_gmf_err_t esp_gmf_pool_iterate_element(esp_gmf_pool_handle_t pool, const void **it, esp_gmf_element_handle_t *el);
```

宏/工具：
```c
ESP_GMF_POOL_SHOW_ITEMS(pool)   // 打印所有已注册 element/IO
```

## GMF-Core: Pipeline（esp_gmf_pipeline.h）

```c
esp_gmf_err_t esp_gmf_pipeline_create(esp_gmf_pool_handle_t pool, esp_gmf_pipeline_handle_t *pipe);
esp_gmf_err_t esp_gmf_pipeline_destroy(esp_gmf_pipeline_handle_t pipe);

esp_gmf_err_t esp_gmf_pipeline_register_el(esp_gmf_pipeline_handle_t pipe, esp_gmf_element_handle_t el, void *set_cfg_cb);
esp_gmf_err_t esp_gmf_pipeline_set_in(esp_gmf_pipeline_handle_t pipe, esp_gmf_io_handle_t in);
esp_gmf_err_t esp_gmf_pipeline_set_out(esp_gmf_pipeline_handle_t pipe, esp_gmf_io_handle_t out);

esp_gmf_err_t esp_gmf_pipeline_bind_task(esp_gmf_pipeline_handle_t pipe, esp_gmf_task_handle_t task);
esp_gmf_err_t esp_gmf_pipeline_loading_jobs(esp_gmf_pipeline_handle_t pipe);

esp_gmf_err_t esp_gmf_pipeline_set_event(esp_gmf_pipeline_handle_t pipe, esp_gmf_event_cb cb, void *ctx);

// 控制（有效状态见 SKILL.md 状态机表）
esp_gmf_err_t esp_gmf_pipeline_run(esp_gmf_pipeline_handle_t pipe);
esp_gmf_err_t esp_gmf_pipeline_stop(esp_gmf_pipeline_handle_t pipe);
esp_gmf_err_t esp_gmf_pipeline_pause(esp_gmf_pipeline_handle_t pipe);
esp_gmf_err_t esp_gmf_pipeline_resume(esp_gmf_pipeline_handle_t pipe);
esp_gmf_err_t esp_gmf_pipeline_reset(esp_gmf_pipeline_handle_t pipe);
esp_gmf_err_t esp_gmf_pipeline_seek(esp_gmf_pipeline_handle_t pipe, uint64_t pos);

esp_gmf_err_t esp_gmf_pipeline_set_in_uri(esp_gmf_pipeline_handle_t pipe, const char *uri);
esp_gmf_err_t esp_gmf_pipeline_set_out_uri(esp_gmf_pipeline_handle_t pipe, const char *uri);
esp_gmf_err_t esp_gmf_pipeline_get_in(esp_gmf_pipeline_handle_t pipe, esp_gmf_io_handle_t *in);
esp_gmf_err_t esp_gmf_pipeline_get_out(esp_gmf_pipeline_handle_t pipe, esp_gmf_io_handle_t *out);
esp_gmf_err_t esp_gmf_pipeline_get_el_by_name(esp_gmf_pipeline_handle_t pipe, const char *name, esp_gmf_element_handle_t *el);

// 应用主动上报格式信息（无上游自动上报时）
esp_gmf_err_t esp_gmf_pipeline_report_info(esp_gmf_pipeline_handle_t pipe, esp_gmf_info_type_t type, void *info, int size);

// 多 pipeline 组合
esp_gmf_err_t esp_gmf_pipeline_reg_event_recipient(esp_gmf_pipeline_handle_t connector, esp_gmf_pipeline_handle_t connectee);
esp_gmf_err_t esp_gmf_pipeline_connect_pipe(esp_gmf_pipeline_handle_t connector, esp_gmf_element_handle_t c_out,
                                            esp_gmf_pipeline_handle_t connectee, esp_gmf_element_handle_t c_in);
esp_gmf_err_t esp_gmf_pipeline_replace_in(esp_gmf_pipeline_handle_t pipe, esp_gmf_io_handle_t new_in);
esp_gmf_err_t esp_gmf_pipeline_replace_out(esp_gmf_pipeline_handle_t pipe, esp_gmf_io_handle_t new_out);
esp_gmf_err_t esp_gmf_pipeline_set_pause_on_start(esp_gmf_pipeline_handle_t pipe, bool pause_on_start);
esp_gmf_err_t esp_gmf_pipeline_set_prev_run_cb(esp_gmf_pipeline_handle_t pipe, esp_gmf_prev_func_t cb, void *ctx);
esp_gmf_err_t esp_gmf_pipeline_set_prev_stop_cb(esp_gmf_pipeline_handle_t pipe, esp_gmf_prev_func_t cb, void *ctx);
```

宏：
```c
ESP_GMF_PIPELINE_GET_IN_INSTANCE(pipe)    // 取头 IO 实例
ESP_GMF_PIPELINE_GET_OUT_INSTANCE(pipe)   // 取尾 IO 实例
```

## GMF-Core: Task（esp_gmf_task.h）

```c
typedef struct {
    struct { uint32_t stack; uint8_t prio; uint8_t core; bool stack_in_ext; } thread;
    const char *name;
} esp_gmf_task_cfg_t;

#define DEFAULT_ESP_GMF_TASK_CONFIG()  { .thread = { .stack=4*1024, .prio=5, .core=0, .stack_in_ext=false }, .name="gmf_task" }

esp_gmf_err_t esp_gmf_task_init(esp_gmf_task_cfg_t *cfg, esp_gmf_task_handle_t *task);
esp_gmf_err_t esp_gmf_task_deinit(esp_gmf_task_handle_t task);
esp_gmf_err_t esp_gmf_task_set_timeout(esp_gmf_task_handle_t task, int timeout_ms);

// 策略函数：FINISH / ABORT 触发点
typedef int (*esp_gmf_task_strategy_func)(esp_gmf_task_handle_t task, gmf_task_strategy_type_t type, void *ctx);
esp_gmf_err_t esp_gmf_task_set_strategy_func(esp_gmf_task_handle_t task, esp_gmf_task_strategy_func fn, void *ctx);
// 返回：GMF_TASK_STRATEGY_ACTION_DEFAULT / _RESET / _STOP
```

## GMF-Core: Element 基类（esp_gmf_element.h / esp_gmf_audio_element.h）

```c
// 端口属性宏
ESP_GMF_ELEMENT_IN_PORT_ATTR_SET(attr, cap, align, buf_align, port_type, data_size);
ESP_GMF_ELEMENT_OUT_PORT_ATTR_SET(attr, cap, align, buf_align, port_type, data_size);
// cap: ESP_GMF_EL_PORT_CAP_SINGLE / ESP_GMF_EL_PORT_CAP_MULTI
// port_type: ESP_GMF_PORT_TYPE_BYTE / ESP_GMF_PORT_TYPE_BLOCK

esp_gmf_err_t esp_gmf_element_get_caps(esp_gmf_element_handle_t handle, esp_gmf_cap_t **caps);
esp_gmf_err_t esp_gmf_element_get_method(esp_gmf_element_handle_t handle, esp_gmf_method_t **method);
esp_gmf_err_t esp_gmf_element_exe_method(esp_gmf_element_handle_t handle, const char *name, uint8_t *buf, int len);

// 派生类初始化
void esp_gmf_audio_el_init(esp_gmf_audio_element_handle_t handle, esp_gmf_element_cfg_t *cfg);

// 端口取用宏
ESP_GMF_ELEMENT_GET(handle)        // 取 element 基类指针
ESP_GMF_ELEMENT_GET_IN_PORT(handle)
ESP_GMF_ELEMENT_GET_OUT_PORT(handle)
```

格式信息上报（依赖型 element 触发下游）：
```c
void esp_gmf_audio_el_set_snd_info(esp_gmf_element_handle_t handle, esp_gmf_info_sound_t *info);
esp_gmf_err_t esp_gmf_element_notify_snd_info(esp_gmf_element_handle_t handle, esp_gmf_info_sound_t *info);
// 宏：写自身 audio info + 上报 REPORT_INFO
#define GMF_AUDIO_UPDATE_SND_INFO(self, rate, bits, ch) ...
```

## GMF-Core: Port 与 Payload（esp_gmf_port.h / esp_gmf_payload.h）

```c
// Port acquire-release（必须成对）
esp_gmf_err_io_t esp_gmf_port_acquire_in(esp_gmf_port_handle_t h, esp_gmf_payload_t **load, uint32_t wanted, int ticks);
esp_gmf_err_io_t esp_gmf_port_release_in(esp_gmf_port_handle_t h, esp_gmf_payload_t *load, int ticks);
esp_gmf_err_io_t esp_gmf_port_acquire_out(esp_gmf_port_handle_t h, esp_gmf_payload_t **load, uint32_t wanted, int ticks);
esp_gmf_err_io_t esp_gmf_port_release_out(esp_gmf_port_handle_t h, esp_gmf_payload_t *load, int ticks);

esp_gmf_err_t esp_gmf_port_enable_payload_share(esp_gmf_port_handle_t h, bool is_shared);

// 错误分支检查宏
ESP_GMF_PORT_ACQUIRE_IN_CHECK(TAG, ret, err, goto_label);
ESP_GMF_PORT_RELEASE_IN_CHECK(TAG, ret, err, goto_label);

// IO 返回码：ESP_GMF_IO_OK(>=0) / ESP_GMF_IO_FAIL(-1) / ESP_GMF_IO_TIMEOUT(-2) / ESP_GMF_IO_ABORT(-3)
// 等待：ESP_GMF_MAX_DELAY（无限等）/ 0（立即返回）
```

Payload 字段：`buf`、`buf_length`、`valid_size`、`is_done`、`pts`、`needs_free`、`meta_flag`（含 `ESP_GMF_META_FLAG_AUD_RECOVERY_PLC`）。

```c
esp_gmf_payload_t *esp_gmf_payload_new(void);
esp_gmf_payload_t *esp_gmf_payload_new_with_len(size_t len);
esp_gmf_err_t      esp_gmf_payload_delete(esp_gmf_payload_t *load);
```

## GMF-Core: Byte Cache（esp_gmf_cache.h）

```c
esp_gmf_err_t esp_gmf_cache_new(uint32_t size, esp_gmf_cache_handle_t *cache);
esp_gmf_err_t esp_gmf_cache_load(esp_gmf_cache_handle_t cache, uint8_t *buf, uint32_t size);
esp_gmf_err_t esp_gmf_cache_acquire(esp_gmf_cache_handle_t cache, uint32_t want, uint8_t *out, int *read);
esp_gmf_err_t esp_gmf_cache_release(esp_gmf_cache_handle_t cache, uint32_t size);
```

## GMF-Core: IO 基类（esp_gmf_io.h）

```c
esp_gmf_err_t esp_gmf_io_set_uri(esp_gmf_io_handle_t io, const char *uri);
esp_gmf_err_t esp_gmf_io_get_uri(esp_gmf_io_handle_t io, const char **uri);
esp_gmf_err_t esp_gmf_io_set_pos(esp_gmf_io_handle_t io, uint64_t pos);
esp_gmf_err_t esp_gmf_io_get_pos(esp_gmf_io_handle_t io, uint64_t *pos);
esp_gmf_err_t esp_gmf_io_set_size(esp_gmf_io_handle_t io, uint64_t size);
esp_gmf_err_t esp_gmf_io_get_size(esp_gmf_io_handle_t io, uint64_t *size);

esp_gmf_err_t esp_gmf_io_done(esp_gmf_io_handle_t io);        // 标记 EOF（异步会挂起 task）
esp_gmf_err_t esp_gmf_io_clear_done(esp_gmf_io_handle_t io);  // 切换数据源前必须调
esp_gmf_err_t esp_gmf_io_abort(esp_gmf_io_handle_t io);
esp_gmf_err_t esp_gmf_io_clear_abort(esp_gmf_io_handle_t io);
esp_gmf_err_t esp_gmf_io_reset(esp_gmf_io_handle_t io);
esp_gmf_err_t esp_gmf_io_reload(esp_gmf_io_handle_t io);      // 复用连接切 URI

int esp_gmf_io_get_score(esp_gmf_io_handle_t io, const char *uri);  // 0/50/100
esp_gmf_err_t esp_gmf_io_enable_speed_monitor(esp_gmf_io_handle_t io, bool enable);
esp_gmf_err_t esp_gmf_io_get_speed_stats(esp_gmf_io_handle_t io, esp_gmf_io_speed_stats_t *stats);
```

### IO 子类（elements/gmf_io）

```c
// file
typedef struct { ... } file_io_cfg_t;
#define FILE_IO_CFG_DEFAULT() ...
esp_gmf_err_t esp_gmf_io_file_init(file_io_cfg_t *cfg, esp_gmf_io_handle_t *io);
// 字段：dir / cache_size(<=512 关闭) / cache_caps(0 默认 MALLOC_CAP_DMA；esp32p4 可 SPIRAM|DMA)

// http
typedef struct { ... } http_io_cfg_t;
#define HTTP_STREAM_CFG_DEFAULT() ...
esp_gmf_err_t esp_gmf_io_http_init(http_io_cfg_t *cfg, esp_gmf_io_handle_t *io);
esp_gmf_err_t esp_gmf_io_http_set_event_callback(esp_gmf_io_handle_t io, void *cb);
esp_gmf_err_t esp_gmf_io_http_reset(esp_gmf_io_handle_t io);
// 默认异步：stack 6K prio10 core0 databus 20K io_block 3K

// embed_flash
typedef struct { ... } embed_flash_io_cfg_t;
#define EMBED_FLASH_CFG_DEFAULT() ...
esp_gmf_err_t esp_gmf_io_embed_flash_init(embed_flash_io_cfg_t *cfg, esp_gmf_io_handle_t *io);
esp_gmf_err_t esp_gmf_io_embed_flash_set_context(esp_gmf_io_handle_t io, embed_item_info_t *ctx, int max_files);
// URL: embed://<group>/<index>_<name>.<ext>

// i2s_pdm
typedef struct { ... } i2s_pdm_io_cfg_t;
#define ESP_GMF_IO_I2S_PDM_CFG_DEFAULT() ...
esp_gmf_err_t esp_gmf_io_i2s_pdm_init(i2s_pdm_io_cfg_t *cfg, esp_gmf_io_handle_t *io);
// 字段：dir / pdm_chan（应用预先创建的 i2s_chan_handle_t）

// codec_dev
typedef struct { ... } codec_dev_io_cfg_t;
#define ESP_GMF_IO_CODEC_DEV_CFG_DEFAULT() ...
esp_gmf_err_t esp_gmf_io_codec_dev_init(codec_dev_io_cfg_t *cfg, esp_gmf_io_handle_t *io);
esp_gmf_err_t esp_gmf_io_codec_dev_set_dev(esp_gmf_io_handle_t io, void *dev);  // 运行时热切换设备
```

IO 方向：`ESP_GMF_IO_DIR_READER` / `ESP_GMF_IO_DIR_WRITER`。
URL 评分：`ESP_GMF_IO_SCORE_NONE(0)` / `STANDARD(50)` / `PERFECT(100)`。

## gmf_audio Elements（elements/gmf_audio/include）

### 编解码
```c
// aud_dec
typedef struct { ... } esp_audio_simple_dec_cfg_t;
#define DEFAULT_ESP_GMF_AUDIO_DEC_CONFIG() ...
esp_gmf_err_t esp_gmf_audio_dec_init(esp_audio_simple_dec_cfg_t *cfg, esp_gmf_element_handle_t *handle);
esp_gmf_err_t esp_gmf_audio_dec_reconfig(esp_gmf_element_handle_t handle, esp_audio_simple_dec_cfg_t *cfg);
esp_gmf_err_t esp_gmf_audio_dec_reconfig_by_sound_info(esp_gmf_element_handle_t handle, esp_gmf_info_sound_t *info);
// dec_type: ESP_AUDIO_SIMPLE_DEC_TYPE_MP3 / AAC / FLAC / ... ; use_frame_dec 跳过解析层

// aud_enc
typedef struct { ... } esp_audio_enc_config_t;
esp_gmf_err_t esp_gmf_audio_enc_init(esp_audio_enc_config_t *cfg, esp_gmf_element_handle_t *handle);
esp_gmf_err_t esp_gmf_audio_enc_reconfig(esp_gmf_element_handle_t handle, esp_audio_enc_config_t *cfg);
esp_gmf_err_t esp_gmf_audio_enc_reconfig_by_sound_info(esp_gmf_element_handle_t handle, esp_gmf_info_sound_t *info);
esp_gmf_err_t esp_gmf_audio_enc_set_bitrate(esp_gmf_element_handle_t handle, uint32_t bitrate);
esp_gmf_err_t esp_gmf_audio_enc_get_bitrate(esp_gmf_element_handle_t handle, uint32_t *bitrate);
```

### 格式转换
```c
// rate_cvt
#define DEFAULT_ESP_GMF_RATE_CVT_CONFIG() ...   // 默认 44100→48000 stereo 16bit；complexity 0-5；perf_type SPEED/QUALITY
esp_gmf_err_t esp_gmf_rate_cvt_set_dest_rate(esp_gmf_element_handle_t handle, uint32_t rate);

// asrc（硬件/软件）
typedef struct { ... } esp_asrc_cfg_t;   // perf_type: AUTO/HW_ONLY/SW_MEMORY/SW_SPEED；complexity 0-3；timeout_ms
#define DEFAULT_ESP_GMF_ASRC_CONFIG() ...
esp_gmf_err_t esp_gmf_asrc_init(esp_asrc_cfg_t *cfg, esp_gmf_element_handle_t *handle);
esp_gmf_err_t esp_gmf_asrc_set_dest_rate(esp_gmf_element_handle_t handle, uint32_t rate);

// ch_cvt / bit_cvt
esp_gmf_err_t esp_gmf_ch_cvt_set_dest_ch(esp_gmf_element_handle_t handle, uint16_t ch);
esp_gmf_err_t esp_gmf_bit_cvt_set_dest_bits(esp_gmf_element_handle_t handle, uint8_t bits);
```

### 音效
```c
// EQ
typedef struct { esp_ae_eq_filter_type_t filter_type; int fc; float q; float gain; } esp_ae_eq_filter_para_t;
// filter_type: ESP_AE_EQ_FILTER_PEAK / LOW_SHELF / HIGH_SHELF / LOW_PASS / HIGH_PASS
esp_gmf_err_t esp_gmf_eq_set_para(esp_gmf_element_handle_t handle, uint8_t idx, esp_ae_eq_filter_para_t *para);
esp_gmf_err_t esp_gmf_eq_enable_filter(esp_gmf_element_handle_t handle, uint8_t idx, bool enable);
esp_gmf_err_t esp_gmf_eq_get_para(esp_gmf_element_handle_t handle, uint8_t idx, esp_ae_eq_filter_para_t *para);
// filter_num=0 时用内置 10 段（31Hz–16kHz）

// ALC（每通道增益 [-64,63] dB，<-64 静音）
esp_gmf_err_t esp_gmf_alc_set_gain(esp_gmf_element_handle_t handle, uint8_t ch_idx, int8_t gain_db);

// Fade
esp_gmf_err_t esp_gmf_fade_set_mode(esp_gmf_element_handle_t handle, esp_gmf_fade_mode_t mode); // FADE_IN/FADE_OUT
esp_gmf_err_t esp_gmf_fade_get_mode(esp_gmf_element_handle_t handle, esp_gmf_fade_mode_t *mode);
esp_gmf_err_t esp_gmf_fade_reset(esp_gmf_element_handle_t handle);  // 重置进度（切轨复用）

// Sonic（speed/pitch ∈ [0.5,2.0]，1.0 不变）
esp_gmf_err_t esp_gmf_sonic_set_speed(esp_gmf_element_handle_t handle, float speed);
esp_gmf_err_t esp_gmf_sonic_set_pitch(esp_gmf_element_handle_t handle, float pitch);

// Mixer（src_num 个输入；第一轨阻塞 0，其余最大延迟）
esp_gmf_err_t esp_gmf_mixer_set_mode(esp_gmf_element_handle_t handle, uint8_t track, esp_gmf_mixer_mode_t mode); // FADE_BY_SAMPLES/MUTE/NONE
esp_gmf_err_t esp_gmf_mixer_set_audio_info(esp_gmf_element_handle_t handle, esp_gmf_info_sound_t *info);

// DRC / MBC
esp_gmf_err_t esp_gmf_drc_set_points(esp_gmf_element_handle_t handle, esp_ae_drc_point_t *points, int num);
```

### 容器封装
```c
typedef struct {
    esp_muxer_type_t          muxer_type;   // ESP_MUXER_TYPE_TS/MP4/FLV/WAV/CAF/OGG/AVI
    esp_muxer_audio_codec_t   codec;        // 如 ESP_MUXER_AUDIO_CODEC_AAC
    esp_gmf_audio_muxer_out_t output_type;  // ESP_GMF_AUDIO_MUXER_OUTPUT_STREAMING / _FILE
    uint32_t                  slice_duration; // 仅 FILE，ms，默认 60000
    void                     *url_pattern;
    void                     *url_ctx;
} esp_gmf_audio_muxer_cfg_t;
esp_gmf_err_t esp_gmf_audio_muxer_init(esp_gmf_audio_muxer_cfg_t *cfg, esp_gmf_element_handle_t *handle);
```

### 运行时方法
```c
// esp_gmf_audio_methods_def.h 提供 AMETHOD(MODULE, METHOD) 与各方法名宏
esp_gmf_err_t esp_gmf_element_exe_method(esp_gmf_element_handle_t handle, const char *name, uint8_t *buf, int len);
```

### helper
```c
// esp_gmf_audio_helper.h
esp_gmf_err_t esp_gmf_audio_helper_get_audio_type_by_uri(const char *uri, uint32_t *format_id);  // 推断 FourCC
```

## gmf_loader（packages/gmf_loader）

```c
// 在 pool 里批量注册（按 menuconfig 选择）
void gmf_loader_setup_io_default(esp_gmf_pool_handle_t pool);
void gmf_loader_setup_audio_codec_default(esp_gmf_pool_handle_t pool);    // aud_dec/aud_enc/aud_muxer
void gmf_loader_setup_audio_effects_default(esp_gmf_pool_handle_t pool);  // rate/ch/bit_cvt/eq/alc/...
void gmf_loader_setup_video_codec_default(esp_gmf_pool_handle_t pool);
void gmf_loader_setup_ai_audio_default(esp_gmf_pool_handle_t pool);
// 配对的 teardown（逆序销毁）
void gmf_loader_teardown_io_default(esp_gmf_pool_handle_t pool);
void gmf_loader_teardown_audio_codec_default(esp_gmf_pool_handle_t pool);
void gmf_loader_teardown_audio_effects_default(esp_gmf_pool_handle_t pool);
```

## esp_audio_simple_player（packages/esp_audio_simple_player/include/esp_audio_simple_player.h）

见 `recipes/simple_player.md`。核心：`esp_audio_simple_player_new / set_event / run / run_to_end / pause / resume / stop / get_state / destroy`，状态 `esp_asp_state_t`，事件 `esp_asp_event_type_t`。

## esp_player（packages/esp_player/include/esp_player.h）

见 `recipes/esp_player.md`。核心：`esp_player_init / set_data_src / set_url / set_av_mask / set_sync_mode / set_event_cb / set_event_queue / run / run_to_end / pause / resume / stop / seek / set_speed / get_duration / get_play_time / get_track_num / get_track_info / enable_track / deinit`。掩码 `ESP_PLAYER_MASK_AUDIO/VIDEO/AV`，同步 `ESP_PLAYER_SYNC_MODE_AUDIO/VIDEO/SYSTEM/NONE`。

## esp_capture（packages/esp_capture/include/esp_capture.h / _sink.h / _advance.h / _types.h）

```c
typedef struct {
    esp_capture_sync_mode_t     sync_mode;      // NONE / SYSTEM / AUDIO
    esp_capture_audio_src_if_t *audio_src;      // 来自 esp_capture_new_audio_dev_src / _audio_aec_src
    esp_capture_video_src_if_t *video_src;      // 来自 esp_capture_new_video_v4l2_src / _video_dvp_src
    bool                        share_overlay;  // true: overlay 先合成再分给所有 sink
} esp_capture_cfg_t;

esp_capture_err_t esp_capture_open(esp_capture_cfg_t *cfg, esp_capture_handle_t *capture);
esp_capture_err_t esp_capture_set_event_cb(esp_capture_handle_t capture, esp_capture_event_cb_t cb, void *ctx);
esp_capture_err_t esp_capture_start(esp_capture_handle_t capture);
esp_capture_err_t esp_capture_stop(esp_capture_handle_t capture);
esp_capture_err_t esp_capture_close(esp_capture_handle_t capture);
esp_capture_err_t esp_capture_set_thread_scheduler(esp_capture_thread_scheduler_cb_t cb);
void             esp_capture_enable_perf_monitor(bool enable);
// 事件：STARTED/STOPPED/ERROR/AUDIO_PIPELINE_BUILT/VIDEO_PIPELINE_BUILT

// sink 级（esp_capture_sink.h）
typedef struct { esp_capture_audio_info_t audio_info; esp_capture_video_info_t video_info; } esp_capture_sink_cfg_t;
esp_capture_err_t esp_capture_sink_setup(esp_capture_handle_t capture, uint8_t sink_idx,
                                         esp_capture_sink_cfg_t *sink_info, esp_capture_sink_handle_t *sink_handle);
esp_capture_err_t esp_capture_sink_add_muxer(esp_capture_sink_handle_t sink, esp_capture_muxer_cfg_t *muxer_cfg);
esp_capture_err_t esp_capture_sink_add_overlay(esp_capture_sink_handle_t sink, esp_capture_overlay_if_t *overlay);
esp_capture_err_t esp_capture_sink_enable_muxer(esp_capture_sink_handle_t sink, bool enable);
esp_capture_err_t esp_capture_sink_enable_overlay(esp_capture_sink_handle_t sink, bool enable);
esp_capture_err_t esp_capture_sink_enable(esp_capture_sink_handle_t sink, esp_capture_run_mode_t run_type);  // DISABLE / ALWAYS / ONESHOT
esp_capture_err_t esp_capture_sink_disable_stream(esp_capture_sink_handle_t sink, esp_capture_stream_type_t stream_type);
esp_capture_err_t esp_capture_sink_set_bitrate(esp_capture_sink_handle_t h, esp_capture_stream_type_t stream_type, uint32_t bitrate);
esp_capture_err_t esp_capture_sink_acquire_frame(esp_capture_sink_handle_t sink, esp_capture_stream_frame_t *frame, bool no_wait);
esp_capture_err_t esp_capture_sink_release_frame(esp_capture_sink_handle_t sink, esp_capture_stream_frame_t *frame);

// 高级（esp_capture_advance.h）：自定义 element + 手工 build pipeline
esp_capture_err_t esp_capture_register_element(esp_capture_handle_t capture, esp_capture_stream_type_t stream_type, esp_gmf_element_handle_t element);
esp_capture_err_t esp_capture_sink_build_pipeline(esp_capture_sink_handle_t sink, esp_capture_stream_type_t stream_type, const char **element_tags, uint8_t element_num);
esp_capture_err_t esp_capture_sink_get_element_by_tag(esp_capture_sink_handle_t sink, esp_capture_stream_type_t stream_type, const char *tag, esp_gmf_element_handle_t *element);
esp_capture_err_t esp_capture_advance_open(esp_capture_advance_cfg_t *cfg, esp_capture_handle_t *capture);

// muxer 包装（_sink.h）
typedef struct { esp_muxer_config_t *base_config; uint32_t cfg_size; esp_capture_muxer_mask_t muxer_mask; } esp_capture_muxer_cfg_t;
// run_mode: DISABLE/ALWAYS/ONESHOT ; muxer_mask: ALL(0)/AUDIO(1)/VIDEO(2)
// format_id: PCM/G711A/G711u/OPUS/AAC/H264/MJPEG/RGB565/RGB565_BE/RGB888/BGR888/YUV420/YUV422P/YUV422/O_UYY_E_VYY/ANY(0xFFFF)
```

source 工厂（`esp_capture_defaults.h`）：`esp_capture_new_audio_dev_src` / `esp_capture_new_audio_aec_src` / `esp_capture_new_video_v4l2_src` / `esp_capture_new_video_dvp_src` / `esp_capture_new_text_overlay`。

## esp_bt_audio（packages/esp_bt_audio/include/esp_bt_audio*.h）

```c
typedef struct {
    void                       *host_config;     // NimBLE/Bluedroid cfg 或 NULL
    esp_bt_audio_event_cb_t     event_cb;
    void                       *event_user_ctx;
    esp_bt_audio_classic_cfg_t  classic;          // .roles 位或 A2DP_SRC/SNK|HFP_HF/AG|AVRC_CT/TG|PBAP_PCE
    esp_bt_audio_le_cfg_t       le;               // .roles .user_case .snk_cnt .src_cnt .pacs .csip .vcp .bsrc
} esp_bt_audio_config_t;

esp_err_t esp_bt_audio_init(esp_bt_audio_config_t *bt_config);
void     esp_bt_audio_deinit(void);

// 经典（_classic.h，需 CONFIG_BT_BLUEDROID_ENABLED）
esp_err_t esp_bt_audio_classic_discovery_start(void);
esp_err_t esp_bt_audio_classic_discovery_stop(void);
esp_err_t esp_bt_audio_classic_connect(uint32_t role, uint8_t *bt_dev_addr);
esp_err_t esp_bt_audio_classic_disconnect(uint32_t role, uint8_t *bt_dev_addr);
esp_err_t esp_bt_audio_classic_set_scan_mode(bool connectable, bool discoverable);

// LE（_le.h，需 CONFIG_BT_NIMBLE_ENABLED + CONFIG_BT_AUDIO + CONFIG_BT_ISO）
esp_err_t esp_bt_audio_le_scan_start(uint32_t timeout_ms);
esp_err_t esp_bt_audio_le_scan_start_ext(const uint8_t *target, uint32_t timeout_ms);
esp_err_t esp_bt_audio_le_scan_stop(void);
esp_err_t esp_bt_audio_le_connect(uint8_t addr_type, const uint8_t *addr, uint32_t timeout_ms);
esp_err_t esp_bt_audio_le_disconnect(void);
esp_err_t esp_bt_audio_le_disconnect_peer(const uint8_t *addr);
esp_err_t esp_bt_audio_le_broadcast_sync(const uint8_t *broadcast_name, const uint8_t *broadcast_code, uint32_t bit_field, uint32_t timeout_ms);
esp_err_t esp_bt_audio_le_pa_sync_terminate(void);

// host（_host.h）
esp_err_t esp_bt_audio_host_init(void *cfg);   // NimBLE/Bluedroid cfg
void     esp_bt_audio_host_deinit(void);
#define ESP_BT_AUDIO_HOST_NIMBLE_CFG_DEFAULT()    { .dev_name = "esp_ble" }
#define ESP_BT_AUDIO_HOST_BLUEDROID_CFG_DEFAULT() { .dev_name="esp_classic", .bluedroid_cfg=BT_BLUEDROID_INIT_CONFIG_DEFAULT(), .pin_type=ESP_BT_PIN_TYPE_FIXED, .pin_code={'1','2','3','4'}, .sp_param=ESP_BT_SP_IOCAP_MODE, .iocap=ESP_BT_IO_CAP_NONE }

// 流（_stream.h）：直读直写或经 esp_gmf_io_bt 接 pipeline
esp_err_t esp_bt_audio_stream_get_codec_info(esp_bt_audio_stream_handle_t h, esp_bt_audio_stream_codec_info_t *ci);
esp_err_t esp_bt_audio_stream_get_dir(esp_bt_audio_stream_handle_t h, esp_bt_audio_stream_dir_t *dir);     // SINK / SOURCE
esp_err_t esp_bt_audio_stream_get_profile(esp_bt_audio_stream_handle_t h, esp_bt_audio_stream_profile_t *p); // CLASSIC_A2DP/HFP / LE_UNICAST/BROADCAST
esp_err_t esp_bt_audio_stream_get_context(esp_bt_audio_stream_handle_t h, uint32_t *ctx);                   // UNSPECIFIED/CONVERSATIONAL/MEDIA
esp_err_t esp_bt_audio_stream_acquire_read(esp_bt_audio_stream_handle_t h, esp_bt_audio_stream_packet_t *pkt, uint32_t wait_ms);
esp_err_t esp_bt_audio_stream_release_read(esp_bt_audio_stream_handle_t h, esp_bt_audio_stream_packet_t *pkt);
esp_err_t esp_bt_audio_stream_acquire_write(esp_bt_audio_stream_handle_t h, esp_bt_audio_stream_packet_t *pkt, uint32_t wanted_size);
esp_err_t esp_bt_audio_stream_release_write(esp_bt_audio_stream_handle_t h, esp_bt_audio_stream_packet_t *pkt, uint32_t wait_ms);
esp_err_t esp_bt_audio_stream_get_iso_interval(esp_bt_audio_stream_handle_t h, uint16_t *iso_interval);

// 播放控制（_playback.h）：作为 AVRCP CT 向远端发指令
esp_err_t esp_bt_audio_playback_play/pause/stop/next/prev(void);
esp_err_t esp_bt_audio_playback_request_metadata(uint32_t mask);      // TITLE/ARTIST/ALBUM/.../COVER_ART
esp_err_t esp_bt_audio_playback_reg_notifications(uint32_t mask);     // PLAY_STATUS_CHANGE/TRACK_CHANGE/...

// 电话（_tel.h）/ 音量（_vol.h）/ 媒体推流（_media.h）
esp_err_t esp_bt_audio_call_answer/reject(uint8_t idx);
esp_err_t esp_bt_audio_call_dial(const char *number);
esp_err_t esp_bt_audio_vol_set_absolute(uint32_t vol);
esp_err_t esp_bt_audio_vol_set_relative(bool up_down);
esp_err_t esp_bt_audio_vol_notify(uint32_t vol);
esp_err_t esp_bt_audio_media_start(uint32_t role, void *config);   // role = A2DP_SRC
esp_err_t esp_bt_audio_media_stop(uint32_t role);

// 事件（_event.h）：CONNECTION_STATE_CHG/DISCOVERY_STATE_CHG/DEVICE_DISCOVERED/STREAM_STATE_CHG/MEDIA_CTRL_CMD/
//                  PLAYBACK_STATUS_CHG/PLAYBACK_METADATA/VOL_ABSOLUTE/VOL_RELATIVE/TEL_STATUS_CHG/CALL_STATE_CHG/
//                  PHONEBOOK_COUNT/ENTRY/HISTORY/BIG_SYNC_LOST/PA_SYNC_LOST
```

GMF 集成：`esp_gmf_io_bt_set_stream(io_handle, stream_handle)` 把 stream 挂到 pipeline 的 in/out IO。

## esp_audio_render（packages/esp_audio_render/include/esp_audio_render.h / _types.h）

```c
typedef struct {
    uint8_t                          max_stream_num;    // 同时混音的最大流数
    esp_audio_render_write_cb_t      out_writer;        // int(*)(uint8_t*, uint32_t, void*)
    void                            *out_ctx;
    esp_audio_render_sample_info_t   out_sample_info;   // sample_rate / bits_per_sample / channel
    void                            *pool;              // GMF pool（用于查处理 element）
    uint16_t                         process_period;    // ms（默认 20）
    uint8_t                          process_buf_align; // 默认 16
} esp_audio_render_cfg_t;

esp_audio_render_err_t esp_audio_render_create(esp_audio_render_cfg_t *cfg, esp_audio_render_handle_t *render);
esp_audio_render_err_t esp_audio_render_task_reconfigure(esp_audio_render_handle_t render, esp_gmf_task_config_t *cfg);  // 必须 open 前
esp_audio_render_err_t esp_audio_render_set_event_cb(esp_audio_render_handle_t render, esp_audio_render_event_cb_t cb, void *ctx);
esp_audio_render_err_t esp_audio_render_set_out_sample_info(esp_audio_render_handle_t render, esp_audio_render_sample_info_t *info);  // open 前
esp_audio_render_err_t esp_audio_render_add_mixed_proc(esp_audio_render_handle_t render, esp_audio_render_proc_type_t proc_type[], uint8_t proc_num);  // open 前
esp_audio_render_err_t esp_audio_render_get_mixed_element(esp_audio_render_handle_t render, esp_audio_render_proc_type_t proc_type, esp_gmf_element_handle_t *element);
esp_audio_render_err_t esp_audio_render_set_solo_stream(esp_audio_render_handle_t render, esp_audio_render_stream_id_t stream_id);  // ALL_STREAM 取消 solo
esp_audio_render_err_t esp_audio_render_stream_get(esp_audio_render_handle_t render, esp_audio_render_stream_id_t stream_id, esp_audio_render_stream_handle_t *stream);
esp_audio_render_err_t esp_audio_render_stream_set_mixer_gain(esp_audio_render_stream_handle_t stream, esp_audio_render_mixer_gain_t *mixer_gain);  // open 前
esp_audio_render_err_t esp_audio_render_stream_open(esp_audio_render_stream_handle_t stream, esp_audio_render_sample_info_t *sample_info);
esp_audio_render_err_t esp_audio_render_stream_add_proc(esp_audio_render_stream_handle_t stream, esp_audio_render_proc_type_t proc_type[], uint8_t proc_num);  // open 前
esp_audio_render_err_t esp_audio_render_stream_get_element(esp_audio_render_stream_handle_t stream, esp_audio_render_proc_type_t proc_type, esp_gmf_element_handle_t *element);
esp_audio_render_err_t esp_audio_render_stream_write(esp_audio_render_stream_handle_t stream, uint8_t *pcm_data, uint32_t pcm_size);
esp_audio_render_err_t esp_audio_render_stream_set_fade(esp_audio_render_stream_handle_t stream, bool fade_in);
esp_audio_render_err_t esp_audio_render_stream_pause(esp_audio_render_stream_handle_t stream, bool pause);
esp_audio_render_err_t esp_audio_render_stream_flush(esp_audio_render_stream_handle_t stream);
esp_audio_render_err_t esp_audio_render_stream_set_speed(esp_audio_render_stream_handle_t stream, float speed);  // 需先 add SONIC
esp_audio_render_err_t esp_audio_render_stream_get_latency(esp_audio_render_stream_handle_t stream, uint32_t *latency_ms);
esp_audio_render_err_t esp_audio_render_stream_close(esp_audio_render_stream_handle_t stream);
esp_audio_render_err_t esp_audio_render_destroy(esp_audio_render_handle_t render);
// proc_type: ALC / SONIC / EQ / FADE / ENC（仅 mixed）
// stream_id: FIRST(0) / SECOND(1) / ... / ALL(0xFF)
// event: OPENED / CLOSED
```

## esp_video_render（packages/esp_video_render/include/esp_video_render.h / _types.h / _dual_stream.h）

```c
typedef struct { void *pool; uint8_t fps; } esp_video_render_cfg_t;

esp_video_render_err_t esp_video_render_create(esp_video_render_cfg_t *cfg, esp_video_render_handle_t *render);
esp_video_render_err_t esp_video_render_task_reconfigure(esp_video_render_handle_t render, esp_video_render_task_cfg_t *task_cfg);  // open 前
esp_video_render_err_t esp_video_render_set_event_cb(esp_video_render_handle_t render, esp_video_render_event_cb_t event_cb, void *ctx);
esp_video_render_err_t esp_video_render_set_display(esp_video_render_handle_t render, esp_video_render_backend_cfg_t *backend);  // 无 stream open 时
esp_video_render_err_t esp_video_render_get_display_info(esp_video_render_handle_t render, esp_video_render_disp_info_t *display);
esp_video_render_err_t esp_video_render_set_bg_image(esp_video_render_handle_t render, esp_video_render_img_t *img);
esp_video_render_err_t esp_video_render_set_bg_color(esp_video_render_handle_t render, esp_video_render_clr_t *color);
esp_video_render_err_t esp_video_render_set_compose_mode(esp_video_render_handle_t render, esp_video_render_compose_mode_t mode);  // AUTO / MANUAL，open 前
esp_video_render_err_t esp_video_render_stream_open(esp_video_render_handle_t render, esp_video_render_stream_info_t *stream_info, esp_video_render_stream_handle_t *stream);
esp_video_render_err_t esp_video_render_stream_render_async(esp_video_render_stream_handle_t stream);
esp_video_render_err_t esp_video_render_stream_get_overlay(esp_video_render_stream_handle_t stream, esp_vui_overlay_handle_t *overlay);
esp_video_render_err_t esp_video_render_stream_set_src_rect(esp_video_render_stream_handle_t stream, esp_video_render_rect_t *src_rect);
esp_video_render_err_t esp_video_render_stream_set_disp_rect(esp_video_render_stream_handle_t stream, esp_video_render_rect_t *disp_rect);
esp_video_render_err_t esp_video_render_stream_set_zorder(esp_video_render_stream_handle_t stream, uint8_t zorder);
esp_video_render_err_t esp_video_render_stream_set_rotate(esp_video_render_stream_handle_t stream, int16_t degree);   // 0/90/180/270
esp_video_render_err_t esp_video_render_stream_set_visible(esp_video_render_stream_handle_t stream, bool visible);
esp_video_render_err_t esp_video_render_stream_set_alpha(esp_video_render_stream_handle_t stream, uint8_t alpha);     // 0-255
esp_video_render_err_t esp_video_render_stream_acquire_fb(esp_video_render_stream_handle_t stream, esp_video_render_fb_t *fb);
esp_video_render_err_t esp_video_render_stream_write_fb(esp_video_render_stream_handle_t stream, esp_video_render_fb_t *fb);
esp_video_render_err_t esp_video_render_stream_release_fb(esp_video_render_stream_handle_t stream, esp_video_render_fb_t *fb);
esp_video_render_err_t esp_video_render_stream_lock/unlock(esp_video_render_stream_handle_t stream);
esp_video_render_err_t esp_video_render_stream_write(esp_video_render_stream_handle_t stream, esp_video_render_frame_t *frame);
esp_video_render_err_t esp_video_render_stream_compose_lock/compose_unlock(esp_video_render_stream_handle_t stream);
esp_video_render_err_t esp_video_render_compose(esp_video_render_handle_t render);   // 仅 MANUAL 模式
esp_video_render_err_t esp_video_render_stream_close(esp_video_render_stream_handle_t stream);
esp_video_render_err_t esp_video_render_get_blender(esp_video_render_handle_t render, esp_video_render_blend_handle_t *blender);
esp_video_render_err_t esp_video_render_get_pool(esp_video_render_handle_t render, void **pool);
esp_video_render_err_t esp_video_render_destroy(esp_video_render_handle_t render);
void                  esp_video_render_measure_enable(bool enable);
// format: H264/MJPEG/RGB565/RGB565_BE/RGB888/BGR888/YUV420P/YUV422P/UYVY/YUV422/O_UYY_E_VYY
// event: NONE/OPENED/CLOSED/VSYNC
// 后端 ops: esp_video_render_get_lcd_backend() / esp_video_render_get_lvgl_backend()

// 双流（_dual_stream.h）
typedef struct {
    esp_video_render_handle_t    render[2];
    esp_video_render_task_cfg_t  task_cfg[2];
    uint32_t                     frame_count;
    uint32_t                     max_frame_size;   // 默认 20 KB
    uint8_t                      fps;              // 0 = 不限速
    bool                         render_async;
} esp_video_render_dual_stream_cfg_t;
esp_video_render_err_t esp_video_render_dual_stream_open(...);
esp_video_render_err_t esp_video_render_dual_stream_set_display_rect(handle, uint8_t stream_idx, rect);
esp_video_render_err_t esp_video_render_dual_stream_get_buffer(handle, uint8_t stream_idx, frame);
esp_video_render_err_t esp_video_render_dual_stream_send_buffer(handle, frame_a, frame_b);
esp_video_render_err_t esp_video_render_dual_stream_release_buffer(handle, uint8_t stream_idx, frame);
esp_video_render_err_t esp_video_render_dual_stream_close(handle);
```

UI 叠加（`vui/`）：`esp_vui_container_create(overlay, frame_info, pos, opaque, &container)` → `esp_vui_container_add_widget(container, widget)`；widget 实现 `redraw`/`destroy` 回调，由框架按 dirty region 调用。

## gmf_ai_audio（elements/gmf_ai_audio/include/esp_gmf_afe.h / _aec.h / _wn.h / _ns.h / _vad.h / _doa.h / _afe_manager.h）

```c
// ai_afe 全功能 element（_afe.h）
typedef struct {
    esp_gmf_afe_manager_handle_t afe_manager;
    uint32_t                     delay_samples;   // 默认 2048（约 64 ms），应 > vad_min_speech_ms
    void                        *models;
    uint32_t                     wakeup_time;     // 默认 10000 ms
    uint32_t                     wakeup_end;      // 默认 2000 ms
    bool                         vcmd_detect_en;
    uint32_t                     vcmd_timeout;    // 默认 5760 ms
    const char                  *mn_language;     // "cn" / "en"
    esp_gmf_afe_event_cb_t       event_cb;
    void                        *event_ctx;
} esp_gmf_afe_cfg_t;
#define DEFAULT_GMF_AFE_CFG(__afe_manager, __event_cb, __event_ctx, __models) {...}
esp_gmf_err_t esp_gmf_afe_init(void *config, esp_gmf_obj_handle_t *handle);
esp_gmf_err_t esp_gmf_afe_vcmd_detection_begin(esp_gmf_element_handle_t handle);
esp_gmf_err_t esp_gmf_afe_vcmd_detection_cancel(esp_gmf_element_handle_t handle);
esp_gmf_err_t esp_gmf_afe_set_event_cb(esp_gmf_element_handle_t handle, esp_gmf_afe_event_cb_t cb, void *ctx);
esp_gmf_err_t esp_gmf_afe_keep_awake(esp_gmf_element_handle_t handle, bool enable);
esp_gmf_err_t esp_gmf_afe_trigger_wakeup(esp_gmf_element_handle_t handle);
esp_gmf_err_t esp_gmf_afe_trigger_sleep(esp_gmf_element_handle_t handle);
// 事件枚举：WAKEUP_START=-100 / WAKEUP_END=-99 / VAD_START=-98 / VAD_END=-97 / VCMD_DECT_TIMEOUT=-96 / VCMD_DECTECTED=phrase_id(>=0)

// afe_manager（_afe_manager.h，可独立使用）
#define DEFAULT_GMF_AFE_MANAGER_CFG(_afe_cfg, _read_cb, _read_ctx, _result_cb, _result_ctx) {...}
esp_gmf_err_t esp_gmf_afe_manager_create(cfg, &handle);
esp_gmf_err_t esp_gmf_afe_manager_destroy(handle);
esp_gmf_err_t esp_gmf_afe_manager_suspend(handle, bool suspend);
esp_gmf_err_t esp_gmf_afe_manager_enable_features(handle, esp_gmf_afe_feature_t feature, bool enable);  // WAKENET/VAD/NS/AEC/SE
esp_gmf_err_t esp_gmf_afe_manager_get_chunk_size(handle, size_t *size);
esp_gmf_err_t esp_gmf_afe_manager_get_input_ch_num(handle, uint8_t *ch_num);

// 单能力 element
typedef struct { uint8_t filter_len; afe_type_t type; afe_mode_t mode; char *input_format; } esp_gmf_aec_cfg_t;   // _aec.h
esp_gmf_err_t esp_gmf_aec_init(esp_gmf_aec_cfg_t *cfg, esp_gmf_obj_handle_t *out_handle);

typedef struct { void *models; det_mode_t det_mode; char *input_format; esp_gmf_wn_detect_cb_t detect_cb; void *user_ctx; } esp_gmf_wn_cfg_t;  // _wn.h

esp_gmf_ns_cfg_t  ESP_GMF_NS_CFG_DEFAULT();   // _ns.h：sample_rate/channel/frame_ms/model_name/partition_label
esp_gmf_vad_cfg_t ESP_GMF_VAD_CFG_DEFAULT();  // _vad.h：含 result_callback(vad_state_t, ctx)
esp_gmf_doa_cfg_t ESP_GMF_DOA_CFG_DEFAULT();  // _doa.h：sample_rate/resolution/d_mics/frame_ms/input_format/result_callback(float angle, ctx)
// 通道格式串 input_format: M 麦 / R 扬声器参考 / N 不用，如 "MMNR"
```

## 工具宏（esp_gmi_info.h / esp_gmf_obj.h）

```c
OBJ_GET_TAG(obj)            // 取对象 tag 字符串
OBJ_GET_CFG(obj)            // 取绑定的配置指针
esp_gmf_obj_set_tag(h, tag);
esp_gmf_obj_set_config(h, cfg, size);
esp_gmf_obj_get_tag(h, &tag);

const char *esp_gmf_event_get_state_str(esp_gmf_event_state_t state);  // 状态名转字符串
```
