# Wi-Fi Easy Connect (DPP) 入网设备端（Enrollee）

> **适用摘要**: 在 host 上以 DPP Responder-Enrollee 模式入网——host 生成并显示 QR 码，由支持 Wi-Fi Easy Connect（Initiator）的设备（如 Android 10+）扫码后把 SSID/密码安全下发，无需在 host 上预置明文密码。可作为无 UI 产品的 WPA-PSK 替代入网方式。

## 触发意图

- "ESP-Hosted DPP enrollee"
- "Wi-Fi Easy Connect"
- "esp_supp_dpp_bootstrap_gen"
- "QR 码扫码入网"
- "无密码入网设备端"
- "DPP Responder-Enrollee"

## 前置条件

| 条件 | 要求 |
|---|---|
| Host | 例程验证于 ESP32-P4（需显示 QR 码的控制台） |
| 协处理器 | ESP32 / C 系列 / S 系列（README 支持矩阵） |
| 依赖 | host 工程含 `espressif/esp_wifi_remote` + `esp_hosted`；`esp_dpp.h`、`qrcode.h` 由 ESP-IDF 提供 |
| 发起端 | 一台支持 Wi-Fi Easy Connect Initiator 的设备（部分 Android 10+，厂商相关） |
| 头文件 | `#include "esp_dpp.h"`、`#include "qrcode.h"` |
| 参考例程 | `examples/host_wifi_easy_connect_dpp_enrollee/` |

> DPP 当前只支持 **Responder-Enrollee** 模式，AKM 类型 PSK 与 DPP。

## 分步说明

### 1. DPP API（5 个，均经 RPC 转发到 slave 的 supplicant，v2.4.3 起）

| Host API | 对应 RPC | 自 |
|---|---|---|
| `esp_supp_dpp_init(...)` | SuppDppInit (261) | 2.4.3 |
| `esp_supp_dpp_deinit()` | SuppDppDeinit (262) | 2.4.3 |
| `esp_supp_dpp_bootstrap_gen(...)` | SuppDppBootstrapGen (263) | 2.4.3 |
| `esp_supp_dpp_start_listen()` | SuppDppStartListen (264) | 2.4.3 |
| `esp_supp_dpp_stop_listen()` | SuppDppStopListen (265) | 2.4.3 |

### 2. 事件分发：版本相关（重要）

ESP-IDF **v5.5.0+** 把 DPP 事件并入 **Wi-Fi 事件**；**v6.0+** 移除了旧的 Supplicant 回调。例程用宏自动切换：

```c
#if ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5, 5, 0)
#define EXAMPLE_DPP_USE_WIFI_EVENTS 1   // 走 WIFI_EVENT_DPP_*
#else
#define EXAMPLE_DPP_USE_WIFI_EVENTS 0   // 走 esp_supp_dpp_event_cb 回调
#endif
```

### 3. 初始化与 bootstrap（节选自 `dpp_enrollee_main.c`）

```c
esp_err_t dpp_enrollee_bootstrap(void)
{
    /* 可选：自定义 bootstrapping 私钥（NIST P-256，需 64 位十六进制 = 32 字节） */
    char *key = NULL;   // 为 NULL 时使用随机密钥
    /* ... 例程会把 64-hex 私钥包成 ASN.1 prefix/postfix 形式 ... */

    /* 当前 bootstrap 方法仅 QR Code */
    return esp_supp_dpp_bootstrap_gen(
        EXAMPLE_DPP_LISTEN_CHANNEL_LIST,   // 默认 "6"，可由 CONFIG_ESP_DPP_LISTEN_CHANNEL_LIST 配置
        DPP_BOOTSTRAP_QR_CODE,
        key,
        EXAMPLE_DPP_DEVICE_INFO);          // 可选设备信息，默认 0
}

void dpp_enrollee_init(void)
{
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    esp_netif_create_default_wifi_sta();

    ESP_ERROR_CHECK(esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL));
    ESP_ERROR_CHECK(esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, &event_handler, NULL));

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    ESP_ERROR_CHECK(esp_wifi_set_mode(WIFI_MODE_STA));

#if EXAMPLE_DPP_USE_WIFI_EVENTS
    #if ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(6, 0, 0)
        ESP_ERROR_CHECK(esp_supp_dpp_init());          // v6.0+ 无参
    #else
        ESP_ERROR_CHECK(esp_supp_dpp_init(NULL));
    #endif
#else
    ESP_ERROR_CHECK(esp_supp_dpp_init(dpp_enrollee_event_cb));  // 旧式回调
#endif
    ESP_ERROR_CHECK(dpp_enrollee_bootstrap());
    ESP_ERROR_CHECK(esp_wifi_start());   // STA_START 后会自动 start_listen
}
```

