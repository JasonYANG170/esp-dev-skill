# AGENTS.md — Supplementary Agent Guide

> 核心规则、状态机、陷阱、recipe 索引与执行流程均在 `SKILL.md`。
> 本文件**只**补充 `SKILL.md` 未覆盖的工程约定与工具链指引，不重复内容。

## Project Context

- **语言**: C（基于 FreeRTOS）
- **目标**: Espressif ESP32 / ESP32-S3（推荐）/ ESP32-P4 / ESP32-S31
- **工具链 / 构建**: ESP-IDF v4.4 或 v5.x（`idf.py`）；模型组件 `espressif/esp-sr` (^2.0.0) 由 `idf_component.yml` 拉取
- **核心依赖组件**: `esp-sr`（提供 WakeNet/MultiNet/AFE/VAD/NS/DOA/TTS）、`hardware_driver`（板级 I2S 抽象）、`perf_tester`（性能测试控制台）、`player`、`sr_ringbuf`

## File Naming

- 应用主源：`main/main.c`（每个 example 均如此）
- 命令词动作回调：`main/speech_commands_action.c` + `main/include/speech_commands_action.h`（导出 `speech_commands_action(int command_id)`、`wake_up_action()`）
- 提示音头：`main/include/m_*.h`（DAC 播放的 PCM 数组）、`wake_up_prompt_tone.h`
- 板级实现：`components/hardware_driver/boards/<board>/bsp_board.c` + `components/hardware_driver/boards/include/<board>_board.h`
- 分区表：工程根 `partitions.csv`；ESP32 旧板另有 `partitions_esp32.csv`
- 配置默认：工程根 `sdkconfig.defaults`（公共）+ `sdkconfig.defaults.<target>`（按芯片）

## Include Pattern

```c
// WakeNet
#include "esp_wn_iface.h"
#include "esp_wn_models.h"          // ESP_WN_PREFIX, esp_wn_handle_from_name

// Audio Front-End (AFE)
#include "esp_afe_sr_iface.h"       // esp_afe_sr_iface_t, afe_fetch_result_t
#include "esp_afe_sr_models.h"
#include "esp_afe_config.h"         // afe_config_t, afe_config_init, afe_type_t, afe_mode_t

// MultiNet
#include "esp_mn_iface.h"           // esp_mn_iface_t, esp_mn_state_t, esp_mn_results_t
#include "esp_mn_models.h"          // ESP_MN_PREFIX, ESP_MN_CHINESE/ENGLISH, esp_mn_handle_from_name
#include "esp_process_sdkconfig.h"  // esp_mn_commands_update_from_sdkconfig

// 模型加载
#include "model_path.h"             // srmodel_list_t, esp_srmodel_init, esp_srmodel_filter

// 板级 / 工具
#include "esp_board_init.h"         // esp_board_init, esp_get_feed_data, esp_audio_play, esp_sdcard_init
#include "ringbuf.h"                // rb_create / rb_write / rb_read（PCM 落盘调试）

// VAD / NS / DOA / TTS（按需）
#include "esp_vadn_models.h"
#include "esp_nsn_models.h"
#include "esp_doa.h"
#include "esp_tts.h"
#include "esp_tts_player.h"
#include "esp_tts_voice_xiaole.h"
#include "esp_tts_voice_template.h"
```

## Standard Project Structure

```
MyVoiceProject/
├── CMakeLists.txt                  # 引用 idf_component.yml 的依赖
├── partitions.csv                  # factory(app) + model(spiffs) 分区
├── sdkconfig.defaults              # 公共（分区表等）
├── sdkconfig.defaults.esp32s3      # PSRAM/flash/缓存 + SR 模型选择
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml           # dependencies: espressif/esp-sr ^2.0.0
│   ├── main.c                      # app_main: 板级init → esp_srmodel_init → afe_config_init → 双任务
│   ├── speech_commands_action.c    # 命令命中回调（播放提示音/控灯）
│   └── include/
│       └── speech_commands_action.h
└── (依赖组件由 idf.py 自动从 component registry 拉到 managed_components/)
```

## partitions.csv 标准模板（ESP32-S3，16 MB Flash）

```
# Name,  Type, SubType, Offset,  Size
factory, app,  factory, 0x010000, 2048k
model,  data, spiffs,         , 5168K
```

> `model`（SPIFFS）的 label 必须与代码里 `esp_srmodel_init("model")` 的参数一致。模型越大分区越大；TTS 还需额外的 `voice_data` 分区。

## Canonical app_main Pattern

