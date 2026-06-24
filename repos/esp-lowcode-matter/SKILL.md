---
name: esp-lowcode-matter-skill
description: >-
  AI Skill for developing Matter-connected products with Espressif's esp-lowcode-matter
  framework. Used when users need to create, customise, or debug LowCode Matter products
  (lights, sockets, sensors, thermostat, occupancy) running on the ESP32-C6 LP core using the
  Arduino-like setup()/loop() + low_code event/feature model.
  Trigger words: "esp-lowcode-matter", "LowCode", "low code", "ESP LowCode", "Matter", "esp32-c6",
  "ESP32-C6", "LP core", "setup loop", "low_code_feature_update_to_system", "嘉立创Matter",
  "低代码 Matter", "Matter 设备", "Matter 产品"
tags:
  - embedded
  - matter
  - esp32
  - esp32-c6
  - low-code
  - espressif
  - smart-home
  - lp-core
  - firmware
  - iot
license: Apache-2.0
compatibility: ESP32-C6 only (HP+LP asymmetric cores); build with ESP-IDF v5.3 + ESP-AMP; flash via esptool.py
metadata:
  author: Community
  version: "1.1.0"
---

# esp-lowcode-matter-skill

面向 Espressif **esp-lowcode-matter** 框架的 AI Skill。该框架在 ESP32-C6 上以非对称双核（HP Core + LP Core）方式运行 Matter：HP Core 跑预编译好的 Wi-Fi/BLE/Matter 协议栈镜像（位于 `pre_built_binaries/`），开发者只需在 **LP Core** 上按 Arduino 风格的 `setup()/loop()` 模型编写产品固件，通过 `low_code_*` 的「事件（event）/ 特性（feature）」消息与 HP Core 通信。本 Skill 提供场景化 recipes、真实 API 参考、配置说明与高频陷阱，全部内容均来自仓库源码与文档，绝不臆造 API。

## Core Principles

1. **只在 LP Core 写产品代码** — HP Core 跑预编译镜像，开发者只写 `products/<name>/main/app_main.cpp` 与 `app_driver.cpp`；HP/LP 通过 ESP-AMP 消息通信，不要尝试直接调用 Wi-Fi/BLE/Matter 栈 API。
2. **遵循 setup/loop 骨架** — `main()` 必须先 `system_setup()`，再 `setup()`，最后 `while(1){ system_loop(); loop(); }`。`system_setup()` 永远第一个调用且不可省略。
3. **回调必须先注册后初始化驱动** — `setup()` 中先 `low_code_register_callbacks(feature_update_from_system, event_from_system)`，再 `app_driver_init()`，否则系统下发的事件/特性更新无法抵达应用。
4. **loop() 只负责拉取消息** — `loop()` 里只调用 `low_code_get_feature_update_from_system()` 与 `low_code_get_event_from_system()`；真正的业务在它们触发的回调 `feature_update_from_system()` / `event_from_system()` 中执行。
5. **单线程，禁止动态内存** — LP Core 固件单线程同步执行；文档明确要求 **不要使用 malloc/calloc**，改用静态缓冲；禁止深递归以防栈溢出。
6. **特性上报用 `low_code_feature_update_to_system()`** — 设备状态变化（按键、传感器读数）主动推给系统时调用；`feature_id` 用 `low_code_feature_id_t` 枚举（如 `LOW_CODE_FEATURE_ID_POWER`），没有映射时设 `LOW_CODE_FEATURE_ID_UNHANDLED` 并填 `low_level.matter.cluster_id/attribute_id`。
7. **事件上报用 `low_code_event_to_system()`** — 触发工厂复位等离散事件时构造 `low_code_event_t{ .event_type = LOW_CODE_EVENT_FACTORY_RESET }` 上报。
8. **系统事件用 switch 全覆盖** — `app_driver_event_handler()` 应 switch `event->event_type` 处理所有 `LOW_CODE_EVENT_*`（至少 SETUP/NETWORK/OTA/READY/IDENTIFICATION 系列），用灯光/显示给出对应指示。
9. **传感器周期上报靠 system_timer** — 用 `system_timer_create(cb, arg, timeout_ms, periodic)` + `system_timer_start()` 创建周期定时器，回调里读取并 `low_code_feature_update_to_system()` 上报。
10. **GPIO 只能用 system_* API** — LP Core 上 `pinMode/digitalWrite` 等价物是 `system_set_pin_mode()` / `system_digital_write()` / `system_digital_read()`，不要用标准 ESP-IDF 的 `gpio_*`。
11. **数据模型在 data_model.zap** — 新增 endpoint/cluster/attribute 要编辑 `configuration/data_model_wifi.zap`（或 thread 版），运行「Upload Configuration」重新生成 `data_model.bin` 烧录；endpoint 0 必须是 root node。
12. **改了 product 配置/数据模型必须重跑 Upload Configuration** — `product_info.json`、`product_config.json`、`*.zap` 任一改动都需重新生成证书与 `data_model.bin` 并烧录。

