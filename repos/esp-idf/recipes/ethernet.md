# 以太网（内部 MAC + PHY）

> **适用摘要**: 用内部以太网 MAC（EMAC）+ 外部 PHY 跑有线网络：`esp_eth_mac_new_esp32` 建 MAC、`esp_eth_phy_new_generic` 建 PHY（通用 802.3 驱动）、`esp_eth_driver_install` 装驱动、`esp_eth_new_netif_glue` 挂到 netif、`esp_eth_start` 启动、事件处理（`ETHERNET_EVENT_*` / `IP_EVENT_ETH_GOT_IP`）。适配自 `examples/ethernet/basic`。

> Version: ESP-IDF version used by the project.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "以太网"
- "Ethernet"
- "有线网络 / 有线联网"
- "RJ45 / LAN8720 / IP101 / RTL8201"
- "RMII / RGMII"
- "esp_eth"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | esp32（内置 EMAC，RMII）、esp32p4（EMAC + RGMII）。其余 SoC（c3/s3 等）无内部 MAC，需 SPI-Ethernet 模块（如 W5500，见 `examples/ethernet/` 下相应样例，不在本 recipe） |
| 硬件 | PHY 芯片（LAN8720/IP101/RTL8201/YT8531 等）+ RJ45（含网络变压器）；RMII 接线（REF_CLK、MDIO/MDC、TXD/RXD 等） |
| Kconfig | `CONFIG_ETH_ENABLED=y`（默认 esp32/p4 启用）；PHY 类型与 SMI GPIO、RMII 时钟来源由 `CONFIG_EXAMPLE_ETH_*`（示例级）或板级配置决定 |
| 组件 | `esp_eth`、`esp_netif`、`esp_event` |
| 头文件 | `esp_eth.h`（伞头，含 `esp_eth_driver.h` + `esp_eth_netif_glue.h`）、`esp_eth_mac_esp.h`、`esp_eth_phy.h`、`esp_netif.h`、`esp_event.h` |
| 参考 | `examples/ethernet/basic`（内部 MAC，本 recipe 主来源）、`iperf`（吞吐测试）、`ptp`（精密时钟） |

> **PHY 驱动选择**：v5.x 推荐 `esp_eth_phy_new_generic`（通用 IEEE 802.3 驱动，支持自动协商，免专用驱动）。仍可按芯片选 `esp_eth_phy_new_ip101` / `lan8720` / `rtl8201` 等（向后兼容），但 generic 适配性最好。

## 分步说明

### MAC + PHY + Driver 三段式初始化（适配自 ethernet/basic）

```c
#include "esp_netif.h"
#include "esp_eth.h"          /* 含 esp_eth_driver.h + esp_eth_netif_glue.h */
#include "esp_event.h"
#include "esp_log.h"
#include "esp_check.h"

static const char *TAG = "ETH";

/* 初始化以太网驱动（返回 eth_handle） */
static esp_err_t eth_init(esp_eth_handle_t *eth_handle_out)
{
    /* MAC 与 PHY 通用配置（宏初始化默认值） */
    eth_mac_config_t mac_config = ETH_MAC_DEFAULT_CONFIG();
    eth_phy_config_t phy_config = ETH_PHY_DEFAULT_CONFIG();
    phy_config.phy_addr = CONFIG_EXAMPLE_ETH_PHY_ADDR;           /* 通常 0 或 1，硬件决定 */
    phy_config.reset_gpio_num = CONFIG_EXAMPLE_ETH_PHY_RST_GPIO; /* PHY 复位脚 */

    /* 厂商特定 MAC 配置（SMI 引脚、RMII 时钟等） */
    eth_esp32_emac_config_t esp32_emac_config = ETH_ESP32_EMAC_DEFAULT_CONFIG();
    esp32_emac_config.smi_gpio.mdc_num = CONFIG_EXAMPLE_ETH_MDC_GPIO;
    esp32_emac_config.smi_gpio.mdio_num = CONFIG_EXAMPLE_ETH_MDIO_GPIO;
    /* RMII 接口与时钟（默认即 RMII，此处按需覆盖） */
    /* esp32_emac_config.interface = EMAC_DATA_INTERFACE_RMII; */
    /* esp32_emac_config.clock_config.rmii.clock_mode = EMAC_CLK_EXT_IN / EMAC_CLK_OUT; */

    /* 创建 MAC 实例（esp32 内部 EMAC） */
    esp_eth_mac_t *mac = esp_eth_mac_new_esp32(&esp32_emac_config, &mac_config);
    if (mac == NULL) { ESP_LOGE(TAG, "create MAC failed"); return ESP_FAIL; }

    /* 创建 PHY 实例（通用驱动，适配多数 802.3 PHY） */
    esp_eth_phy_t *phy = esp_eth_phy_new_generic(&phy_config);
    if (phy == NULL) {
        ESP_LOGE(TAG, "create PHY failed");
        mac->del(mac);
        return ESP_FAIL;
    }

    /* 安装以太网驱动：ETH_DEFAULT_CONFIG 把 mac/phy 装进 esp_eth_config_t */
    esp_eth_handle_t eth_handle = NULL;
    esp_eth_config_t config = ETH_DEFAULT_CONFIG(mac, phy);
    ESP_RETURN_ON_ERROR(esp_eth_driver_install(&config, &eth_handle), TAG, "driver install failed");

    *eth_handle_out = eth_handle;
    return ESP_OK;
}
```

