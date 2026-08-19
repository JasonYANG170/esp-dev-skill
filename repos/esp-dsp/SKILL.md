---
name: esp-dsp-skill
description: >-
  AI Skill for the Espressif ESP-DSP library (FFT / FIR / IIR / matrix / dot product / Kalman
  on ESP32, ESP32-S3, ESP32-P4). Use when developing, integrating, or debugging DSP code as an
  ESP-IDF component: signal generation, windowing, forward/inverse FFT, real-input FFT, FIR/IIR
  filters, resamplers, matrix math, the C++ dspm::Mat class, and the Extended Kalman Filter.
  Trigger words: "ESP-DSP", "esp-dsp", "FFT", "FIR", "IIR", "biquad", "dspm::Mat", "dotprod",
  "Kalman", "EKF", "resampler", "数字信号处理", "DSP库", "快速傅里叶", "滤波器", "矩阵运算", "ESP32-S3", "ESP32"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-dsp-skill

AI Skill for the Espressif **ESP-DSP** library — the official DSP library for Espressif SoCs. Provides scenario-driven recipes (Chinese), a complete real-API reference, Kconfig/build guidance, and the pitfalls that actually bite when integrating FFT/FIR/IIR/matrix/Kalman code into an ESP-IDF project. Covers ESP32, ESP32-S3, and ESP32-P4 optimized paths plus the portable ANSI fallback.

## Core Principles

1. **Never invent an API** — every function/struct/macro below exists in `modules/*/include/`. If it is not in `resources/api_reference.md`, it does not exist; do not fabricate a name.
2. **There is no global `esp_dsp_init()`** — the only module that needs initialization is FFT. Call `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)` (and `dsps_fft4r_init_fc32(...)` if you use radix-4) before any `dsps_fft2r_fc32` / `dsps_fft4r_fc32` call.
3. **Use the extension-less name** — call `dsps_fft2r_fc32`, `dsps_fir_f32`, `dsps_biquad_f32`, `dspm_mult_f32`, etc. The header `#define`s map these to `_aes3`/`_ae32`/`_arp4`/`_ansi` automatically based on `CONFIG_DSP_OPTIMIZED` and the target chip. Never call the `_ansi`/`_ae32`/`_aes3`/`_arp4` variant by hand unless you have a reason.
4. **Optimized code is target-gated** — `_ae32`=ESP32, `_aes3`=ESP32-S3, `_arp4`=ESP32-P4. `CONFIG_DSP_OPTIMIZATIONS_SUPPORTED` is only `y` for `IDF_TARGET_ESP32`, `_ESP32S3`, `_ESP32P4`, `_ESP32S31`. On other targets the build falls back to `CONFIG_DSP_ANSI`.
5. **Complex data is interleaved float** — `Re[0], Im[0], Re[1], Im[1], ...`. A 1024-point complex FFT needs a `float` buffer of length `2*N`. Bit-reverse and the real-split helpers operate in place on the same buffer.
6. **FFT length must be a power of two and ≤ `CONFIG_DSP_MAX_FFT_SIZE`** — verify with `dsp_is_power_of_two(N)`; the table is sized by the Kconfig `DSP_MAX_FFT_SIZE` choice (default 4096). Requesting a larger FFT returns `ESP_ERR_DSP_PARAM_OUTOFRANGE`.
7. **16-byte alignment matters** — examples declare buffers `__attribute__((aligned(16)))` because the ESP32-S3/ESP32 assembly paths and `dspi_*`/`dsps_*_aes3` expect aligned input. Heap allocations should use `memalign(16, ...)`.
8. **FIR/IIR/biquad are stateful** — pass the same `fir_f32_t` / `delay[]` / `w[]` across calls to keep the filter continuous. Re-initializing between blocks destroys history and produces clicks/edges.
9. **`CONFIG_DSP_MAX_FFT_SIZE` drives the sin/cos table** — `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)` allocates internally when the first arg is NULL; pass a preallocated buffer only if you want to control that memory.
10. **C++ `dspm::Mat` is header-gated** — `mat.h` is included by `esp_dsp.h` only under `__cplusplus`. Use `.cpp` source for Mat/EKF code and expose `extern "C" void app_main()`. The EKF classes (`ekf`, `ekf_imu13states`) are C++-only.
11. **Frequency arguments are normalized** — `dsps_tone_gen_f32` and the biquad generators take `freq` in `[-1..1]` / `[0..0.5]` relative to Nyquist / sample rate, not absolute Hz. Off-by-this is the #1 wrong-spectrum bug.
12. **Errors are `esp_err_t`** — most functions return `ESP_OK` or `ESP_ERR_DSP_*` (`ESP_ERR_DSP_PARAM_OUTOFRANGE`, `ESP_ERR_DSP_UNINITIALIZED`, `ESP_ERR_DSP_REINITIALIZED`, `ESP_ERR_DSP_ARRAY_NOT_ALIGNED`, `ESP_ERR_DSP_INVALID_LENGTH`, `ESP_ERR_DSP_INVALID_PARAM`). Always check the return value, especially from init functions.
13. **Filter state outlives coefficients** — in a streaming audio chain (bass/treble/volume adjusted at runtime), you may regenerate `coeffs[5]` with `dsps_biquad_gen_*_f32` on every knob change, but the delay line `w[]` must persist for the lifetime of the stream. Resetting `w[]` when you regenerate coeffs produces an audible click. The `dspi_conv_f32` 2D convolver has the mirror-image pitfall: it overwrites `out_image->size_x/size_y` (to `'same'` mode), so only read those fields after the call.