## When to Use

**Applicable:**
- 基于 esp-lowcode-matter 创建或定制 Matter 产品（灯、插座、传感器、温控器、占用检测等）
- 在 LP Core 上编写 `app_main.cpp` / `app_driver.cpp`，实现 setup/loop 与事件/特性回调
- 使用官方组件：button、relay、light（LED/WS2812）、temperature_sensor_sht30、occupancy_sensor_ld2420、display_ssd1306、system、sw_timer
- 自定义 Matter 数据模型（`data_model.zap`）、`product_info.json`、`product_config.json`
- 本地终端或 VS Code 的构建、烧录、`Upload Configuration`、console 调试流程
- LP Core panic（Breakpoint / Illegal Instruction）的栈定位与日志调试

**Not applicable:**
- 直接用 ESP Matter SDK（`esp-matter`）或 Connectedhomeip SDK 开发（请用对应方案，见 `matter_solutions.md`）
- 在 HP Core 上写多线程 Matter 业务（LowCode 不暴露该层）
- 非 ESP32-C6 芯片（框架目前仅支持 ESP32-C6）
- PCB 硬件设计、原理图设计

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应 recipe**——它包含完整调用链、分步说明、常见错误与可直接复制的代码。

### 入门与产品创建

| recipe | scenario |
|---|---|
| `recipes/getting_started.md` | 本地终端搭建环境（ESP-IDF v5.3 + ESP-AMP + LowCode）、Prepare Device、Upload Configuration、Upload Code 全流程 |
| `recipes/create_product.md` | 基于模板/已有产品创建新产品、目录结构、CMakeLists.txt 依赖配置 |

### 核心 LowCode 模型

| recipe | scenario |
|---|---|
| `recipes/setup_loop_model.md` | setup/loop 骨架、注册回调、`system_setup()`/`system_loop()` 主循环 |
| `recipes/feature_update.md` | 接收系统下发特性（`feature_update_from_system`）、用 `low_code_feature_update_to_system` 上报特性（含 `feature_id` 与 matter 低层标识两种方式） |
| `recipes/event_handling.md` | 处理 `LOW_CODE_EVENT_*` 系统事件、用 `low_code_event_to_system` 触发工厂复位 |

### 外设与驱动

| recipe | scenario |
|---|---|
| `recipes/gpio_system.md` | LP Core GPIO：`system_set_pin_mode` / `system_digital_write` / `system_digital_read`，Arduino→LowCode 映射 |
| `recipes/button_driver.md` | 按键组件：`button_driver_create` + `button_driver_register_cb`，单击/长按回调，工厂复位触发 |
| `recipes/light_driver.md` | 灯光组件：LED(PWM) 与 WS2812，CW/RGB/RGBCW 通道组合，亮度/色温/色调/饱和度，blink/breathe 特效 |
| `recipes/relay_socket.md` | 继电器组件 + 智能插座（单/双通道）`app_driver_set_socket_state` |

### 传感器与显示（周期上报）