### netif 挂载 + 事件注册 + 启动

以太网驱动装好后，需挂到 TCP/IP 栈（netif）才能跑 IP，再注册事件处理器，最后 `esp_eth_start`。

```c
static void eth_event_handler(void *arg, esp_event_base_t event_base,
                              int32_t event_id, void *event_data)
{
    /* event_data 是 eth_handle 指针 */
    esp_eth_handle_t eth_handle = *(esp_eth_handle_t *)event_data;
    switch (event_id) {
    case ETHERNET_EVENT_CONNECTED: {
        uint8_t mac_addr[6] = {0};
        esp_eth_ioctl(eth_handle, ETH_CMD_G_MAC_ADDR, mac_addr);
        eth_speed_t speed; eth_duplex_t duplex;
        esp_eth_ioctl(eth_handle, ETH_CMD_G_SPEED, &speed);
        esp_eth_ioctl(eth_handle, ETH_CMD_G_DUPLEX_MODE, &duplex);
        ESP_LOGI(TAG, "Link Up, HW Addr %02x:%02x:%02x:%02x:%02x:%02x, %sMbps %s",
                 mac_addr[0], mac_addr[1], mac_addr[2], mac_addr[3], mac_addr[4], mac_addr[5],
                 speed == ETH_SPEED_10M ? "10" : speed == ETH_SPEED_100M ? "100" : "1000",
                 duplex == ETH_DUPLEX_HALF ? "half" : "full");
        break;
    }
    case ETHERNET_EVENT_DISCONNECTED:
        ESP_LOGI(TAG, "Link Down");
        break;
    case ETHERNET_EVENT_START:
        ESP_LOGI(TAG, "Driver Started");
        break;
    case ETHERNET_EVENT_STOP:
        ESP_LOGI(TAG, "Driver Stopped");
        break;
    default:
        break;
    }
}

static void got_ip_event_handler(void *arg, esp_event_base_t event_base,
                                 int32_t event_id, void *event_data)
{
    ip_event_got_ip_t *event = (ip_event_got_ip_t *)event_data;
    ESP_LOGI(TAG, "Got IP: " IPSTR " mask " IPSTR " gw " IPSTR,
             IP2STR(&event->ip_info.ip), IP2STR(&event->ip_info.netmask),
             IP2STR(&event->ip_info.gw));
}

void app_main(void)
{
    esp_eth_handle_t eth_handle;
    ESP_ERROR_CHECK(eth_init(&eth_handle));

    /* TCP/IP 栈 + 默认事件循环 */
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    /* 创建以太网 netif（单以太网接口用 ESP_NETIF_DEFAULT_ETH） */
    esp_netif_config_t cfg = ESP_NETIF_DEFAULT_ETH();
    esp_netif_t *eth_netif = esp_netif_new(&cfg);

    /* 用 glue 把以太网驱动接到 netif（驱动收到的帧上送 TCP/IP 栈） */
    esp_eth_netif_glue_handle_t eth_netif_glue = esp_eth_new_netif_glue(eth_handle);
    ESP_ERROR_CHECK(esp_netif_attach(eth_netif, eth_netif_glue));

    /* 注册事件：ETH_EVENT（链路/驱动状态）+ IP_EVENT（DHCP 拿到 IP） */
    ESP_ERROR_CHECK(esp_event_handler_register(ETH_EVENT, ESP_EVENT_ANY_ID, &eth_event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_ETH_GOT_IP, &got_ip_event_handler, NULL));

    /* 启动以太网驱动状态机（自动协商 → 连接 → 收发） */
    ESP_ERROR_CHECK(esp_eth_start(eth_handle));
}
```

### 关键 API

