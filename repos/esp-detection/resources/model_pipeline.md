# 模型-部署流水线（状态机与阶段产物）

> 描述 esp-detection 从数据到芯片推理的完整阶段、每阶段入口函数、产物文件、关键约束。

## 总览状态机

```
[数据] cfg/datasets/*.yaml + YOLO 格式 images/labels
   │
   ▼  train.py::Train()  (先 tasks.parse_model = custom_parse_model)
[.pt] best.pt  (PyTorch 浮点权重, results.save_dir/weights/best.pt)
   │
   ├─ (可选) val.py / model.val()  →  浮点 mAP
   │
   ▼  deploy/export.py::Export()  (绑定 ESPDetect.export_onnx_forward, opset=13, onnxsim)
[.onnx] best.onnx  (静态 batch=1, 6 输出: box0,score0,box1,score1,box2,score2)
   │
   ▼  deploy/quantize.py::quant_espdet()  (esp-ppq PTQ, INT8, Equalization, calib_steps=32)
[.espdl] <name>.espdl  (INT8 FlatBuffers, 按 target 选 p4/s3)
   │
   ├─ (可选) deploy/eval_quantized_model.py  →  量化 mAP
   │
   ▼  espdet_run.py::run()  (git clone esp-dl → 复制 deploy/*_template → rename_project → 拷 espdl+img)
[ESP-IDF 工程] esp-dl/{examples,models}/<class>_detect/
   │
   ▼  idf.py set-target esp32p4|esp32s3 && idf.py flash monitor
[芯片推理] ESPDetDetect::run(img) → vector<{category,score,box[4]}>
```

## 阶段详表

| 阶段 | 入口 | 输入 | 输出 | 关键约束 |
|---|---|---|---|---|
| 数据准备 | 手工 + `cfg/datasets/*.yaml` | 原始图 + 标注 | YOLO 格式目录 + yaml | `nc` 与 `names` 一致；负样本需配 `negative_setting` |
| 训练 | `train.py::Train(...)` | yaml +（可选）预训练 .pt | `best.pt` | 必须 `tasks.parse_model = custom_parse_model`；rect 时 `imgsz=[h,w]` |
| 浮点评估（可选） | `val.py` / `model.val()` | best.pt + yaml | mAP 指标 | 作为量化前基线 |
| 导出 | `deploy/export.py::Export()` | best.pt | best.onnx | opset=13；onnxsim；6 输出；batch=1 静态 |
| 量化 | `deploy/quantize.py::quant_espdet()` | best.onnx + calib_dir | `<name>.espdl` | target 与芯片一致；Equalization 默认 iter=4 |
| 量化评估（可选） | `deploy/eval_quantized_model.py` | espdl/ppq graph + yaml | 量化 mAP | `Detect(reg_max=1,end2end=False,ch=[32,64,128],stride=[8,16,32])` |
| 工程生成 | `espdet_run.py::run()` | espdl + img + class_name + size | `esp-dl/{examples,models}/<class>_detect/` | rename_project 占位符替换；保护 add_custom_command |
| 编译烧录 | `idf.py` | 工程 + ESP-IDF v5.3+ | 固件 | Kconfig 选 model location；分区表匹配 |
| 芯片推理 | `ESPDetDetect::run(img)` | JPEG 嵌入图 | 检测框列表 | letterbox + (S3: RGB565) + anchors 不可改 |

## 一站式 vs 分步

| 方式 | 命令 | 适用 |
|---|---|---|
| 一站式 | `python espdet_run.py --class_name ... --size H W --target ...` | 标准流程，无需中间干预 |
| 分步训练 | `python train.py`（或 `from train import Train`） | 需要精细控制 epochs/batch/设备 |
| 分步导出 | `from deploy.export import Export; Export(pt, size)` | 仅导出，复用量化参数实验 |
| 分步量化 | `from deploy.quantize import quant_espdet; quant_espdet(...)` | 调 Equalization / 换校准集 |
| 分步工程生成 | 复制 `deploy/espdet_*_template` + 手动 `rename_project` | `espdet_run.py` 的 git clone 失败时 |

## 产物文件命名约定

| 产物 | 命名 | 示例 |
|---|---|---|
| 浮点权重 | `best.pt`（固定） | `runs/detect/train/weights/best.pt` |
| ONNX | 与 .pt 同名换后缀 | `best.onnx` |
| ESPDL | `espdet_pico_<H>_<W>_<class>.espdl` | `espdet_pico_224_224_mycat.espdl` |
| 模型组件目录 | `models/<class>_detect/` | `models/mycat_detect/` |
| 示例工程目录 | `examples/<class>_detect/` | `examples/mycat_detect/` |
| 嵌入测试图 | `main/<img>`（由 `--img` 指定） | `main/espdet.jpg` |

## rect 路径 vs 方形路径

| 维度 | 方形 | rect（非方形） |
|---|---|---|
| `--size` | `S S`（如 `224 224`） | `H W`（如 `160 288`，H≠W） |
| Train | `imgsz=S`, `rect=False` | `imgsz=[H,W]`, `rect=True`（可选先方形预训练 `imgsz=max(H,W)`, `rect=False`） |
| Export | `Export(pt, S)` | `Export(pt, [H,W])` |
| quantize | `imgsz=S` | `imgsz=[H,W]` |
| 评估 | `imgsz=S` | `imgsz=[H,W]` |
| 精度/速度 | 见 SKILL.md 芯片表 | 同尺寸通常更快且精度更优 |

## 参考

- `espdet_run.py`（`run()` 编排全流程）
- `train.py`、`deploy/export.py`、`deploy/quantize.py`、`deploy/eval_quantized_model.py`
- `docs/tutorials/how_to_train_and_deploy_model_with_rect_is_True.md`
- `recipes/all_in_one_pipeline.md`（端到端细节）
