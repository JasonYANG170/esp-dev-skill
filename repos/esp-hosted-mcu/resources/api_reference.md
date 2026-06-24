# ESP-Hosted-MCU Host API 快速参考

> 所有签名均来自 `esp-hosted-mcu` 仓库的真实头文件（`host/*.h`、`host/api/include/*.h`）。代码在 host 端调用；协处理器（slave）端无需手写应用代码，按 menuconfig 配置 `slave` 例程即可。Wi-Fi API 走标准 ESP-IDF 签名（weak 定义在 `host/api/src/esp_wifi_weak.c`，由 RPC 实现）。

## 1. 顶层最小 API（`host/esp_hosted.h`）

```c
typedef struct esp_hosted_transport_config esp_hosted_config_t;

int esp_hosted_init(void);
int esp_hosted_deinit(void);

int esp_hosted_connect_to_slave(void);
int esp_hosted_get_coprocessor_fwversion(esp_hosted_coprocessor_fwver_t *ver_info);
int esp_hosted_get_cp_info(uint32_t *cp_chip_id, char *cp_target_name, size_t cp_target_name_len);
```

## 2. 事件 API（`host/esp_hosted_event.h`）

```c
ESP_EVENT_DECLARE_BASE(ESP_HOSTED_EVENT);

enum {
    ESP_HOSTED_EVENT_CP_INIT = 0,
    ESP_HOSTED_EVENT_CP_HEARTBEAT,
    ESP_HOSTED_EVENT_TRANSPORT_FAILURE,
    ESP_HOSTED_EVENT_TRANSPORT_UP,
    ESP_HOSTED_EVENT_TRANSPORT_DOWN,
    ESP_HOSTED_EVENT_MEM_MONITOR,
};

typedef struct { esp_reset_reason_t reason; } esp_hosted_event_init_t;        // CP_INIT
typedef struct { uint32_t heartbeat; }       esp_hosted_event_heartbeat_t;    // HEARTBEAT
```

通过标准 `esp_event_handler_instance_register(ESP_HOSTED_EVENT, ...)` 注册。

## 3. 杂项 API（`host/esp_hosted_misc.h`）

### 协处理器 BT 控制器（v2.5.2+ 必须显式启用）
```c
esp_err_t esp_hosted_bt_controller_init(void);
esp_err_t esp_hosted_bt_controller_enable(void);
esp_err_t esp_hosted_bt_controller_disable(void);
esp_err_t esp_hosted_bt_controller_deinit(bool mem_release);
```

### MAC 地址��Wi-Fi / BT / 802.15.4 等）
```c
esp_err_t esp_hosted_iface_mac_addr_set(uint8_t *mac, size_t mac_len, esp_mac_type_t type);
esp_err_t esp_hosted_iface_mac_addr_get(uint8_t *mac, size_t mac_len, esp_mac_type_t type);
size_t    esp_hosted_iface_mac_addr_len_get(esp_mac_type_t type);   // 0=不支持 / 6=MAC-48 / 8=EUI-64
```

### 应用描述符
```c
#define ESP_HOSTED_APP_DESC_MAGIC_WORD (0xABCD5432)

typedef struct {
    uint32_t magic_word;            // ESP_HOSTED_APP_DESC_MAGIC_WORD
    uint32_t secure_version;
    uint32_t reserv1[2];
    char     version[32];
    char     project_name[32];
    char     time[16];
    char     date[16];
    char     idf_ver[32];
    uint8_t  app_elf_sha256[32];
    uint16_t min_efuse_blk_rev_full;
    uint16_t max_efuse_blk_rev_full;
    uint8_t  mmu_page_size;          // log2
    uint8_t  reserv3[3];
    uint32_t reserv2[18];
} esp_hosted_app_desc_t;

esp_err_t esp_hosted_get_coprocessor_app_desc(esp_hosted_app_desc_t *app_desc);
```

