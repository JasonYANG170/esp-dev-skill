# AGENTS.md — Supplementary Agent Guide

> 核心规则、recipe 索引、陷阱、执行工作流均在 `SKILL.md`。
> 本文件仅记录 `SKILL.md` 未覆盖的工程约定与工具链指引，**不重复**内容。

## Project Context

- **训练/量化侧语言**: Python 3.8 · **框架**: PyTorch 2.2.0 + torchvision 0.17.0 + Ultralytics ≥ 8.3.112 + esp-ppq
- **芯片侧语言**: C++17 · **目标**: ESP32-P4 / ESP32-S3 · **工具链/构建**: ESP-IDF release/v5.3 及以上（模板 `sdkconfig.defaults.*` 由 ESP-IDF 5.4.0 生成）；推理依赖组件 `esp-dl` (^3.1.3)
- **License**: 仓库训练/量化代码 AGPL-3.0；芯片端模型模板 `deploy/espdet_model_template/LICENSE` 为 MIT。

## File Naming

### 训练侧（Python）
- 入口脚本：`espdet_run.py`（端到端）、`train.py`（仅训练）、`val.py`（浮点验证）、`deploy/export.py`（导出）、`deploy/quantize.py`（量化）、`deploy/eval_quantized_model.py`（量化评估）
- 自定义网络模块：`nn/modules/*.py`（`esp_conv.py` / `esp_block.py` / `esp_head.py`），统一在 `nn/modules/__init__.py` 导出
- 模型解析替换：`nn/esp_tasks.py::custom_parse_model`
- 自定义数据集类：`data/esp_dataset.py`（`YOLOPosNegDataset` / `YOLOWeightedDataset`）
- 模型配置 YAML：`cfg/models/espdet_pico.yaml`
- 数据集配置 YAML：`cfg/datasets/*.yaml`（如 `coco_cat.yaml`、`esp_cat.yaml`）

### 芯片侧（C++，来自 `deploy/espdet_*_template`）
- 模型组件目录：`<esp-dl>/models/<class>_detect/`（含 `espdet_detect.cpp` / `espdet_detect.hpp` / `Kconfig` / `CMakeLists.txt` / `idf_component.yml`）
- 示例工程目录：`<esp-dl>/examples/<class>_detect/`（含 `main/app_main.cpp` / `main/CMakeLists.txt` / `main/idf_component.yml` / `partitions*.csv` / `sdkconfig.defaults*`）
- 模型权重：`models/p4/*.espdl`（ESP32-P4）、`models/s3/*.espdl`（ESP32-S3）
- 命名约定：`espdet_pico_<H>_<W>_<class>.espdl`（`<H>`/`<W>` 为输入高/宽，`<class>` 为检测目标类名，全部小写）。

## Include / Import 模式

### Python（训练/导出/量化）
```python
# 任何训练/导出/量化脚本入口的固定三行（注入自定义模块解析）
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
tasks.parse_model = custom_parse_model

# 训练
from train import Train                      # train.py
# 导出
from deploy.export import Export             # deploy/export.py
# 量化
from deploy.quantize import quant_espdet     # deploy/quantize.py
# 量化模型评估
from deploy.eval_quantized_model import (ppq_graph_init, make_quant_validator_class)
```

### C++（芯片端固件）
```cpp
// 模型封装头（模板生成）
#include "espdet_detect.hpp"
// JPEG 解码 / 图像类型（esp-dl 提供）
#include "dl_image_jpeg.hpp"
// 检测基类与后处理
#include "dl_detect_base.hpp"
#include "dl_detect_espdet_postprocessor.hpp"
// ESP-IDF
#include "esp_log.h"
#include "esp_err.h"
// BSP（按目标芯片）
#include "bsp/esp-bsp.h"
```

## Standard Project Structure

