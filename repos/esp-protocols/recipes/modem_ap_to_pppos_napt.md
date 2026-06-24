# WiFi AP ↔ PPPoS NAPT 网关

> **适用摘要**: 把 ESP32 变成一个蜂窝路由器 —— 创建 WiFi soft-AP，用 lwip NAPT 将 AP 侧流量转发到蜂窝模组的 PPP 网络接口，使接入 AP 的客户端共享蜂窝上网。支持用标准 C-API `esp_modem_new`，或启用 `EXAMPLE_USE_MINIMAL_DCE` 走自定义 `NetDCE_Factory` + `NetModule`（仅实现建网所需的极简命令）。

## 触发意图

- "ESP32 当蜂窝路由器"
- "WiFi AP 转 PPPoS"
- "ap_to_pppos NAT 转发"
- "蜂窝流量共享给 WiFi 客户端"
- "esp_modem NAPT gateway"
- "NetDCE_Factory / minimal DCE"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.x，CMake 构建 |
| 组件依赖 | `espressif/esp_modem`（>=2.0.2） |
| lwip Kconfig | `CONFIG_LWIP_IP_FORWARD=y` 且 `CONFIG_LWIP_IPV4_NAPT=y`（NAPT 必备） |
| PPP Kconfig | `CONFIG_LWIP_PPP_SUPPORT=y`（默认由 esp_modem select）、`CONFIG_LWIP_PPP_PAP_SUPPORT=y`、`CONFIG_LWIP_PPP_ENABLE_IPV6=n` |
| 性能缓解 | `CONFIG_UART_ISR_IN_IRAM=y`、`CONFIG_LWIP_TCPIP_TASK_STACK_SIZE=4096` |
| 硬件 | 蜂窝模组经 UART 接入；ESP32 开启 soft-AP |
| 参考示例 | `components/esp_modem/examples/ap_to_pppos/` |

## 分步说明

### 1. sdkconfig.defaults（NAPT 与 PPP 必备项）

```ini
# PPP
CONFIG_LWIP_PPP_SUPPORT=y
CONFIG_LWIP_PPP_PAP_SUPPORT=y
CONFIG_LWIP_TCPIP_TASK_STACK_SIZE=4096
CONFIG_LWIP_PPP_ENABLE_IPV6=n
# UART 稳定性
CONFIG_UART_ISR_IN_IRAM=y
# NAPT 转发（核心）
CONFIG_LWIP_IP_FORWARD=y
CONFIG_LWIP_IPV4_NAPT=y
```

> 不开 `LWIP_IPV4_NAPT` 则 `ip_napt_enable` 无效，AP 客户端无法上网。

### 2. 创建 PPP netif，初始化 DCE 网络（核心顺序）

先建 PPP netif，再把它交给 modem 侧初始化（`modem_init_network` 内部创建 DCE 并持有为单例）：

```c
#include "esp_netif.h"
#include "esp_event.h"
#include "network_dce.h"   // example 提供的 C-API 包装：modem_init_network/start_network/...

// 事件组感知 PPP 拿到/丢失 IP
static EventGroupHandle_t event_group;
static const int CONNECT_BIT = BIT0;
static const int DISCONNECT_BIT = BIT1;

// 初始化 esp_netif 和默认事件循环
ESP_ERROR_CHECK(esp_netif_init());
ESP_ERROR_CHECK(esp_event_loop_create_default());
event_group = xEventGroupCreate();

// 1) 先建 PPP netif
esp_netif_config_t ppp_netif_config = ESP_NETIF_DEFAULT_PPP();
esp_netif_t *ppp_netif = esp_netif_new(&ppp_netif_config);
assert(ppp_netif);

// 2) 用 ppp_netif 初始化 modem 网络（内部创建 DCE 单例）
ESP_ERROR_CHECK(modem_init_network(ppp_netif));
ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, ESP_EVENT_ANY_ID, on_ip_event, NULL));
```

### 3. 等待 PPP 拿到 IP（IP_EVENT_PPP_GOT_IP）

`on_ip_event` 处理三种事件：拿到 IPv4、丢失 IP、拿到 IPv6：

```c
static void on_ip_event(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (id == IP_EVENT_PPP_GOT_IP) {
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)data;
        ESP_LOGI(TAG, "PPP GOT IP: " IPSTR, IP2STR(&event->ip_info.ip));
        xEventGroupSetBits(event_group, CONNECT_BIT);
    } else if (id == IP_EVENT_PPP_LOST_IP) {
        ESP_LOGI(TAG, "PPP LOST IP");
        xEventGroupSetBits(event_group, DISCONNECT_BIT);
    } else if (id == IP_EVENT_GOT_IP6) {
        // 可选：IPv6 地址
    }
}
```

