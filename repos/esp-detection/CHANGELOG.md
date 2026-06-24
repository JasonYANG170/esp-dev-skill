# Changelog

本项目的所有重要变更均记录于此文件。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，版本号遵循 [Semantic Versioning](https://semver.org/lang/zh-CN/)。

## [1.0.0] - 2026-06-18

### Added

- 首个正式版本，面向 Espressif esp-detection 仓库（基于 Ultralytics YOLOv11 的轻量级实时目标检测）。
- `SKILL.md`：YAML frontmatter（name/description/trigger words/tags/license AGPL-3.0/compatibility/version 1.0.0）+ 12 条核心原则、When to Use、9 个 recipe 索引表、芯片支持/延迟表、espdet_pico 网络结构表、流水线状态机、Kconfig 模型位置表、12 条带 WRONG/CORRECT 代码对比的关键陷阱、执行工作流表、失败策略表、参考索引。
- `AGENTS.md`：项目上下文、文件命名（Python + C++）、include/import 模式、标准项目结构（仓库根 + 生成的芯片端工程）、canonical 入口/初始化模式、构建工作流（训练侧 + 芯片侧）、代码生成清单、Do Not Modify 说明。
- `recipes/`（9 个）：env_setup、dataset_prepare、train_square、train_rect、export_onnx、quantize_espdl、eval_quantized、all_in_one_pipeline、firmware_deploy —— 每个 recipe 含适用摘要、触发意图、前置条件表、分步说明（带真实可复制代码）、常见错误表（错误/原因/解决方法）、参考。
- `resources/api_reference.md`：Python 训练/导出/量化函数签名（`Train`/`Export`/`quant_espdet`/`ppq_graph_init`/`ppq_graph_inference`/`run`/`rename_project`）、自定义网络模块（`DSConv`/`DSBottleneck`/`DSC3k2`/`ESPBlock`/`ESPBlockLite`/`ESPSerial`/`ESPSerialLite`/`ESPDetect`/`custom_parse_model`）、数据集类（`YOLOPosNegDataset`/`YOLOWeightedDataset`）、C++ 芯片端封装（`ESPDet`/`ESPDetDetect`/`app_main`）。
- `resources/config_reference.md`：Python 依赖版本约束、espdet_pico.yaml 逐层说明、数据集 YAML（含 negative sampling）、espdet_run.py CLI 参数、芯片端 Kconfig、sdkconfig.defaults（公共/P4/S3）、分区表（partitions.csv/partitions2.csv）、组件 idf_component.yml。
- `resources/model_pipeline.md`：数据→芯片推理状态机、阶段详表、一站式 vs 分步对比、产物命名约定、rect vs 方形路径对比。
- `resources/example_list.md`：examples/cat_detection 预训练权重、deploy/espdet_model_template、deploy/espdet_example_template、校准数据、数据集、关键脚本与配置、docs 索引。
- `resources/pitfalls.md`：26 条按主题（A-G）归类的常见陷阱与解决方法。
- `README.md`：中文介绍、功能特性、支持范围、两种安装方式、目录结构、使用方式。
- `CHANGELOG.md`：本文件。

### Grounding

- 全部 API、结构体、宏、配置项、Kconfig 符号、文件路径、代码片段均来自 `D:/esp-skill/espressif-repos/esp-detection/` 真实源码（README、docs/tutorials、train.py、val.py、espdet_run.py、deploy/{export,quantize,eval_quantized_model}.py、nn/{esp_tasks,modules/*}、data/esp_dataset.py、cfg/{models,datasets}/*.yaml、deploy/espdet_*_template/*、requirements.txt）。
- 未在源码出现的内容一律不写入；文档较薄处以「更少 recipe / 更短参考」处理，未臆造。