| recipe | scenario |
|---|---|
| `recipes/system_timer.md` | `system_timer_create/start/stop/delete` 周期/单次定时器，传感器周期读取上报模式 |
| `recipes/sht30_sensor.md` | SHT30 温度传感器（I2C），周期读取并用 `LOW_CODE_FEATURE_ID_TEMPERATURE_SENSOR_VALUE` 上报 |
| `recipes/ld2420_occupancy.md` | LD2420 雷达占用传感器（UART），normal/report 模式，`LOW_CODE_FEATURE_ID_OCCUPANCY_SENSOR_VALUE` 上报 |
| `recipes/ssd1306_display.md` | SSD1306/SSD1315 OLED（I2C），`display_ssd1306_i2c_create` + draw_string/refresh_gram |
| `recipes/thermostat.md` | 温控器产品（MA-thermostat 769）：**单 endpoint 多 feature**——温度(4004)/制冷设定点(4005)/制热设定点(4006)，均为 `int16_t`(°C×100) |

### 配置与调试

| recipe | scenario |
|---|---|
| `recipes/product_configuration.md` | `product_info.json`（vendor_id/product_id/chip/connection_type 等）与 `data_model.zap` 数据模型定制 |
| `recipes/debugging.md` | LP Core Breakpoint/Illegal Instruction panic、`addr2line` 定位、日志调试、编码实践 |

---

## 官方组件与驱动清单

### 组件（`components/`）

| 组件 | 头文件 | 说明 |
|---|---|---|
| low_code | `low_code.h` | 核心：事件/特性收发、回调注册 |
| system | `system.h` | LP Core 系统工具：GPIO、延时、软件定时器、`system_setup/loop` |
| sw_timer | `sw_timer.h` | LP Core 软件定时器（periodic/one-shot，毫秒级） |
| button | `button_driver.h` | 按键输入，支持 HP/LP GPIO，单击/长按回调 |
| relay | `relay_driver.h` | 基于 GPIO 的继电器通断控制 |
| light | `light_driver.h` + `color_format.h` | LED(PWM)/WS2812 灯光，多通道组合与特效 |
| temperature_sensor_sht30 | `temperature_sensor_sht30.h` | SHT30 I2C 温度传感器 |
| occupancy_sensor_ld2420 | `occupancy_sensor_ld2420.h` | LD2420 UART 雷达占用传感器 |
| display_ssd1306 | `display_ssd1306.h` | SSD1306 OLED I2C 显示 |
| low_code_transport | — | HP/LP 核间通信传输层 |

### 外设驱动（`drivers/`）

| 驱动 | 说明 |
|---|---|
| i2c | 支持 HP/LP I2C master（SHT30、SSD1306 使用） |
| rmt | LP Core RMT 发送（WS2812 使用） |
| uart | 支持 HP/LP UART RX/TX（LD2420 使用） |

---

## Matter 特性 ID 速查（`low_code_feature_id_t`）

| feature_id 枚举 | 数值 | 用途 |
|---|---|---|
| `LOW_CODE_FEATURE_ID_UNHANDLED` | 0 | 未映射，用 `low_level.matter.*` 标识 |
| `LOW_CODE_FEATURE_ID_POWER` | 1001 | 开关通断（插座/灯/温控） |
| `LOW_CODE_FEATURE_ID_BRIGHTNESS` | 1002 | 亮度 |
| `LOW_CODE_FEATURE_ID_COLOR_TEMPERATURE` | 1003 | 色温 |
| `LOW_CODE_FEATURE_ID_HUE` | 1004 | 色调 |
| `LOW_CODE_FEATURE_ID_SATURATION` | 1005 | 饱和度 |
| `LOW_CODE_FEATURE_ID_TEMPERATURE` | 4004 | 温度（温控） |
| `LOW_CODE_FEATURE_ID_COOLING_SETPOINT` | 4005 | 制冷设定点 |
| `LOW_CODE_FEATURE_ID_HEATING_SETPOINT` | 4006 | 制热设定点 |
| `LOW_CODE_FEATURE_ID_TEMPERATURE_SENSOR_VALUE` | 5001 | 温度传感器读数 |
| `LOW_CODE_FEATURE_ID_OCCUPANCY_SENSOR_VALUE` | 6001 | 占用检测值 |

## 事件类型速查（`low_code_event_type_t`，节选）

