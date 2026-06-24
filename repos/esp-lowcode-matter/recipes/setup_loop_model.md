# setup/loop 编程骨架

> **适用摘要**: 理解并实现 LowCode 在 LP Core 上的 `system_setup()` → `setup()` → `while{system_loop();loop();}` 骨架与回调注册。

## 触发意图

- "lowcode 主循环怎么写"
- "system_setup / system_loop"
- "low_code_register_callbacks"
- "app_main.cpp 结构"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考产品 | `products/template/main/app_main.cpp`、`products/socket/main/app_main.cpp` |
| 头文件 | `system.h`、`low_code.h` |

## 分步说明

### 完整骨架（取自 `products/template/main/app_main.cpp`）

```cpp
#include <stdio.h>
#include <system.h>
#include <low_code.h>
#include "app_priv.h"

static const char *TAG = "app_main";

/* —— 应用初始化 —— */
static void setup()
{
    /* 1. 先注册回调（系统下发的事件/特性才会抵达应用） */
    low_code_register_callbacks(feature_update_from_system, event_from_system);

    /* 2. 再初始化驱动 */
    app_driver_init();
}

/* —— 主循环里只负责拉取消息，回调里做业务 —— */
static void loop()
{
    low_code_get_feature_update_from_system();
    low_code_get_event_from_system();
}

/* —— 系统下发特性更新回调 —— */
int feature_update_from_system(low_code_feature_data_t *data)
{
    uint16_t endpoint_id = data->details.endpoint_id;
    uint32_t feature_id  = data->details.feature_id;
    printf("%s: Feature update: endpoint: %u, feature: %lu\n", TAG, endpoint_id, feature_id);
    return app_driver_feature_update();
}

/* —— 系统下发事件回调 —— */
int event_from_system(low_code_event_t *event)
{
    return app_driver_event_handler(event);   /* 处理 low_code_event_type_t */
}

/* —— 入口 —— */
extern "C" int main()
{
    printf("%s: Starting low code\n", TAG);

    /* 系统预初始化：必须最先调用，且永远存在 */
    system_setup();
    setup();

    /* 循环 */
    while (1) {
        system_loop();
        loop();
    }
    return 0;
}
```

### 关键 API（`low_code.h`）

```cpp
/* 注册应用回调（feature 下发 / event 下发） */
int low_code_register_callbacks(low_code_feature_update_callback_t feature_update_from_system,
                                low_code_event_callback_t event_from_system);

/* 在 loop() 中拉取消息，触发上面注册的回调 */
int low_code_get_feature_update_from_system();
int low_code_get_event_from_system();
```

### 关键 API（`system.h`）

```c
void system_setup(void);   /* 系统初始化，main() 中首发 */
void system_loop(void);    /* 系统任务/定时器推进，每轮循环调用 */
```

### 顺序铁律

1. `main()` 第一条应用语句必须是 `system_setup()`
2. `setup()` 内先 `low_code_register_callbacks(...)` 再 `app_driver_init()`
3. `while(1)` 里 `system_loop()` 与 `loop()` 都不可省（`loop()` 内两个 `low_code_get_*` 也不可省）

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 收不到任何事件/特性 | 未注册回调或顺序错 | setup() 内、驱动 init 之前注册回调 |
| 定时器/系统任务不跑 | 漏 `system_loop()` | while 内先 `system_loop()` 再 `loop()` |
| main 返回后系统挂死 | 没有 `while(1)` | 必须有无限循环 |
| 回调里拿不到 data | `loop()` 未调 `low_code_get_*_from_system()` | loop() 内补两个拉取调用 |

## 参考

- `products/template/main/app_main.cpp`
- `products/socket/main/app_main.cpp`
- `docs/programmer_model.md`
