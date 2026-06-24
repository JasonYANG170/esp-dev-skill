# 为新主机平台实现自定义 port

> **适用摘要**: 当内置 port（ESP32/STM32/Zephyr/Pico/Linux）不覆盖你的主机时，实现自己的 `esp_loader_port_ops_t` vtable，把 ESP Serial Flasher 作为 external library（`PORT=USER_DEFINED`）集成。

## 触发意图

- "移植到新 MCU"
- "自定义 port"
- "esp_loader_port_ops_t 实现"
- "USER_DEFINED port"
- "新主机平台"

## 前置条件

| 条件 | 要求 |
|---|---|
| 主机 | 任意 C99 平台 + CMake ≥3.22 |
| 参考文档 | `docs/supporting-new-platform.md` |
| 参考实现 | `port/stm32_port.{c,h}`、`port/pi_pico_port.{c,h}`、`port/linux_port.{c,h}` |

## 分步说明

### 1. 工程结构（external library 方式）

```
my_project/
├── CMakeLists.txt               # set(PORT USER_DEFINED); add_subdirectory(external/esp-serial-flasher)
├── external/esp-serial-flasher/ # git submodule
├── main/
│   ├── main.c
│   ├── my_port.c
│   ├── my_port.h
│   └── CMakeLists.txt
└── .gitmodules
```

```bash
git submodule add https://github.com/espressif/esp-serial-flasher.git external/esp-serial-flasher
git submodule update --init --recursive
```

### 2. 定义 port 结构体（首成员必须是 `esp_loader_port_t port`）

`my_port.h`：
```c
#pragma once
#include "esp_loader_io.h"

typedef struct {
    esp_loader_port_t port;     /* 嵌入式 base，放第一位 */
    /* 配置字段（调用前填） */
    void    *uart_handle;
    uint32_t reset_pin;
    uint32_t boot_pin;
    /* 私有运行状态（约定加 _ 前缀） */
    uint32_t _time_end;
} my_port_t;

extern const esp_loader_port_ops_t my_platform_ops;
```

### 3. 实现 vtable（用 container_of 取回完整结构）

`my_port.c`：
```c
#include "my_port.h"
#include <sys/time.h>

static esp_loader_error_t my_init(esp_loader_port_t *port) {
    my_port_t *p = container_of(port, my_port_t, port);
    /* 用 p->uart_handle / p->reset_pin 初始化硬件 */
    return ESP_LOADER_SUCCESS;
}

static esp_loader_error_t my_write(esp_loader_port_t *port, const uint8_t *data,
                                   uint16_t size, uint32_t timeout) {
    /* 阻塞发送，超时返回 ESP_LOADER_ERROR_TIMEOUT */
    return ESP_LOADER_SUCCESS;
}
static esp_loader_error_t my_read(esp_loader_port_t *port, uint8_t *data,
                                  uint16_t size, uint32_t timeout) {
    return ESP_LOADER_SUCCESS;
}

static void my_enter_bootloader(esp_loader_port_t *port) { /* assert BOOT + toggle RESET */ }
static void my_reset_target(esp_loader_port_t *port)     { /* toggle RESET，保持 RESET_HOLD_TIME_MS */ }

static void my_start_timer(esp_loader_port_t *port, uint32_t ms) {
    my_port_t *p = container_of(port, my_port_t, port);
    p->_time_end = now_ms() + ms;
}
static uint32_t my_remaining_time(esp_loader_port_t *port) {
    my_port_t *p = container_of(port, my_port_t, port);
    int32_t r = (int32_t)(p->_time_end - now_ms());
    return (r > 0) ? (uint32_t)r : 0;
}
static void my_delay_ms(esp_loader_port_t *port, uint32_t ms) { platform_sleep_ms(ms); }

const esp_loader_port_ops_t my_platform_ops = {
    .init                     = my_init,
    .deinit                   = NULL,                 /* 不需要可置 NULL */
    .enter_bootloader         = my_enter_bootloader,
    .reset_target             = my_reset_target,
    .start_timer              = my_start_timer,
    .remaining_time           = my_remaining_time,
    .delay_ms                 = my_delay_ms,
    .log                      = NULL,                 /* 不要文本日志置 NULL */
    .log_hex                  = NULL,
    .change_transmission_rate = NULL,                 /* SDIO 置 NULL；若支持改波特率在此实现 */
    .write                    = my_write,
    .read                     = my_read,
    .spi_set_cs               = NULL,                 /* 非 SPI 置 NULL */
    .sdio_write               = NULL,                 /* 非 SDIO 置 NULL */
    .sdio_read                = NULL,
    .sdio_card_init           = NULL,
};
```

### 4. 构建集成

顶层 `CMakeLists.txt`：
```cmake
cmake_minimum_required(VERSION 3.22)
project(my_project C)
set(PORT USER_DEFINED)          # 关键：不编译内置 port
add_subdirectory(external/esp-serial-flasher)
add_subdirectory(main)
```
`main/CMakeLists.txt`：
```cmake
add_executable(my_app main.c my_port.c)
target_sources(flasher PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/my_port.c)
target_include_directories(flasher PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_SOURCE_DIR}/external/esp-serial-flasher/include)
target_link_libraries(my_app PRIVATE flasher)
```

### 5. 使用

```c
#include "esp_loader.h"
#include "my_port.h"

int main(void) {
    my_port_t port = {
        .port.ops    = &my_platform_ops,
        .uart_handle = &my_uart,
        .reset_pin   = 25,
        .boot_pin    = 26,
    };
    esp_loader_t loader;
    esp_loader_init_serial(&loader, &port.port);   // 自动调 my_init

    esp_loader_connect_args_t args = ESP_LOADER_CONNECT_DEFAULT();
    esp_loader_connect(&loader, &args);
    /* flash / load-ram ... */
}
```

## 回调必填/可选速查（来自 docs/supporting-new-platform.md）

| 回调 | 必填场景 | 可置 NULL |
|---|---|---|
| `init` / `deinit` | 需硬件初始化/释放 | 已外部初始化时 NULL |
| `write` / `read` | UART/USB/SPI | SDIO 用 `sdio_*` |
| `enter_bootloader` / `reset_target` | 必填 | — |
| `start_timer` / `remaining_time` / `delay_ms` | 必填 | — |
| `log` / `log_hex` | 想要日志 | NULL 静默 |
| `change_transmission_rate` | serial 改波特率 | SDIO 必 NULL |
| `spi_set_cs` | SPI | 非 SPI NULL |
| `sdio_write` / `sdio_read` / `sdio_card_init` | SDIO | 非 SDIO NULL |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `container_of` 取回错位 | `port` 不是首成员 | 把 `esp_loader_port_t port` 放结构体第一位 |
| 链接了内置 port | `PORT` 未设 USER_DEFINED | 顶层 `set(PORT USER_DEFINED)` |
| `write`/`read` DMA 对齐崩 | 平台 DMA 要对齐缓冲 | 在 port 内用对齐 bounce buffer |
| timer 不准 | `remaining_time` 用错时钟源 | 用单调时钟（ms），过期返回 0 |

## 参考

- `docs/supporting-new-platform.md` — 完整移植指南（Option A/B、回调详解）
- `include/esp_loader_io.h` — `esp_loader_port_ops_t`、`container_of` 宏定义
- 参考实现：`port/stm32_port.{c,h}`（预初始化外设）、`port/pi_pico_port.{c,h}`、`port/linux_port.{c,h}`
