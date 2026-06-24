# 自定义音频板移植

> **适用摘要**: 当 PCB 不在 `hardware_driver` 已支持板列表时，新建 `esp_custom_board.h`（引脚表）与 `bsp_board.c`（I2S/codec/feed 实现），让 `esp_board_init` / `esp_get_input_format` / `esp_get_feed_data` 跑在新硬件上。

## 触发意图

- "我的板子不在支持列表"
- "移植 bsp_board.c"
- "自定义音频板"
- "esp_custom_board"
- "CONFIG_ESP_CUSTOM_BOARD"
- "新 codec 移植"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考板实现 | `components/hardware_driver/boards/esp32s3-korvo-1/bsp_board.c`（ES7210+ES8311，双 I2S） |
| 参考引脚表 | `components/hardware_driver/boards/include/esp32_s3_korvo_1_v4_board.h` |
| 公共 API 头 | `components/hardware_driver/boards/include/bsp_board.h`、`include/esp_board_init.h` |
| 选板 Kconfig | `components/hardware_driver/Kconfig.projbuild`（`choice AUDIO_BOARD`） |
| 硬件信息 | codec 型号 + I2C 地址、I2S 引脚、麦数、是否有回采（AEC）通道 |

## 分步说明

### 1. 理解调用链（为什么要实现这几个函数）

`esp_board_init.c` 只是一层转发（`esp_board_init → bsp_board_init`，`esp_get_feed_data → bsp_get_feed_data`，`esp_get_input_format → bsp_get_input_format`）。AFE 与上层只认 `esp_*` 这组公共 API，板子差异全部封装在 `bsp_board.c` 里：

```
esp_board_init()      ──►  bsp_board_init(sample_rate, channel_format, bits_per_chan)
esp_get_feed_data()   ──►  bsp_get_feed_data(is_get_raw_channel, buffer, buffer_len)
esp_get_feed_channel()──►  bsp_get_feed_channel()
esp_get_input_format()──►  bsp_get_input_format()   ← 决定 AFE 通道编排
```

`bsp_board.h` 在编译期按 Kconfig 选择对应板子的引脚头文件，`CONFIG_ESP_CUSTOM_BOARD` 分支对应 `esp_custom_board.h`：

```c
// components/hardware_driver/boards/include/bsp_board.h
#if CONFIG_ESP32_S3_KORVO_1_V4_0_BOARD
    #include "esp32_s3_korvo_1_v4_board.h"
#elif CONFIG_ESP32_S3_BOX_BOARD
    #include "esp32_s3_box_board.h"
// ... 其他 7 块板子
#elif CONFIG_ESP_CUSTOM_BOARD
    #include "esp_custom_board.h"     // ← 你的板子在这里
#endif
```

> 必须实现的 5 个函数（声明于 `bsp_board.h`）：`bsp_board_init`、`bsp_get_feed_data`、`bsp_get_feed_channel`、`bsp_get_input_format`，以及（如需 SD/播放）`bsp_sdcard_init`、`bsp_audio_play`、`bsp_audio_set_play_vol`。

### 2. 编写引脚表头文件 esp_custom_board.h

复制 `esp32_s3_korvo_1_v4_board.h`，改成你的原理图引脚。以 ESP32-S3-Korvo-1 V4 为例的真实引脚定义：

