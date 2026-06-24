# ESP8266_RTOS_SDK API 速查（按模块）

> 所有签名、结构体、枚举、宏均取自仓库 `components/esp8266/include/`、`components/*/include/` 的真实头文件。查不到即视为不存在。

## 系统 / 错误

头文件：`esp_err.h`、`esp_system.h`、`esp_log.h`

```c
// 错误码（返回值）
typedef int32_t esp_err_t;
#define ESP_OK          0
#define ESP_FAIL        -1
#define ESP_ERR_NO_MEM  0x101
#define ESP_ERR_INVALID_ARG  0x102
#define ESP_ERR_INVALID_STATE 0x103
#define ESP_ERR_NOT_FOUND 0x104
#define ESP_ERR_NOT_SUPPORTED 0x106

// 错误码转字符串
const char *esp_err_to_name(esp_err_t code);

// 失败即 abort 并打印 backtrace（宏）
void ESP_ERROR_CHECK(x);   // 用法：ESP_ERROR_CHECK(esp_wifi_init(&cfg));

// 复位原因与重启
esp_reset_reason_t esp_reset_reason(void);
void esp_restart(void);
uint32_t esp_get_free_heap_size(void);

// 芯片信息
typedef struct {
    uint32_t cores;
    uint32_t features;   // 位域，如 CHIP_FEATURE_EMB_FLASH
    uint16_t revision;
} esp_chip_info_t;
void esp_chip_info(esp_chip_info_t *out);
uint32_t spi_flash_get_chip_size(void);   // 头文件 esp_spi_flash.h
```

### 日志（esp_log.h）

```c
static const char *TAG = "modulename";
ESP_LOGE(TAG, fmt, ...)   // 错误
ESP_LOGW(TAG, fmt, ...)   // 警告
ESP_LOGI(TAG, fmt, ...)   // 信息
ESP_LOGD(TAG, fmt, ...)   // 调试（默认编译期可能被去掉）
ESP_LOGV(TAG, fmt, ...)   // 详细

// 日志级别枚举（menuconfig: Component config → Log output → Default log verbosity）
typedef enum { ESP_LOG_NONE=0, ESP_LOG_ERROR, ESP_LOG_WARN,
               ESP_LOG_INFO, ESP_LOG_DEBUG, ESP_LOG_VERBOSE } esp_log_level_t;

// 运行期动态设置某 TAG 级别
void esp_log_level_set(const char *tag, esp_log_level_t level);
```

## FreeRTOS（freertos/）

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/event_groups.h"
#include "freertos/semphr.h"

// 任务（ESP8266 栈单位为 byte）
BaseType_t xTaskCreate(TaskFunction_t pxTaskCode, const char *const pcName,
                       const uint32_t usStackDepth, void *const pvParameters,
                       UBaseType_t uxPriority, TaskHandle_t *const pxCreatedTask);
void vTaskDelete(TaskHandle_t xTask);
void vTaskDelay(const TickType_t xTicksToDelay);

#define portTICK_PERIOD_MS  ((TickType_t)1000 / configTICK_RATE_HZ)
#define portMAX_DELAY       (TickType_t)0xffffffffUL

// 队列
QueueHandle_t xQueueCreate(UBaseType_t uxQueueLength, UBaseType_t uxItemSize);
BaseType_t xQueueSend(QueueHandle_t xQueue, const void *pvItemToQueue, TickType_t xTicksToWait);
BaseType_t xQueueReceive(QueueHandle_t xQueue, void *pvBuffer, TickType_t xTicksToWait);
BaseType_t xQueueSendFromISR(QueueHandle_t xQueue, const void *pvItemToQueue, BaseType_t *pxHigherPriorityTaskWoken);
void xQueueReset(QueueHandle_t xQueue);

// 事件组
EventGroupHandle_t xEventGroupCreate(void);
EventBits_t xEventGroupWaitBits(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToWaitFor,
                                BaseType_t xClearOnExit, BaseType_t xWaitForAllBits, TickType_t xTicksToWait);
EventBits_t xEventGroupSetBits(EventGroupHandle_t xEventGroup, const EventBits_t uxBitsToSet);

// 信号量（动态创建宏）
SemaphoreHandle_t xSemaphoreCreateMutex(void);
SemaphoreHandle_t xSemaphoreCreateBinary(void);
void vSemaphoreDelete(SemaphoreHandle_t xSemaphore);
```

## NVS（nvs_flash.h / nvs.h）

```c
// 初始化（WiFi 之前必做）
esp_err_t nvs_flash_init(void);
esp_err_t nvs_flash_erase(void);
esp_err_t nvs_flash_deinit(void);

// 打开句柄
typedef esp_err_t (*nvs_open_mode_t);  // 实为枚举
typedef enum { NVS_READONLY, NVS_READWRITE } nvs_open_mode_t;
typedef uint32_t nvs_handle_t;          // 实际句柄类型
esp_err_t nvs_open(const char *name, nvs_open_mode_t open_mode, nvs_handle_t *out_handle);
void nvs_close(nvs_handle_t handle);

// 读写（按类型）
esp_err_t nvs_get_i8 (nvs_handle_t h, const char *key, int8_t  *out_value);
esp_err_t nvs_get_u8 (nvs_handle_t h, const char *key, uint8_t *out_value);
esp_err_t nvs_get_i16(nvs_handle_t h, const char *key, int16_t *out_value);
esp_err_t nvs_get_u16(nvs_handle_t h, const char *key, uint16_t *out_value);
esp_err_t nvs_get_i32(nvs_handle_t h, const char *key, int32_t *out_value);
esp_err_t nvs_get_u32(nvs_handle_t h, const char *key, uint32_t *out_value);
esp_err_t nvs_set_i8 (nvs_handle_t h, const char *key, int8_t  value);
esp_err_t nvs_set_u8 (nvs_handle_t h, const char *key, uint8_t value);
esp_err_t nvs_set_i32(nvs_handle_t h, const char *key, int32_t value);
// blob / 字符串
esp_err_t nvs_get_str(nvs_handle_t h, const char *key, char *out_value, size_t *length);
esp_err_t nvs_set_str(nvs_handle_t h, const char *key, const char *value);
esp_err_t nvs_get_blob(nvs_handle_t h, const char *key, void *out_value, size_t *length);
esp_err_t nvs_set_blob(nvs_handle_t h, const char *key, const void *value, size_t length);

