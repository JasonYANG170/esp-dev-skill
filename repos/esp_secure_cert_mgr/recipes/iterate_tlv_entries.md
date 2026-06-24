# 遍历 / 列出 / 通用查询 TLV 条目

> **适用摘要**: 当分区中同类型有多条目、或需要读取自定义 TLV（`USER_DATA_1..5`）时，使用通用 TLV API 按 type+subtype 查询、用迭代器遍历全部条目、或调用 `esp_secure_cert_list_tlv_entries()` 打印清单。

## 触发意图

- "读取自定义 TLV 数据"
- "分区里有哪些 TLV"
- "遍历所有 TLV 条目"
- "读取第二条 CA 证书"
- "esp_secure_cert_get_tlv_info"

## 前置条件

| 条件 | 要求 |
|---|---|
| 格式 | TLV（`cust_flash_tlv`） |
| 头文件 | `esp_secure_cert_tlv_read.h`、`esp_secure_cert_tlv_config.h` |
| 参考示例 | `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`、`examples/esp_secure_cert_ota_example/main/app_main.c`（`read_custom_data`） |

## 分步说明

### 1. 按 type + subtype 精确读取（含自定义数据）

```c
#include "esp_log.h"
#include "esp_secure_cert_tlv_read.h"
#include "esp_secure_cert_tlv_config.h"

static const char *TAG = "tlv";

void read_user_data(void)
{
    esp_secure_cert_tlv_config_t cfg = {
        .type = ESP_SECURE_CERT_USER_DATA_1,
        .subtype = ESP_SECURE_CERT_SUBTYPE_0,
    };
    esp_secure_cert_tlv_info_t info = {0};

    if (esp_secure_cert_get_tlv_info(&cfg, &info) == ESP_OK) {
        ESP_LOGI(TAG, "type=%d subtype=%d len=%" PRIu32, cfg.type, cfg.subtype, info.length);
        ESP_LOGI(TAG, "Data:\n%.*s", (int)info.length, info.data);
        esp_secure_cert_free_tlv_info(&info);   // 仅释放内部动态内存，不释放 info 本身
    } else {
        ESP_LOGE(TAG, "Failed to read TLV");
    }
}
```

> 该 API 会自动解密（HMAC 加密条目）并校验 CRC。若 type 设为 `ESP_SECURE_CERT_TLV_END`，返回的是当前有效 TLV 数据的结束地址与总长度。

### 2. 用迭代器遍历所有 TLV

```c
void list_all_tlv(void)
{
    esp_secure_cert_tlv_iterator_t it = {0};   // 零初始化表示从头开始
    esp_secure_cert_tlv_info_t info = {0};

    while (esp_secure_cert_iterate_to_next_tlv(&it) == ESP_OK) {
        if (esp_secure_cert_get_tlv_info_from_iterator(&it, &info) == ESP_OK) {
            ESP_LOGI(TAG, "type=%d subtype=%d len=%" PRIu32,
                     info.type, info.subtype, info.length);
            esp_secure_cert_free_tlv_info(&info);
        }
    }
}
```

### 3. 一键打印 TLV 清单（调试用）

```c
/* 串口日志输出每个 TLV 的简要信息 */
esp_secure_cert_list_tlv_entries();
```

> 该函数无返回值，内部遍历并 `ESP_LOGI` 打印。

### 4. 分区映射的显式管理（内存受限场景）

```c
esp_secure_cert_partition_ctx_t *ctx = NULL;
if (esp_secure_cert_map_partition(&ctx) == ESP_OK) {
    /* 读操作 ... */
}
/* 暂时不用时可解除映射以释放内存；之后任意 esp_secure_cert API 会自动重新映射 */
esp_secure_cert_unmap_partition();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `get_tlv_info` 返回 ESP_FAIL | 该 type/subtype 不存在或 CRC 错 | 用 `list_tlv_entries()` 先确认条目 |
| 迭代器只返回第一条 | 迭代器未零初始化 | `esp_secure_cert_tlv_iterator_t it = {0};` |
| 读到乱码 | 自定义数据按 string 写入但含 `\0` | 用 `info.length` 限定输出长度 |
| `free_tlv_info` 后仍访问 info.data | data 已被释放 | free 后不要再解引用 `info.data` |
| 指针在 unmap 后失效 | cust_flash 下 data 指向内存映射 | unmap 前拷贝，或 unmap 后重新 get |

## 参考

- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_tlv_read.h`（`get_tlv_info` / `iterate_to_next_tlv` / `list_tlv_entries` / `map_partition`）
- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_tlv_config.h`（type / subtype 枚举）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（`test_read_existing_data` 末尾 TLV 段）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_ota_example/main/app_main.c`（`read_custom_data`）
