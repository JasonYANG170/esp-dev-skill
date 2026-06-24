# AI 语音前端（gmf_ai_audio + esp-sr）

> **适用摘要**: 用 `gmf_ai_audio` 把 `esp-sr`（AEC/NS/VAD/唤醒词 WakeNet/命令词 MultiNet/DOA）封装成 6 个 GMF element：`ai_afe`（全功能统一入口，含 feed/fetch 双任务、wakeup+VAD 状态机、命令词与手动唤醒）、`ai_aec`/`ai_wn`/`ai_ns`/`ai_vad`/`ai_doa`（单能力 element，可直接接入 GMF pipeline）。配套 `esp_gmf_afe_manager` 可脱离 pipeline 独立使用。

## 触发意图

- "唤醒词检测"
- "WakeNet / MultiNet 命令词"
- "AEC 回声消除"
- "VAD 语音活动检测"
- "DOA 声源定位"
- "ai_afe / ai_aec / ai_wn"
- "全双工语音交互"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 麦克风阵列板：S3-Korvo2 / P4-Function-EV / S31 等；ai_doa 需双麦；AEC 需 R 通道接扬声器参考 |
| 模型分区 | ai_afe / ai_wn 需 `model` 分区并烧入对应 `esp-sr` 模型（参 `wwe/README_CN.md`） |
| 组件 | `espressif/gmf_ai_audio`（依赖 `esp-sr`、`gmf_core`）；menuconfig 按需启用 `CONFIG_SR_NSN_NSNET2`、`CONFIG_GMF_AI_AUDIO_*` |
| 参考头文件 | `elements/gmf_ai_audio/include/esp_gmf_afe.h`、`esp_gmf_afe_manager.h`、`esp_gmf_aec.h`、`esp_gmf_wn.h`、`esp_gmf_ns.h`、`esp_gmf_vad.h`、`esp_gmf_doa.h` |
| 参考示例 | `elements/gmf_ai_audio/examples/wwe`（ai_afe 唤醒+命令词+VAD-to-file）、`aec_rec`（ai_aec 双 pipeline 边播边录） |

## 分步说明

`gmf_ai_audio` 关键约定：

- **通道格式串** `input_format`：`M` 麦克风 / `R` 扬声器参考 / `N` 不用。如 `"MMNR"` = 4 通道，前两麦、第三空、第四参考。ai_afe 固定 16 kHz/16 bit；ai_aec 还支持 8 kHz（`AFE_TYPE_VC_8K`）；ai_doa 必须恰好两个 `M`。
- **两种用法**：完整交互用 `ai_afe`（自动管 feed/fetch 任务、状态机、事件回调）；只要单能力（AEC/NS/VAD/唤醒）就把对应 element 挂进普通 GMF pipeline。

### 1. 选择 AFE 类型并创建 afe_manager（完整交互场景）

`afe_config_init` 的 `type` 决定 AFE 模式：`AFE_TYPE_SR`（语音识别）/ `AFE_TYPE_VC`/`AFE_TYPE_VC_8K`（语音通信）/ `AFE_TYPE_FD`（全双工，适合边播边录的交互）。

```c
#include "esp_gmf_afe_manager.h"
#include "esp_gmf_afe.h"
#include "model_data.h"   // esp-sr 模型（model_data_get()）

// 1) 加载模型 + 构造 afe_config_t
void *models = model_data_get();   // 从 model 分区加载 wakenet/multinet/ns/vad 模型
afe_config_t *afe_cfg = afe_config_init("MMNR", models, AFE_TYPE_FD, AFE_MODE_LOW_COST);
afe_cfg->wakenet_init = true;      // 启唤醒词
afe_cfg->vad_init     = true;      // 启 VAD
afe_cfg->aec_init     = true;      // 启 AEC（边播边录必开）

// 2) afe_manager（feed/fetch 双任务；read_cb 提供多通道 PCM，result_cb 收算法结果）
//    独立使用时 read_cb/result_cb 直接对接；用作 ai_afe element 时由 element 填充
esp_gmf_afe_manager_cfg_t mgr_cfg = DEFAULT_GMF_AFE_MANAGER_CFG(afe_cfg, my_read_cb, &io_ctx,
                                                                 my_result_cb, &result_ctx);
esp_gmf_afe_manager_handle_t mgr = NULL;
esp_gmf_afe_manager_create(&mgr_cfg, &mgr);

// 3) 查询算法每帧样本数与总输入通道数，调整 IO 缓冲
size_t chunk_size = 0;
uint8_t ch_num    = 0;
esp_gmf_afe_manager_get_chunk_size(mgr, &chunk_size);
esp_gmf_afe_manager_get_input_ch_num(mgr, &ch_num);

// 4) 运行时切换特性（无需重建）
esp_gmf_afe_manager_enable_features(mgr, ESP_AFE_FEATURE_AEC, true);
esp_gmf_afe_manager_enable_features(mgr, ESP_AFE_FEATURE_VAD, false);
// ESP_AFE_FEATURE_WAKENET / _NS / _SE 同理

// 5) 低功耗：同时挂起 feed+fetch
esp_gmf_afe_manager_suspend(mgr, true);   // true=挂起；恢复传 false
```

