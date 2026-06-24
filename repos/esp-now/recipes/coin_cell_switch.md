# 硬币电池低功耗开关（Light Sleep + 控制）

> **适用摘要**: 在硬币电池供电的开关上，利用 light sleep 给电容充电、power lock 保持供电、状态持久化，并按需发送控制/绑定/解绑帧（参考 `examples/coin_cell_demo/switch`）。

## 触发意图

- "硬币电池 ESP-NOW"
- "低功耗开关"
- "light sleep 发包"
- "coin cell switch"
- "纽扣电池"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_ESPNOW_LIGHT_SLEEP=y`，`CONFIG_ESPNOW_LIGHT_SLEEP_DURATION`(15~120，默认 30) |
| 头文件 | `esp_sleep.h`, `espnow.h`, `espnow_ctrl.h` |
| 参考示例 | `examples/coin_cell_demo/switch/main/app_main.c` |

## 分步说明

### 1. app_main 关键配置（参考 switch 示例）

```c
#include "esp_sleep.h"
#include "espnow.h"
#include "espnow_ctrl.h"
#include "espnow_storage.h"

void app_main(void)
{
    espnow_storage_init();
    app_wifi_init();
    app_driver_init();  // 板级 + 按键

#if CONFIG_PM_ENABLE
    power_save_set(true);
#endif

    espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
    cfg.send_max_timeout = portMAX_DELAY;   // 硬币电池场景：阻塞直到发送成功
    espnow_init(&cfg);

#if CONFIG_ESPNOW_LIGHT_SLEEP
    esp_now_set_wake_window(0);   // 关闭唤醒窗口以省电
#endif
    xTaskCreate(control_task, "control_task", 4096*2, NULL, 15, NULL);
}
```

### 2. light sleep 封装（释放/获取 Wi-Fi 唤醒锁）

```c
static void set_light_sleep(uint32_t ms)
{
    esp_wifi_force_wakeup_release();           // 释放 Wi-Fi，进睡眠
    esp_sleep_enable_timer_wakeup(ms * 1000);
    esp_light_sleep_start();
    esp_wifi_force_wakeup_acquire();           // 唤醒后重新获取
    esp_sleep_wakeup_cause_t cause = esp_sleep_get_wakeup_cause();
    ESP_LOGI(TAG, "woke up cause: %d", cause); // TIMER/GPIO/UART
}
```

### 3. 状态持久化（掉电记忆灯状态）

```c
#define BULB_STATUS_KEY "bulb_key"

uint8_t status = 0;
// status: 0=OFF, 1=ON, 2=TOGGLE
#if CONFIG_EXAMPLE_SWITCH_STATUS_PERSISTED
espnow_storage_get(BULB_STATUS_KEY, &status, sizeof(status));
status ^= 1;   // 翻转
espnow_storage_set(BULB_STATUS_KEY, &status, sizeof(status));
#else
status = 2;    // TOGGLE
#endif
```

### 4. 发包任务状态机（参考 coin_cell V1 流程）

```c
#define SEND_GAP_TIME         30     // 发包间隔(ms)，给电容充电
#define LONG_PRESS_SLEEP_TIME 2000

typedef enum {
    ESPNOW_TASK_STATE_SEND_RECORD,   // 发送控制
    ESPNOW_TASK_STATE_DONE,
    ESPNOW_TASK_STATE_SEND_BIND,
    ESPNOW_TASK_STATE_BIND_DONE,
    ESPNOW_TASK_STATE_SEND_UNBIND,
    ESPNOW_TASK_STATE_UNBIND_DONE,
} espnow_task_state_t;

static espnow_task_state_t task_state = ESPNOW_TASK_STATE_SEND_RECORD;

static void control_task(void *pv)
{
    board_led_on(true);
    board_power_lock(true);
    set_light_sleep(SEND_GAP_TIME);   // 先充电

    for (;;) {
        if (task_state == ESPNOW_TASK_STATE_SEND_RECORD) {
            uint8_t status = 2;       // TOGGLE（或从 storage 读）
            espnow_ctrl_initiator_send(ESPNOW_ATTRIBUTE_KEY_1,
                                       ESPNOW_ATTRIBUTE_POWER, status);
            task_state = ESPNOW_TASK_STATE_DONE;
        }
        if (task_state == ESPNOW_TASK_STATE_DONE) {
            set_light_sleep(SEND_GAP_TIME);
            board_power_lock(false);
            board_led_on(false);
            set_light_sleep(LONG_PRESS_SLEEP_TIME);
            board_power_lock(true);
            board_led_on(true);
            task_state = ESPNOW_TASK_STATE_SEND_BIND;
        }
        if (task_state == ESPNOW_TASK_STATE_SEND_BIND) {
            espnow_ctrl_initiator_bind(ESPNOW_ATTRIBUTE_KEY_1, true);
            task_state = ESPNOW_TASK_STATE_BIND_DONE;
        }
        /* ... BIND_DONE / SEND_UNBIND / UNBIND_DONE 同理 ... */
    }
}
```

### 5. 普通按键版（非 coin-cell 板）的 wake 流程

```c
// 按键事件 → 队列 → control_task 处理（参考 switch 非 V1 分支）
if (evt_data == BUTTON_SINGLE_CLICK) {
    esp_wifi_force_wakeup_acquire();
    espnow_ctrl_initiator_send(ESPNOW_ATTRIBUTE_KEY_1, ESPNOW_ATTRIBUTE_POWER, 2);
    esp_wifi_force_wakeup_release();
} else if (evt_data == BUTTON_LONG_PRESS_UP) {
    esp_wifi_force_wakeup_acquire();
    espnow_ctrl_initiator_bind(ESPNOW_ATTRIBUTE_KEY_1, true);
    esp_wifi_force_wakeup_release();
}
// 处理完进 light sleep
esp_sleep_disable_wakeup_source(ESP_SLEEP_WAKEUP_ALL);
esp_light_sleep_start();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 发包后设备复位 | 电容未充满 | 开 `CONFIG_ESPNOW_LIGHT_SLEEP`，发包前 `set_light_sleep(SEND_GAP_TIME)` |
| light sleep 睡死 | 未 `esp_wifi_force_wakeup_acquire` | 唤醒后立即 acquire；发包前 release |
| `send_max_timeout` 太短 | 默认 3000ms，硬币电池可能不够 | 设 `cfg.send_max_timeout = portMAX_DELAY` |
| 唤醒窗口耗电 | `esp_now_set_wake_window` 默认值大 | 显式调 `esp_now_set_wake_window(0)` |
| 灯状态不一致 | 未持久化 | 用 `espnow_storage_set/get` 存 `BULB_STATUS_KEY` |
| GPIO 唤醒不工作 | 未 `gpio_wakeup_enable` + `esp_sleep_enable_gpio_wakeup` | 见 switch 非 V1 分支 |

## 参考

- `examples/coin_cell_demo/switch/main/app_main.c` — coin-cell V1/普通按键双流程
- `examples/coin_cell_demo/bulb/main/app_main.c` — 配套 bulb responder
- Kconfig：`CONFIG_ESPNOW_LIGHT_SLEEP` / `CONFIG_ESPNOW_LIGHT_SLEEP_DURATION`
