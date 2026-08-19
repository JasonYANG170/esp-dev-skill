# CMUX 模式：数据通道上同时收发 AT

> **适用摘要**: 用 CMUX（GSM 07.10 多路复用）在模组上建立两条虚拟通道，一条跑 PPP 数据，一条发 AT 命令，从而在拨号上网的同时查询信号、发短信等。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-protocols/resources/`, source/examples in `repos/esp-protocols/`, and this recipe path `repos/esp-protocols/recipes/modem_cmux.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "CMUX 模式"
- "数据模式下发 AT"
- "同时上网和发 AT 命令"
- "simple_cmux_client"
- "esp_modem 多路复用"

## 前置条件

| 条件 | 要求 |
|---|---|
| 模组支持 | 模组必须支持 CMUX（**SIM7000 不支持**） |
| Kconfig | `CONFIG_ESP_MODEM_CMUX_DEFRAGMENT_PAYLOAD`：A76xx 系列（2 字节 payload）设 n，其余保持默认 y |
| 组件依赖 | `espressif/esp_modem` |
| 参考示例 | `components/esp_modem/examples/simple_cmux_client/` |

## 分步说明

### 1. 标准 PPP netif + DCE 创建（与 PPPoS recipe 相同）

```c
#include "esp_modem_api.h"
#include "esp_modem_config.h"
#include "esp_modem_dce_config.h"

// ... esp_netif_init / event 注册 ...
esp_netif_config_t netif_ppp_config = ESP_NETIF_DEFAULT_PPP();
esp_netif_t *esp_netif = esp_netif_new(&netif_ppp_config);

esp_modem_dte_config_t dte_config = ESP_MODEM_DTE_DEFAULT_CONFIG();
esp_modem_dce_config_t dce_config = ESP_MODEM_DCE_DEFAULT_CONFIG("internet");
esp_modem_dce_t *dce = esp_modem_new_dev(ESP_MODEM_DCE_SIM7600, &dte_config, &dce_config, esp_netif);
```

### 2. 切换到 CMUX 模式

`COMMAND → CMUX` 是允许的转换。切到 CMUX 后，库内部建立两条虚拟终端，一条专用于数据（PPP），一条专用于命令。

```c
esp_err_t err = esp_modem_set_mode(dce, ESP_MODEM_MODE_CMUX);
if (err != ESP_OK) {
    ESP_LOGE(TAG, "Enter CMUX failed: %s", esp_err_to_name(err));
    return;
}
```

### 3. 在 CMUX 下同时拨号与发 AT

```c
// 在 CMUX 下切到 DATA（数据虚拟通道跑 PPP）
esp_modem_set_mode(dce, ESP_MODEM_MODE_DATA);
// 等待 IP_EVENT_PPP_GOT_IP ...

// 此时命令虚拟通道仍可用，可直接发 AT
int rssi, ber;
esp_modem_get_signal_quality(dce, &rssi, &ber);
ESP_LOGI(TAG, "rssi=%d ber=%d (during data session)", rssi, ber);
```

> 在 CMUX 模式下，AT 命令走命令通道、PPP 走数据通道，因此不需要像纯 COMMAND/DATA 那样 `pause_net`。

### 4. 手动 CMUX（高级：精细控制通道）

普通 CMUX 已能满足绝大多数需求。仅当需要对每条虚拟通道单独切换 DATA/COMMAND、手动进入/退出 CMUX 时，才使用手动模式：

| 模式枚举 | 用途 |
|---|---|
| `ESP_MODEM_MODE_CMUX_MANUAL` | 手动进入 CMUX（自行建虚拟通道） |
| `ESP_MODEM_MODE_CMUX_MANUAL_DATA` | 在手动 CMUX 中将 PPP 通道切 DATA |
| `ESP_MODEM_MODE_CMUX_MANUAL_COMMAND` | 将通道切 COMMAND |
| `ESP_MODEM_MODE_CMUX_MANUAL_SWAP` | 交换主/次终端（PPP 掉线时可恢复数据通信） |
| `ESP_MODEM_MODE_CMUX_MANUAL_EXIT` | 退出手动 CMUX |

手动模式状态转换详见 `docs/esp_modem/en/README.rst` 的 "Switching between manual modes" 图。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 切 CMUX 一直不成功 | 模组不支持（SIM7000）或 SABM 无响应 | 确认模组支持 CMUX；部分设备需 `CONFIG_ESP_MODEM_CMUX_DELAY_AFTER_DLCI_SETUP` 加延迟 |
| A76xx CMUX 缓冲溢出 | 使用 2 字节 payload，defrag 触发溢出 | menuconfig 设 `CONFIG_ESP_MODEM_CMUX_DEFRAGMENT_PAYLOAD=n` |
| A7670 退出 CMUX 异常 | 设备未正确响应 DISC | 应用 docs 中针对 A7670 的 cmux.cpp 补丁（见 README.rst Known issues #3） |
| CAVLI C16QS 进 CMUX 失败 | SABM 响应非 UA/DM（响应 0x3F） | 应用 docs 中针对 C16QS 的进入序列补丁（Known issues #4） |
| CMUX 下偶发 AT 回复分片 | defrag 关闭后命令分片 | 启用 `CONFIG_ESP_MODEM_USE_INFLATABLE_BUFFER_IF_NEEDED=y` 或重试 |

## 参考

- `components/esp_modem/examples/simple_cmux_client/` — CMUX 客户端示例
- `components/esp_modem/include/esp_modem_c_api_types.h` — `esp_modem_dce_mode_t`（含 CMUX_MANUAL_*）
- `components/esp_modem/Kconfig` — CMUX 相关 Kconfig
- `docs/esp_modem/en/README.rst` — 模式切换图与 Known issues（设备 CMUX 兼容性）
