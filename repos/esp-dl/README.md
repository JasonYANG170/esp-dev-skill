# esp-dl-skill

面向 [ESP-DL](https://github.com/espressif/esp-dl) 深度学习推理框架的 AI Skill（Claude Code / Agent skill 格式）。ESP-DL 是 Espressif 为 ESP 系列芯片设计的轻量高效神经网络推理框架，提供加载、调试、运行 AI 模型（`.espdl`）的 API，以及 ESP-PPQ 量化工具链（把 ONNX / PyTorch 模型量化为 ESP-DL 标准的 FlatBuffers 模型格式）。

本 Skill 让 AI Agent 在**完全基于真实仓库文档与代码**的前提下，正确使用 ESP-DL 进行模型量化、部署、推理、调试与性能剖析——所有 API、结构体、宏、Kconfig、量化参数、示例路径均取自 `esp-dl` 仓库，不臆造。

## 功能特性

- **场景化食谱（recipes）**：覆盖工程集成、三种模型加载方式（rodata/分区/SD 卡）、推理流程、流式模型、测试与剖析、ONNX/PyTorch/AutoQuant 量化、视觉检测端到端。
- **真实 C++ API 参考**：`dl::Model` / `dl::TensorBase` / `dl::ExponentInfo` / 内存管理 / 视觉图像 API，全部来自头文件。
- **配置参考**：ESP-DL Kconfig（`PIX_CVT_*`）、组件清单字段、构造参数、ESP-PPQ 量化参数、AutoQuant 搜索设置。
- **陷阱汇总**：20 条真实坑点（量化/反量化方向、内存复用、16 字节对齐、target 与芯片匹配等）。
- **示例索引**：仓库内全部真实示例路径与一句话描述。
- **芯片支持表**：各 ESP 芯片的 PIE 指令、舍入策略、量化策略与 ESP-IDF 版本要求。

## 支持范围

- 目标芯片：ESP32、ESP32-S3、ESP32-P4、ESP32-C2/C3/C5/C6、ESP32-S2、ESP32-S31
- ESP-IDF：`>=5.3`（C5 `>=5.5`，S31 `>=6.0`）
- 量化：ESP-PPQ（Python，`pip install esp-ppq`）
- 语言：固件侧 C++，量化侧 Python

## 安装

把本目录放入 Claude Code 的 skills 目录之一即可被自动发现：

- 项目级：`<你的工程>/.claude/skills/esp-dl-skill/`
- 用户级：`~/.claude/skills/esp-dl-skill/`

或直接克隆到对应位置：

```bash
git clone <this-repo> ~/.claude/skills/esp-dl-skill
```

安装后，在对话中提到 ESP-DL / espdl / 模型量化 / 部署 / 推理 等关键词时，Skill 即被触发。

## 目录结构

```
esp-dl-skill/
├── SKILL.md              # 主入口：原则、适用场景、食谱索引、芯片表、关键陷阱、执行流程
├── AGENTS.md             # 补充约定：项目结构、include 模式、构建流程、代码生成清单
├── recipes/              # 场景化食谱（中文）
│   ├── project_setup.md
│   ├── load_model_rodata.md
│   ├── load_model_partition.md
│   ├── load_model_sdcard.md
│   ├── run_inference.md
│   ├── streaming_model.md
│   ├── profile_test.md
│   ├── quantize_onnx.md
│   ├── quantize_torch.md
│   ├── auto_quant.md
│   └── vision_detection.md
├── resources/            # 快速参考
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md             # 本文件
└── CHANGELOG.md
```

## 许可证

随 ESP-DL 仓库，MIT。