## When to Use

**Applicable:**
- Adding ESP-DSP to an ESP-IDF project (`idf.py add-dependency "espressif/esp-dsp"`)
- Computing forward/inverse FFT, radix-2 or radix-4, float (`fc32`) or fixed-point (`sc16`)
- Real-input FFT via the complex engine + `dsps_cplx2reC_fc32` / `dsps_cplx2real_fc32`
- Designing/applying FIR, decimation FIR (`fird`), multi-rate FIR (`firmr`), and the resampler
- Designing/applying IIR biquad filters (LPF/HPF/BPF/notch/shelf/peak/allpass) and stereo biquad
- Vector math: add/sub/mul/mulc/sqrt with `f32`/`s16`/`s8` and per-element step
- Matrix multiply / add / sub / mulc (`dspm_*`), the C++ `dspm::Mat` class and solvers (`solve`, `roots`, `inverse`, `det`)
- Generating test signals (tone, delta, complex LUT generator), windows (Hann/Blackman/Blackman-Harris/Blackman-Nuttall/Nuttall/flat-top), and measuring SNR/SFDR
- Convolution, correlation, 2D convolution (`dspi_conv_f32`), DCT/DCT-IV/DST-IV/inverse DCT
- Real-time streaming audio DSP once you have int16/float samples from I2S/codec: fixed-point `sc16` FFT spectrum (window → FFT → bit-rev → `dsps_cplx2reC_*` → dB → moving average) and streaming IIR effects chains (triple buffer + bass/treble biquad + volume + digital limiter)
- IMU attitude estimation with the 13-state Extended Kalman Filter (`ekf_imu13states`)

**Not applicable:**
- Bare-metal (non-ESP-IDF) builds — ESP-DSP is an IDF component and depends on `esp_err_t`, FreeRTOS, `esp_log`, `sdkconfig.h`
- Real-time audio I/O drivers / I2S configuration (use `driver/i2s` from ESP-IDF, then feed buffers to ESP-DSP)
- Neural-network inference (use ESP-RTOS / ESP-DL / TensorFlow Lite Micro instead)
- Other vendors' DSP libraries (CMSIS-DSP, etc.)

---

## Scenario Quick Reference (Recipes)

