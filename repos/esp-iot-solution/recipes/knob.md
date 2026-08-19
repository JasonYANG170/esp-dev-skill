# 旋转编码器旋钮（knob）

> **适用摘要**: 使用 `knob` 组件接入 AB 相旋转编码器，创建旋钮句柄、注册左右旋/上下限/归零事件回调、读取累计计数值。组件基于 GPIO 中断实现，支持 power-save。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-iot-solution/resources/`, source/examples in `repos/esp-iot-solution/`, and this recipe path `repos/esp-iot-solution/recipes/knob.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "旋转编码器"
- "旋钮"
- "iot_knob"
- "编码器左右旋事件"
- "音量旋钮"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 环境 | ESP-IDF v5.3+ |
| 组件依赖 | `espressif/knob` |
| 参考示例 | `examples/get-started/knob_power_save/main/knob_power_save.c` |

## 分步说明

### 1. 添加组件依赖

```bash
idf.py add-dependency "espressif/knob"
```

### 2. 创建旋钮并注册事件

```c
#include "iot_knob.h"

#define ENCODER_A_GPIO   1
#define ENCODER_B_GPIO   2

static knob_handle_t s_knob = NULL;
static const char *TAG = "knob";

const char *knob_event_table[] = {
    "KNOB_LEFT", "KNOB_RIGHT", "KNOB_H_LIM", "KNOB_L_LIM", "KNOB_ZERO",
};

static void knob_event_cb(void *arg, void *data)
{
    // arg = knob handle，data = 注册时传入的 usr_data（这里用作事件枚举）
    ESP_LOGI(TAG, "knob event %s, count %d",
             knob_event_table[(int)data],
             iot_knob_get_count_value((knob_handle_t)arg));
}

void knob_init(void)
{
    knob_config_t cfg = {
        .default_direction = 0,            // 0: 正向递增；1: 反向
        .gpio_encoder_a = ENCODER_A_GPIO,
        .gpio_encoder_b = ENCODER_B_GPIO,
        .enable_power_save = false,        // 低功耗时设 true
    };
    s_knob = iot_knob_create(&cfg);
    assert(s_knob);

    esp_err_t err = iot_knob_register_cb(s_knob, KNOB_LEFT,  knob_event_cb, (void *)KNOB_LEFT);
    err |= iot_knob_register_cb(s_knob, KNOB_RIGHT, knob_event_cb, (void *)KNOB_RIGHT);
    err |= iot_knob_register_cb(s_knob, KNOB_H_LIM, knob_event_cb, (void *)KNOB_H_LIM);
    err |= iot_knob_register_cb(s_knob, KNOB_L_LIM, knob_event_cb, (void *)KNOB_L_LIM);
    err |= iot_knob_register_cb(s_knob, KNOB_ZERO,  knob_event_cb, (void *)KNOB_ZERO);
    ESP_ERROR_CHECK(err);
}
```

### 3. 读取与清零计数值

```c
int count = iot_knob_get_count_value(s_knob);   // 累计计数
iot_knob_clear_count_value(s_knob);              // 清零

// 当前事件（轮询模式）
knob_event_t ev = iot_knob_get_event(s_knob);
```

> `KNOB_H_LIM` / `KNOB_L_LIM` 分别在计数达到上限/下限时触发；`KNOB_ZERO` 在计数回到 0 时触发。上下限默认由组件内部逻辑决定。

### 4. 启停定时器（用于配合 sleep）

```c
iot_knob_stop();      // 停止扫描定时器
iot_knob_resume();    // 恢复
```

### 5. 生命周期

```c
iot_knob_unregister_cb(s_knob, KNOB_LEFT);
iot_knob_delete(s_knob);
```

## knob_event_t 枚举

| 事件 | 含义 |
|---|---|
| `KNOB_LEFT` | 左旋 |
| `KNOB_RIGHT` | 右旋 |
| `KNOB_H_LIM` | 计数达到上限 |
| `KNOB_L_LIM` | 计数达到下限 |
| `KNOB_ZERO` | 计数回到 0 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直只触发 `KNOB_LEFT` 或 `KNOB_RIGHT` | A/B 相接反 或 `default_direction` 设错 | 互换两引脚，或改 `default_direction` |
| 旋一格触发多次 | 编码器机械抖动 / 没接硬件去抖 | 软件层加 RC 滤波或确认编码器为正交输出 |
| 回调里 `(int)data` 取不到事件 | 注册时 usr_data 传错 | 注册时把 `KNOB_LEFT` 等枚举作为 usr_data 传入 |
| 计数方向相反 | `default_direction` 与硬件转向不符 | `default_direction` 取 0/1 切换 |

## 参考

- 组件头文件：`components/knob/include/iot_knob.h`
- 真实示例：`examples/get-started/knob_power_save/main/knob_power_save.c`
- 示例 README：`examples/get-started/knob_power_save/README.md`
