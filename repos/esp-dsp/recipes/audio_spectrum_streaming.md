# 实时音频流 FFT / 频谱可视化

> **适用摘要**: 用定点 `dsps_fft2r_init_sc16` / `dsps_fft2r_sc16` 对 I2S 麦克风的双声道音频块做实时流式 FFT，配 `dsps_wind_blackman_harris_f32` 加窗、`dsps_cplx2reC_sc16` 拆分两路实信号频谱、转 dB 与滑动平均，最后送显示任务。区别于一次性的 `fft_complex.md` / `fft_real.md` 测试信号链路。

## 触发意图

- "实时频谱"
- "音频 FFT 流式"
- "麦���风频谱可视化"
- "sc16 定点 FFT"
- "I2S 麦克风 -> FFT"
- "spectrum box / 频谱盒"
- "双声道 FFT"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考应用 | `applications/spectrum_box_lite/main/main.c`（ESP32-S3-BOX-Lite） |
| 头文件 | `esp_dsp.h`（含 `dsps_fft2r.h` 的 sc16 部分、`dsps_wind.h`、`dsps_mul.h`） |
| Kconfig | `CONFIG_DSP_MAX_FFT_SIZE` ≥ 块大小 |
| 麦克风驱动 | `esp_codec_dev_read`（由 `bsp/esp-bsp.h` 提供，非 ESP-DSP） |
| 显示 | LVGL（`lv_img_dsc_t`，非 ESP-DSP） |

## 分步说明

### 流式 vs 一次性链路对比

| 维度 | 一次性（`fft_complex.md`） | 流式（本 recipe） |
|---|---|---|
| 数据来源 | 内存里的 `dsps_tone_gen_f32` 测试信号 | I2S 麦克风 `esp_codec_dev_read` 实时块 |
| 数据类型 | float `fc32` | **定点 `sc16`**（Q15 int16 复数） |
| init | `dsps_fft2r_init_fc32` | **`dsps_fft2r_init_sc16`** |
| FFT | `dsps_fft2r_fc32` | `dsps_fft2r_sc16` |
| bit-rev | `dsps_bit_rev_fc32` | `dsps_bit_rev_sc16` |
| 拆两路实信号 | `dsps_cplx2reC_fc32` | `dsps_cplx2reC_sc16` |
| 后处理 | 单次 | 滑动平均（跨块保持 `result_data[]`） |
| 任务模型 | 单次调用 | 音频任务 + 显示任务，双核 pin |

### init：定点 sc16 表（关键）

定点 FFT 用独立的 sc16 sin/cos 表，**必须**在首次 `dsps_fft2r_sc16` 之前调用：

```c
esp_err_t ret = dsps_fft2r_init_sc16(NULL, CONFIG_DSP_MAX_FFT_SIZE);
if (ret != ESP_OK) {
    ESP_LOGE(TAG, "Not possible to initialize FFT esp-dsp from library!");
    return;
}
```

### 窗函数：生成 float 窗 → 转 int16 Q15

`spectrum_box_lite` 用 Blackman-Harris 窗，但 FFT 输入是 `int16`，所以窗先用 float 生成再转 Q15，并复制到左右声道交错位：

```c
#define BUFFER_PROCESS_SIZE 512   /* FFT 点数 / 块大小 */
#define I2S_CHANNEL_NUM     2

float result_data[BUFFER_PROCESS_SIZE];          /* 跨块保持：滑动平均频谱 */
int16_t *audio_buffer = (int16_t *)memalign(16, (BUFFER_PROCESS_SIZE + 16) * sizeof(int16_t) * I2S_CHANNEL_NUM);
int16_t *wind_buffer  = (int16_t *)memalign(16, (BUFFER_PROCESS_SIZE + 16) * sizeof(int16_t) * I2S_CHANNEL_NUM);

/* 生成 float 窗，转 int16 Q15，并交错填到 L/R */
dsps_wind_blackman_harris_f32(result_data, BUFFER_PROCESS_SIZE);
for (int i = 0; i < BUFFER_PROCESS_SIZE; i++) {
    wind_buffer[i * 2 + 0] = (int16_t)(result_data[i] * 32767);   /* 左声道 */
    wind_buffer[i * 2 + 1] = wind_buffer[i * 2 + 0];              /* 右声道（同窗） */
}
```

> 注：`result_data` 之后被复用为"频谱滑动平均"缓冲；初始化窗时先借用，建好后改成跨块的 dB 平滑数组。

### 音频任务：读 I2S → 加窗 → FFT → dB → 滑动平均

直接取自 `applications/spectrum_box_lite/main/main.c` 的 `microphone_read_task`：