### 2. 把 manager 包成 ai_afe element 并接入 pipeline

`ai_afe` 从 codec_dev IO 读多通道 PCM 喂给 manager，把 fetch 出的 16-bit 单声道 PCM 写到输出 port；唤醒/VAD/命令词事件经 `event_cb` 上报（不放音频 payload）。

```c
// event_cb 见第 4 步
esp_gmf_afe_cfg_t afe_el_cfg = DEFAULT_GMF_AFE_CFG(mgr, my_afe_event_cb, &app_ctx, models);
afe_el_cfg.vcmd_detect_en = true;                  // 启命令词（MultiNet）
afe_el_cfg.mn_language    = "cn";                  // 或 "en"，必须与模型语言一致
// 4 个时序参数（默认值见 esp_gmf_afe.h）
afe_el_cfg.wakeup_time    = 10000;                 // 唤醒后无 VAD 多久才 WAKEUP_END
afe_el_cfg.wakeup_end     = 2000;                  // VAD 结束后静音多久才 WAKEUP_END
afe_el_cfg.vcmd_timeout   = 5760;                  // 命令词检测超时
afe_el_cfg.delay_samples  = 2048;                  // 输出 PCM 延迟，补偿 VAD 检测滞后，
                                                    // 时间换算应 > afe_config_t.vad_min_speech_ms
esp_gmf_obj_handle_t ai_afe = NULL;
esp_gmf_afe_init(&afe_el_cfg, &ai_afe);

// 挂入 GMF pipeline（io_codec_dev(4ch) → ai_afe → 应用自管 outport/落盘）
const char *name[] = {"ai_afe"};
esp_gmf_pool_new_pipeline(pool, "io_codec_dev", name, 1, NULL, &pipe);
esp_gmf_io_codec_dev_set_dev(ESP_GMF_PIPELINE_GET_IN_INSTANCE(pipe), record_handle);
// 应用自定义 outport 接 ai_afe 输出 PCM（示例 wwe 用 NEW_ESP_GMF_PORT_OUT_BYTE 写 SD 卡文件）
esp_gmf_port_handle_t outport = NEW_ESP_GMF_PORT_OUT_BYTE(acq_write, rel_write, NULL, NULL, 2048, 100);
esp_gmf_pipeline_reg_el_port(pipe, "ai_afe", ESP_GMF_IO_DIR_WRITER, outport);

// 上报真实输入格式
esp_gmf_info_sound_t info = { .sample_rates = 16000, .channels = 4, .bits = 16 };
esp_gmf_pipeline_report_info(pipe, ESP_GMF_INFO_SOUND, &info, sizeof(info));
```

### 3. wakeup + VAD 状态机（自动维护三种组合）

`afe_config_init` 启用项决定走哪种组合：

- **仅唤醒**：`wakenet_init=true, vad_init=false` → `IDLE ↔ WAKEUP`（`WAKEUP_START`/`WAKEUP_END`）
- **仅 VAD**：`wakenet_init=false, vad_init=true` → `IDLE ↔ SPEECHING`（`VAD_START`/`VAD_END`）
- **唤醒+VAD**：两者都 true → `IDLE → WAKEUP → SPEECHING → WAIT_FOR_SLEEP → IDLE`，避免唤醒间隙外的频繁 VAD 事件

> 手动唤醒三 API（脱离状态机）：`esp_gmf_afe_trigger_wakeup(el)`（按键/外部触发立即唤醒并广播 `WAKEUP_START`）、`esp_gmf_afe_trigger_sleep(el)`（回到 IDLE）、`esp_gmf_afe_keep_awake(el, true)`（关闭自动睡眠定时器，必须 `trigger_sleep` 才回 IDLE）。