### 4. 事件处理（v5.5+ 走 Wi-Fi 事件）

```c
static void event_handler(void *arg, esp_event_base_t base, int32_t id, void *data)
{
    if (base == WIFI_EVENT) {
        switch (id) {
        case WIFI_EVENT_STA_START:
            esp_supp_dpp_start_listen();      // 开始监听鉴权
            break;
        case WIFI_EVENT_STA_DISCONNECTED:
            if (s_retry_num < WIFI_MAX_RETRY_NUM) { esp_wifi_connect(); s_retry_num++; }
            break;
#if EXAMPLE_DPP_USE_WIFI_EVENTS
        case WIFI_EVENT_DPP_URI_READY: {      // 生成 URI，打印 QR 码
            wifi_event_dpp_uri_ready_t *uri_data = data;
            esp_qrcode_config_t cfg = ESP_QRCODE_CONFIG_DEFAULT();
            vTaskDelay(500 / portTICK_PERIOD_MS);   // 避免被后台日志打断
            esp_qrcode_generate(&cfg, (const char *)uri_data->uri);
            break;
        }
        case WIFI_EVENT_DPP_CFG_RECVD: {       // 收到配置，连 AP
            wifi_event_dpp_config_received_t *config = data;
            memcpy(&s_dpp_wifi_config, &config->wifi_cfg, sizeof(s_dpp_wifi_config));
            esp_wifi_set_config(WIFI_IF_STA, &s_dpp_wifi_config);
            esp_wifi_connect();
            break;
        }
        case WIFI_EVENT_DPP_FAILED: {          // 鉴权失败，重试
            wifi_event_dpp_failed_t *f = data;
            if (s_retry_num < 5) {
                esp_supp_dpp_start_listen();
                s_retry_num++;
            }
            break;
        }
#endif
        default: break;
        }
    } else if (base == IP_EVENT && id == IP_EVENT_STA_GOT_IP) {
        /* 获取 IP，置 DPP_CONNECTED_BIT */
    }
}
```

### 5. 使用流程（终端 QR 码 + 手机扫码）

1. 编译并烧录例程，控制台打印 QR 码（dark on white/light background 才有效）。
2. 手机先连上目标 AP（如 "Example-AP"）。
3. 手机：`Settings → Wi-Fi → Example-AP → 高级 → 添加设备`。
4. 扫控制台上的 QR 码，host 收到 `WIFI_EVENT_DPP_CFG_RECVD` 后自动连 AP。

> 若 QR 码出现行间隙，换字体或换终端程序再试。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 扫码后无反应 | 手机不支持 Easy Connect Initiator | 换 Android 10+（厂商相关）或其它发起端 |
| QR 码无法识别 | 行间隙 / 非深色前景 | 换终端字体；确保 dark on white/light |
| `esp_supp_dpp_init` 签名不匹配 | 跨 IDF 版本（v5.5/v6.0）行为变化 | 按上面版本宏选择调用形式 |
| 鉴权反复失败 | 发起端与 enrollee 不在同一信道/噪声 | 检查 listen channel；最多重试 5 次后置 AUTH_FAIL_BIT |
| 旧代码用 `dpp_enrollee_event_cb` 在 v6.0 编译失败 | v6.0 移除 Supplicant DPP 事件 | 切到 `WIFI_EVENT_DPP_*` |
| bootstrapping key 报长度错 | 私钥需 64 hex（32 字节） | 校验 `CURVE_SEC256R1_PKEY_HEX_DIGITS` |
| 自定义私钥不生效 | 未正确拼 ASN.1 prefix/postfix | 参考例程 `dpp_enrollee_bootstrap()` 的拼接逻辑 |

## 参考项目

- `examples/host_wifi_easy_connect_dpp_enrollee/` — DPP Enrollee 完整例程（ESP32-P4）
- `examples/host_wifi_easy_connect_dpp_enrollee/main/dpp_enrollee_main.c` — init/bootstrap/事件处理
- `docs/features.md`（Wi-Fi Easy Connect 段，列出 5 个 DPP API）
- `docs/implemented_rpcs.md`（SuppDpp* RPC，v2.4.3 起）
