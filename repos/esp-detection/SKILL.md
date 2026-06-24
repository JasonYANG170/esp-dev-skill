---
name: esp-detection-skill
description: >-
  AI Skill for developing with Espressif esp-detection, a lightweight real-time object detection
  project based on Ultralytics YOLOv11. Used when users need to train a custom detection model,
  export/quantize it to ESP-DL (.espdl) format, and deploy inference on ESP32-P4 / ESP32-S3 chips.
  Trigger words: "esp-detection", "ESPDet", "espdet_pico", "ESP-DL", "espdl", "YOLOv11", "object detection",
  "目标检测", "esp32p4", "esp32-s3", "esp-dl", "量化", "quantize", "esp-ppq", "deployment", "部署"
tags:
  - embedded
  - esp32
  - esp32-p4
  - esp32-s3
  - object-detection
  - YOLO
  - YOLOv11
  - deep-learning
  - model-deployment
  - ESP-DL
  - quantization
  - computer-vision
license: AGPL-3.0
compatibility: >-
  Training pipeline: Python 3.8, PyTorch 2.2.0, Ultralytics >= 8.3.112, esp-ppq. Firmware targets:
  ESP32-P4 / ESP32-S3 built with ESP-IDF release/v5.3 or above (templates pinned to ESP-IDF 5.4.0).
metadata:
  author: Community
  version: "1.0.0"
---

# esp-detection-skill

AI Skill for developing with Espressif **esp-detection** — a lightweight, ESP-optimized real-time object detection project based on Ultralytics YOLOv11. Provides scenario-driven recipes covering the full flow: dataset preparation, model training, ONNX export, INT8 quantization to ESP-DL `.espdl`, quantized-model evaluation, and firmware deployment/inference on ESP32-P4 and ESP32-S3. All APIs, configs, structs, and code snippets are grounded in the real repository source.

## Core Principles

1. **训练与部署是两个独立阶段** — 训练在 PC 的 Python 环境中完成（PyTorch + Ultralytics + esp-ppq），芯片端推理通过 ESP-IDF 编译的 C++ 固件执行。两套环境、两套 API，绝不混用。
2. **必须替换 Ultralytics 的 `parse_model`** — esp-detection 的自定义模块（`DSConv`、`ESPBlock`、`ESPDetect` 等）只有在执行 `tasks.parse_model = custom_parse_model` 之后才会被模型解析器识别。任何训练/导出脚本入口都需先注入这行代码。
3. **`espdet_run.py` 是一站式入口** — 它串联 train → export → quantize → 生成芯片端工程模板。能用它就别手工拼接四个阶段。
4. **非方形输入走 `rect=True` 路径** — 当 `size=[h, w]` 且 `h != w` 时，`espdet_run.py` 自动启用 `rect=True` 训练（含可选预训练），方形输入走标准路径。`imgsz` 在 `rect=True` 下必须是 `[h, w]` 列表。
5. **量化必须提供校准数据集** — `quant_espdet(...)` 的 `calib_dir` 指向一个含 `.jpg/.jpeg/.png/.bmp` 的目录，`CaliDataset` 会 `Resize` 到目标 `imgsz` 并 `Normalize(mean=0,std=1)`（即不做均值归一化，仅除以 255 由模型内部处理）。
6. **ONNX 导出的 opset 固定为 13** — esp-ppq 默认 opset 18 与 YOLOv11 的 `arange` 不兼容，`ESP_Detect_Exporter.export_onnx` 强制 `opset_version=13`，并用 `onnxsim`（而非 onnxslim）做简化，否则会出现 `NCHW → 1(N*C)HW` 的形状问题。
7. **导出时必须替换 ESPDetect 与 Attention 的 forward** — `Export()` 会把 `ESPDetect.export_onnx_forward` 绑定到模型，输出 6 个张量 `box0,score0,box1,score1,box2,score2`（3 个尺度），与量化后处理期望的输出顺序一致。漏掉这一步量化图的输出语义会错乱。
8. **芯片端模型加载位置三选一** — Kconfig 的 `model location` 提供 `flash_rodata`（默认，嵌入固件）、`flash_partition`（SPIFFS 分区，需 `partitions2.csv`）、`sdcard`。不同位置对应不同的 `dl::Model(path, ...)` 构造参数与 CMake 分支。
9. **ESP32-S3 与 ESP32-P4 的 ImagePreprocessor 不同** — P4 直接 RGB888；S3 需 `DL_IMAGE_CAP_RGB565_BIG_ENDIAN`，且 `enable_letterbox({114,114,114})` 必须调用以做保持比例缩放。
10. **后处理 anchors 是双尺度对** — `ESPDetPostProcessor` 的 anchors 形如 `{{8,8,4,4},{16,16,8,8},{32,32,16,16}}`（stride 下采样与上采样），与 `ESPDetect` 的 3 个检测层（P3/P4/P5）一一对应，切勿改动。
11. **`rename_project` 用占位符替换** — 生成工程时通过字典 `{"custom":..., "CUSTOM":..., "imgH":..., "imgW":..., "espdet.jpg":..., "espdet_jpg":...}` 替换模板，但 `add_custom_command` 行会被临时占位符保护，避免被误替换。
12. **模型命名约定强制** — 固件模板里模型文件名必须形如 `espdet_pico_imgH_imgW_custom.espdl`（Kconfig `CONFIG_FLASH_ESPDET_PICO_imgH_imgW_CUSTOM`、enum `ESPDET_PICO_imgH_imgW_CUSTOM` 全程保持一致），改 class_name 时由 `rename_project` 统一替换。

