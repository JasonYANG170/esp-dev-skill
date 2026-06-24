# 在内存缓冲区生成分区镜像（Buffer 模式）

> **适用摘要**: 主机端工具或测试场景下，不直接写 flash，而是在 RAM 缓冲区中拼装出完整的 `esp_secure_cert` 分区镜像，再整体烧录或下发。使用 `ESP_SECURE_CERT_WRITE_MODE_BUFFER` 模式。

## 触发意图

- "主机端生成分区镜像"
- "在内存里拼 esp_secure_cert.bin"
- "buffer 模式写入"
- "批量生成多设备分区"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 版本 | >= 5.3（写入 API） |
| 头文件 | `esp_secure_cert_write.h`、`esp_secure_cert_tlv_config.h` |
| 参考文档 | `espressif-repos/esp_secure_cert_mgr/docs/write_support.md`（buffer 模式）、`README.md` |

## 分步说明

### 1. 初始化 buffer 模式配置

```c
#include "esp_log.h"
#include "stdlib.h"
#include "string.h"
#include "esp_secure_cert_write.h"
#include "esp_secure_cert_tlv_config.h"

uint8_t buffer[8192];                 // 容量建议 >= 分区大小（TLV 默认 8 KiB）
size_t bytes_written = 0;

esp_secure_cert_write_config_t config;
esp_secure_cert_write_config_init(&config, ESP_SECURE_CERT_WRITE_MODE_BUFFER);
config.buffer.buffer = buffer;
config.buffer.buffer_size = sizeof(buffer);
config.buffer.bytes_written = &bytes_written;
```

> `esp_secure_cert_write_config_init` 在 buffer 模式下不设置任何 flash 字段；必须显式填 `buffer` / `buffer_size` / `bytes_written`，否则返回 `ESP_ERR_SECURE_CERT_BUFFER_CONFIG_INVALID`。

### 2. 逐条追加 TLV

```c
const char *dev_cert = "-----BEGIN CERTIFICATE-----\n...\n-----END CERTIFICATE-----\n";
esp_secure_cert_tlv_info_t dev = {
    .type = ESP_SECURE_CERT_DEV_CERT_TLV,
    .subtype = ESP_SECURE_CERT_SUBTYPE_0,
    .data = (char *)dev_cert,
    .length = strlen(dev_cert) + 1,
    .flags = 0,
};
esp_err_t err = esp_secure_cert_append_tlv(&dev, &config);
```

> buffer 模式不查重、不擦除、不校验读回（in-memory）。每条 TLV 写入会更新 `*bytes_written`。

### 3. 批量写入

```c
esp_secure_cert_tlv_info_t entries[N] = { /* ... */ };
esp_secure_cert_append_tlv_batch(entries, N, &config);
```

### 4. 取出镜像

```c
/* buffer 的前 bytes_written 字节即为分区镜像，可烧到 flash 或下发 */
ESP_LOG_BUFFER_HEX(TAG, buffer, bytes_written);
/* 例如 esptool write_flash 0xD000 ... */
```

### 5. 关于完整性 TLV

> 注意：工具 `configure_esp_secure_cert.py` 生成分区时会自动追加 `ESP_SECURE_CERT_INTEGRITY_TLV`（SHA256）。运行期 buffer 模式 API 本身不自动追加；若需要完整性校验，请由工具生成镜像，或自行追加该 TLV。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `BUFFER_CONFIG_INVALID` | 缺 `buffer` / `buffer_size` / `bytes_written` | 三者全部填写 |
| `BUFFER_OVERFLOW` | 缓冲区太小 | 增大 `buffer_size`（建议 >= 8 KiB） |
| 写出的镜像读不出 | 未对齐 / 缺完整性 TLV | 用工具生成，或确保 16 字节对齐由 API 自动处理 |
| `bytes_written` 不更新 | 未传指针 | 传 `&bytes_written` |

## 参考

- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_write.h`（`esp_secure_cert_write_config_t` / `_init`）
- `espressif-repos/esp_secure_cert_mgr/docs/write_support.md`（Write Modes / Configuration Structure）
- `espressif-repos/esp_secure_cert_mgr/README.md`（"Advanced Usage: Buffer Mode"）
