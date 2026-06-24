# ESP-WHO API 速查参考

> 本文档中的所有类、方法、结构体均来自 ESP-WHO 仓库 `components/` 目录下的真实头文件。命名空间统一为 `who::*`。底层深度学习模型类（`dl::detect::Detect`、`HumanFaceDetect`、`HumanFaceRecognizer` 等）来自 ESP-DL / `human_face_recognition` 组件，ESP-WHO 仅作调用。

---

## 1. 任务基类 `who::task`

> 头文件：`components/who_task/who_task.hpp`

ESP-WHO 中所有可运行单元（采集节点、检测、识别、LCD 显示）都派生自 `WhoTask` / `WhoTaskBase`。运行模型采用 FreeRTOS `xTaskCreatePinnedToCore`，并通过事件组（EventGroup）传递控制位。

### 1.1 `WhoTaskBase` 控制位（static constexpr EventBits_t）

| 常量 | 含义 |
|---|---|
| `TASK_STOPPED` | 任���已停止 |
| `TASK_PAUSED` | 任务已暂停 |
| `TASK_STOP` | 请求停止 |
| `TASK_PAUSE` | 请求暂停 |
| `TASK_RESUME` | 请求恢复 |
| `TASK_EVENT_BIT_LAST` | 子类自定义事件位的起始位（`1 << 5`） |

### 1.2 `WhoTaskBase` 关键方法

```cpp
virtual bool run(const configSTACK_DEPTH_TYPE uxStackDepth,
                 UBaseType_t uxPriority,
                 const BaseType_t xCoreID);       // 创建并启动 FreeRTOS 任务
virtual bool stop();                               // 同步停止（等待 TASK_STOPPED）
virtual bool stop_async();                         // 异步停止（置位后立即返回）
virtual void wait_for_stopped(TickType_t timeout);
virtual bool pause();                              // 同步暂停
virtual bool pause_async();
virtual bool resume();
bool is_active();                                  // 是否正在运行（非停止/暂停）
std::string get_name();
EventGroupHandle_t get_event_group();
TaskHandle_t get_task_handle();
```

### 1.3 `WhoTask`

```cpp
class WhoTask : public WhoTaskBase {
public:
    WhoTask(const std::string &name);
    bool run(const configSTACK_DEPTH_TYPE uxStackDepth,
             UBaseType_t uxPriority,
             const BaseType_t xCoreID) override;  // 注意：xCoreID 不能为 tskNO_AFFINITY
    bool resume() override;
    BaseType_t get_coreid();
    SemaphoreHandle_t get_mutex();
};
```

### 1.4 `WhoTaskGroup`

```cpp
class WhoTaskGroup {
public:
    void register_task(WhoTask *task);
    void unregister_task(WhoTask *task);
    void register_task_group(WhoTaskGroup *task_group);
    void unregister_task_group(WhoTaskGroup *task_group);
    std::vector<WhoTask *> get_all_tasks();
    void stop();      // 异步停止所有任务后等待并清理
    void resume();
    void pause();
    void destroy();   // delete 所有已注册 task / task_group
};
```

> **重要约束**：`register_task` / `register_task_group` 内部有断言 `assert(!WhoYield2Idle::get_instance()->is_active())`。即必须在 App 调用 `run()` 之前完成所有任务注册。

---

## 2. `WhoYield2Idle`（空闲让行监控）

> 头文件：`components/who_task/who_yield2idle.hpp`

单例。每个 `WhoTask` 构造时会自动 `start_monitor(this)`，析构时 `end_monitor`。其作用是在 CPU 空闲回调里让出 CPU，使所有任务得到均衡调度，避免单一任务长时间占用核。

```cpp
class WhoYield2Idle : public task::WhoTaskBase {
public:
    static WhoYield2Idle *get_instance();
    bool run(const configSTACK_DEPTH_TYPE uxStackDepth = 2048);
    void start_monitor(task::WhoTask *task);
    void end_monitor(task::WhoTask *task);
    bool stop_async() override;
    bool pause_async() override;
};
```