### 4. 事件回调（六类事件，命令词 ID 直接做枚举值）

回调在 fetch_task 上下文执行，只做轻量分发（更新状态、入队消息），耗时逻辑放主任务。

```c
static void my_afe_event_cb(esp_gmf_element_handle_t el, esp_gmf_afe_evt_t *event, void *ctx) {
    switch (event->type) {
    case ESP_GMF_AFE_EVT_WAKEUP_START: {            // -100
        esp_gmf_afe_wakeup_info_t *wi = event->event_data;   // data_volume/wake_word_index/wakenet_model_index
        // 唤醒后启动命令词检测（取消旧轮再 begin）
        esp_gmf_afe_vcmd_detection_cancel(el);
        esp_gmf_afe_vcmd_detection_begin(el);
        break;
    }
    case ESP_GMF_AFE_EVT_WAKEUP_END:                 // -99：唤醒超时或手动 sleep
        esp_gmf_afe_vcmd_detection_cancel(el);
        break;
    case ESP_GMF_AFE_EVT_VAD_START:                  // -98
    case ESP_GMF_AFE_EVT_VAD_END:                    // -97
        break;
    case ESP_GMF_AFE_EVT_VCMD_DECT_TIMEOUT:          // -96：超时后需再次 begin 才能继续
        break;
    default: {                                       // >= 0：命令词命中，枚举值即 phrase_id
        esp_gmf_afe_vcmd_info_t *cmd = event->event_data;   // phrase_id / prob / str
        ESP_LOGI(TAG, "cmd id=%d phrase=%d prob=%.2f str=%s",
                 event->type, cmd->phrase_id, cmd->prob, cmd->str);
        break;
    }
    }
}
// 也可运行时再换回调
esp_gmf_afe_set_event_cb(ai_afe, my_afe_event_cb, NULL);
```

事件值：`WAKEUP_START=-100 / WAKEUP_END=-99 / VAD_START=-98 / VAD_END=-97 / VCMD_DECT_TIMEOUT=-96 / VCMD_DECTECTED=phrase_id(>=0)`。

### 5. 单能力 element：ai_aec（仅需 AEC 的录音 pipeline）

`ai_aec` 按 `input_format` 抽取麦+参考通道做 AEC，输出 16-bit 单声道 PCM；不依赖模型分区。

```c
#include "esp_gmf_aec.h"

esp_gmf_aec_cfg_t aec_cfg = {
    .filter_len   = 4,           // S3/P4 推荐 4，C5 推荐 2；越大 CPU 越重
    .type         = AFE_TYPE_SR, // 或 AFE_TYPE_VC / AFE_TYPE_VC_8K(8 kHz)
    .mode         = AFE_MODE_HIGH_PERF,   // 或 AFE_MODE_LOW_POWER
    .input_format = "MMNR",
};

// aec_rec 示例的真实 pipeline：io_codec_dev → aud_rate_cvt → ai_aec → [aud_enc] → 应用自管 outport
const char *name[] = {"aud_rate_cvt", "ai_aec"};     // 可选追加 "aud_enc" 编码
esp_gmf_pool_new_pipeline(pool, "io_codec_dev", name, 2, NULL, &pipe);

// 强制 ai_aec 输入采样率（示例用 aud_rate_cvt 把源转成 AEC_INPUT_SAMPLE_RATE）
esp_gmf_element_handle_t rate_cvt = NULL;
esp_gmf_pipeline_get_el_by_name(pipe, "aud_rate_cvt", &rate_cvt);
esp_gmf_rate_cvt_set_dest_rate(rate_cvt, 16000);
```

### 6. 单能力 element：ai_wn / ai_ns / ai_vad / ai_doa

