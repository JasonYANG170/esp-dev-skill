# 实信号 FFT（Radix-4）

> **适用摘要**: 用 `dsps_fft4r_fc32`（radix-4）对实信号做 FFT，配合 `dsps_bit_rev4r_fc32` 与 `dsps_cplx2real_fc32` 把结果还原为实数频谱；适合单路实信号、点数为 4 的幂的场景。

## 触发意图

- "实数 FFT"
- "radix-4 FFT"
- "fft4r"
- "单路信号频谱"
- "FFT4R / FFT2R 对比"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/fft4real/main/dsps_fft4real_main.c`、`examples/fir/main/dsps_fir_main.c`（`show_FFT` 用到 fft4r） |
| Kconfig | `CONFIG_DSP_MAX_FFT_SIZE` ≥ FFT 点数 |
| 头文件 | `esp_dsp.h`（含 fft4r） |

## 分步说明

### 实信号 FFT 完整链路

```c
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";
#define N 1024

/* 对实信号 fft_len 个样本（已打包为 Re,0,Re,0 的复数）做 radix-4 FFT */
static void real_fft(float *data, int fft_len)
{
    /* radix-4：对 fft_len/2 个复数点做 FFT */
    dsps_fft4r_fc32(data, fft_len >> 1);
    dsps_bit_rev4r_fc32(data, fft_len >> 1);
    /* 把复数 FFT 结果还原为实数频谱（针对实输入） */
    dsps_cplx2real_fc32(data, fft_len >> 1);

    const float correction = fft_len * 3;
    for (int i = 0; i < fft_len / 2; i++) {
        data[i] = 10 * log10f((data[i * 2 + 0] * data[i * 2 + 0]
                             + data[i * 2 + 1] * data[i * 2 + 1] + 1e-7f) / correction);
    }
    dsps_view(data, fft_len / 2, 64, 10, -120, 40, '|');
}

void app_main(void)
{
    /* radix-4 需要单独 init（与 radix-2 的表不同） */
    esp_err_t ret = dsps_fft4r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "FFT4 init error %i", ret);
        return;
    }
    /* 通常也会同时 init radix-2（互不冲突） */
    dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);

    __attribute__((aligned(16))) static float y_cf[N * 2];
    /* ... 把实信号填到 y_cf 的偶数下标，奇数下标填 0 ... */
    real_fft(y_cf, N);
}
```

> 关键点：实信号先按 `y_cf[2i]=x[i]; y_cf[2i+1]=0;` 打包成 N/2 个复数点，再传 `fft_len>>1` 给 fft4r。

### Radix-2 vs Radix-4 对比

`examples/fft4real` 同时跑两套并比较周期数：

```c
dsps_fft2r_fc32(y_cf, N);          /* radix-2，对 N 个复数点 */
dsps_fft4r_fc32(y_cf2, N);         /* radix-4，对 N 个复数点 */
/* 二者各用各自的 bit-reverse：dsps_bit_rev_fc32 / dsps_bit_rev4r_fc32 */
```

### 关键宏对照

| 操作 | radix-2 (fft2r) | radix-4 (fft4r) |
|---|---|---|
| init | `dsps_fft2r_init_fc32` | `dsps_fft4r_init_fc32` |
| 正变换 | `dsps_fft2r_fc32(data, N)` | `dsps_fft4r_fc32(data, N)` |
| bit reverse | `dsps_bit_rev_fc32` | `dsps_bit_rev4r_fc32` |
| 实输入还原 | `dsps_cplx2reC_fc32`（两路实→一路复） | `dsps_cplx2real_fc32`（一路复→实频谱） |
| deinit | `dsps_fft2r_deinit_fc32` | `dsps_fft4r_deinit_fc32` |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 链接找不到 `dsps_fft4r_*` | 未 init 或 Kconfig 未启用 | 先 `dsps_fft4r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)` |
| 用 `dsps_bit_rev_fc32` 配 fft4r | bit-reverse 函数不匹配 | radix-4 必须用 `dsps_bit_rev4r_fc32` |
| 实频谱还原错误 | 漏掉 `dsps_cplx2real_fc32` | 实输入场景必须调用它 |
| `ESP_ERR_DSP_PARAM_OUTOFRANGE` | N 不是 4 的幂或超 max | radix-4 点数应为 4 的幂；必要时调大 Kconfig |
| `dsps_cplx2real_fc32` 崩溃 | fft4r 未 init | 该函数依赖 fft4r 的内部表 |

## 参考

- `examples/fft4real/main/dsps_fft4real_main.c` — radix-2 与 radix-4 对比示例
- `examples/fir/main/dsps_fir_main.c` — `show_FFT()` 函数演示 fft4r 实信号链路
- `modules/fft/include/dsps_fft4r.h` — `dsps_fft4r_init_fc32`、`dsps_fft4r_fc32`、`dsps_bit_rev4r_fc32`、`dsps_cplx2real_fc32`