| 事件 | 含义 | 典型响应 |
|---|---|---|
| `LOW_CODE_EVENT_SETUP_MODE_START` / `_END` | 进入/退出配网模式 | 启动/停止灯光 blink 指示 |
| `LOW_CODE_EVENT_SETUP_DEVICE_CONNECTED` | 配网中设备被连接 | 日志 |
| `LOW_CODE_EVENT_SETUP_STARTED` / `_SUCCESSFUL` / `_FAILED` | 配网开始/成功/失败 | 显示 "Setup Success/Failed" |
| `LOW_CODE_EVENT_NETWORK_CONNECTED` / `_DISCONNECTED` | 网络 up/down | 日志/指示 |
| `LOW_CODE_EVENT_OTA_STARTED` / `_STOPPED` | OTA 开始/结束 | 日志 |
| `LOW_CODE_EVENT_READY` | 设备就绪 | 日志 |
| `LOW_CODE_EVENT_IDENTIFICATION_START` / `_STOP` 等 | 识别流程 | 灯效 |
| `LOW_CODE_EVENT_FACTORY_RESET` | 工厂复位（应用主动上报） | `low_code_event_to_system()` |
| `LOW_CODE_EVENT_TEST_MODE_LOW_CODE/_COMMON/_BLE/_SNIFFER` | 测试模式 | 日志，读 `event->event_data` 取 subtype |

---

## 产品默认引脚映射（取自仓库各 product 源码）

> 引脚为各示例产品的默认值，可按硬件修改。GPIO 编号遵循 ESP32-C6。

| 用途 | 产品 | GPIO | 说明 |
|---|---|---|---|
| Button（按键） | socket / temperature_sensor / occupancy_sensor / display | GPIO9 | pullup，active_level=0 |
| Relay（继电器） | socket | GPIO2 | `relay_driver_init(2)` |
| WS2812 指示灯 | socket / temperature_sensor / occupancy_sensor | GPIO8 | `ws2812_io.ctrl_io` |
| PWM 冷白 | light_cw_pwm | GPIO4 (cold) / GPIO6 (warm) | 2CH CW |
| I2C SCL / SDA | temperature_sensor(_with_display) | GPIO1 / GPIO2 | `I2C_NUM_0` |
| LD2420 UART TX/RX | occupancy_sensor | GPIO3 (TX) / GPIO2 (RX) | `UART_NUM_1` |
| SSD1306 I2C 地址 | temperature_sensor_with_display | 0x3C (`SSD1306_I2C_ADDRESS`) | `I2C_NUM_0` |

---

## 数据流与执行模型

```
HP Core (预编译镜像): Wi-Fi + BLE + Matter 协议栈  ──消息──┐
                                                          │ ESP-AMP
LP Core (产品代码):                                         │
  main() -> system_setup() -> setup() {                    │
      low_code_register_callbacks(feature_cb, event_cb)    │
      app_driver_init()                                    │
  }                                                        │
  while(1){                                                │
      system_loop();                                       │
      loop(){                                              │
          low_code_get_feature_update_from_system() ──┐    │
          low_code_get_event_from_system()         ──┐│    │
      }                                              ││    │
  }                                                  ││    │
  feature_update_from_system() <──── 特性下发 ───────┘│────┘
  event_from_system()          <──── 事件下发 ────────┘
  low_code_feature_update_to_system()  ──── 特性上报 ───>
  low_code_event_to_system()           ──── 事件上报 ───>
```

---

## Critical Pitfalls (Must Read)

以下是最常见、最致命的错误，违反任何一条都会导致固件不工作或 panic。

### 1. `system_setup()` 必须第一个调用

```cpp
// ❌ WRONG — 未调用 system_setup()，LP Core 系统未初始化
extern "C" int main() {
    setup();
    while (1) { loop(); }
}

// ✅ CORRECT — system_setup() 永远第一，再 setup()
extern "C" int main() {
    printf("%s: Starting low code\n", TAG);
    system_setup();   // 必须最先且不可省略
    setup();
    while (1) {
        system_loop();
        loop();
    }
    return 0;
}
```

