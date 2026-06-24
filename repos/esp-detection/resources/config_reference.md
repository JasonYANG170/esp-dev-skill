# esp-detection 配置参考

> 所有配置项均来自仓库真实文件。修改前请先读对应源文件确认上下文。

## Python 依赖（requirements.txt）

| 包 | 版本约束 | 用途 |
|---|---|---|
| ultralytics | >= 8.3.112 | YOLO 训练/导出框架 |
| torch | == 2.2.0 | 训练后端（Windows 避免 2.4.0） |
| torchvision | == 0.17.0 | 数据变换 |
| onnx | == 1.17.0 | ONNX I/O（与 esp-ppq 兼容） |
| onnxsim | == 0.4.36 | ONNX 简化（替代 onnxslim） |
| onnxruntime | >= 1.19.0 | ONNX 验证 |
| opencv-python | == 4.11.0.86 | 图像处理 |
| numpy | == 1.24.4 | 数值计算 |
| esp-ppq | git main | Espressif 量化（PTQ/QAT） |

## 模型结构 YAML — cfg/models/espdet_pico.yaml

| 字段 | 值 | 含义 |
|---|---|---|
| `nc` | 1 | 类别数（单类检测） |
| `activation` | `'nn.ReLU()'` | 默认激活 |
| `scales.n` | `[0.50, 0.25, 512]` | depth / width / max_channels（pico 规模） |
| backbone[0] | `Conv [64,3,2]` | P1/2 |
| backbone[1] | `DSConv [128,3,2]` | P2/4 |
| backbone[2] | `ESPBlockLite [256, False]` | Lite 块 |
| backbone[3] | `DSConv [256,3,2]` | P3/8 |
| backbone[4] | `DSC3k2 [256, False]` ×2 | DS bottleneck |
| backbone[5] | `SCDown [256,3,2]` | P4/16 |
| backbone[6] | `DSC3k2 [256, True]` ×2 | |
| backbone[7] | `SCDown [512,3,2]` | P5/32 |
| backbone[8] | `DSC3k2 [512, True]` ×2 | |
| backbone[9] | `SPPF [512, 5]` | 空间金字塔池化 |
| backbone[10] | `DSConv [512, 7, 1, 3]` | 大核膨胀 DS |
| head (P4 fuse) | `Upsample` → `Concat(P4 idx6)` → `ESPBlock [256,False]` ×2 (idx13) | FPN |
| head (P3 fuse) | `Upsample` → `Concat(P3 idx4)` → `ESPBlock [128,False]` ×2 (idx16, 小目标) | FPN |
| head (P4 out) | `DSConv [128,3,2]` → `Concat(idx13)` → `ESPBlock [512,False]` ×2 (idx19, 中目标) | PAN |
| head (P5 out) | `DSConv [256,3,2]` → `Concat(idx10)` → `ESPBlock [512,False]` ×2 (idx22, 大目标) | PAN |
| head detect | `ESPDetect [nc]` from `[16,19,22]` | 3 尺度检测头 |

## 数据集 YAML

### cfg/datasets/coco_cat.yaml

```yaml
path: datasets/coco_cat
train: images/train
val: images/val
test:
names:
  0: cat
```

### cfg/datasets/esp_cat.yaml（含 negative sampling）

```yaml
path: esp_cat
train: images/train
val: images/val
test:
negative_setting:
  neg_ratio: 0.111
  use_extra_neg: True
  extra_neg_sources: { "esp_cat/negative_images": 100014 }
  fix_dataset_length: 101241
names:
  0: cat
```

## espdet_run.py CLI 参数

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `--class_name` | str (required) | — | 检测目标类名 |
| `--pretrained_path` | str | None | 预训练 .pt（None/'None' 从 YAML 新建） |
| `--dataset` | str (required) | — | dataset yaml |
| `--size` | int nargs=2 | [224,224] | 输入 `[h w]`；h≠w 自动 rect |
| `--target` | str | esp32p4 | esp32p4 / esp32s3 |
| `--calib_data` | str (required) | — | 校准集目录 |
| `--espdl` | str (required) | — | 输出 .espdl 路径 |
| `--img` | str (required) | — | 芯片端测试图 |

## 芯片端 Kconfig — models/`<class>`_detect/Kconfig

