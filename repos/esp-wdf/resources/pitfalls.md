# ESP-WDF 常见陷阱（汇总）

> 全部来自仓库 README.md / README_CN.md（第 5 节"开发注意事项"）以及示例源码与构建系统的真实约束。

## 1. 直接解引用虚拟机分配的结构指针会崩溃（沙箱越界）

WebAssembly 沙箱禁止应用直接访问未授权内存。若 `attr_container_t`、`lv_obj_t` 等结构指针由虚拟机分配，直接访问其成员会触发 `out of bounds memory access`。

- **错误**：`timer->user_data`、`subtitle->coords.y2`
- **正确**：使用访问器 `lv_timer_get_user_data(timer)`、`lv_obj_get_data(obj, LV_OBJ_COORDS, &area, sizeof(area))`。
- 来源：README 5.1、`esp_lvgl.h`。

## 2. LVGL 必须用 lock/unlock，且无需应用层调用 lv_task_handler

动态安装/运行 WASM 应用是异步的，原生 LVGL 不支持多线程。操作 UI 前必须 `lvgl_lock()`，操作完 `lvgl_unlock()`。同时**不要**在应用层写 `while(1){ lv_task_handler(); }` 循环——内核调度由宿主驱动。

```c
lvgl_init();
lvgl_lock();
user_init_lvgl_menu();
lvgl_unlock();
```
- 来源：README 5.2、`esp_lvgl.h`。

## 3. App Framework 与标准 main 二选一，由 Kconfig 决定

开启 `CONFIG_WAMR_APP_FRAMEWORK=y` 时，应用必须实现 `on_init()`/`on_destroy()`，且不要同时定义 `main()`。反之则用 `int main(void)`。混淆会导致导出符号缺失或入口未定义。需要用到请求/响应或定时器原生回调时，还需分别打开 `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_REQUEST_RESPONSE` / `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER`，否则对应 API（`api_register_resource_handler`、`api_timer_create`）回调不会被宿主触发。

- 来源：`CMakeLists.txt` 第 43–55 行、`examples/simple/timer/sdkconfig.defaults`。

## 4. 网络/sockets 示例的 sdkconfig.defaults 必须三件齐备

`examples/protocols/sockets/tcp_client/sdkconfig.defaults`：`CONFIG_WAMR_APP_FRAMEWORK=y` + `CONFIG_COMPILER_WASI_NO_USE_STDLIB=n` + `CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y`。漏掉 `NO_USE_STDLIB=n` 会导致 libc-wasi 的 socket 符号缺失；漏掉 `USE_SHARED_MEMORY=y` 多线程 socket 会出错。

- 来源：sockets tcp_client/sdkconfig.defaults。

## 5. 外设通过 VFS 设备节点 + ioctl 访问，不是直接驱动 API

WASM 应用不能调用 `gpio_set_level`、`i2c_master_*` 等宿主端 ESP-IDF 驱动 API。必须 `open("/dev/...")` 拿到 fd，再用 `ioctl(fd, CMD, &cfg)` 配置、`read/write` 传输、`close` 释放。误用宿主 API 会编译失败或运行时找不到符号。

- 来源：所有 `examples/peripherals/*` 示例、`ioctl/esp_*_ioctl.h`。

## 6. attr_container 写入函数首参是二级指针

`attr_container_set_string(&event, ...)` 传入的是 `attr_container_t **`，因为容器内部可能重建。写成 `attr_container_set_string(event, ...)`（一级指针）会编译报错或写错地址。

- 来源：`attr_container.h`、`event_publisher.c`。

## 7. set_response / init_request / make_response_for_request 的指针不能为 NULL

这三个辅助函数内部不解引用检查，传入 NULL 会崩溃。声明本地结构体时用 `response_t response[1];`（数组形式取址天然非空）。

- 来源：`shared_utils.h` 函数注释 `@warning`、`request_handler.c`。

## 8. 请求 URL 的跨应用寻址前缀

向另一个 WASM 应用的资源发请求时，URL 前缀 `/app/<app_name>`：`request_sender` 示例用 `"/app/request_handler/url1"` 寻址名为 `request_handler` 的应用；不带前缀的 `"url1"` 则是通用请求。

- 来源：`examples/simple/request_sender/main/request_sender.c`。

## 9. 应用不能在根目录直接构建

根 `CMakeLists.txt` 显式检测 `CMAKE_CURRENT_LIST_DIR STREQUAL CMAKE_SOURCE_DIR` 并 `FATAL_ERROR`。必须 `cd` 到 `examples/<name>` 再 `idf.py build`。

- 来源：`CMakeLists.txt` 第 4–8 行、README 第 3 节。

## 10. 编译产物区分 WASM 与 AOT 两条命令

- `idf.py build` → 生成 `build/<name>.wasm`（字节码）。
- `idf.py build aot` → 生成 `build/<name>.aot`（AOT，需自行构建 `wamrc` 工具）。

误把 AOT 固件当作 WASM 安装，或反之，会导致宿主无法加载。

- 来源：README 3.1 / 3.2。

## 11. 线性内存与栈大小须为 65536 的整数倍且栈 < 初始内存

`CONFIG_COMPILER_WASI_INITIAL_MEMORY` 与 `CONFIG_COMPILER_WASI_MAX_MEMORY` 必须是 65536 的整数倍；`CONFIG_COMPILER_WASI_STACK_SIZE`（默认 8192）必须小于初始内存，否则链接/运行报错。大型应用（GUI/多线程）需调大初始内存。

- 来源：`Kconfig` 注释、`CMakeLists.txt` 第 27–41 行。

## 12. printf 可用，但调试输出依赖宿主配置

`hello_world` 等示例直接 `printf`。输出是否可见取决于宿主（ESP-WASMachine）是否开启应用 stdout 回显，而非应用本身。

- 来源：`examples/hello_world/main/hello_world.c`、README 第 4 节。
