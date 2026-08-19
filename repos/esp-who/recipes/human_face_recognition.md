# 人脸识别（检测 + 注册 + 识别 + 删除）

> **适用摘要**: 使用 `WhoRecognitionAppLCD`（或 `WhoRecognitionAppTerm`）跑完整人脸识别流程：实时检测人脸、按键注册新面孔、识别已注册人脸、删除最后一条特征。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-who/resources/`, source/examples in `repos/esp-who/`, and this recipe path `repos/esp-who/recipes/human_face_recognition.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "人脸识别"
- "face recognition 注册"
- "enroll / recognize face id"
- "human_face_recognition 示例怎么用"
- "face.db 在哪"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/human_face_recognition/` |
| 组件 | `espressif/human_face_recognition`（由 `who_recognition/idf_component.yml` 拉取，内含 `HumanFaceDetect` + `HumanFaceFeat` + `HumanFaceRecognizer`） |
| 数据库文件系统 | `CONFIG_DB_FATFS_FLASH`（默认）/ `CONFIG_DB_SPIFFS` / `CONFIG_DB_FATFS_SDCARD` |

## 分步说明

### 1. 选 BSP + target（人脸识别无 `_noglib` 变体）

```bash
cd examples/human_face_recognition
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye set-target esp32s3
# 或 esp32_p4_function_ev_board / esp32p4
```

### 2. `app_main`（照搬 `examples/human_face_recognition/main/app_main.cpp`）

```cpp
#include "frame_cap_pipeline.hpp"
#include "who_recognition_app_lcd.hpp"
#include "who_recognition_app_term.hpp"
#include "who_spiflash_fatfs.hpp"

using namespace who::frame_cap;
using namespace who::app;

extern "C" void app_main(void)
{
    vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);

    // 按数据库文件系统挂载（决定 face.db 路径）
#if CONFIG_DB_FATFS_FLASH
    ESP_ERROR_CHECK(fatfs_flash_mount());
#elif CONFIG_SPIFFS
    ESP_ERROR_CHECK(bsp_spiffs_mount());
#endif
#if CONFIG_DB_FATFS_SDCARD || CONFIG_HUMAN_FACE_DETECT_MODEL_IN_SDCARD || CONFIG_HUMAN_FACE_FEAT_MODEL_IN_SDCARD
    ESP_ERROR_CHECK(bsp_sdcard_mount());
#endif

#ifdef BSP_BOARD_ESP32_S3_EYE
    ESP_ERROR_CHECK(bsp_leds_init());
    ESP_ERROR_CHECK(bsp_led_set(BSP_LED_GREEN, false));
#endif

#if CONFIG_IDF_TARGET_ESP32S3
    auto frame_cap = get_dvp_frame_cap_pipeline();
#elif CONFIG_IDF_TARGET_ESP32P4
    auto frame_cap = get_mipi_csi_frame_cap_pipeline();
#endif
    auto recognition_app = new WhoRecognitionAppLCD(frame_cap);
    // 没有 LCD 时：
    // auto recognition_app = new WhoRecognitionAppTerm(frame_cap);
    recognition_app->run();
}
```

> `WhoRecognitionAppLCD` 构造时已自动：创建 `HumanFaceDetect` + `HumanFaceRecognizer`、根据按键类型（物理 / LVGL）创建 `WhoRecognitionButton`、绑定结果回调、设置 face.db 路径。无需手动 `set_model`/`set_recognizer`。

### 3. 数据库路径（来自 Kconfig）

| Kconfig | face.db 路径 |
|---|---|
| `CONFIG_DB_FATFS_FLASH`（默认） | `CONFIG_SPIFLASH_MOUNT_POINT/face.db`（默认 `/spiflash/face.db`） |
| `CONFIG_SPIFFS` | `CONFIG_BSP_SPIFFS_MOUNT_POINT/face.db` |
| `CONFIG_DB_FATFS_SDCARD` | `CONFIG_BSP_SD_MOUNT_POINT/face.db` |

切换：`idf.py menuconfig` → Component config → esp-who: human_face_recognition → database file system。

### 4. 按键操作（人脸识别交互）

由 `WhoRecognitionButton` 触发 `WhoRecognitionCore` 的事件位 `RECOGNIZE` / `ENROLL` / `DELETE`：

| 板子 | 注册 enroll | 识别 recognize | 删除 delete |
|---|---|---|---|
| ESP32-S3-EYE | `UP` 键 | `PLAY` 键 | `DOWN` 键 |
| ESP32-S3-Korvo-2 | `VOL+` | `PLAY` | `VOL-` |
| ESP32-P4 Function EV Board | 触摸屏 "enroll" | 触摸屏 "recognize" | 触摸屏 "delete" |

识别结果（串口 / LCD label）：
- 命中：`id: <id>, sim: <相似度>`（取 `ret[0].id` / `ret[0].similarity`）
- 未命中：`who?`
- 注册成功：`id: <num_feats> enrolled.`；失败：`Failed to enroll.`
- 删除成功：`id: <num_feats+1> deleted.`；失败：`Failed to delete.`

> 单击触发：物理键 `BUTTON_SINGLE_CLICK`，触摸键 `LV_EVENT_CLICKED`。

### 5. 模型放 SD 卡（可选）

默认模型编进固件。若用 `partitions2.csv`（模型放 spiffs 分区）或 SD 卡，需对应 menuconfig 选 `CONFIG_HUMAN_FACE_DETECT_MODEL_IN_SDCARD` / `CONFIG_HUMAN_FACE_FEAT_MODEL_IN_SDCARD` 并在 `app_main` 挂载 sdcard（见上）。`partitions2.csv` 含 `human_face_det`（200K spiffs）与 `human_face_feat`（5000K spiffs）分区。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `recognizer is nullptr, please call set_recognizer() first.` | 直接用底层 `WhoRecognition` 没设 recognizer | 用 `WhoRecognitionAppLCD`（自动设）；或手动 `recog->set_recognizer(...)` |
| face.db 读写失败 | 文件系统未挂载 | 按 Kconfig 选对应挂载（fatfs_flash_mount / bsp_spiffs_mount / bsp_sdcard_mount） |
| 同人不同背景识别不出（已知问题） | CHANGELOG 1.0.0 列的 known issue | 多注册几帧；换背景光照 |
| P4 触摸键无反应 | LVGL 未刷新 / 没用 LVGL 按钮分支 | 确认 `BSP_BOARD_ESP32_P4_FUNCTION_EV_BOARD` 下走 `WhoRecognitionButtonLVGL` |
| `physical button not supported in BSP` | 选了 `PHYSICAL` 但 `BSP_CAPS_BUTTONS` 未定义 | P4 用 `LVGL` 类型；S3 用 `PHYSICAL` |

## 参考

- `examples/human_face_recognition/main/app_main.cpp`
- `examples/human_face_recognition/README.md`（按键映射表）
- `components/who_app/who_recognition_app/who_recognition_app_lcd.cpp`（构造逻辑）
- `components/who_recognition/who_recognition.cpp`（`RECOGNIZE`/`ENROLL`/`DELETE` 事件处理）
- `components/who_app/who_recognition_app/Kconfig`（数据库 choice）
- `resources/api_reference.md` §6 `WhoRecognition`
- `resources/state_machine.md` §4 识别事件流
