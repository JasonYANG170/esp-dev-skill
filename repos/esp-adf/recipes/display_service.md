# 显示服务：LED 灯效模式(display_service)与 LED 驱动

> **适用摘要**: 用 `display_service_set_pattern(handle, pattern, value)` 按功能语义（WiFi 连接中 / 蓝牙已连 / 唤醒 / 播放中 / 录音中 / 音量 / 低电等）点亮板载 LED，底层驱动由 `audio_board_led_init()` 根据 `CONFIG_*_BOARD` 自动选：PWM 单 LED（`led_indicator`）、AW2013、IS31x、WS2812。`display_pattern_t` 枚举定义了所有可用灯效，驱动里实现不支持的 pattern 会打 `LED_INDI: The led mode is invalid` 警告并安全跳过。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/display_service.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "LED 状态灯 / 显示服务 / display_service"
- "WiFi 连接 / 蓝牙连接时亮什么灯"
- "唤醒后灯效 / 命令识别灯"
- "音量条 / 电量灯 / 录音灯"
- "check_display_led 那个例子"
- "WS2812 / AW2013 / IS31x 灯条"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/checks/check_display_led/main/check_display_led_example.c`、`examples/display/led_pixels`、`examples/display/music_player` |
| 头文件 | `components/display_service/include/display_service.h`；驱动头 `led_bar_ws2812.h`、`led_bar_aw2013.h`、`led_bar_is31x.h`（在 `components/display_service/led_bar/include/`） |
| 板级入口 | `audio_board_led_init()`（由各板 `board.c` 实现，链接对应驱动） |
| 文档 | `docs/en/api-reference/services/display_service.rst` |

## 分步说明

### 1. 核心三步（板级高层用法，绝大多数场景够用）

```c
#include "display_service.h"
#include "board.h"

// 1. 创建 display_service（驱动由板宏自动选）
display_service_handle_t disp = audio_board_led_init();

// 2. 按功能语义设灯效
display_service_set_pattern(disp, DISPLAY_PATTERN_WIFI_CONNECTTING, 0);
// ... WiFi 连上后 ...
display_service_set_pattern(disp, DISPLAY_PATTERN_WIFI_CONNECTED, 0);

