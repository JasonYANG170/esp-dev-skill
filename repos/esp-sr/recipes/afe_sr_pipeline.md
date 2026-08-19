# AFE 语音识别主线（WakeNet + 命令词）

> **适用摘要**: 用 ESP-SR 的 Audio Front-End（AFE）搭建离线语音识别主线：唤醒词触发后切换到命令词识别，包含 menuconfig 选模型、分区配置、AFE 初始化与 feed/fetch 双任务驱动。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-sr/resources/`, source/examples in `repos/esp-sr/`, and this recipe path `repos/esp-sr/recipes/afe_sr_pipeline.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "做语音唤醒 + 命令词"
- "怎么用 AFE"
- "WakeNet + MultiNet 一起用"
- "feed / fetch 任务怎么写"
- "ESP-SR 入门示例"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32-S3 / ESP32-P4 / ESP32-S31（推荐，支持 PSRAM）；ESP32 也可 |
| ESP-IDF | >= 5.0 |
| 组件 | `espressif/esp-sr`（随 esp-skainet 引入或 managed component） |
| 模型 | menuconfig 已选 WakeNet + MultiNet 模型并 `idf.py flash` 烧入 `model` 分区 |
| 参考代码 | `test_apps/esp-sr/main/test_afe.cpp`、`test_multinet.cpp` |

## 分步说明

### 1. partitions.csv 增加 model 分区

```csv
# Name,   Type, SubType, Offset, Size
nvs,      data, nvs
phy_init, data, phy
factory,  app,  factory,        , 1M
model,    data,         ,        , 6000K
```

> `model` 标签固定；6000K 为参考值，按所选模型大小调整（见 `docs/benchmark`）。

### 2. menuconfig 选择模型

```
idf.py menuconfig
ESP Speech Recognition -->
    model data path --> MODEL_IN_FLASH
    Select noise suppression model
    Select voice activity detection --> vadnet1 medium
    Select WakeNet --> wn9_hilexin (或 wn9_hiesp)
    Chinese Speech Commands Model / English Speech Commands Model --> mn7_cn / mn7_en
    Add Chinese/English speech commands  (可选，也可代码里 add)
```

### 3. AFE 初始化与模型加载

```c
#include "esp_afe_sr_iface.h"
#include "esp_afe_config.h"
#include "esp_afe_sr_models.h"
#include "model_path.h"
#include "esp_log.h"

static const char *TAG = "SR";
static const esp_afe_sr_iface_t *afe_handle = NULL;

// 单声道麦克风 + 1 路播放参考（AEC 用）。纯单麦可写 "M"
static void sr_init(esp_afe_sr_data_t **out_afe_data, srmodel_list_t **out_models)
{
    // 1. 加载 flash 里的模型（label 必须是 "model"）
    srmodel_list_t *models = esp_srmodel_init("model");

    // 2. 默认配置：语音识别场景 + 高性能模式
    //    "MR" = 1 麦克风 + 1 播放参考（AEC）
    afe_config_t *cfg = afe_config_init("MR", models, AFE_TYPE_SR, AFE_MODE_HIGH_PERF);

    // 3. 可选微调（按需）
    // cfg->wakenet_model_name = "wn9_hilexin";
    // cfg->vad_init = true;
    // cfg->aec_init = true;

    // 4. handle + 实例
    afe_handle = esp_afe_handle_from_config(cfg);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(cfg);
    afe_config_free(cfg);

    *out_afe_data = afe_data;
    *out_models = models;
    afe_handle->print_pipeline(afe_data);  // 打印 [input]->|...|->[output]
}
```

### 4. feed 任务（喂音频）

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

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
        // TODO: 从 I2S 读取多通道交错音频填入 buff
        // read_i2s(buff, chunksize * nch);
        ctx->afe_handle->feed(ctx->afe_data, buff);
        // 控制喂入速率，避免 ringbuf 溢出
        vTaskDelay(chunksize /
                   (ctx->afe_handle->get_samp_rate(ctx->afe_data) / 1000) /
                   portTICK_PERIOD_MS);
    }
    free(buff);
    vTaskDelete(NULL);
}
```

### 5. fetch 任务（取结果 + 驱动 MultiNet）

```c
#include "esp_mn_iface.h"
#include "esp_mn_models.h"
#include "esp_mn_speech_commands.h"

