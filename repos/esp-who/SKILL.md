---
name: esp-who-skill
description: >-
  AI Skill for Espressif ESP-WHO image processing / AI vision framework. Used when developing
  face detection, face recognition, pedestrian/cat/dog detection, and QR code recognition
  firmware on ESP32-S3 and ESP32-P4 using the ESP-WHO component library and its examples.
  Trigger words: "ESP-WHO", "esp-who", "人脸检测", "人脸识别", "行人检测", "二维码识别",
  "WhoDetect", "WhoRecognition", "WhoQRCode", "WhoFrameCap", "ESP32-S3-EYE", "ESP32-P4",
  "ESP-DL", "目标检测", "face recognition", "object detect"
license: MIT
metadata:
  author: Community
  version: "1.0.0"
---

# esp-who-skill

ESP-WHO（Espressif 人像/视觉处理平台）AI 开发技能。基于 `ESP-DL`，提供人脸检测、人脸识别、行人/猫/狗检测、二维码识别的示例与可复用组件。本技能提供场景化配方、完整 API 速查、配置参考与陷阱清单，所有内容均来自 `esp-who` 仓库真实的头文件、`examples/` 与 `components/`。

## Core Principles

1. **永远基于流水线模型** — ESP-WHO 一切围绕 `WhoFrameCap` 节点链：`WhoFetchNode → [WhoDecodeNode] → [WhoPPAResizeNode] → 下游任务`。先用 `add_node<T>(...)` 组链，再把尾节点喂给 App。
2. **App 是一键封装** — 优先用 `WhoDetectAppLCD/Term`、`WhoRecognitionAppLCD/Term`、`WhoQRCodeAppLCD/Term`，它们内部已编排任务栈深/核绑定/回调。直接用底层 `WhoDetect`/`WhoRecognition` 只在定制时。
3. **detect 类 App 必须 `set_model` 再 `run`** — `WhoDetect::run()` 开头 `if (!m_model) { ESP_LOGE; return false; }`；模型在 `run()` 前注入，且延后创建以减少内存碎片。
4. **任务注册必须在 App `run()` 之前** — `WhoTaskGroup::register_task` 有断言 `assert(!WhoYield2Idle::get_instance()->is_active())`，`run()` 后 `WhoYield2Idle` 已激活，再注册会崩溃。
5. **`IDF_EXTRA_ACTIONS_PATH` 必须先设** — `tools/bsp_ext.py` 是 idf.py 扩展，负责注入 `BSP`/`DETECT_MODEL` 到 CMake cache。不设则顶层 CMake `FATAL_ERROR: BSP is not defined`。
6. **BSP 与 target 强校验** — `BSP2IDF_TARGET`：S3 的 BSP 只能配 `esp32s3`，P4 的只能配 `esp32p4`，不匹配 `bsp_ext.py` 直接退出。
7. **`fb_count` / `ringbuf_len` 要够大** — `WhoFetchNode` 的 ringbuf_len = `cam->get_fb_count()-2`；约定 LCD 模式 `MODEL_TIME+3`、term 模式 `MODEL_TIME+2`。不确定就用 5 或更大。
8. **term 模式配 `*_noglib`，LCD 模式配无后缀 BSP** — `BSP_CONFIG_NO_GRAPHIC_LIB` 决定是否走 LVGL。`WhoDetectAppLCD` 需图形库，`WhoDetectAppTerm` 适配 noglib。
9. **人脸识别数据库路径由 Kconfig 决定** — `CONFIG_DB_FATFS_FLASH`/`CONFIG_DB_SPIFFS`/`CONFIG_DB_FATFS_SDCARD` 各对应一个挂载函数，`app_main` 必须先挂载再 `run()`。
10. **P4 的 PPA 缩放需传显示源节点** — 用 `WhoPPAResizeNode` 时把缩放前的 FetchNode 作为 `WhoDetectAppLCD` 第三参，App 内部自动 `set_rescale_params` 还原检测框坐标。
11. **任务核绑定不能 `tskNO_AFFINITY`** — `WhoTask::run()` 断言 `xCoreID != tskNO_AFFINITY`，采集/LCD 走核0，检测/识别走核1。
12. **模型组件不在本仓库** — `human_face_detect`/`pedestrian_detect`/`cat_detect`/`dog_detect`/`human_face_recognition` 由 `idf_component.yml` 从 Component Registry 拉取；ESP-WHO 只调用，不包含其源码。

