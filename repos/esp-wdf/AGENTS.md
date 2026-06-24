# AGENTS.md — 补充代理指南

> 核心规则、配方索引、陷阱、执行工作流均在 `SKILL.md`。
> 本文件仅覆盖 `SKILL.md` 未提及的约定与工具用法，不重复内容。

## Project Context

**语言**：C · **目标**：ESP32 系列芯片上运行的 WebAssembly 应用（WASM/WAMR） · **工具链**：ESP-WDF（ESP-IDF 风格的 `idf.py`，产物 `.wasm` / `.aot`） · **运行时宿主**：ESP-WASMachine

## Code Generation Conventions

### 文件命名
- 源文件：`<app_name>.c`（如 `hello_world.c`、`timer.c`、`event_publisher.c`）
- 头文件：`<module>.h`
- 示例约定：外设示例用 `<periph>_simple_main.c` / `<periph>_<chip>_main.c`；simple 示例直接用功能名（`timer.c`、`event_publisher.c`）
- 组件 ioctl 头：`ioctl/esp_<periph>_ioctl.h`（位于 `components/wamr/libc-builtin-extended/include/`）
- App Framework 头：`wasm_app.h`、`wa-inc/request.h`、`wa-inc/timer_wasm_app.h`、`bi-inc/attr_container.h`、`bi-inc/shared_utils.h`（位于 `components/wamr/app-framework/include/`）

### Include 模式

```c
/* 标准 WASI 应用（外设/文件/socket） */
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <stdint.h>
#include <sys/ioctl.h>          /* 外设 ioctl */
#include "sdkconfig.h"
#include "ioctl/esp_gpio_ioctl.h"   /* 按外设替换：i2c/spi/ledc */

/* socket 应用额外 */
#include <sys/socket.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#ifdef __wasi__
#include <wasi_socket_ext.h>
#endif

/* WAMR App Framework 应用 */
#include "wasm_app.h"
#include "wa-inc/request.h"
#include "wa-inc/timer_wasm_app.h"   /* 用定时器时 */
/* bi-inc/attr_container.h 由 wasm_app.h 间接包含 */

/* 扩展适配 */
#include "esp_lvgl.h"            /* LVGL（CONFIG_WDF_EXT_WASM_APP_LVGL） */
#include "mqtt_client.h"         /* MQTT */
#include "esp_http_client.h"     /* HTTP Client */
#include "esp_rmaker_core.h"     /* RainMaker */
```

### 标准工程结构

```
my_app/
├── CMakeLists.txt          # 顶层（可空/最小），不能在根 esp-wdf 目录构建
├── sdkconfig.defaults      # 预置关键 Kconfig 开关
└── main/
    ├── CMakeLists.txt      # idf_component_register(SRCS ...)
    └── my_app.c
```

`main/CMakeLists.txt` 模板（取自 `examples/hello_world/main/CMakeLists.txt`）：

```cmake
set(srcs "my_app.c")
idf_component_register(SRCS ${srcs})
```

### 标准入口模式

**标准 WASI 应用：**

```c
int main(void)
{
    /* 直接逻辑，或调用 on_init() 风格的初始化函数 */
    printf("Hello\n");
    return 0;
}

/* 或带 argc/argv */
int main(int argc, char *argv[]) { ... }
```

**WAMR App Framework 应用：**

```c
#include "wasm_app.h"
#include "wa-inc/timer_wasm_app.h"

void
on_init(void)
{
    user_timer_t t = api_timer_create(1000, true, false, timer_cb);
    api_timer_restart(t, 1000);
}

void
on_destroy(void)
{
    /* 实际清理由库版本 on_destroy() 完成 */
}
```

### 外设访问模板（VFS/ioctl）

```c
int fd = open("/dev/gpio/<pin>", O_WRONLY);   /* 或 O_RDWR */
if (fd < 0) { printf("open failed errno=%d\n", errno); return; }

gpioc_cfg_t cfg = { .flags = GPIOC_PULLUP_EN };
if (ioctl(fd, GPIOCSCFG, &cfg) < 0) { printf("cfg failed\n"); close(fd); return; }

uint8_t state = 1;
write(fd, &state, 1);
close(fd);
```

### 调试输出约定

应用直接 `printf(...)` 即可（`hello_world` 示例即如此）。输出是否可见取决于宿主 ESP-WASMachine 是否开启应用 stdout 回显。亦可使用 ESP-IDF 兼容的 `esp_log.h`（`ESP_LOGI/ESP_LOGE` + `TAG`）。

## Build Workflow

1. 在 esp-wdf 根目录安装环境（一次性）：`./install.sh`（Linux/macOS）或 `./install.bat`（Windows）
2. 每次构建前 source 环境：`. ./export.sh`（Windows：`export.bat`）
3. `cd` 到工程目录（如 `examples/hello_world`），**切勿在根目录构建**
4. （可选）`idf.py menuconfig`；多数示例已用 `sdkconfig.defaults` 预置
5. 构建 WASM：`idf.py build` → `build/<name>.wasm`
6. 构建 AOT（需 wamrc）：`idf.py build aot` → `build/<name>.aot`
7. 安装运行：`python tools/host_tool.py -i <app> -f build/<app>.wasm -S <ip> -P <port> --heap <bytes> --timer <n> --watchdog <ms>`

## Code Generation Checklist

- [ ] 入口形态与 Kconfig 一致（main vs on_init/on_destroy），未混用
- [ ] 用到定时器/请求响应时，`EXPORT_TIMER`/`EXPORT_REQUEST_RESPONSE` 已开启
- [ ] 外设访问走 `open/ioctl/read/write/close`，未调宿主驱动 API
- [ ] `attr_container_set_*` 传二级指针 `&container`
- [ ] 请求/响应用 `xxx_t var[1];` 声明，指针非 NULL
- [ ] LVGL 操作前 `lvgl_lock()`、后 `lvgl_unlock()`；无 `lv_task_handler` 循环
- [ ] 未直接解引用虚拟机分配的结构指针（用访问器）
- [ ] 网络示例三件齐备（`FRAMEWORK=y`、`NO_USE_STDLIB=n`、`USE_SHARED_MEMORY=y`）
- [ ] 线性内存/栈为 65536 整数倍，栈 < 初始内存
- [ ] 跨应用请求 URL 带 `/app/<app_name>` 前缀
- [ ] 用完设备 fd / 容器后 `close(fd)` / `attr_container_destroy()`

## Do Not Modify

- `resources/` —— API/配置文档来源（来自 esp-wdf 仓库真实头文件）
- `SKILL.md` frontmatter —— Skill 元数据
- esp-wdf 仓库源码（`components/`、`examples/`）—— 仅作参考与复制起点，不直接改写仓库
