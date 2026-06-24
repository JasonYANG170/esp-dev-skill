# ESP-WHO 常见陷阱汇总

> 每条均来自仓库真实代码与示例行为。SKILL.md 中亦有摘要，此处给出完整说明与可复制代码。

---

## 1. 没设置 `IDF_EXTRA_ACTIONS_PATH` 直接 `set-target`

`bsp_ext.py` 是 idf.py 扩展。不设置该环境变量时，`BSP`/`DETECT_MODEL` 不会被注入 CMake cache，顶层 `CMakeLists.txt` 会 `FATAL_ERROR: BSP is not defined`。

```bash
# ❌ WRONG
idf.py set-target esp32s3

# ✅ CORRECT
export IDF_EXTRA_ACTIONS_PATH=/path_to_esp-who/tools/
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye set-target esp32s3
```

---

## 2. BSP 与 idf_target 不匹配

`bsp_ext.py` 的 `BSP2IDF_TARGET` 强校验。S3 的 BSP 不能配 `esp32p4`，反之亦然。

```bash
# ❌ WRONG
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_p4_function_ev_board set-target esp32s3

# ✅ CORRECT
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_p4_function_ev_board set-target esp32p4
```

---

## 3. `object_detect` 未指定 `-DDETECT_MODEL`

`object_detect` 的 `main/CMakeLists.txt` 用 `idf_component_optional_requires(PUBLIC ${DETECT_MODEL} espressif__${DETECT_MODEL})`，顶层 `CMakeLists.txt` 还会 `idf_build_set_property(DEPENDENCIES_LOCK dependencies.lock.${BSP}.${DETECT_MODEL})`。缺失会构建失败或 `app_main.cpp` 里 `get_detect_model()` 走到 `ESP_LOGE` 返回 `nullptr`。

```bash
# ❌ WRONG
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib set-target esp32s3

# ✅ CORRECT
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib -DDETECT_MODEL=human_face_detect set-target esp32s3
```

---

## 4. `WhoDetect::run()` 前未 `set_model()`

`WhoDetect::run()` 开头：`if (!m_model) { ESP_LOGE(...); return false; }`。任务根本起不来。

```cpp
// ❌ WRONG
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);
app->run();   // 内部 WhoDetect::run() 因 m_model==nullptr 返回 false

// ✅ CORRECT（照搬 object_detect/app_main.cpp）
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);
app->set_model(get_detect_model());   // 注入模型，延后创建避免内存碎片
app->run();
```

---

## 5. `WhoRecognitionCore::run()` 前未 `set_recognizer()`

同理返回 false。`WhoRecognitionAppLCD` 构造时已自动创建并 set，**只有**直接用 `WhoRecognition`/`WhoRecognitionCore` 时才需手动 set。

```cpp
// ❌ WRONG（直接用底层）
auto recog = new who::recognition::WhoRecognition(frame_cap->get_last_node());
recog->get_recognition_task()->run(3584, 2, 1);  // recognizer==nullptr，返回 false

// ✅ CORRECT
recog->set_detect_model(new HumanFaceDetect(...));
recog->set_recognizer(new HumanFaceRecognizer(db_path, ...));
recog->get_detect_task()->run(3584, 2, 1);
recog->get_recognition_task()->run(3584, 2, 1);
```

---

## 6. 在 App `run()` 之后才注册任务

`WhoTaskGroup::register_task` 内部断言 `assert(!WhoYield2Idle::get_instance()->is_active())`。`run()` 之后 `WhoYield2Idle` 已激活，此时注册会触发断言崩溃。

```cpp
// ❌ WRONG
app->run();
app->add_task(...);   // 断言失败

// ✅ CORRECT：所有构造（含节点 add_node、回调 set）在 run() 之前完成
auto frame_cap = get_lcd_dvp_frame_cap_pipeline();
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);
app->set_model(get_detect_model());
app->run();   // 之后不再改结构
```

---