// 提交（set 后必须）
esp_err_t nvs_commit(nvs_handle_t handle);
// 删除键
esp_err_t nvs_erase_key(nvs_handle_t h, const char *key);
```

> 典型用法见各 WiFi 示例：`nvs_flash_init()` → `nvs_open("storage", NVS_READWRITE, &h)` → `nvs_get_*`/`nvs_set_*` → `nvs_commit` → `nvs_close`。

## 事件循环（esp_event.h）

```c
typedef const char *esp_event_base_t;
ESP_EVENT_DECLARE_BASE(WIFI_EVENT);   // 来�� esp_wifi_types.h
ESP_EVENT_DECLARE_BASE(IP_EVENT);     // 来自 esp_event_base.h

esp_err_t esp_event_loop_create_default(void);
esp_err_t esp_event_handler_register(esp_event_base_t event_base, int32_t event_id,
                                     esp_event_handler_t event_handler, void *event_handler_arg);
esp_err_t esp_event_handler_unregister(esp_event_base_t event_base, int32_t event_id,
                                       esp_event_handler_t event_handler);
esp_err_t esp_event_post(esp_event_base_t event_base, int32_t event_id,
                         void *event_data, size_t event_data_size, TickType_t ticks_to_wait);

#define ESP_EVENT_ANY_ID  -1   // 注册该 base 下所有 event_id
```

### IP 事件（IP_EVENT base）

```c
typedef enum {
    IP_EVENT_STA_GOT_IP,
    IP_EVENT_STA_LOST_IP,
    IP_EVENT_AP_STAIPASSIGNED,
    // ...
} ip_event_t;

typedef struct {
    tcpip_adapter_ip_info_t ip_info;
    // ...
} ip_event_got_ip_t;
```

## WiFi（esp_wifi.h / esp_wifi_types.h）

### 初始化配置宏

```c
typedef struct {
    system_event_handler_t event_handler;
    void *osi_funcs;
    uint8_t qos_enable;
    uint8_t ampdu_rx_enable;
    uint8_t rx_ba_win;
    // ...rx/tx 缓冲配置、nvs_enable、nano_enable、wpa3_sae_enable
    uint32_t magic;   // 必须为 WIFI_INIT_CONFIG_MAGIC
} wifi_init_config_t;

#define WIFI_INIT_CONFIG_DEFAULT()  { /* 全部字段默认值，含 magic */ }
#define WIFI_INIT_CONFIG_MAGIC  0x1F2F3F4F
```

### 模式与生命周期

```c
esp_err_t esp_wifi_init(const wifi_init_config_t *config);
esp_err_t esp_wifi_deinit(void);
esp_err_t esp_wifi_set_mode(wifi_mode_t mode);
esp_err_t esp_wifi_get_mode(wifi_mode_t *mode);
esp_err_t esp_wifi_start(void);
esp_err_t esp_wifi_stop(void);
esp_err_t esp_wifi_restore(void);
esp_err_t esp_wifi_connect(void);     // Station 关联 AP
esp_err_t esp_wifi_disconnect(void);
esp_err_t esp_wifi_deauth_sta(uint16_t aid);

esp_err_t esp_wifi_set_storage(wifi_storage_t storage);   // WIFI_STORAGE_FLASH / WIFI_STORAGE_RAM
```

### 配置（union）

```c
typedef union {
    wifi_ap_config_t  ap;
    wifi_sta_config_t sta;
} wifi_config_t;

esp_err_t esp_wifi_set_config(wifi_interface_t interface, wifi_config_t *conf);
esp_err_t esp_wifi_get_config(wifi_interface_t interface, wifi_config_t *conf);

// 接口
#define WIFI_IF_STA  ESP_IF_WIFI_STA
#define WIFI_IF_AP   ESP_IF_WIFI_AP
```

### STA 配置结构体字段（节选）

```c
typedef struct {
    uint8_t ssid[32];
    uint8_t password[64];
    wifi_scan_method_t scan_method;   // WIFI_FAST_SCAN / WIFI_ALL_CHANNEL_SCAN
    bool bssid_set;
    uint8_t bssid[6];
    uint8_t channel;
    uint16_t listen_interval;
    wifi_sort_method_t sort_method;   // WIFI_CONNECT_AP_BY_SIGNAL / _BY_SECURITY
    wifi_fast_scan_threshold_t threshold;   // .rssi / .authmode
    wifi_pmf_config_t pmf_cfg;
} wifi_sta_config_t;
```

### AP 配置结构体字段

```c
typedef struct {
    uint8_t ssid[32];
    uint8_t password[64];
    uint8_t ssid_len;
    uint8_t channel;
    wifi_auth_mode_t authmode;   // SoftAP 不支持 WEP
    uint8_t ssid_hidden;
    uint8_t max_connection;      // 最大 4
    uint16_t beacon_interval;    // 100~60000ms，默认 100
} wifi_ap_config_t;
```

### 扫描

```c
typedef struct {
    uint8_t *ssid;
    uint8_t *bssid;
    uint8_t channel;
    bool show_hidden;
    wifi_scan_type_t scan_type;  // WIFI_SCAN_TYPE_ACTIVE / WIFI_SCAN_TYPE_PASSIVE
    wifi_scan_time_t scan_time;  // .active{.min,.max} 或 .passive，单位 ms，>1500 不建议
} wifi_scan_config_t;

esp_err_t esp_wifi_scan_start(const wifi_scan_config_t *config, bool block);
esp_err_t esp_wifi_scan_stop(void);
esp_err_t esp_wifi_scan_get_ap_num(uint16_t *number);
esp_err_t esp_wifi_scan_get_ap_records(uint16_t *number, wifi_ap_record_t *ap_records);
esp_err_t esp_wifi_sta_get_ap_info(wifi_ap_record_t *ap_info);
```

### 杂项

```c
esp_err_t esp_wifi_set_mac(wifi_interface_t ifx, const uint8_t mac[6]);
esp_err_t esp_wifi_get_mac(wifi_interface_t ifx, uint8_t mac[6]);
esp_err_t esp_wifi_set_channel(uint8_t primary, wifi_second_chan_t second);
esp_err_t esp_wifi_set_protocol(wifi_interface_t ifx, uint8_t protocol_bitmap);  // 1=11b 2=11g 4=11n
esp_err_t esp_wifi_set_bandwidth(wifi_interface_t ifx, wifi_bandwidth_t bw);     // WIFI_BW_HT20 / WIFI_BW_HT40
esp_err_t esp_wifi_set_country(const wifi_country_t *country);
esp_err_t esp_wifi_set_ps(wifi_ps_type_t type);   // WIFI_PS_NONE / _MIN_MODEM / _MAX_MODEM
esp_err_t esp_wifi_set_max_tx_power(int8_t power);
esp_err_t esp_wifi_ap_get_sta_list(wifi_sta_list_t *sta);
esp_err_t esp_wifi_set_event_mask(uint32_t mask);
esp_err_t esp_wifi_set_rssi_threshold(int32_t rssi);
esp_err_t esp_wifi_set_inactive_time(wifi_interface_t ifx, uint16_t sec);
esp_err_t esp_wifi_80211_tx(wifi_interface_t ifx, const void *buffer, int len, bool en_sys_seq);  // 发原始 802.11 帧
```

### Promiscuous / sniffer

```c
typedef void (*wifi_promiscuous_cb_t)(void *buf, wifi_promiscuous_pkt_type_t type);
esp_err_t esp_wifi_set_promiscuous_rx_cb(wifi_promiscuous_cb_t cb);
esp_err_t esp_wifi_set_promiscuous(bool en);
esp_err_t esp_wifi_set_promiscuous_filter(const wifi_promiscuous_filter_t *filter);  // .filter_mask 见 WIFI_PROMIS_FILTER_MASK_*

