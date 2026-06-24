# ESP-IDF API 速查（按模块）

> 所有签名取自 ESP-IDF 仓库 `components/*/include`。版本对应仓库 `D:/esp-skill/espressif-repos/esp-idf`。

## 系统 / 日志

```c
/* esp_log.h */
#define ESP_LOGE(tag, fmt, ...)   /* ERROR */
#define ESP_LOGW(tag, fmt, ...)   /* WARN  */
#define ESP_LOGI(tag, fmt, ...)   /* INFO  */
#define ESP_LOGD(tag, fmt, ...)   /* DEBUG（默认裁剪） */
#define ESP_LOGV(tag, fmt, ...)   /* VERBOSE */
void esp_log_level_set(const char *tag, esp_log_level_t level);

/* esp_err.h */
const char *esp_err_to_name(esp_err_t code);

/* esp_system.h */
void     esp_restart(void);   /* marked __noreturn__ */
uint32_t esp_get_free_heap_size(void);
uint32_t esp_get_minimum_free_heap_size(void);

/* esp_chip_info.h (components/esp_hw_support/include/) */
typedef struct {
    esp_chip_model_t model;        /* CHIP_ESP32 / ESP32S3 / ESP32C3 ... */
    uint8_t  revision;
    uint8_t  cores;
    uint32_t features;             /* CHIP_FEATURE_WIFI_BGN / BT / BLE / IEEE802154 / EMB_FLASH ... */
} esp_chip_info_t;
void esp_chip_info(esp_chip_info_t *out_info);
```

## FreeRTOS

```c
/* freertos/task.h */
BaseType_t xTaskCreate(TaskFunction_t pxTaskCode, const char *name,
                       uint32_t usStackDepth /* 单位: word */,
                       void *arg, UBaseType_t uxPriority, TaskHandle_t *handle);
void       vTaskDelay(const TickType_t xTicksToDelay);
void       vTaskDelete(TaskHandle_t handle);
TickType_t pdMS_TO_TICKS(uint32_t ms);

/* freertos/queue.h */
QueueHandle_t xQueueCreate(UBaseType_t uxQueueLength, UBaseType_t uxItemSize);
BaseType_t xQueueSend(QueueHandle_t, const void *item, TickType_t ticks);
BaseType_t xQueueReceive(QueueHandle_t, void *item, TickType_t ticks);
BaseType_t xQueueSendFromISR(QueueHandle_t, const void *item, BaseType_t *hpw);
void       portYIELD_FROM_ISR(BaseType_t hpw);

/* freertos/event_groups.h */
EventGroupHandle_t xEventGroupCreate(void);
EventBits_t xEventGroupWaitBits(EventGroupHandle_t, EventBits_t, BaseType_t clear, BaseType_t wait_all, TickType_t);
EventBits_t xEventGroupSetBits(EventGroupHandle_t, EventBits_t);
```

## GPIO（组件 `esp_driver_gpio`）

```c
/* driver/gpio.h */
esp_err_t gpio_config(const gpio_config_t *pGPIOConfig);
esp_err_t gpio_reset_pin(gpio_num_t gpio_num);
esp_err_t gpio_set_direction(gpio_num_t gpio_num, gpio_mode_t mode);
esp_err_t gpio_set_pull_mode(gpio_num_t gpio_num, gpio_pull_mode_t pull);
int       gpio_get_level(gpio_num_t gpio_num);
esp_err_t gpio_set_level(gpio_num_t gpio_num, uint32_t level);
esp_err_t gpio_set_intr_type(gpio_num_t gpio_num, gpio_int_type_t intr_type);
esp_err_t gpio_install_isr_service(int intr_alloc_flags);   /* 全局一次 */
esp_err_t gpio_isr_handler_add(gpio_num_t gpio_num, gpio_isr_t isr_handler, void *args);
esp_err_t gpio_isr_handler_remove(gpio_num_t gpio_num);
esp_err_t gpio_hold_en(gpio_num_t gpio_num);
esp_err_t gpio_hold_dis(gpio_num_t gpio_num);

/* gpio_mode_t: GPIO_MODE_DISABLE / INPUT / OUTPUT / INPUT_OUTPUT */
/* gpio_int_type_t: GPIO_INTR_DISABLE / POSEDGE / NEGEDGE / ANYEDGE / LOW_LEVEL / HIGH_LEVEL */

typedef struct {
    uint64_t pin_bit_mask;
    gpio_mode_t mode;
    gpio_pullup_t   pull_up_en;
    gpio_pulldown_t pull_down_en;
    gpio_int_type_t intr_type;
} gpio_config_t;
```

## UART（组件 `esp_driver_uart`）

