# Zigbee 网关与 RCP（Radio Co-Processor）

> **适用摘要**: 在没有 802.15.4 radio 的主芯片（ESP32-C3/S3/P4）上，通过 UART 连接一块 ESP32-H2/C6（烧录 ot_rcp 固件作为 Radio Co-Processor）构建 Zigbee 网关，可选 Wi-Fi/以太网回程、软件共存与 RCP 自动升级。

## 触发意图

- "做 Zigbee 网关"
- "RCP 配置"
- "ESP32 + H2 网关"
- "Wi-Fi Zigbee 共存"
- "Zigbee 接入云"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | 主控（ESP32-C3/S3/P4 等）+ RCP（ESP32-H2 或 C6，烧 `ot_rcp`） |
| 软件 | ESP-IDF v5.2+，`esp-zigbee-lib >=2.0.0`；可选 `CONFIG_EXAMPLE_CONNECT_WIFI`/`ETHERNET` |
| 参考项目 | `examples/zigbee_gateway/`（含 `generate_rcp_image.py`、`sdkconfig.defaults.esp32s3`、`esp32p4`） |

## 分步说明

### 1. 平台配置：UART_RCP 模式（非 15.4 SoC 分支）

主控没有 `CONFIG_SOC_IEEE802154_SUPPORTED`，故走 `#else` 分支，配置 UART 引脚与 460800 波特率。

```c
// main/zigbee_gateway.h
#define ESP_ZIGBEE_ZC_CONFIG()                          \
    { .device_type = EZB_NWK_DEVICE_TYPE_COORDINATOR,   \
      .install_code_policy = false,                     \
      .zczr_config = { .max_children = 10, }, }

#define ESP_ZIGBEE_UART_CONFIG()                        \
    {                                                   \
        .port = 1,                                      \
        .uart_config = {                                \
            .baud_rate = 460800,                        \
            .data_bits = UART_DATA_8_BITS,              \
            .parity = UART_PARITY_DISABLE,              \
            .stop_bits = UART_STOP_BITS_1,              \
            .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,      \
            .rx_flow_ctrl_thresh = 0,                   \
            .source_clk = UART_SCLK_DEFAULT,            \
        },                                              \
        .rx_pin = CONFIG_PIN_TO_RCP_TX,                 \
        .tx_pin = CONFIG_PIN_TO_RCP_RX,                 \
    }

#define ESP_ZIGBEE_PLATFORM_CONFIG()                                 \
    {                                                                \
        .storage_partition_name = ESP_ZIGBEE_STORAGE_PARTITION_NAME, \
        .radio_config = {                                            \
            .radio_mode = ESP_ZIGBEE_RADIO_MODE_UART_RCP,            \
            .radio_uart_config = ESP_ZIGBEE_UART_CONFIG(),           \
        },                                                           \
    }

#define ESP_ZIGBEE_DEFAULT_CONFIG()                      \
    { .device_config = ESP_ZIGBEE_ZC_CONFIG(),           \
      .platform_config = ESP_ZIGBEE_PLATFORM_CONFIG(), }
```

### 2. RCP 自动升级配置（可选，`CONFIG_ZIGBEE_GW_AUTO_UPDATE_RCP`）

定义 RCP 升级参数（reset/boot 引脚、波特率、固件目录、目标芯片）。

```c
#define ESP_ZIGBEE_RCP_CONFIG()                     \
    {                                               \
        .rcp_type = RCP_TYPE_UART,                  \
        .uart_rx_pin = CONFIG_PIN_TO_RCP_TX,        \
        .uart_tx_pin = CONFIG_PIN_TO_RCP_RX,        \
        .uart_port = 1,                             \
        .uart_baudrate = 115200,                    \
        .reset_pin = CONFIG_PIN_TO_RCP_RESET,       \
        .boot_pin  = CONFIG_PIN_TO_RCP_BOOT,        \
        .update_baudrate = 460800,                  \
        .firmware_dir = "/rcp_fw/ot_rcp",           \
        .target_chip = ESP_ZIGBEE_RCP_TARGET_CHIP,  \
    }
```