## When to Use

**适用：**
- 在 ESP32-S3 / ESP32-P4 上构建人脸检测、人脸识别、行人/猫/狗检测、二维码识别应用
- 基于 ESP-WHO 官方示例（`human_face_recognition` / `object_detect` / `qrcode_recognition`）做定制
- 组装或定制 `WhoFrameCap` 帧采集流水线（DVP / MIPI-CSI / USB UVC）
- 继承 `WhoDetectApp*` / `WhoRecognitionApp*` 自定义检测结果处理、画框、上报
- 调试 ESP-WHO 构建问题（BSP 选择、`DETECT_MODEL`、依赖锁、分区表）
- 控制任务生命周期（pause/resume/stop）或编写自定义 `WhoTask`

**不适用：**
- 非 ESP32 芯片（CH57x、STM32、RP2040 等）
- 直接训练/量化模型（参考 ESP-DL / ESP-DETECTION 仓库，非 ESP-WHO 职责）
- esp-who 旧分支（`release/v1.1.0`）的 esp32 / esp32-s2 / 猫脸检测 / 颜色检测 API（本技能仅覆盖重构后的主分支）
- PCB 硬件设计、原理图、摄像头电气连接

---

## Scenario Quick Reference (Recipes)

当用户意图命中下列场景时，**先读对应配方** — 它包含完整调用链、分步说明、常见错误与可复制代码。

### 工程构建

| recipe | 场景 |
|---|---|
| `recipes/project_setup.md` | 从零搭建 ESP-WHO 示例：环境变量、BSP 选择、set-target、烧录 |

### 目标检测

| recipe | 场景 |
|---|---|
| `recipes/object_detect_lcd.md` | 目标检测 + LCD 实时画框（人脸/行人/猫/狗） |
| `recipes/object_detect_term.md` | 目标检测无 LCD，结果打印串口（`*_noglib`） |
| `recipes/custom_detect_app.md` | 继承 `WhoDetectApp*` 自定义结果回调与绘制 |

### 人脸识别

| recipe | 场景 |
|---|---|
| `recipes/human_face_recognition.md` | 人脸识别全流程：检测 + 注册/识别/删除 + 数据库文件系统 |

### 二维码识别

| recipe | 场景 |
|---|---|
| `recipes/qrcode_recognition.md` | 实时二维码识别（quirc）+ LCD/串口输出 |

### 流水线与摄像头

| recipe | 场景 |
|---|---|
| `recipes/frame_cap_pipeline.md` | 自定义 `WhoFrameCap` 节点链（PPA / UVC / fb_count 规则） |
| `recipes/camera_selection.md` | `WhoS3Cam` / `WhoP4Cam` / `WhoUVCCam` 选型与初始化 |

### 任务控制

| recipe | 场景 |
|---|---|
| `recipes/task_lifecycle.md` | `WhoTask` pause/resume/stop（同步/异步）、事件位、自定义任务 |

---

## 开发板与芯片支持表

（来自仓库 `README.md` 与 `bsp_ext.py`）

| 开发板 (BSP 名) | SoC | 摄像头 | LCD | 触摸/按键 | 备注 |
|---|---|---|---|---|---|
| `esp32_s3_eye` | esp32s3 | 板载 camera | st7789 | 物理按键（PLAY/UP/DOWN） | 含 IMU/MIC/uSD |
| `esp32_s3_korvo_2` | esp32s3 | 板载 camera | ili9341 | 物理按键（PLAY/VOL±） + tt21100 触摸 | 含 es7210/es8311 音频 |
| `esp32_p4_function_ev_board` | esp32p4 | MIPI SC2336 | ek79007/ili9881c/lt8912b | gt911 触摸（LVGL 按钮） | 支持 PPA / USB Host |

