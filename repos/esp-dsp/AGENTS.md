# AGENTS.md — Supplementary Agent Guide

> 核心规则、模块速查表、避坑清单、recipe 索引、执行工作流都在 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未覆盖的工程约定与工具链说明，不重复内容。

## Project Context

**语言**: C 与 C++（矩阵 `dspm::Mat`、Kalman `ekf*` 类为 C++-only） · **目标**: Espressif ESP32 / ESP32-S3 / ESP32-P4（优化），其它 IDF 目标走 ANSI 回退 · **工具链**: ESP-IDF ≥ 4.2（xtensa / riscv-esp GCC）· **构建系统**: CMake + `idf_component_register`

## 代码生成约定

### 文件命名与语言选择

| 内容 | 文件扩展名 | 说明 |
|---|---|---|
| 信号 / FFT / 滤波器 / 向量数学（纯 C API） | `.c` | `dsps_*`、`dspm_mult_*`、`dspi_*` |
| `dspm::Mat`、`ekf`、`ekf_imu13states` | `.cpp` | `mat.h`、`ekf*.h` 仅在 `__cplusplus` 下可用 |
| `app_main` 入口（C++ 源内） | `.cpp` 中声明 `extern "C" void app_main()` | 官方 `matrix`、`kalman` 示例均如此 |

### Include Pattern

```c
/* 纯 C 用法 —— 拉入所有 dsps_*/dspm_*/dspi_* C 接口 */
#include "esp_dsp.h"

/* 仅需某个模块时也可单独 include */
#include "dsps_fft2r.h"
#include "dsps_fir.h"
#include "dsps_biquad.h"
#include "dsps_biquad_gen.h"
#include "dspm_mult.h"
```

```cpp
/* C++ 用法 —— esp_dsp.h 在 __cplusplus 下会额外 include "mat.h" */
#include "esp_dsp.h"
/* EKF 类单独 include（不在 esp_dsp.h 内） */
#include "ekf_imu13states.h"
```

`esp_dsp.h` 已经聚合的模块：dotprod、math（add/sub/mul/addc/mulc/sqrt）、fir、resampler、biquad、biquad_gen、wind、conv、corr、d_gen、h_gen、tone_gen、snr、sfdr、fft2r、fft4r、dct、matrix（dspm_matrix.h）、view、dspi_dotprod、dspi_conv。`mat.h` 与 EKF 类需要显式 include。

### 标准 ESP-IDF 项目结构（含 esp-dsp）

```
MyDspProject/
├── CMakeLists.txt                 # 顶层 idf cmake_minimum + include($ENV{IDF_PATH}/...)
├── main/
│   ├── CMakeLists.txt             # idf_component_register(SRCS "main.c" ...)
│   ├── idf_component.yml          # 由 idf.py add-dependency 生成，含 espressif/esp-dsp
│   ├── main.c                     # 或 main.cpp
│   └── ...
└── sdkconfig                      # 由 menuconfig 生成，含 CONFIG_DSP_OPTIMIZED / CONFIG_DSP_MAX_FFT_SIZE
```

依赖既可手动写入 `main/idf_component.yml`：

```yaml
dependencies:
  espressif/esp-dsp: "^1.5.2"
```

也可命令行添加：`idf.py add-dependency "espressif/esp-dsp"`。

### Canonical Init / Main Pattern

```c
/* 信号类最小 app_main —— 以 FFT 频谱为例 */
#include "esp_dsp.h"

#define N_SAMPLES 1024
__attribute__((aligned(16))) static float y_cf[N_SAMPLES * 2];

void app_main(void)
{
    /* 唯一需要 init 的模块是 FFT；NULL = 让库内部分配 sin/cos 表 */
    esp_err_t ret = dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE);
    if (ret != ESP_OK) {
        ESP_LOGE("main", "FFT init error %i", ret);
        return;
    }

    /* ... 填充 y_cf 为交错的 Re,Im,Re,Im ... */
    dsps_fft2r_fc32(y_cf, N_SAMPLES);     /* 扩展名自动映射到 aes3/ae32/arp4/ansi */
    dsps_bit_rev_fc32(y_cf, N_SAMPLES);
    dsps_cplx2reC_fc32(y_cf, N_SAMPLES);
    /* ... 计算 dB、dsps_view ... */

    dsps_fft2r_deinit_fc32();             /* 不再需要时释放内部表 */
}
```