```c
/* driver/uart.h */
esp_err_t uart_driver_install(uart_port_t uart_num, int rx_buffer_size, int tx_buffer_size,
                              int queue_size, QueueHandle_t *uart_queue, int intr_alloc_flags);
esp_err_t uart_driver_delete(uart_port_t uart_num);
esp_err_t uart_param_config(uart_port_t uart_num, const uart_config_t *uart_config);
esp_err_t uart_set_pin(uart_port_t uart_num, int tx_io_num, int rx_io_num, int rts_io_num, int cts_io_num);
int       uart_read_bytes(uart_port_t uart_num, uint8_t *data, uint32_t length, uint32_t ticks_to_wait);
int       uart_write_bytes(uart_port_t uart_num, const void *src, size_t size);
esp_err_t uart_wait_tx_done(uart_port_t uart_num, uint32_t ticks_to_wait);
esp_err_t uart_flush_input(uart_port_t uart_num);
esp_err_t uart_set_baudrate(uart_port_t uart_num, uint32_t baudrate);
esp_err_t uart_get_buffered_data_len(uart_port_t uart_num, size_t *size);

typedef struct {
    int baud_rate;
    uart_word_length_t data_bits;    /* UART_DATA_8_BITS ... */
    uart_parity_t parity;            /* UART_PARITY_DISABLE / EVEN / ODD */
    uart_stop_bits_t stop_bits;      /* UART_STOP_BITS_1 / 2 */
    uart_hw_flowcontrol_t flow_ctrl; /* UART_HW_FLOWCTRL_DISABLE / CTS_RTS */
    uart_sclk_t source_clk;          /* UART_SCLK_DEFAULT */
} uart_config_t;
/* uart_port_t: UART_NUM_0 / UART_NUM_1 / UART_NUM_2（数量随芯片） */
```

## I2C 主机（组件 `esp_driver_i2c`，新总线式 API）

```c
/* driver/i2c_master.h */
esp_err_t i2c_new_master_bus(const i2c_master_bus_config_t *bus_config,
                             i2c_master_bus_handle_t *ret_bus_handle);
esp_err_t i2c_master_bus_add_device(i2c_master_bus_handle_t bus_handle,
                             const i2c_device_config_t *dev_config,
                             i2c_master_dev_handle_t *ret_handle);
esp_err_t i2c_master_bus_rm_device(i2c_master_dev_handle_t handle);
esp_err_t i2c_master_transmit(i2c_master_dev_handle_t i2d_dev, const uint8_t *write_buffer,
                             size_t write_size, int xfer_timeout_ms);
esp_err_t i2c_master_receive(i2c_master_dev_handle_t i2d_dev, uint8_t *read_buffer,
                             size_t read_size, int xfer_timeout_ms);
esp_err_t i2c_master_transmit_receive(i2c_master_dev_handle_t i2d_dev,
                             const uint8_t *write_buffer, size_t write_size,
                             uint8_t *read_buffer, size_t read_size, int xfer_timeout_ms);
esp_err_t i2c_master_probe(i2c_master_bus_handle_t bus_handle, uint16_t address, int xfer_timeout_ms);
esp_err_t i2c_del_master_bus(i2c_master_bus_handle_t bus_handle);

/* 关键字段 */
typedef struct {
    i2c_port_t i2c_port; int sda_io_num; int scl_io_num;
    i2c_clock_source_t clk_source;  /* I2C_CLK_SRC_DEFAULT */
    int glitch_ignore_cnt;
    struct { bool enable_internal_pullup; } flags;
} i2c_master_bus_config_t;

typedef struct {
    i2c_addr_bit_len_t dev_addr_length;  /* I2C_ADDR_BIT_LEN_7 / 10 */
    uint16_t device_address;
    uint32_t scl_speed_hz;
} i2c_device_config_t;
```

## SPI 主机（组件 `esp_driver_spi`）

```c
/* driver/spi_master.h */
esp_err_t spi_bus_initialize(spi_host_device_t host, const spi_bus_config_t *bus_config,
                             spi_dma_chan_t dma_chan);   /* SPI_DMA_CH_AUTO */
esp_err_t spi_bus_add_device(spi_host_device_t host_id,
                             const spi_device_interface_config_t *dev_config,
                             spi_device_handle_t *handle);
esp_err_t spi_bus_remove_device(spi_device_handle_t handle);
esp_err_t spi_device_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);
esp_err_t spi_device_queue_trans(spi_device_handle_t handle, spi_transaction_t *trans_desc,
                                 uint32_t ticks_to_wait);
esp_err_t spi_device_get_trans_result(spi_device_handle_t handle, spi_transaction_t **trans_desc,
                                      uint32_t ticks_to_wait);
esp_err_t spi_device_polling_transmit(spi_device_handle_t handle, spi_transaction_t *trans_desc);

/* spi_transaction_t.length 单位为 bit（4 字节 => 4*8） */
/* host: SPI2_HOST / SPI3_HOST（SPI1_HOST 接主 Flash，勿用） */
```

