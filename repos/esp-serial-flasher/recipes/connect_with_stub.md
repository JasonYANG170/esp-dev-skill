# 用 flasher stub 连接（解锁高波特率 / deflate / 快速读）

> **适用摘要**: 用 `esp_loader_connect_with_stub()` 连接目标，把 ROM bootloader 替换为功能更强的 `esp-flasher-stub`，从而支持更高波特率、>2MB flash、deflate 压缩写、快速 flash 读。仅 serial(SLIP) 接口支持。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/connect_with_stub.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "需要更高烧录速度"
- "stub 连接"
- "flasher stub"
- "deflate 烧录前置"
- "大容量 flash（>2MB）烧录"

## 前置条件

| 条件 | 要求 |
|---|---|
| 接口 | serial(SLIP)：UART / USB CDC-ACM / Linux tty（SPI/SDIO 不适用，SDIO 自动走 stub） |
| host flash | stub 会占用约 +87KB rodata（11 个 per-chip stub 二进制） |
| 参考示例 | `examples/esp32_stub_example/`、`examples/esp32_deflate_example/` |

## 分步说明

### 1. 用 stub 连接（替代 `esp_loader_connect`）

```c
#include "esp_loader.h"
#include "esp32_port.h"
#include "example_common.h"

esp_loader_t loader;
/* ...构造 port + esp_loader_init_serial(&loader, &port.port)... */

esp_loader_connect_args_t connect_args = ESP_LOADER_CONNECT_DEFAULT();
esp_loader_error_t err = esp_loader_connect_with_stub(&loader, &connect_args);
if (err == ESP_LOADER_ERROR_UNSUPPORTED_FUNC) {
    printf("当前接口不支持 stub（SPI/部分协议）\n");
    return err;
} else if (err != ESP_LOADER_SUCCESS) {
    printf("stub connect failed: %d\n", err);
    return err;
}
```

> `example_common.c` 提供 `connect_to_target_with_stub(loader, rate)` helper，内部已处理提速与 ESP8266 特例。

### 2. 提速（stub 支持远高于 ROM 的波特率）

```c
if (esp_loader_get_target(&loader) != ESP8266_CHIP) {
    err = esp_loader_change_transmission_rate(&loader, 921600); // stub 下可更高
    if (err != ESP_LOADER_SUCCESS) return err;
}
```

### 3. 后续烧录流程与 ROM 相同

```c
// 用 stub 也能调用 erase（仅演示，非必须）
esp_loader_flash_erase(&loader);
esp_loader_flash_erase_region(&loader, 0, 0x1000);

target_chip_t chip = esp_loader_get_target(&loader);
flash_binary(&loader, bootloader_bin, bootloader_bin_size, get_bootloader_address(chip));
flash_binary(&loader, partition_table_bin, partition_table_bin_size, 0x8000);
flash_binary(&loader, app_bin, app_bin_size, 0x10000);

esp_loader_reset_target(&loader);
```

## 何时用 stub vs ROM bootloader

| 场景 | 选择 |
|---|---|
| host flash 紧张（如 STM32 小容量） | `esp_loader_connect()`（ROM），靠 `--gc-sections` 剥 stub（≈0KB） |
| 需要高波特率 / deflate / 快速读 / >2MB flash | `esp_loader_connect_with_stub()` |
| SDIO 接口 | 自动走 stub，调用 `esp_loader_connect()` 即可 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `UNSUPPORTED_FUNC`（connect_with_stub） | SPI 等非 serial 接口 | serial 接口才支持 stub；SDIO 自动用 stub |
| stub 上传后无响应 | 高波特率下供电/线缆不稳 | 先在 115200 上传 stub，提速再放到更高 |
| host 链接体积暴涨 ~87KB | stub rodata 被全部链入 | 不用 stub 就只调 `esp_loader_connect()`，或 CMake 层剔除 stub 源 |
| 部分芯片无 stub | 目标未在支持矩阵 | 查 README 支持矩阵；所有支持芯片均已捆绑 stub |

## 参考

- `examples/esp32_stub_example/` — stub 连接 + erase + 多分区烧录
- `examples/common/example_common.c` — `connect_to_target_with_stub()`
- `docs/migration-v1-to-v2.md` — stub 行为说明
- README.md "Flash Size Footprint" — stub rodata 体积
