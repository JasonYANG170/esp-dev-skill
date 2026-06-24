# NVS 键值存储

> **适用摘要**: 用 NVS（Non-Volatile Storage）持久化整数/字符串/blob，含 `nvs_open`/`nvs_set_*`/`nvs_get_*`/`nvs_commit`（适配自 storage/nvs/nvs_rw_value）。

## 触发意图

- "保存配置"
- "掉电存储"
- "NVS 读写"
- "存键值"
- "存 blob"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `nvs_flash`、`nvs` |
| 头文件 | `nvs_flash.h`、`nvs.h` |
| 分区 | 默认含 `data,nvs` 分区（标准分区表） |
| 参考 | `examples/storage/nvs/nvs_rw_value`、`nvs_rw_blob` |

## 分步说明

### 初始化（必须处理升级错误）

```c
#include "nvs_flash.h"
#include "nvs.h"
#include "esp_log.h"

esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());   /* 擦后重试（会清空 NVS） */
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);
```

### 打开命名空间并读写整数

```c
nvs_handle_t h;
ESP_ERROR_CHECK(nvs_open("storage", NVS_READWRITE, &h));

/* 写 */
int32_t counter = 42;
ESP_ERROR_CHECK(nvs_set_i32(h, "counter", counter));
ESP_ERROR_CHECK(nvs_commit(h));          /* 确保落盘 */

/* 读 */
int32_t val = 0;
err = nvs_get_i32(h, "counter", &val);
if (err == ESP_OK) ESP_LOGI(TAG, "counter=%" PRId32, val);
else if (err == ESP_ERR_NVS_NOT_FOUND) ESP_LOGW(TAG, "not set yet");

nvs_close(h);
```

### 字符串

```c
ESP_ERROR_CHECK(nvs_set_str(h, "ssid", "my_wifi"));

char ssid[33] = {0};
size_t len = sizeof(ssid);
nvs_get_str(h, "ssid", ssid, &len);
```

### Blob（二进制块）

```c
typedef struct { int a; float b; } cfg_t;
cfg_t out = { .a = 1, .b = 3.14f };
ESP_ERROR_CHECK(nvs_set_blob(h, "cfg", &out, sizeof(out)));

cfg_t in;
size_t sz = sizeof(in);
nvs_get_blob(h, "cfg", &in, &sz);
```

> blob 写入需先擦，频繁改写大 blob 建议用 wear_levelling/FATFS。

### 关键 API

```c
esp_err_t nvs_flash_init(void);
esp_err_t nvs_flash_erase(void);
esp_err_t nvs_open(const char *name, nvs_open_mode_t open_mode, nvs_handle_t *out_handle);
void      nvs_close(nvs_handle_t handle);
esp_err_t nvs_commit(nvs_handle_t handle);
esp_err_t nvs_set_i8/i16/i32/i64/u8/u16/u32/u64(nvs_handle_t h, const char *key, _t val);
esp_err_t nvs_get_i32/u32/...(nvs_handle_t h, const char *key, _t *out_val);
esp_err_t nvs_set_str(nvs_handle_t h, const char *key, const char *value);
esp_err_t nvs_get_str(nvs_handle_t h, const char *key, char *out_value, size_t *length);
esp_err_t nvs_set_blob(nvs_handle_t h, const char *key, const void *data, size_t length);
esp_err_t nvs_get_blob(nvs_handle_t h, const char *key, void *out_value, size_t *length);
esp_err_t nvs_erase_key(nvs_handle_t h, const char *key);
```

类型枚举：`NVS_TYPE_I8/I16/I32/I64/U8/U16/U32/U64/STR/BLOB/ANY`（见 nvs_iteration 示例）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_ERR_NVS_NOT_FOUND` | 键/命名空间不存在 | 首次读时降级为默认值 |
| `ESP_ERR_NVS_NO_FREE_PAGES` | 分区被截断 | `nvs_flash_erase()` 后重 init |
| 写入不持久 | 未 `nvs_commit` | 写后调 `nvs_commit` |
| `NVS_READWRITE` 打不开 | 用了只读模式 | 需写用 `NVS_READWRITE` |
| blob 频繁写磨损 | blob 每次重写整块 | 大/频繁数据用 wear_levelling |

## 参考

- `examples/storage/nvs/nvs_rw_value` — 整数读写
- `examples/storage/nvs/nvs_rw_blob` — blob 读写
- `examples/storage/nvs/nvs_iteration` — 遍历命名空间
- ESP-IDF `components/nvs_flash/include/nvs_flash.h`、`components/nvs_flash/include/nvs.h`
