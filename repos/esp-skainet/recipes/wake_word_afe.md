# 基于 AFE 的实时唤醒词检测

> **适用摘要**: 使用 Audio Front-End (AFE) 在 ESP32-S3 上实时检测唤醒词（WakeNet），包含 feed/detect 双任务、模型加载、唤醒阈值调节与多模型加载。

## 触发意图

- "wake word detection" / "唤醒词检测"
- "WakeNet" / "Hi Lexin" / "Hi ESP"
- "实时语音唤醒"
- "AFE 实时检测唤醒词"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/wake_word_detection/afe/` |
| 依赖组件 | `espressif/esp-sr` (^2.0.0)，见 `main/idf_component.yml` |
| Kconfig | `ESP Speech Recognition → Select wake words` 勾选至少一个 WakeNet（如 `Hi,ESP (wn9_hiesp)`） |
| Flash | ≥ 8 MB（推荐 16 MB QIO） |
| PSRAM | ESP32-S3 需 octal PSRAM |

## 分步说明

### 1. 引入头文件

```c
#include "esp_wn_iface.h"
#include "esp_wn_models.h"
#include "esp_afe_sr_models.h"
#include "esp_board_init.h"
#include "model_path.h"
```

### 2. app_main：初始化板级 + 模型 + AFE

```c
static const esp_afe_sr_iface_t *afe_handle = NULL;
static volatile int task_flag = 0;

void app_main(void)
{
    ESP_ERROR_CHECK(esp_board_init(16000, 1, 16));

    srmodel_list_t *models = esp_srmodel_init("model");
    // 打印 flash 中所有 WakeNet 模型
    if (models) {
        for (int i = 0; i < models->num; i++) {
            if (strstr(models->model_name[i], ESP_WN_PREFIX) != NULL) {
                printf("wakenet model in flash: %s\n", models->model_name[i]);
            }
        }
    }

    afe_config_t *afe_config = afe_config_init(esp_get_input_format(), models,
                                               AFE_TYPE_SR, AFE_MODE_LOW_COST);
    // 查看配置里默认选中的唤醒词模型（最多两个）
    if (afe_config->wakenet_model_name)
        printf("wakeword model in AFE config: %s\n", afe_config->wakenet_model_name);
    if (afe_config->wakenet_model_name_2)
        printf("wakeword model in AFE config: %s\n", afe_config->wakenet_model_name_2);

    afe_handle = esp_afe_handle_from_config(afe_config);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
    afe_config_free(afe_config);

    task_flag = 1;
    xTaskCreatePinnedToCore(&feed_Task, "feed", 8 * 1024, (void *)afe_data, 5, NULL, 0);
    xTaskCreatePinnedToCore(&detect_Task, "detect", 4 * 1024, (void *)afe_data, 5, NULL, 1);
}
```

### 3. feed 任务：取音并喂给 AFE

```c
void feed_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    int audio_chunksize = afe_handle->get_feed_chunksize(afe_data);
    int nch = afe_handle->get_feed_channel_num(afe_data);
    int feed_channel = esp_get_feed_channel();
    assert(nch == feed_channel);
    int16_t *i2s_buff = malloc(audio_chunksize * sizeof(int16_t) * feed_channel);
    assert(i2s_buff);

    while (task_flag) {
        esp_get_feed_data(true, i2s_buff, audio_chunksize * sizeof(int16_t) * feed_channel);
        afe_handle->feed(afe_data, i2s_buff);
    }
    free(i2s_buff);
    vTaskDelete(NULL);
}
```

### 4. detect 任务：fetch 结果并响应唤醒

```c
void detect_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    printf("------------detect start------------\n");

    // 调节唤醒阈值（model_index 仅 1 或 2，范围 0.4~0.9999）
    afe_handle->set_wakenet_threshold(afe_data, 1, 0.6);
    afe_handle->set_wakenet_threshold(afe_data, 2, 0.6);
    // afe_handle->reset_wakenet_threshold(afe_data, 1);  // 恢复默认

    while (task_flag) {
        afe_fetch_result_t *res = afe_handle->fetch(afe_data);
        if (!res || res->ret_value == ESP_FAIL) {
            printf("fetch error!\n");
            break;
        }

        if (res->wakeup_state == WAKENET_DETECTED) {
            printf("wakeword detected\n");
            printf("model index:%d, word index:%d\n",
                   res->wakenet_model_index, res->wake_word_index);
            printf("-----------LISTENING-----------\n");
        }
    }
    vTaskDelete(NULL);
}
```

### 5. 加载多个唤醒词模型

在 menuconfig 中：
```
ESP Speech Recognition → Select wake words → Hi,Lexin (wn9_hilexin)
                                    → Load Multiple Wake Words
ESP Speech Recognition → Load Multiple Wake Words → Hi,Lexin (wn9_hilexin)
                                                  → Hi,ESP (wn9_hiesp)
```
AFE 最多同时运行两个 WakeNet 模型，`wakenet_model_index` 区分是哪个模型触发。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 唤醒完全无反应 | Kconfig 未勾选 WakeNet，flash 无模型 | menuconfig 勾选 `SR_WN_WN9_*`，重新编译 |
| `esp_srmodel_init` 返回 NULL | partitions.csv 无 `model` spiffs 分区 | 添加 `model, data, spiffs, , 5168K` |
| `assert(nch == feed_channel)` 失败 | 板子 Kconfig 与实际通道不符 | `idf.py menuconfig` 重选 Audio board |
| 唤醒灵敏度太低 | 默认阈值偏高 | `set_wakenet_threshold(afe_data, 1, 0.45)` 调低 |
| 误触发太多 | 阈值偏低 | `set_wakenet_threshold(afe_data, 1, 0.8)` 调高 |
| fetch 偶发 NULL | feed 任务未及时喂数据 | 提高 feed 任务优先级或检查 PSRAM |

## 参考

- `examples/wake_word_detection/afe/main/main.c` — 本 recipe 的真实来源
- `examples/wake_word_detection/afe/README.md` — 双模型加载与阈值说明
- `espressif-repos/esp-sr/include/esp32s3/esp_afe_sr_iface.h` — `esp_afe_sr_iface_t` 接口定义
- `espressif-repos/esp-sr/include/esp32s3/esp_wn_iface.h` — `wakenet_state_t`、`det_mode_t`
