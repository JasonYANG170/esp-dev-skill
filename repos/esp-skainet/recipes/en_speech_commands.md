# 英文命令词识别

> **适用摘要**: 在 ESP32-S3 上实现"唤醒 → 英文命令词识别"。结构与中文版一致，区别仅在 MultiNet 模型过滤关键字（`ESP_MN_ENGLISH`）与 Kconfig 模型符号。注意：英文命令仅支持 ESP32-S3 系列（不支持 ESP32）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-skainet/resources/`, source/examples in `repos/esp-skainet/`, and this recipe path `repos/esp-skainet/recipes/en_speech_commands.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "English speech commands recognition"
- "英文命令词识别"
- "MultiNet7 english"
- "turn on the light"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/en_speech_commands_recognition/` |
| 依赖组件 | `espressif/esp-sr` (^2.0.0) |
| Kconfig | `SR_WN_WN9_*` + `SR_MN_EN_MULTINET7_QUANT`（mn7_en） |
| 目标芯片 | 仅 ESP32-S3（示例中 `#if CONFIG_IDF_TARGET_ESP32` 直接 return） |
| 额外硬件 | 一只喇叭 |

## 分步说明

### 1. app_main：含 ESP32 保护

```c
srmodel_list_t *models = NULL;
static const esp_afe_sr_iface_t *afe_handle = NULL;
int wakeup_flag = 0;
static volatile int task_flag = 0;

void app_main()
{
    models = esp_srmodel_init("model");
    ESP_ERROR_CHECK(esp_board_init(16000, 2, 16));   // 注意：英文示例默认 2 通道

#if CONFIG_IDF_TARGET_ESP32
    printf("This demo only support ESP32S3\n");
    return;
#else
    afe_config_t *afe_config = afe_config_init(esp_get_input_format(), models,
                                               AFE_TYPE_SR, AFE_MODE_LOW_COST);
    afe_handle = esp_afe_handle_from_config(afe_config);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
    afe_config_free(afe_config);

    task_flag = 1;
    xTaskCreatePinnedToCore(&detect_Task, "detect", 8 * 1024, (void *)afe_data, 5, NULL, 1);
    xTaskCreatePinnedToCore(&feed_Task,  "feed",   8 * 1024, (void *)afe_data, 5, NULL, 0);
#endif
}
```

### 2. detect 任务：过滤英文 MultiNet

```c
#include "esp_mn_iface.h"
#include "esp_mn_models.h"            // ESP_MN_PREFIX, ESP_MN_ENGLISH
#include "esp_process_sdkconfig.h"

void detect_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    int afe_chunksize = afe_handle->get_fetch_chunksize(afe_data);

    // 关键：用 ESP_MN_ENGLISH ("en") 过滤英文 MultiNet
    char *mn_name = esp_srmodel_filter(models, ESP_MN_PREFIX, ESP_MN_ENGLISH);
    printf("multinet:%s\n", mn_name);

    esp_mn_iface_t *multinet = esp_mn_handle_from_name(mn_name);
    model_iface_data_t *model_data = multinet->create(mn_name, 6000);
    esp_mn_commands_update_from_sdkconfig(multinet, model_data);

    int mu_chunksize = multinet->get_samp_chunksize(model_data);
    assert(mu_chunksize == afe_chunksize);
    multinet->print_active_speech_commands(model_data);
    printf("------------detect start------------\n");

    while (task_flag) {
        afe_fetch_result_t *res = afe_handle->fetch(afe_data);
        if (!res || res->ret_value == ESP_FAIL) {
            printf("fetch error!\n");
            break;
        }

        if (res->wakeup_state == WAKENET_DETECTED) {
            printf("WAKEWORD DETECTED\n");
            multinet->clean(model_data);
        }
        if (res->raw_data_channels == 1 && res->wakeup_state == WAKENET_DETECTED) {
            wakeup_flag = 1;
        } else if (res->raw_data_channels > 1 && res->wakeup_state == WAKENET_CHANNEL_VERIFIED) {
            printf("AFE_FETCH_CHANNEL_VERIFIED, channel index: %d\n", res->trigger_channel_id);
            wakeup_flag = 1;
        }

        if (wakeup_flag == 1) {
            esp_mn_state_t mn_state = multinet->detect(model_data, res->data);
            if (mn_state == ESP_MN_STATE_DETECTING) continue;

            if (mn_state == ESP_MN_STATE_DETECTED) {
                esp_mn_results_t *mn_result = multinet->get_results(model_data);
                for (int i = 0; i < mn_result->num; i++) {
                    printf("TOP %d, command_id: %d, phrase_id: %d, string: %s, prob: %f\n",
                           i + 1, mn_result->command_id[i], mn_result->phrase_id[i],
                           mn_result->string, mn_result->prob[i]);
                }
            }
            if (mn_state == ESP_MN_STATE_TIMEOUT) {
                esp_mn_results_t *mn_result = multinet->get_results(model_data);
                printf("timeout, string:%s\n", mn_result->string);
                afe_handle->enable_wakenet(afe_data);
                wakeup_flag = 0;
                continue;
            }
        }
    }
    multinet->destroy(model_data);
    vTaskDelete(NULL);
}
```

### 3. 动态添加英文命令（mn6/mn7 推荐）

`esp_mn_commands_update_from_sdkconfig` 导入的是 Kconfig 默认表；运行时自定义见：

```c
// 注意：必须先 create multinet handle 再 add
esp_mn_commands_clear();
esp_mn_commands_add(1, "turn on the light");
esp_mn_commands_add(2, "turn off the light");
esp_mn_commands_update();                       // 刷新语言模型
multinet->print_active_speech_commands(model_data);
```

> 命令字符串用空格分隔音节/单词，例如 `"turn on the light"`，不要写成 `"turn on the light."`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| ESP32 上打印 "only support ESP32S3" | 英文 MultiNet 不支持 ESP32 | 换 ESP32-S3 板子 |
| `mn_name` 为 NULL | 未勾选 `SR_MN_EN_MULTINET7_QUANT` | menuconfig 选英文 MultiNet |
| 识别率低 | 命令字符串拼写/音节不规范 | 用空格分隔单词，参考 README 示例 |
| 多通道板卡在唤醒 | 没等 `WAKENET_CHANNEL_VERIFIED` | 加上 raw_data_channels>1 分支 |
| 喇叭无声 | 未实现 `speech_commands_action` 回调 | 参考 en example 的 speech_commands_action.c |

## 参考

- `examples/en_speech_commands_recognition/main/main.c` — 本 recipe 真实来源
- `examples/en_speech_commands_recognition/README.md` — 修改命令词
- `espressif-repos/esp-sr/include/esp32s3/esp_mn_models.h` — `ESP_MN_PREFIX`/`ESP_MN_ENGLISH`/`esp_mn_handle_from_name`
- `espressif-repos/esp-sr/Kconfig.projbuild` — `SR_MN_EN_MULTINET7_QUANT`