void fetch_task(void *arg)
{
    afe_ctx_t *ctx = (afe_ctx_t *)arg;
    esp_afe_sr_data_t *afe_data = ctx->afe_data;

    // 加载 MultiNet（命令词）
    char *mn_name = esp_srmodel_filter(get_static_srmodels(), ESP_MN_PREFIX, NULL);
    esp_mn_iface_t *multinet = esp_mn_handle_from_name(mn_name);
    model_iface_data_t *mn_data = multinet->create(mn_name, 6000); // timeout 6s

    // 添加命令词（中文 mn6/mn7 用拼音或汉字；英文 mn6/mn7 用 grapheme）
    esp_mn_commands_clear();
    esp_mn_commands_add(1, "da kai dian deng");
    esp_mn_commands_add(2, "guan bi dian deng");
    esp_mn_error_t *err = esp_mn_commands_update();
    if (err != NULL) ESP_LOGE(TAG, "some commands failed to parse");

    int detect_flag = 0;
    while (1) {
        afe_fetch_result_t *res = ctx->afe_handle->fetch(afe_data);
        if (!res || res->ret_value == ESP_FAIL) break;

        // 唤醒检测
        if (res->wakeup_state == WAKENET_DETECTED) {
            ESP_LOGI(TAG, "wake word #%d on ch %d", res->wake_word_index, res->trigger_channel_id);
            detect_flag = 1;
        }
        // 命令词识别（唤醒后才开始）
        if (detect_flag) {
            esp_mn_state_t st = multinet->detect(mn_data, res->data);
            if (st == ESP_MN_STATE_DETECTED) {
                esp_mn_results_t *r = multinet->get_results(mn_data);
                if (r->num > 0)
                    ESP_LOGI(TAG, "cmd id=%d str=%s", r->command_id[0], r->string);
                detect_flag = 0;              // 单次模式：识别到即退出本轮
                multinet->clean(mn_data);
            } else if (st == ESP_MN_STATE_TIMEOUT) {
                ESP_LOGI(TAG, "command timeout");
                detect_flag = 0;              // 连续模式：超时再回到等唤醒
            }
        }
    }
    multinet->destroy(mn_data);
    vTaskDelete(NULL);
}
```

### 6. 在 app_main 启动

```c
void app_main(void)
{
    esp_afe_sr_data_t *afe_data = NULL;
    srmodel_list_t *models = NULL;
    sr_init(&afe_data, &models);

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
| 启动打印 `No wakenet model found` | model 分区未烧 / menuconfig 未选模型 | `idf.py menuconfig` 选模型后 `idf.py flash`（不是 app-flash） |
| `esp_srmodel_filter` 返回 NULL | 同上，或 prefix 写错 | WakeNet 用 `ESP_WN_PREFIX`，MultiNet 用 `ESP_MN_PREFIX` |
| 唤醒后听不到命令词 | 没调 `esp_mn_commands_update()` | add 之后必须 update，且返回 NULL 才算成功 |
| fetch 卡死/无数据 | feed 任务没起或速率不对 | feed 与 fetch 必须同时运行；feed 按 chunksize/samp_rate 节流 |
| 多麦场景通道错乱 | `input_format` 与硬件不一致 | 用 `afe_config_print` 看 pcm_config，核对 M/R/N 顺序 |
| 内存不足崩 | HIGH_PERF + 大模型 | 换 `AFE_MODE_LOW_COST`，MultiNet 用 `ESP_MN_LOAD_FROM_FLASH` |
| 命令词识别为空字符串 | 命令词含数字/特殊字符 | 去掉阿拉伯数字与标点；中文用拼音，英文用全大写 |

## 参考

- `test_apps/esp-sr/main/test_afe.cpp` — AFE feed/fetch 双任务骨架（`test_feed_Task`/`test_fetch_Task`、`afe_task_into_t`）
- `test_apps/esp-sr/main/test_multinet.cpp` — MultiNet create/detect/get_results
- `docs/en/audio_front_end/README.rst` — AFE 官方使用步骤
- `docs/en/speech_command_recognition/README.rst` — MultiNet 输出与状态
- 配套 recipe：`recipes/multinet_commands.md`、`recipes/model_partition.md`、`recipes/migration_v1_v2.md`