```c
// components/hardware_driver/boards/include/esp_custom_board.h
#pragma once
#include "driver/gpio.h"
#include "esp_idf_version.h"
#include "esp_codec_dev.h"
#include "esp_codec_dev_defaults.h"

// ---- I2C（控制 codec） ----
#define FUNC_I2C_EN     (1)
#define I2C_NUM         (0)
#define I2C_CLK         (600000)
#define GPIO_I2C_SCL    (GPIO_NUM_2)     // 改成你的 SCL
#define GPIO_I2C_SDA    (GPIO_NUM_1)     // 改成你的 SDA

// ---- SDMMC（可选） ----
#define FUNC_SDMMC_EN   (1)
#define SDMMC_BUS_WIDTH (1)
#define GPIO_SDMMC_CLK  (GPIO_NUM_18)
#define GPIO_SDMMC_CMD  (GPIO_NUM_17)
#define GPIO_SDMMC_D0   (GPIO_NUM_16)

// ---- 录音 I2S（接 ADC，如 ES7210） ----
#define FUNC_I2S_EN         (1)
#define GPIO_I2S_LRCK       (GPIO_NUM_9)
#define GPIO_I2S_MCLK       (GPIO_NUM_20)
#define GPIO_I2S_SCLK       (GPIO_NUM_10)
#define GPIO_I2S_SDIN       (GPIO_NUM_11)
#define GPIO_I2S_DOUT       (GPIO_NUM_NC)

// ---- 播放 I2S（接 DAC，如 ES8311） ----
#define FUNC_I2S0_EN         (1)
#define GPIO_I2S0_LRCK       (GPIO_NUM_41)
#define GPIO_I2S0_MCLK       (GPIO_NUM_42)
#define GPIO_I2S0_SCLK       (GPIO_NUM_40)
#define GPIO_I2S0_SDIN       (GPIO_NUM_NC)
#define GPIO_I2S0_DOUT       (GPIO_NUM_39)

// ---- 增益 / 功放 ----
#define RECORD_VOLUME   (30.0)
#define PLAYER_VOLUME   (50)
#define FUNC_PWR_CTRL       (1)
#define GPIO_PWR_CTRL       (GPIO_NUM_38)
#define GPIO_PWR_ON_LEVEL   (1)
```

> 这些宏被 `I2S_CONFIG_DEFAULT` / `I2S0_CONFIG_DEFAULT`（同文件下半部分）组装成 `i2s_std_config_t`，也会被 `bsp_board.c` 直接引用。改引脚只需改宏，不动 `bsp_board.c` 的 I2S 配置代码。

### 3. 实现 bsp_board.c（复制最接近的板子再改）

最省力的路径：整份复制 `boards/esp32s3-korvo-1/bsp_board.c`（ES7210 ADC + ES8311 DAC，双 I2S），只在 codec 型号/地址/通道编排不同时改。

`bsp_board_init` 的真实实现（korvo-1）：

```c
// boards/esp32s3-korvo-1/bsp_board.c
esp_err_t bsp_board_init(uint32_t sample_rate, int channel_format, int bits_per_chan)
{
    bsp_i2c_init(I2C_NUM, I2C_CLK);                              // I2C 总线（给 codec 控制口）
    bsp_i2s_init(I2S_NUM_0, sample_rate, channel_format, bits_per_chan);  // 播放 I2S
    bsp_i2s_init(I2S_NUM_1, 16000, 2, 32);                       // 录音 I2S（ES7210 固定 32bit）
    bsp_codec_init(16000, sample_rate, channel_format, bits_per_chan);    // codec 初始化
    return ESP_OK;
}
```

codec 初始化（ES7210 ADC 示例，4 通道麦克风）：

```c
esp_err_t bsp_codec_adc_init(int sample_rate)
{
    audio_codec_i2s_cfg_t i2s_cfg = {
        .port = I2S_NUM_1,
        .rx_handle = rx_handle,
        .tx_handle = NULL,
    };
    record_data_if = audio_codec_new_i2s_data(&i2s_cfg);

    audio_codec_i2c_cfg_t i2c_cfg = {.addr = ES7210_CODEC_DEFAULT_ADDR};  // 改成你的 ADC 地址
    record_ctrl_if = audio_codec_new_i2c_ctrl(&i2c_cfg);

    es7210_codec_cfg_t es7210_cfg = {
        .ctrl_if = record_ctrl_if,
        .mic_selected = ES7120_SEL_MIC1 | ES7120_SEL_MIC2 | ES7120_SEL_MIC3 | ES7120_SEL_MIC4,
    };
    record_codec_if = es7210_codec_new(&es7210_cfg);   // 换 codec 就换这行

    esp_codec_dev_cfg_t dev_cfg = {
        .codec_if = record_codec_if,
        .data_if  = record_data_if,
        .dev_type = ESP_CODEC_DEV_TYPE_IN,
    };
    record_dev = esp_codec_dev_new(&dev_cfg);

    esp_codec_dev_sample_info_t fs = {
        .sample_rate = 16000, .channel = 2, .bits_per_sample = 32,
    };
    esp_codec_dev_open(record_dev, &fs);
    // 每通道增益：麦通道给 RECORD_VOLUME，回采通道给 0
    esp_codec_dev_set_in_channel_gain(record_dev, ESP_CODEC_DEV_MAKE_CHANNEL_MASK(0), RECORD_VOLUME);
    esp_codec_dev_set_in_channel_gain(record_dev, ESP_CODEC_DEV_MAKE_CHANNEL_MASK(1), RECORD_VOLUME);
    esp_codec_dev_set_in_channel_gain(record_dev, ESP_CODEC_DEV_MAKE_CHANNEL_MASK(2), 0.0);   // reference
    esp_codec_dev_set_in_channel_gain(record_dev, ESP_CODEC_DEV_MAKE_CHANNEL_MASK(3), RECORD_VOLUME);
    return ESP_OK;
}
```