### 自定义数据（host ↔ slave 透传，v2.12.4+ 回调含 local_context）
```c
esp_err_t esp_hosted_send_custom_data(uint32_t msg_id_to_send, const uint8_t *data_to_send, size_t data_len_to_send);

esp_err_t esp_hosted_register_custom_callback(uint32_t msg_id_exp,
    void (*callback)(uint32_t msg_id_recvd, const uint8_t *data_recvd, size_t data_len_recvd, void *local_context),
    void *local_context);
```
> `msg_id` 可为任意 uint32，但 `0xFFFFFFFF` 保留。

### 心跳与内存监控
```c
esp_err_t esp_hosted_configure_heartbeat(bool enable, int duration_sec);  // 1s ~ 24h
esp_err_t esp_hosted_set_mem_monitor(esp_hosted_config_mem_monitor_t *config, esp_hosted_curr_mem_info_t *curr_mem_info);
```

## 4. 传输配置 API（`host/api/include/esp_hosted_transport_config.h`）

### 错误码
```c
typedef enum {
    ESP_TRANSPORT_OK = ESP_OK,
    ESP_TRANSPORT_ERR_INVALID_ARG = ESP_ERR_INVALID_ARG,
    ESP_TRANSPORT_ERR_ALREADY_SET = ESP_ERR_NOT_ALLOWED,
    ESP_TRANSPORT_ERR_INVALID_STATE = ESP_ERR_INVALID_STATE,
} esp_hosted_transport_err_t;
```

### 引脚
```c
typedef struct { void *port; int pin; } gpio_pin_t;
```

### 配置结构（节选关键字段）
```c
struct esp_hosted_sdio_config {
    uint32_t clock_freq_khz; uint8_t bus_width; uint8_t slot;
    gpio_pin_t pin_clk, pin_cmd, pin_d0, pin_d1, pin_d2, pin_d3, pin_reset;
    uint8_t rx_mode; bool block_mode; bool iomux_enable;
    uint16_t tx_queue_size, rx_queue_size;
};

struct esp_hosted_spi_config {              // 全双工
    gpio_pin_t pin_mosi, pin_miso, pin_sclk, pin_cs, pin_handshake, pin_data_ready, pin_reset;
    uint16_t tx_queue_size, rx_queue_size; uint8_t mode; uint32_t clk_mhz;
};

struct esp_hosted_spi_hd_config {           // 半双工
    uint8_t num_data_lines;
    gpio_pin_t pin_cs, pin_clk, pin_data_ready, pin_d0, pin_d1, pin_d2, pin_d3, pin_reset;
    uint32_t clk_mhz; uint8_t mode; uint16_t tx_queue_size, rx_queue_size;
    bool checksum_enable; uint8_t num_command_bits, num_address_bits, num_dummy_bits;
};

struct esp_hosted_uart_config {
    uint8_t port; gpio_pin_t pin_tx, pin_rx, pin_reset;
    uint8_t num_data_bits, parity, stop_bits, flow_ctrl, clk_src;
    bool checksum_enable; uint32_t baud_rate; uint16_t tx_queue_size, rx_queue_size;
};

struct esp_hosted_transport_config {
    uint8_t transport_in_use;
    union { struct esp_hosted_sdio_config sdio; struct esp_hosted_spi_hd_config spi_hd;
            struct esp_hosted_spi_config spi; struct esp_hosted_uart_config uart; } u;
};
```

### 默认配置获取宏
```c
#define INIT_DEFAULT_HOST_SDIO_CONFIG()         esp_hosted_get_default_sdio_config()
#define INIT_DEFAULT_HOST_SDIO_IOMUX_CONFIG()   esp_hosted_get_default_sdio_iomux_config()
#define INIT_DEFAULT_HOST_SPI_HD_CONFIG()       esp_hosted_get_default_spi_hd_config()
#define INIT_DEFAULT_HOST_SPI_CONFIG()          esp_hosted_get_default_spi_config()
#define INIT_DEFAULT_HOST_UART_CONFIG()         esp_hosted_get_default_uart_config()
```

