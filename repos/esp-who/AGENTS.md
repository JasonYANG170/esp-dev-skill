# AGENTS.md — 补充约定

> 核心规则、配方索引、陷阱、执行流程均在 `SKILL.md`。本文件只补充 `SKILL.md` 未覆盖的工程约定与工具指引，不重复内容。

## 项目背景

- **语言**：C++17（`app_main` 用 `extern "C"`），命名空间 `who::*`
- **目标**：ESP32-S3、ESP32-P4（基于 ESP-DL 的图像 AI SoC）
- **构建系统**：ESP-IDF（release/v5.4 或 v5.5）+ idf.py + CMake
- **依赖栈**：ESP-DL（模型推理）、ESP-BSP（开发板支持）、esp32-camera（S3）/ esp_video（P4）、quirc（二维码）、LVGL（图形，可选）、usb_host_uvc（USB 摄像头）
- **仓库**：`esp-who` 本身不含模型源码，模型组件（`human_face_detect` 等）由 `idf_component.yml` + `dependencies.lock.*` 从 ESP Component Registry 拉取

## 文件命名约定

- 头文件：`*.hpp`（C++）；少数 C 接口用 `*.h`
- 组件目录：`components/who_<module>/`，类名 `Who<Module>`
- 示例入口固定 `main/app_main.cpp`，流水线构造 `main/frame_cap_pipeline.{cpp,hpp}`
- 开发板默认配置：`sdkconfig.bsp.<bsp_name>`（`_noglib` 后缀表示不链接图形库）
- 依赖锁：`dependencies.lock.<bsp>[.<detect_model>]`

## Include 模式

```cpp
// 用户代码常用聚合头
#include "who_cam.hpp"            // 按 target 自动 include WhoS3Cam / WhoP4Cam + WhoUVCCam
#include "who_frame_cap.hpp"      // WhoFrameCap + 节点
#include "who_detect.hpp"         // WhoDetect（一般经由 App 头引入）

// App 一键封装
#include "who_detect_app_lcd.hpp"        // 或 who_detect_app_term.hpp
#include "who_recognition_app_lcd.hpp"   // 或 _term
#include "who_qrcode_app_lcd.hpp"        // 或 _term

// 模型（由 ESP-DL 模型组件提供，需在 idf_component.yml 声明）
#include "human_face_detect.hpp"
#include "pedestrian_detect.hpp"
#include "cat_detect.hpp"
#include "dog_detect.hpp"
#include "human_face_recognition.hpp"    // HumanFaceRecognizer / HumanFaceFeat

// 文件系统（人脸识别数据库）
#include "who_spiflash_fatfs.hpp"
```

## 标准示例结构

```
examples/<name>/
├── CMakeLists.txt               # 顶层：EXTRA_COMPONENT_DIRS 指向 ../../components/* + 依赖锁
├── README.md
├── partitions.csv               # 分区表
├── sdkconfig.bsp.<bsp>          # 各 BSP 默认配置
├── dependencies.lock.<bsp>[.model]
└── main/
    ├── CMakeLists.txt           # idf_component_register + requires/optional_requires
    ├── idf_component.yml        # 组件依赖（bsp_ext.py 会动态改写）
    ├── app_main.cpp             # extern "C" void app_main(void)
    ├── frame_cap_pipeline.cpp   # 构造 WhoFrameCap 流水线
    └── frame_cap_pipeline.hpp
```

## 标准入口模式（`app_main`）

所有三个官方示例的 `app_main` 第一行相同：

```cpp
extern "C" void app_main(void)
{
    vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);   // 必须：高于子任务优先级 2
    // ... 按需挂载文件系统、关 LED
    auto frame_cap = get_xxx_frame_cap_pipeline();
    auto app = new who::app::WhoXxxApp(frame_cap);
    app->set_model(...);            // 仅 detect 类 App 需要（recognition/qrcode 内部已建）
    app->run();                     // 阻塞，内部启动 WhoYield2Idle + 各任务
}
```

关键点：
1. `vTaskPrioritySet` 提升到 5（子任务多为 2）。
2. `set_model` 在 `run()` 前（detect 类）。
3. `run()` 内部依次起 `WhoYield2Idle` → 采集节点 → LCD 显示 → 检测/识别。
4. 所有任务/节点构造与回调注册**必须**在 `run()` 之前完成（`register_task` 有断言）。