### 4. bsp_get_feed_data：通道重排（input_format 不变量）

这是移植最容易出错的地方。`is_get_raw_channel=false` 时，必须把 codec 读出的原始通道重排成 AFE 期望的 `input_format` 顺序。korvo-1 的真实实现：codec 读出 4 通道 `[ref, mic1, mic2, mic3]`，AFE 要 `"RMNM"`：

```c
#define ADC_I2S_CHANNEL 4   // codec 实际读出的通道数

esp_err_t bsp_get_feed_data(bool is_get_raw_channel, int16_t *buffer, int buffer_len)
{
    esp_err_t ret = esp_codec_dev_read(record_dev, (void *)buffer, buffer_len);
    if (!is_get_raw_channel) {
        int audio_chunksize = buffer_len / (sizeof(int16_t) * ADC_I2S_CHANNEL);
        for (int i = 0; i < audio_chunksize; i++) {
            int16_t ref = buffer[4 * i + 0];            // 原始 ch0 = ref
            buffer[3 * i + 0] = buffer[4 * i + 1];      // mic1 → 输出 ch0
            buffer[3 * i + 1] = buffer[4 * i + 3];      // mic3 → 输出 ch1
            buffer[3 * i + 2] = ref;                    // ref  → 输出 ch2
        }
    }
    return ret;
}

int bsp_get_feed_channel(void) { return ADC_I2S_CHANNEL; }   // 返回原始通道数（4）

char* bsp_get_input_format(void) { return "RMNM"; }           // 决定 AFE 通道编排
```

> **三个值必须自洽**：`bsp_get_feed_channel()` 返回 codec 物理通道数；`bsp_get_input_format()` 字符串长度 = 重排后的输出通道数（喂给 AFE 的帧每样本通道数）；`feed_Task` 里 `buffer_len = chunksize × feed_channel × sizeof(int16_t)`（见 pitfalls #1）。重排逻辑把"物理通道"映射到"逻辑通道"。

### 5. input_format 字符串语义

| 字符 | 含义 | 举例 |
|---|---|---|
| `M` | 麦克风 | `"MR"` = 单麦 + 回采 |
| `R` | 回采参考（AEC 用） | `"MMR"` = 双麦 + 回采 |
| `N` | 未知/未用（占位） | `"MRNN"` = 单麦+回采+2 个空通道 |

各板的真实返回值（来自各 `bsp_board.c`）：

| 板 | `bsp_get_input_format()` |
|---|---|
| esp32s3-korvo-1 | `"RMNM"` |
| esp32-korvo | `"MRNN"` |
| esp32p4-function-ev | `"MR"` |
| esp32s3-eye | `"MN"`（单麦，无回采） |

### 6. 在 Kconfig 注册自定义板（两种方式）

**方式 A（推荐）：在 Kconfig.projbuild 加 choice 项**

```kconfig
# components/hardware_driver/Kconfig.projbuild
config ESP_CUSTOM_BOARD
    bool "My Custom Board"
    depends on IDF_TARGET_ESP32S3   # 改成你的目标
```

这样 `menuconfig` 的 `Audio Media HAL → Audio hardware board` 就会出现你的板子。

**方式 B：直接 sdkconfig 注入宏**

`bsp_board.h` 已经有 `#elif CONFIG_ESP_CUSTOM_BOARD` 分支，若不改 Kconfig，可在 `sdkconfig.defaults` 加：

```makefile
CONFIG_ESP_CUSTOM_BOARD=y
```

> 注意：现仓库 `Kconfig.projbuild` 的 `choice AUDIO_BOARD` 只列了 7 块预支持板，没有 `ESP_CUSTOM_BOARD` 项。所以方式 B 需确保 `CONFIG_ESP_CUSTOM_BOARD` 没被 choice 互斥掉——最稳妥还是用方式 A 把它加进 choice。