When the user intent matches a scenario below, **read the corresponding recipe first** — it carries the full call chain, real code, and a common-errors table.

### 项目集成 / Build & Integration

| recipe | scenario |
|---|---|
| `recipes/project_setup.md` | 把 esp-dsp 作为 ESP-IDF 组件加入项目、选 Kconfig 优化级别、最小可运行 app_main |

### 信号生成与显示

| recipe | scenario |
|---|---|
| `recipes/signal_generation.md` | 用 `dsps_tone_gen_f32` / `dsps_d_gen_f32` / `dsps_cplx_gen` 生成测试信号，`dsps_view`/`dsps_view_spectrum` 打印 |

### 变换 (FFT / DCT)

| recipe | scenario |
|---|---|
| `recipes/fft_complex.md` | 复数 radix-2 FFT：init → 加窗 → `dsps_fft2r_fc32` → bit-rev → `dsps_cplx2reC_fc32` → 功率谱 |
| `recipes/fft_real.md` | 实信号 FFT：radix-4 `dsps_fft4r_fc32` + `dsps_bit_rev4r_fc32` + `dsps_cplx2real_fc32` |
| `recipes/windows.md` | Hann/Blackman/Blackman-Harris/Nuttall/flat-top 窗函数生成与应用 |
| `recipes/dct.md` | DCT-II / DCT-IV / DST-IV / 逆 DCT |
| `recipes/conv2d_image.md` | 2D 卷积 `dspi_conv_f32`：`image2d_t` 的 `stride_x`/`step_x`/`step_y` 语义、`'same'` 输出尺寸、Sobel/边缘核、子采样 ROI |

### 滤波器

| recipe | scenario |
|---|---|
| `recipes/fir_filter.md` | 标准 FIR、FIR 抽取（`dsps_fird_*`）、多速率 FIR（`dsps_firmr_*`） |
| `recipes/iir_biquad.md` | IIR biquad：系数生成器（LPF/HPF/BPF/notch/shelf/peak）+ `dsps_biquad_f32` / 立体声 `dsps_biquad_sf32` |
| `recipes/resampler.md` | 多速率重采样器（`dsps_resampler_mr_*`）与多项式重采样器（`dsps_resampler_ph_*`） |

### 矩阵与向量

| recipe | scenario |
|---|---|
| `recipes/vector_math.md` | `dsps_add/sub/mul/mulc_f32`、`dsps_dotprod_f32`、`dsps_sqrt_f32` 与 step 参数 |
| `recipes/matrix.md` | `dspm_mult_f32` / C++ `dspm::Mat` 运算符重载、`solve`/`roots`/`inverse`/`det` |

### 高级

| recipe | scenario |
|---|---|
| `recipes/kalman_ekf.md` | 13 态 IMU 扩展卡尔曼滤波（`ekf_imu13states`）：标定 → Process → UpdateRefMeasurement |

### 应用 / 实时音频流

| recipe | scenario |
|---|---|
| `recipes/audio_spectrum_streaming.md` | 流式定点 FFT：I2S 麦克风 → `dsps_wind_blackman_harris_f32` 加窗 → `dsps_fft2r_sc16` + `dsps_bit_rev_sc16` + `dsps_cplx2reC_sc16` → dB → 滑动平均 → 显示（双任务） |
| `recipes/audio_iir_streaming.md` | 流式 IIR 音效：三缓冲（A/B/C）+ `dsps_biquad_f32` 串联 lowShelf/highShelf + `dsps_mulc_f32` 音量 + 数字限幅器；运行时重生成系数而延迟线 `w[]` 跨块保持 |

---

## Optimization / Chip Support Matrix

