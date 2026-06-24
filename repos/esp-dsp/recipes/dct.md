# DCT / DST 变换

> **适用摘要**: 用 `dsps_dct_f32`（DCT-II）、`dsps_dct_inv_f32`（逆 DCT）、`dsps_dctiv_f32`（DCT-IV）、`dsps_dstiv_f32`（DST-IV）做离散余弦/正弦变换。这些函数基于 FFT，使用前需 init FFT 表。

## 触发意图

- "DCT"
- "离散余弦变换"
- "DCT-IV / DST-IV"
- "逆 DCT"
- "音频压缩变换"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `esp_dsp.h`（含 `dsps_dct.h`） |
| FFT 表 | DCT/DST 基于 FFT，需先 `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)` |
| Kconfig | `CONFIG_DSP_MAX_FFT_SIZE` ≥ 变换长度 |

## 分步说明

### DCT-II（正变换，未归一化）

```c
#include "esp_log.h"
#include "esp_dsp.h"

static const char *TAG = "main";
#define N 512

/* DCT-II 要求 data 数组大小为 2*N（前 N 个为输入，后半可为任意） */
__attribute__((aligned(16))) static float data[2 * N];

void app_main(void)
{
    esp_err_t ret = dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);
    if (ret != ESP_OK) { ESP_LOGE(TAG, "FFT init %i", ret); return; }

    /* 前 N 个填入实信号，后半可留 0 */
    for (int i = 0; i < N; i++) data[i] = ... ;
    dsps_dct_f32(data, N);   /* 结果存在 data[0..N-1] */
}
```

### 逆 DCT（type III，未归一化）

```c
dsps_dct_inv_f32(data, N);   /* data 大小仍为 2*N */
```

### DCT-IV 与 DST-IV（type IV，未归一化）

```c
/* DCT-IV / DST-IV 的 data 数组大小为 N（不是 2*N） */
dsps_dctiv_f32(data, N);
dsps_dstiv_f32(data, N);
```

### 参考实现（用于验证）

库提供非优化的参考版本，便于交叉验证：

```c
float result[N];
dsps_dct_f32_ref(data, N, result);          /* 直接 DCT-II 参考 */
dsps_dct_inverce_f32_ref(data, N, result);  /* 逆 DCT 参考（注意拼写沿用了头文件中的命名） */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 结果为 0 / 崩溃 | FFT 表未 init | DCT 基于 FFT，必须先 `dsps_fft2r_init_fc32` |
| 数组越界 | DCT-II 用了 N 而非 2*N | `dsps_dct_f32`/`dsps_dct_inv_f32` 需 `2*N`；`dctiv`/`dstiv` 只需 N |
| 还原信号幅度不对 | 变换未归一化 | 这些函数都是 unscaled，需自行按 N 做幅度补偿 |
| 找不到 `dsps_dct_inverse_f32` | 函数名拼写 | 头文件中为 `dsps_dct_inverce_f32_ref`（沿用原拼写） |

## 参考

- `modules/dct/include/dsps_dct.h` — `dsps_dct_f32`、`dsps_dct_inv_f32`、`dsps_dctiv_f32`、`dsps_dstiv_f32`、`dsps_dct_f32_ref`、`dsps_dct_inverce_f32_ref`
- `CHANGELOG.md` v1.6.0 —— DCT-IV / DST-IV 为 1.6.0 新增
