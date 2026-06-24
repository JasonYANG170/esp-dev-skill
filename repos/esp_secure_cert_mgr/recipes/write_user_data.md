# 运行时写入自定义 TLV 用户数据（Flash 模式）

> **适用摘要**: 在设备运行期向 `esp_secure_cert` 分区写入自定义 TLV（`ESP_SECURE_CERT_USER_DATA_1..5`）。演示 flash 模式的擦除、写入、读回校验完整流程，含写配置与备份/恢复策略。

## 触发意图

- "运行时写 esp_secure_cert 分区"
- "存自定义数据到安全证书分区"
- "esp_secure_cert_append_tlv"
- "运行时 provisioning"
- "写 USER_DATA TLV"

## 前置条件

| 条件 | 要求 |
|---|---|
| IDF 版本 | >= 5.3（`ESP_SECURE_CERT_WRITE_SUPPORT` 已定义） |
| 头文件 | `esp_secure_cert_write.h`、`esp_secure_cert_tlv_read.h`、`esp_secure_cert_tlv_config.h` |
| 分区 | TLV 格式，可写 flash |
| 参考示例 | `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（`run_write_demo`） |

> **警告**：写入会修改分区内容。生产设备务必先备份（见示例的 backup/restore 流程）。

## 分步说明

### 1. 默认 flash 模式写入（最简）

```c
#include "esp_log.h"
#include "esp_secure_cert_write.h"
#include "esp_secure_cert_tlv_read.h"
#include "esp_secure_cert_tlv_config.h"

static const char *TAG = "write";

esp_err_t write_sample(void)
{
    /* 1) 先擦除（不可逆；生产设备先备份） */
    esp_secure_cert_unmap_partition();               // 写前解除内存映射
    esp_err_t err = esp_secure_cert_erase_partition();
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "erase failed: %s", esp_err_to_name(err));
        return err;
    }

    /* 2) 准备 TLV */
    const char *sample = "Hello from esp_secure_cert write demo!";
    esp_secure_cert_tlv_info_t tlv_info = {
        .type = ESP_SECURE_CERT_USER_DATA_1,
        .subtype = ESP_SECURE_CERT_SUBTYPE_0,
        .data = (char *)sample,
        .length = strlen(sample) + 1,
        .flags = 0,
    };

    /* 3) 写入（write_config=NULL 走默认 flash 模式） */
    err = esp_secure_cert_append_tlv(&tlv_info, NULL);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "write failed: %s", esp_err_to_name(err));
        return err;
    }
    return ESP_OK;
}
```

### 2. 读回校验

```c
esp_secure_cert_tlv_config_t read_cfg = {
    .type = ESP_SECURE_CERT_USER_DATA_1,
    .subtype = ESP_SECURE_CERT_SUBTYPE_0,
};
esp_secure_cert_tlv_info_t read_info = {0};
if (esp_secure_cert_get_tlv_info(&read_cfg, &read_info) == ESP_OK) {
    ESP_LOGI(TAG, "Read back: \"%s\" (len=%" PRIu32 ")", read_info.data, read_info.length);
    esp_secure_cert_free_tlv_info(&read_info);
}
```

### 3. 带写配置（擦除校验 / 自动擦除）

```c
esp_secure_cert_write_config_t config;
esp_secure_cert_write_config_init(&config, ESP_SECURE_CERT_WRITE_MODE_FLASH);
config.flash.check_erase = true;    // 默认即 true：写前校验目标区是否已擦除
config.flash.auto_erase = false;    // 默认 false：不自动擦除（更安全）

esp_err_t err = esp_secure_cert_append_tlv(&tlv_info, &config);
/* 未擦除会返回 ESP_ERR_SECURE_CERT_FLASH_NOT_ERASED */
```

### 4. 批量写入多条目

```c
esp_secure_cert_tlv_info_t entries[] = {
    { .type = ESP_SECURE_CERT_USER_DATA_1, .subtype = ESP_SECURE_CERT_SUBTYPE_0,
      .data = buf1, .length = len1, .flags = 0 },
    { .type = ESP_SECURE_CERT_USER_DATA_2, .subtype = ESP_SECURE_CERT_SUBTYPE_0,
      .data = buf2, .length = len2, .flags = 0 },
};
/* 单次加锁，逐条写入并刷新缓存；任一条目重复都会使整批失败 */
esp_secure_cert_append_tlv_batch(entries,
                                 sizeof(entries) / sizeof(entries[0]),
                                 NULL);
```

### 5. 生产安全：备份 → 擦 → 写 → 恢复

参考示例 `run_write_demo()`：先用 `esp_partition_find_first(0x3F, ...)` 找到分区，`esp_partition_read` 备份到 RAM，再 erase/write/verify，最后 `esp_partition_erase_range` + `esp_partition_write` 恢复，并调用 `esp_secure_cert_unmap_partition()` 让下次读取拿到恢复后的数据。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `FLASH_NOT_ERASED` | 目标区有旧数据，`check_erase=true` | 先 `esp_secure_cert_erase_partition()` |
| `TLV_ALREADY_EXISTS` | 同 type+subtype 已存在 | 换 subtype，或先 erase |
| `WRITE_IN_PROGRESS` | 另一任务持锁 | 重试，或排查并发写 |
| `PARTITION_NOT_FOUND` | 分区表无 `esp_secure_cert` 行 | 修正 `partitions.csv` |
| 链接报未定义 | IDF < 5.3 | 升级 IDF，或用 `ESP_SECURE_CERT_WRITE_SUPPORT` 宏守卫 |
| 写后读到旧数据 | 内存映射未刷新 | 写后 `esp_secure_cert_unmap_partition()` |

## 参考

- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_write.h`（`append_tlv` / `append_tlv_batch` / `erase_partition` / `write_config_init`）
- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_write_errors.h`
- `espressif-repos/esp_secure_cert_mgr/docs/write_support.md`（写流程、并发、批量）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（`run_write_demo`）
