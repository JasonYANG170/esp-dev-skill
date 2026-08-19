# WAMR App Framework 定时器应用

> **适用摘要**：使用 ESP-WDF 的 WAMR App Framework，以 `on_init()`/`on_destroy()` 为入口，通过 `api_timer_create` 创建周期定时器并在回调中执行逻辑。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wdf/resources/`, source/examples in `repos/esp-wdf/`, and this recipe path `repos/esp-wdf/recipes/app_framework_timer.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "创建定时器"
- "周期任务"
- "WASM app framework 入口"
- "on_init on_destroy"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_WAMR_APP_FRAMEWORK=y`、`CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER=y` |
| sdkconfig.defaults | 见下方模板（取自 `examples/simple/timer/sdkconfig.defaults`） |
| 参考 | `examples/simple/timer/` |

## 分步说明

### 1. sdkconfig.defaults（直接取自 timer 示例）

```
CONFIG_WAMR_APP_FRAMEWORK=y
CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER=y
```

> 开启 `CONFIG_WAMR_APP_FRAMEWORK` 后，构建系统（`CMakeLists.txt`）自动导出 `on_init`、`on_destroy`；开启 `EXPORT_TIMER` 才会导出 `on_timer_callback`，否则定时器回调不会被宿主触发。

### 2. 应用代码（取自 examples/simple/timer/main/timer.c）

```c
#include "wasm_app.h"
#include "wa-inc/timer_wasm_app.h"

/* 用户全局变量 */
static int num = 0;

/* 定时器回调 */
void
timer1_update(user_timer_t timer)
{
    printf("Timer update %d\n", num++);
}

void
on_init()
{
    user_timer_t timer;

    /* 创建周期定时器：interval=1000ms，is_period=true，auto_start=false */
    timer = api_timer_create(1000, true, false, timer1_update);
    /* 用新 interval 重启并启动 */
    api_timer_restart(timer, 1000);
}

void
on_destroy()
{
    /* 实际的销毁工作（停定时器、关传感器）由 wasm app library 版本的 on_destroy() 完成 */
}
```

### 3. 头文件包含要点

- `wasm_app.h` —— 框架主头（声明 `on_init`/`on_destroy` 期望）。
- `wa-inc/timer_wasm_app.h` —— 定时器 API：`api_timer_create/cancel/restart`、`user_timer_t`、`on_user_timer_update_f`。

### 4. 单次定时器

将 `is_period` 设为 `false` 即为单次；若希望创建即启动，把第三个参数 `auto_start` 设为 `true`，则不必再调 `api_timer_restart`：

```c
/* 单次、立即启动 */
timer = api_timer_create(2000, false, true, one_shot_cb);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 定时器回调从不触发 | 未开启 `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER` | sdkconfig.defaults 加 `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER=y` |
| `on_init` 找不到 / 未导出 | 未开启 `CONFIG_WAMR_APP_FRAMEWORK` | 加 `CONFIG_WAMR_APP_FRAMEWORK=y` |
| 同时定义了 `main()` | App Framework 与标准 main 冲突 | 删掉 `main()`，只保留 `on_init/on_destroy` |
| 定时器不周期 | `is_period` 传了 `false` | 周期任务传 `true` |

## 参考

- `examples/simple/timer/main/timer.c`
- `examples/simple/timer/sdkconfig.defaults`
- `components/wamr/app-framework/include/wa-inc/timer_wasm_app.h`
- `resources/api_reference.md` —— 第 1.2 节 定时器 API
- `resources/pitfalls.md` —— 第 3 条 入口形态二选一
