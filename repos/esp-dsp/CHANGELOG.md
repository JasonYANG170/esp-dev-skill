# Changelog

本文件记录 esp-dsp-skill（ESP-DSP 的 AI Skill）的变更。

格式基于 [Keep a Changelog](https://keepachangelog.com/en/1.0.0/)。

## [1.1.0] - 2026-06-18

### Added
- 3 篇新食谱，补齐审计确认的真实缺口（每篇均以 `espressif/esp-dsp` 仓库的真实示例/头文件/实现为依据，未发明 API）：
  - `recipes/conv2d_image.md` — 2D 卷积 `dspi_conv_f32`：`image2d_t` 的 `stride_x`（行宽）/`step_x`/`step_y`（元素步长）字段语义、`'same'` 输出尺寸（函数覆写 `out_image->size_x/size_y`）、Sobel/边缘检测核、子采样 ROI。来源示例 `examples/conv2d/main/conv2d_main.c`（复现 Matlab `conv2(ones(8),ones(4),'same')`）、头文件 `modules/conv/include/dspi_conv.h`、结构 `modules/common/include/dsp_types.h`、实现 `modules/conv/float/dspi_conv_f32_ansi.c`。这是此前唯一没有对应食谱的示例目录。
  - `recipes/audio_spectrum_streaming.md` — 实时流式定点 FFT：I2S 麦克风 → `dsps_wind_blackman_harris_f32` 加窗 → `dsps_mul_s16`（shift=15）→ `dsps_fft2r_sc16` + `dsps_bit_rev_sc16` + `dsps_cplx2reC_sc16` → 绝对功率转 dB → 滑动平均 → LVGL 瀑布图，双任务 pin 到双核。来源应用 `applications/spectrum_box_lite/main/main.c`。补齐了此前食谱从未涉及的 sc16 定点 FFT、三缓冲生产者/消费者模式、滑动平均后处理。
  - `recipes/audio_iir_streaming.md` — 实时流式 IIR 音效：三缓冲（A/B/C）解耦处理任务与 codec 输出任务、`dsps_biquad_gen_lowShelf_f32`/`highShelf_f32` 运行时重生成系数、串联 `dsps_biquad_f32`（bass/treble）、`dsps_mulc_f32` 音量、自实现 `digitalLimiter` 数字限幅器、int16↔float 转换环绕滤波；强调延迟线 `w[]` 在系数变更时仍跨块保持。来源应用 `applications/lyrat_board_app/main/audio_amp_main.c`。

### Changed
- `SKILL.md`：
  - `metadata.version` 1.0.0 → 1.1.0。
  - Scenario Quick Reference 新增"应用 / 实时音频流"分组，收录 `audio_spectrum_streaming.md` 与 `audio_iir_streaming.md`；在"变换 (FFT / DCT)"分组追加 `conv2d_image.md`。
  - 新增 Core Principle #13"Filter state outlives coefficients"——流式音频链中可随时重生成 biquad 系数，但延迟线 `w[]` 必须跨块保持（重置会产生 click）；并附带 `dspi_conv_f32` 覆写输出 `size_x/size_y`（`'same'` 模式）的镜像避坑。
  - "When to Use / Applicable" 追加实时流式音频 DSP 一条（sc16 频谱 + 流式 IIR 音效链）。
- `resources/example_list.md`：为 `applications/spectrum_box_lite/` 与 `applications/lyrat_board_app/` 补注入口源文件（`main/main.c`、`main/audio_amp_main.c`）及与食谱的交叉引用。
- `resources/api_reference.md`：
  - 卷积/相关/2D 卷积小节追加 `dspi_conv_f32` 的 `'same'` 模式行为、边缘不补零、`stride_x` 是行宽而非子采样步长、目前无 ae32/aes3/arp4 优化版等说明。
  - FFT Radix-2 sc16 小节追加与 fc32 是两套独立表、以及到 `recipes/audio_spectrum_streaming.md` 的交叉引用。

### Grounding
- 新增食谱的全部函数（`dspi_conv_f32`、`dsps_fft2r_init_sc16`、`dsps_fft2r_sc16`、`dsps_bit_rev_sc16`、`dsps_cplx2reC_sc16`、`dsps_wind_blackman_harris_f32`、`dsps_mul_s16`、`dsps_biquad_f32`、`dsps_biquad_gen_lowShelf_f32`、`dsps_biquad_gen_highShelf_f32`、`dsps_mulc_f32`）、结构（`image2d_t`）与示例/应用路径（`examples/conv2d/`、`applications/spectrum_box_lite/`、`applications/lyrat_board_app/`）均来自 `espressif/esp-dsp` 仓库的 `modules/*/include/*.h`、`modules/conv/float/dspi_conv_f32_ansi.c`、`examples/*/main/`、`applications/*/main/` 与 `docs/en/`。

## [1.0.0] - 2026-06-18

### Added
- 初始版本，面向 Espressif ESP-DSP 库（仓库 `espressif/esp-dsp`）。
- `SKILL.md`：核心原则、适用场景、食谱速查表、芯片/Kconfig/数据类型命名参考表、12 条带正确/错误代码对照的关键避坑、执行工作流、失败策略。
- `AGENTS.md`：项目上下文、文件命名与 include 约定、标准项目结构、C 与 C++ 的 canonical init/main 模式、构建工作流、代码生成 checklist、Do-Not-Modify 说明。
- `recipes/`（12 篇）：`project_setup`、`signal_generation`、`fft_complex`、`fft_real`、`windows`、`dct`、`fir_filter`、`iir_biquad`、`resampler`、`vector_math`、`matrix`、`kalman_ekf`，每篇含适用摘要、触发意图、前置条件、分步代码、常见错误表、参考示例路径。
- `resources/api_reference.md`：按模块分组的真实 API 签名（FFT radix-2/4、DCT/DST、FIR/fird/firmr、resampler、biquad 与系数生成器、窗、信号生成/测量、向量数学、点积、矩阵、卷积/相关/2D、视图/内存、C++ `dspm::Mat`、C++ EKF）。
- `resources/config_reference.md`：Kconfig 选项、组件依赖、目标芯片与优化后缀映射、示例 CMake 注册、版本要点、函数命名约定。
- `resources/pitfalls.md`：17 条避坑条目（含 FFT 未 init、频率单位、buffer 对齐、C++ 入口、定点 FIR 倒序等）。
- `resources/example_list.md`：`examples/` 10 个示例与 `applications/` 开发板应用的路径与入口源文件索引。
- `README.md`（中文）与 `CHANGELOG.md`。

### Grounding
- 所有函数名、结构体、宏、Kconfig 符号、文件路径与代码片段均来自 `espressif/esp-dsp` 仓库的 `modules/*/include/*.h`、`Kconfig`、`idf_component.yml`、`examples/*/main/` 与 `docs/en/`。