### 7. CMake / 目录放置

把你的板子源码放进 boards 目录，并在 `boards/CMakeLists.txt` 里按目标编译：

```
components/hardware_driver/boards/
├── include/
│   ├── bsp_board.h                 # 公共头（含你的 #elif 分支）
│   └── esp_custom_board.h          # ← 新建：你的引脚表
└── esp32s3-custom/                 # ← 新建：你的板子目录
    └── bsp_board.c                 # ← 新建：你的实现
```

### 8. 验证步骤

1. `idf.py set-target esp32s3 && idf.py menuconfig` 选到你的板子。
2. 烧录任意唤醒词 example，串口看 `afe_handle->print_pipeline()` 输出的流水线是否正确（`[input]->|AEC|->|WakeNet(...)|->[output]`）。
3. 确认 `assert(nch == feed_channel)` 不触发——`nch` 来自 `afe_handle->get_channel_num()`（由 `input_format` 长度决定），`feed_channel` 来自 `esp_get_feed_channel()`，两者必须一致或重排后一致。
4. 对着麦克风说唤醒词，确认能 `WAKENET_DETECTED`。
5. 用 `recipes/perf_benchmarking.md` 的 RAR 测试量化唤醒率，验证移植未引入精度回归。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `assert(nch == feed_channel)` 失败 | `input_format` 长度与 `feed_channel`/重排后通道数不一致 | 三者对齐：`input_format` 字符串长度 = 重排后输出通道数；`feed_channel` = 物理通道数 |
| 录音全是噪声/无声 | I2S 引脚或 codec 地址错 | 核对 `esp_custom_board.h` 的 `GPIO_I2S_*` 与 codec I2C 地址（如 `ES7210_CODEC_DEFAULT_ADDR`） |
| 唤醒率严重下降 | 通道重排顺序错（mic/ref 错位） | 用示波器或单通道录音逐路确认 `bsp_get_feed_data` 重排后的 `[0],[1],[2]` 分别是什么 |
| AEC 无效果 | 回采通道 `R` 没接或增益为 0 | 确认硬件有回采线路；`esp_codec_dev_set_in_channel_gain` 给 ref 通道非 0 增益 |
| 编译找不到 `esp_custom_board.h` | 头文件没放对位置 | 放进 `boards/include/`，确认 `bsp_board.h` 的 `#elif CONFIG_ESP_CUSTOM_BOARD` 分支能 include 到 |
| `CONFIG_ESP_CUSTOM_BOARD` 未定义 | Kconfig 没加 choice 项 | 方式 A 加进 `Kconfig.projbuild`，或方式 B 在 sdkconfig 强制 `=y` |
| 双麦板单麦能唤醒 | input_format 写成 `"MR"` 而非 `"MMR"` | 按真实麦数写：双麦+回采=`"MMR"`，korvo-1 四采三用=`"RMNM"` |

## 参考项目

- `components/hardware_driver/boards/esp32s3-korvo-1/bsp_board.c` — ES7210+ES8311 双 I2S 完整实现（`bsp_board_init`/`bsp_codec_adc_init`/`bsp_codec_dac_init`/`bsp_get_feed_data` 通道重排）
- `components/hardware_driver/boards/esp32s3-box/bsp_board.c` — ESP32-S3-BOX 双麦实现
- `components/hardware_driver/boards/esp32p4-function-ev/bsp_board.c` — ESP32-P4 板（`input_format="MR"`）
- `components/hardware_driver/boards/esp32s3-eye/bsp_board.c` — 单麦无回采（`input_format="MN"`）
- `components/hardware_driver/boards/include/bsp_board.h` — 公共 API 声明 + `CONFIG_ESP_CUSTOM_BOARD` → `esp_custom_board.h` 分支
- `components/hardware_driver/boards/include/esp32_s3_korvo_1_v4_board.h` — 引脚表 + `I2S_CONFIG_DEFAULT`/`I2S0_CONFIG_DEFAULT` 宏模板
- `components/hardware_driver/include/esp_board_init.h` — `esp_*` 公共转发 API
- `components/hardware_driver/esp_board_init.c` — `esp_board_init` → `bsp_board_init` 转发实现
- `components/hardware_driver/Kconfig.projbuild` — `choice AUDIO_BOARD` 选板项