// 帧类型：WIFI_PKT_MGMT / WIFI_PKT_CTRL / WIFI_PKT_DATA / WIFI_PKT_MISC
typedef struct {
    wifi_pkt_rx_ctrl_t rx_ctrl;   // rssi / rate / channel / legacy_length / HT_length 等
    uint8_t payload[0];
} wifi_promiscuous_pkt_t;
```

### 关键枚举速查

```c
typedef enum { WIFI_MODE_NULL, WIFI_MODE_STA, WIFI_MODE_AP, WIFI_MODE_APSTA, WIFI_MODE_MAX } wifi_mode_t;
typedef enum {
    WIFI_AUTH_OPEN, WIFI_AUTH_WEP, WIFI_AUTH_WPA_PSK, WIFI_AUTH_WPA2_PSK,
    WIFI_AUTH_WPA_WPA2_PSK, WIFI_AUTH_WPA2_ENTERPRISE,
    WIFI_AUTH_WPA3_PSK, WIFI_AUTH_WPA2_WPA3_PSK, WIFI_AUTH_MAX
} wifi_auth_mode_t;

// 事件（WIFI_EVENT base，来自 esp_wifi_types.h）
typedef enum {
    WIFI_EVENT_WIFI_READY = 0, WIFI_EVENT_SCAN_DONE,
    WIFI_EVENT_STA_START, WIFI_EVENT_STA_STOP,
    WIFI_EVENT_STA_CONNECTED, WIFI_EVENT_STA_DISCONNECTED,
    WIFI_EVENT_STA_AUTHMODE_CHANGE, WIFI_EVENT_STA_BSS_RSSI_LOW,
    WIFI_EVENT_STA_WPS_ER_SUCCESS, WIFI_EVENT_STA_WPS_ER_FAILED,
    WIFI_EVENT_STA_WPS_ER_TIMEOUT, WIFI_EVENT_STA_WPS_ER_PIN,
    WIFI_EVENT_AP_START, WIFI_EVENT_AP_STOP,
    WIFI_EVENT_AP_STACONNECTED, WIFI_EVENT_AP_STADISCONNECTED,
    WIFI_EVENT_AP_PROBEREQRECVED,
} wifi_event_t;

// 断开原因（节选，来自 wifi_err_reason_t）
WIFI_REASON_NO_AP_FOUND=201, WIFI_REASON_AUTH_FAIL=202,
WIFI_REASON_HANDSHAKE_TIMEOUT=204, WIFI_REASON_BEACON_TIMEOUT=200,
WIFI_REASON_CONNECTION_FAIL=205,
```

## TCPIP Adapter（tcpip_adapter.h，旧风格）

> 与 `esp_netif` 二选一成套使用。station/softap/espnow 示例用此风格。

```c
void tcpip_adapter_init(void);

typedef enum {
    TCPIP_ADAPTER_IF_STA = 0, TCPIP_ADAPTER_IF_AP, TCPIP_ADAPTER_IF_MAX
} tcpip_adapter_if_t;

typedef struct {
    ip4_addr_t ip; ip4_addr_t netmask; ip4_addr_t gw;
} tcpip_adapter_ip_info_t;

esp_err_t tcpip_adapter_start(tcpip_adapter_if_t tcpip_if, uint8_t *mac, tcpip_adapter_ip_info_t *ip_info);
esp_err_t tcpip_adapter_stop(tcpip_adapter_if_t tcpip_if);
esp_err_t tcpip_adapter_get_ip_info(tcpip_adapter_if_t tcpip_if, tcpip_adapter_ip_info_t *ip_info);
esp_err_t tcpip_adapter_set_ip_info(tcpip_adapter_if_t tcpip_if, tcpip_adapter_ip_info_t *ip_info);
esp_err_t tcpip_adapter_dhcps_start(tcpip_adapter_if_t tcpip_if);   // AP DHCP
esp_err_t tcpip_adapter_dhcps_stop(tcpip_adapter_if_t tcpip_if);
esp_err_t tcpip_adapter_dhcpc_start(tcpip_adapter_if_t tcpip_if);   // STA DHCP
esp_err_t tcpip_adapter_dhcpc_stop(tcpip_adapter_if_t tcpip_if);
esp_err_t tcpip_adapter_set_hostname(tcpip_adapter_if_t tcpip_if, const char *hostname);
esp_err_t tcpip_adapter_get_hostname(tcpip_adapter_if_t tcpip_if, const char **hostname);
```

## esp_netif（新风格）

```c
#include "esp_netif.h"
esp_err_t esp_netif_init(void);
// 用于 http_request / simple_ota 等示例，配合 example_connect() 协议示例组件
```

## ESPNOW（esp_now.h）

```c
#define ESP_NOW_ETH_ALEN 6
#define ESP_NOW_KEY_LEN 16
#define ESP_NOW_MAX_TOTAL_PEER_NUM 20
#define ESP_NOW_MAX_ENCRYPT_PEER_NUM 6
#define ESP_NOW_MAX_DATA_LEN 250

esp_err_t esp_now_init(void);
esp_err_t esp_now_deinit(void);
esp_err_t esp_now_get_version(uint32_t *version);

// 回调
typedef void (*esp_now_recv_cb_t)(const uint8_t *mac_addr, const uint8_t *data, int data_len);
typedef void (*esp_now_send_cb_t)(const uint8_t *mac_addr, esp_now_send_status_t status);
// status: ESP_NOW_SEND_SUCCESS / ESP_NOW_SEND_FAIL
esp_err_t esp_now_register_recv_cb(esp_now_recv_cb_t cb);
esp_err_t esp_now_register_send_cb(esp_now_send_cb_t cb);

