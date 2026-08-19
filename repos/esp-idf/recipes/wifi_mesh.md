# Wi-Fi Mesh（ESP-WIFI-MESH）

> **适用摘要**: 用 ESP-WIFI-MESH 协议栈组建自组织 Wi-Fi 网状网络：`esp_mesh_init`、`esp_mesh_set_config`（mesh ID / 路由器 / SoftAP 凭据）、`esp_mesh_start`、收发（`esp_mesh_send` / `esp_mesh_recv`）、路由表查询、事件处理（`MESH_EVENT_*`）。适配自 `examples/mesh/internal_communication`。

> Version: ESP-IDF version used by the project.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Wi-Fi Mesh"
- "ESP-WIFI-MESH"
- "mesh 组网 / 节点通信"
- "esp_mesh"
- "自组织网络 / 多跳"
- "mesh 根节点 / 路由表"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | 需 Wi-Fi（esp32/s2/s3/c2/c3/c5/c6/c61 等；esp32h2/p4 无 Wi-Fi） |
| Kconfig | Wi-Fi + Mesh 启用。menuconfig：`Component config → Wi-Fi`；Mesh 相关 `CONFIG_MESH_*`（如 `CONFIG_MESH_TOPOLOGY` tree/chain、`CONFIG_MESH_MAX_LAYER`、`CONFIG_MESH_ROUTE_TABLE_SIZE`） |
| 组件 | `esp_wifi`、`esp_event`、`esp_netif`、`nvs_flash`、`mesh`（提供 `esp_mesh.h`） |
| 头文件 | `esp_wifi.h`、`esp_event.h`、`esp_netif.h`、`esp_mesh.h`（必要时 `esp_mesh_internal.h`）、`nvs_flash.h` |
| 参考 | `examples/mesh/internal_communication`（P2P 收发）、`ip_internal_network`（根节点 IP 上行）、`manual_networking` |

> **拓扑**：默认树形（tree）。所有节点需同一 mesh ID + 同一路由器 SSID/密码 + 同一信道。根节点（root）连路由器并拿 IP，其余节点通过父节点多跳上行。

## 分步说明

### 初始化顺序：netif → 事件循环 → Wi-Fi → mesh

ESP-WIFI-MESH 建在 Wi-Fi 之上，故先建 Wi-Fi 协议栈（但**不连**路由器），再初始化 mesh 层。

