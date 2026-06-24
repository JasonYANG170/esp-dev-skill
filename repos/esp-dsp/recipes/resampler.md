# 重采样器（多速率 FIR 与多项式）

> **适用摘要**: 用多速率重采样器 `dsps_resampler_mr_init/exec/free` 做基于多速率 FIR 的采样率变换，或用多项式（Farrow）重采样器 `dsps_resampler_ph_init/exec` 做轻量的相位可调重采样。支持 float 与 int16 定点。

## 触发意图

- "重采样 / 采样率变换"
- "resampler"
- "44.1k 转 48k"
- "多速率 / multi-rate"
- "相位可调重采样"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `esp_dsp.h`（含 `dsps_resampler.h`，已聚合进 esp_dsp.h） |
| 系数 | 用户自行提供低通 FIR 系数（多速率重采样器基于它） |
| 版本 | resampler 为 v1.7.0 新增 |

## 分步说明

### 多速率重采样器（float）

`dsps_resample_mr_t` 内部持有一个多速率 FIR。`samplerate_factor = out_rate / in_rate`。

```c
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";

void app_main(void)
{
    /* 假设已有低通 FIR 系数 coeffs[LENGTH]（用于抗混叠/插值） */
    #define LENGTH 64
    extern float coeffs[LENGTH];

    dsps_resample_mr_t rs;
    /* length=系数长度, interp=插值因子, samplerate_factor=out/in,
       fixed_point=0 (float) / 1 (int16), shift 仅定点用 */
    esp_err_t ret = dsps_resampler_mr_init(&rs, coeffs, LENGTH,
                                           /*interp*/ 147,
                                           /*samplerate_factor*/ 48000.0f/44100.0f,
                                           /*fixed_point*/ 0, /*shift*/ 0);
    if (ret != ESP_OK) { ESP_LOGE(TAG, "init %i", ret); return; }

    /* 执行：input/output 可为同一 buffer；返回输出样本数 */
    int32_t out_n = dsps_resampler_mr_exec(&rs, input, output, input_len,
                                           /*length_correction*/ 0);
    ESP_LOGI(TAG, "produced %d samples", out_n);

    /* 运行时微调输出速率：正值提高输出速率，负值降低（用于源/宿时钟漂移） */
    out_n = dsps_resampler_mr_exec(&rs, input2, output2, input_len2, +1);

    dsps_resampler_mr_free(&rs);
}
```

### 定点多速率重采样器（int16）

```c
dsps_resample_mr_t rs;
int16_t coeffs_s16[LENGTH];
dsps_resampler_mr_init(&rs, coeffs_s16, LENGTH, interp,
                       samplerate_factor, /*fixed_point*/1, /*shift*/15);
int16_t in[L], out[L];
int32_t n = dsps_resampler_mr_exec(&rs, in, out, L, 0);
dsps_resampler_mr_free(&rs);
```

### 多项式（Farrow）重采样器

轻量级、4 系数三次插值，相位 `step = out_rate/in_rate` 可运行时调：

```c
dsps_resample_ph_t ph;
/* samplerate_factor = out_rate / in_rate */
dsps_resampler_ph_init(&ph, 48000.0f / 44100.0f);

float in[1024], out[1024];
int32_t n = dsps_resampler_ph_exec(&ph, in, out, 1024);
```

### 两种重采样器对比

| 特性 | 多速率（`mr`） | 多项式（`ph`） |
|---|---|---|
| 内核 | 多速率 FIR（抗混叠好） | 三次 Farrow 插值（轻量） |
| 系数 | 需用户提供 FIR 系数 | 无需系数 |
| 运行时调速率 | `length_correction` 参数 | 调 `step` / `phase` |
| 定点支持 | 是（`fixed_point=1`） | 否（仅 float） |
| 延迟结构 | `dsps_resample_mr_t`（含 FIR） | `dsps_resample_ph_t`（含 4-tap delay） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 输出样本数偏离期望 | `samplerate_factor` 算反 | factor = out_rate / in_rate |
| 高频有混叠 | 低通 FIR 系数截止不当 | 选 cutoff = min(in,out)/2 的 LPF 系数 |
| 长期速率漂移 | 源/宿时钟不同源 | 用 `length_correction` 持续修正（+提高、-降低输出速率） |
| 定点溢出 | `shift` 不合适 | 调整 shift（累加器右移位数），与 FIR 定点用法一致 |
| 内存泄漏 | 未 free | 多速率用完调 `dsps_resampler_mr_free` |

## 参考

- `modules/fir/include/dsps_resampler.h` — `dsps_resample_mr_t`/`dsps_resample_ph_t` 结构与全部函数
- `CHANGELOG.md` v1.7.0 —— Multirate FIR filter 与 Resampler based on Multirate FIR 为 1.7.0 新增