App 的 `run()` 通常会先调用 `WhoYield2Idle::get_instance()->run()`。

---

## 3. 帧采集流水线 `who::frame_cap`

> 头文件：`components/who_frame_cap/who_frame_cap.hpp`、`who_frame_cap_node.hpp`

ESP-WHO 采用“节点链”流水线。`WhoFrameCap` 管理一串 `WhoFrameCapNode`，节点之间通过长度为 1 的 FreeRTOS 队列串联；每个节点内部维护一个 `RingBuf<cam_fb_t *>`。

### 3.1 `WhoFrameCap`

```cpp
class WhoFrameCap : public task::WhoTaskGroup {
public:
    // 模板化添加节点：会自动串联到链尾，并在相邻节点间创建队列
    template <typename T, typename... Args>
    void add_node(Args &&...args);

    // 按启动元组列表启动所有节点（栈深/优先级/核）
    bool run(std::vector<std::tuple<const configSTACK_DEPTH_TYPE, UBaseType_t, const BaseType_t>> args);

    WhoFrameCapNode *get_node(const std::string &name);
    WhoFrameCapNode *get_node(int i);
    WhoFrameCapNode *get_last_node();
    std::vector<WhoFrameCapNode *> get_all_nodes();
};
```

### 3.2 `WhoFrameCapNode` 基类

```cpp
class WhoFrameCapNode : public task::WhoTask {
public:
    static inline constexpr EventBits_t NEW_FRAME = TASK_EVENT_BIT_LAST;  // 新帧就绪事件位

    WhoFrameCapNode(const std::string &name, uint8_t ringbuf_len, bool out_queue_overwrite = true);

    who::cam::cam_fb_t *cam_fb_peek(int index = -1);   // -1 表示最新一帧
    void add_new_frame_signal_subscriber(task::WhoTask *task);  // ringbuf 满时给订阅者置 NEW_FRAME
    WhoFrameCapNode *get_prev_node();
    WhoFrameCapNode *get_next_node();
    virtual uint16_t get_fb_width() = 0;
    virtual uint16_t get_fb_height() = 0;
    virtual std::string get_type() = 0;
};
```

### 3.3 内置节点类型

| 类 | type 字符串 | 作用 |
|---|---|---|
| `WhoFetchNode` | `"FetchNode"` | 从 `WhoCam` 取帧；ringbuf_len 自动取 `cam->get_fb_count() - 2` |
| `WhoDecodeNode` | `"DecodeNode"` | JPEG/MJPEG 软/硬件解码（`CONFIG_SOC_JPEG_CODEC_SUPPORTED` 时用硬件） |
| `WhoPPAResizeNode` | `"PPAResizeNode"` | 仅 `CONFIG_SOC_PPA_SUPPORTED`（ESP32-P4）可用，用 PPA 硬件缩放 |

```cpp
class WhoFetchNode : public WhoFrameCapNode {
public:
    WhoFetchNode(const std::string &name, who::cam::WhoCam *cam, bool out_queue_overwrite = true);
};

class WhoDecodeNode : public WhoFrameCapNode {
public:
    WhoDecodeNode(const std::string &name,
                  dl::image::pix_type_t pix_type,
                  uint8_t ringbuf_len,
                  bool out_queue_overwrite = true);
};

#if CONFIG_SOC_PPA_SUPPORTED
class WhoPPAResizeNode : public WhoFrameCapNode {
public:
    WhoPPAResizeNode(const std::string &name,
                     uint16_t dst_w,
                     uint16_t dst_h,
                     dl::image::pix_type_t dst_pix_type,
                     uint8_t ringbuf_len,
                     bool out_queue_overwrite = true);
};
#endif
```

---

## 4. 摄像头抽象 `who::cam`

