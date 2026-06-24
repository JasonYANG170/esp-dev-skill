# esp-detection-skill

面向 Espressif **esp-detection** 仓库的 AI Skill（Claude Code / Agent skill 格式）。esp-detection 是一个基于 Ultralytics YOLOv11 的轻量级、ESP 优化的实时目标检测项目，支持在 ESP32-P4 / ESP32-S3 上完成「训练 → 导出 → 量化 → 部署推理」全流程。

本 Skill 让 AI 代理在**完全基于仓库真实文档与源码**的前提下，正确地进行 esp-detection 软件开发：准备数据集、训练 espdet_pico、导出 ONNX、量化为 ESP-DL `.espdl`、在芯片上运行推理。所有 API、结构体、宏、配置项、代码片段均来自仓库源码，绝不臆造。

## 功能特性

- **场景化 recipes**：覆盖环境搭建、数据集准备、方形/rect 训练、ONNX 导出、INT8 量化、量化评估、一站式流水线、芯片部署共 9 个真实场景。
- **完整 API 参考**：Python 训练/导出/量化函数签名、自定义网络模块（`DSConv`/`ESPBlock`/`ESPDetect` 等）、芯片端 C++ 封装（`ESPDetDetect`/`ESPDet`）。
- **配置参考**：模型 YAML、数据集 YAML、CLI 参数、Kconfig、sdkconfig、分区表、组件依赖。
- **流水线状态机**：从数据到芯片推理的阶段产物与约束一目了然。
- **陷阱汇总**：26 条按主题归类的常见错误与解决方法。
- **关键陷阱带 WRONG/CORRECT 代码对比**：12 条核心陷阱逐条演示。

## 支持范围

| 维度 | 范围 |
|---|---|
| 训练/量化环境 | Python 3.8 + PyTorch 2.2.0 + Ultralytics ≥ 8.3.112 + esp-ppq |
| 目标芯片 | ESP32-P4、ESP32-S3 |
| 芯片工具链 | ESP-IDF release/v5.3+（模板基于 5.4.0） |
| 推理后端 | ESP-DL（`.espdl` INT8） |
| 任务类型 | 目标检测（单类/多类） |
| License | 仓库 AGPL-3.0；芯片端模型模板 MIT |

## 安装

### 方式一：克隆到项目级 skills 目录

```bash
git clone <this-repo> D:/esp-skill/skills/esp-detection-skill
# 或把整个 esp-detection-skill 目录放入项目的 .claude/skills/
```

### 方式二：用户级 skills 目录（全局可用）

把 `esp-detection-skill/` 复制到：

- Windows: `%USERPROFILE%\.claude\skills\esp-detection-skill\`
- macOS / Linux: `~/.claude/skills/esp-detection-skill/`

Claude Code 启动时会自动发现并加载该 Skill。

## 目录结构

```
esp-detection-skill/
├── SKILL.md                      # 入口：核心原则、recipe 索引、陷阱、工作流
├── AGENTS.md                     # 补充约定：命名、include、结构、构建、清单
├── recipes/                      # 9 个场景 recipe
│   ├── env_setup.md
│   ├── dataset_prepare.md
│   ├── train_square.md
│   ├── train_rect.md
│   ├── export_onnx.md
│   ├── quantize_espdl.md
│   ├── eval_quantized.md
│   ├── all_in_one_pipeline.md
│   └── firmware_deploy.md
├── resources/                    # 快速参考文档
│   ├── api_reference.md          # Python + C++ API 签名
│   ├── config_reference.md       # 依赖 / YAML / Kconfig / sdkconfig
│   ├── model_pipeline.md         # 流水线状态机与阶段产物
│   ├── example_list.md           # 仓库示例与模板索引
│   └── pitfalls.md               # 26 条汇总陷阱
├── README.md                     # 本文件
└── CHANGELOG.md                  # 变更记录
```

## 使用方式

1. AI 代理在收到 esp-detection 相关请求时，先读 `SKILL.md`，按「Scenario Quick Reference」找到对应 recipe。
2. 不确定 API/配置时查 `resources/` 下对应文档。
3. 所有代码可直接复制改造，命名与占位符替换遵循 `AGENTS.md` 约定。

## 参考仓库

- 源码（只读 grounding）：`D:/esp-skill/espressif-repos/esp-detection/`
- 上游：[espressif/esp-detection](https://github.com/espressif/esp-detection)、[espressif/esp-dl](https://github.com/espressif/esp-dl)、[espressif/esp-ppq](https://github.com/espressif/esp-ppq)、[ultralytics/ultralytics](https://github.com/ultralytics/ultralytics)
