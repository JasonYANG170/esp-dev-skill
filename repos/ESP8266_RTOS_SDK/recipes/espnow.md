# ESPNOW 设备间通信

> **适用摘要**: 用 ESPNOW 在 ESP8266 之间做免连接的低延迟通信，支持广播/单播、PMK/LMK 加密、对端列表管理；回调在 WiFi 任务中触发，应用通过队列转交任务处理。

> Version: ESP8266 RTOS SDK version used by the project.
> Evidence: `repos/ESP8266_RTOS_SDK/resources/`, source/examples in `repos/ESP8266_RTOS_SDK/`, and this recipe path `repos/ESP8266_RTOS_SDK/recipes/espnow.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESPNOW"
- "设备间通信"
- "免连接 wifi 通信"
- "广播/单播 espnow"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/wifi/espnow/` |
| 配置 | menuconfig 的 `Example Configuration`：`CONFIG_ESPNOW_CHANNEL`、`CONFIG_ESPNOW_PMK`、`CONFIG_ESPNOW_LMK`、`CONFIG_ESPNOW_SEND_COUNT/DELAY/LEN`（`Kconfig.projbuild`） |
| 硬件 | 至少两块 ESP8266（一发一收） |

## 分步说明

### 1. WiFi 初始化（ESPNOW 前必须先起 WiFi）

```c
#include "esp_now.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "nvs_flash.h"
#include "tcpip_adapter.h"
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/queue.h"

// 示例宏：可改为 WIFI_MODE_STA 或 WIFI_MODE_AP
#define ESPNOW_WIFI_MODE   WIFI_MODE_STA
#define ESPNOW_WIFI_IF     ESP_IF_WIFI_STA

static void example_wifi_init(void)
{
    tcpip_adapter_init();
    ESP_ERROR_CHECK(esp_event_loop_create_default());

    wifi_init_config_t cfg = WIFI_INIT_CONFIG_DEFAULT();
    ESP_ERROR_CHECK(esp_wifi_init(&cfg));
    ESP_ERROR_CHECK(esp_wifi_set_storage(WIFI_STORAGE_RAM));
    ESP_ERROR_CHECK(esp_wifi_set_mode(ESPNOW_WIFI_MODE));
    ESP_ERROR_CHECK(esp_wifi_start());
    // 统一到同一信道（实际应用两端出厂同信道可不设）
    ESP_ERROR_CHECK(esp_wifi_set_channel(CONFIG_ESPNOW_CHANNEL, 0));
}
```

### 2. ESPNOW 初始化 + 注册回调

```c
static xQueueHandle example_espnow_queue;

static void example_espnow_send_cb(const uint8_t *mac_addr, esp_now_send_status_t status)
{
    // 回调在 WiFi 任务上下文，勿做长操作，转交队列
    // ...组装事件并入队
}
static void example_espnow_recv_cb(const uint8_t *mac_addr, const uint8_t *data, int len)
{
    // ...malloc 一份 data，把 {mac,data,len} 入队，在任务里解析
}

static esp_err_t example_espnow_init(void)
{
    example_espnow_queue = xQueueCreate(32, sizeof(/* 你的事件结构 */));
    ESP_ERROR_CHECK(esp_now_init());
    ESP_ERROR_CHECK(esp_now_register_send_cb(example_espnow_send_cb));
    ESP_ERROR_CHECK(esp_now_register_recv_cb(example_espnow_recv_cb));
    ESP_ERROR_CHECK(esp_now_set_pmk((uint8_t *)CONFIG_ESPNOW_PMK));   // 主密钥

    // 添加广播对端
    uint8_t broadcast_mac[ESP_NOW_ETH_ALEN] = {0xFF,0xFF,0xFF,0xFF,0xFF,0xFF};
    esp_now_peer_info_t *peer = malloc(sizeof(esp_now_peer_info_t));
    memset(peer, 0, sizeof(*peer));
    peer->channel = CONFIG_ESPNOW_CHANNEL;
    peer->ifidx = ESPNOW_WIFI_IF;
    peer->encrypt = false;
    memcpy(peer->peer_addr, broadcast_mac, ESP_NOW_ETH_ALEN);
    ESP_ERROR_CHECK(esp_now_add_peer(peer));
    free(peer);
    return ESP_OK;
}
```

### 3. 发送与添加单播加密对端

```c
// 发送（目标 mac 在 peer 列表中）
esp_now_send(dest_mac, buffer, len);

// 收到对端广播后，若不在列表则添加加密对端
if (esp_now_is_peer_exist(recv_mac) == false) {
    esp_now_peer_info_t *peer = malloc(sizeof(esp_now_peer_info_t));
    memset(peer, 0, sizeof(*peer));
    peer->channel = CONFIG_ESPNOW_CHANNEL;
    peer->ifidx = ESPNOW_WIFI_IF;
    peer->encrypt = true;
    memcpy(peer->lmk, CONFIG_ESPNOW_LMK, ESP_NOW_KEY_LEN);   // 本地主密钥（16B）
    memcpy(peer->peer_addr, recv_mac, ESP_NOW_ETH_ALEN);
    ESP_ERROR_CHECK(esp_now_add_peer(peer));
    free(peer);
}

// 退出时
esp_now_deinit();
```

### 4. 关键 API

| API | 作用 |
|---|---|
| `esp_now_init(void)` | 初始化 ESPNOW |
| `esp_now_deinit(void)` | 反初始化 |
| `esp_now_register_send_cb(cb)` / `esp_now_register_recv_cb(cb)` | 注册收发回调 |
| `esp_now_set_pmk(const uint8_t *pmk)` | 设主密钥（16B，用于派生） |
| `esp_now_add_peer(const esp_now_peer_info_t *)` | 添加对端 |
| `esp_now_is_peer_exist(const uint8_t *mac)` | 是否已存在 |
| `esp_now_send(const uint8_t *mac, const uint8_t *data, size_t len)` | 发送 |
| `esp_now_peer_info_t` 字段：`peer_addr`、`lmk`、`channel`、`ifidx`、`encrypt` | 对端信息 |

> 回调在 **WiFi 任务** 中调用，禁止阻塞或长操作；数据应通过 `xQueueSendFromISR`/`xQueueSend` 转交应用任务（示例使用普通任务队列）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 收发都不通 | 两端信道不同 | 统一 `esp_wifi_set_channel` |
| `esp_now_send` 返回 `ESP_ERR_NOT_FOUND` | 目标 mac 不在 peer 列表 | 先 `esp_now_add_peer` |
| 加密通信失败 | PMK/LMK 两端不一致 | 两端 `CONFIG_ESPNOW_PMK/LMK` 配同样值 |
| 回调里 malloc 失败/卡顿 | 在 WiFi 任务里做重活 | 回调只入队，解析放任务 |
| `esp_now_init` 失败 | WiFi 未 start | 先 `esp_wifi_start()` 再 `esp_now_init()` |
| 广播能收单播收不到 | 单播对端未加 / encrypt 配置不一致 | 收到广播后按需 add_peer |

## 参考

- `examples/wifi/espnow/` — 官方 ESPNOW 示例（`main/espnow_example_main.c`、`espnow_example.h`、`Kconfig.projbuild`）
- `components/esp8266/include/esp_now.h` — ESPNOW 原型与 `esp_now_peer_info_t`
