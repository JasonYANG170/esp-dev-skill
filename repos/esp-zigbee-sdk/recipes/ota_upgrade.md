# Zigbee OTA 升级（ota_server 下发 / ota_client 刷写）

> **适用摘要**: 讲解 Zigbee OTA 升级流程——ota_server 作为升级文件提供方（ZC 侧），ota_client 作为接收方接收分块镜像并刷写到 OTA 分区，支持整包与 delta OTA。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-zigbee-sdk/resources/`, source/examples in `repos/esp-zigbee-sdk/`, and this recipe path `repos/esp-zigbee-sdk/recipes/ota_upgrade.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Zigbee OTA 升级"
- "无线固件升级"
- "ota_server / ota_client"
- "delta OTA"
- "Zigbee 固件下发"

## 前置条件

| 条件 | 要求 |
|---|---|
| 分区 | `partitions.csv` 含 `otadata` 与至少一个 OTA 分区（factory + ota_0） |
| 头文件 | `esp_ota_ops.h`（或 `CONFIG_ZB_DELTA_OTA` 时用 `esp_delta_ota_ops.h`） |
| 参考项目 | `examples/ota_upgrade/ota_server/`、`examples/ota_upgrade/ota_client/`、`examples/utils/delta_ota/` |

## 分步说明

### 1. OTA 文件解析器（client 侧）

client 用 `esp_zb_ota_file_parser_t` 解析从 server 收到的镜像分块。

```c
#include "esp_ota_ops.h"
#if CONFIG_ZB_DELTA_OTA
#include "esp_delta_ota_ops.h"
#endif
#include "ota_file_parser.h"

static const esp_partition_t     *s_ota_partition   = NULL;
static esp_zb_ota_file_parser_t  *s_ota_file_parser = NULL;
static esp_ota_handle_t           s_ota_handle      = 0;
```

### 2. OTA cluster（`ezbee/zcl/cluster/ota_upgrade.h`）

OTA 升级基于 ZCL ota cluster（`EZB_ZCL_CLUSTER_ID_OTA` = 0x0019）。server/client 的命令交互（QueryNextImage、ImageBlock、UpgradeEnd 等）由协议栈处理，应用层主要参与：
- server：提供镜像数据
- client：接收数据 → 写 OTA 分区 → 校验 → 切换启动

### 3. 写入 OTA 分区（client 核心逻辑）

收到镜像块后写入 ESP-IDF OTA 分区，完成后切换 boot 分区并重启。

```c
// 伪代码：基于 ESP-IDF esp_ota_ops 的工作流（细节见 examples/ota_upgrade/ota_client/main/ota_client.c）
s_ota_partition = esp_ota_get_next_update_partition(NULL);
esp_ota_begin(s_ota_partition, OTA_WITH_SEQUENTIAL_WRITES, &s_ota_handle);

// 每收到一块：
esp_ota_write(s_ota_handle, block_data, block_len);

// 全部写完：
esp_ota_end(s_ota_handle);
esp_ota_set_boot_partition(s_ota_partition);
esp_restart();
```

> delta OTA（`CONFIG_ZB_DELTA_OTA=y`）用 `esp_delta_ota_ops` 在差分镜像上重建，体积更小，依赖 `examples/utils/delta_ota/`。

### 4. ota_server：下发镜像

server 通常为 ZC，在 OTA cluster server 上响应 client 的查询/块请求，从文件系统或内存提供镜像。镜像头部需符合 Zigbee OTA 文件格式（含 header、升级文件标识、版本、签名等），见 `ezbee/zcl/cluster/ota_file.h`。

### 5. 分区表示例（含 OTA）

```text
# Name,     Type, SubType, Offset, Size, Flags
nvs,        data, nvs,      , 0x6000,
otadata,    data, ota,      , 0x2000,
phy_init,   data, phy,      , 0x1000,
factory,    app,  factory,  , 1M,
zb_storage, data, nvs,      , 16K,
zb_fct,     data, fat,      , 1K,
```

OTA 分区由 ESP-IDF partition table 自动生成（`ota_0`、`ota_1`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| client 不发起 query | OTA cluster 未正确配置 | 确认 client endpoint 含 ota client cluster |
| 写分区失败 | OTA 分区太小/不存在 | partitions.csv 配置足够大的 OTA 分区 |
| 升级后启动旧固件 | 未 `set_boot_partition` | 写完调 `esp_ota_set_boot_partition` + `esp_restart` |
| delta OTA 校验失败 | 基线版本不匹配 | delta 镜像须基于当前运行版本生成 |
| 升级中断 | ImageBlock 超时 | 增加重试；检查链路质量 |

## 参考项目

- `examples/ota_upgrade/ota_client/main/ota_client.c`、`ota_file_parser.c`
- `examples/ota_upgrade/ota_server/main/ota_server.c`
- `examples/utils/delta_ota/` — delta OTA 工具
- `components/esp-zigbee-lib/include/ezbee/zcl/cluster/ota_upgrade.h`、`ota_file.h`
