# 系统事件处理与事件上报

> **适用摘要**: 在 `app_driver_event_handler()` 里 switch 处理 `LOW_CODE_EVENT_*` 系统事件（配网/网络/OTA/就绪/识别），并用 `low_code_event_to_system()` 主动上报事件（如工厂复位）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/event_handling.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "处理配网/网络/OTA 事件"
- "按键触发工厂复位"
- "low_code_event_to_system"
- "LOW_CODE_EVENT_FACTORY_RESET"
- "事件指示灯效"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `low_code.h`（`low_code_event_t`、`low_code_event_type_t`） |
| 参考产品 | `products/socket`（带灯效指示）、`products/light_cw_pwm`（CW 灯效） |

## 分步说明

### 1. 事件结构

```c
typedef struct low_code_event {
    low_code_event_type_t event_type;   /* 见 low_code_event_type_t */
    int event_data_size;                 /* event_data 字节数 */
    void *event_data;                    /* 如测试模式 subtype */
} low_code_event_t;

int low_code_event_to_system(low_code_event_t *event);   /* 上报事件 */
```

### 2. 处理事件：带灯效指示（取自 `products/socket/main/app_driver.cpp`）

```cpp
int app_driver_event_handler(low_code_event_t *event)
{
    printf("%s: Received event: %d\n", TAG, event->event_type);

    /* 灯效默认配置（按灯类型选 COLOR 或 WHITE） */
    light_effect_config_t effect_config = {
        .type = LIGHT_EFFECT_INVALID,
        .mode = LIGHT_WORK_MODE_COLOR,   /* 单通道 LED 用 COLOR */
        .max_brightness = 100,
        .min_brightness = 10
    };

    switch (event->event_type) {
        case LOW_CODE_EVENT_SETUP_MODE_START:
            effect_config.type = LIGHT_EFFECT_BLINK;
            light_driver_effect_start(&effect_config, 2000, 120000); /* 周期2s, 总120s */
            break;
        case LOW_CODE_EVENT_SETUP_MODE_END:
            light_driver_effect_stop();
            break;
        case LOW_CODE_EVENT_SETUP_DEVICE_CONNECTED:
        case LOW_CODE_EVENT_SETUP_STARTED:
        case LOW_CODE_EVENT_SETUP_SUCCESSFUL:
        case LOW_CODE_EVENT_SETUP_FAILED:
        case LOW_CODE_EVENT_NETWORK_CONNECTED:
        case LOW_CODE_EVENT_NETWORK_DISCONNECTED:
        case LOW_CODE_EVENT_OTA_STARTED:
        case LOW_CODE_EVENT_OTA_STOPPED:
        case LOW_CODE_EVENT_READY:
        case LOW_CODE_EVENT_IDENTIFICATION_START:
        case LOW_CODE_EVENT_IDENTIFICATION_STOP:
            /* 日志/按需加指示 */
            break;
        case LOW_CODE_EVENT_TEST_MODE_LOW_CODE: {
            int subtype = *((int*)(event->event_data));
            printf("%s: Low code test mode subtype: %d\n", TAG, subtype);
            break;
        }
        case LOW_CODE_EVENT_TEST_MODE_COMMON:
        case LOW_CODE_EVENT_TEST_MODE_BLE:
        case LOW_CODE_EVENT_TEST_MODE_SNIFFER:
            break;
        default:
            printf("%s: Unhandled event type: %d\n", TAG, event->event_type);
            break;
    }
    return 0;
}
```

> CW/PWM 灯用 `LIGHT_WORK_MODE_WHITE`；WS2812 单灯用 `LIGHT_WORK_MODE_COLOR`（见 `products/light_cw_pwm`）。

### 3. 上报事件：按键长按触发工厂复位

```cpp
static void app_driver_trigger_factory_reset_button_callback(void *arg, void *data)
{
    low_code_event_t event = {
        .event_type = LOW_CODE_EVENT_FACTORY_RESET
    };
    low_code_event_to_system(&event);
    printf("%s: Factory reset triggered\n", TAG);
}
```

> 工厂复位**不要**自己擦 Matter 凭证分区，交给 HP Core 的系统层处理。

### 4. 测试模式事件的数据读取

测试模式事件（`LOW_CODE_EVENT_TEST_MODE_LOW_CODE/_COMMON/_BLE/_SNIFFER`）的 `event_data` 指向一个 int subtype（见 `product_config.json` 的 `test_mode[].subtype`），读取方式：

```cpp
int subtype = *((int*)(event->event_data));
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 灯效不闪 | setup 阶段未初始化 light_driver | 在 app_driver_init 先 light_driver_init |
| 灯效停不掉 | 未调 effect_stop | SETUP_MODE_END 里调 `light_driver_effect_stop()` |
| 自己擦 flash 想复位 | 误以为 LP Core 能复位 | 上报 `LOW_CODE_EVENT_FACTORY_RESET` 交系统处理 |
| 测试模式 subtype 读错 | 直接当 event_type | 读 `*((int*)event->event_data)` |
| 漏处理某个事件 | switch 没覆盖 | 加 default 分支并日志 |

## 参考

- `products/socket/main/app_driver.cpp`（完整事件 switch + 灯效）
- `products/light_cw_pwm/main/app_driver.cpp`（CW 灯 WHITE 模式）
- `products/template/main/app_driver.cpp`（最小事件处理）
- `components/low_code/low_code.h`（`low_code_event_type_t` 全集）
