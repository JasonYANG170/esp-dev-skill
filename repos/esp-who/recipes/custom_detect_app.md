# 定制检测应用（自定义回调与绘制）

> **适用摘要**: 通过继承 `WhoDetectAppLCD` / `WhoDetectAppTerm`，override `detect_result_cb` / `lcd_disp_cb` / `cleanup`，实现自定义的检测结果处理（上报、存盘、自定义画框）。

## 触发意图

- "自定义检测结果处理"
- "override detect_result_cb"
- "检测结果上报"
- "改变检测框颜色 / 画法"
- "把检测结果发出去"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `who_detect_app_lcd.hpp`（或 `_term`） |
| 理解 | `WhoDetect::result_t` 结构（见 `api_reference.md` §5） |

## 分步说明

### 1. `result_t` 结构（来自 `who_detect.hpp`）

```cpp
typedef struct {
    std::list<dl::detect::result_t> det_res;   // ESP-DL 检测结果列表
    struct timeval timestamp;
    dl::image::img_t img;                       // 本帧图像（可隐式转 cam_fb_t）
} result_t;
```

`dl::detect::result_t`（ESP-DL）含 `box[4]`（x1,y1,x2,y2）、`score`、可选 `keypoint`（人脸 5 点，长度 10）、`target`（类别索引）。可用方法 `limit_box(w,h)`、`limit_keypoint(w,h)`。

### 2. 继承 `WhoDetectAppLCD` override 回调

`WhoDetectAppLCD` 把三个回调声明为 `protected virtual`：`detect_result_cb`、`lcd_disp_cb`、`cleanup`。

```cpp
#include "who_detect_app_lcd.hpp"
#include "who_detect_result_handle.hpp"   // print_detect_results

class ReportDetectApp : public who::app::WhoDetectAppLCD {
public:
    using WhoDetectAppLCD::WhoDetectAppLCD;   // 继承构造
protected:
    void detect_result_cb(const who::detect::WhoDetect::result_t &result) override {
        // 1) 上报：每帧把 box 列表通过队列传给网络任务
        for (const auto &r : result.det_res) {
            // 例：仅上报 score>0.6
            if (r.score > 0.6f) {
                push_to_queue(r.box, r.score, result.timestamp);
            }
        }
        // 2) 同时保留默认行为：把结果交给 result_lcd_disp 画框
        WhoDetectAppLCD::detect_result_cb(result);
    }
    void lcd_disp_cb(who::cam::cam_fb_t *fb) override {
        // 默认：m_result_lcd_disp->lcd_disp_cb(fb)
        // 可在此叠加自定义绘制
        WhoDetectAppLCD::lcd_disp_cb(fb);
    }
private:
    void push_to_queue(const std::vector<int> &box, float score, struct timeval ts) {
        // 你的上报逻辑（BLE/HTTP/UART）...
    }
};
```

### 3. 用法

```cpp
extern "C" void app_main(void)
{
    vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);
#if CONFIG_IDF_TARGET_ESP32S3
    auto frame_cap = get_lcd_dvp_frame_cap_pipeline();
#elif CONFIG_IDF_TARGET_ESP32P4
    auto frame_cap = get_lcd_mipi_csi_frame_cap_pipeline();
#endif
    auto app = new ReportDetectApp({{255, 0, 0}}, frame_cap);
    app->set_model(get_detect_model());   // 不忘 set_model
    app->run();
}
```

### 4. 完全自己画（不走 result_lcd_disp）

若不要默认画框，override 时**不**调用基类：

```cpp
void detect_result_cb(const who::detect::WhoDetect::result_t &result) override {
    // 不调基类 → 不画默认框
    // 自己把结果存起来，在 lcd_disp_cb 里画
    m_last = result;   // 注意线程安全：WhoDetect 在核1，lcd_disp 在核0
}
```

> 注意并发：`detect_result_cb` 在检测任务（核1）执行，`lcd_disp_cb` 在 LCD 任务（核0）执行。共享数据需加锁或用 `WhoDetectResultLCDDisp` 内置的 `m_res_mutex` 队列机制（它内部维护 `std::queue<result_t>` 并加锁）。

### 5. 直接在图像上画框（无 LVGL 分支）

`who::detect::draw_detect_results_on_img`（`who_detect_result_handle.hpp`）把框画到 RGB565/RGB888 图像缓冲，无需 LVGL：

```cpp
#include "who_detect_result_handle.hpp"
void lcd_disp_cb(who::cam::cam_fb_t *fb) override {
    dl::image::img_t img = *fb;
    who::detect::draw_detect_results_on_img(img, m_last.det_res, {{0,255,0}});
}
```

带 LVGL 时用 `draw_detect_results_on_canvas(canvas, det_res, lv_palette)`。

### 6. term 模式自定义打印

`WhoDetectAppTerm::detect_result_cb` 默认 `print_detect_results`：

```cpp
class MyTermApp : public who::app::WhoDetectAppTerm {
public:
    using WhoDetectAppTerm::WhoDetectAppTerm;
protected:
    void detect_result_cb(const who::detect::WhoDetect::result_t &result) override {
        ESP_LOGI("MINE", "frame@%ld.%06ld n=%zu",
                 (long)result.timestamp.tv_sec, (long)result.timestamp.tv_usec,
                 result.det_res.size());
    }
};
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| override 不生效 | 方法签名/const 不匹配 | 严格照 `const who::detect::WhoDetect::result_t &result` 签名 |
| 自绘框闪烁/撕裂 | 在 lcd_disp_cb 改了 fb 但未与刷新同步 | 用 `WhoDetectResultLCDDisp` 内置锁，或只读不改 canvas |
| 并发崩溃（核0/核1 同时改数据） | 共享 `m_last` 无锁 | 用 mutex 或复用内置 `m_res_mutex` |
| 漏 `set_model` | 继承构造没带 set | 仍需 `app->set_model(...)` |
| `print_detect_results` 未声明 | 没 include | `#include "who_detect_result_handle.hpp"` |

## 参考

- `components/who_app/who_detect_app/who_detect_app_lcd.hpp`（virtual 回调声明）
- `components/who_app/who_detect_app/who_detect_app_lcd.cpp`（基类实现）
- `components/who_app/who_app_common/who_detect_result_handle/who_detect_result_handle.hpp`（draw/print 函数）
- `components/who_detect/who_detect.hpp`（`result_t`）
- `resources/api_reference.md` §5 `WhoDetect`、§8.1 `WhoDetectAppLCD`
- `resources/state_machine.md` §3 检测循环
