---
name: esp-iot-solution-skill
description: >-
  AI Skill for Espressif ESP-IoT-Solution firmware/component development. Used when users need to
  create, modify, or debug ESP-IoT-Solution based IoT projects, including device drivers
  (button, knob, led_indicator, i2c_bus, spi_bus), USB host/device (usb_stream, iot_usbh_cdc),
  sensors (sensor_hub, power_measure), motor (servo, BLDC), BLE profiles, displays/LCD, and
  power-save workflows. Built on top of ESP-IDF and targeting ESP32 / ESP32-S2 / ESP32-S3 /
  ESP32-C2 / ESP32-C3 / ESP32-C6 / ESP32-H2 / ESP32-P4 chips.
  Trigger words: "ESP-IoT-Solution", "esp-iot-solution", "iot_button", "led_indicator", "i2c_bus",
  "spi_bus", "iot_knob", "sensor_hub", "usb_stream", "iot_usbh_cdc", "iot_servo", "BLE", "蓝牙",
  "IoT 组件", "乐鑫", "Espressif", "ESP32", "ESP-IDF", "传感器", "按键", "灯效"
tags:
  - embedded
  - esp32
  - esp-idf
  - espressif
  - iot
  - bluetooth
  - ble
  - usb
  - sensors
  - display
  - firmware
license: Apache-2.0
compatibility: Build requires ESP-IDF v5.3+ (master branch) or v4.4-v5.3 (release/v2.0 branch); targets ESP32 family SoCs (ESP32, ESP32-S2, ESP32-S3, ESP32-C2, ESP32-C3, ESP32-C6, ESP32-H2, ESP32-P4); components are obtained via the ESP Component Registry (`idf.py add-dependency`)
metadata:
  author: Community
  version: "1.1.0"
---

# esp-iot-solution-skill

面向 Espressif **ESP-IoT-Solution** 的 AI 技能。ESP-IoT-Solution 是构建在 ESP-IDF 之上的 IoT 组件库与方案集合，提供传感器、显示、USB、音频、输入设备、电机、低功耗等大量设备驱动与框架组件。本技能提供场景化的 recipe、完整的 API 速查、配置参考与常见坑点，全部基于仓库真实文档（`docs/`）与组件源码（`components/`、`examples/`），绝不臆造 API。

## Core Principles

1. **绝不臆造 API** — 每个函数名、结构体、宏必须能在 `components/<name>/include/` 或 `docs/` 中找到；找不到就省略，不要凭记忆编造。
2. **ESP-IoT-Solution 是组件库，不是单一 SDK** — 每个组件独立迭代，依赖的 ESP-IDF 版本写在组件自己的 `idf_component.yml` 中。优先通过 ESP Component Registry 用 `idf.py add-dependency "espressif/<component>"` 添加。
3. **入口函数统一为 `app_main`** — ESP-IDF 应用入口是 `void app_main(void)`，不要写传统的 `int main(void)`。
4. **组件 API 风格统一为 `iot_*` / `xxx_new_*_device`** — 例如 `iot_button_new_gpio_device`、`i2c_bus_create`、`led_indicator_new_gpio_device`、`iot_knob_create`、`iot_sensor_create`。新代码统一用 `xxx_new_xxx_device` 工厂函数 + handle 句柄模式。
5. **事件驱动 + 回调注册** — 组件普遍提供 `xxx_register_cb(handle, EVENT, cb, usr_data)`。回调函数签名形如 `void cb(void *button_handle, void *usr_data)`。**回调里禁止阻塞操作**（如 `vTaskDelay`），否则会丢事件。
6. **`idf_component.yml` 决定依赖** — 项目的 `main/idf_component.yml` 声明所需组件；若拷贝示例到别处，必须删除其中的 `override_path` 配置，否则 CMake 阶段会失败。
7. **引脚 / 总线先初始化再使用** — I2C 设备要先 `i2c_bus_create` 建总线，再 `i2c_bus_device_create` 建设备；SPI 同理（`spi_bus_create` → `spi_bus_device_create`）。
8. **LED 指示灯使用 blink_step_t 步进表** — `led_indicator` 用 `blink_step_t const *blink_lists[]` 定义灯效序列，数组下标即 `blink_type`（优先级，0 最高）。
9. **低功耗要显式注册 power-save 回调** — button / knob 组件支持 `enable_power_save`，并需配合 `esp_pm_configure` + `esp_light_sleep_start`；GPIO 唤醒源在创建设备时配置。
10. **目标芯片决定可用外设** — 例如 `usb_stream` 仅支持 ESP32-S2/S3（USB Host OTG）；`touch_button` 仅 ESP32 / S3 / C6 / H2 等带 Touch 外设的芯片。引用前确认目标芯片支持。
11. **示例是最佳起点** — 复杂组件（LCD、USB、BLE、FOC 电机）应从 `examples/` 复制最接近的示例改写，而非从零手写初始化序列。

