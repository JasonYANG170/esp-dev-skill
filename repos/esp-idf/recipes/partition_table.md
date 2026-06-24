# 自定义分区表与分区查询

> **适用摘要**: 用 CSV 自定义 Flash 分区表，运行时用 `esp_partition_find_first` / `esp_partition_find` 查询分区（适配自 storage/partition_api/partition_find）。

## 触发意图

- "分区表"
- "自定义分区"
- "partitions.csv"
- "查询分区"
- "多 app 分区"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `esp_partition.h` |
| 配置 | `CONFIG_PARTITION_TABLE_TYPE=Custom CSV`、`CONFIG_PARTITION_TABLE_FILENAME=partitions.csv` |
| 参考 | `examples/storage/partition_api/partition_find` |

## 分步说明

### 1. 自定义分区表 `partitions.csv`（含 OTA 与多个 data 分区）

```csv
# Name,     Type, SubType, Offset,   Size,    Flags
nvs,        data, nvs,     ,         0x4000,
phy_init,   data, phy,     ,         0x1000,
factory,    app,  factory, ,         1M,
ota_0,      app,  ota_0,   ,         1M,
ota_1,      app,  ota_1,   ,         1M,
otadata,    data, ota,     ,         0x2000,
storage1,   data, fat,     ,         0x40000,
```

字段说明（来自 `docs/en/api-guides/partition-tables.rst`）：
- **Name**：分区标签（`label`）
- **Type**：`app` / `data` / `bootloader` / `partition`
- **SubType**：app 下 `factory`/`ota_0`/`ota_1`/`test`；data 下 `nvs`/`phy`/`fat`/`ota`(otadata)/`spiffs`/`coredump` 等
- **Offset**：留空由 `gen_esp32part.py` 自动计算（推荐）；分区表本身默认位于 0x8000
- **Size**：`1M`/`0x10000` 等
- **Flags**：`encrypted` / `readonly`

### 2. 在 menuconfig 选择自定义表

```
Partition Table (Custom partition table CSV) --->
  Partition Table Type        = Custom partition table CSV
  Custom partition CSV file   = partitions.csv
```

### 3. 运行时查询分区（适配自 partition_find/main.c）

```c
#include "esp_partition.h"
#include "esp_log.h"

/* 精确/首个匹配 */
const esp_partition_t *p = esp_partition_find_first(
        ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_DATA_NVS, NULL);
if (p) ESP_LOGI(TAG, "nvs @0x%" PRIx32 " size=0x%" PRIx32, p->address, p->size);

/* 迭代所有 app 分区 */
esp_partition_iterator_t it = esp_partition_find(
        ESP_PARTITION_TYPE_APP, ESP_PARTITION_SUBTYPE_ANY, NULL);
for (; it != NULL; it = esp_partition_next(it)) {
    const esp_partition_t *part = esp_partition_get(it);
    ESP_LOGI(TAG, "app partition: %s", part->label);
}
esp_partition_iterator_release(it);
```

### 关键 API

```c
const esp_partition_t* esp_partition_find_first(esp_partition_type_t type,
        esp_partition_subtype_t subtype, const char *label);
esp_partition_iterator_t esp_partition_find(esp_partition_type_t type,
        esp_partition_subtype_t subtype, const char *label);
const esp_partition_t* esp_partition_get(esp_partition_iterator_t iterator);
esp_partition_iterator_t esp_partition_next(esp_partition_iterator_t iterator);
void esp_partition_iterator_release(esp_partition_iterator_t iterator);
esp_err_t esp_partition_read(const esp_partition_t *partition, size_t src_offset,
                             void *dst, size_t size);
esp_err_t esp_partition_write(const esp_partition_t *partition, size_t dst_offset,
                              const void *src, size_t size);
esp_err_t esp_partition_erase_range(const esp_partition_t *partition,
                             size_t offset, size_t size);
```

`esp_partition_t` 关键字段：`label`、`address`、`size`、`type`、`subtype`、`encrypted`。

### 关键 Kconfig

- `CONFIG_PARTITION_TABLE_TYPE`：`Single factory app` / `Factory app, two OTA` / `Custom partition table CSV`
- `CONFIG_PARTITION_TABLE_FILENAME`：自定义 CSV 文件名
- `CONFIG_PARTITION_TABLE_OFFSET`：分区表偏移（默认 0x8000）
- `CONFIG_PARTITION_TABLE_MD5`：启用 MD5 校验

> 改分区表后必须 `idf.py erase-flash` 再重烧，否则 NVS/otadata 偏移错位。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 找不到自定义 CSV | menuconfig 未选 Custom | 设 `Partition Table Type = Custom CSV` |
| 分区重叠 | Offset 手填冲突 | Offset 留空由工具自动算 |
| OTA 无 ota_0/ota_1 | 缺 OTA 分区 | 加 ota_0/ota_1 + otadata |
| 改表后运行异常 | 旧数据偏移错 | `idf.py erase-flash` 后重烧 |
| `esp_partition_write` 数据错乱 | 写前未擦 | 先 `esp_partition_erase_range`（4KB 对齐） |

## 参考

- `examples/storage/partition_api/partition_find` — `find_first` + 迭代
- `examples/storage/partition_api/partition_ops` — 分区读写擦
- `examples/storage/partition_api/partition_mmap` — 内存映射读
- ESP-IDF `docs/en/api-guides/partition-tables.rst`
- ESP-IDF `components/spi_flash/include/esp_partition.h`
