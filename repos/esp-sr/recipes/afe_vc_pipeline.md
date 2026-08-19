# AFE 语音通信主线（VC / FD）

> **适用摘要**: 用 ESP-SR AFE 搭建**语音通信 / 全双工**主线：选 `AFE_TYPE_VC` / `AFE_TYPE_VC_8K` / `AFE_TYPE_FD`，得到与 SR 不同的 pipeline（VC 走 AEC(VOIP)→NS(nsnet2)→VAD；FD 走 AEC(FD)→SE(BSS)→VAD→WakeNet），并把 `fetch` 出来的干净单声道音频送往网络传输或录音。与 SR 主线共用 feed/fetch 双任务骨架。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-sr/resources/`, source/examples in `repos/esp-sr/`, and this recipe path `repos/esp-sr/recipes/afe_vc_pipeline.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "语音通信 / VoIP"
- "全双工对话"
- "AFE_TYPE_VC / AFE_TYPE_FD"
- "8kHz 通话怎么配"
- "NS 降噪 + AEC 一起用"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32-S3 / ESP32-P4 / ESP32-S31（推荐，支持 PSRAM）；ESP32 也可 |
| ESP-IDF | >= 5.0 |
| 组件 | `espressif/esp-sr` |
| 模型 | VC 默认需 `nsnet2`（NS）+ `vadnet1_medium`；FD 默认需 `vadnet1_medium`，可选 `wn9_hilexin` + BSS。menuconfig 选好并烧入 `model` 分区 |
| 输入 | 16kHz/16-bit 交错多通道（VC/FD）；`AFE_TYPE_VC_8K` 必须喂 **8kHz** 数据 |
| 参考 | `test_apps/esp-sr/main/test_afe.cpp`（`AFE default setting`、`afe performance test (1ch/2ch)`） |

## VC / FD / SR 怎么选

`afe_type_t`（`esp_afe_config.h`）是 AFE 的顶层场景开关，直接决定内部 AEC 用哪一组 mode、是否启用非线性 NS/BSS：

| `afe_type_t` | 场景 | 内部 AEC mode | 非线性降噪 | 典型用途 |
|---|---|---|---|---|
| `AFE_TYPE_SR` | 语音识别 | `AEC_MODE_SR_*`（仅线性） | 不含 | 唤醒词 + 命令词 |
| `AFE_TYPE_VC` | 语音通信（16kHz） | `AEC_MODE_VOIP_*` | **NS(nsnet2)** | VoIP / 对讲机（16k 通路） |
| `AFE_TYPE_VC_8K` | 语音通信（**8kHz**） | `AEC_MODE_VOIP_*` | **NS(nsnet2)** | VoIP / 对讲机（窄带 8k 通路） |
| `AFE_TYPE_FD` | 全双工对话 | `AEC_MODE_FD_*`（线性+非线性 NLP） | SE(BSS)（双麦时） | 带屏音箱对话、远场通话 |

> `AFE_TYPE_VC` 与 `AFE_TYPE_FD` 的本质差别：VC 输出的目标是**通信码流**（人听着干净即可，AEC 用 VOIP 档 + NS 做非线性降噪），FD 输出的目标是**同时喂唤醒词**（AEC 用 FD 档含 NLP，双麦时加 BSS 做盲源分离，末段保留 WakeNet）。

## Pipeline 对照（来自 docs/en/benchmark/README.rst）

benchmark 文档列出了 ESP32-S3 / ESP32-P4 上各配置的完整 pipeline 与资源占用。VC 与 SR/FD 的 pipeline 明显不同：

| input_format + type + mode | Pipeline |
|---|---|
| `MR`, **VC**, LOW_COST | `|AEC(VOIP_LOW_COST)| -> |NS(nsnet2)| -> |VAD(vadnet1_medium)|` |
| `MR`, **VC**, HIGH_PERF | `|AEC(VOIP_HIGH_PERF)| -> |NS(nsnet2)| -> |VAD(vadnet1_medium)|` |
| `MR`, **FD**, LOW_COST | `|AEC(FD_LOW_COST)| -> |VAD(vadnet1_medium)| -> |WakeNet(wn9_hilexin)|` |
| `MMNR`, **FD**, LOW_COST | `|AEC(FD_LOW_COST)| -> |SE(BSS)| -> |VAD(vadnet1_medium)| -> |WakeNet(wn9_hilexin)|` |
| `MR`, **SR**, LOW_COST | `|AEC(SR_LOW_COST)| -> |VAD(vadnet1_medium)| -> |WakeNet(wn9_hilexin)|` |