## When to Use

**适用：**
- 创建新的 ESP-IoT-Solution / ESP-IDF 固件项目
- 集成 ESP-IoT-Solution 组件（button、knob、led_indicator、i2c_bus、spi_bus、sensor_hub、usb_stream、iot_usbh_cdc、iot_servo 等）
- 配置按键事件、灯效序列、旋钮编码器、I2C/SPI 总线设备
- 实现 USB 主机（UVC/UAC 流、CDC 串口）或 USB 设备（UAC/UVC/UF2）
- 传感器数据采集与分发（sensor_hub）
- 电机控制（舵机、BLDC FOC）
- 低功耗 / light sleep 唤醒方案

**不适用：**
- 纯 ESP-IDF 原生 API 问题（不涉及 ESP-IoT-Solution 组件）—— 应查 ESP-IDF 文档
- 非 Espressif 芯片（STM32、CH57x、RP2040 等）
- 纯硬件 / 原理图 / PCB 设计

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下方某个场景时，**先读对应 recipe** —— 它包含完整调用链、分步说明、常见错误与可复制代码。

### 总线与通信

| recipe | 场景 |
|---|---|
| `recipes/i2c_bus.md` | I2C 总线初始化、设备创建、寄存器读写、扫描 |
| `recipes/spi_bus.md` | SPI 总线初始化、设备创建、字节/多字节/16/32 位传输 |

### 输入设备

| recipe | 场景 |
|---|---|
| `recipes/button_gpio.md` | GPIO 按键创建、事件回调注册、长短按、多击 |
| `recipes/button_power_save.md` | 按键 + light sleep 低功耗唤醒 |
| `recipes/knob.md` | 旋转编码器旋钮创建、左右旋事件、计数读取 |

### 输出与指示

| recipe | 场景 |
|---|---|
| `recipes/led_indicator_gpio.md` | GPIO LED 指示灯、blink_step 灯效表、呼吸/闪烁 |
| `recipes/led_indicator_ledc.md` | LEDC PWM LED 指示灯、亮度调节、呼吸灯 |
| `recipes/led_indicator_strips.md` | RGB LED / WS2812 灯带、HSV/RGB 彩色灯效、颜色渐变、Index 控制 |

### 传感器

| recipe | 场景 |
|---|---|
| `recipes/sensor_hub.md` | sensor_hub 创建传感器实例、事件回调、数据采集 |
| `recipes/power_measure.md` | 功率计量（BL0937 / BL0942 / INA236）初始化与读取 |

### USB

| recipe | 场景 |
|---|---|
| `recipes/usb_stream.md` | USB 主机 UVC/UAC 流（摄像头/麦克风/扬声器）配置 |
| `recipes/usbh_cdc.md` | USB 主机 CDC 串口驱动收发 |

### 电机

| recipe | 场景 |
|---|---|
| `recipes/servo.md` | LEDC 舵机角度控制初始化 |
| `recipes/foc_motor.md` | FOC 无刷电机（esp_simplefoc，开环/闭环速度，AS5600） |

### 蓝牙 (BLE)

| recipe | 场景 |
|---|---|
| `recipes/ble_conn_mgr.md` | `esp_ble_conn_mgr` 外设/中心、周期广播/同步、SPP、L2CAP CoC |
| `recipes/ble_profiles_ota.md` | BLE GATT profile：OTA 固件升级（`0x8018`）、健康体温计 HTP（`0x1809`） |
| `recipes/bthome.md` | BTHome V2 协议（对接 Home Assistant，加密广播/解析） |

### 触摸

| recipe | 场景 |
|---|---|
| `recipes/touch_button.md` | 触摸按键（touch_sensor_lowlevel + iot_button） |

---

## 组件版本与 IDF 兼容参考