### 通用 / 各传输 set/get（set 均 `warn_unused_result`）
```c
esp_err_t esp_hosted_set_default_config(void);
bool esp_hosted_is_config_valid(void);
esp_hosted_transport_err_t esp_hosted_transport_set_default_config(void);
esp_hosted_transport_err_t esp_hosted_transport_get_config(struct esp_hosted_transport_config **config);
esp_hosted_transport_err_t esp_hosted_transport_get_reset_config(gpio_pin_t *pin_config);
bool esp_hosted_transport_is_config_valid(void);

// SDIO
esp_hosted_transport_err_t esp_hosted_sdio_get_config(struct esp_hosted_sdio_config **config);
esp_hosted_transport_err_t esp_hosted_sdio_set_config(struct esp_hosted_sdio_config *config);
esp_hosted_transport_err_t esp_hosted_sdio_iomux_set_config(struct esp_hosted_sdio_config *config);
// SPI HD
esp_hosted_transport_err_t esp_hosted_spi_hd_get_config(struct esp_hosted_spi_hd_config **config);
esp_hosted_transport_err_t esp_hosted_spi_hd_set_config(struct esp_hosted_spi_hd_config *config);
esp_hosted_transport_err_t esp_hosted_spi_hd_2lines_get_config(struct esp_hosted_spi_hd_config **config);
esp_hosted_transport_err_t esp_hosted_spi_hd_2lines_set_config(struct esp_hosted_spi_hd_config *config);
// SPI FD
esp_hosted_transport_err_t esp_hosted_spi_get_config(struct esp_hosted_spi_config **config);
esp_hosted_transport_err_t esp_hosted_spi_set_config(struct esp_hosted_spi_config *config);
// UART
esp_hosted_transport_err_t esp_hosted_uart_get_config(struct esp_hosted_uart_config **config);
esp_hosted_transport_err_t esp_hosted_uart_set_config(struct esp_hosted_uart_config *config);
```

## 5. 协处理器 OTA API（`host/api/include/esp_hosted_ota.h`）

```c
enum {
    ESP_HOSTED_SLAVE_OTA_ACTIVATED,
    ESP_HOSTED_SLAVE_OTA_COMPLETED,
    ESP_HOSTED_SLAVE_OTA_NOT_REQUIRED,
    ESP_HOSTED_SLAVE_OTA_NOT_STARTED,
    ESP_HOSTED_SLAVE_OTA_IN_PROGRESS,
    ESP_HOSTED_SLAVE_OTA_FAILED,
};

esp_err_t esp_hosted_slave_ota_begin(void);
esp_err_t esp_hosted_slave_ota_write(uint8_t *ota_data, uint32_t ota_data_len);
esp_err_t esp_hosted_slave_ota_end(void);
esp_err_t esp_hosted_slave_ota_activate(void);   // 激活并重启 slave

// 已 deprecated，新实现请用上面的三段式：
esp_err_t esp_hosted_slave_ota(const char *image_url)
    __attribute__((deprecated("Use examples/host_slave_ota/ for new OTA implementations")));
```

## 6. GPIO Expander API（`host/api/include/esp_hosted_cp_gpio.h`）