要点：
- **VC 独有 NS(nsnet2)**：SR/FD 的单麦分支不做非线性 NS，只做线性 AEC；VC 用 nsnet2 做非线性稳态噪声抑制。
- **VC 不带 WakeNet**：VC 是纯通信管道，不需要唤醒；fetch 出来的就是干净单声道音频，直接送编码器/网络。
- **FD 带 WakeNet**：FD 兼顾对话与唤醒（带屏音箱边播放边听唤醒词）；双麦（MMNR）时加 SE(BSS) 做盲源分离。
- **AEC mode 跟着 type 走**：不要在 `AFE_TYPE_VC` 下手动把 `aec_mode` 改成 `AEC_MODE_SR_*`——VOIP 档针对双向通话调过参数，混用会劣化。

## 资源占用参考（ESP32-S3，单麦 MR / 双麦 MMNR）

来自 `docs/en/benchmark/README.rst`（单核 CPU 占用）：

| Config | Internal RAM (KB) | PSRAM (KB) | Feed CPU (%) | Fetch CPU (%) |
|---|---|---|---|---|
| MR, VC, LOW_COST | 48.7 | 819.7 | 30.6 | 4.7 |
| MR, VC, HIGH_PERF | 91.1 | 822.2 | 32.2 | 4.7 |
| MR, FD, LOW_COST | 60.2 | 777.7 | 12.1 | 9.8 |
| MR, FD, HIGH_PERF | 49.2 | 813.8 | 12.5 | 9.8 |
| MMNR, FD, LOW_COST | 79.2 | 1191.7 | 29.4 | 22.9 |
| MMNR, FD, HIGH_PERF | 68.1 | 1238.5 | 30.4 | 22.9 |
| MR, SR, LOW_COST（对照） | 60.1 | 739.7 | 8.8 | 9.8 |

> VC 的 feed CPU（30%+）明显高于 SR/FD，因为 nsnet2 是神经网络模型；FD 双麦（MMNR）总开销最高。

## 分步说明

### 1. menuconfig 选模型

VC 需要 NS + VAD；FD 需要 VAD（可选 WakeNet）：

```
idf.py menuconfig
ESP Speech Recognition -->
    Select noise suppression model --> nsnet2          # VC 必选；FD 不需要
    Select voice activity detection --> vadnet1 medium  # VC/FD 都选
    Select WakeNet --> wn9_hilexin                      # 仅 FD 需要；VC 不选
    （SE/BSS 由 AFE 在双麦 MMNR 时自动启用，无需选模型）
```

### 2. partitions.csv + model 分区

与 SR 一致（VC/FD 也要把模型烧进 `model` 分区）：

```csv
# Name,   Type, SubType, Offset, Size
nvs,      data, nvs
phy_init, data, phy
factory,  app,  factory,        , 1M
model,    data,         ,        , 6000K
```

### 3. AFE 初始化（VC，16kHz）

与 SR 主线的差别只有 `afe_type`：把 `AFE_TYPE_SR` 换成 `AFE_TYPE_VC`。feed/fetch 任务骨架完全一致。

