# ESP-WHO 示例与组件清单

> 全部路径相对于仓库根目录 `esp-who/`。

---

## 1. 官方示例 `examples/`

| 路径 | 说明 | 入口 |
|---|---|---|
| `examples/human_face_recognition/` | 人脸识别（检测 + 特征提取 + 注册/识别/删除） | `main/app_main.cpp` |
| `examples/object_detect/` | 目标检测（可选人脸/行人/猫/狗 4 种模型） | `main/app_main.cpp` |
| `examples/qrcode_recognition/` | 二维码识别（基于 quirc） | `main/app_main.cpp` |

每个示例的标准结构：
```
examples/<name>/
├── CMakeLists.txt                 # 顶层，设置 EXTRA_COMPONENT_DIRS + 依赖锁
├── README.md
├── partitions.csv                 # 分区表（factory app + storage fat）
├── sdkconfig.bsp.<bsp_name>       # 各开发板默认配置（含 _noglib 变体）
├── dependencies.lock.<bsp>[.model]# 各 BSP/模型的依赖锁
└── main/
    ├── CMakeLists.txt             # idf_component_register + requires
    ├── idf_component.yml          # 组件依赖（被 bsp_ext.py 动态改写）
    ├── app_main.cpp               # app_main()
    ├── frame_cap_pipeline.cpp     # 摄像头→采集节点流水线构造
    └── frame_cap_pipeline.hpp
```

### 各示例支持的 BSP / 模型组合

| 示例 | 支持的 BSP |
|---|---|
| `human_face_recognition` | `esp32_s3_eye`, `esp32_s3_korvo_2`, `esp32_p4_function_ev_board` |
| `object_detect` | 上述三个 BSP 的 `_noglib` 变体（`*_noglib`）；模型 `human_face_detect`/`pedestrian_detect`/`cat_detect`/`dog_detect` |
| `qrcode_recognition` | `esp32_s3_eye`, `esp32_p4_function_ev_board` |

---

## 2. 可复用组件 `components/`

| 组件路径 | 提供的命名空间 / 关键类 |
|---|---|
| `components/who_task/` | `who::task::WhoTask` / `WhoTaskBase` / `WhoTaskGroup` / `WhoTaskState`；`who::WhoYield2Idle` |
| `components/who_frame_cap/` | `who::frame_cap::WhoFrameCap` / `WhoFrameCapNode` / `WhoFetchNode` / `WhoDecodeNode` / `WhoPPAResizeNode` |
| `components/who_frame_lcd_disp/` | `who::lcd_disp::WhoFrameLCDDisp` |
| `components/who_detect/` | `who::detect::WhoDetect` |
| `components/who_recognition/` | `who::recognition::WhoRecognition` / `WhoRecognitionCore` |
| `components/who_qrcode/` | `who::qrcode::WhoQRCode` |
| `components/who_app/who_app_common/` | `who::app::WhoApp`；`who::detect::print_detect_results`；`who::lcd_disp::WhoDetectResultLCDDisp` / `WhoTextResultLCDDisp` |
| `components/who_app/who_detect_app/` | `who::app::WhoDetectAppBase` / `WhoDetectAppLCD` / `WhoDetectAppTerm` |
| `components/who_app/who_recognition_app/` | `who::app::WhoRecognitionAppBase` / `WhoRecognitionAppLCD` / `WhoRecognitionAppTerm`；`who::button::WhoRecognitionButton*` |
| `components/who_app/who_qrcode_app/` | `who::app::WhoQRCodeAppBase` / `WhoQRCodeAppLCD` / `WhoQRCodeAppTerm` |
| `components/who_peripherals/who_cam/` | `who::cam::WhoCam` / `WhoS3Cam` / `WhoP4Cam` / `WhoUVCCam` |
| `components/who_peripherals/who_lcd/` | `who::lcd::WhoLCD`（LVGL 或裸 esp_lcd） |
| `components/who_peripherals/who_spiflash_fatfs/` | `fatfs_flash_mount()` / `fatfs_flash_unmount()` |
| `components/who_peripherals/who_usb/` | `who::usb::WhoUSB`（单例） |

---

## 3. 构建辅助 `tools/`

| 路径 | 作用 |
|---|---|
| `tools/bsp_ext.py` | idf.py 扩展：解析 `-DSDKCONFIG_DEFAULTS` / `-DDETECT_MODEL`，校验 BSP↔target，动态改写 `main/idf_component.yml`，并写入 `BSP`/`DETECT_MODEL` 到 CMake cache |
| `tools/gen_dependencies_lock.py` | 生成各 BSP/模型的 `dependencies.lock.*` |
| `tools/ci/` | CI 脚本 |

使用前置：`export IDF_EXTRA_ACTIONS_PATH=/path_to_esp-who/tools/`（Windows PowerShell：`$Env:IDF_EXTRA_ACTIONS_PATH="..."`）。