## When to Use

**Applicable:**
- 训练自定义单类/多类目标检测模型（基于 espdet_pico 或自定义 YOLOv11 检测网络）
- 准备 YOLO 格式数据集并配置 `cfg/datasets/*.yaml`（含 negative sampling、weighted sampling）
- 将训练好的 `.pt` 模型导出为 ONNX（opset 13、onnxsim 简化）
- 用 esp-ppq 把 ONNX 量化为 INT8 `.espdl`（PTQ），并提供校准数据集
- 评估量化后模型在 PC 上的 mAP（`deploy/eval_quantized_model.py`）
- 在 ESP32-P4 / ESP32-S3 上用 ESP-DL 运行 `.espdl` 推理（C++ 固件模板）
- 启用 `rect=True` 进行非方形分辨率训练与部署（提升精度与速度）

**Not applicable:**
- 通用 PyTorch / Ultralytics 训练问题（与本仓库自定义模块无关）
- 其他推理后端（TFLite / OpenVINO / CoreML）—— esp-detection 仅面向 ESP-DL
- 非 ESP32 系列芯片的部署
- 图像分类、姿态估计、实例分割任务（本仓库仅做目标检测）
- 硬件原理图 / PCB 设计

---

## Scenario Quick Reference (Recipes)

当用户意图命中下表场景时，**先读对应 recipe** —— 它包含完整调用链、分步说明、可复制代码与常见错误。

### 环境与数据

| recipe | scenario |
|---|---|
| `recipes/env_setup.md` | 搭建 Python 训练/量化环境（conda + requirements + esp-ppq） |
| `recipes/dataset_prepare.md` | 准备 YOLO 格式数据集与 `cfg/datasets/*.yaml`（含 negative sampling） |

### 训练

| recipe | scenario |
|---|---|
| `recipes/train_square.md` | 方形分辨率训练 espdet_pico（`size=[s,s]`） |
| `recipes/train_rect.md` | 非方形分辨率 `rect=True` 训练（含可选预训练） |

### 导出与量化

| recipe | scenario |
|---|---|
| `recipes/export_onnx.md` | 将 `.pt` 导出为 ONNX（opset 13 + onnxsim，6 输出） |
| `recipes/quantize_espdl.md` | 用 esp-ppq 将 ONNX 量化为 INT8 `.espdl` |
| `recipes/eval_quantized.md` | 在 PC 上评估量化后模型的 mAP |

### 部署

| recipe | scenario |
|---|---|
| `recipes/all_in_one_pipeline.md` | 一条命令跑通 train→export→quantize→生成芯片工程（`espdet_run.py`） |
| `recipes/firmware_deploy.md` | 在 ESP32-P4 / ESP32-S3 上编译并运行 `.espdl` 推理固件 |

---

## 芯片支持与延迟

| 芯片 | 输入尺寸 | 模型 | 参数量(M) | FLOPS(G) | 延迟(ms) | 来源 |
|---|---|---|---|---|---|---|
| ESP32-P4 | 224×224 | espdet_pico_224_224 | 0.36 | 0.17 | 51.4 | README |
| ESP32-S3 | 224×224 | espdet_pico_224_224 | 0.36 | 0.17 | 126.2 | README |
| ESP32-P4 | 416×416 | espdet_pico_416_416 | 0.36 | 0.60 | 201.7 | README |
| ESP32-S3 | 416×416 | espdet_pico_416_416 | 0.36 | 0.60 | 449.5 | README |
| ESP32-P4 | 160×288 | espdet_pico_160_288 (rect) | 0.36 | 0.16 | 45.9 | README |
| ESP32-S3 | 160×288 | espdet_pico_160_288 (rect) | 0.36 | 0.16 | 115.5 | README |