### 2. 主循环必须同时跑 `system_loop()` 和 `loop()`

```cpp
// ❌ WRONG — 漏掉 system_loop()，定时器/系统任务不推进
while (1) {
    loop();
}

// ✅ CORRECT
while (1) {
    system_loop();   // 推进系统/定时器
    loop();          // 拉取事件/特性消息
}
```

### 3. 回调注册必须在驱动初始化之前

```cpp
// ❌ WRONG — 先 init 驱动再注册回调，初始化期间的事件丢失
static void setup() {
    app_driver_init();
    low_code_register_callbacks(feature_update_from_system, event_from_system);
}

// ✅ CORRECT — 先注册回调，再初始化驱动
static void setup() {
    low_code_register_callbacks(feature_update_from_system, event_from_system);
    app_driver_init();
}
```

### 4. `feature_update_from_system` 要按 endpoint + feature_id 分发

```cpp
// ❌ WRONG — 不判断 endpoint/feature_id，所有下发都当 POWER 处理
int feature_update_from_system(low_code_feature_data_t *data) {
    bool v = *(bool *)data->value.value;
    return app_driver_set_socket_state(v);
}

// ✅ CORRECT — 按 endpoint + feature_id 精确分发
int feature_update_from_system(low_code_feature_data_t *data) {
    uint16_t endpoint_id = data->details.endpoint_id;
    uint32_t feature_id  = data->details.feature_id;

    if (endpoint_id == 1) {
        if (feature_id == LOW_CODE_FEATURE_ID_POWER) {
            bool power_value = *(bool *)data->value.value;
            return app_driver_set_socket_state(power_value);
        }
    }
    return 0;
}
```

### 5. 上报特性时 `value.type` 与数据类型必须一致

```cpp
// ❌ WRONG — 温度用 UNSIGNED_INTEGER，系统侧解析错误
low_code_feature_value_t v = {
    .type = LOW_CODE_VALUE_TYPE_UNSIGNED_INTEGER,  // 温度是带符号
    .value_len = sizeof(int16_t),
    .value = (uint8_t*)&temperature,
};

// ✅ CORRECT — 温度（°C*100，可负）用 INTEGER
int16_t temperature = temp * 100;
low_code_feature_data_t update_data = {
    .details = { .endpoint_id = 1, .feature_id = LOW_CODE_FEATURE_ID_TEMPERATURE_SENSOR_VALUE },
    .value = {
        .type = LOW_CODE_VALUE_TYPE_INTEGER,
        .value_len = sizeof(int16_t),
        .value = (uint8_t*)&temperature,
    },
};
low_code_feature_update_to_system(&update_data);
```

### 6. 无 `feature_id` 映射时必须填 `low_level.matter.*`

```cpp
// ❌ WRONG — feature_id=UNHANDLED 又不填 matter 标识，系统无法路由
low_code_feature_data_t f = {
    .details = { .endpoint_id = 1, .feature_id = LOW_CODE_FEATURE_ID_UNHANDLED },
    .value = { .type = LOW_CODE_VALUE_TYPE_BOOLEAN, .value_len = sizeof(bool), .value = &on },
};

// ✅ CORRECT — 用 matter cluster/attribute 标识
low_code_feature_data_t f = {
    .details = {
        .endpoint_id = 1,
        .feature_id = LOW_CODE_FEATURE_ID_UNHANDLED,
        .low_level = { .matter = { .cluster_id = 0x0006, .attribute_id = 0x0000 } }
    },
    .value = { .type = LOW_CODE_VALUE_TYPE_BOOLEAN, .value_len = sizeof(bool), .value = &on },
};
```

### 7. 工厂复位是「上报事件」，不是本地执行

```cpp
// ❌ WRONG — 自己擦 flash 想复位
static void btn_cb(void *arg, void *data) {
    // 直接操作 flash/分区...（HP Core 才管 Matter 凭证，这里无效）
}

// ✅ CORRECT — 上报 LOW_CODE_EVENT_FACTORY_RESET 交给系统
static void btn_cb(void *arg, void *data) {
    low_code_event_t event = { .event_type = LOW_CODE_EVENT_FACTORY_RESET };
    low_code_event_to_system(&event);
}
```

