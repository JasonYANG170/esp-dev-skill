# 快速重烧（已知 MD5 比对，跳过相同分区）

> **适用摘要**: 用 `esp_loader_flash_verify_known_md5()` 比对目标的明文 MD5 与已知 MD5，只对不匹配的分区重新烧录，避免每次全量烧录。适合 OTA 旁路、量产复核、固件去重。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/fast_reflash_md5.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "只烧录变化的分区"
- "快速重烧"
- "MD5 比对跳过"
- "避免重复烧录"
- "verify_known_md5"

## 前置条件

| 条件 | 要求 |
|---|---|
| 已知 MD5 | bin2array 生成的 `<bin>_md5`（16 字节原始哈希），或外部计算 |
| 连接状态 | 已连接（stub 或 ROM，serial 接口） |
| 参考示例 | `examples/esp32_fast_reflash_example/` |

## 分步说明

### 1. 准备带 MD5 的镜像（bin2array 自动生成）

```c
extern const uint8_t bootloader_bin[];
extern const uint32_t bootloader_bin_size;
extern const uint8_t bootloader_bin_md5[];   // 16 字节原始 MD5
```

### 2. 逐分区比对，仅烧录不匹配的

```c
#include "esp_loader.h"
#include "example_common.h"

uint32_t boot_addr = get_bootloader_address(esp_loader_get_target(&loader));

if (esp_loader_flash_verify_known_md5(&loader, boot_addr,
        bootloader_bin_size, bootloader_bin_md5) != ESP_LOADER_SUCCESS) {
    printf("Bootloader MD5 mismatch, flashing...\n");
    flash_binary(&loader, bootloader_bin, bootloader_bin_size, boot_addr);
} else {
    printf("Bootloader MD5 match, skipping...\n");
}

if (esp_loader_flash_verify_known_md5(&loader, 0x8000,
        partition_table_bin_size, partition_table_bin_md5) != ESP_LOADER_SUCCESS) {
    printf("Partition table mismatch, flashing...\n");
    flash_binary(&loader, partition_table_bin, partition_table_bin_size, 0x8000);
} else {
    printf("Partition table match, skipping...\n");
}

if (esp_loader_flash_verify_known_md5(&loader, 0x10000,
        app_bin_size, app_bin_md5) != ESP_LOADER_SUCCESS) {
    printf("Application mismatch, flashing...\n");
    flash_binary(&loader, app_bin, app_bin_size, 0x10000);
} else {
    printf("Application match, skipping...\n");
}

esp_loader_reset_target(&loader);
```

### 3. 返回值含义

`esp_loader_flash_verify_known_md5(loader, address, size, expected_md5)`：
- `ESP_LOADER_SUCCESS` — MD5 匹配
- `ESP_LOADER_ERROR_INVALID_MD5` — 不匹配（需重烧）
- `ESP_LOADER_ERROR_IMAGE_SIZE` — 越界 flash 末端
- `ESP_LOADER_ERROR_UNSUPPORTED_FUNC` — 目标/协议不支持（如无 stub 的 ESP8266）
- `ESP_LOADER_ERROR_TIMEOUT` / `INVALID_RESPONSE` — 通信错误

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `UNSUPPORTED_FUNC` | ESP8266 无 stub 不支持 MD5 | 用 stub 连接，或对 8266 跳过比对直接烧 |
| `IMAGE_SIZE` | address+size 超出 flash 容量 | 核对地址与镜像大小 |
| 每次都“mismatch” | expected_md5 与镜像不一致 | 确保用 bin2array 生成的同源 md5，或重新计算 |
| 比对比烧录还慢 | slow read（ROM 模式） | 用 stub 连接获得 fast read |

## 参考

- `examples/esp32_fast_reflash_example/main/main.c` — 快速重烧完整示例
- `examples/common/example_common.c` — `flash_binary()`
- `include/esp_loader.h` — `esp_loader_flash_verify_known_md5()` 签名
