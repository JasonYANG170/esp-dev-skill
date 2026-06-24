# ESP-DSP 配置与构建参考

> 全部内容来自仓库 `Kconfig`、`idf_component.yml`、`CMakeLists.txt` 与示例 `README.md`。

## Kconfig 选项（`menuconfig` → Component config → DSP Library）

| 符号 | 类型 | 取值 | 默认 | 作用 |
|---|---|---|---|---|
| `CONFIG_DSP_OPTIMIZATIONS_SUPPORTED` | bool（隐藏） | y/n | 自动 | 仅 `IDF_TARGET_ESP32 / _ESP32S3 / _ESP32P4 / _ESP32S31` 时为 y |
| `CONFIG_DSP_ANSI` | bool（choice 项） | — | — | 选中 = 用纯 ANSI C 实现（可移植、用于验证/调试） |
| `CONFIG_DSP_OPTIMIZED` | bool（choice 项） | — | 在支持的目标上默认选中 | 用目标优化汇编（`_ae32`/`_aes3`/`_arp4`） |
| `CONFIG_DSP_OPTIMIZATION` | int（派生） | 0 / 1 | — | 0=ANSI, 1=Optimized；驱动头文件中的 `#if CONFIG_DSP_OPTIMIZED` |
| `CONFIG_DSP_MAX_FFT_SIZE_512` ... `_32768` | bool（choice 项） | — | `_4096` 选中 | 选择最大 FFT 长度 |
| `CONFIG_DSP_MAX_FFT_SIZE` | int（派生） | 512/1024/2048/4096/8192/16384/32768 | 4096 | 传入 `dsps_fft2r_init_fc32(NULL, CONFIG_DSP_MAX_FFT_SIZE)`；决定内部 sin/cos 表大小 |

### 关键约束

- `CONFIG_DSP_OPTIMIZED` **仅在受支持目标上可选**。在其它目标（ESP32-C3/S2/C6 等）上，menuconfig 只显示 ANSI 项。
- 若未定义 `CONFIG_DSP_MAX_FFT_SIZE`，`dsps_fft2r.h` 会 `#define CONFIG_DSP_MAX_FFT_SIZE 4096` 作为兜底。
- FFT 点数 N 必须 ≤ `CONFIG_DSP_MAX_FFT_SIZE` 且为 2 的幂（radix-2）/ 4 的幂（radix-4），否则返回 `ESP_ERR_DSP_PARAM_OUTOFRANGE`。

## 组件依赖（`idf_component.yml`）

```yaml
# 仓库根 idf_component.yml 摘录
description: ESP-DSP is the official DSP library for Espressif SoCs.
dependencies:
  idf: ">=4.2"        # 要求 IDF >= 4.2
```

项目内添加：

```bash
idf.py add-dependency "espressif/esp-dsp"
# 或手动写入 main/idf_component.yml:
# dependencies:
#   espressif/esp-dsp: "^1.5.2"
```

## 目标芯片与优化后缀

| 目标 | 优化后缀 | 说明 |
|---|---|---|
| ESP32 | `_ae32` | Xtensa LX6 汇编 |
| ESP32-S3 | `_aes3` | Xtensa LX7 + TIE（含 Q16/accx、优化 memcpy/memset） |
| ESP32-P4 | `_arp4` | RISC-V HP 核向量扩展 |
| ESP32-S31 | （同 S3） | 1.8.0 新增 |
| 其它 | （无，ANSI） | 自动回退 `CONFIG_DSP_ANSI` |

> 扩展名无关的 API（如 `dsps_fft2r_fc32`）由头文件按 `CONFIG_DSP_OPTIMIZED` + 各 `*_enabled` 平台宏映射到具体实现。应用代码应始终调用扩展名无关的形式。

## 示例 main 组件注册

每个示例 `main/CMakeLists.txt` 仅一行：

```cmake
# C 示例
idf_component_register(SRCS "dsps_fft_main.c")

# C++ 示例（matrix / kalman）
idf_component_register(SRCS "dspm_matrix_main.cpp")
idf_component_register(SRCS "ekf_imu13states_main.cpp")
```

esp-dsp 组件本身由 Component Manager 自动作为依赖拉入，无需在示例 CMake 里显式 `REQUIRES`。

## 从示例创建项目

```bash
idf.py create-project-from-example "espressif/esp-dsp:fft"
# 可用名称：basic_math conv2d dotprod fft fft4real fft_window fir iir kalman matrix
```

## 版本要点（来自 `CHANGELOG.md`）

| 版本 | 关键变化 |
|---|---|
| 1.8.2 / 1.8.1 | int8 行点积、int8 矩阵×向量 |
| 1.8.0 | 支持 esp32s31 |
| 1.7.0 | 多速率 FIR、基于多速率 FIR 的 resampler |
| 1.6.x | 立体声 biquad `dsps_biquad_sf32`、DCT-IV/DST-IV、FFT2R/FFT4R 改进、bug 修复 |
| 1.5.x | 支持 esp32p4、2D 卷积、定点 add/mul/sub |
| 1.4.x | FIR f32 decimation（esp32s3 优化）、`dsps_fird_init_f32` 移除 `start_pos`、系数改倒序、Mat 子矩阵、demo 应用 |
| 1.3.0 | 定点 FIR 抽取、`dsp_power_of_two` 扩到 32 位、放弃 IDF 4.0/4.1 |
| 1.2.0 | (历史版本) |

## 函数命名约定速查

`dsp<domain>_<name>_<datatype>[_impl]`

- **domain**: `s` 信号 / `i` 图像 / `m` 矩阵 / `q` 定长 / `r` 渲染
- **datatype**: `f` float / `s` signed / `u` unsigned；`c` = complex；`e` = 带 step；`16`/`32` 位宽
  - 例：`fc32` = float complex 32 位，`sc16` = signed complex 16 位，`s16` = signed 16 位
- **impl**: `_ansi`（可移植）/ `_ae32`（ESP32）/ `_aes3`（S3）/ `_arp4`（P4）；调用时省略

例：`dsps_mac_sc16` = 信号域 mac 操作，16 位有符号复数；`dspm_mult_f32` = 矩阵域乘法，float 32 位。