### 8. 按键长按触发复位用 `BUTTON_LONG_PRESS_UP`

```cpp
// ❌ WRONG — 用 PRESS_DOWN，按下瞬间就复位，易误触
button_driver_register_cb(btn, BUTTON_PRESS_DOWN, factory_reset_cb, NULL);

// ✅ CORRECT — 长按抬起才触发
button_driver_register_cb(btn, BUTTON_LONG_PRESS_UP, factory_reset_cb, NULL);
```

### 9. LED 与 WS2812 配置不同，`io_conf` 是 union

```cpp
// ❌ WRONG — WS2812 却填 led_io 字段
light_driver_config_t cfg = {
    .device_type = LIGHT_DEVICE_TYPE_WS2812,
    .io_conf = { .led_io = { .red = 8 } },   // union 用错成员
};

// ✅ CORRECT — WS2812 用 ws2812_io
light_driver_config_t cfg = {
    .device_type = LIGHT_DEVICE_TYPE_WS2812,
    .channel_comb = LIGHT_CHANNEL_COMB_3CH_RGB,
    .io_conf = { .ws2812_io = { .ctrl_io = INDICATOR_GPIO_NUM } },
    .min_brightness = 0, .max_brightness = 100,
};
```

### 10. PWM 灯亮度的 Matter↔driver 量程换算

```cpp
// ❌ WRONG — 直接把 Matter 的 0-255 亮度塞给 driver(0-100)
int app_driver_set_light_brightness(uint8_t brightness) {
    return light_driver_set_brightness(brightness);  // 越界
}

// ✅ CORRECT — 换算到 0-100
int app_driver_set_light_brightness(uint8_t brightness) {
    brightness = brightness * 100 / 255;
    return light_driver_set_brightness(brightness);
}
// 注：色温同理，Matter mireds 与 driver Kelvin 需转换：temperature = 1000000 / temperature;
```

### 11. 周期定时器回调签名必须匹配 `system_timer_cb_t`

```cpp
// ❌ WRONG — 回调签名不匹配，编译/调用错误
void read_cb(void) {
    ...
}

// ✅ CORRECT — 必须是 (system_timer_handle_t, void*)
void app_driver_read_and_report_feature(system_timer_handle_t timer_handle, void *user_data) {
    /* timer_handle/user_data 故意未用，但签名必须保留 */
    float temperature = 0.0;
    temperature_sensor_sht30_get_celsius(I2C_PORT, &temperature);
    app_driver_report_temperature(temperature);
}
// 创建：
system_timer_handle_t t = system_timer_create(app_driver_read_and_report_feature, NULL, 10000, true);
system_timer_start(t);
```

### 12. LP Core 禁用动态内存与深递归

```cpp
// ❌ WRONG — malloc / 深递归，LP Core 单线程易栈溢出/Illegal Instruction panic
char *buf = (char*)malloc(256);
int fib(int n){ return n<2 ? n : fib(n-1)+fib(n-2); }

// ✅ CORRECT — 静态/栈缓冲，迭代实现
char buf[256];
// 或局部栈缓冲（注意边界，绝不越界）
```

### 13. 新增 endpoint/cluster 后必须改代码 + 重跑 Upload Configuration

```
❌ WRONG — 只改 data_model.zap，app 不按 endpoint/feature 处理，也不重烧 data_model.bin
✅ CORRECT:
  1. 编辑 configuration/data_model_wifi.zap（endpoint 0 保持 root node）
  2. 更新 feature_update_from_system() 中 endpoint/feature 分发逻辑
  3. 运行 "Upload Configuration" 重新生成并烧录 data_model.bin
  4. 测试内存（新增 endpoint/cluster 会增加内存占用）
```

### 14. 新增组件必须在 `main/CMakeLists.txt` 的 REQUIRES 里声明

