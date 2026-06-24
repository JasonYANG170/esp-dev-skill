# 汇总常见陷阱（pitfalls）

> SKILL.md 的 "Critical Pitfalls" 逐条带代码示例；本文件按主题归类，便于快速检索。所有条目均来自仓库源码的约束。

## A. 自定义模块解析（最高频）

1. **必须先 `tasks.parse_model = custom_parse_model`** —— 否则 `DSConv/ESPBlock/ESPDetect` 等模块名无法被 ultralytics 解析，报 `KeyError` 或构建出错误结构。`train.py`、`deploy/export.py`、`deploy/eval_quantized_model.py` 入口均已包含；自定义脚本务必补上。
2. **导出必须用 `deploy/export.py::Export()`** —— 原生 `model.export()` 不会绑定 `ESPDetect.export_onnx_forward`，ONNX 输出不是 6 个分离张量，量化后处理会错乱。

## B. 分辨率与 rect

3. **`espdet_run.py --size` 必须两个值**（nargs=2），顺序 `[h w]`。
4. **`rect=True` 时 `imgsz` 必须是 `[h, w]` 列表**（h≠w），标量会歧义。
5. **rect 两阶段**：预训练用 `imgsz=max(h,w)` 且 `rect=False`；微调才 `imgsz=[h,w]`, `rect=True`, `epochs=30~50`。
6. **导出/量化/评估的 `imgsz` 必须与训练一致**，否则形状/语义错位。

## C. ONNX 导出

7. **opset 固定 13**（esp-ppq 默认 18 与 YOLOv11 `arange` 不兼容）。
8. **用 `onnxsim` 简化，不要用 onnxslim**（onnxslim 会把 `NCHW` 改成 `1(N*C)HW`）。
9. **部署模型 batch=1 静态**；`dynamic=True` 仅用于 QAT gt onnx。

## D. 量化

10. **`target` 必须与最终部署芯片一致**（P4/S3），且 `.espdl` 拷到对应 `models/p4` 或 `models/s3`。
11. **校准集不做 ImageNet 归一化** —— 仓库 `CaliDataset` 用 `mean=[0,0,0], std=[1,1,1]`（仅 `ToTensor` + `Resize`）。
12. **校准集后缀限制**：`.jpg/.jpeg/.png/.bmp`，其它被忽略。
13. **精度不足调 Equalization**：`deploy/quantize.py` 默认 `iterations=4 / value_threshold=0.4`；`deploy/eval_quantized_model.py::quant` 用 `10 / 0.3` 作对照，可按需调整。

## E. 芯片端固件

14. **S3 必须启用 `DL_IMAGE_CAP_RGB565_BIG_ENDIAN`**（P4 不加）；`enable_letterbox({114,114,114})` 必须调用。
15. **`ESPDetPostProcessor` anchors 不可改**：`{{8,8,4,4},{16,16,8,8},{32,32,16,16}}`，对应 P3/P4/P5 三个检测层。
16. **默认阈值**：`score_thr=0.25`、`nms_thr=0.7`（`espdet_detect::ESPDet::default_*`）。
17. **模型位置三处一致**：Kconfig 的 `model location` + CMake 分支 + `app_main` 的 `bsp_sdcard_mount()/unmount()`。
18. **分区表匹配**：rodata 用 `partitions.csv`（factory 8000K）；partition 用 `partitions2.csv`（factory 2000K + custom_det 4M）。
19. **BSP 依赖**：S3 用 `esp32_s3_eye_noglib ^3.1.0~1`，P4 用 `esp32_p4_function_ev_board_noglib ^4.0.1`（模板 `main/idf_component.yml`）。
20. **PSRAM 必开**：P4 `CONFIG_SPIRAM_SPEED_200M` + `XIP_FROM_PSRAM` + L2 256KB cache；S3 `SPIRAM_MODE_OCT` + `SPIRAM_SPEED_80M` + 240MHz。

## F. 命名与工程生成

21. **模型文件名与 Kconfig 全程一致**：`espdet_pico_<H>_<W>_<class>.espdl` ↔ `CONFIG_FLASH_ESPDET_PICO_<H>_<W>_<CLASS>` ↔ enum `ESPDET_PICO_<H>_<W>_<CLASS>`。由 `rename_project` 统一替换。
22. **`rename_project` 保护 `add_custom_command` 行**：勿删该保护逻辑，否则 CMake 命令里的路径会被误替换。
23. **`espdet_run.py` 会 `git clone esp-dl`**：已存在目录会 clone 失败，建议先删除或手动 clone 最新版。

## G. 评估

24. **量化评估的 `Detect` 参数固定**：`nc=NC, reg_max=1, end2end=False, ch=[32,64,128], stride=[8,16,32]`。
25. **量化图输出顺序**：`box0,score0,box1,score1,box2,score2`，cat 时偶数索引为 box、奇数为 score。
26. **`end2end=False`**：`QuantizedModelValidator` 内显式设置（针对 espdet_pico）。

## 参考

- `SKILL.md` § Critical Pitfalls（带 WRONG/CORRECT 代码对比）
- `recipes/*.md` 各自的「常见错误」表
- `resources/api_reference.md`、`resources/config_reference.md`
