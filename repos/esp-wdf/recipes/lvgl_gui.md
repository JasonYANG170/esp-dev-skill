# LVGL 图形界面（WASM 适配）

> **适用摘要**：WASM 应用用 ESP-WDF 提供的 LVGL WASM 适配 API 构建界面——用 `lvgl_init`/`lvgl_lock`/`lvgl_unlock` 异步初始化，用 `lv_obj_get_data` 等访问器替代直接解引用结构指针。

## 触发意图

- "WASM GUI"
- "LVGL 界面"
- "lvgl_lock unlock"
- "out of bounds memory access"

## 前置条件

| 条件 | 要求 |
|---|---|
| Kconfig | `CONFIG_WDF_EXT_WASM_APP_LVGL=y`（默认开） |
| 头文件 | `"esp_lvgl.h"`（含访问器 API），`"lvgl/lvgl.h"` 等 |
| 宿主 | 运行于支持 LVGL 的 ESP-WASMachine BSP（如 esp-box、esp32_p4_function_ev_board） |
| 参考 | `examples/gui/lv_demos/`、`components/extended_wasm_app/esp_lvgl/include/esp_lvgl.h` |

## 分步说明

### 1. 异步初始化与加锁（来自 README 5.2）

原生 LVGL 不支持多线程，动态安装 WASM 应用属异步操作，故操作 UI 前必须 `lvgl_lock()`，��成后 `lvgl_unlock()`。同时**不要**在应用层写 `while(1){ lv_task_handler(); }`。

```c
#include "esp_lvgl.h"

void on_init(void)
{
    if (!lvgl_is_inited()) {
        lvgl_init();            /* 初始化 LVGL */
    }

    lvgl_lock();                /* 暂停 LVGL 调度 */
    user_init_lvgl_menu();      /* 配置用户菜单（自实现） */
    lvgl_unlock();              /* 恢复 LVGL 调度 */
}
```

### 2. 不要直接访问结构成员（来自 README 5.1）

虚拟机分配的结构指针（如 `lv_obj_t`、`lv_timer_t`）不能直接解引用，否则触发 `out of bounds memory access`。改用访问器：

```c
/* ❌ 错误：直接访问 timer->user_data */
static void timer_callback(lv_timer_t *timer)
{
    void *user_data = timer->user_data;   /* 越界访问 */
}

/* ✅ 正确：用访问器 */
static void timer_callback(lv_timer_t *timer)
{
    void *user_data = lv_timer_get_user_data(timer);
}
```

读取对象坐标：

```c
/* ❌ 错误：直接访问 obj->coords.y2 */
static void init_menu(void)
{
    lv_obj_set_size(scene_bg, w, h - subtitle->coords.y2 - LV_DPI_DEF / 30);
}

/* ✅ 正确：用 lv_obj_get_data 读取到 lv_area_t */
static void init_menu(void)
{
    lv_area_t obj_area;
    lv_obj_get_data(subtitle, LV_OBJ_COORDS, &obj_area, sizeof(obj_area));
    lv_obj_set_size(scene_bg, w, h - obj_area.y2 - LV_DPI_DEF / 30);
}
```

### 3. 访问器 API 速查（esp_lvgl.h）

| 函数 | 用途 |
|---|---|
| `lvgl_is_inited(void)` | LVGL 是否已初始化 |
| `lvgl_init(void)` | 初始化 LVGL（异步） |
| `lvgl_deinit(void)` | 反初始化 |
| `lvgl_lock(void)` / `lvgl_unlock(void)` | 暂停/恢复 LVGL 调度 |
| `lv_obj_get_data(obj, type, pdata, n)` | 读对象数据（`type=LV_OBJ_COORDS` 读坐标） |
| `lv_timer_get_user_data(timer)` | 取定时器用户数据 |
| `lv_disp_get_data(disp, pdata, n)` | 读显示设备数据 |
| `lv_release_variable(void)` | 释放变量 |

数据类型常量：`LV_OBJ_COORDS=0`、`LV_OBJ_DRAW_PART_DSC_*`（0–12）。LVGL 配色/字节序由 `LV_COLOR_16_SWAP`（esp-box 启用、esp32_p4_function_ev_board 关闭）与可选 `USING_CUSTOMER_LV_CONF_H` 控制。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `out of bounds memory access` | 直接 `obj->member` / `timer->user_data` | 改用 `lv_obj_get_data`/`lv_timer_get_user_data` 等访问器 |
| UI 操作错乱/崩溃 | 未加锁就操作 UI | `lvgl_lock()` … 操作 … `lvgl_unlock()` |
| 写了 `lv_task_handler` 死循环 | 应用层不该驱动内核 | 删除该循环，内核由宿主驱动 |
| 字节序颠倒（RGB565） | `LV_COLOR_16_SWAP` 与 BSP 不匹配 | esp-box 启用；esp32_p4_function_ev_board 关闭 |

## 参考

- `components/extended_wasm_app/esp_lvgl/include/esp_lvgl.h`
- `examples/gui/lv_demos/README.md`
- `README.md` 第 5.1、5.2 节
- `resources/api_reference.md` —— 第 5.1 节 LVGL
- `resources/pitfalls.md` —— 第 1、2 条
