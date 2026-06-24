# 通过 UART 把程序下载到 RAM 并运��

> **适用摘要**: 用 `esp_loader_mem_start/write/finish` 把可执行镜像直接下载到目标 RAM 并跳转执行（不烧 flash）。常用于临时调试、RAM-only 测试程序。复用 `example_common.c` 的 `load_ram_binary()` helper。

## 触发意图

- "下载到 RAM 运行"
- "RAM 下载"
- "load to RAM"
- "不烧 flash 运行程序"
- "mem_start/write/finish"

## 前置条件

| 条件 | 要求 |
|---|---|
| 镜像 | RAM 可执行镜像（带 8 字节公共头 + 扩展头 + segment） |
| 接口 | serial 或 SPI 或 SDIO（均支持 RAM download） |
| 连接 | 已 `esp_loader_connect`（RAM 下载无需 stub） |
| 参考示例 | `examples/esp32_load_ram_example/` |

## 分步说明

### 1. 镜像头与 segment 解析（来自 example_common.c）

```c
#include "esp_loader.h"
#include <inttypes.h>

#define BIN_HEADER_SIZE     0x8    // ESP8266 头长度
#define BIN_HEADER_EXT_SIZE 0x18   // 其余芯片扩展头长度
#define ESP_RAM_BLOCK       0x1800 // RAM 写块大小（来自 example_common.c）

extern const uint8_t app_bin[];   // RAM 镜像
```

### 2. 用 helper 一键下载（推荐）

```c
#include "example_common.h"

esp_loader_error_t err = load_ram_binary(&loader, app_bin);
if (err != ESP_LOADER_SUCCESS) {
    printf("Load to RAM failed: %d\n", err);
}
```

`load_ram_binary()` 内部：解析 `esp_loader_bin_header_t`（`magic`/`segments`/`entrypoint`），按芯片选头偏移，逐 segment `esp_loader_mem_start` → 循环 `esp_loader_mem_write`，最后 `esp_loader_mem_finish(loader, &mem_cfg, header->entrypoint)` 跳转。

### 3. 手写等价流程（理解原理）

```c
const esp_loader_bin_header_t *header = (const esp_loader_bin_header_t *)app_bin;
uint32_t offset = (esp_loader_get_target(&loader) == ESP8266_CHIP)
                  ? BIN_HEADER_SIZE : BIN_HEADER_EXT_SIZE;

uint32_t *cur = (uint32_t *)(&app_bin[offset]);
for (uint32_t s = 0; s < header->segments; s++) {
    uint32_t seg_addr = *cur++;
    uint32_t seg_size = *cur++;
    const uint8_t *seg_data = (const uint8_t *)cur;
    cur += seg_size / 4;

    esp_loader_mem_cfg_t mem_cfg = {
        .offset     = seg_addr,
        .size       = seg_size,
        .block_size = ESP_RAM_BLOCK,
    };
    esp_loader_mem_start(&loader, &mem_cfg);

    size_t remain = seg_size;
    const uint8_t *p = seg_data;
    while (remain > 0) {
        size_t n = MIN(ESP_RAM_BLOCK, remain);
        esp_loader_mem_write(&loader, &mem_cfg, p, n);
        p += n; remain -= n;
    }
}

// 跳转到 entrypoint 运行
esp_loader_mem_finish(&loader, &mem_cfg, header->entrypoint);
```

> RAM 下载后目标开始执行，主机可改用 UART 监听目标日志输出。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `INVALID_PARAM`（mem_start） | Secure Download Mode 开启 | RAM 下载与 secure download 不兼容 |
| ESP8266 解析 segment 错 | 用了 0x18 头偏移 | ESP8266 头 0x8，其余 0x18 |
| 跳转后无输出 | entrypoint 错 / 监听 UART 未就绪 | 用 helper 自动取 `header->entrypoint`；先装好监听 UART |
| segment 地址非法 | 镜像不是 RAM 镜像 | 确保用 RAM 类型的可执行镜像 |

## 参考

- `examples/esp32_load_ram_example/main/main.c` — UART RAM 下载完整示例
- `examples/common/example_common.c` — `load_ram_binary()` 标准实现
- `include/esp_loader.h` — `esp_loader_bin_header_t`、`esp_loader_bin_segment_t`、`esp_loader_mem_cfg_t`
- `recipes/spi_load_ram.md` — SPI 接口 RAM 下载（同样调用链）
