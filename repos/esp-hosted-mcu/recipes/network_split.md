# Network Split（host 与 slave 共享一个 IP，按端口分流）

> **适用摘要**: 让 host 与协处理器共享同一个 IP 地址，slave 根据端口把入站流量路由到自身或 host 的 lwIP 栈。host 睡眠时 slave 仍可处理 MQTT/DNS 等选定业务，并在收到唤醒包时把 host 拉起。仅支持 C5/C6/S2/S3 协处理器。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-hosted-mcu/resources/`, source/examples in `repos/esp-hosted-mcu/`, and this recipe path `repos/esp-hosted-mcu/recipes/network_split.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-Hosted 网络分流"
- "Network Split 配置"
- "host 和 slave 共享 IP"
- "按端口路由到 host / slave"
- "MQTT 唤醒 host"
- "nw_split_router"

## 前置条件

| 条件 | 要求 |
|---|---|
| Slave 芯片 | 仅 ESP32-C5 / C6 / S2 / S3 |
| 链路 | 已 `esp_hosted_init()` 完成（Network Split 在 init 阶段自动生效） |
| Slave Kconfig | `Example Configuration -> [*] Enable Network Split` |
| Host Kconfig | `Component config -> ESP-Hosted config -> [*] Enable Network Split` |
| 端口范围 | host 与 slave 的 LWIP 端口范围必须完全一致、不重叠 |
| 参考文档 | `docs/feature_network_split.md` |
| 参考例程 | `examples/host_network_split__power_save/` |

## 分步说明

### 1. Slave 侧 menuconfig（`docs/feature_network_split.md`）

```text
Example Configuration
└── [*] Enable Network Split
```

可选自定义（`Network Split Configuration`）：
```text
├── Host Static Port Forwarding
│   ├── TCP dst: 22,80,443,8080,8554
│   └── UDP dst: 53,123
├── Port Ranges
│   ├── Host:  49152–61439
│   └── Slave: 61440–65535
└── Default Destination: slave / host / both
```

### 2. Host 侧 menuconfig

```text
Component config
└── ESP-Hosted config
    └── [*] Enable Network Split
```

并在 host 上把 LWIP 端口范围与 slave 对齐：
```text
└── LWIP port config
    ├── Host LWIP:  49152–61439
    └── Slave LWIP: 61440–65535
```

> 端口范围在 host 与 slave 两侧必须**完全一致**，且**不可重叠**，否则会出现路由错误。

### 3. 启用后无需额外调用

启用 Network Split 后，`esp_hosted_init()` 内部会自动建立分流通道：

```c
#include "esp_hosted.h"

void app_main(void) {
    esp_hosted_init();
    /* host 现在与 slave 共享 IP 并按端口分流 */
}
```

### 4. 路由决策矩阵（来自 `docs/feature_network_split.md`）

slave 收到 Wi-Fi 报文后由 `slave/main/nw_split_router.c` 判定去向：

| 报文类型 | 目的端口条件 | 路由到 |
|---|---|---|
| 广播 / ARP Request / ICMP Request | — | Slave 网络栈 |
| ARP Response / ICMP Response | — | 双栈 |
| DHCP | 任意 | 双栈 |
| TCP/UDP | 命中 Host Static Port Forwarding | Host 网络栈 |
| TCP/UDP | 落在 Host 端口范围 | Host 网络栈 |
| TCP/UDP | 落在 Slave 端口范围 | Slave 网络栈 |
| TCP/UDP | 端口 5001（iperf） | 双栈 |
| MQTT（端口 1883） | 负载包含 `"wakeup-host"` | Host 网络栈（唤醒 host） |
| 其它 | 未命中任何规则 | Default Destination（按配置） |
| 目标是 Host 栈但 host 在 deep sleep | — | 丢弃（除非是唤醒包） |

### 5. 核心路由 API（slave 端，可自定义）

```c
// slave/main/nw_split_router.c
typedef enum {
    SLAVE_LWIP_BRIDGE = 0,
    HOST_LWIP_BRIDGE  = 1,
    BOTH_LWIP_BRIDGE  = 2,
    INVALID_BRIDGE    = 3
} hosted_l2_bridge;

hosted_l2_bridge nw_split_filter_and_route_packet(void *frame_data, uint16_t frame_length);
```

需要改变分流逻辑时，直接编辑 `slave/main/nw_split_router.c` 中的该函数。例如 MQTT 唤醒判定的内置实现：

```c
static bool host_mqtt_wakeup_triggered(const void *payload, uint16_t length) {
    return memcmp(payload, "wakeup-host", strlen("wakeup-host")) == 0;
}
```

### 6. 调试日志

```c
esp_log_level_set("nw_split_router", ESP_LOG_VERBOSE);   // 打开路由判定日志
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 流量全部走到一端 | host/slave 端口范围不一致或重叠 | 两侧范围完全一致、不重叠 |
| 不生效 | host 或 slave 只有一侧开了 Network Split | 双方 menuconfig 都要启用 |
| 在 C3/H2 上报不支持 | 仅 C5/C6/S2/S3 支持 | 换支持的协处理器 |
| Static Forwarding 不工作 | 端口列表格式错误 | 用逗号分隔的数字（见默认 `22,80,443,...`） |
| host 睡眠时业务丢包 | 默认目标设为 host 但 host 在 sleep | 把默认目标设为 slave，或用 MQTT `wakeup-host` 包唤醒 |
| iperf 只在一端看到 | 5001 走双栈是预期行为 | 双栈会同时给 host 与 slave，属正常 |

## 参考项目

- `examples/host_network_split__power_save/` — Network Split + host deep sleep + iperf（含 SPI/SDIO/UART/HD 的 CI sdkconfig）
- `docs/feature_network_split.md` — 路由决策矩阵、配置说明、`nw_split_router.c` 文件清单
- `slave/main/nw_split_router.c` — 可定制的路由判定逻辑
- `host/api/include/esp_hosted_config.h` — host 侧配置入口
- `docs/features.md`（Network Split 段）
