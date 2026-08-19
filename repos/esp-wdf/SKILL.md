---
name: esp-wdf-skill
description: >-
  AI Skill for Espressif ESP-WDF (WebAssembly Development Framework) application development.
  Used when users need to create, modify, build, or debug WebAssembly (WASM/WAMR) applications
  targeting ESP32-series chips, including WAMR App Framework (timer/event/request), VFS/ioctl
  peripheral access (GPIO/UART/I2C/SPI/LEDC), BSD socket networking, pthread concurrency, and
  extended adapters (LVGL/HTTP/MQTT/RainMaker/Wi-Fi Provisioning).
  Trigger words: "ESP-WDF", "WASM", "WebAssembly", "WAMR", "esp-wasmachine", "wasm app", "on_init", "api_register_resource_handler", "esp-wdf", "WASM开发框架", "虚拟机应用", "嘉立创WebAssembly", "ESP32", "Espressif"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-wdf-skill

面向 Espressif ESP-WDF（WebAssembly Development Framework）的 AI Skill。ESP-WDF 是运行于 ESP32 系列芯片上的 WebAssembly 应用程序开发框架，配套的虚拟机（运行时）为 [ESP-WASMachine](https://github.com/espressif/esp-wasmachine)。本 Skill 提供场景化配方、真实 API 速查、Kconfig 配置参考与常见陷阱，全部内容均取自 esp-wdf 仓库的真实文档与源码。

## Core Principles

1. **两种入口形态二选一** —— 标准 WASI 应用实现 `int main(void)`；WAMR App Framework 应用实现 `on_init()`/`on_destroy()`（由 `CONFIG_WAMR_APP_FRAMEWORK=y` 决定，构建系统据此导出符号）。两者不可混用。
2. **外设走 VFS 设备节点，不走宿主驱动 API** —— GPIO/UART/I2C/SPI/LEDC 一律 `open("/dev/...")` → `ioctl(fd, CMD, &cfg)` → `read/write` → `close`。绝不在 WASM 应用里调 `gpio_set_level`、`i2c_master_*`、`ledc_set_duty`。
3. **绝不直接解引用虚拟机分配的结构指针** —— `lv_obj_t`、`lv_timer_t`、`attr_container_t` 等指针来自虚拟机，直接访问成员会触发 `out of bounds memory access`。必须用访问器（`lv_obj_get_data`、`lv_timer_get_user_data`、`attr_container_get_as_*`）。
4. **App Framework 回调导出需显式开启 Kconfig** —— 定时器需 `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER=y`（导出 `on_timer_callback`），请求/响应需 `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_REQUEST_RESPONSE=y`（导出 `on_request`/`on_response`），否则 API 回调不被触发。
5. **attr_container 写函数首参是二级指针** —— `attr_container_set_string(&event, ...)`，因为容器内部可能重建。传一级指针会出错。
6. **请求/响应构造函数的指针不可为 NULL** —— `init_request`/`make_response_for_request`/`set_response` 内部不解引用检查，故用 `response_t response[1];` 形式声明。
7. **网络/sockets 三件齐备** —— `CONFIG_WAMR_APP_FRAMEWORK=y` + `CONFIG_COMPILER_WASI_NO_USE_STDLIB=n` + `CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y`，否则 socket 符号缺失或多线程崩溃。
8. **LVGL 异步初始化，不要驱动内核循环** —— `lvgl_init()` → `lvgl_lock()` → 操作 UI → `lvgl_unlock()`；应用层禁止写 `while(1){ lv_task_handler(); }`。
9. **跨应用请求用 `/app/<app_name>` 前缀寻址** —— `api_send_request` 的 URL 加 `/app/<目标应用名>` 才能跨应用到达。
10. **不在根目录直接 build** —— 顶层 `CMakeLists.txt` 会 `FATAL_ERROR`；必须 `cd examples/<name>` 或自建工程目录再 `idf.py build`。
11. **线性内存与栈须为 65536 整数倍** —— `CONFIG_COMPILER_WASI_INITIAL_MEMORY`/`MAX_MEMORY` 必须是 65536 的整数倍；`STACK_SIZE`（默认 8192）须小于初始内存。
12. **WASM 与 AOT 两条构建命令** —— `idf.py build` 产 `.wasm`；`idf.py build aot` 产 `.aot`（需自行构建 `wamrc`）。安装时固件格式须与运行模式一致。

## When to Use

**Applicable:**
- 新建/编译 ESP-WDF WebAssembly 应用工程
- 使用 WAMR App Framework（定时器、事件发布/订阅、请求/响应）
- 用 VFS/ioctl 访问外设（GPIO/UART/I2C/SPI/LEDC）与文件系统
- 实现 BSD socket 网络通信、pthread 多线程
- 使用 LVGL/HTTP/MQTT/RainMaker/Wi-Fi Provisioning 扩展适配
- 通过 `host_tool.py` 安装/管理 WASM 应用

**Not applicable:**
- 开发 ESP-WASMachine 虚拟机/宿主本身（那是另一个仓库）
- 直接编写原生 ESP-IDF 固件（ESP-WDF 应用运行在 WASM 沙箱内）
- PCB 设计或硬件原理图
- 非 ESP32 系列芯片

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应配方**——其中包含完整调用链、分步说明、常见错误与代码。

### 工程与构建

| recipe | scenario |
|---|---|
| `recipes/new_wasm_app.md` | 新建并编译 WASM 应用，配 sdkconfig.defaults，用 host_tool.py 安装运行 |

### WAMR App Framework

| recipe | scenario |
|---|---|
| `recipes/app_framework_timer.md` | on_init/on_destroy 入口 + api_timer_create 周期定时器 |
| `recipes/event_pub_sub.md` | api_publish_event/api_subscribe_event 应用间事件通信 |
| `recipes/request_response.md` | api_register_resource_handler/api_send_request 应用间请求响应 |

### 外设（VFS/ioctl）

| recipe | scenario |
|---|---|
| `recipes/gpio_vfs.md` | GPIO /dev/gpio/<pin> 配置上下拉、翻转电平 |
| `recipes/i2c_sensor.md` | I2C 主机读 BH1750（I2CIOCSCFG + I2CIOCRDWR/EXCHANGE） |
| `recipes/i2c_touch_gpio_irq.md` | I2C + GPIO READY 门控读 TT21100 触摸屏（两步变长报告，触点/按键） |
| `recipes/spi_ledc.md` | SPI 主机收发 + LEDC 占空比调光 |
| `recipes/uart_filesystem.md` | UART 输出 + /storage 文件读写 |

### 网络与并发

| recipe | scenario |
|---|---|
| `recipes/sockets_network.md` | BSD socket TCP 客户端/服务端（wasi_socket_ext） |
| `recipes/multithread.md` | pthread 线程 + mutex/cond 同步 |

### 图形与云服务

| recipe | scenario |
|---|---|
| `recipes/lvgl_gui.md` | LVGL 异步初始化 + 访问器 API（避免越界） |
| `recipes/cloud_protocols.md` | HTTP / MQTT / RainMaker / Wi-Fi Provisioning |

---

## 外设设备节点与 ioctl 速查

| 外设 | 设备节点 | 配置命令 | 数据命令 |
|---|---|---|---|
| GPIO | `/dev/gpio/<pin>` | `GPIOCSCFG`（`gpioc_cfg_t`） | `write(fd,&state,1)` |
| UART | `/dev/uart/0`、`/dev/usbserjtag` | — | `write(fd,text,len)` |
| I2C | `/dev/i2c/0` | `I2CIOCSCFG`（`i2c_cfg_t`） | `I2CIOCRDWR` / `I2CIOCEXCHANGE` |
| SPI | `/dev/spi/2`、`/dev/spi/3` | `SPIIOCSCFG`（`spi_cfg_t`） | `SPIIOCEXCHANGE`（`spi_ex_msg_t`） |
| LEDC | `/dev/ledc/0`、`/dev/ledc/1`、`/dev/ledc/2` | `LEDCIOCSCFG`（`ledc_cfg_t`） | `LEDCIOCSSETDUTY/SETPHASE/SETFREQ/PAUSE/RESUME` |
| 文件系统 | `/storage/<file>` | — | `open/read/write/lseek/close` |

---

## 入口形态与导出符号对照

| 形态 | Kconfig | 导出符号 | 典型示例 |
|---|---|---|---|
| 标准 WASI | 默认 | `main` | hello_world、coremark、peripherals、sockets、multi_thread��file_system |
| App Framework 基础 | `CONFIG_WAMR_APP_FRAMEWORK=y` | `on_init`、`on_destroy` | event_*、request_* |
| + 定时器 | 再加 `..._EXPORT_TIMER=y` | `on_timer_callback` | timer |
| + 请求/响应 | 再加 `..._EXPORT_REQUEST_RESPONSE=y` | `on_request`、`on_response` | request_* |

---

## Critical Pitfalls (Must Read)

以下是最常见的错误，违反任一条都会导致应用无法工作。

### 1. App Framework 与 main 混用

```c
// ❌ 错误：开了 CONFIG_WAMR_APP_FRAMEWORK 却用 main
int main(void) { ... }

// ✅ 正确：App Framework 应用实现 on_init/on_destroy
void on_init(void) {
    user_timer_t t = api_timer_create(1000, true, false, cb);
    api_timer_restart(t, 1000);
}
void on_destroy(void) {}
```

### 2. 定时器回调不触发（漏开 EXPORT_TIMER）

```c
// ❌ 错误：sdkconfig.defaults 只开 FRAMEWORK
CONFIG_WAMR_APP_FRAMEWORK=y

// ✅ 正确：再加上 EXPORT_TIMER
CONFIG_WAMR_APP_FRAMEWORK=y
CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER=y
```

### 3. 直接解引用虚拟机结构指针（越界）

```c
// ❌ 错误：直接访问 obj->coords.y2
int y2 = obj->coords.y2;

// ✅ 正确：用访问器
lv_area_t area;
lv_obj_get_data(obj, LV_OBJ_COORDS, &area, sizeof(area));
int y2 = area.y2;
```

### 4. attr_container 写函数传一级指针

```c
// ❌ 错误：传一级指针
attr_container_set_string(event, "warning", "overheat");

// ✅ 正确：传二级指针（容器可能被重建）
attr_container_set_string(&event, "warning", "overheat");
```

### 5. 请求/响应构造函数传入 NULL 指针

```c
// ❌ 错误：response 为 NULL
response_t *response = NULL;
make_response_for_request(request, response);   // 崩溃

// ✅ 正确：用数组形式声明
response_t response[1];
make_response_for_request(request, response);
set_response(response, CONTENT_2_05, FMT_ATTR_CONTAINER, payload, len);
api_response_send(response);
```

### 6. WASM 应用调用宿主驱动 API

```c
// ❌ 错误：直接调 ESP-IDF 驱动
gpio_set_level(2, 1);
i2c_master_write_to_device(...);

// ✅ 正确：走 VFS 设备节点
int fd = open("/dev/gpio/2", O_WRONLY);
uint8_t v = 1;
write(fd, &v, 1);
close(fd);
```

### 7. 网络示例 sdkconfig.defaults 不全

```c
// ❌ 错误：缺三件之一 → socket 符号缺失 / 多线程崩溃
CONFIG_WAMR_APP_FRAMEWORK=y

// ✅ 正确：三件齐备
CONFIG_WAMR_APP_FRAMEWORK=y
CONFIG_COMPILER_WASI_NO_USE_STDLIB=n
CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y
```

### 8. LVGL 未加锁或驱动内核循环

```c
// ❌ 错误：无锁操作 UI + 自驱动循环
lv_obj_set_size(btn, 100, 50);
while (1) { lv_task_handler(); sleep_ms(10); }

// ✅ 正确：加锁，不驱动循环
lvgl_init();
lvgl_lock();
lv_obj_set_size(btn, 100, 50);
lvgl_unlock();
```

### 9. 跨应用请求漏前缀

```c
// ❌ 错误：找不到目标应用资源
init_request(req, "url1", COAP_PUT, 0, NULL, 0);   // 仅通用请求

// ✅ 正确：跨应用加 /app/<app_name>
init_request(req, "/app/request_handler/url1", COAP_PUT, 0, NULL, 0);
api_send_request(req, resp_handler, tag);
```

### 10. 在根目录直接 build

```bash
# ❌ 错误：根目录构建 → FATAL_ERROR
cd esp-wdf && idf.py build

# ✅ 正确：进示例/工程目录
cd esp-wdf && . ./export.sh
cd examples/hello_world
idf.py build
```

### 11. 线性内存/栈大小非法

```c
// ❌ 错误：非 65536 整数倍 / 栈 >= 初始内存
CONFIG_COMPILER_WASI_INITIAL_MEMORY=30000
CONFIG_COMPILER_WASI_STACK_SIZE=8192

// ✅ 正确：65536 整数倍，栈 < 初始内存
CONFIG_COMPILER_WASI_INITIAL_MEMORY=65536
CONFIG_COMPILER_WASI_STACK_SIZE=8192
// 大型应用（GUI/多线程）调大 INITIAL_MEMORY
```

### 12. WASM/AOT 固件格式错配

```bash
# ❌ 错误：用 wasm 命令却想要 AOT
idf.py build            # 只产 .wasm

# ✅ 正确：AOT 需单独命令（且需 wamrc 工具）
idf.py build aot        # 产 .aot
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | 理解需求 | 确认是 WASM 应用（ESP-WDF）而非宿主 ESP-IDF 固件；识别入口形态（main vs App Framework） |
| 2 | 选配方 | 在 `recipes/` 找匹配场景；存在则按其调用链推进 |
| 3 | 查 API | 未覆盖的 API 查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | 校验 | 确认头文件包含、Kconfig 开关（FRAMEWORK/EXPORT_*）、sdkconfig.defaults、设备节点路径 |
| 5 | 确认方案 | 向用户说明：入口形态、包含头、Kconfig、初始化顺序、构建命令 |
| 6 | 执行 | 新工程：复制最接近的 `examples/<name>` 改写；已有工程：原地编辑 |
| 7 | 自检 | 检查：入口形态一致、未直接解引用结构指针、未调宿主驱动 API、请求/响应指针非空 |
| 8 | 构建 | `cd` 到工程目录 → `idf.py build`（或 `idf.py build aot`） |
| 9 | 部署 | `host_tool.py -i <app> -f build/<app>.wasm -S <ip> -P <port> --heap ... --timer ...` |

### Step 6 Detail — 工程创建策略

**目标目录无现有工程（首次创建）：**

1. 按需求在 `examples/` 选最接近的示例复制其 `main/` + `CMakeLists.txt` + `sdkconfig.defaults`：
   - 标准 WASI（printf/计算）→ `examples/hello_world/`
   - App Framework 定时器 → `examples/simple/timer/`
   - 事件发布/订阅 → `examples/simple/event_publisher/` / `event_subscriber/`
   - 请求/响应 → `examples/simple/request_handler/` / `request_sender/`
   - GPIO/UART/I2C/SPI/LEDC → `examples/peripherals/...`
   - 文件系统 → `examples/file_system/`
   - 多线程 → `examples/multi_thread/`
   - 网络 socket → `examples/protocols/sockets/...`
   - MQTT/HTTP → `examples/protocols/mqtt/generic/` / `esp_http_client/`
   - RainMaker/配网 → `examples/rainmaker/switch/` / `provisioning/wifi_prov_mgr/`
   - LVGL → `examples/gui/lv_demos/`
2. 修改复制后的代码以匹配需求（改 url、改引脚、改回调）。
3. 说明复制来源与改动点。

**目标目录已有工程：** 原地编辑，除非用户要求否则不覆盖。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 不在 resources 中 | 停下，告知用户该 API 不存在于本框架 |
| 定时器/请求回调不触发 | 检查 `EXPORT_TIMER` / `EXPORT_REQUEST_RESPONSE` 是否开启 |
| `out of bounds memory access` | 排查直接解引用结构指针，改用访问器 |
| socket 符号缺失 | 设 `CONFIG_COMPILER_WASI_NO_USE_STDLIB=n` |
| 多线程/网络崩溃 | 设 `CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y`，调大线性内存 |
| 根目录 build 报错 | `cd` 到 `examples/<name>` 再构建 |
| LVGL UI 错乱 | 确认 `lvgl_lock/unlock` 包裹 UI 操作；删除应用层 `lv_task_handler` 循环 |

## References

- 场景配方 → `recipes/` 目录
- API 速查 → `resources/api_reference.md`
- 配置参考 → `resources/config_reference.md`
- 常见陷阱 → `resources/pitfalls.md`
- 示例索引 → `resources/example_list.md`
- 仓库文档 → esp-wdf `README.md` / `README_CN.md`（第 5 节"开发注意事项"为关键依据）
