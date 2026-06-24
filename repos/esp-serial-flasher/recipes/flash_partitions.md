# 多分区烧录（手写 start/write/finish）

> **适用摘要**: 不依赖 `example_common` helper，手写 `esp_loader_flash_start` → 循环 `esp_loader_flash_write` → `esp_loader_flash_finish` 的标准烧录流程，明确控制 block_size、进度与 MD5 校验。

## 触发意图

- "手写 flash 烧录循环"
- "esp_loader_flash_start/write/finish 用法"
- "自定义烧录 block 大小"
- "烧录时打印进度"

## 前置条件

| 条件 | 要求 |
|---|---|
| 连接状态 | 已 `esp_loader_connect[_with_stub]` 成功 |
| 数据源 | 内存缓冲区指针 + 已知总大小（4 字节对齐） |
| 参考示例 | `examples/common/example_common.c` 的 `flash_binary()` |

## 分步说明

### 1. 准备镜像与缓冲

```c
#include "esp_loader.h"
#include <sys/param.h>   // MIN

extern const uint8_t app_bin[];
extern const uint32_t app_bin_size;
#define HIGHER_BAUDRATE 230400
```

### 2. start（内部擦除目标区间 + 初始化 MD5 累加）

```c
esp_loader_t loader;  // 假定已 init + connect
static uint8_t payload[1024];   // block_size，需与 cfg.block_size 一致

esp_loader_flash_cfg_t flash_cfg = {
    .offset     = 0x10000,            // 必须 4 字节对齐
    .image_size = app_bin_size,       // 必须 4 字节对齐、已知
    .block_size = sizeof(payload),    // 每次写多大
    // .skip_verify 缺省 false → finish 时自动 MD5 校验
};

esp_loader_error_t err = esp_loader_flash_start(&loader, &flash_cfg);
if (err == ESP_LOADER_ERROR_INVALID_PARAM) {
    printf("Secure Download Mode 下请确认 flash_size 正确\n");
    return err;
} else if (err != ESP_LOADER_SUCCESS) {
    return err;
}
```

> `flash_start` 内部会对 `[offset, offset+image_size)` 区间发擦除命令。

### 3. 循环 write（带进度）

```c
size_t remain = app_bin_size;
size_t written = 0;
const uint8_t *p = app_bin;

while (remain > 0) {
    size_t chunk = MIN(flash_cfg.block_size, remain);
    memcpy(payload, p, chunk);

    err = esp_loader_flash_write(&loader, &flash_cfg, payload, chunk);
    if (err != ESP_LOADER_SUCCESS) {
        printf("write failed: %d\n", err);
        return err;
    }
    p       += chunk;
    written += chunk;
    remain  -= chunk;

    printf("\rProgress: %d %%", (int)((float)written / app_bin_size * 100));
}
printf("\n");
```

### 4. finish（MD5 校验 + flash-end 命令）

```c
err = esp_loader_flash_finish(&loader, &flash_cfg);
if (err == ESP_LOADER_ERROR_INVALID_MD5) {
    printf("MD5 mismatch, verification failed\n");
    return err;
} else if (err == ESP_LOADER_ERROR_UNSUPPORTED_FUNC) {
    printf("该目标/协议不支持 MD5 校验（如 ESP8266 无 stub）\n");
    // 可考虑设 skip_verify=true 重试
    return err;
} else if (err != ESP_LOADER_SUCCESS) {
    return err;
}
if (!flash_cfg.skip_verify) {
    printf("Flash verified\n");
}
```

### 5. 复位目标

```c
esp_loader_reset_target(&loader);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `INVALID_PARAM`（start） | offset/image_size 未 4 字节对齐；secure mode 下 flash_size 错 | 对齐地址；核对 secure mode 传入的字节数 |
| `INVALID_MD5`（finish） | 传输错误 / 镜像不完整 | 重烧；ESP8266 设 `skip_verify=true` |
| `UNSUPPORTED_FUNC`（finish） | 目标/协议不支持 MD5 校验 | 设 `skip_verify=true`，或改用 stub 连接 |
| 写到一半 TIMEOUT | 线缆/供电在高波特率下不稳 | 降波特率；缩短线缆 |
| 漏调 finish | flash-end 未发，目标停在 loader | 必须调用 `esp_loader_flash_finish` |

## 参考

- `examples/common/example_common.c` — `flash_binary()` 标准实现
- `examples/esp32_example/main/main.c` — 多分区烧录
- `include/esp_loader.h` — `esp_loader_flash_cfg_t`、`skip_verify` 字段说明
