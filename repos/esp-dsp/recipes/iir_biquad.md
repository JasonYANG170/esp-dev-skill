# IIR Biquad 滤波器

> **适用摘要**: 用系数生成器 `dsps_biquad_gen_*_f32`（LPF/HPF/BPF/notch/shelf/peak/allpass）生成 biquad 系数，再用 `dsps_biquad_f32`（单声道）或 `dsps_biquad_sf32`（立体声）做 IIR 滤波。

## 触发意图

- "IIR 滤波器"
- "biquad"
- "低通 / 高通 / 带通"
- "陷波 / notch"
- "EQ / 峰值 / shelf 滤波"
- "立体声滤波"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/iir/main/dsps_iir_main.c` |
| 头文件 | `esp_dsp.h`（含 `dsps_biquad.h`、`dsps_biquad_gen.h`） |

## 分步说明

### 系数布局约定

`coeffs[5] = { b0, b1, b2, a1, a2 }`，`a0` 不放入数组、恒为 1（IIR 内部假定）。延迟线 `w[2]`（立体声为 `w[4]`，前 2 个为声道 0，后 2 个为声道 1）。

### 单声道 LPF（含冲激响应与频响）

直接取自 `examples/iir`：

```c
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";
#define N 1024
__attribute__((aligned(16))) static float d[N];
__attribute__((aligned(16))) static float y[N];
__attribute__((aligned(16))) static float y_cf[N * 2];

static void show_iir(float freq, float qFactor)
{
    float coeffs_lpf[5];
    float w_lpf[5] = {0, 0};

    /* freq 归一化到采样率，范围 [0..0.5]；qFactor 为 Q 值 */
    esp_err_t ret = dsps_biquad_gen_lpf_f32(coeffs_lpf, freq, qFactor);
    if (ret != ESP_OK) { ESP_LOGE(TAG, "coeff error %i", ret); return; }

    /* 滤波：d 为输入（这里是 delta 冲激），y 为输出 */
    unsigned int start_b = dsp_get_cpu_cycle_count();
    dsps_biquad_f32(d, y, N, coeffs_lpf, w_lpf);
    unsigned int end_b = dsp_get_cpu_cycle_count();

    dsps_view(y, 128, 64, 10, -1, 1, '-');   /* 冲激响应 */

    /* 频响：把 y 当实信号填入复数虚部 0，做 FFT */
    for (int i = 0; i < N; i++) { y_cf[i*2+0] = y[i]; y_cf[i*2+1] = 0; }
    dsps_fft2r_fc32_ansi(y_cf, N);   /* 此处示例显式用 _ansi 演示；应用代码用 dsps_fft2r_fc32 */
    dsps_bit_rev_fc32_ansi(y_cf, N);
    for (int i = 0; i < N/2; i++)
        y_cf[i] = 10*log10f((y_cf[i*2+0]*y_cf[i*2+0] + y_cf[i*2+1]*y_cf[i*2+1]) / N);
    dsps_view(y_cf, N/2, 64, 10, -100, 0, '-');
    ESP_LOGI(TAG, "IIR for %d samples take %u cycles", N, end_b - start_b);
}
```

### 各类系数生成器（参数差异）

```c
float c[5];
dsps_biquad_gen_lpf_f32(c, f, q);            /* LPF：f∈[0..0.5], q */
dsps_biquad_gen_hpf_f32(c, f, q);            /* HPF */
dsps_biquad_gen_bpf_f32(c, f, q);            /* BPF */
dsps_biquad_gen_bpf0db_f32(c, f, q);         /* BPF，通带 0 dB 增益 */
dsps_biquad_gen_notch_f32(c, f, gain, q);    /* 陷波：gain 为阻带增益(dB) */
dsps_biquad_gen_allpass360_f32(c, f, q);     /* 全通 360° */
dsps_biquad_gen_allpass180_f32(c, f, q);     /* 全通 180° */
dsps_biquad_gen_peakingEQ_f32(c, f, q);      /* 峰值 EQ */
dsps_biquad_gen_lowShelf_f32(c, f, gain, q); /* 低 shelf：gain(dB) */
dsps_biquad_gen_highShelf_f32(c, f, gain, q);/* 高 shelf：gain(dB) */
```

### 立体声 biquad

输入/输出按 L/R/L/R... 交错，`len` 为单声道样本数，延迟线 `w[4]`：

```c
float w_stereo[4] = {0,0,0,0};
dsps_biquad_sf32(stereo_in, stereo_out, frames_per_channel, coeffs, w_stereo);
```

### 高阶滤波（级联 biquad）

把前一级 `y` 作为下一级输入，复用各自的 `w[]` 即可级联成 4 阶/6 阶：

```c
dsps_biquad_f32(x, y1, N, c1, w1);
dsps_biquad_f32(y1, y2, N, c2, w2);   /* 第二级 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 截止频率完全错 | `f` 当成 Hz | `f = fc_hz / fs_hz`，范围 [0..0.5] |
| 滤波器不稳定/爆掉 | Q 值过大或系数越界 | 检查 Q；notch/shelf 还需 `gain`；保持 a0=1（不要手填） |
| 输出不连续 | 每块重置 `w[]` | `w[]` 跨块复用，只在开始清零一次 |
| 立体声串音 | 用了单声道函数 | 立体声用 `dsps_biquad_sf32`，`w` 长度 4 |
| 频响图全 0 | 未做 FFT / bit-rev | biquad 后接 `dsps_fft2r_fc32` + `dsps_bit_rev_fc32` |

## 参考

- `examples/iir/main/dsps_iir_main.c` — `dsps_biquad_gen_lpf_f32` + `dsps_biquad_f32` + 冲激/频响显示
- `modules/iir/include/dsps_biquad.h` — `dsps_biquad_f32`、`dsps_biquad_sf32`
- `modules/iir/include/dsps_biquad_gen.h` — 全部系数生成器
- `CHANGELOG.md` v1.6.0 — 立体声 biquad `dsps_biquad_sf32` 为 1.6.0 新增