> 头文件：`components/who_peripherals/who_cam/who_cam_base.hpp`、`who_cam_define.hpp`、`who_cam.hpp`

`who_cam.hpp` 会根据目标芯片选择 `WhoS3Cam`（ESP32-S3）或 `WhoP4Cam`（ESP32-P4），并始终提供 `WhoUVCCam`。

### 4.1 帧缓冲结构 `cam_fb_t`

```cpp
enum class cam_fb_fmt_t { CAM_FB_FMT_RGB565, CAM_FB_FMT_RGB888, CAM_FB_FMT_JPEG, CAM_FB_FMT_UKN };

typedef struct cam_fb_s {
    void *buf;
    size_t len;
    uint16_t width;
    uint16_t height;
    cam_fb_fmt_t format;
    struct timeval timestamp;
    void *ret;
    operator dl::image::img_t() const;   // 可隐式转换为 ESP-DL 图像结构
} cam_fb_t;
```

### 4.2 `WhoCam` 基类

```cpp
class WhoCam {
public:
    WhoCam(uint8_t fb_count);
    WhoCam(uint8_t fb_count, uint16_t fb_width, uint16_t fb_height);
    virtual cam_fb_t *cam_fb_get() = 0;
    virtual void cam_fb_return(cam_fb_t *fb) = 0;
    uint16_t get_fb_width();
    uint16_t get_fb_height();
    uint8_t get_fb_count();
    virtual cam_fb_fmt_t get_fb_format() = 0;
};
```

### 4.3 `WhoS3Cam`（ESP32-S3，基于 esp32-camera）

```cpp
class WhoS3Cam : public WhoCam {
public:
    WhoS3Cam(const pixformat_t pixel_format,
             const framesize_t frame_size,
             const uint8_t fb_count,
             bool vertical_flip = false,
             bool horizontal_flip = true);
    cam_fb_t *cam_fb_get() override;
    void cam_fb_return(cam_fb_t *fb) override;
};
```

辅助函数（仅 ESP32-S3）：
```cpp
framesize_t get_cam_frame_size_from_lcd_resolution();  // 根据 BSP_LCD_H/V_RES 选最大不超过的 framesize
cam_fb_fmt_t pix_fmt2cam_fb_fmt(pixformat_t pix_fmt);
```

### 4.4 `WhoP4Cam`（ESP32-P4，基于 V4L2 / esp_video）

```cpp
class WhoP4Cam : public WhoCam {
public:
    WhoP4Cam(const uint32_t v4l2_fmt,
             const uint8_t fb_count,
             const v4l2_memory fb_mem_type = V4L2_MEMORY_USERPTR,
             bool vertical_flip = false,
             bool horizontal_flip = true);
    cam_fb_t *cam_fb_get() override;
    void cam_fb_return(cam_fb_t *fb) override;
};
```

### 4.5 `WhoUVCCam`（USB UVC 摄像头）

```cpp
class WhoUVCCam : public WhoCam {
public:
    WhoUVCCam(const uvc_host_stream_format fmt,
              uint16_t h_res, uint16_t v_res, float fps,
              const uint8_t fb_count);
};
// fmt 通常为 UVC_VS_FORMAT_MJPEG
```

---

## 5. 目标检测 `who::detect`

> 头文件：`components/who_detect/who_detect.hpp`

`WhoDetect` 是一个 `WhoTask`，订阅上游 `WhoFrameCapNode` 的 `NEW_FRAME`，对最新帧调用注入的 ESP-DL 模型 `run(img)`，并通过回调把结果送出。