> 延迟含 pre/post-processing。mAP 在 COCO val2017 cat 子集上：224 模型 mAP@0.5:0.95=69.9，416 模型 76.6，160×288 rect 模型 71.2。

## espdet_pico 网络结构（cfg/models/espdet_pico.yaml）

| 分组 | 层（YAML 顺序） | 模块 |
|---|---|---|
| Backbone P1 | `Conv [64,3,2]` | 标准 Conv |
| Backbone P2 | `DSConv [128,3,2]` + `ESPBlockLite [256]` | 深度可分离 + Lite 块 |
| Backbone P3 | `DSConv [256,3,2]` + `DSC3k2×2 [256]` | DS bottleneck 堆叠 |
| Backbone P4 | `SCDown [256,3,2]` + `DSC3k2×2 [256,True]` | 下采样卷积 |
| Backbone P5 | `SCDown [512,3,2]` + `DSC3k2×2 [512,True]` + `SPPF [512,5]` + `DSConv [512,7,1,3]` | SPPF + 大核 DS |
| Head (P4 fuse) | `Upsample` + `Concat(P4)` + `ESPBlock×2 [256]` | FPN 上采样 |
| Head (P3 fuse) | `Upsample` + `Concat(P3)` + `ESPBlock×2 [128]` (小目标层 16) | FPN 上采样 |
| Head (P4 out)  | `DSConv [128,3,2]` + `Concat` + `ESPBlock×2 [512]` (中目标层 19) | PAN 下采样 |
| Head (P5 out)  | `DSConv [256,3,2]` + `Concat` + `ESPBlock×2 [512]` (大目标层 22) | PAN 下采样 |
| Detect | `ESPDetect [nc]` from `[16,19,22]` | 3 尺度检测头 |

> `nc: 1`、`activation: 'nn.ReLU()'`、`scales.n: [0.50, 0.25, 512]`（depth/width/max_channels）。`ESPDetect` 的 `reg_max=1`，每个 anchor 输出 `nc + 4` 个值。

## 模型-部署流水线状态机

```
.pt (PyTorch)
  │  Export()  →  tasks.parse_model=custom_parse_model + 绑定 export_onnx_forward
  ▼
.onnx (opset 13, onnxsim, 6 输出: box0,score0,box1,score1,box2,score2)
  │  quant_espdet()  →  esp-ppq PTQ (Equalization, 8-bit, target esp32p4/esp32s3)
  ▼
.espdl (INT8, FlatBuffers)
  │  espdet_run.py rename_project()  →  从 deploy/espdet_*_template 复制并替换占位符
  ▼
ESP-IDF 工程 (esp-dl/examples/<class>_detect)
  │  idf.py set-target esp32p4|esp32s3 && idf.py flash monitor
  ▼
ESPDetDetect::run() → vector<result_t{category, score, box[4]}>
```

## 关键 Kconfig / 模型位置选项（芯片端）

| Kconfig 符号 | 取值 | 含义 |
|---|---|---|
| `FLASH_ESPDET_PICO_imgH_imgW_CUSTOM` | bool, default y | 把 `espdet_pico_imgH_imgW_custom.espdl` 打包进固件 |
| `ESPDET_PICO_imgH_imgW_CUSTOM` | choice | 默认模型选择（与上面配套） |
| `DEFAULT_ESPDET_DETECT_MODEL` | int | 默认模型枚举值（`ESPDET_PICO_imgH_imgW_CUSTOM`→0） |
| `ESPDET_DETECT_MODEL_IN_FLASH_RODATA` | choice (默认) | 模型嵌入 flash rodata，CMake 调 `target_add_aligned_binary_data` |
| `ESPDET_DETECT_MODEL_IN_FLASH_PARTITION` | choice | 模型放 SPIFFS 分区 `espdet_det`（需 `partitions2.csv`） |
| `ESPDET_DETECT_MODEL_IN_SDCARD` | choice | 模型放 SD 卡 `models/p4` 或 `models/s3` |
| `ESPDET_DETECT_MODEL_LOCATION` | int | 0=rodata, 1=partition, 2=sdcard（传给 `fbs::model_location_type_t`） |
| `ESPDET_DETECT_MODEL_SDCARD_DIR` | string | SD 卡上的模型目录，默认 `models/s3`(S3) / `models/p4`(P4) |

