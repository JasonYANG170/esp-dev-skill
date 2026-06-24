# 在 ESP32-P4/S3 上部署与运行推理

> **适用摘要**: 在 `espdet_run.py` 生成的 ESP-IDF 工程中，配置模型加载位置（flash/partition/sdcard）、编译、烧录，并在串口观察检测结果。

## 触发意图

- "部署到芯片"
- "idf.py 烧录"
- "ESPDetDetect 调用"
- "模型放 flash / sdcard"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工程 | `esp-dl/examples/<class>_detect/`（由 `espdet_run.py` 生成） |
| 模型 | `<esp-dl>/models/<class>_detect/models/{p4,s3}/*.espdl` |
| 工具链 | ESP-IDF release/v5.3+（模板基于 5.4.0） |
| 硬件 | ESP32-P4（如 `esp32_p4_function_ev_board`）或 ESP32-S3（如 `esp32_s3_eye`） |

## 分步说明

### 1. 核心运行代码（来自 `deploy/espdet_example_template/main/app_main.cpp`）

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

    ESPDetDetect *detect = new ESPDetDetect();          // 默认模型 + lazy_load
    auto &detect_results = detect->run(img);
    for (const auto &res : detect_results) {
        ESP_LOGI(TAG,
                 "[category: %d, score: %f, x1: %d, y1: %d, x2: %d, y2: %d]",
                 res.category, res.score,
                 res.box[0], res.box[1], res.box[2], res.box[3]);
    }
    delete detect;
    heap_caps_free(img.data);

#if CONFIG_ESPDET_DETECT_MODEL_IN_SDCARD
    ESP_ERROR_CHECK(bsp_sdcard_unmount());
