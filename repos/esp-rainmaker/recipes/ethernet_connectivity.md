# 以太网 / 双网络连接

> **适用摘要**: 以太网作为 RainMaker **主传输**（无 BT/SoftAP 配网），通过 **on-network challenge-response** 完成用户-节点映射；可选 `EXAMPLE_ENABLE_WIFI` 开启双网络（"先连上的胜出"）。覆盖 PHY GPIO 配置、`app_ethernet_init/start`、与 `app_network` 的并存。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "RainMaker 以太网连接"
- "Ethernet 为主网络"
- "以太网设备怎么配网 / 用户-节点映射"
- "双网络 Wi-Fi + 以太网"
- "on-network challenge-response / on-network chal_resp"
- "app_ethernet_init / app_ethernet_start"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 带 Ethernet MAC 的 ESP32 系列 SoC + 外部 PHY（如 LAN8720 / IP101） |
| Kconfig | `CONFIG_ETH_ENABLED=y`；以太网模式 `CONFIG_ESP_RMAKER_NO_CLAIM=y`（claiming 不支持以太网） |
| Claim | 以太网无法用 BT/SoftAP 配网；必须用 on-network chal_resp 完成 user-node mapping（见下） |
| 组件 | `examples/ethernet_switch/components/app_ethernet/`（提供 `app_ethernet_init/start/stop`） |
| 参考示例 | `examples/ethernet_switch/` |

## 以太网 vs Wi-Fi 连接路径

| 维度 | Wi-Fi（默认） | 以太网 |
|---|---|---|
| 配网 | BT/SoftAP 下发 SSID/密码 | 插上网线即获 IP，无配网 |
| Claim | Self / Assisted | **不支持**，强制 `CONFIG_ESP_RMAKER_NO_CLAIM=y` |
| 用户-节点映射 | 传统 user mapping 或配网期 chal_resp | **on-network chal_resp**（设备拿到 IP 后通过 mDNS 发现） |
| MQTT username(PPI) | 默认带 MAC | 以太网下建议关：`CONFIG_ESP_RMAKER_MQTT_SEND_USERNAME=n` |
| 工具 | 手机 App | `esp-rainmaker-cli provision --transport on-network` |

## 分步说明

### 1. sdkconfig.defaults（以太网模式核心项）

```text
# --- Ethernet ---
CONFIG_ETH_ENABLED=y

# 以太网不支持 claiming，必须 No Claim（凭据需预烧或私有部署）
CONFIG_ESP_RMAKER_NO_CLAIM=y

# 避免 MAC 取不到导致 PPI 异常
CONFIG_ESP_RMAKER_MQTT_SEND_USERNAME=n

# on-network chal_resp（用户-节点映射）
CONFIG_ESP_RMAKER_ENABLE_CHALLENGE_RESPONSE=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_AUTO_ENABLE=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_SECURITY_1=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_CHAL_RESP_ENABLE=y   # 走 local control 通道的 chal_resp
# 或独立 HTTP 服务：
# CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE=y  # 与 LOCAL_CTRL 互斥（共用 protocomm_httpd 单例）

# 双网络（可选）：把 Wi-Fi 也打开，两路谁先连上用谁
# CONFIG_EXAMPLE_ENABLE_WIFI=y   # 在 menuconfig -> Example Configuration 里勾
CONFIG_APP_NETWORK_ASYNCHRONOUS_CONNECTION=y       # 双网络异步并发连接

CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y
```

> `CONFIG_ESP_RMAKER_LOCAL_CTRL_CHAL_RESP_ENABLE` 与 `CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE` **互斥**（都基于 protocomm_httpd 单例），二选一。

### 2. PHY 引脚配置（menuconfig → Example Ethernet Configuration）

`examples/ethernet_switch/components/app_ethernet/Kconfig.projbuild` 暴露以下选项，按板子原理图填：

| Kconfig | 含义 | ESP32 默认 |
|---|---|---|
| `EXAMPLE_ETH_PHY_INTERFACE` | `DEFAULT` 或 `RMII` | DEFAULT |
| `EXAMPLE_ETH_RMII_CLK_MODE` | `INPUT`（外部时钟）/ `OUTPUT`（内部 PLL 输出） | INPUT |
| `EXAMPLE_ETH_RMII_CLK_GPIO` | RMII REF_CLK GPIO | 0 |
| `EXAMPLE_ETH_MDC_GPIO` | SMI MDC | 23 |
| `EXAMPLE_ETH_MDIO_GPIO` | SMI MDIO | 18 |
| `EXAMPLE_ETH_PHY_ADDR` | PHY 地址（-1 自动探测） | 1 |
| `EXAMPLE_ETH_PHY_RST_GPIO` | PHY 硬件复位 GPIO（-1 禁用） | 5 |
| `EXAMPLE_ETH_PHY_RST_TIMING_EN` | 启用复位时序微调 | n |