---

## Critical Pitfalls (Must Read)

下面是最常见的错误，违反任一条都会导致训练失败、量化图错乱或芯片推理无结果。

### 1. 训练/导出前必须注入 custom_parse_model

```python
# ❌ WRONG — 直接用 YOLO，自定义模块 ESPBlock/ESPDetect 无法被解析
from ultralytics import YOLO
model = YOLO('cfg/models/espdet_pico.yaml')
model.train(data='cfg/datasets/coco_cat.yaml', imgsz=224)

# ✅ CORRECT — 先替换 parse_model
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
from ultralytics import YOLO

tasks.parse_model = custom_parse_model   # 关键：注册 ESP 自定义模块
model = YOLO('cfg/models/espdet_pico.yaml')
model.train(data='cfg/datasets/coco_cat.yaml', imgsz=224)
```

### 2. rect=True 时 imgsz 必须是 [h, w] 列表

```python
# ❌ WRONG — rect=True 却传标量
results = Train(pretrained_path=None, dataset="cfg/datasets/coco_cat.yaml",
                imgsz=224, rect=True)   # 形状歧义

# ✅ CORRECT — rect=True 时 imgsz=[h,w]，h!=w 才有意义
results = Train(pretrained_path=None, dataset="cfg/datasets/coco_cat.yaml",
                imgsz=[160, 288], rect=True)
```

### 3. espdet_run.py 的 size 是 nargs=2（高度 宽度）

```bash
# ❌ WRONG — 只给一个值，argparse 报错或语义错误
python espdet_run.py --class_name mycat --size 224 --target esp32p4 ...

# ✅ CORRECT — 两个值，顺序 [h w]
python espdet_run.py --class_name mycat --size 224 224 --target esp32p4 ...
python espdet_run.py --class_name mycat --size 160 288 --target esp32p4 ...
```

### 4. 导出 ONNX 必须走 ESP_YOLO/Export，不能直接 model.export

```python
# ❌ WRONG — 用原生 ultralytics 导出，输出语义与后处理不匹配
model = YOLO('best.pt')
model.export(format='onnx', imgsz=224)   # 缺 ESPDetect.export_onnx_forward 绑定

# ✅ CORRECT — 用 deploy/export.py 的 Export()
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
from deploy.export import Export

tasks.parse_model = custom_parse_model
Export("best.pt", [224, 224])   # 内部绑定 ESP_Attention / ESPDetect.export_onnx_forward
# 输出 best.onnx，含 6 个张量：box0,score0,box1,score1,box2,score2
```

### 5. 量化校准集不做均值归一化

```python
# ❌ WRONG — 自定义 CaliDataset 时加了 ImageNet 均值方差
transforms.Normalize(mean=[0.485,0.456,0.406], std=[0.229,0.224,0.225])

# ✅ CORRECT — 仓库 CaliDataset 使用 mean=0, std=1（仅 Resize + ToTensor）
transforms.Compose([
    transforms.ToTensor(),
    transforms.Resize((h, w)),
    transforms.Normalize(mean=[0, 0, 0], std=[1, 1, 1]),
])
```

### 6. 量化 target 必须与最终部署芯片一致

```python
# ❌ WRONG — 量化用 esp32p4，却部署到 S3
quant_espdet(onnx_path=ONNX, target="esp32p4", ...)
# 然后把 espdl 拷到 models/s3 跑

# ✅ CORRECT — target 与芯片一致，且 espdl 拷到对应子目录
quant_espdet(onnx_path=ONNX, target="esp32s3", num_of_bits=8,
             imgsz=[224,224], calib_dir="deploy/cat_calib",
             espdl_model_path="espdet_pico_224_224_cat.espdl")
# espdet_run.py 会按 target 自动拷到 models/s3 或 models/p4
```

### 7. 芯片端 ESP32-S3 必须启用 RGB565 大端 capability

```cpp
// ❌ WRONG — S3 用了 P4 的 RGB888 预处理器，颜色/格式错乱
m_image_preprocessor = new dl::image::ImagePreprocessor(m_model, {0,0,0}, {255,255,255});

// ✅ CORRECT — 按 CONFIG_IDF_TARGET 分支（来自 espdet_detect.cpp）
#if CONFIG_IDF_TARGET_ESP32P4
    m_image_preprocessor = new dl::image::ImagePreprocessor(m_model, {0, 0, 0}, {255, 255, 255});
#else
    m_image_preprocessor = new dl::image::ImagePreprocessor(
        m_model, {0, 0, 0}, {255, 255, 255}, dl::image::DL_IMAGE_CAP_RGB565_BIG_ENDIAN);
#endif
m_image_preprocessor->enable_letterbox({114, 114, 114});   // 必须调用
```

