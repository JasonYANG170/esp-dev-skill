# ESP-DSP 官方示例索引

> 全部路径来自仓库 `examples/` 目录与 `examples/README.md`。每个示例都是独立 ESP-IDF 工程，可用 `idf.py create-project-from-example "espressif/esp-dsp:<name>"` 拉取，或直接复制 `examples/<name>/`。

## 信号处理示例（`examples/`）

| 路径 | 入口源文件 | 说明 |
|---|---|---|
| `examples/basic_math/` | `main/dsps_math_main.c` | 演示基本向量数学：用标准 C 循环加窗做 FFT，对比用 `dsps_mul_f32` / `dsps_mulc_f32` 加窗做 FFT |
| `examples/dotprod/` | `main/dsps_dotproduct_main.c` | 演示 `dsps_dotprod_f32` 点积：初始化数组、计算 0..100 求和、测周期数 |
| `examples/fft/` | `main/dsps_fft_main.c` | 复数 radix-2 FFT：两路正弦（0 dB / -20 dB）打包为复数 → 加窗 → FFT → bit-rev → `dsps_cplx2reC_fc32` → 功率谱 |
| `examples/fft4real/` | `main/dsps_fft4real_main.c` | 实信号 FFT：对比 radix-2 与 radix-4（`dsps_fft4r_fc32`） |
| `examples/fft_window/` | `main/dsps_window_main.c` | 窗 + FFT：Hann 窗与 FFT 配合，演示频谱 |
| `examples/fir/` | `main/dsps_fir_main.c` | FIR 滤波：windowed-sinc 生成系数、`dsps_fird_init_f32` + `dsps_fird_f32` 抽取滤波，含 FFT 频响对比 |
| `examples/iir/` | `main/dsps_iir_main.c` | IIR biquad：`dsps_biquad_gen_lpf_f32` 生成系数、`dsps_biquad_f32` 滤 delta 信号，显示冲激与频响 |
| `examples/matrix/` | `main/dspm_matrix_main.cpp` | C++ `dspm::Mat`：构造 A、x，`A*x`，`Mat::solve` / `Mat::roots` 求解 |
| `examples/kalman/` | `main/ekf_imu13states_main.cpp` | 13 态 IMU EKF：仿真陀螺/加速度/磁力计，标定阶段 + 运行阶段，估计陀螺偏差与欧拉角 |
| `examples/conv2d/` | `main/conv2d_main.c` | 2D 卷积：`dspi_conv_f32` 对 8x8 图像卷 4x4 核，输出 10x10 |

## 应用示例（`applications/`）

`applications/README.md` 列出的完整应用（需配合对应开发板）：

| 路径 | 说明 |
|---|---|
| `applications/spectrum_box_lite/` | ESP32-S3-BOX-Lite 频谱盒 demo |
| `applications/spectrum_box_lite/main/main.c` | 入口源文件：`dsps_fft2r_init_sc16` + 双声道 I2S + `dsps_wind_blackman_harris_f32` + `dsps_fft2r_sc16` + `dsps_cplx2reC_sc16` + dB + 滑动平均 + LVGL 瀑布图（见 `recipes/audio_spectrum_streaming.md`） |
| `applications/lyrat_board_app/` | ESP32-LyraT 板音频放大器应用 |
| `applications/lyrat_board_app/main/audio_amp_main.c` | 入口源文件：三缓冲 + `dsps_biquad_gen_lowShelf_f32`/`highShelf_f32` + 串联 `dsps_biquad_f32` + `dsps_mulc_f32` 音量 + `digitalLimiter` + int16↔float（见 `recipes/audio_iir_streaming.md`） |
| `applications/azure_board_apps/` | Azure IoT 板应用集（含 `3d_graphics`、`kalman_filter`） |
| `applications/azure_board_apps/apps/3d_graphics/` | 3D 图形（矩阵/渲染）demo |
| `applications/azure_board_apps/apps/kalman_filter/` | Kalman 滤波 demo |
| `applications/m5stack_core_s3/apps/3d_graphics/` | M5Stack CoreS3 3D 图形 demo |
| `applications/m5stack_core_s3/apps/kalman_filter/` | M5Stack CoreS3 Kalman demo |

> 应用示例依赖对应开发板硬件与外设驱动，移植前请先阅读各 `README.md`。

## 示例通用说明（`examples/README.md`）

- 这些示例用于演示 esp-dsp 各模块的初始化与执行，代码可直接复制改造到自有项目。
- 示例按类别分子目录组织，每个子目录是一个独立 ESP-IDF 工程。
- 构建/烧录与普通 IDF 工程一致：`idf.py -p PORT flash monitor`。
- 在 menuconfig 的 Component config → DSP Library → DSP Optimization 可切换 Optimized / ANSI 实现做对比。
