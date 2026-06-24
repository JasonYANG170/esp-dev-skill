# 中文 TTS 语音合成

> **适用摘要**: 用 esp-tts 把中文文本合成为 16k/16bit PCM 并通过 `esp_audio_play` 播放，支持从 UART 接收文本实时合成。发音集从 `voice_data` 分区 mmap 加载。

## 触发意图

- "Chinese TTS"
- "中文语音合成"
- "esp_tts"
- "文本转语音"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/chinese_tts/` |
| 依赖组件 | `esp-sr`（含 esp-tts 子模块）、`hardware_driver` |
| 分区 | `voice_data`（发音集）分区，见工程 partitions.csv |
| 头文件 | `esp_tts.h`、`esp_tts_player.h`、`esp_tts_voice_xiaole.h`、`esp_tts_voice_template.h` |
| 硬件 | 喇叭；不支持 ESP32-S3-EYE（示例直接 return） |

## 分步说明

### 1. 引入头文件

```c
#include "esp_tts.h"
#include "esp_tts_voice_xiaole.h"     // 内置发音模板（占位）
#include "esp_tts_voice_template.h"
#include "esp_tts_player.h"
#include "esp_board_init.h"
#include "esp_partition.h"
#include "esp_idf_version.h"
```

### 2. app_main：板级 + 加载发音集

```c
int app_main()
{
#if defined CONFIG_ESP32_S3_EYE_BOARD
    printf("Not Support esp32-s3-eye board\n");
    return 0;
#endif

    ESP_ERROR_CHECK(esp_board_init(16000, 1, 16));

    // 从 voice_data 分区 mmap 出真实发音数据
    const esp_partition_t *part = esp_partition_find_first(
        ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_ANY, "voice_data");
    if (part == NULL) {
        printf("Couldn't find voice data partition!\n");
        return 0;
    }
    printf("voice_data partition size:%d\n", part->size);

    const void *voicedata;
#if ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5, 0, 0)
    esp_partition_mmap_handle_t mmap;
    esp_err_t err = esp_partition_mmap(part, 0, part->size,
                                       ESP_PARTITION_MMAP_DATA, &voicedata, &mmap);
#else
    spi_flash_mmap_handle_t mmap;
    esp_err_t err = spi_flash_mmap(part, 0, part->size,
                                   SPI_FLASH_MMAP_DATA, &voicedata, &mmap);
#endif
    if (err != ESP_OK) {
        printf("Couldn't map voice data partition!\n");
        return 0;
    }

    // 用模板 + 真实发音数据初始化 voice set
    esp_tts_voice_t *voice = esp_tts_voice_set_init(&esp_tts_voice_template, (int16_t *)voicedata);
    esp_tts_handle_t *tts_handle = esp_tts_create(voice);
```

### 3. 合成并播放提示文本

```c
    char *prompt1 = "欢迎使用乐鑫语音合成";
    printf("%s\n", prompt1);
    if (esp_tts_parse_chinese(tts_handle, prompt1)) {
        int len[1] = {0};
        do {
            short *pcm_data = esp_tts_stream_play(tts_handle, len, 3);
            esp_audio_play(pcm_data, len[0] * 2, portMAX_DELAY);   // 16bit → ×2 字节
        } while (len[0] > 0);
    }
    esp_tts_stream_reset(tts_handle);
```

### 4. 从 UART 持续合成（示例核心交互）

```c
    // tts_urat.c 里有一个 uartTask 把串口字符喂进 ringbuf
    xTaskCreatePinnedToCore(&uartTask, "urat", 6 * 1024, NULL, 5, NULL, 0);

    char data[URAT_BUF_LEN + 1];
    char in;
    int data_len = 0;
    printf("\n请输入短语:");

    while (1) {
        rb_read(urat_rb, &in, 1, portMAX_DELAY);   // 阻塞读一个字符
        if (in == '\n') {
            data[data_len] = '\0';
            printf("tts input:%s\n", data);
            if (esp_tts_parse_chinese(tts_handle, data)) {
                int len[1] = {0};
                do {
                    short *pcm_data = esp_tts_stream_play(tts_handle, len, 3);
                    esp_audio_play(pcm_data, len[0] * 2, portMAX_DELAY);
                } while (len[0] > 0);
            }
            esp_tts_stream_reset(tts_handle);
            printf("\n请输入短语:");
            data_len = 0;
        } else if (data_len < URAT_BUF_LEN) {
            data[data_len++] = in;
        } else {
            printf("ERROR: out of range\n");
            data_len = 0;
        }
    }
    return 0;
}
```

### 5. 可选：WAV 落盘评估

```c
// 需 #define SDCARD_OUTPUT_ENABLE 并链接 wav_encoder
void *wav_encoder = wav_encoder_open("/sdcard/prompt.wav", 16000, 16, 1);
wav_encoder_run(wav_encoder, pcm_data, len[0] * 2);
wav_encoder_close(wav_encoder);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `voice_data` 分区找不到 | partitions.csv 缺该分区 | 加 `voice_data, data, ..., 大小` 分区 |
| 发音全乱/无声 | 直接用了 `&esp_tts_voice_template` 而未填 voicedata | 必须 `esp_tts_voice_set_init(template, voicedata)` |
| ESP32-S3-EYE 报错 | 示例显式不支持该板 | 换其他 S3 板 |
| 合成卡住 | `esp_tts_parse_chinese` 对非中文字符返回 false | 先判返回值再播 |
| IDF v4/v5 兼容 | mmap API 不同 | 用 `ESP_IDF_VERSION` 宏分支（见步骤 2） |
| 音量小 | `esp_audio_play` 默认音量 | `esp_audio_set_play_vol(volume)` 调高 |

## 参考

- `examples/chinese_tts/main/main.c` — 本 recipe 真实来源
- `examples/chinese_tts/main/tts_urat.c` — UART ringbuf 任务
- `espressif-repos/esp-sr/esp-tts/esp_tts_chinese/include/esp_tts.h` — `esp_tts_create/parse_chinese`
- `espressif-repos/esp-sr/esp-tts/esp_tts_chinese/include/esp_tts_player.h` — `esp_tts_stream_play/stream_reset`
- `espressif-repos/esp-skainet/components/hardware_driver/include/esp_board_init.h` — `esp_audio_play`
