---
name: esp-dl-skill
description: >-
  AI Skill for ESP-DL deep learning inference framework development on ESP chips. Used when users need to
  deploy, load, run, quantize, or debug neural network models (.espdl) on ESP32 / ESP32-S3 / ESP32-P4 and other
  ESP targets, including model quantization with ESP-PPQ, inference profiling, vision/audio post-processing,
  streaming models, and per-tensor / per-channel quantization.
  Trigger words: "ESP-DL", "espdl", "ESP-PPQ", "model quantization", "模型量化", "模型部署", "神经网络推理",
  "深度学习", "ESP32", "ESP32-S3", "ESP32-P4", "inference", "deployment"
license: MIT
metadata:
  author: Community
  version: "1.1.0"
---

# esp-dl-skill

AI Skill for the ESP-DL deep learning inference framework. ESP-DL is a lightweight, efficient neural-network inference framework designed specifically for ESP series chips. It provides APIs to load (`.espdl`), debug (`test()` / `profile()`), and run AI models, plus the ESP-PPQ quantization toolchain that converts ONNX / PyTorch models into the ESP-DL standard FlatBuffers-based model format. This skill supplies scenario-driven recipes, complete C++ API reference, real Kconfig/quantization configuration guides, and the most common pitfalls — all grounded in the real `esp-dl` repository documentation, headers, and examples.

## Core Principles

1. **Never invent APIs** — Every `dl::Model` / `dl::TensorBase` method, Kconfig symbol, and `.espdl` location enum must come from `resources/`. If a signature is not documented there, it does not exist — stop and say so.
2. **The unit of deployment is the `.espdl` file** — ESP-DL cannot run ONNX/PyTorch directly. A model must first be quantized and exported by ESP-PPQ (`espdl_quantize_onnx` / `espdl_quantize_torch`) to the FlatBuffers-based `.espdl` format before any on-chip inference.
3. **Pick the right `target` at quantization time** — `target` selects both the quantization strategy and the rounding mode. `c` (ESP32/C-series, C ops, round-half-up, per-tensor), `esp32s3` (PIE V1, round-half-up, per-tensor), `esp32p4` (PIE V2, round-half-even, Conv/Gemm per-channel). A model quantized for one target will give wrong results on another.
4. **`dl::Model` construction does everything** — The `Model` constructor loads the file, builds the execution plan, and runs the static memory planner. There is no separate "init" call. Inputs/outputs memory is already allocated after construction.
5. **Inputs must be quantized, outputs must be dequantized** — 8-bit models take `int8_t`, 16-bit models take `int16_t`. Float inputs go through `dl::quantize<T>(value, DL_RESCALE(exp))`; int outputs come back through `dl::dequantize(q, DL_SCALE(exp))`. `Scale = 2^exp`.
6. **The memory planner reuses buffers** — All inputs, intermediate tensors, and outputs share one planned memory block. After `run()`, reading `model_input->data` may already be overwritten by an output or intermediate tensor. Always copy what you need before re-running, or re-fetch via `get_outputs()`.
7. **`max_internal_size` trades speed for internal RAM** — The constructor's `max_internal_size` (bytes) caps internal-RAM use and spills the rest to PSRAM. Setting it `0` (and `param_copy=false`) keeps everything in FLASH/PSRAM, saving RAM but slowing inference.
8. **16-byte alignment is mandatory** — `TensorBase` requires its first address to be 16-byte aligned and the memory size to be a multiple of 16 bytes. Test values in the `.info` file are also 16-byte aligned (zero-padded).
9. **Only `batch_size = 1`** — ESP-DL does not support multi-batch or dynamic batch. Quantize and deploy with `batch_size = 1`.
10. **Dual-core scheduling is opt-in via `runtime_mode_t`** — `model->run()` defaults to `RUNTIME_MODE_SINGLE_CORE`. Pass `RUNTIME_MODE_MULTI_CORE` (or `RUNTIME_MODE_AUTO`) to use both cores for the supported heavy ops (Conv2D, DepthwiseConv2D).
11. **Activation functions use an 8-bit LUT** — All activations except ReLU and PReLU are LUT-based, so any activation has roughly the same cost. Do not hand-replace sigmoid/tanh with approximations "for speed".
12. **`test()` requires test values at export** — `model->test()` only works if ESP-PPQ exported the `.espdl` with `export_test_values=True`. For deployment, re-export without test data to shrink the model.

