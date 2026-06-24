# ESP-WHO 配置（Kconfig / sdkconfig / 环境变量）参考

> 本文所有宏与选项均来自仓库内真实的 `Kconfig`、`sdkconfig.bsp.*`、`CMakeLists.txt` 与 `tools/bsp_ext.py`。

---

## 1. ESP-WHO 自有 Kconfig 符号

### 1.1 `components/who_task/Kconfig` — 菜单 `esp-who: yield2idle`

| 符号 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `CONFIG_MAX_TASK_LOOP_TIME` | int | `1` | 单个任务最大循环耗时（秒），向上取整；若接近 `CONFIG_ESP_TASK_WDT_TIMEOUT_S` 需调大后者 |

### 1.2 `components/who_peripherals/who_spiflash_fatfs/Kconfig` — 菜单 `esp-who: human_face_recognition`

| 符号 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `CONFIG_SPIFLASH_MOUNT_POINT` | string | `/spiflash` | SPI Flash FAT 挂载点 |
| `CONFIG_SPIFLASH_MOUNT_PARTITION` | string | `storage` | FAT 所在分区名 |

### 1.3 `components/who_app/who_recognition_app/Kconfig` — 菜单 `esp-who: human_face_recognition`

数据库文件系统 `choice DB_FILE_SYSTEM`：

| 符号 | 含义 |
|---|---|
| `CONFIG_DB_FATFS_FLASH` | fatfs on flash（默认） |
| `CONFIG_DB_FATFS_SDCARD` | fatfs on sdcard |
| `CONFIG_SPIFFS` | spiffs（仅 flash） |

对应代码里 `face.db` 路径来源：
- `CONFIG_DB_FATFS_FLASH` → `CONFIG_SPIFLASH_MOUNT_POINT/face.db`
- `CONFIG_SPIFFS` → `CONFIG_BSP_SPIFFS_MOUNT_POINT/face.db`
- `CONFIG_DB_FATFS_SDCARD` → `CONFIG_BSP_SD_MOUNT_POINT/face.db`

---

## 2. idf.py 构建期变量（由 `tools/bsp_ext.py` 解析）

> 设置方式：`export IDF_EXTRA_ACTIONS_PATH=/path_to_esp-who/tools/`

| 变量 | 必需 | 取值 | 作用 |
|---|---|---|---|
| `SDKCONFIG_DEFAULTS` | 是 | `sdkconfig.bsp.<bsp_name>` | 选定开发板默认配置 |
| `IDF_TARGET` | 是 | `esp32s3` / `esp32p4` | 目标芯片，须与 BSP 匹配 |
| `BSP` | 自动写入 | bsp 名 | CMake 层用，`dependencies.lock.${BSP}` 选依赖锁 |
| `DETECT_MODEL` | 仅 `object_detect` | `human_face_detect` / `pedestrian_detect` / `cat_detect` / `dog_detect` | 选检测模型组件 |

支持的 BSP（来自 `bsp_ext.py`）：

| BSP 名 | IDF_TARGET | 说明 |
|---|---|---|
| `esp32_s3_eye` | esp32s3 | ESP32-S3-EYE |
| `esp32_s3_korvo_2` | esp32s3 | ESP32-S3-Korvo-2 |
| `esp32_p4_function_ev_board` | esp32p4 | ESP32-P4 Function EV Board |
| 上述任一 + `_noglib` 后缀 | 同上 | 不链接图形库（无 LVGL），仅串口/term 模式 |

`BSP2IDF_TARGET` 校验：BSP 与 idf_target 不匹配时 `bsp_ext.py` 直接 `sys.exit(2)`。

---

## 3. 检测模型选择宏（由 ESP-DL 模型组件 Kconfig 注入）

`object_detect` 示例根据 `DETECT_MODEL` 编译期二选一，对应宏：

