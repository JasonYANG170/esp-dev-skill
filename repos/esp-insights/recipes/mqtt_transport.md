# MQTT(TLS) 传输与 RainMaker Claiming

> **适用摘要**: 把 ESP-Insights 切换为 MQTT(TLS) 传输，通过 ESP RainMaker Claiming 获取证书并复用 RainMaker 的 MQTT 连接上报诊断数据，节省独立 TLS 会话的内存。

## 触发意图

- "用 MQTT 上报诊断"
- "RainMaker claiming"
- "切换 insights transport 到 MQTT"
- "复用 RainMaker 连接上报"
- "fctry 分区"

## 前置条件

| 条件 | 要求 |
|---|---|
| 传输配置 | `CONFIG_ESP_INSIGHTS_TRANSPORT_MQTT=y`（menuconfig 或 sdkconfig） |
| 分区表 | 含 `fctry` 分区用于存放 claiming 证书 |
| CLI | 已安装 esp-rainmaker CLI（`rainmaker.py`），已 signup/login |
| 参考工程 | `examples/minimal_diagnostics`（按其 README 的 “ESP Insights Over MQTT” 段） |

## 分步说明

### 1. 分区表加 fctry（来自 `examples/minimal_diagnostics/partitions.csv`）

```csv
# Name,   Type, SubType,  Offset,   Size,    Flags
nvs,      data, nvs,      0x9000,   24K,
phy_init, data, phy,      0xf000,   4K,
factory,  app,  factory,  0x10000,  0x1E0000,
coredump, data, coredump, 0x330000, 64K,
fctry,    data, nvs,      0x340000, 0x6000,
```
Claiming 把设备证书写入 `fctry`（NVS）。**ESP32-S2 虽支持 self claiming，仍需运行 CLI claim。**

### 2. 切换默认传输

`sdkconfig.defaults`（或 menuconfig：Component config → ESP Insights → Insights default transport → MQTT）：
```
CONFIG_ESP_INSIGHTS_TRANSPORT_MQTT=y
```
CI 配置示例（来自 `examples/minimal_diagnostics/sdkconfig.ci`）：
```
CONFIG_DIAG_MORE_NETWORK_VARS=y
CONFIG_ESP_INSIGHTS_TRANSPORT_MQTT=y
CONFIG_ESP_INSIGHTS_META_VERSION_10=y
CONFIG_ESP_INSIGHTS_DEBUG_ENABLED=y
CONFIG_DIAG_DATA_STORE_DBG_PRINTS=y
```

### 3. MQTT 模式下 auth_key 不需��

MQTT 靠 claiming 证书鉴权，`esp_insights_config_t.auth_key` 仅对 HTTPS 有效，留空即可：

```c
#include "esp_insights.h"
#include "esp_diagnostics.h"

void app_main(void)
{
    /* ... NVS / netif / event loop / example_connect / time sync 同 HTTPS 配方 ... */

    esp_insights_config_t config = {
        .log_type = ESP_DIAG_LOG_TYPE_ERROR | ESP_DIAG_LOG_TYPE_WARNING | ESP_DIAG_LOG_TYPE_EVENT,
        /* MQTT: 不设 auth_key */
    };
    ESP_ERROR_CHECK(esp_insights_init(&config));
}
```

### 4. 烧录并 Claim

```bash
idf.py -p <serial-port> erase_flash build flash

cd path/to/esp-insights/cli
./rainmaker.py claim <serial-port>
```
Claim 成功后设备获得管理员归属，上报数据可在 RainMaker 仪表盘查看。

### 5. （可选）RainMaker 复用 transport 的模式

若项目已用 ESP RainMaker，Insights 可复用其 MQTT 连接（参考 RainMaker `examples/common/app_insights/app_insights.c`）。此时通常用 `esp_insights_enable()` 而非 `esp_insights_init()`，并按需启用 command-response：

```c
/* 伪模式：RainMaker 已建连后启用 Insights（具体接线见 RainMaker app_insights.c） */
esp_insights_transport_register(&transport_cfg);  /* 复用 RainMaker MQTT */
esp_insights_enable(&insights_cfg);
esp_insights_cmd_resp_enable();   /* RainMaker MQTT 节点可远程控制 */
```

### 6. 仪表盘

MQTT 节点数据在 https://dashboard.rainmaker.espressif.com 查看；同样需上传固件包。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 连不上云端 / claim 失败 | `fctry` 分区缺失 | 分区表加 `fctry, data, nvs, 0x340000, 0x6000,` 后 `erase_flash` 重烧 |
| 仍走 HTTPS | 传输配置没生效 | 确认 `CONFIG_ESP_INSIGHTS_TRANSPORT_MQTT=y`；`rm sdkconfig && idf.py reconfigure` |
| command-response 无效 | HTTPS 节点或未启用 | 该特性仅 RainMaker MQTT；启用 `CONFIG_ESP_INSIGHTS_CMD_RESP_ENABLED` |
| claim 报权限错误 | 未 login 或账号与 CLI 不一致 | `./rainmaker.py login` 后重试 |

## 参考

- `examples/minimal_diagnostics/README.md` — “ESP Insights Over MQTT” 完整说明
- `examples/minimal_diagnostics/sdkconfig.ci` — MQTT/调试配置样例
- `examples/minimal_diagnostics/partitions.csv` — 含 `fctry` 分区
- `components/esp_insights/Kconfig` — `ESP_INSIGHTS_TRANSPORT_MQTT` / `ESP_INSIGHTS_CMD_RESP_ENABLED`
- RainMaker `examples/common/app_insights/app_insights.c` — transport 复用参考