```c
void app_main(void)
{
    // 1. 板级初始化：sample_rate=16000, channel, bits=16
    ESP_ERROR_CHECK(esp_board_init(16000, 1, 16));

    // 2. 加载模型（"model" = partitions.csv 的 spiffs label）
    srmodel_list_t *models = esp_srmodel_init("model");

    // 3. 生成 AFE 默认配置并取 handle / data
    afe_config_t *afe_config = afe_config_init(esp_get_input_format(),
                                               models,
                                               AFE_TYPE_SR,
                                               AFE_MODE_LOW_COST);
    // 可在此微调：afe_config->vad_mode = VAD_MODE_1; 等
    const esp_afe_sr_iface_t *afe_handle = esp_afe_handle_from_config(afe_config);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(afe_config);
    afe_config_free(afe_config);

    // 4. 启动 feed / detect 双任务（分别绑定核）
    static volatile int task_flag = 1;
    xTaskCreatePinnedToCore(&feed_Task,  "feed",   8 * 1024, (void *)afe_data, 5, NULL, 0);
    xTaskCreatePinnedToCore(&detect_Task,"detect", 8 * 1024, (void *)afe_data, 5, NULL, 1);
}
```

## feed / detect 任务模板

```c
void feed_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    int audio_chunksize = afe_handle->get_feed_chunksize(afe_data);
    int feed_channel = esp_get_feed_channel();
    int16_t *i2s_buff = malloc(audio_chunksize * sizeof(int16_t) * feed_channel);

    while (task_flag) {
        esp_get_feed_data(true, i2s_buff, audio_chunksize * sizeof(int16_t) * feed_channel);
        afe_handle->feed(afe_data, i2s_buff);
    }
    free(i2s_buff);
    vTaskDelete(NULL);
}

void detect_Task(void *arg)
{
    esp_afe_sr_data_t *afe_data = (esp_afe_sr_data_t *)arg;
    while (task_flag) {
        afe_fetch_result_t *res = afe_handle->fetch(afe_data);
        if (!res || res->ret_value == ESP_FAIL) {
            printf("fetch error!\n");
            break;
        }
        // 处理 res->wakeup_state / res->data / res->vad_state ...
    }
    vTaskDelete(NULL);
}
```

## Build Workflow

1. `idf.py set-target esp32s3`
2. 复制目标芯片默认配置：`cp sdkconfig.defaults.esp32s3 sdkconfig`（可选，menuconfig 也会读 defaults）
3. `idf.py menuconfig`：
   - `Audio Media HAL → Audio hardware board` 选板
   - `ESP Speech Recognition → Select wake words` 选 WakeNet
   - `ESP Speech Recognition → Chinese/English Speech Commands Model` 选 MultiNet
   - `Component config → ESP32S3 SPIRAM` 确认 octal/80M
4. `idf.py flash monitor`（退出监控 `Ctrl-]`）
5. 调试：串口看 `print_pipeline()`、`print_active_speech_commands()`、各 `printf`

## Codegen Checklist

- [ ] `esp_board_init(16000, channel, 16)` 已调用且 channel 与板子一致
- [ ] `esp_srmodel_init("model")` 的 label 与 partitions.csv 一致
- [ ] Kconfig 已勾选目标芯片支持的 WakeNet (`SR_WN_*`) 与 MultiNet (`SR_MN_*`)
- [ ] `afe_config_init` 的 `input_format` 用 `esp_get_input_format()`，不要硬编码
- [ ] feed 帧大小 = `get_feed_chunksize() × feed_channel × sizeof(int16_t)`
- [ ] `assert(nch == feed_channel)` 放在 feed 任务开头
- [ ] detect 任务对 fetch 结果判空 + 检查 `ret_value`
- [ ] 多通道场景等到 `WAKENET_CHANNEL_VERIFIED` 再进命令模式
- [ ] MultiNet 命令 `add` 之后调了 `esp_mn_commands_update()`（或 `esp_mn_commands_update_from_sdkconfig`）
- [ ] `ESP_MN_STATE_TIMEOUT` 分支里重新 `enable_wakenet` 并复位 wakeup_flag
- [ ] `set_wakenet_threshold` 的 model_index ∈ {1, 2}，threshold ∈ [0.4, 0.9999]
- [ ] sdkconfig.defaults.<target> 含 PSRAM octal / flash 16MB QIO / 240MHz / cache 配置
- [ ] partitions.csv 有 `model,data,spiffs` 分区（TTS 另需 `voice_data`）

## Do Not Modify

- `espressif-repos/esp-skainet/components/`、`espressif-repos/esp-sr/` —— 仓库源码，仅作为参考依据，不要在生成代码里改动其头文件或实现
- `SKILL.md` frontmatter —— Skill 元数据
- 任何 `managed_components/` 下由 `idf.py` 自动拉取的组件
