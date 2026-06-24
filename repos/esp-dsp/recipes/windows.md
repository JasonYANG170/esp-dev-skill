# 窗函数生成与应用

> **适用摘要**: 用 `dsps_wind_*_f32` 生成 Hann / Blackman / Blackman-Harris / Blackman-Nuttall / Nuttall / flat-top 窗，并将其与信号相乘（可用 `dsps_mul_f32` 完成基本运算版本）。

## 触发意图

- "加窗 / 窗函数"
- "Hann / Blackman / Nuttall"
- "FFT 频谱泄漏"
- "窗系数"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/fft_window/main/dsps_window_main.c`、`examples/basic_math/main/dsps_math_main.c` |
| 头文件 | `esp_dsp.h`（含 `dsps_wind.h`，已聚合所有窗） |

## 分步说明

### 生成各种窗（签名相同）

所有窗函数签名一致：`void dsps_wind_<type>_f32(float *window, int len);`

```c
#include "esp_dsp.h"

#define N 1024
__attribute__((aligned(16))) static float wind[N];

dsps_wind_hann_f32(wind, N);             /* Hann */
dsps_wind_blackman_f32(wind, N);         /* Blackman, alpha=0.16 */
dsps_wind_blackman_harris_f32(wind, N);  /* Blackman-Harris */
dsps_wind_blackman_nuttall_f32(wind, N); /* Blackman-Nuttall */
dsps_wind_nuttall_f32(wind, N);          /* Nuttall */
dsps_wind_flat_top_f32(wind, N);         /* flat-top */
```

### 应用窗：手写循环

```c
for (int i = 0; i < N; i++) {
    y_cf[i * 2 + 0] = x1[i] * wind[i];   /* 实部 */
    y_cf[i * 2 + 1] = 0;                 /* 虚部 */
}
```

### 应用窗：用基本数学函数（更“DSP”，可对照优化）

`basic_math` 示例同时演示两种写法并比较结果：

```c
/* 把信号乘以窗，结果交错存为复数的实部（step_out=2） */
dsps_mul_f32(x1, wind, y_cf, N, 1, 1, 2);
/* 虚部清零：对 &y_cf[1] 起始、step 2，乘常数 0 */
dsps_mulc_f32(&y_cf[1], &y_cf[1], N, 0, 2, 2);
```

### 窗类型选择速查

| 窗 | 主瓣宽 | 旁瓣衰减 | 适用场景 |
|---|---|---|---|
| Hann | 中 | ~31 dB | 通用频谱分析 |
| Blackman | 较宽 | ~58 dB | 需要更低旁瓣 |
| Blackman-Harris | 宽 | ~92 dB | 高动态范围测量 |
| Blackman-Nuttall | 宽 | ~93 dB | 近似 Blackman-Harris |
| Nuttall | 宽 | ~93 dB | 同上 |
| flat-top | 最宽 | ~44 dB | 幅度精度优先（频率分辨率次要） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 加窗后幅度变小 | 未做幅度补偿 | 窗会改变增益，按窗的相干增益补偿（如 Hann ≈ ×2） |
| 频谱仍有泄漏 | 窗长度与 FFT 点数不一致 | 窗 `len` 必须等于 FFT 点数 N |
| `dsps_mul_f32` 结果错位 | step 参数用错 | 复数交错时 `step_out=2`，源 step 为 1 |
| 用了不存在的窗名 | 拼写错误 | 仅以上 6 种窗存在；无 `hamming`/`kaiser` |

## 参考

- `examples/fft_window/main/dsps_window_main.c` — Hann 窗 + FFT 示例
- `examples/basic_math/main/dsps_math_main.c` — 用 `dsps_mul_f32`/`dsps_mulc_f32` 应用窗
- `modules/windows/include/dsps_wind.h` 及各子目录头（hann/blackman/blackman_harris/blackman_nuttall/nuttall/flat_top）