| DETECT_MODEL | 定义宏 | CONFIG_DEFAULT_* | MODEL_TIME | MODEL_INPUT_W×H |
|---|---|---|---|---|
| `human_face_detect` | `CONFIG_HUMAN_FACE_DETECT_MODEL_LOCATION` | `CONFIG_DEFAULT_HUMAN_FACE_DETECT_MODEL` | 2 | 160×120 |
| `pedestrian_detect` | `CONFIG_PEDESTRIAN_DETECT_MODEL_LOCATION` | `CONFIG_DEFAULT_PEDESTRIAN_DETECT_MODEL` | 3 | 224×224 |
| `cat_detect` (`pico_224_224`) | `CONFIG_CAT_DETECT_MODEL_LOCATION` + `CONFIG_ESPDET_PICO_224_224_CAT` | `CONFIG_DEFAULT_CAT_DETECT_MODEL` | 3 | 224×224 |
| `cat_detect` (`pico_416_416`) | `CONFIG_ESPDET_PICO_416_416_CAT` | 同上 | 8 | 416×416 |
| `dog_detect` (`pico_224_224`) | `CONFIG_DOG_DETECT_MODEL_LOCATION` + `CONFIG_ESPDET_PICO_224_224_DOG` | `CONFIG_DEFAULT_DOG_DETECT_MODEL` | 3 | 224×224 |
| `dog_detect` (`pico_416_416`) | `CONFIG_ESPDET_PICO_416_416_DOG` | 同上 | 8 | 416×416 |

`MODEL_TIME` 决定 `fb_count`（= `MODEL_TIME + 3` 用于 LCD 流水线，`MODEL_TIME + 2` 用于 term）。

人脸识别示例另有：
- `CONFIG_DEFAULT_HUMAN_FACE_FEAT_MODEL`（`HumanFaceFeat::model_type_t`）
- `CONFIG_DEFAULT_HUMAN_FACE_DETECT_MODEL`（`HumanFaceDetect::model_type_t`）

---

## 4. 硬件能力宏（影响代码路径）

| 宏 | 含义 | 出现位置 |
|---|---|---|
| `CONFIG_SOC_PPA_SUPPORTED` | SoC 支持 PPA（仅 ESP32-P4），启用 `WhoPPAResizeNode` | `who_frame_cap_node.hpp` |
| `CONFIG_SOC_JPEG_CODEC_SUPPORTED` | SoC 支持硬件 JPEG 解码，`WhoDecodeNode` 走 `hw_decode_jpeg` | `who_frame_cap_node.cpp` |
| `BSP_CONFIG_NO_GRAPHIC_LIB` | BSP 不含图形库，走裸 `esp_lcd` 分支、禁用 LVGL 类 | `who_lcd.hpp`、`who_detect_result_handle.hpp` |
| `BSP_CAPS_BUTTONS` | BSP 支持物理按键 | `who_recognition_button.hpp` |
| `BSP_BOARD_ESP32_S3_EYE` / `BSP_BOARD_ESP32_S3_KORVO_2` / `BSP_BOARD_ESP32_P4_FUNCTION_EV_BOARD` | 具体板子宏，决定按键映射/LED 初始化 | 示例 `app_main.cpp` |

---

## 5. 关键 sdkconfig 项（来自各 `sdkconfig.bsp.*`）

### 5.1 通用项（所有 BSP）

| 配置 | 值 | 说明 |
|---|---|---|
| `CONFIG_PARTITION_TABLE_CUSTOM` | y | 使用自定义分区表 `partitions.csv` |
| `CONFIG_COMPILER_OPTIMIZATION_PERF` | y | 性能优化 |
| `CONFIG_SPIRAM` | y | 必须，模型与帧缓冲需要 PSRAM |
| `CONFIG_SPIRAM_XIP_FROM_PSRAM` | y | PSRAM 上执行 |
| `CONFIG_ESP_SYSTEM_ALLOW_RTC_FAST_MEM_AS_HEAP` | n | 关闭，避免 cache 一致性问题 |
| `CONFIG_FATFS_LFN_HEAP` | y | 长文件名 |
| `CONFIG_FREERTOS_HZ` | 1000 | 任务调度频率 |
| `CONFIG_FREERTOS_VTASKLIST_INCLUDE_COREID` | y | 状态打印需要 |
| `CONFIG_FREERTOS_GENERATE_RUN_TIME_STATS` | y | 状态打印需要 |
| `CONFIG_FREERTOS_RUN_TIME_COUNTER_TYPE_U64` | y | 64 位运行时计数 |