### 4. 启动蜂窝网络（带重试的 start_network）

example 用一个循环：先 sync、再查信号、再切 DATA 模式；任一步失败就 stop/reset 重试：

```c
void start_network(void)
{
    EventBits_t bits = 0;
    while ((bits & CONNECT_BIT) == 0) {
        if (!modem_check_sync()) {                       // esp_modem_sync / dce->get_module()->sync()
            modem_stop_network();                        // 切回 COMMAND
            if (!modem_check_sync()) { modem_reset(); }  // esp_modem_reset
            continue;
        }
        if (!modem_check_signal()) {                     // rssi!=99 && rssi>5
            vTaskDelay(pdMS_TO_TICKS(5000)); continue;
        }
        if (!modem_start_network()) {                    // set_mode(DATA_MODE)
            vTaskDelay(pdMS_TO_TICKS(10000)); continue;
        }
        bits = xEventGroupWaitBits(event_group, DISCONNECT_BIT | CONNECT_BIT,
                                   pdTRUE, pdFALSE, pdMS_TO_TICKS(30000));
        if (bits & DISCONNECT_BIT) { modem_stop_network(); }
    }
}
```

### 5. 建 soft-AP，开 NAPT，把 PPP 的 DNS 下发给 AP 客户端

PPP 拿到 IP 后，建 AP netif，把 PPP 链路的 DNS 通过 DHCP 下发给 AP 客户端，再对 AP 接口 IP 开启 NAPT：

```c
#include "lwip/lwip_napt.h"
#include "dhcpserver/dhcpserver.h"
#include "esp_wifi.h"

// 建 AP netif（默认 192.168.4.1）
esp_netif_t *ap_netif = esp_netif_create_default_wifi_ap();
assert(ap_netif);

// 把 PPP 的主 DNS 设给 AP 的 DHCP server，客户端能正确解析
esp_netif_dns_info_t dns;
esp_netif_get_dns_info(ppp_netif, ESP_NETIF_DNS_MAIN, &dns);
set_dhcps_dns(ap_netif, dns.ip.u_addr.ip4.addr);

wifi_init_softap();   // 标准 soft-AP 初始��（WIFI_MODE_AP）

// 核心：对 soft-AP 接口 IP 开启 NAPT —— 之后 AP 客户端流量会被翻译后转发到 PPP 出口
ip_napt_enable(_g_esp_netif_soft_ap_ip.ip.addr, 1);
```

`set_dhcps_dns` 的实现（来自 example）：
```c
static esp_err_t set_dhcps_dns(esp_netif_t *netif, uint32_t addr)
{
    esp_netif_dns_info_t dns;
    dns.ip.u_addr.ip4.addr = addr;
    dns.ip.type = IPADDR_TYPE_V4;
    dhcps_offer_t dhcps_dns_value = OFFER_DNS;
    ESP_ERROR_CHECK(esp_netif_dhcps_option(netif, ESP_NETIF_OP_SET,
                      ESP_NETIF_DOMAIN_NAME_SERVER, &dhcps_dns_value, sizeof(dhcps_dns_value)));
    ESP_ERROR_CHECK(esp_netif_set_dns_info(netif, ESP_NETIF_DNS_MAIN, &dns));
    return ESP_OK;
}
```

### 6. 断线恢复循环

AP/NAPT 起来后，主任务进入等 `DISCONNECT_BIT` 的循环，一旦蜂窝掉线就 stop + 重新 `start_network`：

```c
while (true) {
    EventBits_t bits = xEventGroupWaitBits(event_group, DISCONNECT_BIT,
                                           pdTRUE, pdFALSE, portMAX_DELAY);
    if (bits & DISCONNECT_BIT) {
        modem_stop_network();
        start_network();
    }
}
```

### 7. 可选：Minimal Network DCE（EXAMPLE_USE_MINIMAL_DCE=y）

默认 `network_dce.c` 用 `esp_modem_new` 创建标准 DCE（会发完整 AT 初始化序列）。若只需建网最小命令，menuconfig 设 `CONFIG_EXAMPLE_USE_MINIMAL_DCE=y`，CMake 会改用 `network_dce.cpp`：

```cmake
if(CONFIG_EXAMPLE_USE_MINIMAL_DCE)
    set(NETWORK_DCE "network_dce.cpp")
else()
    set(NETWORK_DCE "network_dce.c")
endif()
```

C++ 版本继承 `ModuleIf`，仅实现 `setup_data_mode`（设 PDP context）与 `set_mode`（DATA/COMMAND），用自定义 `NetDCE_Factory`（继承 `dce_factory::Factory`）创建模块与 DCE：

