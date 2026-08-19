# 二维码识别

> **适用摘要**: 使用 `WhoQRCodeAppLCD`（或 `WhoQRCodeAppTerm`）实时识别摄像头画面中的二维码，把解码文本显示到 LCD / 打印到串口。底层是 `quirc` 库。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-who/resources/`, source/examples in `repos/esp-who/`, and this recipe path `repos/esp-who/recipes/qrcode_recognition.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "二维码识别"
- "qrcode decode"
- "扫二维码"
- "quirc 怎么用"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/qrcode_recognition/` |
| 组件 | `espressif/quirc`（由 `who_qrcode/idf_component.yml` 拉取） |
| 支持 BSP | `esp32_s3_eye`、`esp32_p4_function_ev_board` |

## 分步说明

### 1. 选 BSP + target

```bash
cd examples/qrcode_recognition
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye set-target esp32s3
# 或 esp32_p4_function_ev_board / esp32p4
```

> qrcode_recognition 没有 `_noglib` 变体（LCD 模式需图形库）。

### 2. `app_main`（照搬 `examples/qrcode_recognition/main/app_main.cpp`）

```cpp
#include "frame_cap_pipeline.hpp"
#include "who_qrcode_app_lcd.hpp"
#include "who_qrcode_app_term.hpp"
#include "bsp/esp-bsp.h"

using namespace who::frame_cap;
using namespace who::app;

extern "C" void app_main(void)
{
    vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);

#ifdef BSP_BOARD_ESP32_S3_EYE
    ESP_ERROR_CHECK(bsp_leds_init());
    ESP_ERROR_CHECK(bsp_led_set(BSP_LED_GREEN, false));
#endif

#if CONFIG_IDF_TARGET_ESP32S3
    auto frame_cap = get_dvp_frame_cap_pipeline();
#elif CONFIG_IDF_TARGET_ESP32P4
    auto frame_cap = get_mipi_csi_frame_cap_pipeline();
    // USB UVC 摄像头：
    // auto frame_cap = get_uvc_frame_cap_pipeline();
#endif
    auto qrcode_app = new WhoQRCodeAppLCD(frame_cap);
    // 无 LCD：
    // auto qrcode_app = new WhoQRCodeAppTerm(frame_cap);
    qrcode_app->run();
}
```

### 3. 帧采集流水线（`examples/qrcode_recognition/main/frame_cap_pipeline.cpp`）

`MODEL_TIME = 2`，最简的单节点 Fetch 流水线：

```cpp
#define MODEL_TIME 2

#if CONFIG_IDF_TARGET_ESP32S3
WhoFrameCap *get_dvp_frame_cap_pipeline() {
    auto cam = new WhoS3Cam(PIXFORMAT_RGB565, FRAMESIZE_240X240, MODEL_TIME + 2);
    auto frame_cap = new WhoFrameCap();
    frame_cap->add_node<WhoFetchNode>("FrameCapFetch", cam);
    return frame_cap;
}
#elif CONFIG_IDF_TARGET_ESP32P4
WhoFrameCap *get_mipi_csi_frame_cap_pipeline() {
    auto cam = new WhoP4Cam(V4L2_PIX_FMT_RGB565, MODEL_TIME + 2);
    auto frame_cap = new WhoFrameCap();
    frame_cap->add_node<WhoFetchNode>("FrameCapFetch", cam);
    return frame_cap;
}
#endif
```

### 4. 内部解码流程（来自 `who_qrcode.cpp`）

`WhoQRCode` 订阅 `NEW_FRAME`，收到帧后：
1. `quirc_begin()` 拿灰度缓冲（S3 尺寸 = `BSP_LCD_H_RES × BSP_LCD_V_RES`；P4 = `/2`）
2. 用 `dl::image::ImageTransformer` 把 RGB565 帧转灰度填入
3. `quirc_end()` → `quirc_count()` → 对每个码 `quirc_extract` + `quirc_decode`
4. 若 `QUIRC_ERROR_DATA_ECC`：`quirc_flip` 后再 decode 一次
5. 命中即 `result_cb(payload)` 并 `break`（每帧只处理一个码）

### 5. 自定义结果处理

`WhoQRCodeAppTerm` 默认 `ESP_LOGI("QRCode", "%s", result.c_str())`。继承 override `qrcode_result_cb` 可改：

```cpp
class MyApp : public who::app::WhoQRCodeAppTerm {
public:
    using WhoQRCodeAppTerm::WhoQRCodeAppTerm;
protected:
    void qrcode_result_cb(const std::string &result) override {
        ESP_LOGI("QR", "decoded: %s (len=%zu)", result.c_str(), result.size());
        // 例如：通过 BLE / HTTP 上报
    }
};
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 识别率低 / 漏码 | 帧太小或模糊 | 用 `FRAMESIZE_240X240` 或更大；保证光照；二维码占画面比例足够 |
| P4 上 quirc 尺寸不对报错 | 误以为是 LCD 全尺寸 | P4 内部固定 `/2`，由 `who_qrcode.cpp` 处理，不要手动改 |
| USB UVC 模式无图像 | UVC 流水线需多节点（Fetch+Decode+PPAResize） | 用 `get_uvc_frame_cap_pipeline()`，含 Decode 与 PPAResize 到 800×600 |
| `idf_component.yml` 缺 quirc | 组件未拉取 | 确认 `who_qrcode/idf_component.yml` 有 `espressif/quirc` |

## 参考

- `examples/qrcode_recognition/main/app_main.cpp`
- `examples/qrcode_recognition/main/frame_cap_pipeline.cpp`
- `components/who_qrcode/who_qrcode.cpp`（quirc 调用链）
- `components/who_app/who_qrcode_app/who_qrcode_app_lcd.hpp`
- `resources/api_reference.md` §7 `WhoQRCode`
- `resources/state_machine.md` §5 QRCode 循环