## 7. `cam_fb_peek()` 取到的帧不可长期持有 / 不可跨节点归还

`WhoFetchNode` 的 ringbuf 满会自动 `cam_fb_return(旧帧)`。peek 返回的是 ringbuf 内指针，下一帧到达后该内存可能被回收或覆盖。

```cpp
// ❌ WRONG：把 fb 指针存起来延迟用
cam_fb_t *fb = node->cam_fb_peek();
vTaskDelay(pdMS_TO_TICKS(500));
use(fb->buf);   // 可能已被回收

// ✅ CORRECT：在回调内立即消费，或自行拷贝
void detect_result_cb(const who::detect::WhoDetect::result_t &res) {
    // res.img / res.det_res 已是本帧快照，安全使用
    who::detect::print_detect_results(res.det_res);
}
```

---

## 8. `fb_count` / `ringbuf_len` 过小导致丢帧或显示错位

`WhoFetchNode` 的 ringbuf_len = `cam->get_fb_count() - 2`。示例用 `MODEL_TIME + 3`（LCD）或 `MODEL_TIME + 2`（term）。设太小会让检测结果相对显示帧出现延迟。

```cpp
// ❌ WRONG
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, fs, 2);   // fb_count=2 → fetch ringbuf_len=0，断言失败

// ✅ CORRECT（照搬 object_detect frame_cap_pipeline.cpp）
#define MODEL_TIME 2   // 人脸检测
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, frame_size, MODEL_TIME + 3);
```

> 注释原话："If you have no idea how to set them, try with 5 and larger."

---

## 9. `WhoS3Cam` 构造参数顺序易错

`WhoS3Cam(pixel_format, frame_size, fb_count, vertical_flip=false, horizontal_flip=true)`。把 `fb_count` 漏掉会误用默认值；翻转方向默认水平翻转。

```cpp
// ❌ WRONG（误以为 (format, fb_count, flip...)）
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, 5, true);   // 5 当 framesize，true 当 fb_count

// ✅ CORRECT
framesize_t fs = get_cam_frame_size_from_lcd_resolution();
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, fs, MODEL_TIME + 3);
// Korvo-2 额外需要双翻转：
// new WhoS3Cam(PIXFORMAT_RGB565, fs, MODEL_TIME + 3, true, true);
```

---

## 10. P4 上想用 PPA 缩放却没传 `lcd_disp_frame_cap_node`

`WhoDetectAppLCD` 第三参默认 `nullptr`，此时取 `frame_cap->get_last_node()` 显示。若流水线尾端是 `WhoPPAResizeNode`（缩到模型输入），显示的是缩放后小图。需要把**缩放前**的 FetchNode 作为显示源，并通过 `set_rescale_params` 还原坐标。

```cpp
// ❌ WRONG：显示用缩放后节点，且不设 rescale，检测框画在小图上
auto frame_cap = get_lcd_mipi_csi_ppa_frame_cap_pipeline(&lcd_node);
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);   // 第三参漏了

// ✅ CORRECT
WhoFrameCapNode *lcd_node = nullptr;
auto frame_cap = get_lcd_mipi_csi_ppa_frame_cap_pipeline(&lcd_node);
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap, lcd_node);
// App 内部检测到 PPAResizeNode 后自动 set_rescale_params 还原坐标
```

---

## 11. 用 term 模式却选了带图形库的 BSP（或反之）

`*_noglib` 变体定义 `BSP_CONFIG_NO_GRAPHIC_LIB`，禁用 LVGL 类。`WhoDetectAppTerm` 不依赖 LCD，应配 `*_noglib`；`WhoDetectAppLCD` 需要图形库，必须用非 noglib BSP。

```bash
# ❌ WRONG：LCD 应用配 noglib
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib set-target esp32s3
# 然后 app_main 里 new WhoDetectAppLCD(...) → WhoLCD/WhoDetectResultLCDDisp 编译分支错乱

# ✅ CORRECT
# 串口模式：选 *_noglib + WhoDetectAppTerm
# LCD 模式：选无后缀 BSP + WhoDetectAppLCD
```