| Target | Optimized suffix | `CONFIG_DSP_OPTIMIZATIONS_SUPPORTED` | Notes |
|---|---|---|---|
| ESP32 | `_ae32` | yes | Xtensa LX6 assembly |
| ESP32-S3 | `_aes3` | yes | Xtensa LX7 + TIE (Q16 accx, memcpy/memset) |
| ESP32-P4 | `_arp4` | yes | RISC-V with HP-core vector extensions |
| ESP32-S31 | (as S3) | yes | added in 1.8.0 |
| ESP32-C3/C6/S2/etc. | — (ANSI) | no | falls back to `CONFIG_DSP_ANSI` |

> Toggle optimization in `menuconfig` → Component config → DSP Library → DSP Optimization → **Optimized** (default on supported targets) or **ANSI C** (portable, used for verification/debug).

---

## Key Kconfig Options

| Symbol | Type | Values | Effect |
|---|---|---|---|
| `CONFIG_DSP_OPTIMIZATIONS_SUPPORTED` | bool (auto) | y | Hidden; `y` only on ESP32/S3/P4/S31 |
| `CONFIG_DSP_OPTIMIZATION` | choice | `DSP_OPTIMIZED` / `DSP_ANSI` | Picks optimized assembly vs portable C |
| `CONFIG_DSP_MAX_FFT_SIZE` | choice | 512 / 1024 / 2048 / **4096** / 8192 / 16384 / 32768 | Sizes the internal sin/cos table; must be ≥ largest FFT you call |
| `CONFIG_DSP_MAX_FFT_SIZE` (int) | int | derived | Resolved numeric value passed to `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)` |

---

## Data Type Naming Convention

`dsp<domain>_<name>_<datatype>[_impl]`, where:

| Element | Meaning | Examples |
|---|---|---|
| domain | `s` signal, `i` image, `m` matrix, `q` fixed-length | `dsps_*`, `dspi_*`, `dspm_*` |
| datatype | `f32` float, `s16` int16, `s8` int8; `c` = complex, `e` = with step | `fc32`, `sc16`, `s16` |
| impl | `_ansi` portable · `_ae32` ESP32 · `_aes3` S3 · `_arp4` P4 | call without impl unless debugging |

---

## Critical Pitfalls (Must Read)

### 1. Calling FFT before `dsps_fft2r_init_fc32`

```c
// ❌ WRONG — table not initialized, returns ESP_ERR_DSP_UNINITIALIZED or crashes
dsps_fft2r_fc32(y_cf, N);

// ✅ CORRECT — init once, pass NULL to allocate the table internally
esp_err_t ret = dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);
if (ret != ESP_OK) {
    ESP_LOGE(TAG, "FFT init failed: %i", ret);
    return;
}
dsps_fft2r_fc32(y_cf, N);
```

### 2. Using the `_ansi`/`_aes3` suffix in application code

```c
// ❌ WRONG — hard-codes the implementation, loses target optimization
dsps_fft2r_fc32_aes3(y_cf, N);
dsps_biquad_f32_ansi(d, y, N, coeffs, w);

// ✅ CORRECT — the header #define maps to the right impl per target/Kconfig
dsps_fft2r_fc32(y_cf, N);
dsps_biquad_f32(d, y, N, coeffs, w);
```

### 3. FFT length exceeds `CONFIG_DSP_MAX_FFT_SIZE`

```c
// ❌ WRONG — N larger than the configured table -> ESP_ERR_DSP_PARAM_OUTOFRANGE
// (and default DSP_MAX_FFT_SIZE is 4096)
dsps_fft2r_fc32(y_cf, 8192);

// ✅ CORRECT — either stay within the limit, or raise the Kconfig value
// menuconfig: Component config -> DSP Library -> Maximum FFT length -> 8192
dsps_fft2r_fc32(y_cf, 1024);   // 1024 is a power of two and <= CONFIG_DSP_MAX_FFT_SIZE
```

### 4. Missing bit-reverse / real-split after FFT