// peer 管理
typedef struct {
    uint8_t peer_addr[ESP_NOW_ETH_ALEN];
    uint8_t lmk[ESP_NOW_KEY_LEN];
    uint8_t channel;
    wifi_interface_t ifidx;
    bool encrypt;
    void *priv;
} esp_now_peer_info_t;

esp_err_t esp_now_add_peer(const esp_now_peer_info_t *peer);
esp_err_t esp_now_del_peer(const uint8_t *peer_addr);
esp_err_t esp_now_mod_peer(const esp_now_peer_info_t *peer);
esp_err_t esp_now_get_peer(const uint8_t *peer_addr, esp_now_peer_info_t *peer);
bool esp_now_is_peer_exist(const uint8_t *peer_addr);

esp_err_t esp_now_set_pmk(const uint8_t *pmk);   // 16 字节主密钥
esp_err_t esp_now_send(const uint8_t *peer_addr, const uint8_t *data, size_t len);  // peer_addr=NULL=广播
```

> 必须先 `esp_wifi_init` + `esp_wifi_start` 再 `esp_now_init`。详见 `examples/wifi/espnow`。

## SmartConfig（esp_smartconfig.h）

```c
typedef enum {
    SC_TYPE_ESPTOUCH = 0, SC_TYPE_AIRKISS, SC_TYPE_ESPTOUCH_AIRKISS, SC_TYPE_ESPTOUCH_V2
} sc_type_t;

typedef enum {
    SC_STATUS_WAIT = 0, SC_STATUS_FIND_CHANNEL, SC_STATUS_GETTING_SSID_PSWD,
    SC_STATUS_LINK, SC_STATUS_LINK_OVER
} sc_status_t;

esp_err_t esp_smartconfig_start(sc_config_t *config, ...);
esp_err_t esp_smartconfig_stop(void);
esp_err_t esp_smartconfig_get_version(...);
```

## GPIO（driver/gpio.h）

```c
#define GPIO_PIN_COUNT 17   // GPIO0~GPIO16
#define GPIO_IS_VALID_GPIO(n) ((n) < GPIO_PIN_COUNT)
#define RTC_GPIO_IS_VALID_GPIO(n) ((n) == 16)   // 只有 GPIO16 是 RTC GPIO

typedef enum { GPIO_NUM_0..GPIO_NUM_16, GPIO_NUM_MAX=17 } gpio_num_t;
typedef enum {
    GPIO_MODE_DISABLE, GPIO_MODE_INPUT, GPIO_MODE_OUTPUT, GPIO_MODE_OUTPUT_OD
} gpio_mode_t;
typedef enum {
    GPIO_INTR_DISABLE=0, GPIO_INTR_POSEDGE, GPIO_INTR_NEGEDGE,
    GPIO_INTR_ANYEDGE, GPIO_INTR_LOW_LEVEL, GPIO_INTR_HIGH_LEVEL
} gpio_int_type_t;
typedef enum { GPIO_PULLUP_ONLY, GPIO_PULLDOWN_ONLY, GPIO_FLOATING } gpio_pull_mode_t;
typedef enum { GPIO_PULLUP_DISABLE=0, GPIO_PULLUP_ENABLE=1 } gpio_pullup_t;
typedef enum { GPIO_PULLDOWN_DISABLE=0, GPIO_PULLDOWN_ENABLE=1 } gpio_pulldown_t;

// 统一配置（推荐）
typedef struct {
    uint32_t pin_bit_mask;       // 位掩码，如 (1ULL<<4)
    gpio_mode_t mode;
    gpio_pullup_t pull_up_en;
    gpio_pulldown_t pull_down_en;
    gpio_int_type_t intr_type;
} gpio_config_t;
esp_err_t gpio_config(const gpio_config_t *gpio_cfg);

esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level);
int      gpio_get_level(gpio_num_t gpio_num);
esp_err_t gpio_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
esp_err_t gpio_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);
esp_err_t gpio_set_intr_type(gpio_num_t gpio_num, gpio_int_type_t intr_type);
esp_err_t gpio_pullup_en(gpio_num_t gpio_num);
esp_err_t gpio_pulldown_en(gpio_num_t gpio_num);   // 仅对 GPIO16 有意义

// 中断服务（per-pin 模型）
esp_err_t gpio_install_isr_service(int no_use);   // 参数填 0
void      gpio_uninstall_isr_service(void);
typedef void (*gpio_isr_t)(void *);
esp_err_t gpio_isr_handler_add(gpio_num_t gpio_num, gpio_isr_t isr_handler, void *args);
esp_err_t gpio_isr_handler_remove(gpio_num_t gpio_num);
// 全局 ISR 模型（与 isr_service 互斥）
esp_err_t gpio_isr_register(void (*fn)(void *), void *arg, int no_use, gpio_isr_handle_t *handle_no_use);

// 唤醒
esp_err_t gpio_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type);   // 仅 LOW_LEVEL/HIGH_LEVEL
esp_err_t gpio_wakeup_disable(gpio_num_t gpio_num);
```

> ESP8266 普通 GPIO 只能上拉、不能下拉；只有 GPIO16 反过来（见 gpio.h 注释）。GPIO6~11 接 flash。

## UART（driver/uart.h）

```c
#define UART_FIFO_LEN 128      // 硬件 FIFO
#define UART_INTR_MASK 0x1ff

typedef enum { UART_NUM_0=0, UART_NUM_1, UART_NUM_MAX } uart_port_t;
typedef enum { UART_DATA_5_BITS..UART_DATA_8_BITS } uart_word_length_t;
typedef enum { UART_STOP_BITS_1, UART_STOP_BITS_1_5, UART_STOP_BITS_2 } uart_stop_bits_t;
typedef enum { UART_PARITY_DISABLE, UART_PARITY_EVEN, UART_PARITY_ODD } uart_parity_t;
typedef enum { UART_HW_FLOWCTRL_DISABLE, UART_HW_FLOWCTRL_RTS, UART_HW_FLOWCTRL_CTS, UART_HW_FLOWCTRL_CTS_RTS } uart_hw_flowcontrol_t;

typedef struct {
    int baud_rate;
    uart_word_length_t data_bits;
    uart_parity_t parity;
    uart_stop_bits_t stop_bits;
    uart_hw_flowcontrol_t flow_ctrl;
    uint8_t rx_flow_ctrl_thresh;
} uart_config_t;

esp_err_t uart_param_config(uart_port_t uart_num, uart_config_t *uart_conf);
esp_err_t uart_driver_install(uart_port_t uart_num, int rx_buffer_size, int tx_buffer_size,
                              int queue_size, QueueHandle_t *uart_queue, int no_use);