---

## 12. 人脸识别数据库路径未挂载文件系统就 run

`WhoRecognitionAppLCD` 构造时读 `CONFIG_DB_FATFS_FLASH` 等宏拼 `face.db` 路径，但**挂载动作在 `app_main`**。漏挂载会让 `HumanFaceRecognizer` 无法读写。

```cpp
// ❌ WRONG
auto app = new WhoRecognitionAppLCD(frame_cap);
app->run();   // storage 未挂载，face.db 不可用

// ✅ CORRECT（照搬 human_face_recognition/app_main.cpp）
#if CONFIG_DB_FATFS_FLASH
    ESP_ERROR_CHECK(fatfs_flash_mount());
#elif CONFIG_SPIFFS
    ESP_ERROR_CHECK(bsp_spiffs_mount());
#endif
#if CONFIG_DB_FATFS_SDCARD || CONFIG_HUMAN_FACE_DETECT_MODEL_IN_SDCARD || CONFIG_HUMAN_FACE_FEAT_MODEL_IN_SDCARD
    ESP_ERROR_CHECK(bsp_sdcard_mount());
#endif
auto app = new WhoRecognitionAppLCD(frame_cap);
app->run();
```

---

## 13. `xCoreID` 传 `tskNO_AFFINITY`

`WhoTask::run()` 开头 `assert(xCoreID != tskNO_AFFINITY)`。所有任务必须绑定到具体核（示例用 0 或 1）。

```cpp
// ❌ WRONG
detect->run(4096, 2, tskNO_AFFINITY);   // 断言失败

// ✅ CORRECT
detect->run(4096, 2, 1);   // 核1
```

---

## 14. 直接 `delete` 任务对象而不先 `stop()`

任务还在跑时析构会留下悬挂的 FreeRTOS 任务。`WhoApp::~WhoApp()` 已先 `stop()` 再 `destroy()`；自己管理底层对象时要照做。

```cpp
// ❌ WRONG
auto detect = new WhoDetect("d", node);
detect->set_model(m);
detect->run(4096, 2, 1);
delete detect;   // 任务仍在跑

// ✅ CORRECT
detect->stop();   // 或 detect->stop_async(); detect->wait_for_stopped(portMAX_DELAY);
delete detect;
```

---

## 15. 调整任务栈深后触发 Task WDT

`who_task/Kconfig` 提示：`CONFIG_MAX_TASK_LOOP_TIME` 与 `CONFIG_ESP_TASK_WDT_TIMEOUT_S` 相关。模型推理慢时（尤其 416×416）单次循环可能超过 WDT 阈值。

```
# ❌ WRONG：保持默认 WDT，跑 cat_detect pico_416_416
# （MODEL_TIME=8，单帧推理可能数秒）

# ✅ CORRECT：menuconfig 调大
CONFIG_ESP_TASK_WDT_TIMEOUT_S=10   # 大于最慢单帧推理耗时
# 或 menuconfig → Component config → ESP Task WDT
```

---

## 16. `app_main` 未提升自身优先级

三个示例 `app_main()` 第一行都是 `vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);`。`run()` 内部启动的子任务优先级多为 2；若 main 任务优先级更低，`run()` 返回后 main 立即退出（对象析构 → `stop()`）会导致应用跑不起来。

```cpp
// ❌ WRONG
extern "C" void app_main(void) {
    auto frame_cap = get_lcd_dvp_frame_cap_pipeline();
    auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);
    app->set_model(get_detect_model());
    app->run();
    // run() 返回，main 结束，app 被 delete
}

// ✅ CORRECT
extern "C" void app_main(void) {
    vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);   // 高于子任务的 2
    // ...
    app->run();   // run() 阻塞式管理（WhoYield2Idle 持续运行）
}
```
