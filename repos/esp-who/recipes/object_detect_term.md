# 目标检测（串口输出，无 LCD）

> **适用摘要**: 没有 LCD 或不想用图形库时，用 `WhoDetectAppTerm` 把每帧检测结果（box / score / keypoint）打印到串口。对应 `*_noglib` BSP。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-who/resources/`, source/examples in `repos/esp-who/`, and this recipe path `repos/esp-who/recipes/object_detect_term.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "object detect 不用 LCD"
- "检测结果打印到串口"
- "noglib 模式"
- "headless 检测"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/object_detect/`（`run_detect_term()` 分支） |
| BSP | `*_noglib` 变体（定义 `BSP_CONFIG_NO_GRAPHIC_LIB`） |

## 分步说明

### 1. 选 BSP 与模型

```bash
cd examples/object_detect
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib -DDETECT_MODEL=pedestrian_detect set-target esp32s3
```

### 2. 用 `WhoDetectAppTerm` 替换 `WhoDetectAppLCD`

```cpp
#include "frame_cap_pipeline.hpp"
#include "who_detect_app_term.hpp"
#include "bsp/esp-bsp.h"
#if defined(CONFIG_PEDESTRIAN_DETECT_MODEL_LOCATION)
#include "pedestrian_detect.hpp"
#endif

using namespace who::frame_cap;
using namespace who::app;

dl::detect::Detect *get_detect_model()
{
#if defined(CONFIG_PEDESTRIAN_DETECT_MODEL_LOCATION)
    return new PedestrianDetect(static_cast<PedestrianDetect::model_type_t>(CONFIG_DEFAULT_PEDESTRIAN_DETECT_MODEL), false);
#endif
    return nullptr;
}

void run_detect_term()
{
#if CONFIG_IDF_TARGET_ESP32S3
    auto frame_cap = get_term_dvp_frame_cap_pipeline();      // term 模式 fb_count 少 1
#elif CONFIG_IDF_TARGET_ESP32P4
    auto frame_cap = get_term_mipi_csi_frame_cap_pipeline();
    // 或 get_term_mipi_csi_ppa_frame_cap_pipeline();
#endif
    auto detect_app = new WhoDetectAppTerm(frame_cap);
    detect_app->set_model(get_detect_model());
    detect_app->run();
}

extern "C" void app_main(void)
{
    vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);
    run_detect_term();
}
```

### 3. 结果回调（`WhoDetectAppTerm::detect_result_cb`）

term 模式默认回调直接调 `who::detect::print_detect_results(result.det_res)`，把每条 `dl::detect::result_t`（box、score、可选 keypoint）打到 `ESP_LOGI`。如需自定义输出，可继承并 override：

```cpp
class MyDetectAppTerm : public who::app::WhoDetectAppTerm {
public:
    using WhoDetectAppTerm::WhoDetectAppTerm;
protected:
    void detect_result_cb(const detect::WhoDetect::result_t &result) override {
        for (const auto &r : result.det_res) {
            printf("[%ld] box=(%d,%d,%d,%d) score=%.3f\n",
                   (long)result.timestamp.tv_usec,
                   r.box[0], r.box[1], r.box[2], r.box[3], r.score);
        }
    }
};
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 LVGL 相关未定义 | 选了非 noglib BSP 但用了 term | term 用 `*_noglib`；LCD 用无后缀 BSP |
| term 模式帧率仍低 | `get_term_*` 已比 lcd 少 1 个 fb，仍慢则是模型本身 | 用更小输入的模型（224×224 而非 416×416） |
| 串口看不到结果 | `CONFIG_LOG_DEFAULT_LEVEL` 太低 | menuconfig 调到 INFO 或更高；确认 `idf.py monitor` 波特率匹配 |

## 参考

- `examples/object_detect/main/app_main.cpp`（`run_detect_term()`）
- `examples/object_detect/main/frame_cap_pipeline.cpp`（`get_term_*` 系列）
- `components/who_app/who_detect_app/who_detect_app_term.hpp`
- `resources/api_reference.md` §8.1 `WhoDetectAppTerm`