```cpp
class WhoDetect : public task::WhoTask {
public:
    static inline constexpr EventBits_t NEW_FRAME = frame_cap::WhoFrameCapNode::NEW_FRAME;

    typedef struct {
        std::list<dl::detect::result_t> det_res;   // ESP-DL 结果（box + 可选 keypoint + score）
        struct timeval timestamp;
        dl::image::img_t img;
    } result_t;

    WhoDetect(const std::string &name, frame_cap::WhoFrameCapNode *frame_cap_node);
    ~WhoDetect();   // 析构会 delete 注入的 m_model
    void set_model(dl::detect::Detect *model);                       // 运行前必须调用
    void set_rescale_params(float rescale_x, float rescale_y,        // PPA 缩放后坐标还原用
                            uint16_t rescale_max_w, uint16_t rescale_max_h);
    void set_fps(float fps);                                          // fps>0 时按间隔节流
    void set_detect_result_cb(const std::function<void(const result_t &)> &result_cb);
    void set_cleanup_func(const std::function<void()> &cleanup_func);
    bool run(const configSTACK_DEPTH_TYPE uxStackDepth,
             UBaseType_t uxPriority, const BaseType_t xCoreID) override;  // 未 set_model 直接返回 false
};
```

辅助绘图/打印（`who_detect_result_handle.hpp`）：
```cpp
namespace who::detect {
void draw_detect_results_on_img(const dl::image::img_t &img,
                                const std::list<dl::detect::result_t> &detect_res,
                                const std::vector<std::vector<uint8_t>> &palette);
#if !BSP_CONFIG_NO_GRAPHIC_LIB
void draw_detect_results_on_canvas(lv_obj_t *canvas,
                                   const std::list<dl::detect::result_t> &detect_res,
                                   const std::vector<lv_color_t> &palette);
#endif
void print_detect_results(const std::list<dl::detect::result_t> &detect_res);
}
```

---

## 6. 人脸识别 `who::recognition`

> 头文件：`components/who_recognition/who_recognition.hpp`

`WhoRecognition` 是 `WhoTaskGroup`，内部组合 `WhoDetect`（人脸检测）+ `WhoRecognitionCore`（特征提取/比对）。`WhoRecognitionCore` 通过事件位 `RECOGNIZE` / `ENROLL` / `DELETE` 触发对应动作。

```cpp
class WhoRecognitionCore : public task::WhoTask {
public:
    static inline constexpr EventBits_t RECOGNIZE = TASK_EVENT_BIT_LAST;
    static inline constexpr EventBits_t ENROLL    = TASK_EVENT_BIT_LAST << 1;
    static inline constexpr EventBits_t DELETE    = TASK_EVENT_BIT_LAST << 2;

    WhoRecognitionCore(const std::string &name, detect::WhoDetect *detect);
    void set_recognizer(HumanFaceRecognizer *recognizer);   // 运行前必须调用
    void set_recognition_result_cb(const std::function<void(const std::string &)> &result_cb);
    void set_detect_result_cb(const std::function<void(const detect::WhoDetect::result_t &)> &result_cb);
    void set_cleanup_func(const std::function<void()> &cleanup_func);
    bool run(...) override;   // 未 set_recognizer 直接返回 false
};

class WhoRecognition : public task::WhoTaskGroup {
public:
    WhoRecognition(frame_cap::WhoFrameCapNode *frame_cap_node);
    void set_detect_model(dl::detect::Detect *model);
    void set_recognizer(HumanFaceRecognizer *recognizer);
    detect::WhoDetect *get_detect_task();
    WhoRecognitionCore *get_recognition_task();
};
```

> 底层 `HumanFaceRecognizer`（`recognize()` / `enroll()` / `delete_last_feat()` / `get_num_feats()`）来自 ESP-DL `human_face_recognition` 组件。

---

## 7. 二维码识别 `who::qrcode`

> 头文件：`components/who_qrcode/who_qrcode.hpp`（依赖 `quirc` 库）

```cpp
class WhoQRCode : public task::WhoTask {
public:
    static inline constexpr EventBits_t NEW_FRAME = frame_cap::WhoFrameCapNode::NEW_FRAME;
    WhoQRCode(const std::string &name, frame_cap::WhoFrameCapNode *frame_cap_node);
    ~WhoQRCode();
    void set_qrcode_result_cb(const std::function<void(const std::string &)> &result_cb);
    void set_cleanup_func(const std::function<void()> &cleanup_func);
};
```