## LEDC（组件 `esp_driver_ledc`）

```c
/* driver/ledc.h */
esp_err_t ledc_timer_config(const ledc_timer_config_t *timer_conf);
esp_err_t ledc_channel_config(const ledc_channel_config_t *ledc_conf);
esp_err_t ledc_set_duty(ledc_mode_t speed_mode, ledc_channel_t channel, uint32_t duty);
esp_err_t ledc_update_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
uint32_t  ledc_get_duty(ledc_mode_t speed_mode, ledc_channel_t channel);
esp_err_t ledc_set_freq(ledc_mode_t speed_mode, ledc_timer_t timer_num, uint32_t freq_hz);
esp_err_t ledc_set_fade_with_time(ledc_mode_t speed_mode, ledc_channel_t channel,
                                  uint32_t target_duty, int desired_fade_time_ms);
esp_err_t ledc_fade_start(ledc_mode_t speed_mode, ledc_channel_t channel, ledc_fade_mode_t fade_mode);

/* ledc_mode_t: LEDC_LOW_SPEED_MODE（全芯片） / LEDC_HIGH_SPEED_MODE（仅 esp32） */
```

## GPTimer（组件 `esp_driver_gptimer`）

```c
/* driver/gptimer.h */
esp_err_t gptimer_new_timer(const gptimer_config_t *config, gptimer_handle_t *ret_timer);
esp_err_t gptimer_set_alarm_action(gptimer_handle_t timer, const gptimer_alarm_config_t *config);
esp_err_t gptimer_register_event_callbacks(gptimer_handle_t timer,
                                           const gptimer_event_callbacks_t *cbs, void *user_data);
esp_err_t gptimer_enable(gptimer_handle_t timer);
esp_err_t gptimer_disable(gptimer_handle_t timer);
esp_err_t gptimer_start(gptimer_handle_t timer);
esp_err_t gptimer_stop(gptimer_handle_t timer);
esp_err_t gptimer_get_raw_count(gptimer_handle_t timer, uint64_t *value);
esp_err_t gptimer_set_raw_count(gptimer_handle_t timer, uint64_t value);

typedef struct {
    gptimer_clock_source_t clk_src;  /* GPTIMER_CLK_SRC_DEFAULT */
    gptimer_count_direction_t direction;  /* GPTIMER_COUNT_UP / DOWN */
    uint32_t resolution_hz;          /* 1MHz => 1 tick = 1 µs */
} gptimer_config_t;

/* 回调原型（运行在 ISR） */
typedef bool (*gptimer_alarm_cb_t)(gptimer_handle_t timer,
                                   const gptimer_alarm_event_data_t *edata, void *user_data);
```

## ADC oneshot（组件 `esp_adc`）

```c
/* esp_adc/adc_oneshot.h */
esp_err_t adc_oneshot_new_unit(const adc_oneshot_unit_init_cfg_t *init_config,
                               adc_oneshot_unit_handle_t *ret_unit);
esp_err_t adc_oneshot_config_channel(adc_oneshot_unit_handle_t handle, adc_channel_t channel,
                                     const adc_oneshot_chan_cfg_t *config);
esp_err_t adc_oneshot_read(adc_oneshot_unit_handle_t handle, adc_channel_t chan, int *out_raw);
```

## 事件循环 / Netif

```c
/* esp_event.h */
esp_err_t esp_event_loop_create_default(void);
esp_err_t esp_event_handler_instance_register(esp_event_base_t event_base, int32_t event_id,
                                              esp_event_handler_t event_handler, void *arg,
                                              esp_event_handler_instance_t *instance);
esp_err_t esp_event_handler_instance_unregister(esp_event_base_t event_base, int32_t event_id,
                                               esp_event_handler_instance_t instance);
esp_err_t esp_event_post(esp_event_base_t event_base, int32_t event_id, void *event_data,
                         size_t event_data_size, TickType_t ticks_to_wait);
/* 常用 base: WIFI_EVENT / IP_EVENT / ETH_EVENT / MESH_EVENT */

/* esp_netif.h */
esp_err_t esp_netif_init(void);
esp_netif_t* esp_netif_create_default_wifi_sta(void);
esp_netif_t* esp_netif_create_default_wifi_ap(void);
esp_err_t   esp_netif_create_default_wifi_mesh_netifs(esp_netif_t **p_netif_sta, esp_netif_t **p_netif_ap);
#define ESP_NETIF_DEFAULT_ETH()   /* 以太网默认 netif 配置 */
esp_err_t esp_netif_attach(esp_netif_t *netif, void *base);
```

## Wi-Fi