```cpp
class NetDCE_Factory: public esp_modem::dce_factory::Factory {
public:
    template <typename Module, typename ...Args>
    static DCE_T<Module> *create(const esp_modem::config *cfg, Args &&... args) {
        return build_generic_DCE<Module>(cfg, std::forward<Args>(args)...);
    }
    template <typename Module, typename ...Args>
    static std::shared_ptr<Module> create_module(const esp_modem::config *cfg, Args &&... args) {
        return build_shared_module<Module>(cfg, std::forward<Args>(args)...);
    }
};

class NetModule: public esp_modem::ModuleIf {
public:
    explicit NetModule(std::shared_ptr<esp_modem::DTE> dte, const esp_modem_dce_config *cfg)
        : dte(std::move(dte)), apn(std::string(cfg->apn)) {}

    bool setup_data_mode() override {
        esp_modem::PdpContext pdp(apn);
        return set_pdp_context(pdp) == esp_modem::command_result::OK;
    }
    bool set_mode(esp_modem::modem_mode mode) override {
        if (mode == esp_modem::modem_mode::DATA_MODE) {
            if (set_data_mode() != esp_modem::command_result::OK)
                return resume_data_mode() == esp_modem::command_result::OK;
            return true;
        }
        return set_command_mode() == esp_modem::command_result::OK;
    }
    // ... sync/reset/check_signal 走 dce_commands:: 命名空间
};

// 创建：
auto uart_dte = esp_modem::create_uart_dte(&dte_config);
auto dev = NetDCE_Factory::create_module<NetModule>(&dce_config, uart_dte, netif);
dce = NetDCE_Factory::create<NetModule>(&dce_config, uart_dte, netif, dev);
```

> minimal DCE 路径适合「只关心建网、不想跑完整 AT 初始化」的场景，演示了 `dce_factory` 扩展点。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| AP 客户端能连 WiFi 但上不了网 | 未开 NAPT | `CONFIG_LWIP_IP_FORWARD=y` + `CONFIG_LWIP_IPV4_NAPT=y`，且 `ip_napt_enable(ap_ip, 1)` |
| AP 客户端 DNS 解析失败 | DHCP 没下发 PPP 的 DNS | `set_dhcps_dns(ap_netif, ppp_dns)` 把 PPP 主 DNS 设给 AP DHCP |
| `ip_napt_enable` 找不到符号 | NAPT 编译开关没开 | 确认 `CONFIG_LWIP_IPV4_NAPT=y` |
| `start_network` 一直重试 | sync 失败/信号差/APN 错 | 检查模组上电、`modem_check_signal`、`CONFIG_EXAMPLE_MODEM_PPP_APN` 是否正确 |
| 高吞吐下 PPP 丢包 | UART ISR 抢不到时间 | `CONFIG_UART_ISR_IN_IRAM=y`、增大 `rx_buffer_size`（minimal DCE 已设 16384）、提高任务优先级、启用 HW 流控 |
| minimal DCE 链接报 undefined reference | CMake 选错源文件 | 确认 `CONFIG_EXAMPLE_USE_MINIMAL_DCE=y` 触发 `network_dce.cpp`；C++ 编译需 esp_modem C++ headers |
| NAPT 后某些协议不通 | NAPT 仅做 IPv4 地址/端口翻译 | NAPT 不处理应用层 ALG；某些主动模式 FTP/SIP 可能受影响 |

## 参考项目

- `components/esp_modem/examples/ap_to_pppos/main/ap_to_pppos.c` — 主程序：建 PPP netif、建 AP、开 NAPT、断线恢复
- `components/esp_modem/examples/ap_to_pppos/main/network_dce.c` — C-API 包装（标准 `esp_modem_new`）
- `components/esp_modem/examples/ap_to_pppos/main/network_dce.cpp` — Minimal DCE（`NetDCE_Factory` + `NetModule : ModuleIf`）
- `components/esp_modem/examples/ap_to_pppos/main/Kconfig.projbuild` — `EXAMPLE_USE_MINIMAL_DCE`、`EXAMPLE_MODEM_PPP_APN`、`EXAMPLE_NEED_SIM_PIN` 等
- `components/esp_modem/examples/ap_to_pppos/sdkconfig.defaults` — NAPT/PPP/IRAM 关键配置
- `components/esp_modem/examples/ap_to_pppos/README.md` — minimal DCE 与 dce_factory 说明
- `docs/esp_modem/en/README.rst` — ap_to_pppos 在 example 列表中的描述（WiFi AP NAT 转发到 PPPoS）