## When to Use

**Applicable:**
- Quantizing an ONNX / PyTorch / TensorFlow (via ONNX) model to `.espdl` for an ESP chip
- Loading a `.espdl` model from FLASH rodata, a SPIFFS partition, or SD card and running inference
- Writing C++ firmware that calls `dl::Model::run()` and processes `int8_t`/`int16_t` tensors
- Debugging quantization accuracy, choosing `target`, bit-width, or per-channel vs per-tensor
- Profiling model latency (`profile_module`) and memory (`profile_memory`) on chip
- Deploying vision detection/classification/recognition pipelines (YOLO11, MobileNetV2, etc.)
- Deploying streaming/time-series (audio) models with chunked input
- Auto-searching quantization configs with AutoQuant

**Not applicable:**
- Training models from scratch (use PyTorch/TensorFlow outside ESP-DL; ESP-DL only deploys)
- Pure ESP-IDF / driver questions unrelated to AI inference
- Running `.tflite` or `.onnx` directly on chip — convert to `.espdl` first
- Non-ESP platforms (STM32, RP2040, etc.)

---

## Scenario Quick Reference (Recipes)

When user intent matches a scenario below, **read the corresponding recipe first** — it contains the complete call chain, step-by-step instructions, common errors, and real code adapted from the repo.

### Setup & Loading

| recipe | scenario |
|---|---|
| `recipes/project_setup.md` | Add esp-dl as an ESP-IDF managed component and configure the project (`idf_component.yml`, `CMakeLists.txt`) |
| `recipes/load_model_rodata.md` | Embed a `.espdl` in `.rodata` and load it (simplest, re-flashes model with app) |
| `recipes/load_model_partition.md` | Store the model in a SPIFFS partition; flash independently with `idf.py app-flash` |
| `recipes/load_model_sdcard.md` | Load the model from an SD card (FAT32) for easy model swapping |

### Inference

| recipe | scenario |
|---|---|
| `recipes/run_inference.md` | Core inference loop: get inputs/outputs, quantize input, `run()`, dequantize output |
| `recipes/streaming_model.md` | Deploy a streaming/time-series (audio) model — chunk input, `run()` per chunk, internal state auto-managed |
| `recipes/profile_test.md` | Use `test()`, `profile_memory()`, `profile_module()`, `profile()` to validate and tune a model |

### Quantization (ESP-PPQ)

| recipe | scenario |
|---|---|
| `recipes/quantize_onnx.md` | Quantize an ONNX model to `.espdl` with `espdl_quantize_onnx` (PTQ, choose target/bits) |
| `recipes/quantize_torch.md` | Quantize a PyTorch model with `espdl_quantize_torch` |
| `recipes/auto_quant.md` | Use AutoQuant (`espdl_auto_quantize_onnx`) to auto-search optimal quantization strategies |
| `recipes/quantize_tqt.md` | Use TQT (Trained Quantization Thresholds) to learn scale=2^k + finetune weights without labels — the next step when default PTQ accuracy is insufficient |
| `recipes/quantize_advanced_ptq.md` | Deterministic accuracy recovery on per-tensor targets (S3): dispatch worst layers to int16 (mixed precision) and layerwise weight equalization |

### Vision

| recipe | scenario |
|---|---|
| `recipes/vision_detection.md` | Run a YOLO11/COCO detection model end-to-end: JPEG decode → model → post-process (DetectWrapper, box+score) |
| `recipes/vision_pose.md` | Run a YOLO11n-pose keypoint model end-to-end: JPEG decode → `COCOPose` → 17 COCO keypoints per instance (yolo11posePostProcessor) |
| `recipes/vision_classification.md` | Run a MobileNetV2/ImageNet classifier end-to-end: JPEG decode → `ImageNetCls` → Top-K (cat_name, score), no boxes (ClsWrapper/ImageNetClsPostprocessor) |
| `recipes/vision_segmentation.md` | Run a YOLO11n-seg instance-segmentation model end-to-end: JPEG decode → `COCOSeg` → per-instance mask raster (yolo11segPostProcessor), alpha-blend + BMP export |