```cmake
# ❌ WRONG — 用了 temperature_sensor_sht30 但未声明
idf_component_register(SRC_DIRS . INCLUDE_DIRS . REQUIRES low_code system)

# ✅ CORRECT
idf_component_register(SRC_DIRS .
                        INCLUDE_DIRS .
                        REQUIRES low_code system button light temperature_sensor_sht30)
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确产品类型（灯/插座/传感器/温控…）、所需组件、endpoint 与 feature_id |
| 2 | Recipe | 匹配 `recipes/` 场景，先读对应 recipe 获取调用链与代码骨架 |
| 3 | Scaffold | 新产品优先复制最接近的 product（无把握用 `products/template/`）改写，不凭空写 |
| 4 | Code | 在 `app_main.cpp`（setup/loop/回调）与 `app_driver.cpp`（驱动/事件处理）中实现 |
| 5 | Query | 未覆盖的 API 查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 6 | Validate | 校验：`system_setup()` 首发、回调先注册、endpoint/feature_id 分发、union 成员匹配、无动态内存 |
| 7 | Configure | 改 `product_info.json` / `data_model.zap` 后必须运行 Upload Configuration |
| 8 | Build/Flash | `idf.py set-target esp32c6` → `idf.py build` → esptool 烧录（见 `recipes/getting_started.md`） |
| 9 | Debug | console 看日志；panic 用 `riscv32-esp-elf-addr2line -e <elf> <MEPC>` 定位（见 `recipes/debugging.md`） |

### Step 3 Detail — 产品创建策略

**新产品（目标目录无 product 时）：**
1. 选最接近的现有 product 作参考：
   - 灯（CW PWM）→ `products/light_cw_pwm/`
   - 灯（RGB WS2812）→ `products/light_rgbcw_ws2812/`
   - 单通道插座 → `products/socket/`
   - 多通道插座 → `products/socket_2_channel/`
   - 温度传感器 → `products/temperature_sensor/`
   - 温度传感器+显示 → `products/temperature_sensor_with_display/`
   - 占用传感器 → `products/occupancy_sensor/`
   - 温控器 → `products/thermostat/`
   - 通用空白 → `products/template/`
2. 复制整个 product 目录（保留 `configuration/`、`main/`、`CMakeLists.txt`、`sdkconfig.defaults`）
3. 修改 `app_main.cpp` / `app_driver.cpp` / `app_priv.h` / `configuration/*` 适配需求
4. 在 `main/CMakeLists.txt` 的 `REQUIRES` 补齐新依赖组件

**已有 product：** 就地编辑，勿整体覆盖。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 不在 `resources/api_reference.md` | 停下，告知用户该 API 不存在，勿臆造 |
| 不确定 setup/loop 顺序 | 默认 `system_setup()` → `setup()`(注册回调→init驱动) → `while{system_loop();loop();}` |
| LP Core Breakpoint panic | 用 `addr2line` + MEPC 地址定位，多为空指针访问 |
| Illegal Instruction panic | 多为缓冲区溢出/栈溢出；依赖日志而非栈转储，检查数组边界、去掉 malloc/深递归 |
| 事件/特性收不到 | 检查回调是否在 `setup()` 内、驱动 init 之前注册；`loop()` 是否调了两个 `low_code_get_*` |
| 按键不响应 | 确认 `system_enable_software_interrupt()`（button 需软件中断）与 Kconfig 选 HP/LP GPIO |
| 数据模型改动不生效 | 必须重跑「Upload Configuration」重新生成并烧录 `data_model.bin` |
| 组件符号未定义 | 检查 `main/CMakeLists.txt` 的 `REQUIRES` 是否声明了该组件 |
| 仅 ESP32-C6 可用 | 框架当前只支持 ESP32-C6，其它芯片告知用户不支持 |

## References

- 场景 recipes → `recipes/` 目录
- LowCode + system + 驱动 API 速查 → `resources/api_reference.md`
- 配置（product_info / Kconfig / 引脚）→ `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 真实示例产品索引 → `resources/example_list.md`
- 原仓库文档（只读参考）：`README.md`、`docs/programmer_model.md`、`docs/create_product.md`、`docs/product_configuration.md`、`docs/debugging.md`、`docs/getting_started_terminal.md`、`docs/matter_solutions.md`