> 每个 BSP 都有对应 `sdkconfig.bsp.<name>`；`object_detect` 示例统一用 `<name>_noglib` 变体。ESP-WHO 主分支**不支持** esp32 / esp32-s2（需回退 `release/v1.1.0`）。

## ESP-IDF 版本支持

| ESP-IDF release/v5.4 | ESP-IDF release/v5.5 |
|---|---|
| 支持 | 支持 |

---

## 关键配置速查（节选）

完整列表见 `resources/config_reference.md`。

| 符号 | 默认 | 含义 |
|---|---|---|
| `CONFIG_MAX_TASK_LOOP_TIME` | 1 | 单任务最大循环秒数，接近 WDT 阈值需调大 `CONFIG_ESP_TASK_WDT_TIMEOUT_S` |
| `CONFIG_DB_FATFS_FLASH` | 选中 | 人脸库走 SPI Flash FAT（`/spiflash/face.db`） |
| `CONFIG_SPIFLASH_MOUNT_POINT` | `/spiflash` | FAT 挂载点 |
| `CONFIG_SPIFLASH_MOUNT_PARTITION` | `storage` | FAT 分区名 |
| `-DSDKCONFIG_DEFAULTS` | — | 选 BSP 默认配置（必需） |
| `-DDETECT_MODEL` | — | `object_detect` 必需，4 选 1 |
| `CONFIG_SOC_PPA_SUPPORTED` | P4 | 启用 `WhoPPAResizeNode` |
| `CONFIG_SOC_JPEG_CODEC_SUPPORTED` | P4 | `WhoDecodeNode` 用硬件 JPEG |
| `BSP_CONFIG_NO_GRAPHIC_LIB` | `*_noglib` | 禁用 LVGL 分支 |

## 检测模型与 MODEL_TIME 映射

（来自 `examples/object_detect/main/frame_cap_pipeline.cpp`）

| DETECT_MODEL | 定义宏 | MODEL_TIME | 输入尺寸 |
|---|---|---|---|
| `human_face_detect` | `CONFIG_HUMAN_FACE_DETECT_MODEL_LOCATION` | 2 | 160×120 |
| `pedestrian_detect` | `CONFIG_PEDESTRIAN_DETECT_MODEL_LOCATION` | 3 | 224×224 |
| `cat_detect` (pico_224_224) | `CONFIG_ESPDET_PICO_224_224_CAT` | 3 | 224×224 |
| `cat_detect` (pico_416_416) | `CONFIG_ESPDET_PICO_416_416_CAT` | 8 | 416×416 |
| `dog_detect` (pico_224_224) | `CONFIG_ESPDET_PICO_224_224_DOG` | 3 | 224×224 |
| `dog_detect` (pico_416_416) | `CONFIG_ESPDET_PICO_416_416_DOG` | 8 | 416×416 |

---

## 任务状态机（摘要）

完整状态机与数据流见 `resources/state_machine.md`。

```
WhoTaskBase:  [TASK_STOPPED] ─run()─▶ [RUNNING] ⇄ [TASK_PAUSED] ─stop()─▶ [TASK_STOPPED]
                  控制位: TASK_STOP / TASK_PAUSE / TASK_RESUME（调用方置位，任务内消费）

FrameCap 数据流:  WhoCam ─▶ WhoFetchNode ─▶ [WhoDecodeNode] ─▶ [WhoPPAResizeNode] ─▶ 下游 App
                       ringbuf 满 → 给订阅者(WhoDetect/WhoQRCode)置 NEW_FRAME

WhoRecognition 事件:  RECOGNIZE / ENROLL / DELETE（由按键触发，动态挂载一次性回调到 WhoDetect）
```

---

## Critical Pitfalls (Must Read)

下列是最常见错误。违反任一条都会导致构建失败或运行异常。完整说明见 `resources/pitfalls.md`。

### 1. 漏设 `IDF_EXTRA_ACTIONS_PATH`