```c
#include "esp_wifi.h"
#include "esp_event.h"
#include "esp_netif.h"
#include "esp_mesh.h"
#include "nvs_flash.h"
#include "esp_log.h"
#include "esp_mac.h"

#define MESH_TAG      "MESH"
static const uint8_t MESH_ID[6] = { 0x77, 0x77, 0x77, 0x77, 0x77, 0x77 };
static bool is_mesh_connected = false;
static int mesh_layer = -1;
static esp_netif_t *netif_sta = NULL;

void app_main(void)
{
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    /* 为 mesh 创建 STA + SoftAP netif（mesh 同时用两者：STA 连父节点，AP 给子节点连入） */
    ESP_ERROR_CHECK(esp_netif_create_default_wifi_mesh_netifs(&netif_sta, NULL));

    /* Wi-Fi 初始化并 start（不 connect —— mesh 接管连接） */
    wifi_init_config_t config = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&config));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &ip_event_handler, NULL));
    ESP_ERROR_CHECK(esp_wifi_set_storage(WIFI_STORAGE_FLASH));
    ESP_ERROR_CHECK(esp_wifi_start());

    /* mesh 初始化 */
    ESP_ERROR_CHECK(esp_mesh_init());
    ESP_ERROR_CHECK(esp_event_handler_register(MESH_EVENT, ESP_EVENT_ANY_ID, &mesh_event_handler, NULL));

    /* 拓扑与层级（tree/chain 由 CONFIG_MESH_TOPOLOGY 选） */
    ESP_ERROR_CHECK(esp_mesh_set_topology(CONFIG_MESH_TOPOLOGY));
    ESP_ERROR_CHECK(esp_mesh_set_max_layer(CONFIG_MESH_MAX_LAYER));
    ESP_ERROR_CHECK(esp_mesh_set_vote_percentage(1));   /* 100% 投票，根节点选举 */
    ESP_ERROR_CHECK(esp_mesh_set_xon_q_size(128));

    /* 配置 mesh ID / 路由器 / SoftAP 凭据 */
    mesh_cfg_t cfg = MESH_INIT_CONFIG_DEFAULT();
    memcpy((uint8_t *)&cfg.mesh_id, MESH_ID, 6);
    cfg.channel = CONFIG_MESH_CHANNEL;
    cfg.router.ssid_len = strlen(CONFIG_MESH_ROUTER_SSID);
    memcpy((uint8_t *)&cfg.router.ssid, CONFIG_MESH_ROUTER_SSID, cfg.router.ssid_len);
    memcpy((uint8_t *)&cfg.router.password, CONFIG_MESH_ROUTER_PASSWD, strlen(CONFIG_MESH_ROUTER_PASSWD));
    ESP_ERROR_CHECK(esp_mesh_set_ap_authmode(CONFIG_MESH_AP_AUTHMODE));
    cfg.mesh_ap.max_connection = CONFIG_MESH_AP_CONNECTIONS;
    cfg.mesh_ap.nonmesh_max_connection = CONFIG_MESH_NON_MESH_AP_CONNECTIONS;
    memcpy((uint8_t *)&cfg.mesh_ap.password, CONFIG_MESH_AP_PASSWD, strlen(CONFIG_MESH_AP_PASSWD));

    ESP_ERROR_CHECK(esp_mesh_set_config(&cfg));

    /* 启动 mesh（开始选举根节点、自动组网） */
    ESP_ERROR_CHECK(esp_mesh_start());
}
```

> `esp_netif_create_default_wifi_mesh_netifs(&sta, &ap)` 一次创建 mesh 用的 STA 与 SoftAP netif。根节点当选后会自动启 DHCP client 拿路由器 IP。

### 事件处理：父节点连接、层级变化、根节点

```c
static void mesh_event_handler(void *arg, esp_event_base_t event_base,
                               int32_t event_id, void *event_data)
{
    mesh_addr_t id = {0};
    switch (event_id) {
    case MESH_EVENT_STARTED:
        esp_mesh_get_id(&id);
        is_mesh_connected = false;
        mesh_layer = esp_mesh_get_layer();
        ESP_LOGI(MESH_TAG, "mesh started, layer=%d", mesh_layer);
        break;

    case MESH_EVENT_PARENT_CONNECTED: {
        /* 已连上父节点（或自己当选根节点）。此后可收发 */
        mesh_event_connected_t *conn = (mesh_event_connected_t *)event_data;
        mesh_layer = conn->self_layer;
        is_mesh_connected = true;
        ESP_LOGI(MESH_TAG, "parent connected, layer=%d %s",
                 mesh_layer, esp_mesh_is_root() ? "<ROOT>" : "");
        /* 根节点需启 DHCP client 拿路由器 IP */
        if (esp_mesh_is_root()) {
            esp_netif_dhcpc_stop(netif_sta);
            esp_netif_dhcpc_start(netif_sta);
        }
        esp_mesh_comm_p2p_start();   /* 启动 P2P 收发任务（见下） */
        break;
    }

    case MESH_EVENT_PARENT_DISCONNECTED: {
        mesh_event_disconnected_t *disc = (mesh_event_disconnected_t *)event_data;
        is_mesh_connected = false;
        ESP_LOGI(MESH_TAG, "parent disconnected, reason=%d", disc->reason);
        mesh_layer = esp_mesh_get_layer();
        break;
    }

    case MESH_EVENT_LAYER_CHANGE: {
        mesh_event_layer_change_t *lc = (mesh_event_layer_change_t *)event_data;
        mesh_layer = lc->new_layer;
        ESP_LOGI(MESH_TAG, "layer -> %d %s", mesh_layer,
                 esp_mesh_is_root() ? "<ROOT>" : "");
        break;
    }

    case MESH_EVENT_ROOT_ADDRESS: {
        /* 拿到根节点地址 */
        mesh_event_root_address_t *ra = (mesh_event_root_address_t *)event_data;
        ESP_LOGI(MESH_TAG, "root address: " MACSTR, MAC2STR(ra->addr));
        break;
    }

    case MESH_EVENT_ROUTING_TABLE_ADD: {
        mesh_event_routing_table_change_t *rt = (mesh_event_routing_table_change_t *)event_data;
        ESP_LOGI(MESH_TAG, "routing table +%d (total=%d)", rt->rt_size_change, rt->rt_size_new);
        break;
    }

    case MESH_EVENT_SCAN_DONE: {
        mesh_event_scan_done_t *sd = (mesh_event_scan_done_t *)event_data;
        ESP_LOGI(MESH_TAG, "scan done, found %d", sd->number);
        break;
    }

    default:
        break;
    }
}

static void ip_event_handler(void *arg, esp_event_base_t event_base,
                             int32_t event_id, void *event_data)
{
    ip_event_got_ip_t *e = (ip_event_got_ip_t *)event_data;
    ESP_LOGI(MESH_TAG, "got IP: " IPSTR, IP2STR(&e->ip_info.ip));
}
```

