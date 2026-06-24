# ESP-DL 真实示例索引

> 路径相对仓库根 `esp-dl/`。描述取自各示例 README / 代码 / `idf_component.yml`。`[S3]/[P4]` 表示该示例提供对应芯片的模型与编译配置。

## 教程示例（`examples/tutorial/`）

| 路径 | 说明 |
|---|---|
| `examples/tutorial/how_to_run_model` | 最基本的模型推理示例（sin 模型）：展示 `run()` 的三种用法（单值量化、`assign`、`run(user_inputs,...)`）。含 S3/P4 模型，rodata 加载 |
| `examples/tutorial/how_to_load_test_profile_model/model_in_flash_rodata` | 从 `.rodata` 加载、`test()`、`profile()` |
| `examples/tutorial/how_to_load_test_profile_model/model_in_flash_partition` | 从 SPIFFS 分区加载（`param_copy=false`），`test()` + `profile()` |
| `examples/tutorial/how_to_load_test_profile_model/model_in_sdcard` | 从 SD 卡加载，`test()` + `profile()` |
| `examples/tutorial/how_to_deploy_streaming_model` | 流式（时序/音频，TCN）模型量化与部署：`auto_streaming`、chunk 推理 |
| `examples/tutorial/how_to_quantize_model/quantize_sin_model` | 用 `espdl_quantize_onnx`/`_torch` 量化 sin 模型的最小示例（`sin_model.py`、`quantize_onnx_model.py`、`quantize_torch_model.py`） |
| `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2` | MobileNetV2 量化示例（`quantize_onnx_model.py`、`quantize_torch_model.py`、`quantize_tf_model.py`） |
| `examples/tutorial/how_to_quantize_model/quantize_yolo26` | YOLO26 训练与量化（含 TQT + 聚焦 logit，见 `quantize_tqt.md`） |
| `examples/tutorial/how_to_quantize_model/quantize_yolo11n-pose` | YOLO11n-pose 量化：`quantize_onnx_model.py`（PTQ）、`yolo11n_pose_qat.py` + `trainer.py`（QAT）、`yolo11n_pose_eval.py` |
| `examples/tutorial/how_to_quantize_model/quantize_yolo11n_seg` | YOLO11n-seg 量化 |
| `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2` | MobileNetV2 量化（含 TQT、mixed precision、layerwise equalization 分支，见 `quantize_advanced_ptq.md` / `quantize_tqt.md`） |

## 视觉：检测（`examples/`）

| 路径 | 说明 |
|---|---|
| `examples/yolo11_detect` | YOLO11n COCO 目标检测：JPEG 解码 → `COCODetect` → 结果（S3/P4） |
| `examples/yolo11_pose` | YOLO11n 姿态估计：JPEG 解码 → `COCOPose` → 17 COCO 关键点（见 `recipes/vision_pose.md`） |
| `examples/yolo11_seg` | YOLO11n 实例分割：JPEG 解码 → `COCOSeg` → 逐实例 mask + BMP 导出（见 `recipes/vision_segmentation.md`） |
| `examples/yolo26_detect` | YOLO26 目标检测（S3/P4） |
| `examples/cat_detect` | ESPDet-Pico 猫检测模型（esp-detection 训练） |
| `examples/dog_detect` | 狗检测 |
| `examples/pedestrian_detect` | 行人检测 |
| `examples/hand_detect` | 手部检测 |
| `examples/color_detect` | 颜色检测 |
| `examples/motion_detect` | 运动检测 |

## 视觉：分类 / 识别 / 手势（`examples/`）

| 路径 | 说明 |
|---|---|
| `examples/mobilenetv2_cls` | MobileNetV2 ImageNet 分类：JPEG 解码 → `ImageNetCls` → Top-K（见 `recipes/vision_classification.md`） |
| `examples/human_face_detect` | 人脸检测 |
| `examples/human_face_recognition` | 人脸识别 |
| `examples/hand_gesture_recognition` | 手势识别 |
| `examples/speaker_verification` | 说话人验证（音频） |

## 模型库（`models/`，作为 ESP-IDF 组件提供 `.espdl`）

> 每个模型组件提供量化后的 `.espdl` 与后处理 Wrapper 类。例如 `models/coco_detect/`（`COCODetect`）、`models/cat_detect/`、`models/coco_detect` 对应 `examples/yolo11_detect`。完整列表见仓库 `models/README.md`。

## 模型存放方式（示例中的 Kconfig 选择）

部分示例（如 `yolo11_detect`）通过 Kconfig（如 `CONFIG_COCO_DETECT_MODEL_IN_SDCARD`）在 rodata / SD 卡之间切换模型来源，SD 卡分支需先 `bsp_sdcard_mount()`。

## 参考脚本（量化）

| 路径 | 说明 |
|---|---|
| `examples/tutorial/how_to_quantize_model/quantize_sin_model/quantize_onnx_model.py` | `espdl_quantize_onnx` 最小用法 |
| `examples/tutorial/how_to_quantize_model/quantize_sin_model/quantize_torch_model.py` | `espdl_quantize_torch` 最小用法 |
| `examples/tutorial/how_to_deploy_streaming_model/quantize_streaming_model/quantize_torch_model.py` | `auto_streaming=True` 流式量化 |
| `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/quantize_tf_model.py` | TensorFlow → ONNX → 量化 |
| `examples/tutorial/how_to_quantize_model/quantize_mobilenetv2/quantize_onnx_model.py` | MobileNetV2 含 TQT / mixed precision / layerwise equalization 分支（`quant_setting_mobilenet_v2`） |
| `examples/tutorial/how_to_quantize_model/quantize_yolo26/quantize_onnx_model.py` | YOLO26n TQT + 聚焦 logit 量化（见 `recipes/quantize_tqt.md`） |
| `examples/tutorial/how_to_quantize_model/quantize_yolo11n-pose/quantize_onnx_model.py` | YOLO11n-pose PTQ 量化 |
| `examples/tutorial/how_to_quantize_model/quantize_yolo11n-pose/yolo11n_pose_qat.py` | YOLO11n-pose QAT（含 `trainer.py`） |
| ESP-PPQ 样例 | `python -m esp_ppq.samples.AutoQuant.mobilenetv2` / `...yolo11n` |
