# 目标检测（LCD 显示）

> **适用摘要**: 在带 LCD 的开发板上跑目标检测（人脸/行人/猫/狗），实时画框并显示。基于 `WhoDetectAppLCD`。

## 触发意图

- "目标检测 LCD"
- "object detect 显示画面"
- "人脸检测画框"
- "pedestrian / cat / dog detect 怎么跑"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/object_detect/` |
| BSP | 带 LCD 的 BSP（**不要** `_noglib` 后缀） |
| 模型 | 4 选 1：`human_face_detect` / `pedestrian_detect` / `cat_detect` / `dog_detect` |

## 分步说明

### 1. 选定 BSP + 模型 + target（不带 `_noglib`）

```bash
cd examples/object_detect
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib -DDETECT_MODEL=human_face_detect set-target esp32s3
# 注意：object_detect 的 BSP 文件名统一是 *_noglib；DETECT_MODEL 决定引入哪个 ESP-DL 模型组件
```

> 此处 `_noglib` 是示例自带的文件名约定（见 `examples/object_detect/sdkconfig.bsp.*_noglib`），它定义了 `BSP_CONFIG_NO_GRAPHIC_LIB` 用于 term 分支。要 LCD 画框，本应由 `WhoDetectAppLCD`（依赖 LVGL）；object_detect 示例默认 `run_detect_lcd()` 用的是 `WhoDetectAppLCD`，LVGL 通过 ESP-BSP 提供。具体见 `app_main.cpp` 注释的 `run_detect_term()` 备选。

### 2. `app_main` 标准写法（照搬 `examples/object_detect/main/app_main.cpp`）

```cpp
#include "frame_cap_pipeline.hpp"
#include "who_detect_app_lcd.hpp"
#include "who_detect_app_term.hpp"
#include "bsp/esp-bsp.h"
#if defined(CONFIG_HUMAN_FACE_DETECT_MODEL_LOCATION)
#include "human_face_detect.hpp"
#elif defined(CONFIG_PEDESTRIAN_DETECT_MODEL_LOCATION)
#include "pedestrian_detect.hpp"
#elif defined(CONFIG_CAT_DETECT_MODEL_LOCATION)
#include "cat_detect.hpp"
#elif defined(CONFIG_DOG_DETECT_MODEL_LOCATION)
#include "dog_detect.hpp"
#endif

using namespace who::frame_cap;
using namespace who::app;

// 模型工厂：根据 Kconfig 宏选模型
dl::detect::Detect *get_detect_model()
{
#if defined(CONFIG_HUMAN_FACE_DETECT_MODEL_LOCATION)
    return new HumanFaceDetect(static_cast<HumanFaceDetect::model_type_t>(CONFIG_DEFAULT_HUMAN_FACE_DETECT_MODEL), false);
#elif defined(CONFIG_PEDESTRIAN_DETECT_MODEL_LOCATION)
    return new PedestrianDetect(static_cast<PedestrianDetect::model_type_t>(CONFIG_DEFAULT_PEDESTRIAN_DETECT_MODEL), false);
#elif defined(CONFIG_CAT_DETECT_MODEL_LOCATION)
    return new CatDetect(static_cast<CatDetect::model_type_t>(CONFIG_DEFAULT_CAT_DETECT_MODEL), false);
#elif defined(CONFIG_DOG_DETECT_MODEL_LOCATION)
    return new DogDetect(static_cast<DogDetect::model_type_t>(CONFIG_DEFAULT_DOG_DETECT_MODEL), false);
#else
    ESP_LOGE("MAIN", "No detect model component in idf_component.yml");
    return nullptr;
#endif
}

void run_detect_lcd()
{
    WhoFrameCapNode *lcd_disp_frame_cap_node = nullptr;
#if CONFIG_IDF_TARGET_ESP32S3
    auto frame_cap = get_lcd_dvp_frame_cap_pipeline();
#elif CONFIG_IDF_TARGET_ESP32P4
    auto frame_cap = get_lcd_mipi_csi_frame_cap_pipeline();
    // P4 想用 PPA 硬件缩放（节省 CPU）：
    // auto frame_cap = get_lcd_mipi_csi_ppa_frame_cap_pipeline(&lcd_disp_frame_cap_node);
#endif
    // 调色板：每个类一组 {R,G,B}。单类用 {{255,0,0}}
    auto detect_app = new WhoDetectAppLCD({{255, 0, 0}}, frame_cap, lcd_disp_frame_cap_node);
    detect_app->set_model(get_detect_model());   // 延后创建，避免内存碎片
    detect_app->run();
}

extern "C" void app_main(void)
{
    vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);   // 必须高于子任务优先级 2
#if CONFIG_HUMAN_FACE_DETECT_MODEL_IN_SDCARD || CONFIG_PEDESTRIAN_DETECT_MODEL_IN_SDCARD || \
    CONFIG_CAT_DETECT_MODEL_IN_SDCARD || CONFIG_DOG_DETECT_MODEL_IN_SDCARD
    ESP_ERROR_CHECK(bsp_sdcard_mount());
#endif
#ifdef BSP_BOARD_ESP32_S3_EYE
    ESP_ERROR_CHECK(bsp_leds_init());
    ESP_ERROR_CHECK(bsp_led_set(BSP_LED_GREEN, false));   // 关 LED
#endif
    run_detect_lcd();
}
```

### 3. 调色板与多类

`WhoDetectAppLCD` 第一个参数是 `std::vector<std::vector<uint8_t>>`，按类别索引取颜色（RGB888）。例如行人+人脸两类：

```cpp
auto app = new WhoDetectAppLCD({{255,0,0}, {0,255,0}}, frame_cap);
```

### 4. P4 上启用 PPA 硬件缩放（可选，降 CPU）

P4 流水线尾端可插 `WhoPPAResizeNode`，把帧缩到模型输入（如 224×224）。此时需把**缩放前**的 FetchNode 作为 LCD 显示源，App 内部会自动 `set_rescale_params` 还原检测框坐标。

```cpp
WhoFrameCapNode *lcd_node = nullptr;
auto frame_cap = get_lcd_mipi_csi_ppa_frame_cap_pipeline(&lcd_node);
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap, lcd_node);
app->set_model(get_detect_model());
app->run();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `detect model is nullptr, please call set_model() first.` | 漏 `set_model()` 或模型组件未引入 | 在 `run()` 前 `set_model(get_detect_model())`；确认 `-DDETECT_MODEL=` 已设 |
| 检测框画在小图上 / 位置偏 | P4 用了 PPAResize 却没传 `lcd_disp_frame_cap_node` | 把 FetchNode 传出并作为 `WhoDetectAppLCD` 第三参 |
| fps 很低 / 卡顿 | `fb_count`/`ringbuf_len` 太小或模型太大 | 用示例的 `MODEL_TIME+3`；416×416 模型 (`pico_416_416`) 本身慢 |
| `assert(xCoreID != tskNO_AFFINITY)` | 自定义任务时传了 `tskNO_AFFINITY` | 显式传 `0` 或 `1` |
| 运行后立即退出 | `app_main` 未提升优先级 | 第一行 `vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);` |

## 参考

- `examples/object_detect/main/app_main.cpp`
- `examples/object_detect/main/frame_cap_pipeline.cpp`
- `components/who_app/who_detect_app/who_detect_app_lcd.hpp`
- `resources/api_reference.md` §5 `WhoDetect`、§8.1 `WhoDetectAppLCD`
