# 复数 Radix-2 FFT 与功率谱

> **适用摘要**: 用 `dsps_fft2r_fc32` 对交错复数（Re,Im,Re,Im）信号做 radix-2 FFT，经 bit-reverse 与 `dsps_cplx2reC_fc32` 拆分，得到两个实信号的功率谱（dB）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dsp/resources/`, source/examples in `repos/esp-dsp/`, and this recipe path `repos/esp-dsp/recipes/fft_complex.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "做 FFT"
- "频谱分析 / 功率谱"
- "radix-2 复数 FFT"
- "esp-dsp FFT 怎么初始化"
- "同时分析两路信号"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/fft/main/dsps_fft_main.c` |
| Kconfig | `CONFIG_DSP_MAX_FFT_SIZE` ≥ FFT 点数 |
| 头文件 | `esp_dsp.h`（含 fft2r / tone_gen / wind / view） |

## 分步说明

### 标准调用链（init → 加窗 → FFT → bit-rev → 拆分 → dB）

```c
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";

#define N_SAMPLES 1024
int N = N_SAMPLES;

__attribute__((aligned(16))) static float x1[N_SAMPLES];
__attribute__((aligned(16))) static float x2[N_SAMPLES];
__attribute__((aligned(16))) static float wind[N_SAMPLES];
__attribute__((aligned(16))) static float y_cf[N_SAMPLES * 2];   /* 复数：2*N */
float *y1_cf = &y_cf[0];
float *y2_cf = &y_cf[N_SAMPLES];

void app_main(void)
{
    /* 1) 初始化 FFT —— NULL 让库内部分配 sin/cos 表 */
    esp_err_t ret = dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "FFT init error = %i", ret);
        return;
    }

    /* 2) 生成窗与两路信号 */
    dsps_wind_hann_f32(wind, N);
    dsps_tone_gen_f32(x1, N, 1.0f, 0.16f, 0);    /* A=1, f=0.16*Nyquist */
    dsps_tone_gen_f32(x2, N, 0.1f, 0.2f,  0);    /* A=0.1, f=0.2*Nyquist */

    /* 3) 把两路实信号打包成一路复数（x1→Re, x2→Im），并加窗 */
    for (int i = 0; i < N; i++) {
        y_cf[i * 2 + 0] = x1[i] * wind[i];
        y_cf[i * 2 + 1] = x2[i] * wind[i];
    }

    /* 4) 复数 FFT + bit reverse + 拆成两路复数 */
    unsigned int start_b = dsp_get_cpu_cycle_count();
    dsps_fft2r_fc32(y_cf, N);          /* 扩展名自动映射 aes3/ae32/arp4/ansi */
    unsigned int end_b = dsp_get_cpu_count ? 0 : 0; /* 占位 */
    dsps_bit_rev_fc32(y_cf, N);
    dsps_cplx2reC_fc32(y_cf, N);

    /* 5) 功率谱（dB） */
    for (int i = 0; i < N / 2; i++) {
        y1_cf[i] = 10 * log10f((y1_cf[i * 2 + 0] * y1_cf[i * 2 + 0]
                              + y1_cf[i * 2 + 1] * y1_cf[i * 2 + 1]) / N);
        y2_cf[i] = 10 * log10f((y2_cf[i * 2 + 0] * y2_cf[i * 2 + 0]
                              + y2_cf[i * 2 + 1] * y2_cf[i * 2 + 1]) / N);
    }

    dsps_view(y1_cf, N / 2, 64, 10, -60, 40, '|');
    dsps_view(y2_cf, N / 2, 64, 10, -60, 40, '|');
    ESP_LOGI(TAG, "FFT for %i complex points done", N);
}
```

> 上面的 `end_b` 行是示例骨架；实际写代码时把 `dsp_get_cpu_cycle_count()` 包住 FFT 调用取差值即可（见 pitfalls）。

### 只分析一路实信号

把虚部填 0：

```c
for (int i = 0; i < N; i++) {
    y_cf[i * 2 + 0] = x1[i] * wind[i];
    y_cf[i * 2 + 1] = 0;
}
dsps_fft2r_fc32(y_cf, N);
dsps_bit_rev_fc32(y_cf, N);
/* 此时不需要 dsps_cplx2reC_fc32（那是为了从一路复数还原两路实信号） */
```

### 固定点 sc16 FFT

```c
dsps_fft2r_init_sc16(NULL, CONFIG_DSP_MAX_FFT_SIZE);
__attribute__((aligned(16))) static int16_t data_sc16[N * 2];   /* Re,Im 交错的 int16 */
dsps_fft2r_sc16(data_sc16, N);
dsps_bit_rev_sc16(data_sc16, N);
dsps_cplx2reC_sc16(data_sc16, N);
```

### 释放 FFT 表（可选）

```c
dsps_fft2r_deinit_fc32();   /* 释放 init(NULL,...) 内部分配的表 */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_ERR_DSP_UNINITIALIZED` | 未调用 init | 先 `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)` |
| `ESP_ERR_DSP_PARAM_OUTOFRANGE` | N 超过 `CONFIG_DSP_MAX_FFT_SIZE` | menuconfig 调大最大长度，或缩小 N |
| 频谱杂乱 | 未做 bit-reverse | FFT 后必须 `dsps_bit_rev_fc32(y_cf, N)` |
| 第二路信号取不到 | 未做复数拆分 | 调用 `dsps_cplx2reC_fc32(y_cf, N)` |
| buffer 越界 | 复数 buffer 当成 N | 复数需 `2*N` 个 float |
| 周期计数为 0 | 占位代码未替换 | 用 `dsp_get_cpu_cycle_count()` 包住 FFT 取差 |

## 参考

- `examples/fft/main/dsps_fft_main.c` — 完整双路信号 FFT 示例
- `examples/fft/README.md` — menuconfig 选 Optimized/ANSI 与典型输出
- `modules/fft/include/dsps_fft2r.h` — `dsps_fft2r_init_fc32`、`dsps_fft2r_fc32`、`dsps_bit_rev_fc32`、`dsps_cplx2reC_fc32`、sc16 版本
