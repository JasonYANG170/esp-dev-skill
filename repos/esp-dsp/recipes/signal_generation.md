# 信号生成与文本显示

> **适用摘要**: 用 esp-dsp 生成测试信号（正弦、delta、复数 LUT 信号），并通过 `dsps_view` / `dsps_view_spectrum` 在串口打印文本波形与频谱图。

## 触发意图

- "生成正弦信号 / 测试信号"
- "esp-dsp 画图 / 打印波形"
- "复数信号发生器"
- "delta 函数"
- "怎么看频谱"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/basic_math/main/dsps_math_main.c`、`examples/iir/main/dsps_iir_main.c` |
| 头文件 | `esp_dsp.h`（已聚合 tone_gen / d_gen / cplx_gen / view） |

## 分步说明

### 生成正弦信号

`dsps_tone_gen_f32(output, len, Ampl, freq, phase)` —— `freq` 归一化到 Nyquist，范围 `[-1..1]`（1 = 采样率/2）；`phase` 为度。

```c
#include "esp_dsp.h"

#define N 1024
__attribute__((aligned(16))) static float x1[N];

void app_main(void)
{
    /* 幅度 1.0，归一化频率 0.16（即 0.16 * fs/2 Hz），相位 0 */
    dsps_tone_gen_f32(x1, N, 1.0f, 0.16f, 0);
}
```

### 生成 delta（脉冲）函数

`dsps_d_gen_f32(output, len, pos)` —— 仅 `output[pos]=1`，其余为 0。常用于求滤波器冲激响应。

```c
dsps_d_gen_f32(d, N, 0);   /* 第 0 个样本为 1，常作为 IIR 冲激输入 */
```

### 复数信号发生器（基于 LUT，频率/相位可运行时改）

```c
#include "esp_dsp.h"

cplx_sig_t gen;
/* d_type: S16_FIXED 或 F32_FLOAT；lut 传 NULL 让库内部生成，lut_len 为表长；
   freq 归一化 [-1..1]（1=Nyquist），initial_phase 归一化 [-1..1]（1=2Pi） */
dsps_cplx_gen_init(&gen, F32_FLOAT, NULL, 256, 0.2f, 0.0f);

float out[2 * 128];                 /* 复数：Re,Im 交错 */
dsps_cplx_gen(&gen, out, 128);      /* 生成 128 个复数样本 */

/* 运行时改频率/相位 */
dsps_cplx_gen_freq_set(&gen, -0.3f);
dsps_cplx_gen_phase_set(&gen, 0.25f);
/* 或同时设置：dsps_cplx_gen_set(&gen, freq, phase); */

cplx_gen_free(&gen);                /* 释放 init 内部分配的 LUT */
```

辅助查询：`dsps_cplx_gen_freq_get(&gen)`、`dsps_cplx_gen_phase_get(&gen)`。

### 串口文本波形 `dsps_view`

```c
/* (data, len, width, height, min, max, view_char) */
dsps_view(x1, N, 64, 10, -1.0f, 1.0f, '|');
```

### 串口频谱图 `dsps_view_spectrum`

```c
/* 64x10 的频谱视图，Y 轴 dB 范围 [-100, 0] */
dsps_view_spectrum(y_cf, N / 2, -100.0f, 0.0f);
```

### 完整示例（生成 + 显示 + SNR/SFDR）

```c
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";
#define N 1024
__attribute__((aligned(16))) static float sig[N];

void app_main(void)
{
    dsps_tone_gen_f32(sig, N, 1.0f, 0.2f, 0);
    dsps_view(sig, 128, 64, 10, -1.0f, 1.0f, '-');

    float snr  = dsps_snr_f32(sig, N, 0);   /* 单频信号的 SNR(dB)，use_dc=0 不计 DC */
    float sfdr = dsps_sfdr_f32(sig, N, 0);
    ESP_LOGI(TAG, "SNR=%.2f dB, SFDR=%.2f dB", snr, sfdr);
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 频率不对 | `freq` 当成了 Hz | `freq` 归一化到 Nyquist：`f = fhz / (fs/2)` |
| 波形幅度异常 | `Ampl` 单位错 | `Ampl` 直接是峰值幅度（非 dB） |
| `dsps_view` 全空 | min/max 范围不含数据 | 放宽 min/max，或先看 `ESP_LOG` 打印的 Data min/max |
| SNR/SFDR 偏低 | 信号非纯正弦 / 含直流 | 设 `use_dc=0` 排除 DC；SNR/SFDR 仅适用于单音 |
| `cplx_gen_free` 崩溃 | 未 init 就 free 或重复 free | 与 `dsps_cplx_gen_init` 成对调用 |

## 参考

- `examples/basic_math/main/dsps_math_main.c` — `dsps_tone_gen_f32` + `dsps_view` 用法
- `examples/iir/main/dsps_iir_main.c` — `dsps_d_gen_f32` 作冲激输入
- `modules/support/include/dsps_tone_gen.h`、`dsps_d_gen.h`、`dsps_cplx_gen.h`、`dsps_view.h`、`dsps_snr.h`、`dsps_sfdr.h`
