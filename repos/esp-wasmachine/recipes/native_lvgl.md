# WASM Native LVGL 与 BSP 显示

> **适用摘要**: 在 ESP32-S3-BOX 或 ESP32-P4-Function-EV-Board 上启用 LVGL native，注册 BSP 显示回调，让 WASM 应用调用 LVGL 绘制 UI。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wasmachine/resources/`, source/examples in `repos/esp-wasmachine/`, and this recipe path `repos/esp-wasmachine/recipes/native_lvgl.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "WASM 应用画 UI"
- "LVGL native 怎么开"
- "S3-BOX 跑 wasm lvgl"
- "wm_ext_wasm_native_lvgl_register_ops"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标 | `esp32s3`（esp-box）或 `esp32p4`（esp32_p4_function_ev_board） |
| Kconfig | `CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL=y`（默认 n） |
| 依赖 | `esp-box 3.1.*`（S3）或 `esp32_p4_function_ev_board 5.0.*`（P4），`lvgl/lvgl 9.3.0`（见 `main/idf_component.yml`） |
| BSP ops | 固件 `app_main` 必须调用 `wm_ext_wasm_native_lvgl_register_ops(...)` |

## 分步说明

### 1. 启用并选板（sdkconfig）

```ini
CONFIG_WASMACHINE_WASM_EXT_NATIVE=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL=y
# 可选：把 LVGL 内存分配到 WASM 应用堆（需要 configNUM_THREAD_LOCAL_STORAGE_POINTERS >= 3）
# CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL_USE_WASM_HEAP=y
```

选板（叠加板级默认）：

```bash
# S3-BOX
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp-box" set-target esp32s3
# P4-Function-EV-Board
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.esp32_p4_function_ev_board" set-target esp32p4
```

### 2. 固件侧注册 BSP ops

来源 `examples/wasmachine/main/wm_main.c` 的 `bsp_init()` / `bsp_display_config()`：

```c
#if CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL
#include "wm_ext_wasm_native.h"
#include "bsp/esp-bsp.h"

static void bsp_display_config(void)
{
#if CONFIG_IDF_TARGET_ESP32S3
    bsp_i2c_init();
    bsp_display_cfg_t cfg = {
        .lvgl_port_cfg = ESP_LVGL_PORT_INIT_CONFIG(),
        .buffer_size = BSP_LCD_H_RES * CONFIG_BSP_LCD_DRAW_BUF_HEIGHT,
#if CONFIG_BSP_LCD_DRAW_BUF_DOUBLE
        .double_buffer = 1,
#else
        .double_buffer = 0,
#endif
        .flags = { .buff_dma = true, .buff_spiram = true, }
    };
#else
    /* ESP32-P4：含 MIPI-DSI / HDMI 配置，见 wm_main.c 原文 */
    bsp_display_cfg_t cfg = { /* ... */ };
#endif
    cfg.lvgl_port_cfg.task_stack = 16384;
    assert(bsp_display_start_with_config(&cfg));
}

static void bsp_init(void)
{
    bsp_display_config();
    wm_ext_wasm_native_lvgl_ops_t lvgl_ops = {
        .backlight_on  = bsp_display_backlight_on,
        .backlight_off = bsp_display_backlight_off,
        .lock          = bsp_display_lock,
        .unlock        = bsp_display_unlock,
    };
    ESP_ERROR_CHECK(wm_ext_wasm_native_lvgl_register_ops(&lvgl_ops));
}
#endif
```

### 3. `wm_ext_wasm_native_lvgl_ops_t` 结构

来源 `components/wasmachine_ext_wasm_native/include/wm_ext_wasm_native.h`：

```c
typedef struct wm_ext_wasm_native_lvgl_ops {
    esp_err_t (*backlight_on)(void);
    esp_err_t (*backlight_off)(void);
    bool      (*lock)(uint32_t timeout_ms);
    void      (*unlock)(void);
} wm_ext_wasm_native_lvgl_ops_t;

esp_err_t wm_ext_wasm_native_lvgl_register_ops(wm_ext_wasm_native_lvgl_ops_t *ops);
```

### 4. WASM 应用侧 LVGL 调用

LVGL native 与 HTTP 一样，是单个 import 按函数 ID 派发。ID 常量在 `components/wasmachine_ext_wasm_native/private_include/wm_ext_wasm_native_lvgl.h`，共 460+ 个，覆盖 LVGL v9 主流 API。常用 ID 举例：

| ID 宏 | 值 | 映射 |
|---|---|---|
| `LV_OBJ_CREATE` | 11 | 创建对象 |
| `LV_LABEL_CREATE` | 31 | 创建标签 |
| `LV_LABEL_SET_TEXT` | 32 | 设置文本 |
| `LV_OBJ_ALIGN` | 15 | 对齐 |
| `LV_OBJ_SET_SIZE` | 14 | 设尺寸 |
| `LV_BTN_CREATE` | 89 | 创建按钮 |
| `LV_SLIDER_CREATE` | 99 | 创建滑块 |
| `LV_CHART_CREATE` | 108 | 创建图表 |
| `LV_SCREEN_ACTIVE` | 316 | 取当前活动屏 |
| `LV_TIMER_CREATE` | 38 | 创建定时器 |
| `LV_ANIM_START` | 73 | 启动动画 |

LVGL 版本戳：`WM_LV_VERSION_MAJOR/MINOR/PATCH` = `1.0.0`。

### 5. WASM 内存隔离（可选）

`CONFIG_WASMACHINE_WASM_EXT_NATIVE_LVGL_USE_WASM_HEAP=y` 会把 LVGL 内存分配重定向到 WASM 应用堆，实现 app 间内存隔离并在卸载时清理。前提：FreeRTOS `configNUM_THREAD_LOCAL_STORAGE_POINTERS >= 3`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 屏幕不亮 / WASM 调 LVGL 无效 | 未注册 BSP ops | 在 `bsp_init` 调 `wm_ext_wasm_native_lvgl_register_ops` |
| 非 S3/P4 链接失败 | LVGL/BSP 仅 S3、P4 引入 | 在别的目标上关掉 `_LVGL` |
| `USE_WASM_HEAP` 后崩溃 | TLS 指针槽不足 | 调 `configNUM_THREAD_LOCAL_STORAGE_POINTERS >= 3` |
| BOX 编译缺 BSP | 没叠加 `sdkconfig.esp-box` | 用 `-DSDKCONFIG_DEFAULTS=...` 重新 `set-target` |

## 参考

- `examples/wasmachine/main/wm_main.c` — `bsp_display_config` / `bsp_init`
- `components/wasmachine_ext_wasm_native/include/wm_ext_wasm_native.h` — ops 结构
- `components/wasmachine_ext_wasm_native/private_include/wm_ext_wasm_native_lvgl.h` — 全部 LVGL 函数 ID
- `examples/wasmachine/main/idf_component.yml` — `esp-box` / `esp32_p4_function_ev_board` / `lvgl` 依赖规则
