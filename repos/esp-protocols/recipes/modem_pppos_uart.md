# 蜂窝模组 PPPoS 拨号上网（UART）

> **适用摘要**: 通过 UART 连接蜂窝模组（如 SIM7600/SIM800/BG96），用 esp_modem 创建 PPP 网络接口并拨号上网，读取信号质量与 SIM 状态，切换 DATA 模式获取 IP。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-protocols/resources/`, source/examples in `repos/esp-protocols/`, and this recipe path `repos/esp-protocols/recipes/modem_pppos_uart.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "esp_modem 拨号上网"
- "PPPoS 上网"
- "SIM7600/SIM800 连接网络"
- "蜂窝模组拿 IP"
- "esp_modem UART 配置"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF | v5.x，CMake 构建 |
| 组件依赖 | `espressif/esp_modem`（>=2.0.2） |
| 硬件 | 蜂窝模组通过 UART 接入 ESP32（TX/RX/可选 RTS/CTS） |
| 参考示例 | `components/esp_modem/examples/pppos_client/` |

## 分步说明

### 1. 依赖与 menuconfig 关键项

在 `main/idf_component.yml`：
```yaml
dependencies:
  espressif/esp_modem: "^2.0.2"
  idf:
    version: ">=5.0"
```

确保 `CONFIG_ESP_MODEM_USE_PPP_MODE=y`（默认），它会自动 select `LWIP_PPP_SUPPORT`。

### 2. 初始化 netif 与事件，注册 IP/PPP 事件

```c
#include "esp_netif.h"
#include "esp_netif_ppp.h"
#include "esp_event.h"
#include "esp_log.h"
#include "freertos/event_groups.h"

static const char *TAG = "pppos";
static EventGroupHandle_t event_group = NULL;
static const int CONNECT_BIT = BIT0;
static const int DISCONNECT_BIT = BIT1;

static void on_ppp_changed(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    ESP_LOGI(TAG, "PPP state changed event %" PRIu32, id);
    if (id == NETIF_PPP_ERRORUSER) {
        ESP_LOGI(TAG, "User interrupted event from netif");
    }
}

static void on_ip_event(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (id == IP_EVENT_PPP_GOT_IP) {
        ip_event_got_ip_t *event = (ip_event_got_ip_t *)data;
        ESP_LOGI(TAG, "GOT IP: " IPSTR, IP2STR(&event->ip_info.ip));
        xEventGroupSetBits(event_group, CONNECT_BIT);
    } else if (id == IP_EVENT_PPP_LOST_IP) {
        ESP_LOGI(TAG, "Lost IP");
        xEventGroupSetBits(event_group, DISCONNECT_BIT);
    }
}
```

### 3. 创建 PPP netif 与 DCE（核心顺序）

```c
#include "esp_modem_api.h"
#include "esp_modem_config.h"
#include "esp_modem_dce_config.h"

void app_main(void)
{
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, ESP_EVENT_ANY_ID, &on_ip_event, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(NETIF_PPP_STATUS, ESP_EVENT_ANY_ID, &on_ppp_changed, NULL));

    event_group = xEventGroupCreate();

    esp_modem_dce_config_t dce_config = ESP_MODEM_DCE_DEFAULT_CONFIG(CONFIG_EXAMPLE_MODEM_PPP_APN);
    esp_netif_config_t netif_ppp_config = ESP_NETIF_DEFAULT_PPP();
    esp_netif_t *esp_netif = esp_netif_new(&netif_ppp_config);   // 1) 先建 netif
    assert(esp_netif);

    esp_modem_dte_config_t dte_config = ESP_MODEM_DTE_DEFAULT_CONFIG();
    dte_config.uart_config.tx_io_num = CONFIG_EXAMPLE_MODEM_UART_TX_PIN;
    dte_config.uart_config.rx_io_num = CONFIG_EXAMPLE_MODEM_UART_RX_PIN;
    dte_config.uart_config.baud_rate = 115200;

    // 2) 用具体模组枚举创建 DCE
    esp_modem_dce_t *dce = esp_modem_new_dev(ESP_MODEM_DCE_SIM7600, &dte_config, &dce_config, esp_netif);
    assert(dce);
```

### 4. AT 命令交互（COMMAND 模式下）

```c
    // 检查 SIM PIN（如需要）
    bool pin_ok = false;
    if (esp_modem_read_pin(dce, &pin_ok) == ESP_OK && !pin_ok) {
        esp_modem_set_pin(dce, CONFIG_EXAMPLE_SIM_PIN);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }

    // 信号质量
    int rssi, ber;
    if (esp_modem_get_signal_quality(dce, &rssi, &ber) == ESP_OK) {
        ESP_LOGI(TAG, "Signal: rssi=%d, ber=%d", rssi, ber);
    }

    // IMSI（缓冲 >= ESP_MODEM_C_API_STR_BUF_SIZE）
    char imsi[ESP_MODEM_C_API_STR_BUF_SIZE];
    if (esp_modem_get_imsi(dce, imsi) == ESP_OK) {
        ESP_LOGI(TAG, "IMSI=%s", imsi);
    }
```

### 5. 切换 DATA 模式拨号，等待 IP

```c
    esp_err_t err = esp_modem_set_mode(dce, ESP_MODEM_MODE_DATA);
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "set DATA mode failed: %s", esp_err_to_name(err));
        return;
    }

    ESP_LOGI(TAG, "Waiting for IP address");
    xEventGroupWaitBits(event_group, CONNECT_BIT | DISCONNECT_BIT, pdFALSE, pdFALSE,
                        pdMS_TO_TICKS(60000));

    // 拨通后可执行标准网络操作，如 ping / socket
    int ret;
    esp_console_run("ping www.espressif.com", &ret);

    // 暂停 netif 发 AT（不切模式）
    esp_modem_pause_net(dce, true);
    esp_modem_get_signal_quality(dce, &rssi, &ber);
    esp_modem_pause_net(dce, false);

    esp_modem_destroy(dce);
    esp_netif_destroy(esp_netif);
}
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_modem_new_dev` 返回 NULL | netif 未建或内存不足 | 先 `esp_netif_new()`，再创建 DCE |
| 一直拿不到 IP | APN 错误/SIM 未注册/信号弱 | 核对 APN、`esp_modem_get_signal_quality`、确认 SIM 已注网 |
| AT 命令在 DATA 模式后失败 | UART 已变成 PPP 流 | 用 `esp_modem_pause_net(true)` 临时发 AT，或切回 COMMAND 模式 |
| IMSI/IMEI 被截断 | 缓冲区过小 | 用 `char buf[ESP_MODEM_C_API_STR_BUF_SIZE]` |
| OTA 频繁 buffer overflow | UART ISR 处理不及时 | ISR 放 IRAM、增大 `rx_buffer_size`、提高 `task_priority`、启用 HW 流控 |

## 参考

- `components/esp_modem/examples/pppos_client/main/pppos_client_main.c` — 完整 PPPoS 客户端（C）
- `components/esp_modem/include/esp_modem_c_api_types.h` — DCE 枚举与生命周期 API
- `components/esp_modem/command/include/esp_modem_api.h` — AT 命令 API
- `docs/esp_modem/en/README.rst` — 模式状态机与 Known issues
- `docs/esp_modem/en/api_docs.rst` — C API 文档
