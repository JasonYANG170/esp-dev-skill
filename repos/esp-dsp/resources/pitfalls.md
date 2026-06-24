# ESP-DSP 避坑汇总

> 汇总 `SKILL.md` “Critical Pitfalls” 与各 recipe 常见错误。每条给出错误原因与正确做法，全部基于真实 API。

## 1. FFT 未初始化

- **现象**：`ESP_ERR_DSP_UNINITIALIZED` 或崩溃。
- **原因**：未调用 init 就调用 `dsps_fft2r_fc32`。
- **解决**：首次使用前 `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)`；用 radix-4 再加 `dsps_fft4r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)`。

## 2. 应用代码硬编码实现后缀

- **现象**：换目标后性能下降或链接失败。
- **原因**：直接调 `dsps_fft2r_fc32_aes3` / `dsps_biquad_f32_ansi`。
- **解决**：调用扩展名无关形式（`dsps_fft2r_fc32`、`dsps_biquad_f32`），由头文件宏按目标/Kconfig 自动映射。

## 3. FFT 点数超过 `CONFIG_DSP_MAX_FFT_SIZE`

- **现象**：`ESP_ERR_DSP_PARAM_OUTOFRANGE`。
- **解决**：menuconfig 调大 Maximum FFT length，或缩小 N。N 必须是 2 的幂（radix-2）。

## 4. 漏做 bit-reverse / 实数拆分

- **现象**：频谱杂乱、第二路信号取不到。
- **解决**：FFT 后必须 `dsps_bit_rev_fc32`；若输入是两路实信号打包为一路复数，还需 `dsps_cplx2reC_fc32`；radix-4 实输入用 `dsps_cplx2real_fc32`。

## 5. 每块重新 init 滤波器状态

- **现象**：滤波输出有咔哒声、块边界不连续。
- **原因**：每块都 `dsps_fird_init_f32` / 重置 `w[]`。
- **解决**：`fir_f32_t`、`delay[]`、biquad 的 `w[]` 只 init/清零一次，跨块复用。

## 6. biquad / tone 的频率当成 Hz

- **现象**：截止频率完全错。
- **原因**：`freq` 是归一化值（`[0..0.5]` 相对采样率，`[-1..1]` 相对 Nyquist），不是 Hz。
- **解决**：`f = fc_hz / fs_hz`。

## 7. C++ 类（Mat / EKF）写在 `.c` 文件

- **现象**：编译错误（`dspm::Mat` 找不到、`mat.h` 未被 include）。
- **原因**：`mat.h`、`ekf*.h` 仅 C++ 可用；`esp_dsp.h` 只在 `__cplusplus` 下 include `mat.h`。
- **解决**：源文件改 `.cpp`，`app_main` 用 `extern "C" void app_main()`。

## 8. FIR 延迟线过短

- **现象**：内存越界 / esp32s3 上结果错乱。
- **原因**：`delay` 短于系数长度，或未 16 字节对齐。
- **解决**：`delay` 长度 ≥ `coeffs_len + 4`，`__attribute__((aligned(16)))` 或 `memalign(16, ...)`。

## 9. FFT 表重复 init

- **现象**：`ESP_ERR_DSP_REINITIALIZED`。
- **原因**：第二次 `dsps_fft2r_init_fc32(NULL, ...)`（内部已分配过表）。
- **解决**：先 `dsps_fft2r_deinit_fc32()` 再 init，或全局只 init 一次。

## 10. buffer 未 16 字节对齐

- **现象**：`ESP_ERR_DSP_ARRAY_NOT_ALIGNED`，或 aes3/arp4 路径性能骤降甚至 fault。
- **解决**：静态 buffer 加 `__attribute__((aligned(16)))`；堆用 `memalign(16, size)`（见 `conv2d` 示例）。

## 11. 复数 FFT buffer 长度当 N

- **现象**：越界读/写。
- **原因**：复数交错（Re,Im,Re,Im）需要 `2*N` 个 float。
- **解决**：`float y_cf[N * 2];`，传 `N` 给函数。

## 12. radix-4 用错 bit-reverse

- **现象**：结果错乱。
- **原因**：radix-4 用了 `dsps_bit_rev_fc32`。
- **解决**：radix-4 必须用 `dsps_bit_rev4r_fc32`；实输入还原用 `dsps_cplx2real_fc32`。

## 13. `dsps_dct_f32` 数组大小用错

- **现象**：越界。
- **原因**：DCT-II/逆 DCT 要求 `data` 大小为 `2*N`；DCT-IV/DST-IV 为 `N`。
- **解决**：按函数要求分配；DCT 基于 FFT，仍需先 init FFT 表。

## 14. 定点 FIR 在 esp32s3 上结果错

- **现象**：`dsps_fird_s16_aes3` 输出错误。
- **原因**：aes3 要求系数倒序，且 coeffs_len 能被 4 整除、16 字节对齐。
- **解决**：init 后调 `dsps_16_array_rev(coeffs, len)`；用 `dsps_fird_s16_aexx_free(&fir)` 释放。

## 15. `dsp_get_cpu_cycle_count` 用法

- **现象**：找不到符号 / 宿主编译失败。
- **原因**：它是 `dsp_common.h` 的宏，按 IDF 版本与目标映射（`esp_cpu_get_cycle_count` / `esp_cpu_get_ccount` / `xthal_get_ccount`，Linux 为 `__rdtsc`）。
- **解决**：`#include "dsp_common.h"` 后直接当函数用即可。

## 16. EKF 未 Init / 单位错误

- **现象**：崩溃或姿态不收敛。
- **原因**：构造后漏 `Init()`；`accel` 未换算为 g（1g≈9.81m/s²）、陀螺未换算为 rad/sec。
- **解决**：先 `ekf13->Init()`；运行前用 `UpdateRefMeasurementMagn` 标定，再切 `UpdateRefMeasurement`。

## 17. 重采样器速率算反 / 漂移

- **现象**：输出样本数偏离期望。
- **原因**：`samplerate_factor = out_rate / in_rate`；源/宿时钟不同源时长期漂移。
- **解决**：确认 factor 方向；用 `length_correction`（+ 提高输出速率、- 降低）持续修正。