```c
typedef struct {
    uint64_t pin_bit_mask; uint32_t mode;
    uint32_t pull_up_en, pull_down_en, intr_type;
} esp_hosted_cp_gpio_config_t;

#define H_CP_GPIO_MODE_DISABLE         (0)
#define H_CP_GPIO_MODE_INPUT           (H_BIT0)
#define H_CP_GPIO_MODE_OUTPUT          (H_BIT1)
#define H_CP_GPIO_MODE_OUTPUT_OD       (H_BIT1 | H_BIT2)
#define H_CP_GPIO_MODE_INPUT_OUTPUT_OD (H_BIT0 | H_BIT1 | H_BIT2)
#define H_CP_GPIO_MODE_INPUT_OUTPUT    (H_BIT0 | H_BIT1)
#define H_CP_GPIO_PULL_UP   (1)
#define H_CP_GPIO_PULL_DOWN (0)

esp_err_t esp_hosted_cp_gpio_config(const esp_hosted_cp_gpio_config_t *pGPIOConfig);
esp_err_t esp_hosted_cp_gpio_reset_pin(uint32_t gpio_num);
esp_err_t esp_hosted_cp_gpio_set_level(uint32_t gpio_num, uint32_t level);
esp_err_t esp_hosted_cp_gpio_get_level(uint32_t gpio_num, int *level);
esp_err_t esp_hosted_cp_gpio_set_direction(uint32_t gpio_num, uint32_t mode);
esp_err_t esp_hosted_cp_gpio_input_enable(uint32_t gpio_num);
esp_err_t esp_hosted_cp_gpio_set_pull_mode(uint32_t gpio_num, uint32_t pull_mode);
```

## 7. 主机省电 API（`host/api/include/esp_hosted_power_save.h`）

```c
typedef enum { HOSTED_WAKEUP_UNDEFINED=0, HOSTED_WAKEUP_NORMAL_REBOOT, HOSTED_WAKEUP_DEEP_SLEEP } esp_hosted_wakeup_reason_t;
typedef enum { HOSTED_POWER_SAVE_TYPE_NONE=0, HOSTED_POWER_SAVE_TYPE_LIGHT_SLEEP, HOSTED_POWER_SAVE_TYPE_DEEP_SLEEP } esp_hosted_power_save_type_t;

int esp_hosted_power_save_init(void);     // 通常由 esp_hosted_init() 自动调用
int esp_hosted_power_save_deinit(void);
int esp_hosted_power_save_enabled(void);
int esp_hosted_woke_from_power_save(void);
int esp_hosted_power_saving(void);
int esp_hosted_power_save_start(esp_hosted_power_save_type_t power_save_type);  // deep sleep 不返回
int esp_hosted_power_save_timer_start(uint32_t time_ms);
int esp_hosted_power_save_timer_stop(void);
```

## 8. 外部共存 API（`host/api/include/esp_hosted_cp_ext_coex.h`，受 CONFIG_ESP_HOSTED_CP_EXT_COEX 控制）

```c
#define ESP_HOSTED_EXT_COEX_WIRE_1 0
#define ESP_HOSTED_EXT_COEX_WIRE_2 1
#define ESP_HOSTED_EXT_COEX_WIRE_3 2
#define ESP_HOSTED_EXT_COEX_WIRE_4 3

typedef enum { ESP_HOSTED_EXT_COEX_LEADER_ROLE=0, ESP_HOSTED_EXT_COEX_FOLLOWER_ROLE=2, ESP_HOSTED_EXT_COEX_UNKNOWN_ROLE } esp_hosted_ext_coex_work_mode_t;

typedef struct { int32_t request; int32_t priority; int32_t grant; int32_t tx_line; } esp_hosted_ext_coex_gpio_set_t;
```

## 9. BlueDroid HCI 衔接（`host/esp_hosted_bluedroid.h`、`host/esp_hosted_bt.h`）

```c
int hci_rx_handler(uint8_t *buf, size_t buf_len);   // esp_hosted_bt.h

void     hosted_hci_bluedroid_open(void);
void     hosted_hci_bluedroid_close(void);
void     hosted_hci_bluedroid_send(uint8_t *data, uint16_t len);
bool     hosted_hci_bluedroid_check_send_available(void);
esp_err_t hosted_hci_bluedroid_register_host_callback(const esp_bluedroid_hci_driver_callbacks_t *callback);
```

## 10. OpenThread RCP（`host/api/include/esp_hosted_openthread.h`）