// 3. 销毁（可选）
display_destroy(disp);
```

> `audio_board_led_init()` 在所有 ADF 板的 `board.h` 都声明，内部按 `CONFIG_*_BOARD` 链接 PWM/AW2013/IS31x/WS2812 之一。没有 LED 的板（如 `ESP32-S3-Korvo-2 v3`）返回 `NULL`，调用 `display_service_set_pattern(NULL,...)` 是安全的（内部判空）。

### 2. display_pattern_t 全量枚举（来自 display_service.h）

按功能分组，记住语义即可，不必关心具体 LED 怎么亮：

| 分组 | pattern |
|---|---|
| WiFi | `WIFI_SETTING` / `WIFI_CONNECTTING` / `WIFI_CONNECTED` / `WIFI_DISCONNECTED` / `WIFI_SETTING_FINISHED` / `WIFI_NO_CFG` |
| 蓝牙 | `BT_CONNECTTING` / `BT_CONNECTED` / `BT_DISCONNECTED` |
| 录音/识别 | `RECORDING_START` / `RECORDING_STOP` / `RECOGNITION_START` / `RECOGNITION_STOP` |
| 唤醒/播放 | `WAKEUP_ON` / `WAKEUP_FINISHED` / `MUSIC_ON` / `MUSIC_FINISHED` |
| 音量/静音 | `VOLUME`（带 value）/ `MUTE_ON` / `MUTE_OFF` |
| 电源 | `TURN_ON` / `TURN_OFF` / `BATTERY_LOW` / `BATTERY_CHARGING` / `BATTERY_FULL` / `POWERON_INIT` |
| 语音交互 | `SPEECH_BEGIN` / `SPEECH_OVER` |

> 第一个值 `DISPLAY_PATTERN_UNKNOWN = 0`，最大 `DISPLAY_PATTERN_MAX`。`check_display_led` 示例就是从 0 循环到 `MAX-1` 逐个点亮。

### 3. VOLUME 这类带数值的 pattern

```c
// value 表示音量百分比 0..100
display_service_set_pattern(disp, DISPLAY_PATTERN_VOLUME, 60);
```

### 4. 与播放/蓝牙事件联动（典型智能音箱灯效）

在 `audio_event_iface_listen` 主循环里，按事件切灯：

```c
if (msg.source_type == PERIPH_ID_WIFI) {
    if (msg.cmd == PERIPH_WIFI_CONNECTING) {
        display_service_set_pattern(disp, DISPLAY_PATTERN_WIFI_CONNECTTING, 0);
    } else if (msg.cmd == PERIPH_WIFI_CONNECTED) {
        display_service_set_pattern(disp, DISPLAY_PATTERN_WIFI_CONNECTED, 0);
    }
}
// 蓝牙 / 唤醒 / 播放 类似，分别在对应事件分支里 set_pattern
```

> 语音唤醒场景：唤醒回调里 `DISPLAY_PATTERN_WAKEUP_ON`，命令识别到 `DISPLAY_PATTERN_RECOGNITION_START`，结束 `WAKEUP_FINISHED`。

### 5. 手动指定驱动后端（自定义板或想绕过 board.c）

直接用底层驱动 init，再装配 `display_service_config_t`：

```c
// WS2812（NeoPixel 灯带）
#include "led_bar_ws2812.h"
led_bar_ws2812_handle_t ws = led_bar_ws2812_init(GPIO_NUM_8, 4);  // GPIO + 灯数
display_service_config_t display_cfg = {
    .based_cfg = {
        .task_stack = 0, .task_prio = 0, .task_core = 0, .task_func = NULL,
        .service_start = NULL, .service_stop = NULL, .service_destroy = NULL,
        .service_ioctl = led_bar_ws2812_pattern,   // 关键：把 pattern 函数挂上
        .service_name = "DISPLAY_serv",
    },
    .instance = ws,
};
display_service_handle_t disp = display_service_create(&display_cfg);
```

> 其他驱动同理：AW2013 用 `led_bar_aw2013_init()` + `led_bar_aw2013_pattern`；IS31x 用 `led_bar_is31x_init()` + `led_bar_is31x_pattern`；PWM 单 LED 用 `led_indicator_init()` + `led_indicator_pattern`（LyraT V4.3 的 `audio_board_led_init` 即此模式）。

### 6. 四种驱动对照

| 驱动 | init 函数 | pattern 函数 | 典型板 |
|---|---|---|---|
| PWM 单 LED | `led_indicator_init(gpio)` | `led_indicator_pattern` | ESP32-LyraT V4.3 / V4.2（绿色单 LED） |
| AW2013 | `led_bar_aw2013_init()` | `led_bar_aw2013_pattern` | ESP32-LyraTD-MSC（3 颗 RGB） |
| IS31x | `led_bar_is31x_init()` | `led_bar_is31x_pattern` | 旧 LyraT 灯条 |
| WS2812 | `led_bar_ws2812_init(gpio, num)` | `led_bar_ws2812_pattern` | Korvo-2L / 自定义 NeoPixel |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `LED_INDI: The led mode is invalid` 警告 | 当前驱动不支持该 pattern | 正常现象——驱动只实现部分 pattern，不支持的会安全跳过；换支持的板或驱动 |
| `audio_board_led_init()` 返回 NULL | 该板无 LED（如 Korvo-2 v3） | 选有 LED 的板，或自己用 WS2812 驱动接 GPIO |
| LED 不亮 | GPIO/灯数配错或 PA 未使能 | `led_bar_ws2812_init(gpio, led_num)` 核对引脚；部分板 LED 与 PA 共供电 |
| pattern 没反应 | 传了 `DISPLAY_PATTERN_UNKNOWN` 或 `MAX` | 用枚举里的有效值（1 ~ MAX-1） |
| 多次 set_pattern 闪烁异常 | 上一个 pattern 还在跑 | 驱动内部会切换，一般无需手动停；若需要，先 set 一个 `TURN_OFF` 再设新的 |
| WS2812 颜色错 | 灯珠 RGB 顺序不同 | 见 `led_bar_ws2812.c` 实现，部分灯条是 GRB |

## 参考项目

- `examples/checks/check_display_led/main/check_display_led_example.c` — 循环点亮所有 pattern（默认 LyraT V4.3）
- `examples/display/led_pixels/main/main.c` — WS2812 低层灯效（energy/spectrum 模式，独立于 display_service）
- `examples/display/music_player` — 音乐播放器 UI 集成
- `docs/en/api-reference/services/display_service.rst` — API 说明
- `components/display_service/include/display_service.h` — `display_pattern_t` 枚举与 API
- `components/display_service/led_bar/include/led_bar_ws2812.h`、`led_bar_aw2013.h`、`led_bar_is31x.h` — 驱动头
- `components/audio_board/<board>/board.c` 的 `audio_board_led_init()` — 各板驱动装配
