# esp-dsp-skill

面向 **Espressif ESP-DSP** 官方 DSP 库的 AI Skill（Claude Code / Agent skill 格式）。本 Skill 让 AI 助手能够**完全基于真实仓库文档与代码**，正确地用 esp-dsp 在 ESP-IDF 项目中开发信号处理、滤波、矩阵运算与卡尔曼滤波的固件代码——绝不臆造 API。

ESP-DSP 是 Espressif 官方 DSP 库，为 ESP32 / ESP32-S3 / ESP32-P4 提供优化实现（Xtensa/RISC-V 汇编），并为其它 IDF 目标提供可移植 ANSI C 回退。覆盖 FFT（radix-2 / radix-4）、FIR / 抽取 FIR / 多速率 FIR、IIR biquad、重采样器、向量数学、矩阵运算、C++ `dspm::Mat`、2D 卷积、DCT/DST，以及 13 态 IMU 扩展卡尔曼滤波。

## 特性

- **场景食谱（recipes）**：12 个中文食谱，覆盖项目集成、信号生成、复数/实数 FFT、窗函数、DCT、FIR/IIR、重采样、向量数学、矩阵、Kalman EKF——每篇都含真实调用链、可复制代码与常见错误表。
- **真实 API 参考**：`resources/api_reference.md` 中所有函数签名、结构体、宏均摘自 `modules/*/include/*.h`。
- **Kconfig / 构建指南**：`resources/config_reference.md` 详列 `CONFIG_DSP_OPTIMIZED`、`CONFIG_DSP_MAX_FFT_SIZE` 等真实选项与目标芯片映射。
- **避坑汇总**：`resources/pitfalls.md` 收录 17 条最容易踩的坑（FFT 未 init、频率单位、buffer 对齐、C++ 入口等）。
- **官方示例索引**：`resources/example_list.md` 列出 `examples/` 与 `applications/` 下全部真实示例路径与入口源文件。
- **零臆造**：找不到的 API 一律不写。

## 安装

将本 Skill 克隆/复制到 Claude Code 的 skills 目录（项目级或用户级）：

- 项目级：`<project>/.claude/skills/esp-dsp-skill/`
- 用户级：`~/.claude/skills/esp-dsp-skill/`

目录结构需保持：

```
esp-dsp-skill/
├── SKILL.md            # 入口（YAML frontmatter + 主体）
├── AGENTS.md           # 工程约定补充
├── recipes/            # 12 个场景食谱
├── resources/          # api_reference / config_reference / pitfalls / example_list
├── README.md
└── CHANGELOG.md
```

Claude Code 会自动发现 `SKILL.md` 中的 `description` 与 trigger words，并在用户提到 ESP-DSP、FFT、FIR/IIR、矩阵、Kalman、ESP32-S3 DSP 等场景时加载本 Skill。

## 支持范围

- **目标芯片**：ESP32（`_ae32`）、ESP32-S3（`_aes3`）、ESP32-P4（`_arp4`）、ESP32-S31；其它 IDF 目标走 ANSI 回退。
- **IDF 版本**：≥ 4.2（`idf_component.yml` 要求）。
- **构建系统**：ESP-IDF CMake（作为 component）。
- **语言**：C（绝大多数 API）与 C++（`dspm::Mat` 与 EKF 类）。

不在范围内：裸机（非 IDF）构建、I2S/音频驱动本身、神经网络推理、其它厂商 DSP 库。

## 许可

ESP-DSP 库本身为 Apache-2.0（见仓库 `LICENSE`）。本 Skill 文档同样可按 Apache-2.0 使用。