### 8. 不能改动 ESPDetPostProcessor 的 anchors

```cpp
// ❌ WRONG — 自调 anchors，与 ESPDetect 3 层 stride (8/16/32) 失配
m_postprocessor = new dl::detect::ESPDetPostProcessor(
    m_model, m_image_preprocessor, score_thr, nms_thr, 10,
    {{4,4,2,2}, {8,8,4,4}, {16,16,8,8}});   // 错配

// ✅ CORRECT — 与模板一致（来自 espdet_detect.cpp）
m_postprocessor = new dl::detect::ESPDetPostProcessor(
    m_model, m_image_preprocessor, score_thr, nms_thr, 10,
    {{8, 8, 4, 4}, {16, 16, 8, 8}, {32, 32, 16, 16}});
```

### 9. 模型位置切换必须同步 Kconfig + CMake + 分区表

```text
# ❌ WRONG — 选 sdcard 但没有挂载 SD，也没有 partitions2.csv
CONFIG_ESPDET_DETECT_MODEL_IN_SDCARD=y
# app_main 里却没调 bsp_sdcard_mount()

# ✅ CORRECT — 三处一致（参考模板）
# 1) sdkconfig: CONFIG_ESPDET_DETECT_MODEL_IN_FLASH_PARTITION=y 时用 partitions2.csv
#    （含 custom_det, data, spiffs, 4M 分区）
# 2) sdcard 模式: app_main 里
#      #if CONFIG_ESPDET_DETECT_MODEL_IN_SDCARD
#          ESP_ERROR_CHECK(bsp_sdcard_mount());
#      #endif
# 3) CMake: 非 sdcard 时按 IDF_TARGET 选 models/s3 或 models/p4 并 pack
```

### 10. 模型文件名与 Kconfig 命名全程一致

```text
# ❌ WRONG — 改了 class_name 但只改了 .espdl 文件名
espdet_pico_224_224_mycat.espdl   # 文件名
# 而 Kconfig 里仍是 CONFIG_FLASH_ESPDET_PICO_imgH_imgW_CUSTOM

# ✅ CORRECT — 由 rename_project 统一替换，保持一致
# class_name="mycat", h=224, w=224 →
#   文件名: espdet_pico_224_224_mycat.espdl
#   Kconfig: CONFIG_FLASH_ESPDET_PICO_224_224_MYCAT
#   enum:    ESPDET_PICO_224_224_MYCAT
```

### 11. ESPDetect 输出 6 个张量，评估时要正确 cat

```python
# ❌ WRONG — 评估量化模型时输出顺序/reshape 维度错误
boxes = torch.cat([graph_outputs[i] for i in range(0, 6, 2)], dim=-1)

# ✅ CORRECT — 来自 deploy/eval_quantized_model.py 的 ppq_graph_inference
boxes_ls  = [graph_outputs[i] for i in range(0, 6, 2)]   # 3 个 box 分支
bs = inputs.shape[0]
boxes  = torch.cat([graph_outputs[2*i].view(bs, 4, -1)   for i in range(3)], dim=-1)
scores = torch.cat([graph_outputs[2*i+1].view(bs, NC, -1) for i in range(3)], dim=-1)
preds = dict(boxes=boxes, scores=scores, feats=boxes_ls)
detect_model = Detect(nc=NC, reg_max=1, end2end=False, ch=[32, 64, 128])
detect_model.stride = [8.0, 16.0, 32.0]
y = detect_model._inference(preds)
```

### 12. 导出动态 batch 仅用于 QAT，芯片部署必须静态

```python
# ❌ WRONG — 部署模型却开 dynamic
model.export(format="onnx", dynamic=True)   # 输入维度带 batch 轴，esp-ppq/芯片不友好

# ✅ CORRECT — 部署用静态（Export 默认 dynamic 由 self.args.dynamic 控制，部署时为 False）
model.export(format="onnx", simplify=True, opset=13, imgsz=input_size)  # batch=1 静态
# dynamic=True 只用于 QAT gt onnx 推理场景
```

---

## Execution Workflow

