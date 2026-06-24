# ESP-NOW Wi-Fi 配网（Provisioning）

> **适用摘要**: 已联网的 responder 广播配网 beacon 并在回调校验 initiator 后下发 Wi-Fi 配置；未联网的 initiator 扫描 beacon、发送身份请求并应用收到的 SSID/密码（参考 `examples/provisioning`）。

## 触发意图

- "ESP-NOW 配网"
- "Wi-Fi provisioning"
- "espnow_prov"
- "SSID 分发"
- "一键配网"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `espnow_prov.h` |
| 参考示例 | `examples/provisioning/main/app_main.c` |

## 分步说明

### 1. 公共初始化

```c
#include "espnow.h"
#include "espnow_prov.h"
#include "espnow_storage.h"

void app_main(void)
{
    espnow_storage_init();
    app_wifi_init();

    espnow_config_t cfg = ESPNOW_INIT_CONFIG_DEFAULT();
    espnow_init(&cfg);

#ifdef CONFIG_APP_ESPNOW_PROV_INITIATOR
    app_initiator_init();
#elif CONFIG_APP_ESPNOW_PROV_RESPONDER
    app_responder_init();
#endif
}
```

### 2. Responder：广播 beacon 并校验 initiator 后下发 Wi-Fi 配置

```c
static esp_err_t app_prov_recv_cb(uint8_t *src_addr, void *data,
                                  size_t size, wifi_pkt_rx_ctrl_t *rx_ctrl)
{
    espnow_prov_initiator_t *info = (espnow_prov_initiator_t *)data;
    // 校验 initiator 身份：返回 ESP_OK 才回发 Wi-Fi 配置，否则 ESP_FAIL
    ESP_LOGI(TAG, "initiator product_id=%s device=%s auth=%d",
             info->product_id, info->device_name, info->auth_mode);
    if (strcmp(info->product_id, "my_product") != 0) return ESP_FAIL;
    return ESP_OK;
}

static esp_err_t app_responder_init(void)
{
    espnow_prov_responder_t responder_info = { .product_id = "my_product" };
    espnow_prov_wifi_t wifi_config = {
        .sta = {
            .ssid     = CONFIG_APP_ESPNOW_WIFI_SSID,
            .password = CONFIG_APP_ESPNOW_WIFI_PASSWORD,
        },
    };
    // beacon 发送时长 30s，第 4 参为 initiator 信息校验回调
    return espnow_prov_responder_start(&responder_info, pdMS_TO_TICKS(30 * 1000),
                                       &wifi_config, app_prov_recv_cb);
}
```

### 3. Initiator：扫描 beacon → 发送身份 → 应用收到的 Wi-Fi 配置

```c
static esp_err_t app_prov_recv_cb(uint8_t *src_addr, void *data,
                                  size_t size, wifi_pkt_rx_ctrl_t *rx_ctrl)
{
    espnow_prov_wifi_t *wifi_config = (espnow_prov_wifi_t *)data;
    ESP_LOGI(TAG, "got wifi ssid=%s", wifi_config->sta.ssid);

    // 注意：wifi_config 是 packed 结构，拷贝后再用
    wifi_config_t sta_config = {0};
    memcpy(&sta_config.sta, &wifi_config->sta, sizeof(sta_config.sta));
    ESP_ERROR_CHECK(esp_wifi_set_config(WIFI_IF_STA, &sta_config));
    ESP_ERROR_CHECK(esp_wifi_connect());
    return ESP_OK;
}

static esp_err_t app_initiator_init(void)
{
    espnow_prov_initiator_t initiator_info = { .product_id = "my_product" };
    espnow_addr_t responder_addr = {0};
    espnow_prov_responder_t responder_info = {0};
    wifi_pkt_rx_ctrl_t rx_ctrl = {0};

    for (;;) {
        esp_err_t ret = espnow_prov_initiator_scan(responder_addr, &responder_info,
                                                   &rx_ctrl, portMAX_DELAY);
        ESP_ERROR_CONTINUE(ret != ESP_OK, "scan");
        ESP_LOGI(TAG, "responder " MACSTR " product=%s",
                 MAC2STR(responder_addr), responder_info.product_id);

        ret = espnow_prov_initiator_send(responder_addr, &initiator_info,
                                         app_prov_recv_cb, pdMS_TO_TICKS(3 * 1000));
        ESP_ERROR_CONTINUE(ret != ESP_OK, "send");
        break;
    }
    return ESP_OK;
}
```

### 关键结构（来自 espnow_prov.h）

| 结构 | 用途 |
|---|---|
| `espnow_prov_initiator_t` | initiator 身份（product_id/device_name/auth_mode + secret + custom_data） |
| `espnow_prov_responder_t` | responder 身份（product_id/device_name） |
| `espnow_prov_wifi_t` | Wi-Fi 配置（mode + ap/sta 联合体 + token + custom_data） |
| `espnow_prov_auth_mode_t` | `ESPNOW_PROV_AUTH_PRODUCT` / `_DEVICE` / `_CERT` |

> `ESPNOW_PROV_CUSTOM_MAX_SIZE = 64`，自定义数据附在结构末尾（柔性数组 `custom_data[0]`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| responder 不下发配置 | 回调返回非 ESP_OK | 校验通过返回 ESP_OK |
| 取 packed 成员地址告警 | 直接 `&wifi_config->sta` | 先 memcpy 到本地 `wifi_config_t` 再用（见示例注释） |
| initiator 连不上 AP | SSID/密码错或信道不符 | 检查 responder 下发的 `wifi_config` |
| 扫描不到 beacon | responder 未启动或已超时 | `espnow_prov_responder_start` 的 `wait_ticks` 内才有 beacon |
| 自定义数据丢失 | 未填 `custom_size` | 设 `initiator_info.custom_size` 后填 `custom_data` |

## 参考

- `examples/provisioning/main/app_main.c` — initiator/responder 双角色示例
- `src/provisioning/include/espnow_prov.h`
- `examples/solution/` — 综合示例含配网与控制组合