```c
// ❌ WRONG — raw FFT output is bit-reversed; spectrum looks scrambled
dsps_fft2r_fc32(y_cf, N);
for (int i=0;i<N/2;i++) magnitude = sqrtf(y_cf[2*i]*y_cf[2*i] + y_cf[2*i+1]*y_cf[2*i+1]);

// ✅ CORRECT — bit-reverse, then split the complex result into two real spectra
dsps_fft2r_fc32(y_cf, N);
dsps_bit_rev_fc32(y_cf, N);
dsps_cplx2reC_fc32(y_cf, N);   // only when input was two real signals packed as one complex
```

### 5. Re-initializing the FIR/IIR state every block

```c
// ❌ WRONG — destroys filter history between blocks -> clicks / discontinuities
for (each block) {
    dsps_fird_init_f32(&fir, coeffs, delay, N, decim);  // reset!
    dsps_fird_f32(&fir, in, out, len);
}

// ✅ CORRECT — init once, reuse the same fir_f32_t / delay line across blocks
dsps_fird_init_f32(&fir, coeffs, delay, N, decim);
for (each block) {
    dsps_fird_f32(&fir, in, out, len);
}
```

### 6. Biquad `freq` argument treated as Hz

```c
// ❌ WRONG — freq is normalized to Nyquist, range [0..0.5], NOT Hz
dsps_biquad_gen_lpf_f32(coeffs, 1000.0f, 1.0f);   // 1000 is way out of range

// ✅ CORRECT — freq = cutoff_hz / sample_rate_hz
float f = 1000.0f / 48000.0f;   // ~0.0208, inside [0..0.5]
dsps_biquad_gen_lpf_f32(coeffs, f, 0.707f);
```

### 7. Calling `dspm::Mat` / EKF from a `.c` file

```c
// ❌ WRONG — mat.h and ekf.h are C++-only; esp_dsp.h includes mat.h only under __cplusplus
// main.c:
#include "esp_dsp.h"
dspm::Mat A(3,3);   // compile error

// ✅ CORRECT — use a .cpp source, expose app_main with C linkage
// main.cpp:
#include "esp_dsp.h"
extern "C" void app_main() {
    dspm::Mat A(3,3);
}
```

### 8. Mis-sized FIR delay line

```c
// ❌ WRONG — delay buffer shorter than coeffs (esp32s3 also wants 16-byte alignment + padding)
float delay[8];
dsps_fir_init_f32(&fir, coeffs, delay, 64);

// ✅ CORRECT — delay length = coeffs_len (+4 safe padding), aligned
__attribute__((aligned(16))) float delay_line[FIR_COEFFS_LEN + 4];
dsps_fir_init_f32(&fir1, fir_coeffs, delay_line, FIR_COEFFS_LEN);
```

### 9. Reusing the FFT table without deinit, or re-init after internal alloc

```c
// ❌ WRONG — second init returns ESP_ERR_DSP_REINITIALIZED because the table is already owned
dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);
dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);

// ✅ CORRECT — deinit first, or init only once
dsps_fft2r_deinit_fc32();
dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);
```

### 10. Forgetting 16-byte alignment for S3/P4 optimized paths

```c
// ❌ WRONG — aes3/arp3 paths and dspi_* expect 16-byte aligned buffers; misalign can fault or slow down
float x1[N_SAMPLES];

// ✅ CORRECT — align static buffers; use memalign(16, ...) for heap
__attribute__((aligned(16))) float x1[N_SAMPLES];
// or:
float *p = (float *)memalign(16, N * sizeof(float));
```

### 11. Storing a 1024-point *complex* FFT in a 1024-float buffer

```c
// ❌ WRONG — complex interleaved needs 2*N floats
float y_cf[N];
dsps_fft2r_fc32(y_cf, N);

// ✅ CORRECT — 2*N floats for N complex points
__attribute__((aligned(16))) float y_cf[N_SAMPLES * 2];
dsps_fft2r_fc32(y_cf, N);
```

### 12. `dsp_get_cpu_cycle_count` only on target