```bash
# ❌ WRONG — 直接 set-target，CMake 报 BSP is not defined
idf.py set-target esp32s3

# ✅ CORRECT — 先设扩展路径，再带 SDKCONFIG_DEFAULTS
export IDF_EXTRA_ACTIONS_PATH=/path_to_esp-who/tools/
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye set-target esp32s3
```

### 2. BSP 与 target 不匹配

```bash
# ❌ WRONG
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_p4_function_ev_board set-target esp32s3
# bsp_ext.py: "BSP esp32_p4_function_ev_board does not match idf_target esp32s3."

# ✅ CORRECT
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_p4_function_ev_board set-target esp32p4
```

### 3. `object_detect` 未指定 `-DDETECT_MODEL`

```bash
# ❌ WRONG — get_detect_model() 走到 ESP_LOGE 返回 nullptr
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib set-target esp32s3

# ✅ CORRECT
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib -DDETECT_MODEL=human_face_detect set-target esp32s3
```

### 4. detect App 漏 `set_model`

```cpp
// ❌ WRONG
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);
app->run();   // WhoDetect::run() 因 m_model==nullptr 返回 false

// ✅ CORRECT
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);
app->set_model(get_detect_model());
app->run();
```

### 5. 在 App `run()` 之后注册任务

```cpp
// ❌ WRONG — assert(!WhoYield2Idle::get_instance()->is_active()) 失败
app->run();
app->add_task(custom);

// ✅ CORRECT — 所有结构在 run() 之前就绪
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);
app->set_model(get_detect_model());
app->run();
```

### 6. `app_main` 未提升优先级

```cpp
// ❌ WRONG — run() 返回后 main 退出，app 被 delete
extern "C" void app_main(void) {
    app->run();
}

// ✅ CORRECT
extern "C" void app_main(void) {
    vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);   // 高于子任务 2
    app->run();
}
```

### 7. `fb_count` 太小导致断言

```cpp
// ❌ WRONG — fb_count=2 → FetchNode ringbuf_len=0 → assert(ringbuf_len>=1)
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, fs, 2);

// ✅ CORRECT
#define MODEL_TIME 2
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, fs, MODEL_TIME + 3);
```

### 8. P4 用 PPA 却没传显示源节点

```cpp
// ❌ WRONG — 显示缩放后小图，检测框错位
auto frame_cap = get_lcd_mipi_csi_ppa_frame_cap_pipeline(&lcd_node);
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap);   // 第三参漏

// ✅ CORRECT
WhoFrameCapNode *lcd_node = nullptr;
auto frame_cap = get_lcd_mipi_csi_ppa_frame_cap_pipeline(&lcd_node);
auto app = new WhoDetectAppLCD({{255,0,0}}, frame_cap, lcd_node);
```

### 9. 人脸识别未挂载文件系统

```cpp
// ❌ WRONG
auto app = new WhoRecognitionAppLCD(frame_cap);
app->run();   // face.db 不可用

// ✅ CORRECT
#if CONFIG_DB_FATFS_FLASH
    ESP_ERROR_CHECK(fatfs_flash_mount());
#elif CONFIG_SPIFFS
    ESP_ERROR_CHECK(bsp_spiffs_mount());
#endif
auto app = new WhoRecognitionAppLCD(frame_cap);
app->run();
```

### 10. term / LCD 与 BSP 后缀错配

```bash
# ❌ WRONG — LCD 应用配 noglib（BSP_CONFIG_NO_GRAPHIC_LIB 禁用 LVGL）
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib set-target esp32s3
# 然后 new WhoDetectAppLCD(...) → 图形库分支错乱

# ✅ CORRECT
#   LCD 模式: sdkconfig.bsp.<name>      + WhoDetectAppLCD
#   term 模式: sdkconfig.bsp.<name>_noglib + WhoDetectAppTerm
```

### 11. `xCoreID` 传 `tskNO_AFFINITY`