| ESP-IoT-Solution | 依赖 ESP-IDF | 主要变化 | 支持状态 |
|:---:|:---:|:---:|:---:|
| master | >= v5.3 | 新芯片支持、新组件 | 新特性开发 |
| release/v2.0 | <= v5.3, >= v4.4 | 组件管理器支持 | Bugfix 维护 |
| release/v1.1 | v4.0.1 | IDF 升级、移除弃用代码 | 归档 |
| release/v1.0 | v3.2.2 | 历史版本 | 归档 |

> 各组件实际依赖的 IDF 版本以组件自身 `idf_component.yml` 为准。

---

## 关键组件 / API 速览

| 组件 | 主要头文件 | 关键 API（创建） |
|---|---|---|
| button | `iot_button.h`, `button_gpio.h`, `button_adc.h`, `button_matrix.h`, `button_rtc.h`, `button_types.h` | `iot_button_new_gpio_device` / `iot_button_new_adc_device` / `iot_button_new_matrix_device` / `iot_button_new_rtc_device` |
| touch_button（与 iot_button 集成） | `components/touch/touch_button/touch_button.h`、`components/touch/touch_button_sensor/include/touch_button_sensor.h` | `iot_button_new_touch_button_device`（触摸接入 iot_button 事件体系）；`touch_button_sensor_create`（独立触摸 FSM） |
| knob | `iot_knob.h` | `iot_knob_create` |
| led_indicator | `led_indicator.h`, `led_indicator_gpio.h`, `led_indicator_ledc.h`, `led_indicator_rgb.h`, `led_indicator_strips.h` | `led_indicator_new_gpio_device` / `led_indicator_new_ledc_device` / `led_indicator_new_rgb_device` / `led_indicator_new_strips_device` |
| i2c_bus | `i2c_bus.h` | `i2c_bus_create` → `i2c_bus_device_create` |
| spi_bus | `spi_bus.h` | `spi_bus_create` → `spi_bus_device_create` |
| sensor_hub | `iot_sensor_hub.h` | `iot_sensor_create` → `iot_sensor_start` |
| power_measure | `power_measure.h`（含 bl0937/bl0942/ina236） | `power_measure_new_bl0937_device` / `power_measure_new_bl0942_device` / `power_measure_new_ina236_device` |
| usb_stream | `usb_stream.h` | `uvc_streaming_config` / `uac_streaming_config` → `usb_streaming_start` |
| iot_usbh_cdc | `iot_usbh_cdc.h` | `usbh_cdc_driver_install` → `usbh_cdc_port_open` |
| servo | `iot_servo.h` | `iot_servo_init` → `iot_servo_write_angle` |
| touch_button | `touch_button.h`, `touch_sensor_lowlevel.h` | `touch_sensor_lowlevel_create` + `iot_button_new_touch_button_device` |
| iot_eth | `iot_eth.h` | `iot_eth_install` → `iot_eth_start` |
| ble_conn_mgr | `esp_ble_conn_mgr.h` | `esp_ble_conn_init` → `esp_ble_conn_start`（+ `esp_ble_conn_add_svc`） |
| ble_profiles (ota_raw) | `esp_ble_ota_raw.h` | `esp_ble_ota_raw_init` → `esp_ble_ota_raw_recv_fw_data_callback` |
| ble_profiles (htp) | `esp_htp.h` | `esp_ble_htp_init` → 事件 `BLE_HTP_EVENTS` |
| bthome_v2 | `bthome_v2.h` | `bthome_create` → `bthome_parse_adv_data` / `bthome_make_adv_data` |
| esp_simplefoc | `esp_simplefoc.h`（C++） | `BLDCDriver3PWM::init` + `BLDCMotor::init` / `initFOC` → `loopFOC` |
| led_indicator (rgb/strips) | `led_indicator_rgb.h`、`led_indicator_strips.h`、`led_convert.h` | `led_indicator_new_rgb_device` / `led_indicator_new_strips_device` |

---

## Critical Pitfalls (Must Read)

这些是最常见的错误。违反任意一条都会导致固件无法正常工作。

### 1. 应用入口必须是 app_main，不是 main

```c
// ❌ 错误 —— ESP-IDF 不使用 int main
int main(void) {
    init_all();
    while (1) { /* ... */ }
    return 0;
}

// ✅ 正确 —— ESP-IDF 入口签名
void app_main(void)
{
    button_init(BOOT_BUTTON_NUM);
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}
```

### 2. button 创建用工厂函数 iot_button_new_gpio_device，不是旧的 iot_button_create

