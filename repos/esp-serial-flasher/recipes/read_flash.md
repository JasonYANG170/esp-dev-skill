# 从目标 flash 读取数据

> **适用摘要**: 用 `esp_loader_flash_read()` 从目标 flash 读取指定地址/长度的数据到主机缓冲区，并与写入数据比对校验。仅 serial(SLIP) 接口支持。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/read_flash.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "读 ESP flash"
- "flash read"
- "回读 flash 校验"
- "dump 目标 flash"

## 前置条件

| 条件 | 要求 |
|---|---|
| 接口 | serial(SLIP)：UART / USB CDC-ACM / Linux tty |
| 连接 | 已 `esp_loader_connect`（slow read）或 `connect_with_stub`（fast read） |
| 参考示例 | `examples/esp32_read_flash_example/` |

## 分步说明

### 1. 写入已知数据

```c
#include "esp_loader.h"
#include <string.h>
#include "example_common.h"

static const uint8_t example_data[] = "To be, or not to be: that is the question ...";
static uint8_t read_buf[sizeof(example_data)];

// 先擦整片（演示），再写
esp_loader_flash_erase(&loader);
flash_binary(&loader, example_data, sizeof(example_data), 0x00000000);
```

### 2. 回读并比对

```c
esp_loader_error_t err = esp_loader_flash_read(&loader, read_buf, 0x00000000, sizeof(read_buf));
if (err == ESP_LOADER_ERROR_UNSUPPORTED_FUNC) {
    printf("非 serial 接口不支持 flash_read\n");
    return;
} else if (err != ESP_LOADER_SUCCESS) {
    printf("read failed: %d\n", err);
    return;
}

if (memcmp(example_data, read_buf, sizeof(read_buf)) == 0) {
    printf("Flash contents match\n");
} else {
    printf("Mismatch!\n");
}
```

> `esp_loader_flash_read(loader, buf, address, length)`：从 `address` 读 `length` 字节到 `buf`。

### 3. 快速读需 stub

```c
// ROM 模式为 slow read；需要 fast read 用 stub 连接
connect_to_target_with_stub(&loader, 230400);
esp_loader_flash_read(&loader, big_buf, 0x0, 0x10000);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `UNSUPPORTED_FUNC` | SPI 接口 / 非 serial | flash_read 仅 serial 支持 |
| 读回数据错位 | address/length 错 | 核对地址与缓冲区大小 |
| 读取很慢 | ROM 模式 slow read | 用 `connect_with_stub` 获得 fast read |
| 读未擦除区域 | flash 残留旧数据 | 先 `esp_loader_flash_erase[_region]` |

## 参考

- `examples/esp32_read_flash_example/main/main.c` — 写后回读比对完整示例
- `include/esp_loader.h` — `esp_loader_flash_read()` 签名
- README.md "Feature Support by Interface" — fast/slow read 与 stub 关系
