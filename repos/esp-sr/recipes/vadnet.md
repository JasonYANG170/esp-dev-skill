# VADNet 语音活动检测

> **适用摘要**: 使用 VADNet（神经网络 VAD，替代 WebRTC VAD）检测语音/噪声状态，处理 VAD cache 防止首字截断。可经 AFE pipeline 默认启用，或单独运行。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-sr/resources/`, source/examples in `repos/esp-sr/`, and this recipe path `repos/esp-sr/recipes/vadnet.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "VAD 检测"
- "VADNet 怎么用"
- "vad_cache 首字丢失"
- "替换 WebRTC VAD"
- "语音端点检测"

## 前置条件

| 条件 | 要求 |
|---|---|
| 模型 | menuconfig: `ESP Speech Recognition → Select voice activity detection → vadnet1 medium`（`SR_VADN_VADNET`） |
| 音频 | 16kHz / 16-bit / 单声道 |
| 参考 | `test_apps/esp-sr/main/test_vadnet.cpp`、`docs/en/vadnet/README.rst` |

## 分步说明

### 方式 A：经 AFE 启用（推荐，生产用法）

VADNet 默认在 AFE pipeline 中开启。VAD 相关配置：

```c
#include "esp_afe_config.h"

afe_config_t *cfg = afe_config_init("M", models, AFE_TYPE_SR, AFE_MODE_HIGH_PERF);
cfg->vad_init          = true;       // 默认 true
cfg->vad_mode          = VAD_MODE_1; // 0~4，越大越严格
cfg->vad_min_speech_ms = 128;        // 触发语音的最短时长（ms，>32）
cfg->vad_min_noise_ms  = 1000;       // 触发噪声/静音的最短时长（ms，>64）
cfg->vad_delay_ms      = 128;        // 首帧语音延迟（ms）；cache 不够覆盖时增大
cfg->vad_model_name    = NULL;       // NULL=用 menuconfig 选的，或显式给名
```

运行时控制：

```c
afe_handle->disable_vad(afe_data);   // 临时关
afe_handle->enable_vad(afe_data);    // 临时开
afe_handle->reset_vad(afe_data);     // 重置状态
```

取 VAD 结果：

```c
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
// res->vad_state  : VAD_SILENCE / VAD_SPEECH
// res->vad_cache  : 首字截断补偿数据
// res->vad_cache_size : cache 字节数，>0 时需写入
```

### 关键：处理 vad_cache 防止“吃字”

VAD 算法本身有 1~3 帧固有延迟，加上 `vad_min_speech_ms` 防误触，直接用首帧会截掉第一个字。AFE V2.0 内置 VAD cache：

```c
// ❌ WRONG — 录音/识别丢首字
fwrite(res->data, 1, res->data_size, fp);

// ✅ CORRECT — 先写 cache 再写 data
if (res->vad_cache_size > 0) {
    printf("vad cache %d bytes\n", res->vad_cache_size);
    fwrite(res->vad_cache, 1, res->vad_cache_size, fp);
    // 或把 cache 接到送给 ASR/识别缓冲区的前面
}
fwrite(res->data, 1, res->data_size, fp);
printf("vad: %s\n", res->vad_state == VAD_SILENCE ? "noise" : "speech");
```

### 方式 B：单独运行 VADNet（测试场景）

```c
#include "model_path.h"
#include "esp_vadn_iface.h"
#include "esp_vadn_models.h"

srmodel_list_t *models = esp_srmodel_init("model");
char *name = esp_srmodel_filter(models, ESP_VADN_PREFIX, NULL);  // ESP_VADN_PREFIX
esp_vadn_iface_t *vadnet = (esp_vadn_iface_t *)esp_vadn_handle_from_name(name);

// create(model_name, vad_mode, channel_num, min_speech_ms, min_noise_ms)
model_iface_data_t *md = vadnet->create(name, VAD_MODE_0, 1, 32, 64);
int chunksize = vadnet->get_samp_chunksize(md);
int16_t *buf  = malloc(chunksize * sizeof(int16_t));

// detect 返回 vad_state_t：VAD_SILENCE / VAD_SPEECH
vad_state_t st = vadnet->detect(md, buf);
if (st == VAD_SPEECH) { /* 有人说话 */ }

vadnet->destroy(md);
esp_srmodel_deinit(models);
```

### WebRTC VAD（备选，非神经网络）

若不需要 VADNet，menuconfig 选 `SR_VADN_WEBRTC`，直接用 `esp_vad.h`：

```c
#include "esp_vad.h"
vad_handle_t vad = vad_create(VAD_MODE_3);   // 或 vad_create_with_param(mode, 16000, 30, ms, ns)
// 每帧（30ms = 480 样本 @16k）:
vad_state_t st = vad_process(vad, buf, 16000, 30);  // VAD_SILENCE / VAD_SPEECH
vad_destroy(vad);
```

`vad_mode_t`: `VAD_MODE_0`(Normal) ~ `VAD_MODE_4`(Very Very Very Aggressive)。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 录音/识别丢第一个字 | 没处理 vad_cache | `res->vad_cache_size>0` 时先写 cache |
| VAD 一直报 silence | mode 太严 / min_speech_ms 太大 | 降 mode 到 `VAD_MODE_0`；减 `vad_min_speech_ms` |
| VAD 误触发频繁 | mode 太宽 | 升 mode 到 `VAD_MODE_3/4` |
| `esp_srmodel_filter(ESP_VADN_PREFIX)` 返回 NULL | menuconfig 没选 VADNet | 选 `vadnet1 medium` 再 flash |
| cache 还是覆盖不全 | `vad_delay_ms` 太小 | 增大 `cfg->vad_delay_ms`（如 256） |
| 想用 WebRTC VAD 但效果差 | 用了旧的 vad_process | WebRTC VAD 对稳态噪声尚可，复杂场景换 VADNet |

## 参考

- `test_apps/esp-sr/main/test_vadnet.cpp` — create/detect/cpu loading 用例
- `include/esp32s3/esp_vadn_iface.h` — `esp_vadn_iface_t`
- `include/esp32s3/esp_vad.h` — WebRTC VAD（`vad_create`、`VAD_MODE_*`）
- `include/esp32s3/esp_afe_config.h` — `afe_config_t` 的 vad_* 字段
- `docs/en/vadnet/README.rst` — VADNet 原理、vad_cache 说明
- 配套 recipe：`recipes/afe_sr_pipeline.md`