```c
// ❌ 错误 —— 旧 API（v2.x 之前），新版本 button_config_t 不含 gpio_num
button_config_t cfg = {
    .gpio_num = 0,        // 该字段在新版 button_config_t 中不存在
    .active_level = 0,
};
iot_button_create(&cfg, &btn);   // 编译失败

// ✅ 正确 —— 分离 button_config_t 与 button_gpio_config_t
button_config_t btn_cfg = {0};
button_gpio_config_t gpio_cfg = {
    .gpio_num = 0,
    .active_level = 0,           // BOOT 按键通常按下为低电平
    .enable_power_save = false,
};
iot_button_new_gpio_device(&btn_cfg, &gpio_cfg, &btn);
```

### 3. 回调函数中禁止阻塞操作

```c
// ❌ 错误 —— 回调里延时，会丢事件并阻塞 button 扫描定时器
static void button_cb(void *arg, void *data)
{
    vTaskDelay(pdMS_TO_TICKS(50));   // 禁止！
    ESP_LOGI(TAG, "pressed");
}

// ✅ 正确 —— 回调里只做轻量操作；耗时任务用 xTaskCreate 或队列转交
static void button_cb(void *arg, void *data)
{
    iot_button_print_event((button_handle_t)arg);
}
```

### 4. I2C 设备要先建总线再建设备

```c
// ❌ 错误 —— 未创建总线直接用设备句柄
i2c_bus_device_handle_t dev = i2c_bus_device_create(NULL, 0x44, 0);
i2c_bus_read_byte(dev, 0x00, &val);   // dev 为 NULL，失败

// ✅ 正确 —— 先建总线（含 sda/scl/clk），再在总线上建设备
i2c_config_t conf = {
    .mode = I2C_MODE_MASTER,
    .sda_io_num = 8, .scl_io_num = 9,
    .sda_pullup_en = true, .scl_pullup_en = true,
    .master.clk_speed = 100000,
};
i2c_bus_handle_t bus = i2c_bus_create(I2C_NUM_0, &conf);
i2c_bus_device_handle_t dev = i2c_bus_device_create(bus, 0x44, 0);
i2c_bus_read_byte(dev, 0x00, &val);
```

### 5. led_indicator 的 blink_lists 数组下标即优先级，且最后一项要 NULL

```c
// ❌ 错误 —— 数组未以 NULL 结尾，或下标与枚举不连续
blink_step_t const *led_mode[] = {
    [BLINK_DOUBLE] = double_blink,
    [BLINK_FAST] = fast_blink,        // 缺 BLINK_TRIPLE、BLINK_SLOW，blink_list_num 与枚举值不匹配
};

// ✅ 正确 —— 下标连续 0..BLINK_MAX-1，BLINK_MAX 位置为 NULL
enum { BLINK_DOUBLE = 0, BLINK_TRIPLE, BLINK_SLOW, BLINK_FAST, BLINK_MAX };
blink_step_t const *led_mode[] = {
    [BLINK_DOUBLE] = double_blink,
    [BLINK_TRIPLE] = triple_blink,
    [BLINK_SLOW]   = slow_blink,
    [BLINK_FAST]   = fast_blink,
    [BLINK_MAX]    = NULL,
};
// led_indicator_config_t.blink_list_num = BLINK_MAX
```

### 6. iot_button_register_cb 第三个参数 event_args 不能省略语义

```c
// ❌ 错误 —— 把回调指针放到第三个参数位置
iot_button_register_cb(btn, BUTTON_SINGLE_CLICK, my_cb, NULL);

// ✅ 正确 —— 签名为 (handle, event, event_args*, cb, usr_data)
iot_button_register_cb(btn, BUTTON_SINGLE_CLICK, NULL, my_cb, NULL);
// 注册多击时 event_args 才非空：
button_event_args_t args = { .multiple_clicks.clicks = 3 };
iot_button_register_cb(btn, BUTTON_MULTIPLE_CLICK, &args, triple_cb, NULL);
```

### 7. 拷贝示例后必须删除 idf_component.yml 里的 override_path

```yaml
# ❌ 错误 —— 拷贝示例到别处后仍保留 override_path，CMake 找不到原仓库路径
dependencies:
  espressif/button:
    override_path: ../../../components/button

# ✅ 正确 —— 删除 override_path，改用 Component Registry 拉取
dependencies:
  espressif/button: "^3.2.0"
# 或在工程根目录执行：idf.py add-dependency "espressif/button"
```