| 符号 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `FLASH_ESPDET_PICO_imgH_imgW_CUSTOM` | bool | y | 把模型打包进固件（depends on ! SDCARD） |
| `ESPDET_PICO_imgH_imgW_CUSTOM` | choice | (默认模型) | 默认模型枚举 |
| `DEFAULT_ESPDET_DETECT_MODEL` | int | 0 | 默认模型 enum 值 |
| `ESPDET_DETECT_MODEL_IN_FLASH_RODATA` | choice | (默认) | 嵌入 flash rodata，LOCATION=0 |
| `ESPDET_DETECT_MODEL_IN_FLASH_PARTITION` | choice | — | SPIFFS 分区 `espdet_det`，LOCATION=1 |
| `ESPDET_DETECT_MODEL_IN_SDCARD` | choice | — | SD 卡，LOCATION=2 |
| `ESPDET_DETECT_MODEL_LOCATION` | int | 0 | 0/1/2，传给 `fbs::model_location_type_t` |
| `ESPDET_DETECT_MODEL_SDCARD_DIR` | string | models/s3 或 models/p4 | SD 卡模型目录（depends on SDCARD） |

> `imgH`/`imgW`/`CUSTOM` 由 `rename_project` 按 `class_name` 与 `size` 替换为具体值，如 `224_224_MYCAT`。

## 芯片端 sdkconfig.defaults（公共）

```
CONFIG_ESPTOOLPY_FLASHMODE_QIO=y
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_COMPILER_OPTIMIZATION_PERF=y
CONFIG_SPIRAM=y
CONFIG_ESP_SYSTEM_ALLOW_RTC_FAST_MEM_AS_HEAP=n
CONFIG_FATFS_LFN_HEAP=y
CONFIG_BSP_SD_FORMAT_ON_MOUNT_FAIL=y
```

### ESP32-P4 专属（sdkconfig.defaults.esp32p4）

```
CONFIG_IDF_TARGET="esp32p4"
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_SPIRAM_SPEED_200M=y
CONFIG_SPIRAM_XIP_FROM_PSRAM=y
CONFIG_CACHE_L2_CACHE_256KB=y
CONFIG_CACHE_L2_CACHE_LINE_128B=y
CONFIG_JD_FASTDECODE_BASIC=y
CONFIG_IDF_EXPERIMENTAL_FEATURES=y
```

### ESP32-S3 专属（sdkconfig.defaults.esp32s3）

```
CONFIG_IDF_TARGET="esp32s3"
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y
CONFIG_ESP32S3_INSTRUCTION_CACHE_32KB=y
CONFIG_ESP32S3_DATA_CACHE_64KB=y
CONFIG_ESP32S3_DATA_CACHE_LINE_64B=y
CONFIG_ESP_TASK_WDT_TIMEOUT_S=40
```

## 分区表

### partitions.csv（rodata 默认）

```
nvs,      data, nvs,     0x9000, 24K,
phy_init, data, phy,     0xf000, 4K,
factory,  app,  factory, 0x010000, 8000K,
```

### partitions2.csv（partition 模式大模型）

```
nvs,        data, nvs,    0x9000, 24K,
phy_init,   data, phy,    0xf000, 4K,
factory,    app,  factory,0x010000, 2000K,
custom_det, data, spiffs, ,        4M,
```

## 组件依赖 idf_component.yml

### models/`<class>`_detect/idf_component.yml（模板）

```yaml
version: "0.1.1"
license: "MIT"
description: <class> detect model.
dependencies:
  espressif/esp-dl:
    version: "^3.1.3"
    override_path: "../../esp-dl"
```

### examples/`<class>`_detect/main/idf_component.yml（模板）

```yaml
dependencies:
  espressif/<class>_detect:
    version: "^0.1.0"
    override_path: "../../../models/<class>_detect"
  espressif/esp32_p4_function_ev_board_noglib:
    version: "^4.0.1"
    rules: [{if: "target == esp32p4"}]
  espressif/esp32_s3_eye_noglib:
    version: "^3.1.0~1"
    rules: [{if: "target == esp32s3"}]
```

## 参考（源文件）

- `requirements.txt`, `pyproject.toml`
- `cfg/models/espdet_pico.yaml`, `cfg/datasets/coco_cat.yaml`, `cfg/datasets/esp_cat.yaml`
- `espdet_run.py`
- `deploy/espdet_model_template/Kconfig`, `idf_component.yml`, `CMakeLists.txt`
- `deploy/espdet_example_template/{sdkconfig.defaults,sdkconfig.defaults.esp32p4,sdkconfig.defaults.esp32s3}`
- `deploy/espdet_example_template/{partitions.csv,partitions2.csv}`
- `deploy/espdet_example_template/main/{CMakeLists.txt,idf_component.yml}`