---

## Chip & Quantization Support

| Target | PIE | Rounding | Conv/Gemm quant | Notes |
|---|---|---|---|---|
| ESP32, ESP32-C2/C3/C5/C6, ESP32-S2 | none (C ops) | round-half-up | per-tensor | Slowest; set `target = "c"` in ESP-PPQ, build `idf.py set-target esp32` etc. |
| ESP32-S3 | PIE V1 | round-half-up | per-tensor | Recommended low-cost AI target |
| ESP32-P4 | PIE V2 | round-half-even | per-channel (Conv/Gemm); others per-tensor | Best AI performance; needs ESP-PPQ >= 1.2.10 & ESP-DL >= 3.3.1 for per-channel |
| ESP32-S31 | PIE V2 | round-half-even | per-channel | Requires ESP-IDF >= 6.0 |

ESP-IDF version requirements: `>= 5.3` for most targets, `>= 5.5` for ESP32-C5, `>= 6.0` for ESP32-S31.

## Model Location Enums (`fbs::model_location_type_t`)

| Enum | Use | Constructor 1st arg |
|---|---|---|
| `MODEL_LOCATION_IN_FLASH_RODATA` (=0) | Model embedded in app `.rodata` | rodata symbol address (from `asm("_binary_<name>_espdl_start")`) |
| `MODEL_LOCATION_IN_FLASH_PARTITION` (=1) | Model in a SPIFFS `data` partition | partition label string (must match `partition.csv` Name) |
| `MODEL_LOCATION_IN_SDCARD` (=2) | Model file on FAT32 SD card | VFS path, e.g. `"/sdcard/model.espdl"` |

## Key Runtime Types

| Type / Enum | Header | Purpose |
|---|---|---|
| `dl::Model` | `dl_model_base.hpp` | Load + build + run a model |
| `dl::TensorBase` | `dl_tensor_base.hpp` | The tensor container (inputs/outputs/intermediates) |
| `dl::ExponentInfo` | `dl_tensor_base.hpp` | Per-tensor or per-channel exponent |
| `dl::MemoryManagerBase` / `MEMORY_MANAGER_GREEDY` | `dl_memory_manager.hpp` | Static memory planner (only GREEDY implemented) |
| `dtype_t` (`DATA_TYPE_INT8`, `DATA_TYPE_INT16`, `DATA_TYPE_FLOAT`, ...) | `dl_tensor_base.hpp` | Tensor element dtype (mirrors FlatBuffers dtype) |
| `runtime_mode_t` (`RUNTIME_MODE_SINGLE_CORE`, `RUNTIME_MODE_MULTI_CORE`, `RUNTIME_MODE_AUTO`) | `dl_define.hpp` | Core scheduling for `run()` |
| `dl::quantize<RT>(input, inv_scale)` / `dl::dequantize<T>(input, scale)` | `dl_tensor_base.hpp` | Quantize/dequantize helpers |
| `DL_SCALE(exp)` / `DL_RESCALE(exp)` | `dl_define.hpp` | `2^exp` / `1/2^exp` |
| `dl::image::img_t`, `pix_type_t` | `dl_image_define.hpp` | Decoded image container + pixel formats |
| `dl::image::sw_decode_jpeg` / `hw_decode_jpeg` | `dl_image_jpeg.hpp` | Software / hardware JPEG decode |

---

## Critical Pitfalls (Must Read)

These are the most common errors. Violating any of these produces wrong inference results or build/runtime failures.

### 1. ESP-DL cannot run ONNX/PyTorch directly — quantize to `.espdl` first

```cpp
// ❌ WRONG — trying to feed an ONNX path to dl::Model
dl::Model *model = new dl::Model("model.onnx", fbs::MODEL_LOCATION_IN_SDCARD);

// ✅ CORRECT — quantize with ESP-PPQ first, then load the .espdl
// (python) espdl_quantize_onnx(onnx_import_file="model.onnx",
//                              espdl_export_file="model.espdl", target="esp32s3", ...)
dl::Model *model = new dl::Model("/sdcard/model.espdl", fbs::MODEL_LOCATION_IN_SDCARD);
```