```c
/* esp_wifi.h */
esp_err_t esp_wifi_init(const wifi_init_config_t *config);   /* 用 WIFI_INIT_CONFIG_DEFAULT() */
esp_err_t esp_wifi_set_mode(wifi_mode_t mode);
        /* WIFI_MODE_NULL / STA / AP / APSTA / NAN / MESH */
esp_err_t esp_wifi_set_config(wifi_interface_t interface, wifi_config_t *conf);
        /* WIFI_IF_STA / WIFI_IF_AP */
esp_err_t esp_wifi_start(void);
esp_err_t esp_wifi_stop(void);
esp_err_t esp_wifi_connect(void);
esp_err_t esp_wifi_disconnect(void);
esp_err_t esp_wifi_scan_start(const wifi_scan_config_t *config, bool block);
esp_err_t esp_wifi_scan_get_ap_records(uint16_t *number, wifi_ap_record_t *ap_records);

/* 事件: WIFI_EVENT_STA_START, WIFI_EVENT_STA_DISCONNECTED, WIFI_EVENT_AP_STACONNECTED, ... */
/* IP 事件: IP_EVENT_STA_GOT_IP（含 ip_event_got_ip_t.ip_info） */
```

## NVS（组件 `nvs_flash`）

```c
/* nvs_flash.h */
esp_err_t nvs_flash_init(void);
esp_err_t nvs_flash_erase(void);
esp_err_t nvs_flash_deinit(void);

/* nvs.h */
esp_err_t nvs_open(const char *name, nvs_open_mode_t open_mode, nvs_handle_t *out_handle);
        /* NVS_READONLY / NVS_READWRITE */
void     nvs_close(nvs_handle_t handle);
esp_err_t nvs_commit(nvs_handle_t handle);
esp_err_t nvs_erase_key(nvs_handle_t handle, const char *key);

esp_err_t nvs_set_i8/i16/i32/i64/u8/u16/u32/u64(nvs_handle_t h, const char *key, _t val);
esp_err_t nvs_get_i32(... nvs_handle_t h, const char *key, _t *out);  /* 同族 get_* */
esp_err_t nvs_set_str(nvs_handle_t h, const char *key, const char *value);
esp_err_t nvs_get_str(nvs_handle_t h, const char *key, char *out, size_t *length);
esp_err_t nvs_set_blob(nvs_handle_t h, const char *key, const void *data, size_t length);
esp_err_t nvs_get_blob(nvs_handle_t h, const char *key, void *out, size_t *length);
/* NVS_TYPE_I8/I16/I32/I64/U8/U16/U32/U64/STR/BLOB/ANY */
```

## 分区表（组件 `spi_flash`）

```c
/* esp_partition.h */
const esp_partition_t* esp_partition_find_first(esp_partition_type_t type,
        esp_partition_subtype_t subtype, const char *label);
esp_partition_iterator_t esp_partition_find(esp_partition_type_t type,
        esp_partition_subtype_t subtype, const char *label);
const esp_partition_t* esp_partition_get(esp_partition_iterator_t it);
esp_partition_iterator_t esp_partition_next(esp_partition_iterator_t it);
void esp_partition_iterator_release(esp_partition_iterator_t it);

esp_err_t esp_partition_read(const esp_partition_t *partition, size_t src_offset,
                             void *dst, size_t size);
esp_err_t esp_partition_write(const esp_partition_t *partition, size_t dst_offset,
                              const void *src, size_t size);
esp_err_t esp_partition_erase_range(const esp_partition_t *partition,
                             size_t offset, size_t size);  /* offset/size 须 4KB 对齐 */

typedef struct {
    esp_partition_type_t type;
    esp_partition_subtype_t subtype;
    uint32_t address;
    uint32_t size;
    char label[17];
    bool encrypted;
} esp_partition_t;
/* type: ESP_PARTITION_TYPE_APP / DATA */
/* subtype app: ESP_PARTITION_SUBTYPE_APP_FACTORY / OTA_0 / OTA_1 / TEST */
/* subtype data: ESP_PARTITION_SUBTYPE_DATA_NVS / PHY / FAT / SPIFFS / OTA / COREDUMP */
```

## Flash 裸操作（组件 `spi_flash`，新 API）

```c
/* esp_flash.h（推荐用 esp_partition_* 封装，少用裸 esp_flash_*） */
esp_err_t esp_flash_get_size(esp_flash_t *chip, uint32_t *out_size);
esp_err_t esp_flash_read(esp_flash_t *chip, void *buffer, uint32_t address, uint32_t length);
esp_err_t esp_flash_write(esp_flash_t *chip, const void *buffer, uint32_t address, uint32_t length);
esp_err_t esp_flash_erase_region(esp_flash_t *chip, uint32_t start, uint32_t len);  /* 4KB 对齐 */
esp_err_t esp_flash_erase_chip(esp_flash_t *chip);
/* SPI Flash 最小擦除单位：4KB 扇区 */
```

## OTA（组件 `app_update`、`esp_https_ota`）