### 收发：`esp_mesh_send` / `esp_mesh_recv`

mesh 节点间用 `mesh_data_t`（含数据、协议、tos）收发。根节点可遍历整个路由表向每个节点单播。

```c
#define RX_SIZE  1500
#define TX_SIZE  1460
static uint8_t tx_buf[TX_SIZE], rx_buf[RX_SIZE];

/* 发送任务（仅根节点向所有节点单播示意；普通节点可向 root 发） */
void esp_mesh_p2p_tx_main(void *arg)
{
    mesh_addr_t route_table[CONFIG_MESH_ROUTE_TABLE_SIZE];
    int route_table_size = 0;
    mesh_data_t data = { .data = tx_buf, .size = sizeof(tx_buf),
                         .proto = MESH_PROTO_BIN, .tos = MESH_TOS_P2P };

    while (true) {
        if (!esp_mesh_is_root()) { vTaskDelay(pdMS_TO_TICKS(10000)); continue; }  /* 仅根节点发 */
        esp_mesh_get_routing_table((mesh_addr_t *)&route_table,
                CONFIG_MESH_ROUTE_TABLE_SIZE * 6, &route_table_size);
        for (int i = 0; i < route_table_size; i++) {
            /* MESH_DATA_P2P：点对点；MESH_DATA_FROMDS / TODS：与外部 IP 网络交互 */
            esp_mesh_send(&route_table[i], &data, MESH_DATA_P2P, NULL, 0);
        }
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
    vTaskDelete(NULL);
}

/* 接收任务（所有节点都跑，阻塞等数据） */
void esp_mesh_p2p_rx_main(void *arg)
{
    mesh_addr_t from;
    int flag = 0;
    mesh_data_t data = { .data = rx_buf, .size = RX_SIZE };

    while (true) {
        data.size = RX_SIZE;
        esp_err_t err = esp_mesh_recv(&from, &data, portMAX_DELAY, &flag, NULL, 0);
        if (err != ESP_OK || data.size == 0) continue;
        ESP_LOGI(MESH_TAG, "recv %d bytes from " MACSTR, data.size, MAC2STR(from.addr));
        /* 处理 data.data ... */
    }
    vTaskDelete(NULL);
}

void esp_mesh_comm_p2p_start(void)
{
    static bool started = false;
    if (!started) {
        started = true;
        xTaskCreate(esp_mesh_p2p_tx_main, "MPTX", 3072, NULL, 5, NULL);
        xTaskCreate(esp_mesh_p2p_rx_main, "MPRX", 3072, NULL, 5, NULL);
    }
}
```

### 关键 API