esp_err_t uart_driver_delete(uart_port_t uart_num);

int      uart_write_bytes(uart_port_t uart_num, const char *src, size_t size);   // 阻塞写到 tx buffer/fifo
int      uart_read_bytes(uart_port_t uart_num, uint8_t *buf, uint32_t length, TickType_t ticks_to_wait);
int      uart_tx_chars(uart_port_t uart_num, const char *buffer, uint32_t len);  // 不阻塞，仅填 FIFO
esp_err_t uart_wait_tx_done(uart_port_t uart_num, TickType_t ticks_to_wait);
esp_err_t uart_flush(uart_port_t uart_num);
esp_err_t uart_flush_input(uart_port_t uart_num);
esp_err_t uart_set_baudrate(uart_port_t uart_num, uint32_t baudrate);
esp_err_t uart_set_rx_timeout(uart_port_t uart_num, const uint8_t tout_thresh);  // TOUT 阈值
esp_err_t uart_get_buffered_data_len(uart_port_t uart_num, size_t *size);
bool     uart_is_driver_installed(uart_port_t uart_num);

// 事件队列模型
typedef enum { UART_DATA, UART_BUFFER_FULL, UART_FIFO_OVF, UART_FRAME_ERR, UART_PARITY_ERR, UART_EVENT_MAX } uart_event_type_t;
typedef struct { uart_event_type_t type; size_t size; } uart_event_t;

esp_err_t uart_isr_register(uart_port_t uart_num, void (*fn)(void *), void *arg);
esp_err_t uart_enable_swap(void);   // UART0 切到 MTCK/MTDO
esp_err_t uart_disable_swap(void);
```

> rx_buffer_size 必须 > UART_FIFO_LEN；tx_buffer_size 为 0 时 `uart_write_bytes` 阻塞直到 FIFO 推完。

## I2C（driver/i2c.h，软件主模式）

```c
typedef enum { I2C_MODE_MASTER, I2C_MODE_MAX } i2c_mode_t;   // 仅主模式
typedef enum { I2C_NUM_0=0, I2C_NUM_MAX } i2c_port_t;        // 仅一个端口
typedef enum { I2C_MASTER_WRITE=0, I2C_MASTER_READ } i2c_rw_t;
typedef enum { I2C_MASTER_ACK=0x0, I2C_MASTER_NACK=0x1, I2C_MASTER_LAST_NACK=0x2 } i2c_ack_type_t;
typedef enum { I2C_CMD_RESTART=0, I2C_CMD_WRITE, I2C_CMD_READ, I2C_CMD_STOP } i2c_opmode_t;

typedef struct {
    i2c_mode_t mode;
    gpio_num_t sda_io_num;
    gpio_pullup_t sda_pullup_en;
    gpio_num_t scl_io_num;
    gpio_pullup_t scl_pullup_en;
    uint32_t clk_stretch_tick;
} i2c_config_t;

typedef void *i2c_cmd_handle_t;

esp_err_t i2c_param_config(i2c_port_t i2c_num, const i2c_config_t *i2c_conf);
esp_err_t i2c_driver_install(i2c_port_t i2c_num, i2c_mode_t mode);
esp_err_t i2c_driver_delete(i2c_port_t i2c_num);
esp_err_t i2c_set_pin(i2c_port_t i2c_num, int sda_io_num, int scl_io_num,
                      gpio_pullup_t sda_pullup_en, gpio_pullup_t scl_pullup_en, i2c_mode_t mode);

// 命令链
i2c_cmd_handle_t i2c_cmd_link_create(void);
esp_err_t i2c_master_start(i2c_cmd_handle_t cmd_handle);
esp_err_t i2c_master_write_byte(i2c_cmd_handle_t cmd_handle, uint8_t data, bool ack_en);
esp_err_t i2c_master_write(i2c_cmd_handle_t cmd_handle, uint8_t *data, size_t data_len, bool ack_en);
esp_err_t i2c_master_read_byte(i2c_cmd_handle_t cmd_handle, uint8_t *data, i2c_ack_type_t ack);
esp_err_t i2c_master_read(i2c_cmd_handle_t cmd_handle, uint8_t *data, size_t data_len, i2c_ack_type_t ack);
esp_err_t i2c_master_stop(i2c_cmd_handle_t cmd_handle);
esp_err_t i2c_master_cmd_begin(i2c_port_t i2c_num, i2c_cmd_handle_t cmd_handle, TickType_t ticks_to_wait);
void i2c_cmd_link_delete(i2c_cmd_handle_t cmd_handle);
```

## SPI（driver/spi.h，HSPI）

```c
#define SPI_NUM_MAX 2
#define SPI_CPOL_LOW 0 / SPI_CPOL_HIGH 1
#define SPI_CPHA_LOW 0 / SPI_CPHA_HIGH 1
#define SPI_BIT_ORDER_MSB_FIRST 1 / SPI_BIT_ORDER_LSB_FIRST 0

typedef enum { CSPI_HOST=0, HSPI_HOST } spi_host_t;   // ESP8266 仅 HSPI_HOST 可用
typedef enum {
    SPI_2MHz_DIV=40, SPI_4MHz_DIV=20, SPI_5MHz_DIV=16, SPI_8MHz_DIV=10, SPI_10MHz_DIV=8,
    SPI_16MHz_DIV=5, SPI_20MHz_DIV=4, SPI_40MHz_DIV=2, SPI_80MHz_DIV=1
} spi_clk_div_t;
typedef enum { SPI_MASTER_MODE, SPI_SLAVE_MODE } spi_mode_t;

// 配置接口参数（位域 union spi_interface_config_t）与中断（spi_intr_enable_t）
esp_err_t spi_init(spi_host_t host, spi_config_t *config);
esp_err_t spi_deinit(spi_host_t host);
// 主从收发等（见头文件 hspi_logic_layer.h 与 spi.h）
```

## PWM（driver/pwm.h，软件 PWM）

```c
// 初始化：period(us) + 各通道占空比数组 + 通道数 + 引脚数组（最多 8 通道）
esp_err_t pwm_init(uint32_t period, uint32_t *duties, uint8_t channel_num, const uint32_t *pin_num);
esp_err_t pwm_deinit(void);

esp_err_t pwm_set_duty(uint8_t channel_num, uint32_t duty);     // 改后需 pwm_start
esp_err_t pwm_get_duty(uint8_t channel_num, uint32_t *duty_p);
esp_err_t pwm_set_period(uint32_t period);                       // us，>=20us
esp_err_t pwm_get_period(uint32_t *period_p);
esp_err_t pwm_set_duties(uint32_t *duties);
esp_err_t pwm_set_phase(uint8_t channel_num, float phase);       // -180~180
esp_err_t pwm_set_phases(float *phases);
esp_err_t pwm_set_period_duties(uint32_t period, uint32_t *duties);
esp_err_t pwm_set_channel_invert(uint16_t channel_mask);
esp_err_t pwm_clear_channel_invert(uint16_t channel_mask);

