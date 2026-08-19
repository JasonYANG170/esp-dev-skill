# FIR 滤波器（标准 / 抽取 / 多速率）

> **适用摘要**: 用 `dsps_fir_f32`（标准 FIR）、`dsps_fird_f32`（抽取 FIR）、`dsps_firmr_f32`（多速率 FIR）做滤波、降采样与任意速率变换。涵盖 `fir_f32_t` 初始化、分块复用与延迟线对齐。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-dsp/resources/`, source/examples in `repos/esp-dsp/`, and this recipe path `repos/esp-dsp/recipes/fir_filter.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "FIR 滤波器"
- "降采样 / 抽取滤波"
- "多速率 FIR"
- "fir_f32_t 怎么用"
- "音频低通滤波"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/fir/main/dsps_fir_main.c` |
| 头文件 | `esp_dsp.h`（含 `dsps_fir.h`） |
| 系数 | 由用户给出，或用 windowed-sinc 自行生成（示例 `generate_FIR_coefficients`） |

## 分步说明

### 标准 FIR

`dsps_fir_init_f32(fir, coeffs, delay, coeffs_len)` —— `delay` 长度需 ≥ `coeffs_len + 4`（注释明确 +4）。

```c
#include "esp_log.h"
#include "esp_dsp.h"

#define FIR_COEFFS_LEN 64
__attribute__((aligned(16))) static float fir_coeffs[FIR_COEFFS_LEN];
__attribute__((aligned(16))) static float delay_line[FIR_COEFFS_LEN + 4];

fir_f32_t fir1;

void app_main(void)
{
    /* 1) 准备系数（这里用示例里的 windowed-sinc 生成 LPF，或直接填你的系数） */
    /* ... generate_FIR_coefficients(fir_coeffs, FIR_COEFFS_LEN, ft); ... */

    /* 2) 初始化（只做一次！） */
    dsps_fir_init_f32(&fir1, fir_coeffs, delay_line, FIR_COEFFS_LEN);

    /* 3) 分块滤波：同一 fir_f32_t 跨块复用以保持连续性 */
    dsps_fir_f32(&fir1, input_block, output_block, block_len);
    /* ... 后续块继续调用 dsps_fir_f32(&fir1, ...) ... */

    /* 4) 不再使用时释放（若 init 时 delay 为 NULL 会内部分配） */
    dsps_fir_f32_free(&fir1);
}
```

### 抽取 FIR（Decimation）

`dsps_fird_init_f32(fir, coeffs, delay, N, decim)` —— 返回输出样本数。

```c
#define DECIMATION 2
fir_f32_t fir_dec;
__attribute__((aligned(16))) static float delay2[FIR_COEFFS_LEN + 4];

dsps_fird_init_f32(&fir_dec, fir_coeffs, delay2, FIR_COEFFS_LEN, DECIMATION);

/* input 长度 = block_len * DECIMATION（实际按需），output 期望长度 block_len */
int produced = dsps_fird_f32(&fir_dec, tone_combined, fir_out, N_buff / DECIMATION);
ESP_LOGI(TAG, "produced %d samples", produced);
```

> 注意：`dsps_fird_f32` 第 4 个参数 `len` 是**结果数组**长度，返回实际写入数（受 decim 与历史状态影响，范围 `[0..len/decim]`）。前若干样本因延迟线未填满会被忽略，示例中用 `fir_out_offset = (FIR_DELAY/2) - 1` 跳过瞬态。

### 多速率 FIR（Interpolation + Decimation）

`dsps_firmr_init_f32(fir, coeffs, delay, length, interp, decim, start_pos)`，对应 `dsps_firmr_f32(fir, input, output, input_len)`，返回输出样本数（约 `input_len*interp/decim`）。

```c
fir_f32_t fir_mr;
dsps_firmr_init_f32(&fir_mr, coeffs, delay_mr, FIR_COEFFS_LEN, /*interp*/3, /*decim*/2, /*start_pos*/0);

int out_n = dsps_firmr_f32(&fir_mr, input, output, input_len);
```

### 16 位定点抽取 FIR

```c
fir_s16_t fir_s16;
int16_t coeffs_s16[N]; int16_t delay_s16[N + 4];
/* coeffs_len / decim / start_pos(0..d-1) / shift（累加器右移位数） */
dsps_fird_init_s16(&fir_s16, coeffs_s16, delay_s16, N, decim, 0, shift);
int32_t n = dsps_fird_s16(&fir_s16, input_s16, output_s16, len);
/* esp32s3 aes3 实现要求系数倒序：可用 dsps_16_array_rev(arr, len)；释放用 dsps_fird_s16_aexx_free(&fir_s16) */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 滤波输出有咔哒声 / 不连续 | 每块都重新 init | `fir_f32_t` 与 `delay[]` 只 init 一次，跨块复用 |
| 输出样本数少于预期 | 抽取比/延迟线瞬态 | `dsps_fird_f32` 的 `len` 是输出长度；跳过前 `(coeffs_len/decim/2)-1` 个瞬态样本 |
| `esp32s3` 上结果错乱 | s16 系数未倒序 | 调 `dsps_16_array_rev(coeffs, len)`，或确认 coeffs_len 能被 4 整除且 16 字节对齐 |
| `dsps_fir_f32_free` 崩溃 | delay 非内部分配 | 仅当 init 时 delay 为 NULL（内部 alloc）才需 free |
| 内存越界 | delay 太短 | delay 长度 ≥ `coeffs_len + 4` |

## 参考

- `examples/fir/main/dsps_fir_main.c` — 含 `generate_FIR_coefficients`（windowed-sinc）、`dsps_fird_init_f32`、`dsps_fird_f32` 完整链路
- `modules/fir/include/dsps_fir.h` — `fir_f32_t`/`fir_s16_t` 结构、`dsps_fir_init_f32`、`dsps_fird_*`、`dsps_firmr_*`、`dsps_16_array_rev`、`dsps_fird_s16_aexx_free`
- `CHANGELOG.md` v1.7.0 — 多速率 FIR 与 resampler 新增
