# 设备控制：Initiator（开关/传感器）侧

> **适用摘要**: 在 initiator 设备上通过按键触发绑定、解绑与控制数据发送，控制 responder（灯/插座）动作（参考 `examples/control`）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-NOW 控制"
- "开关控制灯"
- "initiator 绑定"
- "espnow_ctrl_initiator"
- "按键发送控制命令"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/esp-now`，外加 `iot_button`（按键） |
| 头文件 | `espnow_ctrl.h` |
| 参考示例 | `examples/control/main/app_main.c` |

## 分步说明

### 1. 初始化（storage + wifi + espnow + control）

```c
#include "espnow.h"
#include "espnow_ctrl.h"
#include "espnow_storage.h"

void app_main(void)
{
    espnow_storage_init();
    app_wifi_init();

    espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
    cfg.receive_enable.control_bind = 1;   // 需要接收 bind 反馈可开启
    espnow_init(&cfg);

    // 可选：注册事件回调查看绑定结果
    esp_event_handler_register(ESP_EVENT_ESPNOW, ESP_EVENT_ANY_ID,
                               app_espnow_event_handler, NULL);

    app_driver_init();  // 按键初始化
}
```

### 2. 发起绑定（双击触发，广播 bind 帧）

```c
// 双击：绑定；长按：解绑；单击：发送控制数据
static void app_bind_press_cb(void *arg, void *usr_data)
{
    ESP_ERROR_CHECK(!(BUTTON_DOUBLE_CLICK == iot_button_get_event(arg)));
    ESP_LOGI(TAG, "initiator bind press");
    // initiator_attribute 标识本发起方，responder 据此记录绑定来源
    espnow_ctrl_initiator_bind(ESPNOW_ATTRIBUTE_KEY_1, true);
}
```

### 3. 发送控制数据（单击触发）

```c
static void app_send_press_cb(void *arg, void *usr_data)
{
    static bool status = 0;
    ESP_ERROR_CHECK(!(BUTTON_SINGLE_CLICK == iot_button_get_event(arg)));
    ESP_LOGI(TAG, "initiator send press");
    // initiator_attribute=KEY_1, responder_attribute=POWER, value=0/1
    espnow_ctrl_initiator_send(ESPNOW_ATTRIBUTE_KEY_1, ESPNOW_ATTRIBUTE_POWER, status);
    status = !status;
}
```

### 4. 解绑（长按触发）

```c
static void app_unbind_press_cb(void *arg, void *usr_data)
{
    ESP_ERROR_CHECK(!(BUTTON_LONG_PRESS_START == iot_button_get_event(arg)));
    ESP_LOGI(TAG, "initiator unbind press");
    espnow_ctrl_initiator_bind(ESPNOW_ATTRIBUTE_KEY_1, false);  // enable=false 解绑
}
```

### 5. 按键注册（button 4.x API）

```c
#include "iot_button.h"
#include "button_gpio.h"

// 默认 GPIO 基于各芯片 DevKitC（见 examples/control 的 CONFIG_IDF_TARGET 分支）
#define CONTROL_KEY_GPIO   GPIO_NUM_9   // ESP32-C3/C2/C6

static void app_driver_init(void)
{
    button_config_t btn_cfg = { .long_press_time = 2000, .short_press_time = 180 };
    button_gpio_config_t gpio_cfg = { .gpio_num = CONTROL_KEY_GPIO, .active_level = 0 };
    button_handle_t h = NULL;
    ESP_ERROR_CHECK(iot_button_new_gpio_device(&btn_cfg, &gpio_cfg, &h));

    iot_button_register_cb(h, BUTTON_SINGLE_CLICK,    NULL, app_send_press_cb,   NULL);
    iot_button_register_cb(h, BUTTON_DOUBLE_CLICK,    NULL, app_bind_press_cb,   NULL);
    iot_button_register_cb(h, BUTTON_LONG_PRESS_START,NULL, app_unbind_press_cb, NULL);
}
```

### 6. 处理绑定结果事件（可选）

```c
static void app_espnow_event_handler(void *a, esp_event_base_t base, int32_t id, void *data)
{
    if (base != ESP_EVENT_ESPNOW) return;
    switch (id) {
    case ESP_EVENT_ESPNOW_CTRL_BIND: {
        espnow_ctrl_bind_info_t *info = data;
        ESP_LOGI(TAG, "bind " MACSTR " type %d", MAC2STR(info->mac), info->initiator_attribute);
        break;
    }
    case ESP_EVENT_ESPNOW_CTRL_UNBIND: {
        espnow_ctrl_bind_info_t *info = data;
        ESP_LOGI(TAG, "unbind " MACSTR, MAC2STR(info->mac));
        break;
    }
    case ESP_EVENT_ESPNOW_CTRL_BIND_ERROR: {
        espnow_ctrl_bind_error_t *e = data;
        ESP_LOGW(TAG, "bind error %d", *e);  // TIMEOUT / RSSI / LIST_FULL
        break;
    }
    }
}
```

### 可用 initiator_attribute（来自 `espnow_attribute_t`）

| 类别 | 取值示例 |
|---|---|
| 基础 | `ESPNOW_ATTRIBUTE_POWER`, `_POWER_ADD`, `_ATTRIBUTE` |
| 灯 | `ESPNOW_ATTRIBUTE_BRIGHTNESS`, `_HUE`, `_SATURATION`, `_WARM`, `_COLD`, `_RED/GREEN/BLUE`, `_MODE` 及对应 `_ADD` |
| 按键 | `ESPNOW_ATTRIBUTE_KEY_1` … `ESPNOW_ATTRIBUTE_KEY_10` |
| 电池 | `ESPNOW_ATTRIBUTE_STATUS_LOW_BATTERY`, `_BATTERY_LEVEL`, `_CHARGING_STATE` |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| responder 不动作 | responder 未进入绑定窗口 | responder 先调 `espnow_ctrl_responder_bind(30s, -55, NULL)` |
| 绑定列表满 | 超过 `ESPNOW_BIND_LIST_MAX_SIZE(32)` | responder 用 `espnow_ctrl_responder_clear_bindlist` 清空 |
| 控制值类型不对 | 以为 value 是 bool | 签名固定 `uint32_t responder_value`，0/1/2 等自定义编码 |
| 按键无反应 | GPIO 选错 | 按 `CONFIG_IDF_TARGET_*` 选 CONTROL_KEY_GPIO（C3/C2/C6=9，ESP32/S2/S3=0） |
| 自动信道发送不稳 | 未配 `CONFIG_ESPNOW_CONTROL_AUTO_CHANNEL_SENDING` | 需要时开启并配 `WAIT_ACK_DURATION` / `RETRANSMISSION_TIMES` |

## 参考

- `examples/control/main/app_main.c` — initiator + responder 一体示例（按键+灯）
- `examples/coin_cell_demo/switch/main/app_main.c` — 低功耗开关 initiator
- `resources/api_reference.md` — `espnow_ctrl_initiator_bind` / `espnow_ctrl_initiator_send`