```c
/* esp_ota_ops.h */
const esp_partition_t* esp_ota_get_next_update_partition(const esp_partition_t *start_from);
const esp_partition_t* esp_ota_get_running_partition(void);
const esp_partition_t* esp_ota_get_boot_partition(void);
esp_err_t esp_ota_begin(const esp_partition_t *partition, size_t image_size, esp_ota_handle_t *out_handle);
        /* image_size 传 OTA_SIZE_UNKNOWN (0xffffffff) 表示未知 */
esp_err_t esp_ota_write(esp_ota_handle_t handle, const void *data, size_t size);
esp_err_t esp_ota_end(esp_ota_handle_t handle);
esp_err_t esp_ota_abort(esp_ota_handle_t handle);
esp_err_t esp_ota_set_boot_partition(const esp_partition_t *partition);

/* esp_https_ota.h */
esp_err_t esp_https_ota(const esp_https_ota_config_t *ota_config);
```

## 睡眠 / 电源（组件 `esp_hw_support`）

```c
/* esp_sleep.h */
esp_err_t esp_sleep_enable_timer_wakeup(uint64_t time_in_us);
esp_err_t esp_sleep_enable_ext0_wakeup(gpio_num_t gpio_num, int level);   /* 仅 RTC GPIO */
esp_err_t esp_sleep_enable_ext1_wakeup(uint64_t io_mask, esp_sleep_ext1_wakeup_mode_t level_mode);
esp_err_t esp_sleep_enable_gpio_wakeup_on_hp_periph_powerdown(uint64_t io_mask,
                                                              esp_sleep_gpio_wake_up_mode_t mode);
void     esp_deep_sleep_start(void);   /* 不返回 */
esp_err_t esp_light_sleep_start(void);
uint32_t esp_sleep_get_wakeup_causes(void);
        /* 位掩码: BIT(ESP_SLEEP_WAKEUP_TIMER/EXT0/EXT1/GPIO/UART/...) */
```

## 蓝牙控制器 / Host（组件 `bt`，Bluedroid）

> 启动固定顺序：NVS → `esp_bt_controller_init` → `esp_bt_controller_enable(mode)` → `esp_bluedroid_init_with_cfg` → `esp_bluedroid_enable` → 注册回调 → 注册 app。`esp_bt_mode_t`: `ESP_BT_MODE_BLE` / `ESP_BT_MODE_CLASSIC_BT` / `ESP_BT_MODE_BTDM`（双模）。Classic BT 仅 esp32 支持。

```c
/* esp_bt.h（components/bt/include/esp32/include/esp_bt.h） */
esp_err_t esp_bt_controller_mem_release(esp_bt_mode_t mode);  /* 释放不用的控制器内存 */
esp_err_t esp_bt_controller_init(esp_bt_controller_config_t *cfg);  /* BT_CONTROLLER_INIT_CONFIG_DEFAULT() */
esp_err_t esp_bt_controller_enable(esp_bt_mode_t mode);
esp_err_t esp_bt_controller_disable(void);

/* esp_bt_main.h —— Bluedroid Host */
typedef struct { bool ssp_en; bool sc_en; } esp_bluedroid_config_t;
#define BT_BLUEDROID_INIT_CONFIG_DEFAULT()   /* { .ssp_en = true, .sc_en = false } */
esp_err_t esp_bluedroid_init_with_cfg(esp_bluedroid_config_t *cfg);   /* v5.x（取代旧 init） */
esp_err_t esp_bluedroid_enable(void);
esp_err_t esp_bluedroid_disable(void);
esp_err_t esp_bluedroid_deinit(void);

/* esp_bt_device.h */
const uint8_t *esp_bt_dev_get_address(void);   /* 本机 BD 地址 */
esp_err_t esp_bt_dev_register_callback(esp_bt_dev_cb_t cb);
```

### BLE GAP（esp_gap_ble_api.h）

```c
esp_err_t esp_ble_gap_register_callback(esp_gap_ble_cb_t cb);
esp_err_t esp_ble_gap_set_device_name(const char *name);
esp_err_t esp_ble_gap_config_adv_data(esp_ble_adv_data_t *adv_data);        /* 结构化广播 */
esp_err_t esp_ble_gap_config_adv_data_raw(uint8_t *raw_data, uint32_t len); /* 原始字节广播 */
esp_err_t esp_ble_gap_config_scan_rsp_data_raw(uint8_t *raw_data, uint32_t len);
esp_err_t esp_ble_gap_start_advertising(esp_ble_adv_params_t *adv_params);
esp_err_t esp_ble_gap_stop_advertising(void);
esp_err_t esp_ble_gap_set_scan_params(esp_ble_scan_params_t *scan_params);
esp_err_t esp_ble_gap_start_scanning(uint32_t duration);    /* 秒；0=持续 */
esp_err_t esp_ble_gap_stop_scanning(void);
uint8_t  *esp_ble_resolve_adv_data_by_type(const uint8_t *adv_data, uint8_t len,
                                           esp_ble_adv_data_type type, uint8_t *out_len);
esp_err_t esp_ble_gap_update_conn_params(esp_ble_conn_update_params_t *params);
/* esp_gatt_common_api.h —— client/server 通用 */
esp_err_t esp_ble_gatt_set_local_mtu(uint16_t mtu);  /* enable 后、连接前调 */
```