```c
// ❌ WRONG — assumes a symbol that is target-specific and only exists with esp_cpu
// (on Linux host builds it maps to __rdtsc)

// ✅ CORRECT — it is a macro defined by dsp_common.h; include the header and use it directly
#include "dsp_common.h"
unsigned int start_b = dsp_get_cpu_cycle_count();
dsps_fft2r_fc32(y_cf, N);
unsigned int end_b = dsp_get_cpu_cycle_count();
ESP_LOGI(TAG, "FFT took %u cycles", end_b - start_b);
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 识别场景（FFT？FIR？矩阵？EKF？），确认目标芯片（决定 `_aes3`/`_ae32`/`_arp4`/ANSI） |
| 2 | Recipe | 在 `recipes/` 中找到匹配场景，照其调用链编写 |
| 3 | Query | recipe 未覆盖的 API 查 `resources/api_reference.md`；Kconfig 查 `resources/config_reference.md` |
| 4 | Validate | 校验：FFT 已 init、buffer 对齐 16、复数 buffer 长度 2*N、N 为 2 的幂且 ≤ CONFIG_DSP_MAX_FFT_SIZE |
| 5 | Confirm | 向用户给出方案：includes、Kconfig、init 顺序、main 循环 |
| 6 | Execute | 新项目：用 `idf.py create-project-from-example "espressif/esp-dsp:<name>"` 或复制 `examples/<name>/`；已有项目：原地编辑 |
| 7 | Build | `idf.py set-target esp32s3` → `idf.py build`；menuconfig 选 DSP Optimization 与 Maximum FFT length |
| 8 | Flash | `idf.py -p PORT flash monitor` |
| 9 | Debug | 串口看 `dsps_view` 文本图与 ESP_LOG 周期计数；用 `dsps_snr_f32`/`dsps_sfdr_f32` 量化信号质量 |

### Step 6 Detail — Project Creation Strategy

**No existing project:** pick the closest official example and clone it:
- `idf.py create-project-from-example "espressif/esp-dsp:fft"` (or `fir`, `iir`, `matrix`, `basic_math`, `dotprod`, `kalman`, `fft4real`, `fft_window`, `conv2d`).
- Or copy `examples/<name>/` wholesale and edit `main/<...>_main.c|cpp`.

**Existing project:** add the dependency in place — `idf.py add-dependency "espressif/esp-dsp"` (writes `main/idf_component.yml`), then `#include "esp_dsp.h"` and call from your `app_main`.

---

## Failure Strategies

| Situation | Action |
|---|---|
| API not found in `resources/` | Stop; tell the user it does not exist in this version of esp-dsp |
| `ESP_ERR_DSP_UNINITIALIZED` from FFT | Call `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)` first |
| `ESP_ERR_DSP_PARAM_OUTOFRANGE` | FFT N too big — raise `CONFIG_DSP_MAX_FFT_SIZE` or shrink N |
| `ESP_ERR_DSP_REINITIALIZED` | Call `dsps_fft2r_deinit_fc32()` before re-init, or init once |
| `ESP_ERR_DSP_ARRAY_NOT_ALIGNED` | Add `__attribute__((aligned(16)))` or `memalign(16, ...)` |
| Spectrum looks wrong | Forgot `dsps_bit_rev_fc32` and/or `dsps_cplx2reC_fc32` |
| Filter clicks at block boundaries | Re-initializing `fir_f32_t`/delay each block — init once |
| Wrong cutoff frequency | biquad/tone `freq` is normalized, not Hz |
| C++ Mat/EKF won't compile | Source file is `.c` — rename to `.cpp`, declare `extern "C" void app_main()` |

## References

- 场景食谱 → `recipes/` 目录
- 完整 API 签名 → `resources/api_reference.md`
- Kconfig / 构建配置 → `resources/config_reference.md`
- 汇总避坑 → `resources/pitfalls.md`
- 官方示例索引 → `resources/example_list.md`
