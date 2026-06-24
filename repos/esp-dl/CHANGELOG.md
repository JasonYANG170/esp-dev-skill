# Changelog

本项目的所有重要变更都记录在此文件中。

格式基于 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

## [1.1.0] - 2026-06-18

### Added

- 补齐视觉与量化场景的 5 篇高价值食谱（均取自仓库真实文档 / 头文件 / 示例）：
  - `recipes/vision_pose.md` — YOLO11n-pose 关键点估计端到端：`COCOPose` + `yolo11posePostprocessor` → 17 个 COCO 关键点（`result_t.keypoint`）。含 PTQ/QAT 精度对比（mAP50-95：PTQ 43.1% → QAT 44.9%，float 50.0%）。源：`docs/en/tutorials/how_to_deploy_yolo11n-pose.rst`、`examples/yolo11_pose`、`examples/tutorial/how_to_quantize_model/quantize_yolo11n-pose`、`esp-dl/vision/detect/dl_pose_yolo11_postprocessor.hpp`。
  - `recipes/vision_classification.md` — MobileNetV2/ImageNet 分类端到端：`ImageNetCls` + `ImageNetClsPostprocessor` → Top-K `(cat_name, score)`（`dl::cls::result_t`，无框）。含 per-channel(P4) vs per-tensor(S3) 及混合精度/均衡精度表（S3: 60.5%→69.2%→69.8%）。源：`docs/en/tutorials/how_to_deploy_mobilenetv2.rst`、`examples/mobilenetv2_cls`、`esp-dl/vision/classification/*`、`models/imagenet_cls/imagenet_cls.hpp`。
  - `recipes/vision_segmentation.md` — YOLO11n-seg 实例分割端到端：`COCOSeg` + `yolo11segPostprocessor`（`m_nm=32`）→ 逐实例 `result_t.mask`，alpha 混合 + `write_bmp` 导出。源：`examples/yolo11_seg`、`examples/tutorial/how_to_quantize_model/quantize_yolo11n_seg`、`esp-dl/vision/detect/dl_seg_yolo11_postprocessor.hpp`、`dl_image_draw.hpp`/`dl_image_bmp.hpp`。（注：分割无独立 `.rst` 教程，recipe 基于示��� + 头文件。）
  - `recipes/quantize_tqt.md` — TQT（Trained Quantization Thresholds）训练量化阈值：无需标签，在 log 域学习 `scale=2^k` 并联合微调权重。含 `TQTSetting` 参数表、GPU 加速、聚焦 logit 量化、实测精度（YOLO26n mAP50-95 0.342→0.371；MobileNetV2 Top-1 70.85%→71.775%）。源：`docs/en/tutorials/quantize_model_with_TQT.rst`、`examples/tutorial/how_to_quantize_model/quantize_yolo26`、`.../quantize_mobilenetv2`。
  - `recipes/quantize_advanced_ptq.md` — 高级 PTQ：混合精度（坏层 int16，`dispatching_table.append` + `get_target_platform`）与逐层权重均衡（`equalization_setting`，ReLU6→ReLU）。源：`docs/en/tutorials/how_to_deploy_mobilenetv2.rst`（Mixed precision / Layerwise equalization 两节）、`examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/quantize_onnx_model.py`。

### Changed

- `SKILL.md`：
  - `metadata.version` 1.0.0 → 1.1.0。
  - Scenario Quick Reference：Vision 表新增 `vision_pose` / `vision_classification` / `vision_segmentation`；Quantization 表新增 `quantize_tqt` / `quantize_advanced_ptq`。
  - 新增 Core Principle 13：视觉任务各自有独立后处理器/Wrapper，不可混用检测 wrapper（pose→`COCOPose`、seg→`COCOSeg`、cls→`ImageNetCls`）。
  - References 补充 `quantize_model_with_TQT.rst` 与视觉部署教程列表。
- `resources/example_list.md`：新增/补全 `quantize_yolo11n-pose`、`quantize_yolo11n_seg`、`quantize_yolo26`、`quantize_mobilenetv2` 脚本说明；`yolo11_pose`/`yolo11_seg`/`mobilenetv2_cls` 行补 recipe 交叉引用；参考脚本表补 TQT / 姿态 QAT 条目。
- `resources/api_reference.md`：新增姿态/分割后处理器签名（`yolo11posePostProcessor` / `yolo11segPostProcessor`）、`coco_pose`/`coco_seg` 模型组件、分类基类与结果结构（`Cls`/`ClsWrapper`/`ClsImpl`/`ClsPostprocessor`/`ImageNetClsPostprocessor`/`dl::cls::result_t`）、`imagenet_cls` 组件、`draw_hollow_rectangle`/`write_bmp`，以及 ESP-PPQ 的 TQT / 混合精度 / 逐层均衡 `quant_setting` 用法。

### Notes

- 本版本未删除或重命名任何已有 recipe；所有新增 API/结构体/宏/Kconfig 符号/示例路径均取自 `esp-dl` 仓库（`docs/en/`、`esp-dl/vision/`、`models/`、`examples/`）。
- 基于 ESP-DL 组件版本 `3.3.6`，ESP-PPQ >= 1.2.7（TQT 要求）。

## [1.0.0] - 2026-06-18

### Added

- 初始版本：面向 ESP-DL 深度学习推理框架的 AI Skill。
- `SKILL.md`：12 条核心原则、适用/不适用场景、11 篇食谱索引、芯片与量化支持表、模型位置枚举表、关键运行时类型表、12 条带错误/正确代码块的必读陷阱、执行流程表、失败策略表。
- `AGENTS.md`：项目上下文、文件命名、include 模式、标准推理工程结构、规范推理范式、构建与量化工作流、固件代码生成清单、Do-Not-Modify 说明。
- `recipes/`（11 篇）：
  - `project_setup.md` — 把 esp-dl 作为托管组件加入工程
  - `load_model_rodata.md` — 从 `.rodata` 加载模型
  - `load_model_partition.md` — 从 FLASH 分区加载模型
  - `load_model_sdcard.md` — 从 SD 卡加载模型
  - `run_inference.md` — 执行推理（量化输入 / 运行 / 反量化输出）
  - `streaming_model.md` — 部署流式/时序模型
  - `profile_test.md` — 测试与剖析（test / profile）
  - `quantize_onnx.md` — 用 ESP-PPQ 量化 ONNX 模型
  - `quantize_torch.md` — 用 ESP-PPQ 量化 PyTorch 模型
  - `auto_quant.md` — 用 AutoQuant 自动搜索量化配置
  - `vision_detection.md` — 视觉目标检测端到端
- `resources/`：
  - `api_reference.md` — `dl::Model` / `dl::TensorBase` / `ExponentInfo` / 内存管理 / 视觉图像 / 检测后处理 真实签名
  - `config_reference.md` — Kconfig（`PIX_CVT_*`）、组件清单、构造参数、ESP-PPQ 与 AutoQuant 参数、平台舍入策略
  - `pitfalls.md` — 20 条真实坑点汇总
  - `example_list.md` — 仓库内真实示例与脚本索引
- `README.md` 与 `CHANGELOG.md`。

### Notes

- 所有 API、结构体、宏、Kconfig 符号、量化参数与示例路径均取自 `esp-dl` 仓库（文档 `docs/en/`、头文件 `esp-dl/dl/`、示例 `examples/`、`operator_support_state.md`、`esp-dl/Kconfig`、`esp-dl/idf_component.yml`）。
- 基于 ESP-DL 组件版本 `3.3.6`。