### BLE GATT Server（esp_gatts_api.h）

```c
esp_err_t esp_ble_gatts_register_callback(esp_gatts_cb_t cb);
esp_err_t esp_ble_gatts_app_register(uint16_t app_id);
esp_err_t esp_ble_gatts_create_attr_tab(const esp_gatts_attr_db_t *gatts_attr_db,
        esp_gatt_if_t gatts_if, uint16_t max_nb_attr, uint8_t srvc_inst_id);  /* 属性表式（推荐） */
esp_err_t esp_ble_gatts_start_service(uint16_t service_handle);
esp_err_t esp_ble_gatts_stop_service(uint16_t service_handle);
esp_err_t esp_ble_gatts_send_indicate(esp_gatt_if_t gatts_if, uint16_t conn_id, uint16_t attr_handle,
        uint16_t value_len, uint8_t *value, bool need_confirm);   /* need_confirm=false 即 notify */
esp_err_t esp_ble_gatts_send_response(esp_gatt_if_t gatts_if, uint16_t conn_id, uint32_t trans_id,
        esp_gatt_status_t status, esp_gatt_rsp_t *rsp);
/* 事件: ESP_GATTS_REG_EVT / CREAT_ATTR_TAB_EVT / CONNECT_EVT / DISCONNECT_EVT / WRITE_EVT / READ_EVT */
/* esp_gatts_attr_db_t 项: { {ESP_GATT_AUTO_RSP}, {uuid_len, uuid_p, perm, max_len, present_len, value_p} } */
```

### BLE GATT Client（esp_gattc_api.h）

```c
esp_err_t esp_ble_gattc_register_callback(esp_gattc_cb_t cb);
esp_err_t esp_ble_gattc_app_register(uint16_t app_id);
esp_err_t esp_ble_gattc_open(esp_gatt_if_t gattc_if, esp_bd_addr_t remote_bda,
        esp_ble_addr_type_t remote_addr_type, bool is_direct);          /* 简化版 */
esp_err_t esp_ble_gattc_enh_open(esp_gatt_if_t gattc_if, esp_ble_gatt_creat_conn_params_t *p); /* v5.x 完整 */
esp_err_t esp_ble_gattc_send_mtu_req(esp_gatt_if_t gattc_if, uint16_t conn_id);
esp_err_t esp_ble_gattc_search_service(esp_gatt_if_t gattc_if, uint16_t conn_id, esp_bt_uuid_t *filter_uuid);
esp_err_t esp_ble_gattc_get_attr_count(esp_gatt_if_t, uint16_t conn_id, esp_gatt_db_element_type_t type,
        uint16_t start_handle, uint16_t end_handle, uint16_t char_handle, uint16_t *count);
esp_err_t esp_ble_gattc_get_char_by_uuid(esp_gatt_if_t, uint16_t conn_id, uint16_t start_handle,
        uint16_t end_handle, esp_bt_uuid_t char_uuid, esp_gattc_char_elem_t *result, uint16_t *count);
esp_err_t esp_ble_gattc_get_descr_by_char_handle(esp_gatt_if_t, uint16_t conn_id, uint16_t char_handle,
        esp_bt_uuid_t descr_uuid, esp_gattc_descr_elem_t *result, uint16_t *count);
esp_err_t esp_ble_gattc_read_char(esp_gatt_if_t, uint16_t conn_id, uint16_t handle, esp_gatt_auth_req_t);
esp_err_t esp_ble_gattc_write_char(esp_gatt_if_t, uint16_t conn_id, uint16_t handle, uint16_t value_len,
        uint8_t *value, esp_gatt_write_type_t write_type, esp_gatt_auth_req_t auth_req);
esp_err_t esp_ble_gattc_write_char_descr(esp_gatt_if_t, uint16_t conn_id, uint16_t handle, uint16_t value_len,
        uint8_t *value, esp_gatt_write_type_t write_type, esp_gatt_auth_req_t auth_req);
esp_err_t esp_ble_gattc_register_for_notify(esp_gatt_if_t, esp_bd_addr_t remote_bda, uint16_t handle);
esp_err_t esp_ble_gattc_close(esp_gatt_if_t gattc_if, uint16_t conn_id);
/* 事件: ESP_GATTC_REG_EVT / CONNECT_EVT / DIS_SRVC_CMPL_EVT / SEARCH_RES_EVT / SEARCH_CMPL_EVT /
 *       REG_FOR_NOTIFY_EVT / NOTIFY_EVT / WRITE_CHAR_EVT / WRITE_DESCR_EVT / DISCONNECT_EVT */
```

