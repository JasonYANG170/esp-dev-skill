# AGENTS.md — Supplementary Agent Guide

> Core rules, chip/target table, pitfalls, recipe index, and execution workflow are all in `SKILL.md`.
> This file covers **only** conventions and tooling guidance not present in `SKILL.md`. Do not duplicate content.

## Project Context

**Language**: C++ (firmware side) + Python (quantization side) · **Target**: ESP32 family (ESP32, ESP32-S3, ESP32-P4, ESP32-C2/C3/C5/C6, ESP32-S2, ESP32-S31) · **Framework**: ESP-DL (neural-network inference) on ESP-IDF · **Build**: ESP-IDF `idf.py` (CMake), C++ component.

ESP-DL is consumed as an ESP-IDF managed component named `esp-dl` (or `espressif/esp-dl` from the registry). The repo version covered here is `3.3.x`.

## Code Generation Conventions

### File Naming
- Firmware entry point: `main/app_main.cpp` (C++; the symbol called is still `extern "C" void app_main(void)`).
- Headers are `.hpp` (e.g. `dl_model_base.hpp`); model post-processors live in model components like `models/coco_detect/coco_detect.hpp`.
- Model binaries: `*.espdl`. Quantization scripts: `quantize_onnx_model.py` / `quantize_torch_model.py`.

### Include Pattern (firmware)
```cpp
// Core inference
#include "dl_model_base.hpp"     // dl::Model
#include "dl_tensor_base.hpp"    // dl::TensorBase, dtype_t, quantize/dequantize

// Image decode / preprocessing (vision)
#include "dl_image_jpeg.hpp"     // dl::image::sw_decode_jpeg / hw_decode_jpeg
#include "dl_image_define.hpp"   // img_t, jpeg_img_t, pix_type_t

// Optional: a model component's post-processor
#include "coco_detect.hpp"       // COCODetect (wraps dl::detect::DetectWrapper)
```

### Standard Inference Project Structure
```
MyEspDLProject/
├── CMakeLists.txt
├── partitions.csv                  # needed for partition / large rodata models
├── sdkconfig.defaults
├── sdkconfig.defaults.esp32s3
├── sdkconfig.defaults.esp32p4
├── main/
│   ├── CMakeLists.txt              # uses target_add_aligned_binary_data / esptool_py_flash_to_partition
│   ├── idf_component.yml           # dependencies: espressif/esp-dl, plus board BSP if needed
│   ├── app_main.cpp
│   └── models/
│       ├── s3/model.espdl          # per-target model binaries
│       └── p4/model.espdl
```

### Canonical Inference Pattern (single input, single output)
```cpp
#include "dl_model_base.hpp"

extern const uint8_t model_espdl[] asm("_binary_model_espdl_start");

extern "C" void app_main(void)
{
    // 1. Construct -> loads, builds execution plan, runs the memory planner.
    dl::Model *model = new dl::Model((const char *)model_espdl,
                                     fbs::MODEL_LOCATION_IN_FLASH_RODATA);

    // 2. Get the pre-allocated input/output tensors.
    dl::TensorBase *in  = model->get_input();
    dl::TensorBase *out = model->get_output();

    // 3. Quantize float input with DL_RESCALE (inverse scale).
    float x = 1.5f;
    *((int8_t *)in->data) = dl::quantize<int8_t>(x, DL_RESCALE(in->exponent));

    // 4. Run (opt-in dual core).
    model->run(dl::RUNTIME_MODE_MULTI_CORE);

    // 5. Dequantize int output with DL_SCALE.
    int8_t q = *((int8_t *)out->data);
    float y = dl::dequantize(q, DL_SCALE(out->exponent));
    printf("result = %f\n", y);

    delete model;
}
```

### Build Workflow
1. `idf.py set-target esp32s3` (or `esp32p4`, `esp32`, ...) — must match the ESP-PPQ `target` used at quantization.
2. `idf.py menuconfig` — adjust ESP-DL pixel-convert Kconfig options if needed (see `resources/config_reference.md`).
3. `idf.py build`
4. `idf.py flash monitor -p <PORT>`
5. For partition models: `idf.py app-flash` re-flashes only the app (model partition untouched).

### Quantization Workflow (Python, off-chip)
1. Install ESP-PPQ: `pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cpu` then `pip install esp-ppq`.
2. Convert your model to ONNX (if PyTorch/TensorFlow/Paddle).
3. Check every operator is in `operator_support_state.md` (64 supported ops, opset 18 recommended).
4. Run `espdl_quantize_onnx` / `espdl_quantize_torch` with `target` = `c` / `esp32s3` / `esp32p4`, `num_of_bits` = 8 or 16, and `export_test_values=True` for on-chip `model->test()`.
5. Outputs: `*.espdl` (deploy), `*.info` (debug, view test values), `*.json` (quant info).
6. (Optional) Use AutoQuant (`espdl_auto_quantize_onnx`) to auto-search strategies.

## Code Generation Checklist (firmware)

- [ ] `idf_component.yml` lists `espressif/esp-dl` (and any model component / BSP) under `dependencies`
- [ ] `main/CMakeLists.txt` includes `fbs_loader/cmake/utilities.cmake` and uses `target_add_aligned_binary_data` for rodata models (NOT `EMBED_FILES`)
- [ ] Partition models use `esptool_py_flash_to_partition(flash "<Name>" ...)` with the name matching `partitions.csv`
- [ ] rodata symbol name follows `_binary_<filename_dots_to_underscores>_start`
- [ ] `dl::Model` constructor `location` enum matches how the model is stored
- [ ] Input is quantized (`dl::quantize` + `DL_RESCALE`) before `run()`
- [ ] Output is read/dequantized (`dl::dequantize` + `DL_SCALE`) immediately after `run()` (memory planner reuses buffers)
- [ ] `run()` mode chosen deliberately (`RUNTIME_MODE_SINGLE_CORE` default; `MULTI_CORE`/`AUTO` to use both cores)
- [ ] `idf.py set-target` chip matches the ESP-PPQ `target`
- [ ] `app_main.cpp` uses `extern "C" void app_main(void)`
- [ ] `model->test()` / `model->profile()` used during bring-up, removed or guarded for production

## Do Not Modify

- The `esp-dl` component source (`esp-dl/dl/`, `esp-dl/fbs_loader/`, `esp-dl/vision/`, `esp-dl/audio/`) — it is a managed dependency; update the version in `idf_component.yml` instead.
- `SKILL.md` frontmatter (skill metadata).
- Pre-compiled FlatBuffers library under `esp-dl/fbs_loader/lib/`.