### 仓库根（esp-detection）
```
esp-detection/
├── espdet_run.py            # 端到端入口（train→export→quantize→生成工程）
├── train.py / val.py        # 浮点训练 / 验证
├── requirements.txt         # Python 依赖（含 esp-ppq git 安装）
├── pyproject.toml           # Ultralytics 沿用的打包配置
├── cfg/
│   ├── models/espdet_pico.yaml      # 网络结构定义
│   └── datasets/*.yaml              # 数据集定义（coco_cat / esp_cat）
├── nn/
│   ├── esp_tasks.py         # custom_parse_model（注入自定义模块）
│   └── modules/             # DSConv / DSBottleneck / DSC3k2 / ESPBlock* / ESPDetect
├── data/esp_dataset.py      # YOLOPosNegDataset / YOLOWeightedDataset
├── deploy/
│   ├── export.py            # Export() → ONNX(opset13)
│   ├── quantize.py          # quant_espdet() → .espdl
│   ├── eval_quantized_model.py  # 量化模型 mAP 评估
│   ├── cat_calib/           # 校准图样例
│   ├── espdet_model_template/   # 芯片端「模型组件」模板（CMake/Kconfig/.cpp/.hpp）
│   └── espdet_example_template/ # 芯片端「示例工程」模板（main/partitions/sdkconfig）
├── examples/cat_detection/  # 预训练权重 .pt + 测试图
└── datasets/coco_cat/       # 示例数据（train/val 划分）
```

### 生成的芯片端工程（espdet_run.py 产物，落在 `esp-dl/` 下）
```
esp-dl/
├── examples/<class>_detect/
│   ├── CMakeLists.txt                  # project(<class>_detect)
│   ├── partitions.csv                  # rodata/partition 默认分区
│   ├── partitions2.csv                 # 含 custom_det SPIFFS 分区（大模型）
│   ├── sdkconfig.defaults              # 公共
│   ├── sdkconfig.defaults.esp32p4      # P4：16MB flash、200M PSRAM、L2 256KB cache
│   ├── sdkconfig.defaults.esp32s3      # S3：8MB flash、80M octal PSRAM、240MHz
│   └── main/
│       ├── app_main.cpp                # 含 ESPDetDetect::run 调用
│       ├── CMakeLists.txt              # EMBED_FILES espdet.jpg，requires <class>_detect
│       ├── idf_component.yml           # 依赖 <class>_detect + BSP 组件
│       └── espdet.jpg                  # 测试图（由 --img 指定）
└── models/<class>_detect/
    ├── CMakeLists.txt                  # 调 esp-dl 的 pack_espdl_models.py
    ├── Kconfig                         # model location / flash 选择 / sdcard 目录
    ├── idf_component.yml               # requires espressif/esp-dl ^3.1.3
    ├── espdet_detect.hpp               # ESPDet / ESPDetDetect 声明
    ├── espdet_detect.cpp               # ESPDet 构造 + ImagePreprocessor + ESPDetPostProcessor
    ├── LICENSE                         # MIT
    └── models/{p4,s3}/<class>.espdl    # 量化后权重（按 target��
```

## Canonical Entry / Init 模式

### Python 一站式入口
```python
# 推荐：直接命令行调用 espdet_run.py（见 SKILL.md Execution Workflow）
python espdet_run.py --class_name mycat --dataset "cfg/datasets/coco_cat.yaml" \
  --size 224 224 --target esp32p4 --calib_data "deploy/cat_calib" \
  --espdl "espdet_pico_224_224_mycat.espdl" --img "espdet.jpg"
```