esp_err_t pwm_start(void);   // 任何配置修改后必须调用
esp_err_t pwm_stop(uint32_t stop_level_mask);   // mask 位决定各通道停止电平
```

> period 不要低于 20us。占空比是绝对值（不是百分比），`real_duty = duties[x] / period`。

## ADC（driver/adc.h）

```c
typedef enum { ADC_READ_TOUT_MODE=0, ADC_READ_VDD_MODE } adc_mode_t;
typedef struct {
    adc_mode_t mode;
    uint8_t clk_div;   // 采样时钟 = 80M/clk_div，范围 [8,32]
} adc_config_t;

esp_err_t adc_init(adc_config_t *config);
esp_err_t adc_deinit(void);
esp_err_t adc_read(uint16_t *data);                 // TOUT 单位 1/1023 V；VDD 单位 1mV
esp_err_t adc_read_fast(uint16_t *data, uint16_t len);   // 批量；需先关 WiFi 与中断
```

> 模式由 menuconfig `Component config → PHY → vdd33_const` 决定：255=测系统电压（TOUT 悬空），[18,36]=测外部电压。

## hw_timer（driver/hw_timer.h）

```c
#define TIMER_BASE_CLK  APB_CLK_FREQ   // 80MHz

typedef void (*hw_timer_callback_t)(void *arg);
typedef enum { TIMER_CLKDIV_1=0, TIMER_CLKDIV_16=4, TIMER_CLKDIV_256=8 } hw_timer_clkdiv_t;
typedef enum { TIMER_EDGE_INT=0, TIMER_LEVEL_INT=1 } hw_timer_intr_type_t;

esp_err_t hw_timer_init(hw_timer_callback_t callback, void *arg);
esp_err_t hw_timer_deinit(void);
esp_err_t hw_timer_set_clkdiv(hw_timer_clkdiv_t clkdiv);
esp_err_t hw_timer_set_intr_type(hw_timer_intr_type_t intr_type);
esp_err_t hw_timer_set_reload(bool reload);        // true=周期, false=单次
esp_err_t hw_timer_set_load_data(uint32_t load_data);
esp_err_t hw_timer_alarm_us(uint32_t value, bool reload);  // reload:50~0x199999, 单次:10~0x199999
esp_err_t hw_timer_disarm(void);
esp_err_t hw_timer_enable(bool en);
uint32_t  hw_timer_get_count_data(void);
```

> hw_timer 中断里不要调用任何带 FreeRTOS 阻塞或非 ISR 安全的 API。

## LEDC（driver/ledc.h）

```c
#define LEDC_APB_CLK_HZ (APB_CLK_FREQ)
#define LEDC_ERR_DUTY (0xFFFFFFFF)

typedef enum { LEDC_HIGH_SPEED_MODE=0, LEDC_LOW_SPEED_MODE, LEDC_SPEED_MODE_MAX } ledc_mode_t;
typedef enum { LEDC_TIMER_0..LEDC_TIMER_3, LEDC_TIMER_MAX } ledc_timer_t;
typedef enum { LEDC_CHANNEL_0..LEDC_CHANNEL_5, LEDC_CHANNEL_MAX } ledc_channel_t;
// 完整 API 见头文件；典型：ledc_timer_config / ledc_channel_config / ledc_set_duty / ledc_update_duty
```

> ESP8266 上更常用的是更简单的 `driver/pwm.h` 的 `pwm_*` 系列。

## Sleep / 低功耗（esp_sleep.h）

```c
typedef enum { ESP_CPU_WAIT=0, ESP_CPU_LIGHTSLEEP } esp_sleep_mode_t;
typedef enum {
    ESP_SLEEP_WAKEUP_UNDEFINED, ESP_SLEEP_WAKEUP_ALL,
    ESP_SLEEP_WAKEUP_TIMER, ESP_SLEEP_WAKEUP_GPIO   // GPIO 仅 light sleep
} esp_sleep_source_t;

void esp_deep_sleep(uint64_t time_in_us);            // 唤醒等同重启，从 app_main 重新跑
esp_err_t esp_sleep_enable_timer_wakeup(uint32_t time_in_us);
esp_err_t esp_light_sleep_start(void);               // 返回发生在唤醒后
esp_err_t esp_sleep_enable_gpio_wakeup(void);
esp_err_t esp_sleep_disable_wakeup_source(esp_sleep_source_t source);
esp_err_t esp_pm_configure(const void *config);     // 传 esp_pm_config_esp8266_t*
void esp_deep_sleep_set_rf_option(uint8_t option);  // 0/1/2/4 射频校准策略
// 已废弃（仍可用）：esp_wifi_fpm_open / esp_wifi_fpm_do_sleep / esp_wifi_fpm_do_wakeup 等
```

## SPI Flash 与分区（spi_flash.h / esp_partition.h）

```c
// 直接 flash 操作（地址为绝对 flash 地址）
size_t    spi_flash_get_chip_size(void);
esp_err_t spi_flash_erase_sector(size_t sector);
esp_err_t spi_flash_erase_range(size_t start_address, size_t size);
esp_err_t spi_flash_write(size_t dest_addr, const void *src, size_t size);
esp_err_t spi_flash_read(size_t src_addr, void *dest, size_t size);

// 分区抽象
typedef struct {
    esp_partition_type_t type;            // 0=app, 1=data
    esp_partition_subtype_t subtype;
    uint32_t address;
    uint32_t size;
    char label[17];
    bool encrypted;
} esp_partition_t;

const esp_partition_t *esp_partition_find_first(esp_partition_type_t type,
                                                esp_partition_subtype_t subtype, const char *label);
esp_err_t esp_partition_read(const esp_partition_t *partition, size_t dst_offset, void *dst, size_t size);
esp_err_t esp_partition_write(const esp_partition_t *partition, size_t dst_offset, const void *src, size_t size);
esp_err_t esp_partition_erase_range(const esp_partition_t *partition, size_t start_addr, size_t size);
```

## SPIFFS（esp_spiffs.h）

```c
typedef struct {
    const char *base_path;          // 如 "/spiffs"
    const char *partition_label;    // NULL=第一个 spiffs 分区
    size_t max_files;               // 同时可打开最大文件数
    bool format_if_mount_failed;
} esp_vfs_spiffs_conf_t;

