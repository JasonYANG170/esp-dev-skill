# OpenThread RCP / Zigbee（协处理器作 RCP）

> **适用摘要**: 把 ESP 协处理器配置为 802.15.4 的 RCP（Radio Co-Processor），host 上运行 OpenThread Host 或 Zigbee Host。当前 OpenThread/Zigbee 数据通过**专用 UART** 通道在 host 与 RCP 间传输（与 ESP-Hosted 主传输分离）；Wi-Fi 仍走 ESP-Hosted 传输。可工作于基础模式或 Border Router / Gateway 模式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-hosted-mcu/resources/`, source/examples in `repos/esp-hosted-mcu/`, and this recipe path `repos/esp-hosted-mcu/recipes/openthread_zigbee_rcp.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-Hosted 跑 OpenThread"
- "把 C6/H2 配成 RCP"
- "OpenThread border router"
- "Zigbee gateway over ESP-Hosted"
- "esp_hosted_openthread"

## 前置条件

| 条件 | 要求 |
|---|---|
| 协处理器 | 支持 RCP 的芯片：ESP32-H2、ESP32-C5、ESP32-C6；Border Router/Gateway 还需协处理器支持 Wi-Fi（C5/C6） |
| 通道 | OpenThread/Zigbee 数据走**专用 UART**（与 Wi-Fi 主传输分离） |
| 参考文档 | `docs/openthread_zigbee.md` |
| 参考例程 | `examples/host_openthread_border_router/`、`examples/host_openthread_cli/`、`examples/host_zigbee_thermostat/` |

## 分步说明

### 1. 协处理器（RCP）配置（以 ESP32-C6 为例）

在 slave 工程编辑 `sdkconfig.defaults.esp32c6`，启用 OpenThread 并裁剪为 RCP：

```text
CONFIG_OPENTHREAD_ENABLED=y
CONFIG_OPENTHREAD_RADIO=y
CONFIG_OPENTHREAD_DIAG=n
CONFIG_OPENTHREAD_COMMISSIONER=n
CONFIG_OPENTHREAD_JOINER=n
CONFIG_OPENTHREAD_BORDER_ROUTER=n
CONFIG_OPENTHREAD_CLI=n
CONFIG_OPENTHREAD_SRP_CLIENT=n
CONFIG_OPENTHREAD_DNS_CLIENT=n
CONFIG_OPENTHREAD_TASK_SIZE=3072
CONFIG_OPENTHREAD_CONSOLE_ENABLE=n
CONFIG_OPENTHREAD_LOG_LEVEL_DYNAMIC=n

CONFIG_ESP_COEX_SW_COEXIST_ENABLE=y
```

然后：
```bash
idf.py set-target esp32c6
idf.py menuconfig
# Example Configuration -> [*] Enable OpenThread RCP (Radio Co-Processor)
#   -> OpenThread RCP Configuration
#       -> OpenThread Transport = UART
#       -> (配置 UART 参数)
idf.py build
idf.py -p <port> flash
```

> Zigbee 仅支持专用 UART 通道与 RCP 通信；OpenThread 目前也需要专用 UART（OpenThread over ESP-Hosted 主传输将来的版本支持）。

### 2. host 侧 OpenThread 配置（以 ESP32-P4 为例）

OpenThread host 配置取决于需要的功能。ESP-Hosted 提供两个例程：
- `examples/host_openthread_cli/`：基础 OpenThread CLI
- `examples/host_openthread_border_router/`：Border Router（需要 Wi-Fi 回传 + RCP，建议用两个协处理器避免单射频共存性能问题）

host 通过 `host/api/include/esp_hosted_openthread.h` 初始化与 RCP 的连接（专用 UART），详见例程。

### 3. 共存建议（单射频 vs 双协处理器）

协处理器只有一个射频，Wi-Fi 与 802.15.4 需软件共存，高流量时可能丢包。**推荐用两个协处理器**：

```
+------------+    ESP-Hosted Transport    +-----------+
| OpenThread |<--------------------------->|   Wi-Fi   | (例: ESP32-C6)
| Border     |                            +-----------+
| Router     |          UART               +-----------+
| / Zigbee   |<--------------------------->|    RCP    | (例: ESP32-H2)
| Gateway    |                            +-----------+
+------------+
```

ESP32-P4 上的 Border Router 完整说明见 ESP-IDF `esp-thread-br` 的 `basic_thread_border_router` 例程（README_esp32p4）；Zigbee Gateway 见 `esp-zigbee-sdk` 的 `zigbee_gateway` 例程。

### 4. Zigbee（host）

Zigbee host 同样通过专用 UART 与 RCP 通信。host 侧 Zigbee 初始化参考 `examples/host_zigbee_thermostat/` 与 `esp-zigbee-sdk`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 协处理器不支持 RCP | 选了非 802.15.4 芯片 | 用 ESP32-H2 / C5 / C6 |
| Border Router 丢包 | 单射频共存冲突 | 用两个协处理器（Wi-Fi + RCP 分离） |
| OpenThread 数据不通 | 误用主传输而非专用 UART | OpenThread/Zigbee 走专用 UART 通道 |
| RCP CLI 没响应 | 协处理器未配为 Radio Only / CLI 关闭 | 按上表配 `OPENTHREAD_RADIO=y`、`OPENTHREAD_CLI=n` |
| Zigbee 不工作 | 期望走主传输 | Zigbee 仅支持专用 UART |

## 参考

- `docs/openthread_zigbee.md`（RCP 配置、host 配置、共存说明）
- `host/api/include/esp_hosted_openthread.h`
- `examples/host_openthread_border_router/`、`examples/host_openthread_cli/`、`examples/host_zigbee_thermostat/`
- `slave/sdkconfig.ci.openthread_rcp`（CI 用的 RCP 配置参考）