### 经典蓝牙 SPP（esp_spp_api.h）—— 仅 esp32

```c
typedef enum { ESP_SPP_MODE_CB = 0, ESP_SPP_MODE_VFS = 1 } esp_spp_mode_t;
typedef struct { esp_spp_mode_t mode; bool enable_l2cap_ertm; uint16_t tx_buffer_size; } esp_spp_cfg_t;
esp_err_t esp_spp_register_callback(esp_spp_cb_t cb);
esp_err_t esp_spp_enhanced_init(const esp_spp_cfg_t *cfg);   /* v5.x（取代旧 esp_spp_init） */
esp_err_t esp_spp_deinit(void);
esp_err_t esp_spp_start_srv(esp_spp_sec_t sec_mask, esp_spp_role_t role, uint8_t local_scn, const char *name);
esp_err_t esp_spp_connect(esp_spp_sec_t sec_mask, esp_spp_role_t role, uint8_t remote_scn, esp_bd_addr_t peer);
esp_err_t esp_spp_disconnect(uint32_t handle);
esp_err_t esp_spp_write(uint32_t handle, int len, uint8_t *p_data);   /* 仅 CB 模式 */
esp_err_t esp_spp_start_discovery(esp_bd_addr_t bd_addr);             /* SDP 发现 SCN */
esp_err_t esp_spp_vfs_register(void);                                 /* VFS 模式 */
/* 事件: ESP_SPP_INIT_EVT / START_EVT / SRV_OPEN_EVT / DATA_IND_EVT / WRITE_EVT / CONG_EVT / CLOSE_EVT */
/* 经典 GAP: esp_gap_bt_api.h（注意非 BLE 的 esp_gap_ble_api.h） */
esp_err_t esp_bt_gap_register_callback(esp_bt_gap_cb_t cb);
esp_err_t esp_bt_gap_set_device_name(const char *name);
esp_err_t esp_bt_gap_set_scan_mode(esp_bt_scan_mode_t mode);   /* ESP_BT_CONNECTABLE + general/limited */
```

### 经典蓝牙 A2DP / AVRCP（esp_a2dp_api.h / esp_avrc_api.h）—— 仅 esp32

```c
/* A2DP Sink（接收端） */
esp_err_t esp_a2d_register_callback(esp_a2d_cb_t cb);
esp_err_t esp_a2d_sink_init(void);
esp_err_t esp_a2d_sink_register_audio_data_callback(esp_a2d_sink_audio_data_cb_t cb);  /* PCM 数据入口 */
esp_err_t esp_a2d_sink_connect(esp_bd_addr_t remote_bda);
esp_err_t esp_a2d_sink_disconnect(esp_bd_addr_t remote_bda);
esp_err_t esp_a2d_media_ctrl(esp_a2d_media_ctrl_t ctrl);   /* ESP_A2D_MEDIA_CTRL_START / SUSPEND */
/* A2DP Source（发送端） */
esp_err_t esp_a2d_source_init(void);
esp_err_t esp_a2d_source_connect(esp_bd_addr_t remote_bda);
esp_err_t esp_a2d_source_audio_data_send(esp_a2d_conn_hdl_t conn_hdl, esp_a2d_audio_buff_t *audio_buf);
/* 事件: ESP_A2D_PROF_STATE_EVT / CONNECTION_STATE_EVT / AUDIO_STATE_EVT / AUDIO_CFG_EVT */
/* AVRCP Controller（Sink 端作 CT 控制手机） */
esp_err_t esp_avrc_ct_register_callback(esp_avrc_ct_cb_t cb);
esp_err_t esp_avrc_ct_init(void);
esp_err_t esp_avrc_ct_send_passthrough_cmd(uint8_t tl, uint8_t key_code, uint8_t key_state);
        /* key_code: ESP_AVRC_PT_CMD_PLAY / PAUSE / STOP / FORWARD / BACKWARD */
esp_err_t esp_avrc_ct_send_set_absolute_volume_cmd(uint8_t tl, uint8_t volume);  /* 0~127 */
esp_err_t esp_avrc_ct_send_metadata_cmd(uint8_t tl, uint8_t attr_mask);
```

### ESP-BLE-MESH（组件 `esp_ble_mesh`）

