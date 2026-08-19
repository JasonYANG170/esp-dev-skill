# 触摸按键（touch_button_sensor / touch_button）

> **适用摘要**: 使用 ESP-IoT-Solution 的触摸按键组件在 ESP32 / ESP32-S2 / ESP32-S3（ESP32-P4 支持多频采样）上实现电容触摸检测。本 recipe 以仓库公开文档化的 `touch_button_sensor` 组件为主：配置通道列表与阈值、创建实例、注册触摸状态回调、周期性 `touch_button_sensor_handle_events` 处理事件。另提供与 `iot_button` 框架集成的 `iot_button_new_touch_button_device` 接口。注意：ESP32/S2/S3 的触摸抗干扰能力有限，仅供测试/演示，不建议量产通过 EMS 测试；本组件需 IDF >= v5.3。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-iot-solution/resources/`, source/examples in `repos/esp-iot-solution/`, and this recipe path `repos/esp-iot-solution/recipes/touch_button.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "触摸按键"
- "电容触摸"
- "touch_button"
- "touch_button_sensor"
- "iot_button_new_touch_button_device"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | ESP32、ESP32-S2、ESP32-S3（ESP32-P4 多频采样）；须带 Touch 外设 |
| IDF 环境 | ESP-IDF >= v5.3 |
| 组件依赖 | `espressif/touch_button_sensor`（独立组件）或 `espressif/touch_button`（iot_button 集成） |
| 量产注意 | ESP32/S2/S3 触摸抗干扰有限，建议仅测试/演示 |
| 参考示例 | `examples/touch/touch_button_sensor/`、`examples/touch/touch_button/` |

## 分步说明

### 方式 A：touch_button_sensor（独立组件，本仓库文档化）

#### 1. 添加依赖

```bash
idf.py add-dependency "espressif/touch_button_sensor"
```

#### 2. 定义通道与阈值

通道号随芯片而异（见下方速查）。阈值 `channel_threshold` 为 0.0~1.0 的相对变化率（如 `0.02` 表示 2%）。`debounce_times` 为连续确认次数。

```c
#include "touch_button_sensor.h"

// 以 ESP32-S3 / S2 为例（GPIO 8/10/12）
uint32_t channel_list[]      = {8, 10, 12};
float     channel_threshold[] = {0.02, 0.02, 0.02};
```

> 各芯片典型通道号（取自示例）：ESP32-P4 → `{7,9,11}`（GPIO 9/11/13，阈值 0.01）；ESP32-S3/S2 → `{8,10,12}`（阈值 0.02）；ESP32 → `{8,6,4}`（GPIO 33/14/13，阈值 0.01）。

#### 3. 创建实例并注册回调

```c
static const char *TAG = "touch";

static void touch_state_callback(touch_button_handle_t handle,
                                 uint32_t channel, touch_state_t state, void *user_data)
{
    if (state == TOUCH_STATE_ACTIVE) {
        ESP_LOGI(TAG, "Channel %lu is Active", (unsigned long)channel);
    } else if (state == TOUCH_STATE_INACTIVE) {
        ESP_LOGI(TAG, "Channel %lu is Inactive", (unsigned long)channel);
    }
}

void touch_init(void)
{
    touch_button_config_t config = {
        .channel_num        = sizeof(channel_list) / sizeof(channel_list[0]),
        .channel_list       = channel_list,
        .channel_threshold  = channel_threshold,
        .debounce_times     = 2,
        .skip_lowlevel_init = false,    // 由本组件初始化底层 touch
    };
    touch_button_handle_t handle = NULL;
    ESP_ERROR_CHECK(touch_button_sensor_create(&config, &handle, touch_state_callback, NULL));
}
```

#### 4. 周期性处理事件（FSM）

`touch_button_sensor` 基于有限状态机，需周期调用 `touch_button_sensor_handle_events` 触发状态判断与回调：

```c
static void touch_button_task(void *pv)
{
    touch_button_handle_t handle = (touch_button_handle_t)pv;
    while (1) {
        touch_button_sensor_handle_events(handle);
        vTaskDelay(pdMS_TO_TICKS(20));
    }
}
// app_main 中：xTaskCreate(touch_button_task, "touch", 4096, handle, 5, NULL);
```

#### 5. 查询状态 / 数据（可选）

```c
touch_state_t st = TOUCH_STATE_INACTIVE;
touch_button_sensor_get_state(handle, 8, &st);

uint32_t raw = 0;
touch_button_sensor_get_data(handle, 8, 0, &raw);   // channel_alt 为频率实例索引
```

#### 6. 删除

```c
touch_button_sensor_delete(handle);
```

### 方式 B：与 iot_button 框架集成（touch_button 组件）

若想把触摸接入 `iot_button` 的统一事件体系（单击/长按等），用 `iot_button_new_touch_button_device`，配置结构体为 `button_touch_config_t`：

```c
#include "iot_button.h"
#include "touch_button.h"

button_config_t btn_cfg = {0};
button_touch_config_t touch_cfg = {
    .touch_channel     = 8,       // 单通道
    .channel_threshold = 0.02f,   // 0.0~1.0
    .skip_lowlevel_init = false,
};
button_handle_t btn = NULL;
iot_button_new_touch_button_device(&btn_cfg, &touch_cfg, &btn);

iot_button_register_cb(btn, BUTTON_PRESS_DOWN, NULL, press_cb, NULL);
iot_button_register_cb(btn, BUTTON_PRESS_UP,   NULL, press_cb, NULL);
```

> `touch_button` 组件底层依赖 `touch_button_sensor`（含 `touch_sensor_fsm`、`touch_sensor_lowlevel`）。底层是否由组件自动初始化取决于 `skip_lowlevel_init`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 回调从不触发 | 未周期调用 `touch_button_sensor_handle_events` | 建一个任务每 10~30ms 调用一次 |
| 通道号无效 | 用了芯片不支持的 touch 通道 | 按芯片选通道（见上方速查）；查芯片 datasheet |
| 误触发 / 不灵敏 | 阈值不合理或硬件走线差 | 调 `channel_threshold`（更小更灵敏、更大更稳）；PCB 走线短、加隔离 |
| `touch_button_sensor_create` 失败 | IDF 版本 < 5.3 或芯片无 Touch 外设 | 升级 IDF 至 >= v5.3；换 ESP32/S2/S3/P4 |
| EMS 测试不过 | ESP32/S2/S3 触摸抗干扰有限 | 仅供测试/演示；量产改用专用触摸 IC 或其他方案 |
| 多通道无反应 | `channel_num` 与数组长度不符 | `channel_num = sizeof(list)/sizeof(list[0])` |

## 参考

- 组件头文件：`components/touch/touch_button_sensor/include/touch_button_sensor.h`；`components/touch/touch_button/touch_button.h`
- 在线文档：`docs/en/touch/touch_button_sensor.rst`
- 真实示例：`examples/touch/touch_button_sensor/main/`、`examples/touch/touch_button/`