### 2. `target` at quantization must match the chip you deploy on

```python
# ❌ WRONG — quantize for P4, run on S3 → inaccurate results
espdl_quantize_onnx(..., target="esp32p4")   # then deploy on esp32s3

# ✅ CORRECT — match target and build target
# For ESP32-S3:
espdl_quantize_onnx(..., target="esp32s3")
# then:  idf.py set-target esp32s3
# For ESP32 (C ops): target="c", then: idf.py set-target esp32
```

### 3. Feed quantized input, not float, into the model's input tensor

```cpp
// ❌ WRONG — writing float directly into model_input->data
float *p = (float *)model_input->data;
*p = 1.57f;
model->run();

// ✅ CORRECT — quantize using the input's exponent
int8_t q = dl::quantize<int8_t>(1.57f, DL_RESCALE(model_input->exponent));
*((int8_t *)model_input->data) = q;
model->run();
// or use TensorBase::assign to quantize a whole float tensor:
// model_input->assign(float_tensor);
```

### 4. Read output BEFORE re-running — the memory planner reuses buffers

```cpp
// ❌ WRONG — re-running overwrites the output you wanted
model->run();
model->run();                     // overwrites first output's buffer
float v = dl::dequantize(*(int8_t*)model_output->data, DL_SCALE(model_output->exponent));

// ✅ CORRECT — copy/dequantize the value you need right after run()
model->run();
int8_t q = *((int8_t *)model_output->data);
float v = dl::dequantize(q, DL_SCALE(model_output->exponent));
// now safe to run again
```

### 5. `quantize` takes INVERSE scale (`DL_RESCALE`), `dequantize` takes scale (`DL_SCALE`)

```cpp
// ❌ WRONG — passing scale to quantize (or inverse scale to dequantize)
int8_t q = dl::quantize<int8_t>(v, DL_SCALE(model_input->exponent));
float  r = dl::dequantize(q, DL_RESCALE(model_output->exponent));

// ✅ CORRECT
int8_t q = dl::quantize<int8_t>(v, DL_RESCALE(model_input->exponent));
float  r = dl::dequantize(q, DL_SCALE(model_output->exponent));
```

### 6. The rodata symbol name is derived from the filename — `.` → `_`

```cpp
// ❌ WRONG — symbol name does not match the file "model.espdl"
extern const uint8_t model[] asm("_binary_model_start");

// ✅ CORRECT — "_binary_" + filename with dots->underscores + "_start"
extern const uint8_t model_espdl[] asm("_binary_model_espdl_start");
dl::Model *model = new dl::Model((const char *)model_espdl, fbs::MODEL_LOCATION_IN_FLASH_RODATA);
```

### 7. Partition label must match `partition.csv` Name (≤ 16 chars incl. null)

```cpp
// ❌ WRONG — partition label mismatch
// partition.csv:  model, data, spiffs, ...
dl::Model *m = new dl::Model("mymodel", fbs::MODEL_LOCATION_IN_FLASH_PARTITION);

// ✅ CORRECT — names agree, SubType is "spiffs"
// partition.csv:  model, data, spiffs, , 4000K,
dl::Model *m = new dl::Model("model", fbs::MODEL_LOCATION_IN_FLASH_PARTITION);
// and in CMakeLists.txt:
//   esptool_py_flash_to_partition(flash "model" "${image_file}")
```

### 8. `target_add_aligned_binary_data` for `.espdl`, not `EMBED_FILES`