## 标准自定义任务循环模板

```cpp
class MyTask : public who::task::WhoTask {
public:
    MyTask() : who::task::WhoTask("MyTask") {}
private:
    void task() override {
        while (true) {
            EventBits_t bits = xEventGroupWaitBits(
                m_event_group, MY_EVT | TASK_PAUSE | TASK_STOP, pdTRUE, pdFALSE, portMAX_DELAY);
            if (bits & TASK_STOP) break;
            if (bits & TASK_PAUSE) {
                xEventGroupSetBits(m_event_group, TASK_PAUSED);
                EventBits_t p = xEventGroupWaitBits(
                    m_event_group, TASK_RESUME | TASK_STOP, pdTRUE, pdFALSE, portMAX_DELAY);
                if (p & TASK_STOP) break;
                continue;
            }
            // 处理 MY_EVT
        }
        xEventGroupSetBits(m_event_group, TASK_STOPPED);
        vTaskDelete(NULL);
    }
};
```

- 自定义事件位从 `TASK_EVENT_BIT_LAST`（`1<<5`）之后起。
- 退出前必须置 `TASK_STOPPED` 并 `vTaskDelete(NULL)`。

## 构建流程

1. **设环境变量**：`export IDF_EXTRA_ACTIONS_PATH=/path_to_esp-who/tools/`（让 `bsp_ext.py` 生效）
2. **进示例目录**：`cd examples/<name>`
3. **设 target + BSP**：`idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.<bsp> set-target <esp32s3|esp32p4>`
4. **（object_detect）加模型**：`-DDETECT_MODEL=<model>`
5. **（可选）menuconfig**
6. **编译烧录监视**：`idf.py -p PORT flash monitor`

> PowerShell 下 `-D` 参数值加引号。BSP 与 target 必须按 `BSP2IDF_TARGET` 对应（见 `config_reference.md` §2）。

## 代码生成 Checklist

新写/改写一个 ESP-WHO 应用时逐项核对：

- [ ] `app_main` 第一行 `vTaskPrioritySet(xTaskGetCurrentTaskHandle(), 5);`
- [ ] 摄像头类与 target 匹配（S3→`WhoS3Cam`，P4→`WhoP4Cam`，USB→`WhoUVCCam`+`WhoDecodeNode`）
- [ ] `fb_count` ≥ `MODEL_TIME + 3`（LCD）或 `MODEL_TIME + 2`（term）
- [ ] 流水线用 `WhoFrameCap::add_node<T>(...)` 串联，不手动 new+注册
- [ ] P4 用 `WhoPPAResizeNode` 时把缩放前节点作为 `WhoDetectAppLCD` 第三参（显示源）
- [ ] detect 类 App 在 `run()` 前 `set_model(...)`
- [ ] `run()` 之前完成所有任务注册与回调绑定（断言约束）
- [ ] 任务 `xCoreID` 为 `0` 或 `1`，不传 `tskNO_AFFINITY`
- [ ] 人脸识别按 `CONFIG_DB_*` 挂载对应文件系统
- [ ] term 模式配 `*_noglib` BSP；LCD 模式配无后缀 BSP
- [ ] 自定义事件位从 `TASK_EVENT_BIT_LAST` 之后起
- [ ] `idf_component.yml` 含所需模型组件（`-DDETECT_MODEL=` 或人脸识别的 `human_face_recognition`）
- [ ] 若模型放 SD 卡，`partitions.csv` 含对应分区并在 `app_main` 挂载 sdcard

## Do Not Modify

- 仓库 `components/` 下各组件源码（应通过继承/override 扩展，而非改源）
- `tools/bsp_ext.py`（BSP↔target 校验逻辑）
- `SKILL.md` frontmatter（Skill 元数据）
- `resources/` 文档（API/配置依据）

## 外部资源（不在本仓库）

- 模型组件文档：`https://components.espressif.com/components/espressif/<model>`
- ESP-DL：`https://github.com/espressif/esp-dl`
- ESP-DETECTION：`https://github.com/espressif/esp-detection`
- ESP-BSP：`https://github.com/espressif/esp-bsp`
- esp32-camera：`https://github.com/espressif/esp32-camera`
- esp-video-components：`https://github.com/espressif/esp-video-components`
