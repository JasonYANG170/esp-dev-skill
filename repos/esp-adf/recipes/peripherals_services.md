# 外设集（Wi-Fi / SD / 触摸 / 按键）与事件集成

> **适用摘要**: ESP-ADF 用 `esp_peripherals` 统一管理外设（Wi-Fi、SD 卡、触摸、ADC 按键、GPIO 按键等），所有外设事件汇入同一个 `audio_event_iface`，与 pipeline 事件一起处理。本 recipe 覆盖外设初始化与事件分发。

## 触发意图

- "按键控制播放"
- "触摸按键 / ADC 按键"
- "Wi-Fi 连接 / 配网"
- "外设事件怎么收"
- "periph_touch / periph_wifi / periph_adc_button"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/get-started/play_mp3_control/`（触摸按键）、`checks/check_board_buttons/` |
| 头文件 | `components/esp_peripherals/include/esp_peripherals.h` 及 `periph_*.h` |

## 分步说明

### 1. 创建外设集

```c
#include "esp_peripherals.h"

esp_periph_config_t periph_cfg = DEFAULT_ESP_PERIPH_SET_CONFIG();
esp_periph_set_handle_t set = esp_periph_set_init(&periph_cfg);
```

### 2. 板级按键（统一入口，由 board.h 提供引脚）

```c
#include "board.h"
audio_board_key_init(set);   // 注册板载触摸/ADC 按键，引脚由 CONFIG_*_BOARD 决定
```

> `board.h` 还提供按键 ID 查询宏：`get_input_play_id()`、`get_input_set_id()`、`get_input_mode_id()`、`get_input_volup_id()`、`get_input_voldown_id()`。

### 3. 触摸按键（如需单独配）

```c
#include "periph_touch.h"
periph_touch_cfg_t touch_cfg = PERIPH_TOUCH_CFG_DEFAULT();   // 默认配置
esp_periph_handle_t touch = periph_touch_init(&touch_cfg);
esp_periph_start(set, touch);
// 事件命令：PERIPH_TOUCH_TAP / PERIPH_TOUCH_RELEASE / PERIPH_TOUCH_LONG_TAP / PERIPH_TOUCH_LONG_RELEASE
```

### 4. ADC 按键 / GPIO 按键

```c
#include "periph_adc_button.h"
#include "periph_button.h"
esp_periph_handle_t adc_btn = periph_adc_button_init(&periph_adc_button_cfg);
esp_periph_start(set, adc_btn);
// 命令：PERIPH_ADC_BUTTON_PRESSED / PERIPH_BUTTON_PRESSED
```

### 5. Wi-Fi 外设

```c
#include "periph_wifi.h"
periph_wifi_cfg_t wifi_cfg = {
    .wifi_config.sta.ssid     = CONFIG_WIFI_SSID,
    .wifi_config.sta.password = CONFIG_WIFI_PASSWORD,
};
esp_periph_handle_t wifi_handle = periph_wifi_init(&wifi_cfg);
esp_periph_start(set, wifi_handle);
periph_wifi_wait_for_connected(wifi_handle, portMAX_DELAY);
// 状态：PERIPH_WIFI_CONNECTING / CONNECTED / DISCONNECTED / ERROR ...
```

### 6. SD 卡外设

```c
#include "periph_sdcard.h"
audio_board_sdcard_init(set, SD_MODE_1_LINE);   // 内部注册 periph_sdcard
// 事件：PERIPH_SD_CARD_CHANGED（插拔）
```

### 7. 把外设事件挂到统一 evt（关键）

```c
audio_event_iface_cfg_t evt_cfg = AUDIO_EVENT_IFACE_DEFAULT_CFG();
audio_event_iface_handle_t evt = audio_event_iface_init(&evt_cfg);

audio_pipeline_set_listener(pipeline, evt);                                   // pipeline 事件
audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt);    // 外设事件

audio_pipeline_run(pipeline);

while (1) {
    audio_event_iface_msg_t msg;
    if (audio_event_iface_listen(evt, &msg, portMAX_DELAY) != ESP_OK) continue;

    // 按 source_type + cmd 分发
    if ((msg.source_type == PERIPH_ID_TOUCH
         || msg.source_type == PERIPH_ID_BUTTON
         || msg.source_type == PERIPH_ID_ADC_BTN)
        && (msg.cmd == PERIPH_TOUCH_TAP
            || msg.cmd == PERIPH_BUTTON_PRESSED
            || msg.cmd == PERIPH_ADC_BUTTON_PRESSED)) {

        if ((int)msg.data == get_input_play_id()) {
            // 播放/暂停切换：根据 audio_element_get_state 决定
        } else if ((int)msg.data == get_input_volup_id()) {
            audio_hal_set_volume(board->audio_hal, vol + 10);
        }
        // ...
    }
    if (msg.source_type == PERIPH_ID_WIFI && msg.cmd == PERIPH_WIFI_CONNECTED) {
        // 联网成功后再 run HTTP pipeline
    }
}
```

> 常用 `PERIPH_ID_*`：`PERIPH_ID_TOUCH`、`PERIPH_ID_BUTTON`、`PERIPH_ID_ADC_BTN`、`PERIPH_ID_WIFI`、`PERIPH_ID_SDCARD` 等（来自 esp_peripherals）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 收不到按键事件 | peripherals 事件未挂到 evt | `audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt)` |
| `get_input_play_id` 未定义 | 未选板或板无此按键 | `menuconfig` 选对 board；不同板按键 ID 不同 |
| 触摸误触 | 默认灵敏度不适合 | 调 `periph_touch_cfg_t` 阈值 |
| Wi-Fi 反复重连 | SSID/密码错或信号差 | 检查 `CONFIG_WIFI_SSID/PASSWORD` |
| SD 拔插不响应 | 未处理 `PERIPH_SD_CARD_CHANGED` | 加 source_type == PERIPH_ID_SDCARD 分支 |

## 参考项目

- `examples/get-started/play_mp3_control/main/play_mp3_control_example.c` — 触摸按键完整控制
- `examples/checks/check_board_buttons/` — 按键自检
- `examples/player/pipeline_http_mp3/` — periph_wifi 连接
- `examples/player/pipeline_sdcard_mp3_control/` — SD 卡 + 按键