### 8. knob 回调里的 usr_data 用作事件类型而非随便传

```c
// ❌ 错误 —— 回调里凭空假设 data 含按键状态
static void knob_cb(void *arg, void *data) {
    ESP_LOGI(TAG, "event %d", (int)data);   // data 来自注册时的 usr_data
}

// ✅ 正确 —— 注册时把 KNOB_LEFT/KNOB_RIGHT 作为 usr_data 传入，回调里强转
iot_knob_register_cb(knob, KNOB_LEFT,  knob_cb, (void *)KNOB_LEFT);
iot_knob_register_cb(knob, KNOB_RIGHT, knob_cb, (void *)KNOB_RIGHT);

static void knob_cb(void *arg, void *data) {
    ESP_LOGI(TAG, "event %d, count %d",
             (int)data, iot_knob_get_count_value((knob_handle_t)arg));
}
```

### 9. servo 的 channels 数组要按 channel_number 填满

```c
// ❌ 错误 —— channel_number 与 channels 数组实际项数不一致
servo_config_t cfg = {
    .channel_number = 2,
    .channels = {
        .servo_pin = { 2 },        // 只填了 1 个，但声明 2 通道
        .ch = { LEDC_CHANNEL_0 },
    },
};

// ✅ 正确 —— channel_number 与数组项数一致
servo_config_t cfg = {
    .max_angle = 180,
    .min_width_us = 500,
    .max_width_us = 2500,
    .freq = 50,
    .timer_number = LEDC_TIMER_0,
    .channels = {
        .servo_pin = { 2 },
        .ch = { LEDC_CHANNEL_0 },
    },
    .channel_number = 1,
};
iot_servo_init(LEDC_LOW_SPEED_MODE, &cfg);
```

### 10. usb_stream 仅支持 ESP32-S2/S3，且 UVC 摄像头须兼容 USB1.1 全速 + MJPEG

```c
// ❌ 错误 —— 在 ESP32（无 USB OTG）上启用 usb_stream，链接/运行失败
// idf.py set-target esp32   ← usb_stream 不支持

// ✅ 正确 —— 目标设为 ESP32-S2 或 ESP32-S3
// idf.py set-target esp32s3
uvc_config_t uvc_config = {
    .frame_width  = 320,
    .frame_height = 240,
    .frame_interval = FPS2INTERVAL(15),   // 摄像头须支持 MJPEG 输出
    // ... buffers / callback
};
uvc_streaming_config(&uvc_config);
usb_streaming_start();
```

### 11. touch_button_sensor 须周期调 handle_events，且用真实 API

```c
// ❌ 错误 —— 用了不存在的 lowlevel 接口名（凭记忆臆造）
touch_sensor_lowlevel_create(&low);     // 该函数名在组件里不存在
iot_button_new_touch_button_device(&btn_cfg, &touch_cfg, &btn);
touch_sensor_lowlevel_start();

// ✅ 正确 —— 用仓库公开文档化的 touch_button_sensor API，
//          且 FSM 须周期调用 handle_events 才会出回调
touch_button_config_t cfg = {
    .channel_num = sizeof(ch_list)/sizeof(ch_list[0]),
    .channel_list = ch_list,
    .channel_threshold = thr_list,
    .debounce_times = 2,
    .skip_lowlevel_init = false,
};
touch_button_handle_t h = NULL;
touch_button_sensor_create(&cfg, &h, touch_cb, NULL);
// 建任务每 10~30ms 调一次：
//   touch_button_sensor_handle_events(h);
```

### 12. IDF 版本宏决定 i2c_bus 的 backend（v5.3 起默认新 master 驱动）

```c
// ❌ 错误 —— 在 IDF >= 5.3 上直接 include "driver/i2c.h" 又同时用 i2c_bus，类型冲突
#include "driver/i2c.h"
#include "i2c_bus.h"   // i2c_bus 在 5.3+ 默认走 i2c_master.h，i2c_config_t 定义冲突

// ✅ 正确 —— 只 include "i2c_bus.h"，让组件按 CONFIG_I2C_BUS_BACKWARD_CONFIG 自动选 backend
#include "i2c_bus.h"
// 若必须用旧驱动，在 menuconfig 启用 CONFIG_I2C_BUS_BACKWARD_CONFIG=y
```

### 13. led_indicator_set_brightness 仅对支持亮度调节的硬件（LEDC/RGB/Strips）有效

