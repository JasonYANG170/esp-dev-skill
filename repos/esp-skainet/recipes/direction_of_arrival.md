# 双麦方向角估计（DOA）

> **适用摘要**: 用 `esp_doa` 模块基于双麦克风做声源方向角（DOA）估计。需要关闭 AEC、按 input_format 中 `M` 的位置手动拆出左右声道，再喂给 `esp_doa_process`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-skainet/resources/`, source/examples in `repos/esp-skainet/`, and this recipe path `repos/esp-skainet/recipes/direction_of_arrival.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "direction of arrival"
- "DOA 方向角"
- "双麦声源定位"
- "esp_doa"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/direction_of_arrival/` |
| 依赖组件 | `espressif/esp-sr` (^2.0.0) |
| 头文件 | `esp_doa.h` |
| 硬件 | 双麦克风板（如 ESP32-S3-Korvo-1），麦距需已知 |
| SD 卡 | 可选（落盘原始 PCM 评估） |

## 分步说明

### 1. 引入头文件

```c
#include "esp_afe_sr_iface.h"
#include "esp_afe_sr_models.h"
#include "esp_board_init.h"
#include "model_path.h"
#include "ringbuf.h"
#include "esp_doa.h"      // doa_handle_t, esp_doa_create/process/destroy
```

### 2. feed 任务：创建 DOA handle + 拆声道 + 处理

```c
void feed_Task(void *arg)
{
    int fs = 16000;
    float resolution = 20.f;     // 角度分辨率（度）
    float d_mics = 0.065f;       // 两麦物理间距（米），需与实际板子一致

    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    int audio_chunksize = afe_handle->get_feed_chunksize(afe_data);
    doa_handle_t *doa_handle = esp_doa_create(fs, resolution, d_mics, audio_chunksize);

    int nch = afe_handle->get_feed_channel_num(afe_data);
    int feed_channel = esp_get_feed_channel();
    assert(nch == feed_channel);

    int16_t *i2s_buff = malloc(audio_chunksize * sizeof(int16_t) * feed_channel);
    int16_t *ileft    = malloc(audio_chunksize * sizeof(int16_t));
    int16_t *iright   = malloc(audio_chunksize * sizeof(int16_t));

    // 解析 input_format 中 'M' 的位置
    char *str = esp_get_input_format();
    int positions[10], count = 0;
    for (int i = 0; str[i]; i++) {
        if (str[i] == 'M') positions[count++] = i;
    }

    while (task_flag) {
        esp_get_feed_data(true, i2s_buff, audio_chunksize * sizeof(int16_t) * feed_channel);

        // 交错数据 → 左右声道
        for (int i = 0; i < audio_chunksize; i++) {
            ileft[i]  = i2s_buff[i * feed_channel + positions[0]];
            iright[i] = i2s_buff[i * feed_channel + positions[1]];
        }

        float fdoa = esp_doa_process(doa_handle, ileft, iright);
        printf("fdoa: %f\n", fdoa);
    }

    free(i2s_buff); free(ileft); free(iright);
    esp_doa_destroy(doa_handle);
    vTaskDelete(NULL);
}
```

### 3. app_main：关闭 AEC

```c
void app_main()
{
    ESP_ERROR_CHECK(esp_board_init(16000, 1, 16));

    srmodel_list_t *models = esp_srmodel_init("model");
    afe_config_t *afe_config = afe_config_init(esp_get_input_format(), models,
                                               AFE_TYPE_SR, AFE_MODE_LOW_COST);
    printf("%s\n", esp_get_input_format());

    // 关键：DOA 不需要 AEC，且 AEC 会消耗回采通道
    afe_config->aec_init = false;
    afe_config_print(afe_config);

    afe_handle = esp_afe_handle_from_config(afe_config);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
    afe_config_free(afe_config);

    task_flag = 1;
    xTaskCreatePinnedToCore(&feed_Task, "feed", 8 * 1024, (void *)afe_data, 5, NULL, 0);
    // DOA 示例只跑 feed 任务做角度估计，detect 任务被注释
}
```

### 4. 参数说明

| 参数 | 含义 | 示例值 |
|---|---|---|
| `fs` | 采样率 | 16000 |
| `resolution` | 角度分辨率（度） | 20.f |
| `d_mics` | 双麦间距（米） | 0.065f（Korvo-1 约 6.5cm） |
| `audio_chunksize` | 帧样本数（来自 AFE） | 由 `get_feed_chunksize` 返回 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 角度恒为 0 | AEC 没关，通道被吞 | `afe_config->aec_init = false` |
| 左右声道相同 | 没按 `M` 位置拆，取了同一通道 | 用 input_format 解析 positions |
| 角度跳变剧烈 | `d_mics` 与实际麦距不符 | 量准实际麦距填入 |
| 只有一个 `M` | input_format 是单麦 | DOA 需双麦板，换板或改 input_format |
| `positions[1]` 越界 | input_format 里只有一个 M | 检查板子配置，确保双麦 |

## 参考

- `examples/direction_of_arrival/main/main.c` — 本 recipe 真实来源
- `espressif-repos/esp-sr/include/esp32s3/esp_doa.h` — `doa_handle_t`、`esp_doa_create/process/destroy`
- `espressif-repos/esp-sr/include/esp32s3/esp_afe_config.h` — `aec_init` 字段