```c
// ai_wn：轻量 WakeNet，process 内同步检测，无 feed/fetch 任务；命中走 detect_cb，PCM 透传输出
#include "esp_gmf_wn.h"
esp_gmf_wn_cfg_t wn_cfg = {
    .models       = models,
    .det_mode     = DET_MODE_2CH_90,   // 通道数与 input_format 的 M 数必须匹配
    .input_format = "MMNR",
    .detect_cb    = my_wakeup_cb,      // void(*)(obj, int32_t trigger_ch, void *ctx)
    .user_ctx     = &ctx,
};

// ai_ns：单通道降噪，16 kHz/16 bit/mono；NSNet2 模型或 WebRTC NS 后端
#include "esp_gmf_ns.h"
esp_gmf_ns_cfg_t ns_cfg = ESP_GMF_NS_CFG_DEFAULT();   // sample_rate/channel/frame_ms/model_name/partition_label

// ai_vad：单通道 VAD，状态变化走 result_callback
#include "esp_gmf_vad.h"
static void vad_cb(vad_state_t state, void *ctx) { /* state: SPEECH / SILENCE */ }
esp_gmf_vad_cfg_t vad_cfg = ESP_GMF_VAD_CFG_DEFAULT();
vad_cfg.result_callback = vad_cb;     // sample_rate 支持 8/16/32 kHz；frame_ms 10/20/30

// ai_doa：双麦声源定位，结果走 angle 回调，不输出 PCM
#include "esp_gmf_doa.h"
static void doa_cb(float angle, void *ctx) { /* 角度 */ }
esp_gmf_doa_cfg_t doa_cfg = ESP_GMF_DOA_CFG_DEFAULT();   // sample_rate/resolution/d_mics/frame_ms/input_format
doa_cfg.result_callback = doa_cb;     // input_format 必须恰好两个 M
```

### 7. 销毁

```c
esp_gmf_pipeline_stop(pipe);
esp_gmf_pipeline_destroy(pipe);
esp_gmf_afe_manager_destroy(mgr);   // element 销毁由 pipeline_destroy 处理；manager 需单独 destroy
esp_gmf_pool_deinit(pool);
```

> ai_afe 与 ai_wn 处理的都是原始多通道 PCM，二者不可在同一个 pipeline 串联；二选一。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 唤醒无事件 | `wakenet_init=false` / 模型未烧 / `M` 通道数与硬件不符 / 麦克风电平太低 | 检查 `afe_config_init` 第 4 参；烧 `model` 分区；用 `esp_gmf_afe_wakeup_info_t.data_volume` 反推电平 |
| `vcmd_detection_begin` 返回 `INVALID_STATE` | `vcmd_detect_en=false` | `esp_gmf_afe_cfg_t.vcmd_detect_en=true` 且 `mn_language` 与模型语言（cn/en）一致 |
| 命令词无响应 | `vcmd_timeout` 内未说话 / 超时未再 begin | 超时后会发 `VCMD_DECT_TIMEOUT`，需再次 `begin` |
| feed_task 看门狗超时 | CPU 争抢，单核 SoC 尤其严重 | feed/fetch 分到不同核（默认 core 0/1）；提高 `fetch_task_setting.prio`；或用专用 timer 任务喂数据 |
| ai_aec 残留回声 | R 通道未接扬声器参考 / 采样率非 16 kHz / 麦与参考时序错位 / `filter_len` 过小 | 确认参考接线；S3/P4 用 `filter_len=4`；见 `esp_gmf_aec.c` 头注与 esp-sr AEC 文档 |
| ai_doa 不工作 | `input_format` 不是恰好两个 `M` | 双麦板用 `"MMNR"`/`"MM"` 等，确保 M 数 == 2 |
| SoC 不支持 | 元素依赖 esp-sr 加速，矩阵见文档 | ai_afe：S3/S31/P4（C3/C5 不支持）；ai_doa：S3/S31；ai_aec：除 C3 外支持 |
| ai_afe 输出开头丢音 | `delay_samples` 换算时间 < `vad_min_speech_ms` | 调大 `delay_samples`（默认 2048 样本，约 64 ms） |

## 参考

- `elements/gmf_ai_audio/examples/wwe/main/main.c`（ai_afe 完整流程：pool→pipeline→event_cb→手动唤醒 keep/trigger 命令→VAD-to-file）
- `elements/gmf_ai_audio/examples/aec_rec/main/main.c`（双 pipeline 边播 test.mp3 边 ai_aec 录音）
- `elements/gmf_ai_audio/include/esp_gmf_afe.h`、`esp_gmf_afe_manager.h`、`esp_gmf_aec.h`、`esp_gmf_wn.h`、`esp_gmf_ns.h`、`esp_gmf_vad.h`、`esp_gmf_doa.h`、`esp_gmf_ai_audio_methods.h`
- `docs/en/gmf-framework/gmf-elements/gmf-ai-audio.rst`、`docs/zh_CN/gmf-framework/gmf-elements/gmf-ai-audio.rst`（含元素层次图、AFE manager 数据流、状态机、SoC 支持矩阵）