```c
/* esp_eth_mac_esp.h —— 内部 MAC 实例 */
esp_eth_mac_t *esp_eth_mac_new_esp32(const eth_esp32_emac_config_t *esp32_config,
                                     const eth_mac_config_t *config);
#define ETH_ESP32_EMAC_DEFAULT_CONFIG()
#define ETH_MAC_DEFAULT_CONFIG()
/* esp_eth_phy.h —— PHY 实例 */
esp_eth_phy_t *esp_eth_phy_new_generic(const eth_phy_config_t *config);  /* 推荐：通用 802.3 */
#define ETH_PHY_DEFAULT_CONFIG()
/* esp_eth_driver.h —— 驱动 */
#define ETH_DEFAULT_CONFIG(emac, ephy)
esp_err_t esp_eth_driver_install(const esp_eth_config_t *config, esp_eth_handle_t *out_hdl);
esp_err_t esp_eth_driver_uninstall(esp_eth_handle_t hdl);
esp_err_t esp_eth_start(esp_eth_handle_t hdl);
esp_err_t esp_eth_stop(esp_eth_handle_t hdl);
esp_err_t esp_eth_ioctl(esp_eth_handle_t hdl, esp_eth_io_cmd_t cmd, void *data);
        /* ETH_CMD_G/S_MAC_ADDR, ETH_CMD_G_SPEED, ETH_CMD_G_DUPLEX_MODE,
         * ETH_CMD_S_AUTONEGO, ETH_CMD_READ_PHY_REG, ETH_CMD_WRITE_PHY_REG ... */
/* esp_eth_netif_glue.h —— 挂到 netif */
esp_eth_netif_glue_handle_t esp_eth_new_netif_glue(esp_eth_handle_t eth_hdl);
esp_err_t esp_eth_del_netif_glue(esp_eth_netif_glue_handle_t eth_netif_glue);
/* esp_netif.h */
#define ESP_NETIF_DEFAULT_ETH()      /* 返回 eth 默认 netif 配置 */
esp_err_t esp_netif_attach(esp_netif_t *netif, void *base);   /* base = glue handle */
/* 事件: ETH_EVENT base, ETHERNET_EVENT_CONNECTED / DISCONNECTED / START / STOP
 *      IP_EVENT base,  IP_EVENT_ETH_GOT_IP */
```

关键事件链：`esp_eth_start` → 自动协商 → `ETHERNET_EVENT_CONNECTED`（链路 up，可读速度/双工）→ DHCP（netif 默认启 DHCP client）→ `IP_EVENT_ETH_GOT_IP`（拿到 IP，可联网）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `Link Down` 一直不上 | PHY 地址错 / 复位脚错 / RMII 时钟缺 | 核对 `phy_addr`（硬件 PHY 地址，常见 0/1）；确认 `reset_gpio_num`；esp32 RMII 需 50MHz REF_CLK（外部输入或 GPIO0 输出） |
| MAC 地址全 0 | 未 `esp_eth_ioctl(G_MAC_ADDR)` 或硬件未就绪 | `CONNECTED` 事件后再读；驱动启动时会自动设默认 MAC |
| 拿不到 IP | DHCP 超时或网线没插 | 确认网线插好、路由器 DHCP 开；netif 默认启 DHCP client，勿手动禁 |
| 用了 `create_default_wifi_*` | netif 类型错 | 以太网用 `ESP_NETIF_DEFAULT_ETH()` + `esp_netif_new` + `esp_eth_new_netif_glue` 挂载 |
| RMII 时钟模式不对 | esp32 的 REF_CLK 来源 | `clock_config.rmii.clock_mode`：`EMAC_CLK_EXT_IN`（外部供）或 `EMAC_CLK_OUT`（GPIO0 输出），由硬件设计决定 |
| `esp_eth_phy_new_generic` 不识别 PHY | 极少数非标 PHY | 改用专用驱动如 `esp_eth_phy_new_ip101` / `lan8720`；或用 `esp_eth_ioctl` 直接读写 PHY 寄存器（见 basic 示例 YT8531 配置） |
| esp32c3/s3 上无 EMAC | SoC 无内部以太网 MAC | 这些芯片需外挂 SPI-Ethernet 模块（W5500/DM9051），用对应样例（非本 recipe） |
| RGMII 时序不对（p4） | TX/RX 延时未配 | RGMII 需 ~2ns 延时，按 PHY datasheet 配寄存器（见 basic 示例 `eth_phy_yt8531_specific_init`） |

## 参考

- `examples/ethernet/basic` — 内部 MAC + 通用 PHY（本 recipe 主来源，含 RMII/RGMII 配置与 YT8531 特殊寄存器配置示例）
- `examples/ethernet/iperf` — 以太网吞吐测试
- `examples/ethernet/ptp` — IEEE 1588 PTP 精密时钟
- ESP-IDF `components/esp_eth/include/esp_eth_driver.h`、`esp_eth_mac_esp.h`、`esp_eth_phy.h`、`esp_eth_netif_glue.h`
- 文档 `docs/en/api-reference/network/esp_eth.rst`、`docs/en/api-reference/network/esp_netif_programming.rst`
