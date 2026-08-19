# 设备控制：Responder（灯/插座）侧

> **适用摘要**: 在 responder 设备上进入绑定窗口、注册控制数据回调、维护绑定列表（持久化到 NVS），并依据控制数据执行动作（参考 `examples/control` 与 `examples/coin_cell_demo/bulb`）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "responder 接收控制"
- "灯端绑定"
- "维护绑定列表 bindlist"
- "espnow_ctrl_responder"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `espnow_ctrl.h` |
| 参考示例 | `examples/control/main/app_main.c`, `examples/coin_cell_demo/bulb/main/app_main.c` |

## 分步说明

### 1. 进入绑定窗口（带超时与 RSSI 阈值）

```c
#include "espnow_ctrl.h"

// wait_ms: 绑定窗口；rssi: 最小 bind 帧 RSSI；cb: 收到 bind 帧时回调（可 NULL）
// cb 返回 true 且未超时且 RSSI 达标，responder 才执行绑定/解绑
ESP_ERROR_CHECK(espnow_ctrl_responder_bind(30 * 1000, -55, NULL));
```

> `espnow_ctrl_bind_cb_t` 签名：`bool cb(espnow_attribute_t initiator_attribute, uint8_t mac[6], int8_t rssi)`。返回 true 表示接受该绑定帧。

### 2. 注册控制数据回调

```c
static void app_ctrl_data_cb(espnow_attribute_t initiator_attribute,
                             espnow_attribute_t responder_attribute,
                             uint32_t responder_value)
{
    ESP_LOGI(TAG, "ctrl: initiator %d, responder %d, value %" PRIu32,
             initiator_attribute, responder_attribute, responder_value);

    if (responder_attribute == ESPNOW_ATTRIBUTE_POWER) {
        if (responder_value) app_led_set_color(255, 255, 255);
        else                 app_led_set_color(0, 0, 0);
    }
}

// 注册（内部会 espnow_set_config_for_data_type 开启接收）
espnow_ctrl_responder_data(app_ctrl_data_cb);
```

### 3. 完整 responder 初始化（参考 control 示例）

```c
static void app_responder_init(void)
{
    ESP_ERROR_CHECK(espnow_ctrl_responder_bind(30 * 1000, -55, NULL));
    espnow_ctrl_responder_data(app_ctrl_data_cb);
}

void app_main(void)
{
    espnow_storage_init();
    app_wifi_init();

    espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
    cfg.receive_enable.control_bind = 1;
    cfg.receive_enable.control_data = 1;
    espnow_init(&cfg);

    esp_event_handler_register(ESP_EVENT_ESPNOW, ESP_EVENT_ANY_ID,
                               app_espnow_event_handler, NULL);
    app_responder_init();
}
```

### 4. 绑定列表管理（持久化到 flash）

```c
espnow_ctrl_bind_info_t list[ESPNOW_BIND_LIST_MAX_SIZE];  // 32
size_t size = ESPNOW_BIND_LIST_MAX_SIZE;

// 读取当前绑定列表
espnow_ctrl_responder_get_bindlist(list, &size);

// 手动添加一条绑定（持久化）
espnow_ctrl_bind_info_t info = {
    .mac = {0xAA,0xBB,0xCC,0xDD,0xEE,0xFF},
    .initiator_attribute = ESPNOW_ATTRIBUTE_KEY_1,
};
espnow_ctrl_responder_set_bindlist(&info);

// 删除一条绑定
espnow_ctrl_responder_remove_bindlist(&info);

// 清空全部绑定
espnow_ctrl_responder_clear_bindlist(void);
```

### 5. 事件处理（绑定/解绑/错误）

```c
static void app_espnow_event_handler(void *a, esp_event_base_t base, int32_t id, void *data)
{
    if (base != ESP_EVENT_ESPNOW) return;
    switch (id) {
    case ESP_EVENT_ESPNOW_CTRL_BIND:
        { espnow_ctrl_bind_info_t *i = data;
          ESP_LOGI(TAG, "bound " MACSTR, MAC2STR(i->mac)); }
        break;
    case ESP_EVENT_ESPNOW_CTRL_UNBIND:
        { espnow_ctrl_bind_info_t *i = data;
          ESP_LOGI(TAG, "unbound " MACSTR, MAC2STR(i->mac)); }
        break;
    case ESP_EVENT_ESPNOW_CTRL_BIND_ERROR:
        { espnow_ctrl_bind_error_t *e = data;
          // ESPNOW_BIND_ERROR_TIMEOUT / _RSSI / _LIST_FULL
          ESP_LOGW(TAG, "bind error %d", *e); }
        break;
    }
}
```

### 6. 状态持久化（参考 bulb 示例：掉电记忆灯状态）

```c
#include "espnow_storage.h"
#define BULB_STATUS_KEY "bulb_key"
static uint32_t s_bulb_status = 0;

static void app_save_status(void) {
    espnow_storage_set(BULB_STATUS_KEY, &s_bulb_status, sizeof(s_bulb_status));
}
static void app_load_status(void) {
    espnow_storage_get(BULB_STATUS_KEY, &s_bulb_status, sizeof(s_bulb_status));
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 绑定不上 | `espnow_ctrl_responder_bind` 未调或已超时 | 重启或延长 `wait_ms`；确认 initiator 已发 bind |
| bind 帧被忽略 | RSSI 低于阈值 | 调低 `rssi` 参数（如 -70）或靠近设备 |
| bindlist 满 | 超过 32 条 | `espnow_ctrl_responder_clear_bindlist` |
| 重启后绑定丢失 | 误用 `remove` 替代 `set` | `set_bindlist` 才持久化；重启后 `get_bindlist` 应能读回 |
| 控制回调不触发 | `receive_enable.control_data` 未开 或 未 `espnow_ctrl_responder_data` 注册 | 二者其一即可（API 内部会 set_config） |

## 参考

- `examples/control/main/app_main.c` — responder 绑定+控制回调+LED
- `examples/coin_cell_demo/bulb/main/app_main.c` — bulb responder，状态持久化
- `resources/api_reference.md` — `espnow_ctrl_responder_bind` / `data` / `*_bindlist` 系列