C++ 矩阵示例入口约定（与 `examples/matrix/main/dspm_matrix_main.cpp` 一致）：

```cpp
#include "esp_dsp.h"
extern "C" void app_main();   /* 关键：C 链接 */
void app_main() {
    dspm::Mat A(3,3), x(3,1);
    dspm::Mat b = A * x;
    dspm::Mat x1 = dspm::Mat::solve(A, b);
}
```

### 调试输出与性能测量约定

- 日志：`ESP_LOGI/ESP_LOGE`（来自 `esp_log.h`，esp-dsp 自身也用）。
- 文本波形 / 频谱图：`dsps_view(data, len, width, height, min, max, '|')` 与 `dsps_view_spectrum(data, len, min, max)`，直接打印到串口。
- 周期计数：`dsp_get_cpu_cycle_count`（`dsp_common.h` 中的宏，按 IDF 版本映射到 `esp_cpu_get_cycle_count` / `esp_cpu_get_ccount` / `xthal_get_ccount`；Linux 宿主为 `__rdtsc`）。
- 信号质量量化：`dsps_snr_f32(input, len, use_dc)`、`dsps_sfdr_f32(input, len, use_dc)`（仅用于调试/单测，非实时路径）。

## Build Workflow

1. 设置目标芯片：`idf.py set-target esp32s3`（决定 `CONFIG_DSP_OPTIMIZED` 是否生效）。
2. 添加依赖（首次）：`idf.py add-dependency "espressif/esp-dsp"`。
3. 配置：`idf.py menuconfig` → Component config → DSP Library →
   - DSP Optimization = Optimized / ANSI C
   - Maximum FFT length = 512 … 32768（必须 ≥ 你要调用的最大 FFT 点数）
4. 编译：`idf.py build`。
5. 烧录监听：`idf.py -p PORT flash monitor`（`Ctrl-]` 退出）。
6. 从示例起步：`idf.py create-project-from-example "espressif/esp-dsp:fft"`（可换 `fir`/`iir`/`matrix`/`basic_math`/`dotprod`/`kalman`/`fft4real`/`fft_window`/`conv2d`）。

## ESP-DSP 代码生成 Checklist

- [ ] `#include "esp_dsp.h"`（C++ 矩阵/EKF 额外 include `mat.h` / `ekf_imu13states.h`）
- [ ] 用到 FFT → 在首次调用前 `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)`（radix-4 另加 `dsps_fft4r_init_fc32`）
- [ ] 检查每个返回 `esp_err_t` 的调用，特别是 init
- [ ] FFT 点数 N 为 2 的幂且 ≤ `CONFIG_DSP_MAX_FFT_SIZE`
- [ ] 复数 FFT 的 buffer 长度为 `2*N`（交错 Re,Im）
- [ ] buffer `__attribute__((aligned(16)))` 或 `memalign(16, ...)`
- [ ] 调用扩展名无关的名字（`dsps_fft2r_fc32`，而非 `..._aes3`）
- [ ] biquad / tone 的 `freq` 已归一化到 Nyquist / 采样率
- [ ] FIR/IIR 的 `fir_f32_t` / `delay[]` / `w[]` 在分块处理时复用（不要每块重新 init）
- [ ] FFT 后接 `dsps_bit_rev_fc32`，必要时再 `dsps_cplx2reC_fc32` / `dsps_cplx2real_fc32`
- [ ] C++ 源里的 `app_main` 用 `extern "C"` 声明
- [ ] EKF 在 `Process` / `UpdateRefMeasurement` 前调用 `Init()`

## Do Not Modify

- `managed_components/espressif__esp-dsp/` 下由 Component Manager 拉取的库源码（`modules/`、`include/`、`test/`）—— 不要直接改，应通过升级依赖版本或向上游提 PR。
- `sdkconfig.defaults` 中由 menuconfig 生成的 `CONFIG_DSP_*` 项应通过 `idf.py menuconfig` 修改，避免手写出不一致的值。