```cmake
# ❌ WRONG — EMBED_FILES does not guarantee the 16-byte alignment .espdl needs
idf_component_register(... EMBED_FILES model.espdl)

# ✅ CORRECT — use the esp-dl helper from fbs_loader/cmake/utilities.cmake
idf_build_get_property(component_targets __COMPONENT_TARGETS)
if ("___idf_espressif__esp-dl" IN_LIST component_targets)
   idf_component_get_property(espdl_dir espressif__esp-dl COMPONENT_DIR)
elseif("___idf_esp-dl" IN_LIST component_targets)
   idf_component_get_property(espdl_dir esp-dl COMPONENT_DIR)
endif()
include(${espdl_dir}/fbs_loader/cmake/utilities.cmake)
idf_component_register(SRCS app_main.cpp REQUIRES esp-dl)
target_add_aligned_binary_data(${COMPONENT_LIB} model.espdl BINARY)
```

### 9. `test()` only works if you exported with `export_test_values=True`

```cpp
// ❌ WRONG — calling test() on a deployment model without test data
ESP_ERROR_CHECK(model->test());   // returns ESP_FAIL

// ✅ CORRECT — export with test values for validation, without for deployment
// python: espdl_quantize_onnx(..., export_test_values=True)   # for test()
// then for production re-export with export_test_values=False to save size
esp_err_t r = model->test();
if (r == ESP_OK) ESP_LOGI(TAG, "passed");
```

### 10. Use `RUNTIME_MODE_MULTI_CORE` to actually use the second core

```cpp
// ❌ WRONG — defaults to single core, wastes ESP32-S3/P4 second core
model->run();

// ✅ CORRECT — explicitly request dual core (Conv2D / DepthwiseConv2D benefit)
model->run(dl::RUNTIME_MODE_MULTI_CORE);
// or RUNTIME_MODE_AUTO to let the framework decide
```

### 11. INT16 outputs allow ±1 in `test()` comparison

```cpp
// ❌ WRONG — assuming INT16 board output is bit-exact vs PC reference
// (quantization rounding makes ±1 legal per the docs)

// ✅ CORRECT — for INT16 models, model->test() tolerates ±1 per element;
// a passing test() is sufficient. Do not override the comparison manually.
ESP_ERROR_CHECK(model->test());
```

### 12. ESP32 (non-PIE) is much slower — choose S3/P4 for real workloads

```
// ❌ WRONG — expecting real-time detection on plain ESP32
// target = "c" compiles operators in plain C; no PIE acceleration.

// ✅ CORRECT — use ESP32-S3 (PIE V1) or ESP32-P4 (PIE V2) for AI workloads.
// Keep ESP32/C-series only for tiny models or prototyping.
```

### 13. Each vision task has its own post-processor — do not reuse the detection wrapper

ESP-DL vision tasks share `dl::detect::result_t` / `dl::cls::result_t` but use **task-specific wrappers and post-processors**. Picking the wrong one silently drops task-specific outputs (keypoints, masks) or mis-shapes the result.