### 3. 网络回程与共存（Wi-Fi 场景）

C6 等支持 15.4 + Wi-Fi 共存的芯片可启用 `CONFIG_ESP_COEX_SW_COEXIST_ENABLE`，在联网前调 `esp_coex_wifi_i154_enable()`。

```c
static esp_err_t esp_zigbee_connect_netif(void)
{
    ESP_RETURN_ON_ERROR(esp_netif_init(), TAG, "netif init");
    ESP_RETURN_ON_ERROR(esp_event_loop_create_default(), TAG, "event loop");
#if CONFIG_ESP_COEX_SW_COEXIST_ENABLE
    esp_coex_wifi_i154_enable();
#endif
    ESP_RETURN_ON_ERROR(example_connect(), TAG, "connect");
#if CONFIG_EXAMPLE_CONNECT_WIFI
    ESP_RETURN_ON_ERROR(esp_wifi_set_ps(WIFI_PS_MIN_MODEM), TAG, "wifi ps");
#endif
    return ESP_OK;
}
```

### 4. 主流程（ZC + 网关 endpoint）

网关通常做 ZC，创建自定义 gateway endpoint（`ezb_zha_create_custom_gateway`）。Signal handler 与普通 ZC 一致（FORMATION → STEERING，失败用 `alarm_timer_schedule` 重试）。

```c
esp_err_t esp_zigbee_create_gateway_device(void)
{
    ezb_af_device_desc_t            dev_desc = ezb_af_create_device_desc();
    ezb_zha_custom_gateway_config_t gw_cfg  = EZB_ZHA_CUSTOM_GATEWAY_CONFIG();
    ezb_af_ep_desc_t ep_desc = ezb_zha_create_custom_gateway(ESP_ZIGBEE_CUSTOM_GATEWAY_EP_ID, &gw_cfg);
    ESP_ERROR_CHECK(ezb_af_device_add_endpoint_desc(dev_desc, ep_desc));
    ESP_ERROR_CHECK(ezb_af_device_desc_register(dev_desc));
    return ESP_OK;
}
```

### 5. 构建：按主控芯片 set-target

```bash
# 主控为 ESP32-S3
idf.py set-target esp32s3
idf.py -p PORT1 erase_flash flash monitor     # 烧主控
# RCP（H2）单独烧 ot_rcp，或由网关自动升级
```

RCP 固件位于 ESP-IDF `examples/openthread/ot_rcp`；仓库 `tools/` 与 `examples/zigbee_gateway/generate_rcp_image.py` 可生成可烧录镜像。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_ZIGBEE_RADIO_MODE_NATIVE` 在 C3/S3 报错 | 主控无 15.4 radio | 改用 `ESP_ZIGBEE_RADIO_MODE_UART_RCP` |
| RCP 无响应 | UART 接线/波特率错 | `rx_pin=主控收=RCP的TX`，460800；RCP 已烧 ot_rcp |
| 共存时 Wi-Fi 不稳 | 未设 Wi-Fi 省电 | `esp_wifi_set_ps(WIFI_PS_MIN_MODEM)`，开 coex |
| C6 单芯片网关 radio 选错 | C6 有 15.4，可 NATIVE | C6 上用 `ESP_ZIGBEE_RADIO_MODE_NATIVE` 即可单芯片 |
| RCP 升级失败 | 固件路径/目标芯片错 | `firmware_dir=/rcp_fw/ot_rcp`，`target_chip` 匹配 H2/C6 |

## 参考项目

- `examples/zigbee_gateway/` — 完整网关（含 `main/zigbee_rcp.c`、`generate_rcp_image.py`）
- `examples/zigbee_gateway/sdkconfig.defaults.esp32s3`、`esp32p4` — 各主控默认配置
- ESP-IDF `examples/openthread/ot_rcp` — RCP 固件源