### 5.2 ESP32-S3 专项

| 配置 | 值 |
|---|---|
| `CONFIG_ESPTOOLPY_FLASHMODE_QIO` | y |
| `CONFIG_ESPTOOLPY_FLASHSIZE_8MB` | y |
| `CONFIG_SPIRAM_MODE_OCT` | y（八线 PSRAM） |
| `CONFIG_SPIRAM_SPEED_80M` | y |
| `CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240` | y |
| `CONFIG_ESP32S3_INSTRUCTION_CACHE_32KB` | y |
| `CONFIG_ESP32S3_DATA_CACHE_64KB` | y |
| `CONFIG_ESP32S3_DATA_CACHE_LINE_64B` | y |
| `CONFIG_CAMERA_PSRAM_DMA` | y |

### 5.3 ESP32-P4 专项

| 配置 | 值 |
|---|---|
| `CONFIG_ESPTOOLPY_FLASHSIZE_16MB` | y |
| `CONFIG_SPIRAM_SPEED_200M` | y |
| `CONFIG_CACHE_L2_CACHE_256KB` | y |
| `CONFIG_CACHE_L2_CACHE_LINE_128B` | y |
| `CONFIG_USB_HOST_CONTROL_TRANSFER_MAX_SIZE` | 4096 |
| `CONFIG_USB_HOST_HW_BUFFER_BIAS_IN` | y |
| `CONFIG_USB_HOST_HUBS_SUPPORTED` | y |
| `CONFIG_BSP_LCD_DPI_BUFFER_NUMS` | 2 |
| `CONFIG_BSP_DISPLAY_LVGL_AVOID_TEAR` | y |
| `CONFIG_CAMERA_SC2336` | y |
| `CONFIG_CAMERA_SC2336_MIPI_RAW8_1024X600_30FPS` | y |
| `CONFIG_CAMERA_SC2336_CUSTOMIZED_IPA_JSON_CONFIGURATION_FILE` | y |
| `CONFIG_CAMERA_SC2336_CUSTOMIZED_IPA_JSON_CONFIGURATION_FILE_PATH` | `../../components/who_peripherals/who_cam/who_p4_cam/sc2336.json` |
| `CONFIG_ESP_VIDEO_ENABLE_ISP_PIPELINE_CONTROLLER` | y |
| `CONFIG_LV_DEF_REFR_PERIOD` | 50 |
| `CONFIG_IDF_EXPERIMENTAL_FEATURES` | y |

---

## 6. 分区表（`partitions.csv`）

### 6.1 `human_face_recognition`（模型在 flash，`partitions.csv`）

```
nvs,      data, nvs,     0x9000,  24K,
phy_init, data, phy,     0xf000,  4K,
factory,  app,  factory, 0x010000,7000K,
storage,  data, fat,     ,        1M,
```

### 6.2 `human_face_recognition`（模型在 SD 卡/分区，`partitions2.csv`）

```
nvs,              data, nvs,     0x9000,  24K,
phy_init,         data, phy,     0xf000,  4K,
factory,          app,  factory, 0x010000,1900K,
human_face_det,   data, spiffs,  ,        200K,
human_face_feat,  data, spiffs,  ,        5000K,
storage,          data, fat,     ,        1M,
```

> `object_detect` / `qrcode_recognition` 示例也各自提供 `partitions.csv`，结构与 6.1 类似。
