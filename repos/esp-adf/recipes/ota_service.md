# OTA 固件升级服务

> **适用摘要**: 用 `ota_service` 组件从网络（HTTP）下载新固件并写入 OTA 分区。服务内置版本比较、分区管理、流式写入与错误码（`OTA_SERV_ERR_REASON_*`）。

> Evidence: `repos/esp-adf/resources/`, source/examples in `repos/esp-adf/`, and this recipe path `repos/esp-adf/recipes/ota_service.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "固件 OTA 升级"
- "在线更新固件"
- "双分区 / 版本比较"
- "ota_service"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/ota/main/` |
| 头文件 | `components/ota_service/include/ota_service.h` |
| 分区表 | `partitions.csv` 含至少两个 OTA 分区（`ota_0` / `ota_1`）+ `ota_data` |
| 网络 | Wi-Fi 已连接 |

## 分步说明

### 1. 关键 API 与错误码（来自 `ota_service.h`）

```c
#define OTA_SERVICE_DEFAULT_CONFIG()  ...   // 默认服务配置

typedef enum {
    OTA_SERV_ERR_REASON_NULL_POINTER            = 0x90000 + 1,
    OTA_SERV_ERR_REASON_URL_PARSE_FAIL          = 0x90000 + 2,
    OTA_SERV_ERR_REASON_ERROR_VERSION           = 0x90000 + 3,
    OTA_SERV_ERR_REASON_NO_HIGHER_VERSION       = 0x90000 + 4,
    OTA_SERV_ERR_REASON_ERROR_MAGIC_WORD        = 0x90000 + 5,
    OTA_SERV_ERR_REASON_ERROR_PROJECT_NAME      = 0x90000 + 6,
    OTA_SERV_ERR_REASON_FILE_NOT_FOUND          = 0x90000 + 7,
    OTA_SERV_ERR_REASON_PARTITION_NOT_FOUND     = 0x90000 + 8,
    OTA_SERV_ERR_REASON_PARTITION_WT_FAIL       = 0x90000 + 9,
    OTA_SERV_ERR_REASON_PARTITION_RD_FAIL       = 0x90000 + 10,
    OTA_SERV_ERR_REASON_STREAM_INIT_FAIL        = 0x90000 + 11,
    OTA_SERV_ERR_REASON_STREAM_RD_FAIL          = 0x90000 + 12,
    OTA_SERV_ERR_REASON_GET_NEW_APP_DESC_FAIL   = 0x90000 + 13,
} ota_service_err_reason_t;
```

> `OTA_SERVICE_ERR_REASON_BASE = 0x90000`。`ota_service_cfg_t` / `ota_service_t` 等结构体定义见头文件。

### 2. 分区表（必备两个 OTA 分区）

```
# Name,   Type, SubType, Offset,  Size
nvs,      data, nvs,     ,        0x4000
otadata,  data, ota,     ,        0x2000
phy_init, data, phy,     ,        0x1000
factory,  app,  factory, ,        1M
ota_0,    app,  ota_0,   ,        2M
ota_1,    app,  ota_1,   ,        2M
```

### 3. 配置并启动 OTA 服务

```c
#include "ota_service.h"

ota_service_cfg_t ota_cfg = OTA_SERVICE_DEFAULT_CONFIG();
// 设置升级 URL、版本比较回调、进度回调等（字段见头文件，按示例填）
// ...
// 启动服务（HTTP 流下载 + 分区写入）
// 返回 ota_service_err_reason_t，非 0 表示失败原因
```

完整字段填充与回调注册见 `examples/ota/main/`，因其依赖具体 cfg 结构体成员，请以示例和头文件为准，不要凭空构造。

### 4. 处理升级结果

```c
// 升级成功后通常需要重启切到新分区
esp_restart();
// 失败则按 ota_service_err_reason_t 给用户反馈
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PARTITION_NOT_FOUND` | 分区表无 ota_0/ota_1 | 加两个 OTA 分区 + otadata |
| `NO_HIGHER_VERSION` | 版本号未变或更旧 | 升级包版本号高于当前 |
| `STREAM_RD_FAIL` | Wi-Fi 断 / URL 失效 | 确认网络与服务器可达 |
| `URL_PARSE_FAIL` | URL 格式错 | 用完整 `http(s)://host/path.bin` |
| 升级后仍跑旧固件 | otadata 未更新 | 用 ESP-IDF `esp_ota_*` 确认 active 分区 |

## 参考项目

- `examples/ota/main/` — OTA 服务完整示例（含分区表、版本比较、进度回调）