```c
/* esp_mesh.h */
esp_err_t esp_mesh_init(void);
esp_err_t esp_mesh_set_config(const mesh_cfg_t *config);
esp_err_t esp_mesh_get_config(mesh_cfg_t *config);
esp_err_t esp_mesh_set_topology(mesh_type_t topo);    /* MESH_TOPOLOGY_TREE / CHAIN */
esp_err_t esp_mesh_set_max_layer(int max_layer);
esp_err_t esp_mesh_start(void);
esp_err_t esp_mesh_stop(void);
esp_err_t esp_mesh_send(const mesh_addr_t *to, const mesh_data_t *data,
        int flag, const mesh_opt_t opt[], int opt_count);
esp_err_t esp_mesh_recv(mesh_addr_t *from, mesh_data_t *data, int timeout_ms,
        int *flag, mesh_opt_t opt[], int opt_count);
bool      esp_mesh_is_root(void);
int       esp_mesh_get_layer(void);
esp_err_t esp_mesh_get_id(mesh_addr_t *id);
esp_err_t esp_mesh_get_routing_table(mesh_addr_t *route_table, int size, int *number);
int       esp_mesh_get_routing_table_size(void);
esp_err_t esp_mesh_get_parent_bssid(mesh_addr_t *bssid);
esp_err_t esp_mesh_set_vote_percentage(float percentage);
esp_err_t esp_mesh_set_active_duty_cycle(int dev_duty, mesh_ps_type_t type);   /* 配合 Mesh 省电 */
/* 事件 base: MESH_EVENT; 关键事件见上 handler */
/* netif: esp_netif_create_default_wifi_mesh_netifs(sta_out, ap_out) */
```

关键事件链：`MESH_EVENT_STARTED`（mesh 已起）→ 自动选举根节点 → `MESH_EVENT_PARENT_CONNECTED`（连上父节点或自己成根，layer 已定）→ `MESH_EVENT_ROOT_ADDRESS`（广播根地址）→ 根节点 `IP_EVENT_STA_GOT_IP`（拿路由器 IP）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 节点不组网 / 找不到父节点 | mesh ID / 信道 / 路由器 SSID 不一致 | 所有节点同 `MESH_ID`、同 `channel`、同路由器 SSID/密码；信道须与路由器一致 |
| 两个节点都当根 / 选举抖动 | `vote_percentage` 太低 | 设 `esp_mesh_set_vote_percentage(1)`（100%）减少分裂 |
| 根节点拿不到 IP | DHCP client 未启 | `PARENT_CONNECTED` 且 `is_root()` 时 `dhcpc_stop` 再 `dhcpc_start` |
| `esp_mesh_send` 报错 | 未连父节点或参数错 | 先等 `PARENT_CONNECTED`；`MESH_DATA_P2P` 的 `to` 须是路由表内地址 |
| 收不到数据 | recv 任务未起或超时太短 | 用 `portMAX_DELAY` 阻塞等；确认 `PARENT_CONNECTED` 后才启收发任务 |
| `esp_wifi_*` API 与 mesh 冲突 | 自组织模式下调了 Wi-Fi API | `esp_mesh_start` 后、`esp_mesh_stop` 前禁止调 `esp_wifi_connect/scan` 等（mesh 接管） |
| 路由表不更新 / 节点丢失 | `CONFIG_MESH_ROUTE_TABLE_SIZE` 太小 | 按节点数调大该 Kconfig；增大 `set_xon_q_size` |
| 用了 `create_default_wifi_sta/ap` | netif 类型错 | mesh 用专用 `esp_netif_create_default_wifi_mesh_netifs` |

## 参考

- `examples/mesh/internal_communication` — 节点间 P2P 收发（本 recipe 主来源，含 `mesh_light` 控制演示）
- `examples/mesh/ip_internal_network` — 根节点 IP 上行（mesh 内部 IP 路由）
- `examples/mesh/manual_networking` — 手动选父节点组网
- ESP-IDF `examples/mesh/internal_communication/main/mesh_main.c`、`components/esp_wifi/include/esp_mesh.h`
- 文档 `docs/en/api-reference/network/esp-wifi-mesh.rst`（编程指南）、`docs/en/api-guides/esp-wifi-mesh.rst`（协议架构）