```cpp
// ❌ WRONG — using the detection wrapper for pose/segmentation/classification
COCODetect *m = new COCODetect();   // has NO keypoint / mask / cat_name fields
m->run(img);

// ✅ CORRECT — match the wrapper to the task
COCOPose    *pose = new COCOPose();    // dl::detect::yolo11posePostprocessor → result_t.keypoint (17×2)
COCOSeg     *seg  = new COCOSeg();     // dl::detect::yolo11segPostprocessor  → result_t.mask (per-instance raster)
ImageNetCls *cls  = new ImageNetCls(); // dl::cls::ImageNetClsPostprocessor  → cls::result_t{cat_name, score}, NO box
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | Identify the target chip (drives quantization `target`), model framework (ONNX/PyTorch), and deployment location (rodata/partition/sdcard) |
| 2 | Recipe | Match the user intent to a recipe in `recipes/` and follow its call chain |
| 3 | Query | For any API/Kconfig not in the recipe, look it up in `resources/` (api_reference.md, config_reference.md) |
| 4 | Quantize | If the user has no `.espdl` yet: run ESP-PPQ (`espdl_quantize_onnx`/`_torch`) or AutoQuant with the correct `target` |
| 5 | Validate | Confirm: target matches chip, `idf_component.yml` lists esp-dl, `CMakeLists.txt` uses `target_add_aligned_binary_data`, inputs are quantized, outputs dequantized |
| 6 | Confirm | Present the plan: includes, location enum, constructor args, `run()` mode, post-processing |
| 7 | Execute | **New project:** copy the closest real example (e.g. `examples/tutorial/how_to_run_model`, `examples/yolo11_detect`) and modify. **Existing project:** edit in place. |
| 8 | Build | `idf.py set-target <chip>` then `idf.py build` (C++ component; main file can be `.cpp`) |
| 9 | Profile | Flash with `idf.py flash monitor`; in code call `model->profile()` to inspect latency/memory |
| 10 | Debug | If results are wrong, first check `model->test()` passes, then verify the `target`/chip match and quantize/dequantize exponent usage |

### Step 7 Detail — Project Creation Strategy

**When creating a new project:**

1. Pick the closest real example by goal:
   - Bare inference / learning the API → `examples/tutorial/how_to_run_model`
   - Loading + test + profile (rodata / partition / sdcard variants) → `examples/tutorial/how_to_load_test_profile_model/*`
   - Streaming / audio model → `examples/tutorial/how_to_deploy_streaming_model`
   - YOLO object detection → `examples/yolo11_detect`, `examples/yolo26_detect`, `examples/pedestrian_detect`, `examples/cat_detect`, `examples/dog_detect`, `examples/hand_detect`
   - YOLO11 pose / segmentation → `examples/yolo11_pose`, `examples/yolo11_seg`
   - Classification → `examples/mobilenetv2_cls`
   - Recognition / face / speaker → `examples/human_face_recognition`, `examples/speaker_verification`
   - Motion / color → `examples/motion_detect`, `examples/color_detect`
   - Quantization scripts (Python) → `examples/tutorial/how_to_quantize_model/quantize_sin_model`, `.../quantize_mobilenetv2`

2. Copy the example directory and its `main/CMakeLists.txt`, `partitions.csv`, `sdkconfig.defaults*` into the new project.

3. Swap the model file and adjust `set(embed_files ...)` / partition label / constructor args to match the new model.

**When editing an existing project:** modify files in place; never overwrite the esp-dl component itself.

---

## Failure Strategies

| Situation | Action |
|---|---|
| API not found in `resources/` | Stop; tell the user the API does not exist in this ESP-DL version |
| `model->test()` returns `ESP_FAIL` | Re-export `.espdl` with `export_test_values=True`; verify `target` matches chip; for INT16, ±1 tolerance is allowed |
| Inference result wrong/inaccurate | Check (a) `target` == deployed chip, (b) input quantized with `DL_RESCALE(input->exponent)`, (c) output dequantized with `DL_SCALE(output->exponent)`, (d) pre-processing matches training |
| `partition.csv` model not found | Partition label in constructor must equal the `Name` column; SubType must be `spiffs` |
| App partition too small for rodata model | Increase app partition size in `partitions.csv`, or move model to a partition / SD card |
| Flashing too slow (large rodata model) | Switch to partition or SD-card loading; use `idf.py app-flash` to skip the model partition |
| Out of internal RAM / PSRAM | Set constructor `max_internal_size` to a small value, or `param_copy=false` to keep params in FLASH |
| Unsupported ONNX operator | Check `operator_support_state.md`; request it in the repo issues or implement via the operator skill |

## References

- Scenario recipes → `recipes/` directory
- Model / Tensor / Memory C++ API → `resources/api_reference.md`
- Kconfig + ESP-PPQ quantization params → `resources/config_reference.md`
- Consolidated gotchas → `resources/pitfalls.md`
- Real example index → `resources/example_list.md`
- Supported operators (64) → `operator_support_state.md` in the repo root
- Quantization tutorials → `docs/en/tutorials/how_to_quantize_model.rst`, `how_to_deploy_mobilenetv2.rst`, `how_to_deploy_yolo11n.rst`, `quantize_model_with_TQT.rst`
- Vision deployment tutorials → `docs/en/tutorials/how_to_deploy_yolo11n.rst`, `how_to_deploy_yolo11n-pose.rst`, `how_to_deploy_mobilenetv2.rst`
- Load/test/profile tutorial → `docs/en/tutorials/how_to_load_test_profile_model.rst`
- Inference tutorial → `docs/en/tutorials/how_to_run_model.rst`