```c
/* esp_ble_mesh_common_api.h */
esp_err_t esp_ble_mesh_init(esp_ble_mesh_prov_t *prov, esp_ble_mesh_comp_t *comp);
/* esp_ble_mesh_provisioning_api.h */
esp_err_t esp_ble_mesh_register_prov_callback(esp_ble_mesh_prov_cb_t cb);
esp_err_t esp_ble_mesh_node_prov_enable(esp_ble_mesh_prov_bearer_t bearers);  /* ESP_BLE_MESH_PROV_ADV | _GATT */
/* esp_ble_mesh_networking_api.h */
esp_err_t esp_ble_mesh_server_model_send_msg(esp_ble_mesh_model_t *model, esp_ble_mesh_msg_ctx_t *ctx,
        uint32_t op, uint16_t length, uint8_t *data);
esp_err_t esp_ble_mesh_model_publish(esp_ble_mesh_model_t *model, uint32_t op,
        uint16_t length, uint8_t *data, esp_ble_mesh_dev_role_t role);
/* esp_ble_mesh_generic_model_api.h / config_model_api.h（注册回调） */
esp_err_t esp_ble_mesh_register_generic_server_callback(esp_ble_mesh_generic_server_cb_t cb);
esp_err_t esp_ble_mesh_register_config_server_callback(esp_ble_mesh_cfg_server_cb_t cb);
/* 模型声明宏: ESP_BLE_MESH_MODEL_CFG_SRV / ESP_BLE_MESH_MODEL_GEN_ONOFF_SRV / ESP_BLE_MESH_ELEMENT */
```

## Wi-Fi Mesh（组件 `mesh`，esp_mesh.h）

```c
esp_err_t esp_mesh_init(void);
esp_err_t esp_mesh_set_config(const mesh_cfg_t *config);
esp_err_t esp_mesh_set_topology(mesh_type_t topo);   /* MESH_TOPOLOGY_TREE / CHAIN */
esp_err_t esp_mesh_set_max_layer(int max_layer);
esp_err_t esp_mesh_set_vote_percentage(float percentage);
esp_err_t esp_mesh_start(void);
esp_err_t esp_mesh_stop(void);
esp_err_t esp_mesh_send(const mesh_addr_t *to, const mesh_data_t *data, int flag,
        const mesh_opt_t opt[], int opt_count);      /* flag: MESH_DATA_P2P / FROMDS / TODS */
esp_err_t esp_mesh_recv(mesh_addr_t *from, mesh_data_t *data, int timeout_ms,
        int *flag, mesh_opt_t opt[], int opt_count);
bool      esp_mesh_is_root(void);
int       esp_mesh_get_layer(void);
esp_err_t esp_mesh_get_routing_table(mesh_addr_t *route_table, int size, int *number);
int       esp_mesh_get_routing_table_size(void);
/* 事件 base: MESH_EVENT; 关键: STARTED / PARENT_CONNECTED / PARENT_DISCONNECTED /
 *          LAYER_CHANGE / ROOT_ADDRESS / ROUTING_TABLE_ADD / SCAN_DONE */
```

## 以太网（组件 `esp_eth`）

```c
/* esp_eth_mac_esp.h —— 内部 MAC（esp32/p4） */
esp_eth_mac_t *esp_eth_mac_new_esp32(const eth_esp32_emac_config_t *esp32_config,
                                     const eth_mac_config_t *config);
#define ETH_ESP32_EMAC_DEFAULT_CONFIG()
#define ETH_MAC_DEFAULT_CONFIG()
/* esp_eth_phy.h —— PHY */
esp_eth_phy_t *esp_eth_phy_new_generic(const eth_phy_config_t *config);   /* 推荐：通用 802.3 */
#define ETH_PHY_DEFAULT_CONFIG()
/* esp_eth_driver.h —— 驱动 */
#define ETH_DEFAULT_CONFIG(emac, ephy)     /* 组装 esp_eth_config_t */
esp_err_t esp_eth_driver_install(const esp_eth_config_t *config, esp_eth_handle_t *out_hdl);
esp_err_t esp_eth_driver_uninstall(esp_eth_handle_t hdl);
esp_err_t esp_eth_start(esp_eth_handle_t hdl);
esp_err_t esp_eth_stop(esp_eth_handle_t hdl);
esp_err_t esp_eth_ioctl(esp_eth_handle_t hdl, esp_eth_io_cmd_t cmd, void *data);
        /* ETH_CMD_G/S_MAC_ADDR, ETH_CMD_G_SPEED, ETH_CMD_G_DUPLEX_MODE,
         * ETH_CMD_S_AUTONEGO, ETH_CMD_READ/WRITE_PHY_REG */
/* esp_eth_netif_glue.h —— 挂到 netif */
esp_eth_netif_glue_handle_t esp_eth_new_netif_glue(esp_eth_handle_t eth_hdl);
esp_err_t esp_eth_del_netif_glue(esp_eth_netif_glue_handle_t eth_netif_glue);
/* 事件: ETH_EVENT base, ETHERNET_EVENT_CONNECTED / DISCONNECTED / START / STOP
 *      IP_EVENT base,  IP_EVENT_ETH_GOT_IP */
```