内部使用 `quirc_new()` / `quirc_resize()` / `quirc_begin()` / `quirc_end()` / `quirc_count()` / `quirc_extract()` / `quirc_decode()` / `quirc_flip()`，并通过 `dl::image::ImageTransformer` 把彩色帧转灰度后送入 quirc。S3 上 quirc 尺寸为 `BSP_LCD_H_RES x BSP_LCD_V_RES`，P4 上为 `BSP_LCD_H_RES/2 x BSP_LCD_V_RES/2`。

---

## 8. 应用层封装 `who::app`

> 头文件：`components/who_app/who_app_common/who_app.hpp` 及各 `who_*_app` 子目录

`WhoApp` 是面向用户的“一键 run”封装，内部管理 task group，并提供 `run()` / `pause()` / `resume()` / `stop()`。

```cpp
class WhoApp {
public:
    virtual bool run() = 0;
    virtual bool pause();
    virtual bool resume();
    virtual bool stop();
};
```

### 8.1 检测类应用（`who_detect_app`）

| 类 | 头文件 | 用途 |
|---|---|---|
| `WhoDetectAppBase` | `who_detect_app_base.hpp` | 基类，需 `set_model()` |
| `WhoDetectAppLCD` | `who_detect_app_lcd.hpp` | 检测结果画框 + LCD 显示 |
| `WhoDetectAppTerm` | `who_detect_app_term.hpp` | 检测结果打印到串口 |

```cpp
class WhoDetectAppLCD : public WhoDetectAppBase {
public:
    WhoDetectAppLCD(const std::vector<std::vector<uint8_t>> &palette,     // 调色板，每类一组 RGB
                    frame_cap::WhoFrameCap *frame_cap,
                    frame_cap::WhoFrameCapNode *lcd_disp_frame_cap_node = nullptr);
    void set_model(dl::detect::Detect *model);   // 继承自基类
    void set_fps(float fps);
    bool run() override;
};
```

### 8.2 人脸识别应用（`who_recognition_app`）

| 类 | 头文件 | 用途 |
|---|---|---|
| `WhoRecognitionAppBase` | `who_recognition_app_base.hpp` | 基类 |
| `WhoRecognitionAppLCD` | `who_recognition_app_lcd.hpp` | LCD + 按键（注册/识别/删除） |
| `WhoRecognitionAppTerm` | `who_recognition_app_term.hpp` | 串口输出 |

```cpp
class WhoRecognitionAppLCD : public WhoRecognitionAppBase {
public:
    WhoRecognitionAppLCD(frame_cap::WhoFrameCap *frame_cap);   // 内部已创建 HumanFaceDetect/HumanFaceRecognizer
    bool run() override;
};
```

> `WhoRecognitionAppLCD` 构造时即创建 `HumanFaceRecognizer` 与 `HumanFaceDetect`，并依据 Kconfig 选择数据库路径（`CONFIG_DB_FATFS_FLASH` / `CONFIG_DB_SPIFFS` / `CONFIG_DB_FATFS_SDCARD`）。

### 8.3 二维码应用（`who_qrcode_app`）

| 类 | 头文件 | 用途 |
|---|---|---|
| `WhoQRCodeAppBase` | `who_qrcode_app_base.hpp` | 基类 |
| `WhoQRCodeAppLCD` | `who_qrcode_app_lcd.hpp` | LCD 显示解码文本 |
| `WhoQRCodeAppTerm` | `who_qrcode_app_term.hpp` | 串口输出解码文本 |

```cpp
class WhoQRCodeAppLCD : public WhoQRCodeAppTerm {
public:
    WhoQRCodeAppLCD(frame_cap::WhoFrameCap *frame_cap);
    bool run() override;
};
```

---