### 芯片端 app_main（来自模板）
```cpp
#include "espdet_detect.hpp"
#include "dl_image_jpeg.hpp"
#include "esp_log.h"
#include "bsp/esp-bsp.h"

extern const uint8_t espdet_jpg_start[] asm("_binary_espdet_jpg_start");
extern const uint8_t espdet_jpg_end[]   asm("_binary_espdet_jpg_end");
const char *TAG = "custom_detect";

extern "C" void app_main(void)
{
#if CONFIG_ESPDET_DETECT_MODEL_IN_SDCARD
    ESP_ERROR_CHECK(bsp_sdcard_mount());
#endif
    dl::image::jpeg_img_t jpeg_img = {
        .data = (void *)espdet_jpg_start,
        .data_len = (size_t)(espdet_jpg_end - espdet_jpg_start)};
    auto img = dl::image::sw_decode_jpeg(jpeg_img, dl::image::DL_IMAGE_PIX_TYPE_RGB888);

    ESPDetDetect *detect = new ESPDetDetect();          // lazy_load=true
    auto &results = detect->run(img);
    for (const auto &res : results) {
        ESP_LOGI(TAG, "[category: %d, score: %f, x1: %d, y1: %d, x2: %d, y2: %d]",
                 res.category, res.score, res.box[0], res.box[1], res.box[2], res.box[3]);
    }
    delete detect;
    heap_caps_free(img.data);
#if CONFIG_ESPDET_DETECT_MODEL_IN_SDCARD
    ESP_ERROR_CHECK(bsp_sdcard_unmount());
#endif
}
```

## Build Workflow

### 训练/量化侧（PC）
1. `conda create -n espdet python=3.8 && conda activate espdet`
2. `pip install -r requirements.txt`（含 `git+https://github.com/espressif/esp-ppq`）
3.（端到端）`python espdet_run.py ...`  或  分步：`python train.py` → `python deploy/export.py` → `python deploy/quantize.py`

### 芯片侧（ESP-IDF）
1. 设置 ESP-IDF 环境（release/v5.3+，模板基于 5.4.0）
2. `cd <esp-dl>/examples/<class>_detect`
3. `idf.py set-target esp32p4`（或 `esp32s3`）
4. `idf.py menuconfig` → 按需选 `models: <class>_detect` → model location（rodata/partition/sdcard）
5. `idf.py build`
6. `idf.py flash monitor` → 串口观察 `[category: ..., score: ..., x1..y2: ...]`

## Code Generation Checklist

### 训练 / 导出 / 量化
- [ ] 入口已 `tasks.parse_model = custom_parse_model`
- [ ] 数据集 YAML 的 `path` / `train` / `val` / `names` 与实际目录一致
- [ ] `rect=True` 时 `imgsz=[h, w]`（h≠w）；方形时 `imgsz=s` 或 `[s, s]`
- [ ] 导出走 `deploy/export.py::Export()`（绑定 ESPDetect.export_onnx_forward，opset=13）
- [ ] 量化 `target` 与最终部署芯片一致；`calib_dir` 含真实校准图
- [ ] `quant_setting.equalization` 已按需开启（默认 iterations=4）

### 芯片端工程
- [ ] `idf_component.yml` 的 `override_path` 正确指向 `<esp-dl>/models/<class>_detect` 与 `../../esp-dl`
- [ ] Kconfig 的 model location 与 CMake 分支、app_main 的 mount/unmount 三处一致
- [ ] `.espdl` 已放入 `models/p4` 或 `models/s3`（按 target）
- [ ] S3 启用了 `DL_IMAGE_CAP_RGB565_BIG_ENDIAN`，并调用 `enable_letterbox({114,114,114})`
- [ ] `ESPDetPostProcessor` anchors 未改动：`{{8,8,4,4},{16,16,8,8},{32,32,16,16}}`
- [ ] 模型文件名与 Kconfig `CONFIG_FLASH_ESPDET_PICO_<H>_<W>_<CLASS>` / enum 全程一致
- [ ] 大模型使用 `partitions2.csv`（factory 2000K + custom_det 4M）或切 sdcard 模式

## Do Not Modify

- 仓库源码 `D:/esp-skill/espressif-repos/esp-detection/` —— 仅作为只读 grounding 来源
- `SKILL.md` frontmatter —— Skill 元数据
- `nn/modules/` 自定义模块的类签名与 `__all__` 导出顺序
- 模板 `deploy/espdet_*_template/` 的 anchors、forward 绑定、letterbox 逻辑
