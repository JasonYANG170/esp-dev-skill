# 读取目标信息（MAC / flash 容量 / security info / 芯片型号）

> **适用摘要**: 连接目标后读取芯片型号、MAC 地址、flash 容量、安全信息（secure boot / flash encryption / JTAG / USB 等）。常用于烧录前校验目标身份、安全状态盘点。

## 触发意图

- "读 ESP MAC 地址"
- "检测 flash 容量"
- "查询 security info"
- "判断目标芯片型号"
- "检测 secure boot / flash encryption"

## 前置条件

| 条件 | 要求 |
|---|---|
| 连接状态 | 已 `esp_loader_connect` 成功 |
| 参考示例 | `examples/esp32_get_target_info_example/` |

## 分步说明

### 1. 获取芯片型号

```c
#include "esp_loader.h"

target_chip_t chip = esp_loader_get_target(&loader);
// chip ∈ { ESP8266_CHIP, ESP32_CHIP, ESP32S2_CHIP, ESP32C3_CHIP, ESP32S3_CHIP,
//          ESP32C2_CHIP, ESP32C5_CHIP, ESP32H2_CHIP, ESP32C6_CHIP, ESP32P4_CHIP,
//          ESP32C61_CHIP, ESP_UNKNOWN_CHIP }
```

> `esp_loader_get_target()` 必须在连接成功后调用，否则未定义。

### 2. 读 MAC 地址

```c
uint8_t mac[6] = {0};
if (esp_loader_read_mac(&loader, mac) == ESP_LOADER_SUCCESS) {
    printf("MAC: %02x:%02x:%02x:%02x:%02x:%02x\n",
           mac[0],mac[1],mac[2],mac[3],mac[4],mac[5]);
}
```

### 3. 检测 flash 容量

```c
uint32_t flash_size = 0;
if (esp_loader_flash_detect_size(&loader, &flash_size) == ESP_LOADER_SUCCESS) {
    printf("Flash size [B]: %u\n", (unsigned)flash_size);
} else {
    printf("Could not read flash size!\n");
}
```

> 注意：ROM bootloader 模式下检测 2MB 以上 flash 可能受限；stub 模式更可靠（见 `connect_with_stub.md`）。

### 4. 读 security info（ESP32/ESP8266 不支持）

```c
esp_loader_target_security_info_t info;
esp_loader_error_t err = esp_loader_get_security_info(&loader, &info);
if (err == ESP_LOADER_SUCCESS) {
    printf("Chip: %d\n", info.target_chip);
    if (info.target_chip != ESP32S2_CHIP) {
        printf("Eco version: %lu\n", (unsigned long)info.eco_version);
    }
    printf("Secure boot: %s\n", info.secure_boot_enabled ? "ENABLED" : "DISABLED");
    printf("Flash encryption: %s\n", info.flash_encryption_enabled ? "ENABLED" : "DISABLED");
    printf("Secure download mode: %s\n", info.secure_download_mode_enabled ? "ENABLED" : "DISABLED");
    for (size_t k = 0; k < sizeof(info.secure_boot_revoked_keys); k++) {
        printf("SB key %lu revoked: %s\n", (unsigned long)k,
               info.secure_boot_revoked_keys[k] ? "TRUE" : "FALSE");
    }
    // JTAG
    const char *jtag =
        info.jtag_hardware_disabled ? "PERMANENTLY DISABLED" :
        info.jtag_software_disabled ? "DISABLED IN SOFTWARE" : "ENABLED";
    printf("JTAG: %s\n", jtag);
    printf("USB: %s\n", !info.usb_disabled ? "ENABLED" : "DISABLED");
    printf("Dcache in DL mode: %s\n", !info.dcache_in_uart_download_disabled ? "ENABLED" : "DISABLED");
    printf("Icache in DL mode: %s\n", !info.icache_in_uart_download_disabled ? "ENABLED" : "DISABLED");
} else {
    printf("security info 不可用（ESP32/ESP8266 不支持该命令）\n");
}
```

`esp_loader_target_security_info_t` 字段（来自 `include/esp_loader.h`）：`target_chip`、`eco_version`（S2 无）、`secure_boot_enabled`、`secure_boot_aggressive_revoke_enabled`、`secure_download_mode_enabled`、`secure_boot_revoked_keys[3]`、`jtag_software_disabled`、`jtag_hardware_disabled`、`usb_disabled`、`flash_encryption_enabled`、`dcache_in_uart_download_disabled`、`icache_in_uart_download_disabled`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `get_target` 返回 `ESP_UNKNOWN_CHIP` | 未连接就调用 / 不支持的芯片 | 必须先 `esp_loader_connect` 成功 |
| `flash_detect_size` 失败 | ROM 模式下 >2MB 检测受限 | 改用 stub 连接 |
| `get_security_info` 失败 | ESP32 / ESP8266 不支持该命令 | 这是预期行为，仅较新芯片支持 |
| eco_version 字段无意义 | ESP32-S2 无 eco_version | 按例对 S2 跳过该字段 |

## 参考

- `examples/esp32_get_target_info_example/main/main.c` — 完整信息读取示例
- `include/esp_loader.h` — `esp_loader_target_security_info_t` 定义
- `recipes/connect_with_stub.md` — stub 模式以可靠检测大 flash
