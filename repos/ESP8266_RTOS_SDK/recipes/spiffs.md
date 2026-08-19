# SPIFFS 文件系统读写

> **适用摘要**: 在 ESP8266 上挂载 SPIFFS 文件系统，用 POSIX/C 标准库函数（fopen/fprintf/fgets/rename/unlink/stat）进行文件读写，并查询分区容量。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/spiffs.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "SPIFFS 文件系统"
- "存储文件到 flash"
- "保存配置/日志到文件"
- "挂载文件系统"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/storage/spiffs/` |
| 分区表 | 必须包含 `data, spiffs` 子类型分区；默认 "Single factory app" 表没有该分区，需用自定义 CSV |
| 配置 | menuconfig：`Partition Table` → `Custom partition table CSV` → 文件名填 `partitions_example.csv`（见工程根或示例目录）|

## 分步说明

### 1. 自定义分区表（含 SPIFFS 分区）

默认分区表里没有 SPIFFS 分区，必须改用自定义 CSV（改编自 `examples/storage/spiffs/partitions_example.csv`）：

```csv
# Name,   Type, SubType, Offset,  Size, Flags
nvs,      data, nvs,     0x9000,  0x6000,
phy_init, data, phy,     0xf000,  0x1000,
factory,  app,  factory, 0x10000, 512K,
storage,  data, spiffs,  ,        448K,
```

然后在 `make menuconfig`：
- `Partition Table` → `Partition Table` → 选 `Custom partition table CSV`
- `Custom partition CSV file` 填 `partitions_example.csv`
- 同时确认 `Flash size` 与实际一致（如 4MB）

### 2. 挂载 SPIFFS（VFS 一站式注册）

```c
#include <stdio.h>
#include <string.h>
#include <sys/unistd.h>
#include <sys/stat.h>
#include "esp_err.h"
#include "esp_log.h"
#include "esp_spiffs.h"

static const char *TAG = "spiffs";

void app_main(void)
{
    ESP_LOGI(TAG, "Initializing SPIFFS");

    esp_vfs_spiffs_conf_t conf = {
        .base_path = "/spiffs",
        .partition_label = NULL,        // NULL = 第一个 spiffs 分区
        .max_files = 5,                 // 同时可打开的最大文件数
        .format_if_mount_failed = true  // 挂载失败时自动格式化
    };

    esp_err_t ret = esp_vfs_spiffs_register(&conf);
    if (ret != ESP_OK) {
        if (ret == ESP_FAIL) {
            ESP_LOGE(TAG, "Failed to mount or format filesystem");
        } else if (ret == ESP_ERR_NOT_FOUND) {
            ESP_LOGE(TAG, "Failed to find SPIFFS partition");
        } else {
            ESP_LOGE(TAG, "Failed to initialize SPIFFS (%s)", esp_err_to_name(ret));
        }
        return;
    }
```

### 3. 查询容量

```c
    size_t total = 0, used = 0;
    ret = esp_spiffs_info(NULL, &total, &used);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "Failed to get SPIFFS info (%s)", esp_err_to_name(ret));
    } else {
        ESP_LOGI(TAG, "Partition size: total: %d, used: %d", total, used);
    }
```

### 4. 用 POSIX/C 标准库读写文件

```c
    // 写
    ESP_LOGI(TAG, "Opening file");
    FILE *f = fopen("/spiffs/hello.txt", "w");
    if (f == NULL) {
        ESP_LOGE(TAG, "Failed to open file for writing");
        return;
    }
    fprintf(f, "Hello World!\n");
    fclose(f);

    // 改名前先删旧目标
    struct stat st;
    if (stat("/spiffs/foo.txt", &st) == 0) {
        unlink("/spiffs/foo.txt");
    }
    rename("/spiffs/hello.txt", "/spiffs/foo.txt");

    // 读
    f = fopen("/spiffs/foo.txt", "r");
    if (f == NULL) {
        ESP_LOGE(TAG, "Failed to open file for reading");
        return;
    }
    char line[64];
    fgets(line, sizeof(line), f);
    fclose(f);
    char *pos = strchr(line, '\n');
    if (pos) *pos = '\0';
    ESP_LOGI(TAG, "Read from file: '%s'", line);
```

### 5. 卸载

```c
    esp_vfs_spiffs_unregister(NULL);   // NULL = 与注册时同一 label
    ESP_LOGI(TAG, "SPIFFS unmounted");
}
```

### 关键 API

| API | 作用 |
|---|---|
| `esp_vfs_spiffs_register(const esp_vfs_spiffs_conf_t *)` | 一站式：挂载 + 注册到 VFS（POSIX 路径可用）|
| `esp_vfs_spiffs_unregister(const char *partition_label)` | 注销并卸载 |
| `esp_spiffs_info(label, &total, &used)` | 查询分区总容量与已用字节 |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_ERR_NOT_FOUND` | 分区表里没有 `data, spiffs` 子类型分区 | 改用含 storage 分区的自定义 CSV 并在 menuconfig 指定 |
| `ESP_FAIL` 挂载失败 | 分区数据损坏 / 首次烧录未格式化 | `format_if_mount_failed = true`，或先 `esp_spiffs_format` |
| `fopen` 返回 NULL | base_path 拼写错 / 未 register | 路径必须以 `base_path`（如 `/spiffs/`）开头 |
| 写入丢失 | 未 `fclose` / 突然掉电 | SPIFFS 无掉电保护，务必 `fclose`；重要数据建议双写或用 NVS |
| 容量比标称小 | SPIFFS 有元数据开销 | 以 `esp_spiffs_info` 返回的 `total` 为准 |

## 参考

- `examples/storage/spiffs/` — 官方 SPIFFS 示例（`main/spiffs_example_main.c`、`partitions_example.csv`、`sdkconfig.defaults`）
- `docs/en/api-guides/partition-tables.rst` — 分区表与 CSV 格式
- `resources/api_reference.md` — SPIFFS 函数签名速查