esp_err_t esp_vfs_spiffs_register(const esp_vfs_spiffs_conf_t *conf);
esp_err_t esp_vfs_spiffs_unregister(const char *partition_label);
esp_err_t esp_spiffs_info(const char *partition_label, size_t *total, size_t *used);
esp_err_t esp_spiffs_format(const char *partition_label);
```

## OTA（esp_ota_ops.h / esp_https_ota.h）

```c
// 底层（手动 begin/write/end/set_boot_partition）
typedef uint32_t esp_ota_handle_t;
const esp_partition_t *esp_ota_get_next_update_partition(const esp_partition_t *start_from);
esp_err_t esp_ota_begin(const esp_partition_t *partition, size_t image_size, esp_ota_handle_t *out_handle);
esp_err_t esp_ota_write(esp_ota_handle_t handle, const void *data, size_t size);
esp_err_t esp_ota_end(esp_ota_handle_t handle);
esp_err_t esp_ota_set_boot_partition(const esp_partition_t *partition);
const esp_partition_t *esp_ota_get_boot_partition(void);
const esp_partition_t *esp_ota_get_running_partition(void);

// 高层（HTTPS 简易）
typedef struct {
    const char *url;
    const char *cert_pem;
    esp_http_client_handle_t http_client;   // 可选
    esp_http_client_event_handle_t event_handler;
    // ...
} esp_http_client_config_t;   // 同 esp_http_client.h
esp_err_t esp_https_ota(const esp_http_client_config_t *config);
```

> OTA 需分区表选 "Two OTA app"（含 ota_0/ota_1/otadata）。简易流程：`esp_https_ota()` 成功后 `esp_restart()`。详见 `examples/system/ota/simple_ota_example`。

## HTTP Client（esp_http_client.h，节选）

```c
typedef enum {
    HTTP_EVENT_ERROR, HTTP_EVENT_ON_CONNECTED, HTTP_EVENT_HEADER_SENT,
    HTTP_EVENT_ON_HEADER, HTTP_EVENT_ON_DATA, HTTP_EVENT_ON_FINISH, HTTP_EVENT_DISCONNECTED
} esp_http_client_event_id_t;

typedef struct {
    esp_http_client_event_id_t event_id;
    esp_http_client_handle_t client;
    void *data;
    int data_len;
    char *header_key;
    char *header_value;
} esp_http_client_event_t;

typedef struct {
    const char *url;
    const char *cert_pem;
    esp_http_client_method_t method;   // HTTP_METHOD_GET/POST/...
    esp_http_client_event_handle_t event_handler;
    // ...
} esp_http_client_config_t;

esp_http_client_handle_t esp_http_client_init(const esp_http_client_config_t *config);
esp_err_t esp_http_client_perform(esp_http_client_handle_t client);
esp_err_t esp_http_client_cleanup(esp_http_client_handle_t client);
int      esp_http_client_read(esp_http_client_handle_t client, char *buffer, int len);
int      esp_http_client_write(esp_http_client_handle_t client, const char *buffer, int len);
esp_err_t esp_http_client_set_url(esp_http_client_handle_t client, const char *url);
```

## HTTP Server（esp_http_server.h）

```c
typedef void* httpd_handle_t;
typedef enum http_method httpd_method_t;   // HTTP_GET / HTTP_POST / HTTP_PUT ...（来自 http_parser.h）

// 默认配置：task_priority=tskIDLE_PRIORITY+5, stack_size=4096, server_port=80,
//           ctrl_port=32768, max_open_sockets=7, max_uri_handlers=8, max_resp_headers=8,
//           backlog_conn=5, recv/send_wait_timeout=5
#define HTTPD_DEFAULT_CONFIG()  { /* 见 esp_http_server.h */ }

// 错误码
#define ESP_ERR_HTTPD_BASE              (0x8000)
#define ESP_ERR_HTTPD_HANDLERS_FULL     (ESP_ERR_HTTPD_BASE + 1)
#define ESP_ERR_HTTPD_HANDLER_EXISTS    (ESP_ERR_HTTPD_BASE + 2)
#define ESP_ERR_HTTPD_INVALID_REQ       (ESP_ERR_HTTPD_BASE + 3)
#define ESP_ERR_HTTPD_RESP_HDR          (ESP_ERR_HTTPD_BASE + 5)
#define ESP_ERR_HTTPD_RESP_SEND         (ESP_ERR_HTTPD_BASE + 6)
#define ESP_ERR_HTTPD_ALLOC_MEM         (ESP_ERR_HTTPD_BASE + 7)
#define ESP_ERR_HTTPD_TASK              (ESP_ERR_HTTPD_BASE + 8)

// socket 收发错误
#define HTTPD_SOCK_ERR_FAIL     -1
#define HTTPD_SOCK_ERR_INVALID  -2
#define HTTPD_SOCK_ERR_TIMEOUT  -3

// 常用响应宏
#define HTTPD_200  "200 OK"
#define HTTPD_404  "404 Not Found"
#define HTTPD_408  "408 Request Timeout"
#define HTTPD_500  "500 Internal Server Error"
#define HTTPD_TYPE_JSON  "application/json"
#define HTTPD_TYPE_TEXT  "text/html"

typedef struct httpd_config { /* task_priority, stack_size, server_port, ctrl_port,
                                 max_open_sockets, max_uri_handlers, max_resp_headers,
                                 backlog_conn, lru_purge_enable, recv_wait_timeout,
                                 send_wait_timeout, global_user_ctx,
                                 global_user_ctx_free_fn, global_transport_ctx,
                                 global_transport_ctx_free_fn, open_fn, close_fn */ } httpd_config_t;

typedef struct httpd_req {
    httpd_handle_t  handle;
    int             method;
    const char      uri[HTTPD_MAX_URI_LEN + 1];
    size_t          content_len;
    void           *aux;
    void           *user_ctx;
    void           *sess_ctx;
    httpd_free_ctx_fn_t free_ctx;
} httpd_req_t;

typedef struct httpd_uri {
    const char       *uri;
    httpd_method_t    method;
    esp_err_t (*handler)(httpd_req_t *r);
    void             *user_ctx;
} httpd_uri_t;

esp_err_t httpd_start(httpd_handle_t *handle, const httpd_config_t *config);
esp_err_t httpd_stop(httpd_handle_t handle);

esp_err_t httpd_register_uri_handler(httpd_handle_t handle, const httpd_uri_t *uri_handler);
esp_err_t httpd_unregister_uri_handler(httpd_handle_t handle, const char *uri, httpd_method_t method);
esp_err_t httpd_unregister_uri(httpd_handle_t handle, const char *uri);