```cpp
// ❌ WRONG
detect->run(4096, 2, tskNO_AFFINITY);   // assert(xCoreID != tskNO_AFFINITY)

// ✅ CORRECT
detect->run(4096, 2, 1);   // 采集/LCD 核0，检测/识别核1
```

### 12. `WhoS3Cam` 参数顺序错

```cpp
// ❌ WRONG
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, 5, true);   // 5 当 framesize

// ✅ CORRECT
framesize_t fs = get_cam_frame_size_from_lcd_resolution();
auto cam = new WhoS3Cam(PIXFORMAT_RGB565, fs, MODEL_TIME + 3);
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | 明确需求 | 确定场景（检测/识别/二维码）、芯片（S3/P4）、是否有 LCD、模型选择 |
| 2 | 查配方 | 在 `recipes/` 找匹配场景；命中则按其调用链推进 |
| 3 | 查 API | 配方未覆盖的 API/结构体查 `resources/api_reference.md` |
| 4 | 查配置 | 配置项/分区表/构建变量查 `resources/config_reference.md` |
| 5 | 查陷阱 | 对照 `resources/pitfalls.md` 与上方 Critical Pitfalls |
| 6 | 复制示例 | 优先从 `examples/<name>/` 复制最接近的示例再改，而非从零写 |
| 7 | 核对 | 按上方 12 条 Critical Pitfalls 与 `AGENTS.md` Checklist 逐项核对 |
| 8 | 构建 | 设 `IDF_EXTRA_ACTIONS_PATH` → `set-target` + `-DSDKCONFIG_DEFAULTS`（+`-DDETECT_MODEL`）→ `flash monitor` |
| 9 | 调试 | 串口 monitor 查 ESP_LOG；用 `WhoTaskState` 周期打印任务状态；LCD 模式看画面 |

### Step 6 — 工程创建策略

**目标目录无现有工程（首次创建）：**
1. 按 BSP/场景选最近示例：
   - 人脸识别 → `examples/human_face_recognition/`
   - 目标检测 → `examples/object_detect/`（带 `DETECT_MODEL`）
   - 二维码 → `examples/qrcode_recognition/`
2. 复制整个示例目录到用户工程目录，保留 `main/`、`sdkconfig.bsp.*`、`partitions.csv`、顶层 `CMakeLists.txt`（修正 `EXTRA_COMPONENT_DIRS` 指向真实的 `components/` 相对路径）。
3. 改 `main/app_main.cpp` 与 `frame_cap_pipeline.cpp` 满足需求。
4. 说明复制来源与改动点。

**目标目录已有工程：** 原地编辑，不覆盖。

---

## Failure Strategies

| 情况 | 行动 |
|---|---|
| API 不在 `resources/api_reference.md` | 停下，告知用户该 API 非本仓库提供（可能在 ESP-DL/ESP-BSP） |
| 不确定 BSP 选哪个 | 列 `examples/<name>/sdkconfig.bsp.*` 文件名给用户选 |
| 不确定 `fb_count` | 默认 `MODEL_TIME + 3`（LCD）/ `MODEL_TIME + 2`（term） |
| 构建 `BSP is not defined` | 检查 `IDF_EXTRA_ACTIONS_PATH` 是否设且 echo 正确 |
| 模型组件拉取失败 | 确认 `idf_component.yml` 与 `dependencies.lock.<bsp>[.model]`；必要时 `tools/gen_dependencies_lock.py` |
| 任务起不来（run 返回 false） | 多半是漏 `set_model` / `set_recognizer`；查 ESP_LOGE |
| 任务卡死不退出 | 节点类需 override `stop_async`/`pause_async` 发 nullptr 帧唤醒阻塞点 |
| 检测框与画面错位 | 增大 `fb_count`/`ringbuf_len`；P4 检查是否漏传显示源节点 |

## References

- 场景配方 → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置参考 → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 示例与组件清单 → `resources/example_list.md`
- 状态机与数据流 → `resources/state_machine.md`
- 工程约定 → `AGENTS.md`
- 源仓库 → `esp-who`（`components/`、`examples/`、`tools/`）