> ESP32 同时用 Wi-Fi/BT 时**不要**选 RMII CLK OUTPUT（时钟不稳，见 Kconfig help 的 Errata 提示）。

### 3. `app_main` 初始化顺序（以太网为主）

注意与纯 Wi-Fi 流程的差异：网络 init 在 `node_init` 之前，`app_network_start`/`app_ethernet_start` 在 `esp_rmaker_start()` 之后。

```c
#include <esp_rmaker_core.h>
#include <esp_rmaker_standard_devices.h>
#include <esp_rmaker_ota.h>
#include <app_ethernet.h>      /* app_ethernet_init/start/stop */
#include <app_insights.h>
#include "app_on_network_test.h"
#ifdef CONFIG_EXAMPLE_ENABLE_WIFI
#include <app_network.h>
#endif

void app_main(void)
{
    esp_rmaker_console_init();
    app_driver_init();

    esp_err_t err = nvs_flash_init();
    if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        err = nvs_flash_init();
    }
    ESP_ERROR_CHECK(err);

    /* 网络 init 必须在 node_init 之前 */
#ifdef CONFIG_EXAMPLE_ENABLE_WIFI
    app_network_init();        /* 可选：Wi-Fi 配网通道 */
#endif
    ESP_ERROR_CHECK(app_ethernet_init());   /* 以太网 */

    /* 事件 handler：RMAKER_EVENT / RMAKER_COMMON_EVENT / RMAKER_OTA_EVENT
     * 双网络时再注册 APP_NETWORK_EVENT */
    esp_event_handler_register(RMAKER_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL);
    esp_event_handler_register(RMAKER_COMMON_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL);
    esp_event_handler_register(RMAKER_OTA_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL);

    /* 节点 + 设备 + 服务（与标准流程一致） */
    esp_rmaker_node_t *node = esp_rmaker_node_init(&rainmaker_cfg, "ESP RainMaker Device", "Ethernet Switch");
    /* ... switch_device, add_param, add_device ... */
    esp_rmaker_ota_enable_default();
    esp_rmaker_timezone_service_enable();
    esp_rmaker_schedule_enable();
    esp_rmaker_scenes_enable();

    /* 以太网一般不做 Wi-Fi reset，只保留 reboot + factory reset */
    esp_rmaker_system_serv_config_t sys_cfg = {
        .flags = SYSTEM_SERV_FLAG_REBOOT | SYSTEM_SERV_FLAG_FACTORY_RESET,
        .reboot_seconds = 2, .reset_seconds = 2, .reset_reboot_seconds = 2,
    };
    esp_rmaker_system_service_enable(&sys_cfg);

    /* on-network chal_resp 的 PoP（与 App/CLI 输入一致，需 8 字符） */
    esp_rmaker_local_ctrl_set_pop(CONFIG_EXAMPLE_LOCAL_CTRL_POP);
    app_on_network_chal_resp_init();   /* 注册 IP_EVENT，拿 IP 后启 chal_resp */

    esp_rmaker_start();

#ifdef CONFIG_EXAMPLE_ENABLE_WIFI
    /* 双网络：Wi-Fi 起不来不致命，继续用以太网 */
    if (app_network_start(POP_TYPE_RANDOM) != ESP_OK) {
        ESP_LOGE(TAG, "Could not start Wi-Fi. Continuing with Ethernet only...");
    }
#endif
    /* 阻塞等以太网连上并拿到 IP；RainMaker 用先连上的网络 */
    err = app_ethernet_start();
    if (err != ESP_OK) {
#ifdef CONFIG_EXAMPLE_ENABLE_WIFI
        ESP_LOGW(TAG, "Could not start Ethernet. Wi-Fi may be available as fallback.");
#else
        ESP_LOGE(TAG, "Could not start Ethernet. Aborting!!!");
        abort();
#endif
    }
}
```

### 4. on-network challenge-response 的两种实现

`examples/ethernet_switch/main/app_on_network_test.c` 用编译宏二选一，逻辑都是"拿到 IP 后启服务，mDNS 可被发现"：

**(a) 独立 HTTP 服务**（`CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE`）

