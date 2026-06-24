# DOA 声源定位

> **适用摘要**: 用 ESP-SR 的 SRP-PHAT 声源定位（DOA）从双麦克风估计语音方向角（0~180°）。覆盖独立 `esp_doa_*`（左右声道分开）与 AFE-aware 的 `afe_doa_*`（按 `input_format` 自动从交错多通道数据里抽取左右声道）两条路径。

## 触发意图

- "声源定位"
- "DOA 怎么用"
- "声音从哪个方向来"
- "esp_doa_create / afe_doa_create"
- "SRP-PHAT 双麦"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 双麦克风线性阵列（左右两个 MIC 朝向同一平面）；ESP32-S3 / ESP32-P4 / ESP32 等 |
| 输入 | 16kHz / 16-bit PCM，左右各一帧（帧长由 `input_timedate_samples` 决定，推荐 1024） |
| 物理 | `d_mics`（两麦中心距，米）与实物一致；测试代码默认 0.06m（6cm） |
| 组件 | `espressif/esp-sr`（DOA 不依赖模型，无需 `model` 分区） |
| 参考 | `test_apps/esp-sr/main/test_afe.cpp`（`test doa interface`） |

## 关键参数

`esp_doa_create(fs, resolution, d_mics, input_timedate_samples)` 的四个参数是 SRP-PHAT 估计的核心。头文件 `esp_doa.h` 明确标注 "Recommend using the above configuration for better performance"——即 16000 / 20.0 / 0.06 / 1024 这一组推荐值。

| 参数 | 含义 | 推荐值 | 影响 |
|---|---|---|---|
| `fs` | 采样率（Hz） | 16000 | 决定可分辨的最高时延，越高角度分辨率越好 |
| `resolution` | 角度搜索步长（度） | 20.0 | 越小越细但越费 CPU；返回角度也按此量化 |
| `d_mics` | 两麦中心距（米） | 0.06 | **必须与实物一致**，否则角度估计整体偏差 |
| `input_timedate_samples` | 单帧样本数（每声道） | 1024 | 帧长，越大越稳但延迟越高 |

## 分步说明

### 方式 A：独立 esp_doa_*（左右声道分开）

最简单的用法，适合你已经把左右两路 PCM 各自放在独立 buffer 里的场景（例如直接从两路独立 I2S 读取）。完全照搬 `test_afe.cpp` 的 `test doa interface`：

```c
#include "esp_doa.h"
#include "esp_log.h"
#include <math.h>

static const char *TAG = "DOA";

// 推荐配置（与 test_apps/esp-sr/main/test_afe.cpp 的 "test doa interface" 一致）
#define SAMPLE_RATE   16000
#define RESOLUTION    20.0f
#define D_MICS        0.06f       // 两麦中心距，必须与实物一致
#define FRAME_SAMPLES 1024        // 单声道每帧样本数

void doa_task(void *arg)
{
    int16_t *left  = malloc(FRAME_SAMPLES * sizeof(int16_t));
    int16_t *right = malloc(FRAME_SAMPLES * sizeof(int16_t));

    doa_handle_t *doa = esp_doa_create(SAMPLE_RATE, RESOLUTION, D_MICS, FRAME_SAMPLES);

    while (1) {
        // TODO: 从两路 I2S / 文件读取 FRAME_SAMPLES 个 16-bit 样本
        // read_left(left, FRAME_SAMPLES);
        // read_right(right, FRAME_SAMPLES);

        // 处理一帧，返回估计角度（0~180，单位度）
        // 角度含义：0 与 180 表示正对其中一只麦；90 表示声源在两麦正中前方
        float angle = esp_doa_process(doa, left, right);
        ESP_LOGI(TAG, "DOA angle = %.1f", angle);

        // 按帧长节流：1024 样本 / 16000 Hz = 64 ms
        vTaskDelay(pdMS_TO_TICKS(FRAME_SAMPLES * 1000 / SAMPLE_RATE));
    }

    esp_doa_destroy(doa);   // 用完必须 destroy，避免内存泄漏
    free(left);
    free(right);
    vTaskDelete(NULL);
}
```

> `esp_doa_process` 每帧返回一个角度，单帧估计会有抖动。实际产品通常对多帧做平滑/投票（例如取最近 5~10 帧的中位数）再驱动灯环或波束切换。

### 方式 B：afe_doa_*（带 input_format 自动解交错）

当双麦数据是**通道交错（interleaved）**排布（I2S TDM / 多通道 Codec 的常见输出），用 `esp_afe_doa.h` 的封装可以省去手动拆分。`afe_doa_create` 多接受一个 `input_format` 参数，内部按 `pcm_config.mic_ids` 抽取前两个麦克风通道作为 left/right。