```c
#include "esp_afe_sr_iface.h"
#include "esp_afe_config.h"
#include "esp_afe_sr_models.h"
#include "model_path.h"
#include "esp_log.h"

static const char *TAG = "VC";
static const esp_afe_sr_iface_t *afe_handle = NULL;

static void vc_init(esp_afe_sr_data_t **out_afe_data)
{
    // 加载模型（VC 需要 nsnet2 + vadnet1）
    srmodel_list_t *models = esp_srmodel_init("model");

    // 单麦 + 1 播放参考。VC 16kHz 输入
    afe_config_t *cfg = afe_config_init("MR", models, AFE_TYPE_VC, AFE_MODE_HIGH_PERF);

    // 可选微调（通常 afe_config_init 已按 type 给出合理默认）
    // cfg->aec_mode = AEC_MODE_VOIP_HIGH_PERF;  // 默认就是 VOIP_*
    // cfg->ns_model_name = "nsnet2";

    afe_handle = esp_afe_handle_from_config(cfg);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(cfg);
    afe_config_free(cfg);

    *out_afe_data = afe_data;
    afe_handle->print_pipeline(afe_data);  // 应打印 |AEC(VOIP_*)| -> |NS(nsnet2)| -> |VAD|
}
```

### 4. 8kHz 通话：用 `AFE_TYPE_VC_8K`

窄带 VoIP（G.711 等码流）常用 8kHz。**输入数据必须是 8kHz**，AFE 内部不再做 16k 重采样：

```c
// ⚠️ 输入必须是 8kHz/16-bit；feed 进来的就是 8k 采样
afe_config_t *cfg = afe_config_init("MR", models, AFE_TYPE_VC_8K, AFE_MODE_LOW_COST);
```

> 选错 type 会导致采样率不匹配：把 8k 数据喂给 `AFE_TYPE_VC`（期望 16k）会得到变速/变调的输出。

### 5. feed 任务（喂音频）

与 SR 主线**完全一致**——按 `get_feed_chunksize` / `get_feed_channel_num` 查询帧长，喂交错多通道数据：

```c
typedef struct {
    esp_afe_sr_data_t *afe_data;
    const esp_afe_sr_iface_t *afe_handle;
} afe_ctx_t;

void feed_task(void *arg)
{
    afe_ctx_t *ctx = (afe_ctx_t *)arg;
    int chunksize = ctx->afe_handle->get_feed_chunksize(ctx->afe_data);
    int nch       = ctx->afe_handle->get_feed_channel_num(ctx->afe_data);
    int16_t *buff = malloc(chunksize * sizeof(int16_t) * nch);

    while (1) {
        // TODO: 从 I2S 读取交错多通道（含 1 路播放参考 R）
        ctx->afe_handle->feed(ctx->afe_data, buff);
        vTaskDelay(chunksize /
                   (ctx->afe_handle->get_samp_rate(ctx->afe_data) / 1000) /
                   portTICK_PERIOD_MS);
    }
    free(buff);
    vTaskDelete(NULL);
}
```

### 6. fetch 任务（取干净音频 + 送网络）

VC 场景下 `fetch` 出来的 `res->data` 是去回声 + 降噪后的单声道 PCM，直接送编码器/网络；VAD 状态用来做舒适噪声 / 静音抑制（DTX）：

```c
void fetch_task(void *arg)
{
    afe_ctx_t *ctx = (afe_ctx_t *)arg;
    esp_afe_sr_data_t *afe_data = ctx->afe_data;

    while (1) {
        afe_fetch_result_t *res = ctx->afe_handle->fetch(afe_data);
        if (!res || res->ret_value == ESP_FAIL) break;

        // VC：res->data 是干净单声道，送 Opus/G.711 编码 + 网络
        // encode_and_send(res->data, res->data_size);

        // 用 VAD 状态做静音处理（可选）
        if (res->vad_state == VAD_SPEECH) {
            // 有语音：正常编码发送
        } else {
            // 静音：可发舒适噪声帧或 DTX
        }
    }
    vTaskDelete(NULL);
}
```

> VC 场景下 `res->wakeup_state` 始终为 `WAKENET_NO_DETECT`（VC pipeline 不含 WakeNet）；FD 场景下才会出现 `WAKENET_DETECTED`。

### 7. FD 全双工：唤醒 + 对话并行

把 type 换成 `AFE_TYPE_FD`，pipeline 自动变成 AEC(FD)→[SE(BSS)]→VAD→WakeNet。fetch 同时给出干净音频（送对话/通话）与唤醒状态：

```c
afe_config_t *cfg = afe_config_init("MMNR", models, AFE_TYPE_FD, AFE_MODE_HIGH_PERF);
// "MMNR" = 双麦 + 1 参考 → FD 会启用 SE(BSS) 盲源分离
```