```c
while (true) {
    /* 1) 从 I2S / codec 读一块双声道 int16（L0,R0,L1,R1,...） */
    int result = esp_codec_dev_read(mic_codec_dev, audio_buffer,
                                    BUFFER_PROCESS_SIZE * sizeof(int16_t) * I2S_CHANNEL_NUM);

    /* 2) 原地乘窗：shift=15 表示 Q15*Q15>>15 = Q15 */
    dsps_mul_s16_ansi(audio_buffer, wind_buffer, audio_buffer,
                      BUFFER_PROCESS_SIZE * 2, 1, 1, 1, 15);

    /* 3) 定点复数 FFT（N 个复数点 = BUFFER_PROCESS_SIZE） */
    dsps_fft2r_sc16_ae32(audio_buffer, BUFFER_PROCESS_SIZE);
    dsps_bit_rev_sc16_ansi(audio_buffer, BUFFER_PROCESS_SIZE);
    /* 4) 把一路复数频谱拆成两路实信号频谱（左/右声道） */
    dsps_cplx2reC_sc16(audio_buffer, BUFFER_PROCESS_SIZE);

    /* 5) 转绝对功率(dB) + 滑动平均 */
    for (int i = 0; i < BUFFER_PROCESS_SIZE; i++) {
        float spectrum_sqr = audio_buffer[i * 2 + 0] * audio_buffer[i * 2 + 0]
                           + audio_buffer[i * 2 + 1] * audio_buffer[i * 2 + 1];
        float spectrum_dB = 10 * log10f(0.1f + spectrum_sqr);
        spectrum_dB = 4 * spectrum_dB;                              /* 屏幕增益 */
        result_data[i] = 0.8f * result_data[i] + 0.2f * spectrum_dB; /* 滑动平均 */
    }
    vTaskDelay(10);
}
```

要点：
- `dsps_mul_s16` 末参 `shift=15` 是 Q15→Q15 的标准移位；填 0 会饱和，填 16 会失精度。
- `dsps_cplx2reC_sc16` 把左声道频谱放前 N/2 个 bin，右声道放后 N/2 个 bin（显示时左右各取屏幕一半）。
- 滑动平均的 `0.8 / 0.2` 系数是经验值，控制余晖（瀑布图效果）。

### 显示任务：读 `result_data[]` 画瀑布图

`main.c` 的 `image_display_task` 把 `result_data` 的前 N/2（左声道）和后 N/2（右声道）映射成屏幕每一行的颜色（蓝→绿→红渐变），每帧把整图上移一行、在底部写新行（LVGL 不属于 ESP-DSP，此处省略细节，见原文件 `spectrum2d_picture()`）。

### 任务划分（pin 到双核）

```c
xTaskCreatePinnedToCore(&microphone_read_task, "Microphone read Task", 8 * 1024, NULL, 3, NULL, 0);  /* core 0 */
xTaskCreatePinnedToCore(&image_display_task,   "Draw task",            10 * 1024, NULL, 5, NULL, 1); /* core 1 */
```

音频任务和显示任务分别 pin 到 core 0 / core 1；`result_data[]` 是它们之间唯一的共享（音频任务写、显示任务读），没有加锁——这是一种"最终一致"的折中，显示偶尔读到半新半旧行也无伤大雅。如果改成 f32 FFT 或做精确频谱分析需要严格同步，请自行加 `xQueue` 或环形缓冲。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| FFT 返回 `ESP_ERR_DSP_UNINITIALIZED` | 用了 fc32 的 init | 定点路径必须 `dsps_fft2r_init_sc16`（与 fc32 是两套独立的表） |
| `dsps_fft2r_sc16` 链接找不到符号 | 未 init 或 Kconfig 未启用 | 先 init；确认 `CONFIG_DSP_MAX_FFT_SIZE` ≥ 块大小 |
| 频谱整体偏低 / 饱和 | `dsps_mul_s16` 的 shift 错 | 窗乘 Q15×Q15 用 `shift=15` 保持 Q15 |
| 左右声道串到一起 | 漏了 `dsps_cplx2reC_sc16` | 该函数把一路复数频谱还原为两路实频谱；不做拆分会看到镜像 |
| 屏幕抖动剧烈 | 没做滑动平均 | 保留跨块的 `result_data[]`，用 `α*old + (1-α)*new` 平滑 |
| `audio_buffer` 越界 | I2S 读字节数算错 | `esp_codec_dev_read` 第三参是**字节数**：`N * sizeof(int16_t) * channel` |
| 用 `_ae32` 后缀调用 | hard-code 后缀 | 应用层用 `dsps_fft2r_sc16` 等无后缀名（`main.c` 中 `_ae32`/`_ansi` 是历史遗留，照原样照抄不报错但不推荐） |

## 参考项目

- `applications/spectrum_box_lite/main/main.c` — 完整双声道实时频谱盒：`dsps_fft2r_init_sc16` + `dsps_wind_blackman_harris_f32` + `dsps_mul_s16` + `dsps_fft2r_sc16` + `dsps_bit_rev_sc16` + `dsps_cplx2reC_sc16` + dB + 滑动平均 + LVGL 瀑布图
- `applications/spectrum_box_lite/README.md` — 音频处理流水线说明（I2S 读 → 加窗 → FFT → bit-rev → 拆两路 → dB → 滑动平均）
- `applications/README.md` — 应用列表（含 LyraT、Azure、M5Stack）
- `modules/fft/include/dsps_fft2r.h` — sc16 系列：`dsps_fft2r_init_sc16` / `dsps_fft2r_sc16` / `dsps_bit_rev_sc16` / `dsps_cplx2reC_sc16`
- `modules/wind/include/dsps_wind.h` — `dsps_wind_blackman_harris_f32`
- `modules/math/mul/include/dsps_mul.h` — `dsps_mul_s16`（含 `shift` 参数）