## 9. LCD 显示 `who::lcd` / `who::lcd_disp`

> 头文件：`components/who_peripherals/who_lcd/who_lcd.hpp`、`who_lvgl_lcd.hpp`；`components/who_frame_lcd_disp/who_frame_lcd_disp.hpp`

`WhoLCD` 在未定义 `BSP_CONFIG_NO_GRAPHIC_LIB` 时走 LVGL（`who_lvgl_lcd.hpp`），否则走裸 `esp_lcd`（`who_lcd.hpp`）。

```cpp
// 带 LVGL
class WhoLCD {
public:
    WhoLCD(const lvgl_port_cfg_t &lvgl_port_cfg = {4, 6144, 0, 500, MALLOC_CAP_INTERNAL, 5});
    void init(const lvgl_port_cfg_t &lvgl_port_cfg);
    void deinit();
};

// 帧显示任务
class WhoFrameLCDDisp : public task::WhoTask {
public:
    static inline constexpr EventBits_t NEW_FRAME = frame_cap::WhoFrameCapNode::NEW_FRAME;
    WhoFrameLCDDisp(const std::string &name,
                    frame_cap::WhoFrameCapNode *frame_cap_node,
                    int peek_index = 0);
    void set_lcd_disp_cb(const std::function<void(who::cam::cam_fb_t *)> &lcd_disp_cb);
#if !BSP_CONFIG_NO_GRAPHIC_LIB
    lv_obj_t *get_canvas();
#endif
};
```

---

## 10. 其它外设组件

### 10.1 SPI Flash FAT 文件系统（`who_spiflash_fatfs`）

> 头文件：`components/who_peripherals/who_spiflash_fatfs/who_spiflash_fatfs.hpp`

```cpp
esp_err_t fatfs_flash_mount();
esp_err_t fatfs_flash_unmount();
```

挂载点由 `CONFIG_SPIFLASH_MOUNT_POINT`（默认 `/spiflash`）控制，分区由 `CONFIG_SPIFLASH_MOUNT_PARTITION`（默认 `storage`）控制。

### 10.2 USB Host（`who_usb`）

> 头文件：`components/who_peripherals/who_usb/who_usb.hpp`

单例任务，运行 USB Host 栈（UVC 摄像头依赖）。

```cpp
class WhoUSB : public task::WhoTaskBase {
public:
    static inline constexpr EventBits_t USB_HOST_INSTALLED = TASK_EVENT_BIT_LAST;
    static WhoUSB *get_instance();
    bool stop_async() override;
};
```

---

## 11. ESP-DL 检测模型（外部组件，ESP-WHO 调用）

ESP-WHO 通过 `idf_component_optional_requires` 按需引入下列模型组件（在 `object_detect` 示例中由 `-DDETECT_MODEL=` 选择）：

| 组件 | 头文件 | 类 |
|---|---|---|
| `human_face_detect` | `human_face_detect.hpp` | `HumanFaceDetect` |
| `pedestrian_detect` | `pedestrian_detect.hpp` | `PedestrianDetect` |
| `cat_detect` | `cat_detect.hpp` | `CatDetect` |
| `dog_detect` | `dog_detect.hpp` | `DogDetect` |
| `human_face_recognition` | `human_face_recognition.hpp` | `HumanFaceRecognizer`、`HumanFaceFeat` |

构造形如（均继承自 `dl::detect::Detect`）：
```cpp
new HumanFaceDetect(static_cast<HumanFaceDetect::model_type_t>(CONFIG_DEFAULT_HUMAN_FACE_DETECT_MODEL), false);
new HumanFaceRecognizer(db_path,
    static_cast<HumanFaceFeat::model_type_t>(CONFIG_DEFAULT_HUMAN_FACE_FEAT_MODEL), false);
```

> 这些类的具体成员由对应 ESP-DL 组件提供，本仓库不包含其源码；在 ESP-WHO 中按上方签名调用即可。
