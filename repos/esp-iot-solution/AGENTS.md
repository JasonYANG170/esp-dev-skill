# AGENTS.md — 补充代理指南

> 核心规则、组件速览、坑点、recipe 索引、执行工作流都在 `SKILL.md`。
> 本文件**仅**补充 `SKILL.md` 未覆盖的工程约定与工具链细节，不重复内容。

## 项目上下文

- **语言**：C（ESP-IDF C 框架；少量组件含 C++ 接口）
- **目标**：Espressif ESP32 系列 SoC（ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C2 / ESP32-C3 / ESP32-C6 / ESP32-H2 / ESP32-P4）
- **构建系统**：CMake + ESP-IDF 工具链（`idf.py`）
- **组件分发**：ESP Component Registry（`idf.py add-dependency "espressif/<name>"`）

## 文件命名约定

- 示例入口：`main/<example_name>_main.c` 或 `main/main.c`，函数 `app_main`
- 组件头文件：`components/<name>/include/<name>.h`（如 `components/button/include/iot_button.h`）
- 组件依赖清单：每个组件根目录的 `idf_component.yml`
- 组件配置：每个组件根目录的 `Kconfig`（通过 `idf.py menuconfig` 访问）

## Include 模式

```c
// 通用基础
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_err.h"

// 各组件（按需）
#include "iot_button.h"          // button 核心
#include "button_gpio.h"         // GPIO 后端
#include "button_adc.h"          // ADC 后端
#include "button_matrix.h"       // 矩阵键盘后端
#include "button_rtc.h"          // RTC GPIO 按键后端
#include "iot_knob.h"            // 旋钮
#include "led_indicator.h"       // LED 指示灯核心
#include "led_indicator_gpio.h"  // LED GPIO 后端
#include "led_indicator_ledc.h"  // LED LEDC 后端
#include "i2c_bus.h"             // I2C 总线（自动适配新/旧 IDF 驱动）
#include "spi_bus.h"             // SPI 总线
#include "iot_sensor_hub.h"      // sensor_hub
#include "power_measure.h"       // 功率计量
#include "usb_stream.h"          // USB UVC/UAC 流
#include "iot_usbh_cdc.h"        // USB 主机 CDC
#include "iot_servo.h"           // 舵机
#include "touch_button.h"        // 触摸按键
#include "touch_sensor_lowlevel.h"
```

## 标准 ESP-IoT-Solution 工程结构

```
my_project/
├── CMakeLists.txt              # 顶层 cmake（include IDF）
├── sdkconfig                   # menuconfig 生成
├── main/
│   ├── CMakeLists.txt
│   ├── idf_component.yml       # 声明依赖（idf.py add-dependency 自动生成）
│   └── main.c                  # 含 app_main
└── managed_components/         # idf.py 自动拉取的组件（勿手改）
    └── espressif__button/
```

> 拷贝 `examples/` 中的示例到独立目录后，必须删除 `main/idf_component.yml` 中的 `override_path` 配置，否则 CMake 会找不到原仓库相对路径。

## 标准入口与初始化模式

```c
static const char *TAG = "app";

void app_main(void)
{
    /* 1. 按需初始化电源管理 / 总线 */
    /* 2. 创建外设实例（new_xxx_device / xxx_create）*/
    /* 3. 注册回调（xxx_register_cb）*/
    /* 4. 启动（xxx_start / sensor start 等，按组件而定）*/
    /* 5. app_main 退出后，FreeRTOS idle 任务接管；或自建事件循环 */
    ESP_LOGI(TAG, "started");
}
```

要点：
- `app_main` 运行在 IDLE 优先级的 FreeRTOS 任务中，**返回后该任务被删除**——不要把主循环写在 `app_main` 里依赖它常驻，要么显式 `while (1) vTaskDelay(...)`，要么把工作放到自建任务。
- 组件普遍后台跑自己的定时器/任务（button 的扫描定时器、sensor_hub 的采集任务），无需用户手写轮询。

## 构建工作流

1. 设置 IDF 环境（`export.bat` / `. ./export.sh`）
2. `idf.py set-target esp32s3`（首次或换芯片）
3. `idf.py add-dependency "espressif/button"` 添加组件依赖
4. `idf.py menuconfig`（按组件 Kconfig 调整，如 `BUTTON_LONG_PRESS_TIME_MS`）
5. `idf.py build`
6. `idf.py -p COMx flash monitor`（`Ctrl-]` 退出 monitor）
7. 日志级别可在 menuconfig → Component config → Log levels 调整

## 代码生成清单

生成或修改 ESP-IoT-Solution 固件代码时逐项核对：

- [ ] 入口签名为 `void app_main(void)`，不是 `int main`
- [ ] 用工厂函数 `iot_button_new_xxx_device` / `led_indicator_new_xxx_device`，而非旧 `xxx_create` 直接传 GPIO
- [ ] I2C/SPI 总线已 `xxx_bus_create`，再 `xxx_bus_device_create`
- [ ] 回调函数中**无阻塞调用**（无 `vTaskDelay`、无长 `printf` 链）
- [ ] `idf_component.yml` 已声明所有用到的组件；若拷贝自示例，已删除 `override_path`
- [ ] 目标芯片支持所选外设（`usb_stream` → S2/S3；`touch_button` → 带 Touch 的芯片）
- [ ] IDF 版本满足组件 `idf_component.yml` 的 `idf_version` 约束
- [ ] `led_indicator` 的 `blink_lists` 数组下标连续且末尾 `NULL`，`blink_list_num` 与枚举数一致
- [ ] 检查返回值（`ESP_ERROR_CHECK(ret)` 或显式判断），关键 API 失败要 `goto` 清理

## 日志约定

```c
static const char *TAG = "my_module";
ESP_LOGI(TAG, "value=%d", v);   // 组件自身也用 ESP_LOGx，TAG 通常��组件名
```
组件日志级别统一在 menuconfig → Component config → Log levels 设置（如 `Button (3000)`、`I2C_BUS`）。

## Do Not Modify（勿改）

- `managed_components/` —— 由 Component Registry 自动拉取，改了会被覆盖；要定制依赖版本就改 `idf_component.yml`。
- 仓库源码 `components/` 与 `examples/` —— 仅作为只读参考；如需贡献修改，走仓库 `CONTRIBUTING.rst` 流程。
- 本技能的 `SKILL.md` frontmatter —— 技能元数据。
