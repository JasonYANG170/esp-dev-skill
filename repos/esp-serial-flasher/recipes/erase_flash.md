# 整片擦除 / 区域擦除

> **适用摘要**: 用 `esp_loader_flash_erase()` 擦除目标整片 flash，或用 `esp_loader_flash_erase_region()` 擦除指定 4KB 对齐区间。常用于烧录前清场、安全擦除。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/erase_flash.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "擦除 ESP flash"
- "整片擦除"
- "erase chip"
- "擦除某个区域"
- "erase region"

## 前置条件

| 条件 | 要求 |
|---|---|
| 连接 | 已 `esp_loader_connect[_with_stub]` 成功 |
| 区域对齐 | `erase_region` 的 offset/size 必须 4096 字节对齐 |
| 参考示例 | `examples/esp32_example/`、`examples/esp32_stub_example/`、`examples/esp32_read_flash_example/` |

## 分步说明

### 1. 整片擦除（耗时，请耐心）

```c
#include "esp_loader.h"

esp_loader_error_t err = esp_loader_flash_erase(&loader);
if (err != ESP_LOADER_SUCCESS) {
    printf("chip erase failed: %d\n", err);
    return;
}
```

> 整片擦除大容量 flash 可能耗时数十秒。

### 2. 区域擦除（必须 4KB 对齐）

```c
// 擦除 0x0 起的 0x1000 字节（一个 4KB 扇区）
err = esp_loader_flash_erase_region(&loader, 0x0, 0x1000);
if (err != ESP_LOADER_SUCCESS) {
    printf("region erase failed: %d\n", err);
    return;
}
```

`esp_loader_flash_erase_region(loader, offset, size)`：
- `offset` 必须 4096 字节对齐
- `size` 必须 4096 字节对齐

### 3. 典型用法：擦除后烧录

```c
// 演示性擦除（非必须，flash_start 内部会擦对应区间）
esp_loader_flash_erase(&loader);
esp_loader_flash_erase_region(&loader, 0, 0x1000);

// 之后照常多分区烧录
flash_binary(&loader, bootloader_bin, bootloader_bin_size,
             get_bootloader_address(esp_loader_get_target(&loader)));
```

> 注意：`esp_loader_flash_start()` 内部已对 `[offset, offset+image_size)` 发擦除命令，常规烧录无需额外 erase。单独 erase 用于清场或安全擦除。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `INVALID_PARAM`（region） | offset/size 未 4KB 对齐 | 向上取整到 4096 倍数 |
| `TIMEOUT`（chip erase） | 大 flash 擦除超时 | 增大连接超时；或改用 region 分批擦 |
| 误擦整片 | 调用了 `flash_erase` 而非 `erase_region` | 按需选择；整片擦除不可逆 |

## 参考

- `examples/esp32_example/main/main.c` — erase + erase_region 演示
- `examples/esp32_stub_example/main/main.c` — stub 下擦除
- `include/esp_loader.h` — `esp_loader_flash_erase()`、`esp_loader_flash_erase_region()` 签名