// 请求处理（仅在 handler 上下文调用）
int      httpd_req_recv(httpd_req_t *r, char *buf, size_t buf_len);
size_t   httpd_req_get_hdr_value_len(httpd_req_t *r, const char *field);
esp_err_t httpd_req_get_hdr_value_str(httpd_req_t *r, const char *field, char *val, size_t val_size);
size_t   httpd_req_get_url_query_len(httpd_req_t *r);
esp_err_t httpd_req_get_url_query_str(httpd_req_t *r, char *buf, size_t buf_len);
esp_err_t httpd_query_key_value(const char *qry, const char *key, char *val, size_t val_size);
int      httpd_req_to_sockfd(httpd_req_t *r);

// 响应
esp_err_t httpd_resp_send(httpd_req_t *r, const char *buf, ssize_t buf_len);     // buf_len=-1 用 strlen
esp_err_t httpd_resp_send_chunk(httpd_req_t *r, const char *buf, ssize_t buf_len); // 结束调 (r,NULL,0)
esp_err_t httpd_resp_set_status(httpd_req_t *r, const char *status);
esp_err_t httpd_resp_set_type(httpd_req_t *r, const char *type);
esp_err_t httpd_resp_set_hdr(httpd_req_t *r, const char *field, const char *value);
esp_err_t httpd_resp_send_408(httpd_req_t *r);
```

> 详见 `recipes/http_server.md` 与 `examples/protocols/http_server/simple`。`HTTPD_MAX_URI_LEN` / `HTTPD_MAX_REQ_HDR_LEN` 来自 menuconfig。

## MQTT（mqtt_client.h，esp-mqtt）

```c
#include "mqtt_client.h"

// 事件 id
typedef enum {
    MQTT_EVENT_ERROR = 0,
    MQTT_EVENT_CONNECTED,
    MQTT_EVENT_DISCONNECTED,
    MQTT_EVENT_SUBSCRIBED,
    MQTT_EVENT_UNSUBSCRIBED,
    MQTT_EVENT_PUBLISHED,
    MQTT_EVENT_DATA,
    MQTT_EVENT_ANY,
} esp_mqtt_event_id_t;

typedef struct {
    int           event_id;       // 上面枚举
    esp_mqtt_client_handle_t client;
    char         *data;
    int           data_len;
    char         *topic;
    int           topic_len;
    int           msg_id;
    void         *user_context;
} esp_mqtt_event_t;
typedef esp_mqtt_event_t *esp_mqtt_event_handle_t;

// 配置（关键字段；完整结构见 mqtt_client.h）
typedef struct {
    const char *uri;                // mqtt:// mqtts:// ws:// wss://
    const char *host;               // 可与 port 合用替代 uri
    uint32_t    port;
    const char *client_id;
    const char *username;
    const char *password;
    const char *cert_pem;           // CA（mqtts/wss）
    const char *client_cert_pem;    // 双向认证客户端证书
    const char *client_key_pem;     // 双向认证客户端私钥
    const psk_hint_key_t *psk_hint_key;   // PSK（见 esp_tls.h）
    esp_mqtt_event_handle_t event_handle; // 老式回调（与 register_event 二选一）
    int          task_stack;        // 默认 CONFIG_MQTT_TASK_STACK_SIZE=6144
    int          buffer_size;       // 默认 CONFIG_MQTT_BUFFER_SIZE=1024
} esp_mqtt_client_config_t;

esp_mqtt_client_handle_t esp_mqtt_client_init(const esp_mqtt_client_config_t *config);
esp_err_t esp_mqtt_client_register_event(esp_mqtt_client_handle_t client, esp_mqtt_event_id_t event,
                                         esp_event_handler_t event_handler, void *event_handler_arg);
esp_err_t esp_mqtt_client_start(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_stop(esp_mqtt_client_handle_t client);
esp_err_t esp_mqtt_client_destroy(esp_mqtt_client_handle_t client);
int      esp_mqtt_client_publish(esp_mqtt_client_handle_t client, const char *topic,
                                 const char *data, int len, int qos, int retain);
int      esp_mqtt_client_subscribe(esp_mqtt_client_handle_t client, const char *topic, int qos);
int      esp_mqtt_client_unsubscribe(esp_mqtt_client_handle_t client, const char *topic);
```

> PSK：`psk_hint_key_t { const uint8_t *key; size_t key_size; const char *hint; }`（esp_tls.h）。
> 传输由 `uri` scheme 决定：`mqtt://`=TCP、`mqtts://`=SSL、`ws://`=WebSocket、`wss://`=WSS。SSL/WS 需 `CONFIG_MQTT_TRANSPORT_SSL=y`、WS 需 `CONFIG_MQTT_TRANSPORT_WEBSOCKET=y`（默认开，见 `components/mqtt/Kconfig`）。详见 `recipes/mqtt.md`。

## SNTP（lwip/apps/sntp.h）

```c
#include "lwip/apps/sntp.h"
#include <time.h>

// 操作模式
#define SNTP_OPMODE_POLL    0    // 轮询（客户端常用）

void sntp_setoperatingmode(u8_t operating_mode);
void sntp_setservername(u8_t idx, const char *server);   // 如 "pool.ntp.org"
void sntp_setserver(u8_t idx, const ip_addr_t *server);
void sntp_init(void);
void sntp_stop(void);

// POSIX 时间（sntp 收到响应后内部调 settimeofday）
time_t time(time_t *tloc);
struct tm *localtime_r(const time_t *timep, struct tm *result);
int gettimeofday(struct timeval *tv, struct timezone *tz);
int settimeofday(const struct timeval *tv, const struct timezone *tz);

// 时区（POSIX）
int setenv(const char *name, const char *value, int overwrite);   // setenv("TZ","CST-8",1)
void tzset(void);
```

> 详见 `recipes/sntp_time.md` 与 `examples/protocols/sntp`。

## Socket（POSIX lwIP）

```c
#include <netdb.h>
#include <sys/socket.h>

int getaddrinfo(const char *node, const char *service, const struct addrinfo *hints, struct addrinfo **res);
void freeaddrinfo(struct addrinfo *res);
int socket(int domain, int type, int protocol);          // AF_INET, SOCK_STREAM
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
ssize_t read(int fd, void *buf, size_t count);
ssize_t write(int fd, const void *buf, size_t count);
int setsockopt(int sockfd, int level, int optname, const void *optval, socklen_t optlen);  // 如 SO_RCVTIMEO
int close(int fd);
char *inet_ntoa(struct in_addr in);
```

> 详见 `examples/protocols/http_request`（直接 socket）与 `examples/protocols/sockets/*`。