```c
#include "esp_afe_doa.h"
#include "esp_log.h"

static const char *TAG = "DOA";

// input_format 与 AFE 一致：M=麦克风，N=未用，R=参考
// "MMNR" = 麦、麦、未用、参考（双麦 + 1 参考）
#define INPUT_FORMAT   "MMNR"
#define SAMPLE_RATE    16000
#define RESOLUTION     20.0f
#define D_MICS         0.06f
#define FRAME_SAMPLES  1024

void doa_task(void *arg)
{
    // input_format 让封装自动解析 pcm_config（total_ch_num / mic_ids / ref_ids）
    afe_doa_handle_t *h = afe_doa_create(INPUT_FORMAT, SAMPLE_RATE,
                                         RESOLUTION, D_MICS, FRAME_SAMPLES);

    int frame_size = h->frame_size;                 // = FRAME_SAMPLES（单声道每帧样本数）
    int total_ch   = h->pcm_config.total_ch_num;    // = strlen(INPUT_FORMAT)，例如 4
    int mic0       = h->pcm_config.mic_ids[0];      // 第一只麦在交错流里的索引
    int mic1       = h->pcm_config.mic_ids[1];      // 第二只麦在交错流里的索引

    // indata 是交错的整帧多通道数据：frame_size * total_ch 个 16-bit 样本
    int16_t *indata = malloc(frame_size * total_ch * sizeof(int16_t));

    ESP_LOGI(TAG, "frame=%d total_ch=%d mic_ids=[%d,%d]",
             frame_size, total_ch, mic0, mic1);

    while (1) {
        // TODO: 从多通道 I2S 读取一帧交错数据
        // read_i2s_multich(indata, frame_size * total_ch);

        // 内部自动抽取 left/right 并跑 SRP-PHAT，返回角度 0~180
        float angle = afe_doa_process(h, indata);
        ESP_LOGI(TAG, "DOA angle = %.1f", angle);

        vTaskDelay(pdMS_TO_TICKS(frame_size * 1000 / SAMPLE_RATE));
    }

    afe_doa_destroy(h);
    free(indata);
    vTaskDelete(NULL);
}
```

> `afe_doa_handle_t` 是**公开结构体**（非不透明指针），成员 `doa_handle` / `pcm_config` / `leftdata` / `rightdata` / `frame_size` 可直接读（见 `esp_afe_doa.h`）。`leftdata` / `rightdata` 是封装内部为抽取出的左右声道分配的缓冲，一般不需要手动改。

### 两种方式怎么选

| 条件 | 推荐 |
|---|---|
| 双麦各自一路独立 buffer / 独立 I2S | 独立 `esp_doa_*`（方式 A） |
| 多通道交错 I2S（TDM、Codec 合并输出） | `afe_doa_*`（方式 B，自动解交错） |
| 已经在跑 AFE pipeline，只想顺带测角度 | 在 feed 侧另起一路 `afe_doa_process`，复用同一份交错输入 |

### 头文件按 target 自动选择

DOA 头文件位于 `include/<target>/`，由组件 CMake 按 `IDF_TARGET` 自动挑选，签名在所有 target 上完全一致：

| target | 独立 DOA | AFE-aware DOA |
|---|---|---|
| ESP32 | `include/esp32/esp_doa.h` | （该 target 不提供 `esp_afe_doa.h`） |
| ESP32-S3 / ESP32-S31 | `include/esp32s3/esp_doa.h` | `include/esp32s3/esp_afe_doa.h` |
| ESP32-P4 | `include/esp32p4/esp_doa.h` | `include/esp32p4/esp_afe_doa.h` |

代码里只需 `#include "esp_doa.h"`（或 `"esp_afe_doa.h"`），无需写 target 前缀。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 角度恒为某个固定值（如一直 90） | 左右声道数据相同 / 接线错 | 核对两路 I2S，确认 left/right 是两个不同麦克风的信号 |
| 角度整体偏移、方向反 | `d_mics` 与实物不符，或 left/right 接反 | 量准两麦中心距填入 `D_MICS`；交换 left/right 验证 |
| 角度抖动剧烈 | 单帧估计噪声大 | 对最近 N 帧取中位数/均值平滑；增大 `input_timedate_samples` |
| 角度只有 0 / 90 / 180 三档 | `resolution` 设太大（如 90） | 用推荐值 20；越小越细但越费 CPU |
| `esp_doa_process` 返回 -1 / NaN | 帧长不足 / 缓冲未填满 | 每帧必须喂满 `input_timedate_samples` 个样本 |
| 反复 create 不 destroy，内存涨 | 漏调 `esp_doa_destroy` | 循环里 create/process/destroy 配套；`test_afe.cpp` 即做此泄漏检查 |
| `afe_doa_create` 返回 NULL | `input_format` 里 `M` 少于 2 个 | DOA 需要至少 2 个麦克风通道，format 含 `MM` |
| 编译报 `esp_afe_doa.h: No such file` | 用了不提供该头的 target（如 ESP32） | 该 target 只支持独立 `esp_doa_*`；或换 S3/P4 |

## 参考

- `test_apps/esp-sr/main/test_afe.cpp` — `TEST_CASE("test doa interface", "[afe]")`：`esp_doa_create(16000, 20.0f, 0.06f, 1024)` → `esp_doa_process` → 打印角度与内存/CPU 占用，并做 5 次 create/destroy 泄漏检查
- `include/esp32s3/esp_doa.h` — `doa_handle_t` / `esp_doa_create` / `esp_doa_process` / `esp_doa_destroy`
- `include/esp32s3/esp_afe_doa.h` — `afe_doa_handle_t`（含 `pcm_config` / `leftdata` / `rightdata` / `frame_size`）/ `afe_doa_create` / `afe_doa_process` / `afe_doa_destroy`
- `include/esp32/esp_doa.h`、`include/esp32p4/esp_afe_doa.h` — 其他 target 同名同签名
- 配套 recipe：`recipes/afe_sr_pipeline.md`（feed/fetch 双任务骨架，可在 feed 侧并行跑 DOA）
