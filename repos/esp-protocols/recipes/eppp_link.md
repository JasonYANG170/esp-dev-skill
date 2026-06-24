# eppp_link 双 MCU PPP 组网

> **适用摘要**: 用 eppp_link 在两个 MCU 之间通过 UART/SPI/SDIO/以太网建立 PPP 通道。典型用途是 WiFi 协处理器（通信协处理器跑 PPP server + NAT，主控跑 PPP client 拿到联网能力）。

## 触发意图

- "双 MCU 联网"
- "WiFi 协处理器"
- "eppp_link"
- "PPP over UART/SPI"
- "通信协处理器 SLAVE/HOST"

## 前置条件

| 条件 | 要求 |
|---|---|
| 角色 | 一端为 server（`eppp_listen`），一端为 client（`eppp_connect`） |
| 传输层 | 两端一致：UART / SPI / SDIO / Ethernet |
| 组件依赖 | `espressif/eppp_link` |
| 参考示例 | `components/eppp_link/examples/`（含 server/client） |

## 分步说明

### 1. 选择传输层（menuconfig）

eppp_link 通过 Kconfig 选择传输层设备，`EPPP_DEFAULT_TRANSPORT_CONFIG()` 宏会自动展开为对应配置：

```
EPPP Link → Device → UART / SPI / SDIO / Ethernet
```

各传输默认引脚由 `EPPP_DEFAULT_*_CONFIG()` 宏给出（见 `eppp_link.h`）：

| 宏 | 默认关键参数 |
|---|---|
| `EPPP_DEFAULT_UART_CONFIG()` | port=1, baud=921600, tx=25, rx=26, rx_buffer=1024 |
| `EPPP_DEFAULT_SPI_CONFIG()` | host=1, mosi=11, miso=13, sclk=12, cs=10, intr=2, freq=16MHz |
| `EPPP_DEFAULT_SDIO_CONFIG()` | width=4, clk=18, cmd=19, d0..d3=14..17 |
| `EPPP_DEFAULT_ETH_CONFIG()` | mdc=23, mdio=18, phy_addr=1, rst=5 |

### 2. server 端（WiFi 协处理器侧）

```c
#include "eppp_link.h"

void app_main(void)
{
    esp_netif_init();
    esp_event_loop_create_default();

    // server：our=192.168.11.1, their=192.168.11.2（已交叉）
    eppp_config_t cfg = EPPP_DEFAULT_SERVER_CONFIG();
    esp_netif_t *netif = eppp_listen(&cfg);   // 阻塞监听 client 接入
    assert(netif);
    // 之后可在该 netif 上做 NAPT 转发到 WiFi STA/AP（参考 examples 中 server 例程）
}
```

### 3. client 端（主控侧）

```c
#include "eppp_link.h"

void app_main(void)
{
    esp_netif_init();
    esp_event_loop_create_default();

    // client：our=192.168.11.2, their=192.168.11.1（已交叉）
    eppp_config_t cfg = EPPP_DEFAULT_CLIENT_CONFIG();
    esp_netif_t *netif = eppp_connect(&cfg);   // 连接 server
    assert(netif);
    // 拿到 netif 后，默认路由走该 PPP 接口即可访问外网
}
```

> `EPPP_DEFAULT_SERVER_CONFIG()` / `EPPP_DEFAULT_CLIENT_CONFIG()` 已经把两端 IP 正确交叉（server our=server_ip/their=client_ip，client our=client_ip/their=server_ip）。手写配置时务必保证两端 our/their 对应一致。

### 4. 其它生命周期 API（按需）

| API | 用途 |
|---|---|
| `eppp_init(role, config)` | 仅初始化，返回 netif（不自动建链） |
| `eppp_open(role, config, timeout_ms)` | 初始化并尝试在超时内建链 |
| `eppp_netif_start(netif)` | 启动已初始化的 netif |
| `eppp_netif_stop(netif, timeout_ms)` | 停止 netif |
| `eppp_perform(netif)` | 手动驱动一次协议处理 |
| `eppp_add_channels(netif, tx, rx, ctx)` | 添加自定义逻辑通道（如 802.11 帧） |

默认 IP：
```c
EPPP_DEFAULT_SERVER_IP()   // 192.168.11.1
EPPP_DEFAULT_CLIENT_IP()   // 192.168.11.2
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| PPP 协商失败 | 两端 IP 没交叉对应 | 用 `EPPP_DEFAULT_SERVER_CONFIG` / `EPPP_DEFAULT_CLIENT_CONFIG` |
| client 连不上 server | 传输层引脚/配置不一致 | 两端 menuconfig 选同一传输层，引脚对应 |
| server 收不到数据 | 接线 TX/RX 反 / 共地缺失 | 交叉 TX/RX，确保共地 |
| 想传自定义数据（非 IP） | 默认只跑 IP 流 | 用 `eppp_add_channels` 注册自定义 tx/rx 通道 |
| 想用 TUN 而非 PPP | 默认 PPP 模式 | eppp_link 也支持 TUN 模式（简化隧道），见 README |

## 参考

- `components/eppp_link/include/eppp_link.h` — 配置宏、`eppp_connect/listen/init/open/netif_*`
- `components/eppp_link/README.md` — 架构图（通信协处理器 + 主控）与 usecase
- `components/eppp_link/examples/` — server/client 真实工程