```c
#include <esp_rmaker_on_network_chal_resp.h>

static void on_got_ip(void *arg, esp_event_base_t base, int32_t id, void *data) {
    esp_rmaker_on_network_chal_resp_config_t cfg = ESP_RMAKER_ON_NETWORK_CHAL_RESP_DEFAULT_CONFIG();
    /* cfg.port / cfg.sec_ver / cfg.pop 可覆盖 */
    esp_rmaker_on_network_chal_resp_start(&cfg);   /* 启动后即可被 mDNS 发现 */
}
esp_event_handler_register(IP_EVENT, IP_EVENT_ETH_GOT_IP, on_got_ip, NULL);
```

**(b) 复用 Local Control 通道**（`CONFIG_ESP_RMAKER_LOCAL_CTRL_CHAL_RESP_ENABLE`，示例默认）

拿到 IP **且** local control 已启动后，调 `esp_rmaker_local_ctrl_enable_chal_resp(instance_name)`。实例名按以太网 MAC 后 3 字节拼成 `PROV_xxyyzz`：

```c
char instance_name[16];
uint8_t mac[6];
esp_read_mac(mac, ESP_MAC_ETH);
snprintf(instance_name, sizeof(instance_name), "PROV_%02X%02X%02X", mac[3], mac[4], mac[5]);
esp_rmaker_local_ctrl_enable_chal_resp(instance_name);
```

### 5. 设备端配网（CLI 侧）

设备插上网线、拿到 IP 后，在主机上用 CLI 发现并映射：

```bash
esp-rainmaker-cli provision --transport on-network
```

PoP 与固件里的 `CONFIG_EXAMPLE_LOCAL_CTRL_POP`（默认 `abcd1234`，须 8 字符）一致。映射成功后 chal_resp 服务会按 timer 清理。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 以太网拿不到 IP | PHY 引脚 / 地址 / 时钟方向错 | 按原理图设 `EXAMPLE_ETH_*` GPIO，PHY 地址 -1 自动探测；ESP32 勿选 RMII CLK OUTPUT |
| 编译报 claiming 相关错 | 以太网仍开着 Self/Assisted Claim | 设 `CONFIG_ESP_RMAKER_NO_CLAIM=y`（claiming 不支持以太网） |
| on-network chal_resp 不启动 | 两选项都没开，或互斥项同时开 | 二选一：`LOCAL_CTRL_CHAL_RESP_ENABLE` 或 `ON_NETWORK_CHAL_RESP_ENABLE` |
| 设备 mDNS 不可见 | chal_resp 在拿到 IP 前就启了 | 注册 `IP_EVENT_ETH_GOT_IP`（双网络再加 `IP_EVENT_STA_GOT_IP`），回调里再启 |
| PoP 校验失败 | App/CLI 输入与固件不一致 | 固件 `CONFIG_EXAMPLE_LOCAL_CTRL_POP` 须 8 字符，与 CLI 输入一致 |
| 双网络下 Wi-Fi 起不来直接 abort | 把 Wi-Fi 失败当致命 | 仿照示例：Wi-Fi 失败只打日志，继续用以太网 |
| PPI / MAC 异常 | 以太网下仍发 MQTT username(PPI) | `CONFIG_ESP_RMAKER_MQTT_SEND_USERNAME=n` |
| 同时编译报 protocomm_httpd 冲突 | local_ctrl 与 on-network chal_resp 都开 | 二者互斥，只留一个 |

## 参考项目

- `examples/ethernet_switch/` — 以太网 Switch 完整示例
  - `examples/ethernet_switch/main/app_main.c` — 双网络 init 顺序、PoP、chal_resp init
  - `examples/ethernet_switch/main/app_on_network_test.c` — on-network chal_resp 两种实现的完整样板
  - `examples/ethernet_switch/main/Kconfig.projbuild` — `EXAMPLE_ENABLE_WIFI`、`EXAMPLE_LOCAL_CTRL_POP`
  - `examples/ethernet_switch/sdkconfig.defaults` — 以太网 + No Claim + chal_resp 配置
- `examples/ethernet_switch/components/app_ethernet/` — 以太网驱动封装
  - `app_ethernet.h`（`app_ethernet_init/start/stop`）、`Kconfig.projbuild`（PHY/RMII GPIO）
- `recipes/claiming_and_provisioning.md` — Wi-Fi 配网 / claiming 主流程（对比参照）
- `recipes/local_control.md` — local control 安全等级与 chal_resp 端点