OpenThread host 经专用 UART 与 RCP 通信，初始化接口在该头文件中；具体用法见 `examples/host_openthread_cli/`、`examples/host_openthread_border_router/`。

## 11. Wi-Fi iTWT（经 esp_wifi_remote RPC，C5/C6 协处理器）

host 调用标准 ESP-IDF 签名（`esp_wifi_he.h`），由 RPC 转发到 slave。仅 Wi-Fi 6 协处理器，STA + HE20。

```c
// setup 配置（IDF v5.3.1+ 为 wifi_itwt_setup_config_t，以下为 wifi_twt_setup_config_t，字段一致）
esp_err_t esp_wifi_sta_itwt_setup(const wifi_twt_setup_config_t *setup_config);
esp_err_t esp_wifi_sta_itwt_teardown(uint8_t flow_id);
esp_err_t esp_wifi_sta_itwt_suspend(uint8_t flow_id_bitmap, uint32_t suspend_time_ms[]);
esp_err_t esp_wifi_sta_itwt_send_probe_req(uint32_t timeout_ms);
esp_err_t esp_wifi_sta_itwt_set_target_wake_time_offset(int64_t target_wake_time_offset_us);
// 可选全局 TWT 配置（IDF v5.3.1+）
esp_err_t esp_wifi_sta_twt_config(const wifi_twt_config_t *twt_config);
```

事件（注册于 `WIFI_EVENT`）：`WIFI_EVENT_ITWT_SETUP` / `WIFI_EVENT_ITWT_TEARDOWN` / `WIFI_EVENT_ITWT_SUSPEND` / `WIFI_EVENT_ITWT_PROBE`。

对应 RPC（`docs/implemented_rpcs.md`，自 v2.2.2）：WifiStaItwtSetup=355 / Teardown=356 / Suspend=357 / GetFlowIdStatus=358 / SendProbeReq=359 / SetTargetWakeTimeOffset=360。

## 12. Wi-Fi Easy Connect DPP（经 esp_wifi_remote RPC，v2.4.3 起）

host 调用标准 ESP-IDF `esp_dpp.h` 签名，由 RPC 转发到 slave supplicant。当前仅 Responder-Enrollee 模式。

```c
// v6.0+ 无参；v5.5+ 传 NULL；更早版本传 esp_supp_dpp_event_cb 回调
esp_err_t esp_supp_dpp_init(void *event_cb);
esp_err_t esp_supp_dpp_deinit(void);
esp_err_t esp_supp_dpp_bootstrap_gen(const char *channel_list, wifi_dpp_bootstrap_t type,
                                     const char *key, const char *info);
esp_err_t esp_supp_dpp_start_listen(void);
esp_err_t esp_supp_dpp_stop_listen(void);
```

对应 RPC（自 v2.4.3）：SuppDppInit=261 / Deinit=262 / BootstrapGen=263 / StartListen=264 / StopListen=265；异步事件 SuppDppUriReady=782 / CfgRecvd=783 / Fail=784。ESP-IDF v5.5+ 这些事件并入 `WIFI_EVENT_DPP_*`（URI_READY / CFG_RECVD / FAILED）。

## 备注：Wi-Fi API（经 esp_wifi_remote）

host 上调用的 Wi-Fi API 与原生 ESP-IDF 完全同名（`esp_wifi_init`、`esp_wifi_set_mode`、`esp_wifi_set_config`、`esp_wifi_start`、`esp_wifi_connect`、`esp_wifi_scan_*` 等）。它们的 RPC 命令对应关系见 `docs/implemented_rpcs.md`（WifiInit=278, WifiStart=280, WifiConnect=282, WifiDisconnect=283, WifiSetConfig=284, WifiScanStart=286, ...）。本文件不重复列举 ESP-IDF Wi-Fi 签名——直接参考 ESP-IDF Wi-Fi API 文档。