```c
// ❌ 错误 —— 对纯 GPIO 指示灯调亮度，无效果（GPIO 只有 0/1）
led_indicator_new_gpio_device(&cfg, &gpio_cfg, &handle);
led_indicator_set_brightness(handle, 128);   // GPIO 模式下无效

// ✅ 正确 —— 调亮度用 LEDC / RGB / Strips 后端
led_indicator_ledc_config_t ledc_cfg = { .gpio_num = 2, .channel = LEDC_CHANNEL_0, /* ... */ };
led_indicator_new_ledc_device(&cfg, &ledc_cfg, &handle);
led_indicator_set_brightness(handle, 128);
```

---

## Execution Workflow

| 步骤 | 名称 | 说明 |
|---|---|---|
| 1 | 明确需求 | 确定目标芯片、所需组件、是否低功耗、是否需要 USB/显示 |
| 2 | 选 recipe | 匹配 `recipes/` 中的场景；命中则按其调用链实现 |
| 3 | 查 API | 未覆盖的 API 查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | 加依赖 | 在工程根目录 `idf.py add-dependency "espressif/<component>"`，或编辑 `main/idf_component.yml` |
| 5 | 设目标 | `idf.py set-target esp32s3`（或 esp32 / esp32c3 等） |
| 6 | 配置 | `idf.py menuconfig` 按需调整组件 Kconfig（如 `BUTTON_LONG_PRESS_TIME_MS`） |
| 7 | 写代码 | 入口为 `app_main`；总线先建、设备后建；回调里不阻塞 |
| 8 | 编译烧录 | `idf.py build` → `idf.py -p <PORT> flash monitor` |
| 9 | 验证 | 串口 monitor 查日志；外设用万用表/示波器/手机 BLE App 验证 |

### 步骤 7 详——从示例起步

复杂组件（LCD、USB、BLE、FOC 电机、camera）应优先复制最接近的示例改写：

- I2C 总线 / 功率计量 → `examples/sensors/power_measure/`
- 按键低功耗 → `examples/get-started/button_power_save/`
- 旋钮低功耗 → `examples/get-started/knob_power_save/`
- LED 指示灯（GPIO/LEDC/RGB/WS2812） → `examples/indicator/`
- sensor_hub → `examples/sensors/sensor_hub_monitor/`
- USB 主机 CDC → `examples/usb/host/usb_cdc_basic/`
- USB UVC/UAC 流 → `examples/usb/host/usb_camera_mic_spk/`、`usb_audio_player/`
- 舵机 → `examples/motor/servo_control/`
- 触摸按键 → `examples/touch/touch_button/`
- LCD QSPI → `examples/display/lcd/qspi_without_ram/`

复制后**务必删除 `main/idf_component.yml` 中的 `override_path`**，否则路径错误导致 CMake 失败。

---

## Failure Strategies

| 情况 | 处理 |
|---|---|
| API 在 `resources/api_reference.md` 中查不到 | 停下，告知用户该 API 不在已知组件范围内，需查仓库源码或组件 CHANGELOG |
| 组件未拉取到 | 执行 `idf.py add-dependency "espressif/<component>"`；或确认 `idf_component.yml` 未残留 `override_path` |
| 编译报 `undefined reference` | 检查 `idf_component.yml` 是否声明依赖；检查 `REQUIRES`/CMakeLists |
| IDF 版本不兼容 | 查组件 `idf_component.yml` 的 `idf_version` 约束；必要时切 release/v2.0 分支 |
| USB 组件报错 | 确认目标为 ESP32-S2/S3（USB OTG）；`usb_stream` 摄像头须兼容 USB1.1 + MJPEG |
| 回调事件丢失 | 检查回调内是否有阻塞调用；调大 `BUTTON_PERIOD_TIME_MS` / 提高任务优先级 |
| 引脚冲突 | 查芯片 GPIO 矩阵，避开 strapping / flash 占用引脚；Touch 引脚随芯片而异 |

## References

- 场景 recipe → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置参考 → `resources/config_reference.md`
- 常见坑点合集 → `resources/pitfalls.md`
- 示例索引 → `resources/example_list.md`
- 仓库在线文档（中文）：https://docs.espressif.com/projects/esp-iot-solution/zh_CN
- 仓库在线文档（英文）：https://docs.espressif.com/projects/esp-iot-solution/en
- ESP Component Registry：https://components.espressif.com/
