# 中文语音合成（esp-tts）

> **适用摘要**: 使用 ESP-SR 内置的中文 TTS 模块把 UTF-8 中文文本流式合成为 16k/16bit 语音。仅支持中文，需 `voice_data` 分区存放声音集。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-sr/resources/`, source/examples in `repos/esp-sr/`, and this recipe path `repos/esp-sr/recipes/chinese_tts.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "中文语音合成"
- "TTS 播报"
- "esp_tts 怎么用"
- "支付宝收款播报"
- "声音集 voice_data 分区"

## 前置条件

| 条件 | 要求 |
|---|---|
| 语言 | **仅中文**（UTF-8 编码） |
| 分区 | `partitions.csv` 需含 `voice_data` 分区（FAT，存声音集） |
| I2S | 已配置 DAC/I2S 输出（16k/16bit/单声道） |
| 参考 | `test_apps/esp-tts/main/test_chinese_tts.cpp`、`docs/en/speech_synthesis/readme.rst` |

## 分步说明

### 1. 增加 voice_data 分区

```csv
# Name,      Type, SubType, Offset, Size
nvs,         data, nvs
phy_init,    data, phy
model,       data,                 ,       , 6000K
voice_data,  data, fat,            ,       1500K
```

将仓库 `esp-tts/esp_tts_chinese/` 提供的声音集二进制烧入 `voice_data`。

### 2. mmap 声音集并创建 TTS handle

```c
#include "esp_tts.h"
#include "esp_tts_voice_xiaole.h"     // 声音集索引（xiaole）
#include "esp_tts_voice_template.h"   // 模板
#include "esp_partition.h"
#include "esp_idf_version.h"

const esp_partition_t *part = esp_partition_find_first(
    ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_ANY, "voice_data");
assert(part != NULL);

const void *voicedata;
#if ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5, 0, 0)
    esp_partition_mmap_handle_t mmap_handle;
    ESP_ERROR_CHECK(esp_partition_mmap(part, 0, part->size,
        ESP_PARTITION_MMAP_DATA, &voicedata, &mmap_handle));
#else
    spi_flash_mmap_handle_t mmap_handle;
    ESP_ERROR_CHECK(esp_partition_mmap(part, 0, part->size,
        SPI_FLASH_MMAP_DATA, &voicedata, &mmap_handle));
#endif

// 用模板 + 声音集二进制初始化声音集
esp_tts_voice_t *voice = esp_tts_voice_set_init(&esp_tts_voice_template, (int16_t *)voicedata);
esp_tts_handle_t *tts = (esp_tts_handle_t *)esp_tts_create(voice);
```

> 仓库默认提供 `xiaole` 声音（`esp_tts_voice_xiaole.h` / `libvoice_set_xiaole.a`）。`esp_tts_voice_template` 是模板结构，声音数据来自 voice_data 分区。

### 3. 解析中文并流式合成

```c
char *text = "欢迎使用乐鑫语音合成";   // UTF-8 中文

if (esp_tts_parse_chinese(tts, text)) {     // 解析为拼音序列
    int len[1] = {0};
    do {
        // 流式合成：每次返回一段 short PCM，len[0] 为样本数
        short *data = esp_tts_stream_play(tts, len, 4);   // 参数 4 为语速档
        // 写到 I2S（单声道 16k/16bit），len[0]*2 字节
        i2s_audio_play(data, len[0] * 2, portMAX_DELAY);
    } while (len[0] > 0);
    i2s_zero_dma_buffer(0);
}
```

`esp_tts_stream_play` 第三个参数控制语速（如 1=正常，2/4 加速）。可调输出语速以适配不同播报场景。

### 4. 清理

```c
esp_tts_voice_set_free(voice);
esp_tts_destroy(tts);
#if ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5, 0, 0)
    esp_partition_munmap(mmap_handle);
#else
    spi_flash_munmap(mmap_handle);
#endif
```

### 5. 示例语音（仓库自带）

| 文件 | 声音 | 语速 | 文本 |
|---|---|---|---|
| `esp-tts/samples/xiaoxin_speed1.wav` | xiaoxin | 1 | 欢迎使用乐鑫语音合成，支付宝收款 72.1 元… |
| `esp-tts/samples/S2_xiaole_speed2.wav` | xiaole | 2 | 支付宝收款 1111.11 元 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `Couldn't find voice data partition` | partitions.csv 没有 voice_data | 加 `voice_data, data, fat, , <size>` 并烧入声音集 |
| 解析返回 0 | 文本非中文或非 UTF-8 | 仅支持中文 UTF-8；英文/数字支持有限（金额可） |
| 播报卡顿 | I2S DMA buffer 太小 | 增大 I2S DMA buffer，按 stream 段长配 |
| 多音字读错 | 内置词典未覆盖 | TTS 支持多音字但非完美；关键词条可改写 |
| 语速太快/慢 | `esp_tts_stream_play` 第 3 参数 | 调整（1/2/4） |
| v4/v5 mmap API 不同 | IDF 版本差异 | 用 `ESP_IDF_VERSION` 宏区分（见步骤 2） |

## 参考

- `test_apps/esp-tts/main/test_chinese_tts.cpp` — create/parse/stream_play/destroy 完整用例
- `esp-tts/esp_tts_chinese/include/esp_tts.h` — TTS API
- `docs/en/speech_synthesis/readme.rst` — TTS 流程与示例 wav
- `esp-tts/samples/*.wav` — 示例合成音频
- esp-skainet `examples/chinese_tts`（产品级示例，仓库外）
