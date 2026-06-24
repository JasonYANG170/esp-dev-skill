# 仓库内示例与模板索引

> 仅列出仓库 `D:/esp-skill/espressif-repos/esp-detection/` 下真实存在的示例、模板与关键文件。

## 预训练模型示例 — examples/cat_detection/

| 路径 | 类型 | 说明 |
|---|---|---|
| `examples/cat_detection/espdet_pico_224_224_cat.pt` | 权重 | espdet_pico 224×224 猫检测模型（mAP@0.5:0.95≈69.9） |
| `examples/cat_detection/espdet_pico_416_416_cat.pt` | 权重 | espdet_pico 416×416 猫检测模型（mAP@0.5:0.95≈76.6） |
| `examples/cat_detection/espdet_pico_160_288_cat.pt` | 权重 | espdet_pico 160×288 rect 猫检测模型（mAP@0.5:0.95≈71.2） |
| `examples/cat_detection/cat.jpg` | 测试图 | 猫检测样例图 |

> 注意：`examples/cat_detection/` 仅含权重与测试图，**不含**可编译固件。可编译固件由 `espdet_run.py` 从模板生成到 `esp-dl/` 仓库下。

## 芯片端模板 — deploy/

### deploy/espdet_model_template/（模型组件模板，生成到 `esp-dl/models/<class>_detect/`）

| 路径 | 说明 |
|---|---|
| `deploy/espdet_model_template/espdet_detect.hpp` | `ESPDet` / `ESPDetDetect` 声明（model_type_t 枚举、score/nms 阈值常量） |
| `deploy/espdet_model_template/espdet_detect.cpp` | `ESPDet` 构造（Model + ImagePreprocessor + ESPDetPostProcessor），`load_model()` 分支 |
| `deploy/espdet_model_template/Kconfig` | 模型 flash 选择 / model location / sdcard 目录 |
| `deploy/espdet_model_template/CMakeLists.txt` | 调 `esp-dl` 的 `pack_espdl_models.py` 打包；按 target 选 models/p4 或 models/s3 |
| `deploy/espdet_model_template/idf_component.yml` | requires `espressif/esp-dl ^3.1.3`（override_path `../../esp-dl`） |
| `deploy/espdet_model_template/LICENSE` | MIT |
| `deploy/espdet_model_template/models/p4/.gitkeep` | P4 模型占位目录 |
| `deploy/espdet_model_template/models/s3/.gitkeep` | S3 模型占位目录 |

### deploy/espdet_example_template/（示例工程模板，生成到 `esp-dl/examples/<class>_detect/`）

| 路径 | 说明 |
|---|---|
| `deploy/espdet_example_template/CMakeLists.txt` | `project(<class>_detect)`，include ESP-IDF project.cmake |
| `deploy/espdet_example_template/main/app_main.cpp` | `app_main()`：JPEG 解码 → `ESPDetDetect::run()` → 打印检测框 |
| `deploy/espdet_example_template/main/CMakeLists.txt` | EMBED_FILES espdet.jpg；requires `<class>_detect` + BSP |
| `deploy/espdet_example_template/main/idf_component.yml` | 依赖 `<class>_detect` + P4/S3 BSP |
| `deploy/espdet_example_template/partitions.csv` | 默认分区（factory 8000K） |
| `deploy/espdet_example_template/partitions2.csv` | 大模型分区（factory 2000K + custom_det SPIFFS 4M） |
| `deploy/espdet_example_template/sdkconfig.defaults` | 公共 sdkconfig |
| `deploy/espdet_example_template/sdkconfig.defaults.esp32p4` | P4 专属（16MB flash, 200M PSRAM, L2 256KB） |
| `deploy/espdet_example_template/sdkconfig.defaults.esp32s3` | S3 专属（8MB flash, 80M octal PSRAM, 240MHz） |

## 校准数据 — deploy/cat_calib/

| 路径 | 说明 |
|---|---|
| `deploy/cat_calib/000000574810.jpg` | 校准图样例（仅 1 张占位，真实使用需补充同类场景图） |

## 数据集 — datasets/coco_cat/

| 路径 | 说明 |
|---|---|
| `datasets/coco_cat/images/val/*.jpg` | COCO val2017 猫子集验证图（用于 mAP 评估） |

## 关键脚本与配置（仓库根）

| 路径 | 说明 |
|---|---|
| `espdet_run.py` | 一站式 train→export→quantize→生成工程入口 |
| `espdet_run.sh` | 命令行调用示例 |
| `train.py` | `Train()` 训练入口 |
| `val.py` | 浮点模型验证示例 |
| `deploy/export.py` | `Export()` ONNX 导出（opset 13） |
| `deploy/quantize.py` | `quant_espdet()` INT8 量化 |
| `deploy/eval_quantized_model.py` | 量化模型 mAP 评估 |
| `cfg/models/espdet_pico.yaml` | 网络结构定义 |
| `cfg/datasets/coco_cat.yaml` | cat 数据集（基础） |
| `cfg/datasets/esp_cat.yaml` | cat 数据集（含 negative sampling） |
| `nn/esp_tasks.py` | `custom_parse_model`（注入自定义模块） |
| `nn/modules/esp_conv.py` | `DSConv` |
| `nn/modules/esp_block.py` | `DSBottleneck`/`DSC3k2`/`ESPBlock`/`ESPBlockLite`/`ESPSerial`/`ESPSerialLite` |
| `nn/modules/esp_head.py` | `ESPDetect`（含 `export_onnx_forward`） |
| `nn/modules/__init__.py` | 模块导出 |
| `data/esp_dataset.py` | `YOLOPosNegDataset` / `YOLOWeightedDataset` |
| `requirements.txt` | Python 依赖 |
| `README.md` | 项目说明（含延迟/mAP 表） |

## 文档 — docs/

| 路径 | 说明 |
|---|---|
| `docs/tutorials/how_to_train_and_deploy_model_with_rect_is_True.md` | rect 训练+部署完整教程（Train/Export/Quantize/Deployment 四段） |