| 步骤 | 名称 | 说明 |
|---|---|---|
| 1 | 明确需求 | 确定检测目标类别数（`nc`）、输入分辨率（方形/非方形）、目标芯片（P4/S3） |
| 2 | 查 recipe | 按场景读 `recipes/` 对应文件：端到端用 `all_in_one_pipeline.md`，分步用其余 |
| 3 | 查 resources | API/签名不确定时查 `resources/api_reference.md`；Kconfig/sdkconfig 查 `config_reference.md` |
| 4 | 校验环境 | 训练侧：Python 3.8 + `pip install -r requirements.txt` + esp-ppq；芯片侧：ESP-IDF v5.3+ |
| 5 | 准备数据 | YOLO 格式 + `cfg/datasets/*.yaml`（含 negative/weighted 采样按需） |
| 6 | 训练+导出+量化 | 推荐 `python espdet_run.py ...`；或分步 Train/Export/quant_espdet |
| 7 | （可选）评估 | `deploy/eval_quantized_model.py` 验证量化后 mAP |
| 8 | 生成工程 | `espdet_run.py` 自动复制 `deploy/espdet_*_template` 并 `rename_project` |
| 9 | 编译烧录 | `idf.py set-target esp32p4\|esp32s3 && idf.py flash monitor` |
| 10 | 验证输出 | 串口看到 `[category: %d, score: %f, x1: %d, y1: %d, x2: %d, y2: %d]` |

### 步骤 6 细节 —— 一条命令 vs 分步

**首选（端到端）：**

```bash
python espdet_run.py \
  --class_name mycat --pretrained_path None \
  --dataset "cfg/datasets/coco_cat.yaml" --size 224 224 \
  --target esp32p4 --calib_data "deploy/cat_calib" \
  --espdl "espdet_pico_224_224_mycat.espdl" --img "espdet.jpg"
```

**分步（需要精细控制时）：** 先 `train.py` 的 `Train(...)`，再 `deploy/export.py` 的 `Export(...)`，再 `deploy/quantize.py` 的 `quant_espdet(...)`，最后手动复制模板并 `rename_project`（参考 `recipes/all_in_one_pipeline.md` 末尾的部署段）。

---

## Failure Strategies

| 情况 | 处理 |
|---|---|
| 模型 YAML 里出现未知模块名 | 确认是否执行了 `tasks.parse_model = custom_parse_model`；模块名必须来自 `nn/modules`（DSConv/DSBottleneck/DSC3k2/ESPBlock/ESPBlockLite/ESPSerial/ESPSerialLite/ESPDetect） |
| ONNX 导出报 onnxslim 形状错误 | 改用 `onnxsim`（仓库已默认），opset 固定 13；不要用 ultralytics 默认 onnxslim |
| 量化后 mAP 大幅下降 | 增大校准集（默认 `calib_steps=32`）、调 `equalization_setting.iterations`（默认 4，可调到 10）、检查校准图与训练分布是否一致 |
| 芯片端无检测结果（全空） | 检查 `enable_letterbox({114,114,114})`、S3 的 RGB565 capability、anchors 是否被改动、score_thr（默认 0.25）是否过高 |
| `idf.py` 找不到 esp-dl 组件 | 模板 `idf_component.yml` 用 `override_path` 指向 `../../esp-dl`；确认 `espdet_run.py` 已 `git clone esp-dl` 到本地 |
| SD 卡模式启动失败 | `app_main` 必须先 `bsp_sdcard_mount()`；`CONFIG_BSP_SD_FORMAT_ON_MOUNT_FAIL=y`；模型放 `models/p4` 或 `models/s3` |
| 分区不够 / 烧录溢出 | 大模型用 `partitions2.csv`（factory 2000K + custom_det 4M）；或改 sdcard 模式 |
| rect 训练形状不匹配 | 预训练阶段用 `imgsz=max(h,w)` 且 `rect=False`；微调阶段才 `imgsz=[h,w]`, `rect=True`, `epochs=30~50` |

## References

- 场景 recipes → `recipes/` 目录
- Python 训练/导出/量化 API → `resources/api_reference.md`
- Kconfig / sdkconfig / YAML 配置 → `resources/config_reference.md`
- 流水线状态机与阶段产物 → `resources/model_pipeline.md`
- 仓库内示例与模板索引 → `resources/example_list.md`
- 汇总常见陷阱 → `resources/pitfalls.md`
- 仓库源码 → `D:/esp-skill/espressif-repos/esp-detection/`（不可修改）