#endif
}
```

### 2. 模型封装（来自 `deploy/espdet_model_template/espdet_detect.hpp`）

```cpp
class ESPDetDetect : public dl::detect::DetectWrapper {
public:
    typedef enum {
        ESPDET_PICO_imgH_imgW_CUSTOM,        // rename_project 替换 imgH/imgW/CUSTOM
    } model_type_t;
    ESPDetDetect(model_type_t model_type = static_cast<model_type_t>(CONFIG_DEFAULT_ESPDET_DETECT_MODEL),
                 bool lazy_load = true);
private:
    void load_model() override;
    model_type_t m_model_type;
};
```

默认 `score_thr=0.25`、`nms_thr=0.7`（来自 `espdet_detect::ESPDet::default_score_thr / default_nms_thr`）。

### 3. 模型构造与预处理（来自 `espdet_detect.cpp`）

```cpp
ESPDet::ESPDet(const char *model_name, float score_thr, float nms_thr)
{
#if !CONFIG_ESPDET_DETECT_MODEL_IN_SDCARD
    m_model = new dl::Model(path, model_name,
                            static_cast<fbs::model_location_type_t>(CONFIG_ESPDET_DETECT_MODEL_LOCATION));
#else
    auto sd_path = std::filesystem::path(CONFIG_BSP_SD_MOUNT_POINT)
                 / CONFIG_ESPDET_DETECT_MODEL_SDCARD_DIR / model_name;
    m_model = new dl::Model(sd_path.c_str(), fbs::MODEL_LOCATION_IN_SDCARD);
#endif
    m_model->minimize();

#if CONFIG_IDF_TARGET_ESP32P4
    m_image_preprocessor = new dl::image::ImagePreprocessor(m_model, {0, 0, 0}, {255, 255, 255});
#else
    m_image_preprocessor = new dl::image::ImagePreprocessor(
        m_model, {0, 0, 0}, {255, 255, 255}, dl::image::DL_IMAGE_CAP_RGB565_BIG_ENDIAN);
#endif
    m_image_preprocessor->enable_letterbox({114, 114, 114});   // 必须调用

    m_postprocessor = new dl::detect::ESPDetPostProcessor(
        m_model, m_image_preprocessor, score_thr, nms_thr, 10,
        {{8, 8, 4, 4}, {16, 16, 8, 8}, {32, 32, 16, 16}});    // anchors 不可改
}
```

### 4. 选择模型加载位置（menuconfig）

`models/<class>_detect/Kconfig` 提供三选一（默认 `flash_rodata`）：

| 选项 | 对应宏 | 说明 |
|---|---|---|
| flash_rodata | `CONFIG_ESPDET_DETECT_MODEL_IN_FLASH_RODATA` | CMake 调 `target_add_aligned_binary_data` 嵌入固件，`CONFIG_ESPDET_DETECT_MODEL_LOCATION=0` |
| flash_partition | `CONFIG_ESPDET_DETECT_MODEL_IN_FLASH_PARTITION` | 烧到 SPIFFS 分区 `espdet_det`（需 `partitions2.csv`），`LOCATION=1` |
| sdcard | `CONFIG_ESPDET_DETECT_MODEL_IN_SDCARD` | 从 SD 卡 `models/p4` 或 `models/s3` 读，`LOCATION=2`，需 `bsp_sdcard_mount()` |

### 5. 分区表选择

- 默认 `partitions.csv`：`nvs / phy_init / factory(8000K)`，适合 rodata 模式的小模型。
- `partitions2.csv`：`factory(2000K) + custom_det(data,spiffs,4M)`，适合 partition 模式的大模型。
- 在 `sdkconfig.defaults` 设 `CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=partitions2.csv` 切换。

### 6. 芯片相关 sdkconfig（来自模板）

ESP32-P4（`sdkconfig.defaults.esp32p4`）关键项：
```
CONFIG_IDF_TARGET="esp32p4"
CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y
CONFIG_SPIRAM=y
CONFIG_SPIRAM_SPEED_200M=y
CONFIG_SPIRAM_XIP_FROM_PSRAM=y
CONFIG_CACHE_L2_CACHE_256KB=y
CONFIG_CACHE_L2_CACHE_LINE_128B=y
CONFIG_COMPILER_OPTIMIZATION_PERF=y
CONFIG_BSP_SD_FORMAT_ON_MOUNT_FAIL=y
```

ESP32-S3（`sdkconfig.defaults.esp32s3`）关键项：
```
CONFIG_IDF_TARGET="esp32s3"
CONFIG_ESPTOOLPY_FLASHSIZE_8MB=y
CONFIG_SPIRAM=y
CONFIG_SPIRAM_MODE_OCT=y
CONFIG_SPIRAM_SPEED_80M=y
CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y
CONFIG_ESP32S3_INSTRUCTION_CACHE_32KB=y
CONFIG_ESP32S3_DATA_CACHE_64KB=y
CONFIG_ESP32S3_DATA_CACHE_LINE_64B=y
CONFIG_ESP_TASK_WDT_TIMEOUT_S=40
```

### 7. 编译烧录

```bash
cd esp-dl/examples/<class>_detect
idf.py set-target esp32p4      # 或 esp32s3
idf.py menuconfig              # Component config → models: <class>_detect → 选 model location
idf.py build
idf.py flash monitor
```

串口预期输出（检测到目标时）：
```
I (xxxx) custom_detect: [category: 0, score: 0.8723, x1: 12, y1: 34, x2: 200, y2: 210]
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 无检测结果（全空） | letterbox / S3 RGB565 / anchors 被改 | 确认 `enable_letterbox({114,114,114})`；S3 加 `DL_IMAGE_CAP_RGB565_BIG_ENDIAN`；anchors 不改 |
| 找不到 esp-dl 组件 | `idf_component.yml` 的 override_path 失效 | 确认 `<esp-dl>` 已 clone 且相对路径正确（`../../esp-dl`、`../../../models/<class>_detect`） |
| partition 模式烧录失败 | 未用 `partitions2.csv` | 设 `CONFIG_PARTITION_TABLE_CUSTOM_FILENAME=partitions2.csv` |
| sdcard 模式启动失败 | 未挂载 SD | `app_main` 先 `bsp_sdcard_mount()`；模型放 `models/p4`/`models/s3` |
| S3 OOM / 速度慢 | PSRAM 未启用 | 确认 `CONFIG_SPIRAM=y`、`CONFIG_SPIRAM_MODE_OCT=y`、`CONFIG_SPIRAM_SPEED_80M=y` |
| `espdet_pico_imgH_imgW_custom is not selected` | menuconfig 未勾选对应 flash 宏 | menuconfig 勾选 `flash espdet_pico_<H>_<W>_<CLASS>` |
| BSP 组件缺失 | S3/P4 BSP 版本不符 | 按模板 `idf_component.yml`：S3 用 `esp32_s3_eye_noglib ^3.1.0~1`，P4 用 `esp32_p4_function_ev_board_noglib ^4.0.1` |

## 参考

- `deploy/espdet_example_template/main/app_main.cpp`
- `deploy/espdet_model_template/espdet_detect.hpp` / `espdet_detect.cpp`
- `deploy/espdet_model_template/Kconfig`（模型位置选项）
- `deploy/espdet_example_template/partitions.csv` / `partitions2.csv`
- `deploy/espdet_example_template/sdkconfig.defaults{,.esp32p4,.esp32s3}`
- `deploy/espdet_example_template/main/idf_component.yml`（BSP 依赖）
- `recipes/all_in_one_pipeline.md`（生成此工程）