在 fetch 里既判唤醒、又把 `res->data` 送远场通话/对话 ASR：

```c
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
if (res->wakeup_state == WAKENET_DETECTED) {
    ESP_LOGI(TAG, "wake word #%d", res->wake_word_index);
    // 进入对话态…
}
// res->data 送对话/通话通路（已 AEC + BSS）
```

### 8. 在 app_main 启动

与 SR 主线一致，两个任务分别钉在一个核：

```c
void app_main(void)
{
    esp_afe_sr_data_t *afe_data = NULL;
    vc_init(&afe_data);

    static afe_ctx_t ctx = {0};
    ctx.afe_data = afe_data;
    ctx.afe_handle = afe_handle;
    xTaskCreatePinnedToCore(feed_task, "feed", 8 * 1024, &ctx, 5, NULL, 0);
    xTaskCreatePinnedToCore(fetch_task, "fetch", 4 * 1024, &ctx, 5, NULL, 1);
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 输出变速/变调 | `AFE_TYPE_VC` 喂了 8k 数据（或反之） | 8k 通路用 `AFE_TYPE_VC_8K`，16k 用 `AFE_TYPE_VC` |
| 回声没消干净 | 用了 SR 档 AEC | VC 必须让 `afe_type=AFE_TYPE_VC`（内部自动用 `AEC_MODE_VOIP_*`）；FD 用 `AFE_TYPE_FD`（自动 `AEC_MODE_FD_*`） |
| pipeline 打印里没有 NS | menuconfig 没选 nsnet2 | VC 下 menuconfig 选 `nsnet2` 并重新 `idf.py flash` |
| pipeline 打印里没有 WakeNet（VC） | 正常现象 | VC pipeline 本就不含 WakeNet；要唤醒改用 FD 或 SR |
| VC 下 `res->wakeup_state` 永远是 NO_DETECT | 正常现象 | VC 不含 WakeNet；如需唤醒换 `AFE_TYPE_FD`/`AFE_TYPE_SR` |
| FD 双麦没启 BSS | `input_format` 是 `MR` 而非 `MMNR` | BSS 需要双麦，format 含 `MM`；`afe_config_check` 会按双麦优先 SE 而非 NS |
| VC feed CPU 飙到 30%+ | nsnet2 是神经网络，本身较重 | 用 `AFE_MODE_LOW_COST`；或对讲机类窄带用 `AFE_TYPE_VC_8K` |
| 双麦 FD 内存不足 | MMNR + FD_HIGH_PERF 最耗资源 | 降 `AFE_MODE_LOW_COST`；`memory_alloc_mode` 设 `MORE_PSRAM` |
| 回声反而比 SR 还大 | 把 `aec_mode` 手动改成了 `AEC_MODE_SR_*` | 不要在 VC/FD 下手动覆盖 `aec_mode`，让它跟着 type 走 VOIP/FD 档 |

## 参考

- `test_apps/esp-sr/main/test_afe.cpp` — `TEST_CASE(">>>>>>>> AFE default setting")` 遍历 SR/FD/VC × LOW/HIGH × MR/MMNR；`TEST_CASE("afe performance test (1ch/2ch)")` 用 `AFE_TYPE_VC` + `test_feed_Task`/`test_fetch_Task` 跑满帧
- `docs/en/audio_front_end/README.rst` — "Voice Communication" 场景说明、input_format 定义、feed/fetch 基本步骤
- `docs/en/benchmark/README.rst` — VC/FD/SR 各配置的完整 pipeline 与 Internal RAM/PSRAM/CPU 占用表（ESP32-S3、ESP32-P4）
- `include/esp32s3/esp_afe_config.h` — `afe_type_t`（SR/VC/VC_8K/FD）、`afe_config_init(input_format, models, type, mode)`、`afe_config_check` 自动按双麦优先 SE
- 配套 recipe：`recipes/afe_sr_pipeline.md`（SR 主线 feed/fetch 骨架，VC/FD 完全复用）、`recipes/aec_usage.md`（SR/FD/VOIP 三类 AEC mode 详解）
