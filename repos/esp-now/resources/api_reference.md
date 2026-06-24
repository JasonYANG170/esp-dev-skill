# ESP-NOW 组件 API 速查

> 全部签名取自 `espressif/esp-now` 仓库 `src/*/include/*.h`。函数名/结构体/宏必须在此或头文件中查到，否则视为不存在。

## 通用宏与类型（espnow.h）

```c
#define ESPNOW_PACKED_STRUCT        __attribute__((packed))
#define ESPNOW_PAYLOAD_LEN          230
#define ESPNOW_ADDR_LEN             6
#define ESPNOW_CHANNEL_CURRENT      0x0
#define ESPNOW_CHANNEL_ALL          0x0f
#define ESPNOW_RETRANSMIT_MAX_COUNT 0x1f
#define ESPNOW_FORWARD_MAX_COUNT    0x1f

// ESPNOW_DATA_LEN 在开启 CONFIG_ESPNOW_APP_SECURITY 时为 ESPNOW_SEC_PACKET_MAX_SIZE，否则为 ESPNOW_PAYLOAD_LEN
#define ESPNOW_DATA_LEN   (CONFIG_ESPNOW_APP_SECURITY ? ESPNOW_SEC_PACKET_MAX_SIZE : ESPNOW_PAYLOAD_LEN)

typedef uint8_t espnow_addr_t[6];
typedef uint8_t espnow_group_t[6];

// 地址判定宏
ESPNOW_ADDR_IS_EMPTY(addr)
ESPNOW_ADDR_IS_BROADCAST(addr)
ESPNOW_ADDR_IS_SELF(addr)
ESPNOW_ADDR_IS_EQUAL(addr1, addr2)

// 预定义地址
extern const uint8_t ESPNOW_ADDR_NONE[6];
extern const uint8_t ESPNOW_ADDR_BROADCAST[6];
extern const uint8_t ESPNOW_ADDR_GROUP_OTA[6];
extern const uint8_t ESPNOW_ADDR_GROUP_SEC[6];
extern const uint8_t ESPNOW_ADDR_GROUP_PROV[6];
```

## 核心收发（espnow.h）

```c
// 配置结构
typedef struct {
    const uint8_t pmk[16];            // 主密钥
    bool forward_enable        : 1;
    bool forward_switch_channel: 1;
    bool sec_enable            : 1;   // 是否加密用户数据
    uint8_t reserved1          : 5;
    uint8_t qsize;                    // 包队列大小(默认 32)
    uint8_t send_retry_num;           // 重传次数(默认 10)
    uint32_t send_max_timeout;        // 最大发送超时(默认 3000 ticks)
    struct {
        bool ack, forward, group, provisioning, control_bind, control_data;
        bool ota_status, ota_data, debug_log, debug_command, data;
        bool sec_status, sec, sec_data, timesync;
        uint32_t reserved2 : 17;
    } receive_enable;
} espnow_config_t;

#define ESPNOW_INIT_CONFIG_DEFAULT()  /* 见头文件，pmk="ESP_NOW" 等 */

// 帧头
typedef struct espnow_frame_head_s {
    uint16_t magic;
    uint8_t channel              : 4;   // ESPNOW_CHANNEL_CURRENT / _ALL
    bool filter_adjacent_channel : 1;
    bool filter_weak_signal      : 1;
    bool security                : 1;   // true=加密
    uint16_t                     : 4;
    bool broadcast               : 1;
    bool group                   : 1;
    bool ack                     : 1;
    uint16_t retransmit_count    : 5;   // ≤ 0x1f
    uint8_t forward_ttl          : 5;   // ≤ 0x1f
    int8_t forward_rssi          : 8;
} ESPNOW_PACKED_STRUCT espnow_frame_head_t;

#define ESPNOW_FRAME_CONFIG_DEFAULT() { .broadcast = true, .retransmit_count = 10 }

// 数据类型枚举 espnow_data_type_t: ACK, FORWARD, GROUP, PROV, CONTROL_BIND, CONTROL_DATA,
//   OTA_STATUS, OTA_DATA, DEBUG_LOG, DEBUG_COMMAND, DATA, SECURITY_STATUS, SECURITY,
//   SECURITY_DATA, TIMESYNC, RESERVED

esp_err_t espnow_init(const espnow_config_t *config);
esp_err_t espnow_deinit(void);
esp_err_t espnow_send(espnow_data_type_t type, const espnow_addr_t dest_addr,
                      const void *data, size_t size,
                      const espnow_frame_head_t *frame_config, TickType_t wait_ticks);

typedef esp_err_t (*handler_for_data_t)(uint8_t *src_addr, void *data,
                                        size_t size, wifi_pkt_rx_ctrl_t *rx_ctrl);
esp_err_t espnow_set_config_for_data_type(espnow_data_type_t type, bool enable, handler_for_data_t handle);
esp_err_t espnow_get_config_for_data_type(espnow_data_type_t type, bool *enable);
```

## Peer 与分组（espnow.h）

```c
esp_err_t espnow_add_peer(const espnow_addr_t addr, const uint8_t *lmk);  // lmk 可 NULL
esp_err_t espnow_del_peer(const espnow_addr_t addr);

esp_err_t espnow_add_group(const espnow_group_t group_id);
esp_err_t espnow_del_group(const espnow_group_t group_id);
int       espnow_get_group_num(void);
esp_err_t espnow_get_group_list(espnow_group_t *group_id_list, size_t num);
bool      espnow_is_my_group(const espnow_group_t group_id);

// 动态分组下发
esp_err_t espnow_set_group(const espnow_addr_t *addrs_list, size_t addrs_num,
                           const espnow_group_t group_id,
                           espnow_frame_head_t *frame_head, bool enable, TickType_t wait_ticks);
```

## 密钥管理（espnow.h，需 sec_enable=1）

```c
esp_err_t espnow_set_key(uint8_t key_info[APP_KEY_LEN]);
esp_err_t espnow_get_key(uint8_t key_info[APP_KEY_LEN]);
esp_err_t espnow_erase_key(void);
esp_err_t espnow_set_dec_key(uint8_t key_info[APP_KEY_LEN]);   // 接收解密
esp_err_t espnow_get_dec_key(uint8_t key_info[APP_KEY_LEN]);
esp_err_t espnow_erase_dec_key(void);
```

## 设备控制（espnow_ctrl.h）

```c
// 事件 ID
#define ESP_EVENT_ESPNOW_CTRL_BIND        (ESP_EVENT_ESPNOW_CTRL_BASE + 0)
#define ESP_EVENT_ESPNOW_CTRL_UNBIND      (ESP_EVENT_ESPNOW_CTRL_BASE + 1)
#define ESP_EVENT_ESPNOW_CTRL_BIND_ERROR  (ESP_EVENT_ESPNOW_CTRL_BASE + 2)
#define ESPNOW_BIND_LIST_MAX_SIZE  32

typedef enum { /* ESPNOW_ATTRIBUTE_POWER, _BRIGHTNESS, _HUE, _KEY_1..10, _BATTERY_LEVEL ... */ } espnow_attribute_t;
typedef enum { ESPNOW_BIND_ERROR_NONE, _TIMEOUT, _RSSI, _LIST_FULL } espnow_ctrl_bind_error_t;

typedef struct espnow_ctrl_bind_info_s { uint8_t mac[6]; espnow_attribute_t initiator_attribute; } espnow_ctrl_bind_info_t;
typedef struct espnow_ctrl_data_s { espnow_attribute_t initiator_attribute; espnow_attribute_t responder_attribute;
    union { bool responder_value_b; int responder_value_i; float responder_value_f; struct{...}; };
    char responder_value_s[0]; } espnow_ctrl_data_t;

typedef bool  (*espnow_ctrl_bind_cb_t)(espnow_attribute_t initiator_attribute, uint8_t mac[6], int8_t rssi);
typedef void  (*espnow_ctrl_data_cb_t)(espnow_attribute_t initiator_attribute, espnow_attribute_t responder_attribute, uint32_t responder_value);
typedef void  (*espnow_ctrl_data_raw_cb_t)(espnow_addr_t src_addr, espnow_ctrl_data_t *data, wifi_pkt_rx_ctrl_t *rx_ctrl);

// Initiator
esp_err_t espnow_ctrl_initiator_bind(espnow_attribute_t initiator_attribute, bool enable);
esp_err_t espnow_ctrl_initiator_send(espnow_attribute_t initiator_attribute, espnow_attribute_t responder_attribute, uint32_t responder_value);

// Responder
esp_err_t espnow_ctrl_responder_bind(uint32_t wait_ms, int8_t rssi, espnow_ctrl_bind_cb_t cb);
esp_err_t espnow_ctrl_responder_data(espnow_ctrl_data_cb_t cb);
esp_err_t espnow_ctrl_responder_get_bindlist(espnow_ctrl_bind_info_t *list, size_t *size);
esp_err_t espnow_ctrl_responder_set_bindlist(const espnow_ctrl_bind_info_t *info);
esp_err_t espnow_ctrl_responder_remove_bindlist(const espnow_ctrl_bind_info_t *info);
esp_err_t espnow_ctrl_responder_clear_bindlist(void);

// 底层收发
esp_err_t espnow_ctrl_send(const espnow_addr_t dest_addr, const espnow_ctrl_data_t *data, const espnow_frame_head_t *frame_head, TickType_t wait_ticks);
esp_err_t espnow_ctrl_recv(espnow_ctrl_data_raw_cb_t cb);
```

## 安全（espnow_security.h / espnow_security_handshake.h）

```c
#define APP_KEY_LEN 32
#define KEY_LEN     16
#define IV_LEN      8
#define TAG_LEN     4
#define ESPNOW_SEC_PACKET_MAX_SIZE  (ESPNOW_PAYLOAD_LEN - TAG_LEN - IV_LEN)
#define ESP_EVENT_ESPNOW_SEC_OK   0x600
#define ESP_EVENT_ESPNOW_SEC_FAIL 0x601

typedef struct espnow_sec_s { int state; uint8_t key[KEY_LEN]; uint8_t iv[IV_LEN]; uint8_t key_len, iv_len, tag_len; void *cipher_ctx; } espnow_sec_t;

// 底层加解密
esp_err_t espnow_sec_init(espnow_sec_t *sec);
esp_err_t espnow_sec_deinit(espnow_sec_t *sec);
esp_err_t espnow_sec_setkey(espnow_sec_t *sec, uint8_t app_key[APP_KEY_LEN]);
esp_err_t espnow_sec_auth_encrypt(espnow_sec_t *sec, const uint8_t *input, size_t ilen, uint8_t *output, size_t output_len, size_t *olen, size_t tag_len);
esp_err_t espnow_sec_auth_decrypt(espnow_sec_t *sec, const uint8_t *input, size_t ilen, uint8_t *output, size_t output_len, size_t *olen, size_t tag_len);

// 握手
typedef enum { ESPNOW_SEC_TYPE_REQUEST, _INFO, _HANDSHAKE, _KEY, _KEY_RESP, _REST } espnow_sec_type_t;
typedef struct espnow_sec_responder_s { uint8_t mac[6]; int8_t rssi; uint8_t channel; uint8_t sec_ver; } espnow_sec_responder_t;
typedef struct espnow_sec_result_s { size_t unfinished_num, successed_num, requested_num; espnow_addr_t *unfinished_addr, *successed_addr, *requested_addr; } espnow_sec_result_t;

esp_err_t espnow_sec_initiator_scan(espnow_sec_responder_t **info_list, size_t *num, TickType_t wait_ticks);
esp_err_t espnow_sec_initiator_scan_result_free(void);
esp_err_t espnow_sec_initiator_start(uint8_t key_info[APP_KEY_LEN], const char *pop_data, const uint8_t addrs_list[][6], size_t addrs_num, espnow_sec_result_t *res);
esp_err_t espnow_sec_initiator_stop();
esp_err_t espnow_sec_initiator_result_free(espnow_sec_result_t *result);
esp_err_t espnow_sec_responder_start(const char *pop_data);
esp_err_t espnow_sec_responder_stop();
```

## OTA（espnow_ota.h）

```c
#define ESPNOW_OTA_HASH_LEN     16
#define ESPNOW_OTA_PROGRESS_MAX_SIZE  (ESPNOW_DATA_LEN - 30)
#define ESPNOW_OTA_PACKET_MAX_SIZE    ((ESPNOW_DATA_LEN - 4) - (ESPNOW_DATA_LEN -4) % 16)
// 事件: OTA_STARTED, OTA_STATUS, OTA_FINISH, OTA_STOPED, FIRMWARE_DOWNLOAD, SEND_FINISH
// 错误码: ESP_ERR_ESPNOW_OTA_FIRMWARE_NOT_INIT/PARTITION/INVALID/INCOMPLETE/DOWNLOAD/FINISH/DEVICE_NO_EXIST/SEND_PACKET_LOSS/NOT_INIT/STOP/FINISH

typedef struct espnow_ota_info_s     { uint8_t type; esp_app_desc_t app_desc; } espnow_ota_info_t;
typedef struct espnow_ota_responder_s{ uint8_t mac[6]; int8_t rssi; uint8_t channel; esp_app_desc_t app_desc; } espnow_ota_responder_t;
typedef struct espnow_ota_config_s   { bool skip_version_check; uint8_t progress_report_interval; } espnow_ota_config_t;
typedef struct espnow_ota_result_s   { size_t unfinished_num, successed_num, requested_num; espnow_addr_t *unfinished_addr, *successed_addr, *requested_addr; } espnow_ota_result_t;

typedef esp_err_t (*espnow_ota_initiator_data_cb_t)(size_t src_offset, void *dst, size_t size);

esp_err_t espnow_ota_initiator_send(const espnow_addr_t *addrs_list, size_t addrs_num, const uint8_t sha_256[ESPNOW_OTA_HASH_LEN], size_t size, espnow_ota_initiator_data_cb_t ota_data_cb, espnow_ota_result_t *res);
esp_err_t espnow_ota_initiator_stop();
esp_err_t espnow_ota_initiator_result_free(espnow_ota_result_t *result);
esp_err_t espnow_ota_initiator_scan(espnow_ota_responder_t **info_list, size_t *num, TickType_t wait_ticks);
esp_err_t espnow_ota_initiator_scan_result_free(void);
esp_err_t espnow_ota_responder_get_status(espnow_ota_status_t *status);
esp_err_t espnow_ota_responder_stop();
esp_err_t espnow_ota_responder_start(const espnow_ota_config_t *config);
```

## 配网（espnow_prov.h）

```c
#define ESPNOW_PROV_CUSTOM_MAX_SIZE 64
typedef enum { ESPNOW_PROV_AUTH_INVALID, _PRODUCT, _DEVICE, _CERT } espnow_prov_auth_mode_t;
typedef struct espnow_prov_initiator_s { char product_id[16]; char device_name[16]; espnow_prov_auth_mode_t auth_mode; union { char device_secret[32]; char product_secret[32]; char cert_secret[32]; }; uint8_t custom_size; uint8_t custom_data[0]; } espnow_prov_initiator_t;
typedef struct espnow_prov_responder_s { char product_id[16]; char device_name[16]; } espnow_prov_responder_t;
typedef struct espnow_prov_wifi_s { wifi_mode_t mode; union { wifi_ap_config_t ap; wifi_sta_config_t sta; }; char token[32]; uint8_t custom_size; uint8_t custom_data[0]; } espnow_prov_wifi_t;
typedef esp_err_t (*espnow_prov_cb_t)(uint8_t *src_addr, void *data, size_t size, wifi_pkt_rx_ctrl_t *rx_ctrl);

esp_err_t espnow_prov_initiator_scan(espnow_addr_t responder_addr, espnow_prov_responder_t *responder_info, wifi_pkt_rx_ctrl_t *rx_ctrl, TickType_t wait_ticks);
esp_err_t espnow_prov_initiator_send(const espnow_addr_t responder_addr, const espnow_prov_initiator_t *initiator_info, espnow_prov_cb_t cb, TickType_t wait_ticks);
esp_err_t espnow_prov_responder_start(const espnow_prov_responder_t *responder_info, TickType_t wait_ticks, const espnow_prov_wifi_t *wifi_config, espnow_prov_cb_t cb);
```

## 时间同步（espnow_time.h 节点间）

```c
#define ESP_EVENT_ESPNOW_TIMESYNC_STARTED/STOPPED/SYNCED/TIMEOUT  (ESP_EVENT_ESPNOW_TIMESYNC_BASE + 0/1/2/3)
typedef struct { uint8_t src_addr[6]; int32_t drift_ms; int64_t synced_time_us; } espnow_timesync_event_t;
typedef struct { uint32_t sync_interval_ms; } espnow_time_initiator_config_t;
typedef struct { int32_t max_drift_ms; } espnow_time_responder_config_t;
#define ESPNOW_TIME_INITIATOR_CONFIG_DEFAULT()  { .sync_interval_ms = 0 }
#define ESPNOW_TIME_RESPONDER_CONFIG_DEFAULT()  { .max_drift_ms = 100 }

esp_err_t espnow_time_initiator_start(const espnow_time_initiator_config_t *config);
esp_err_t espnow_time_initiator_stop(void);
esp_err_t espnow_time_initiator_broadcast(void);
esp_err_t espnow_time_responder_start(const espnow_time_responder_config_t *config);
esp_err_t espnow_time_responder_stop(void);
esp_err_t espnow_time_responder_request(void);
```

## 工具：存储/内存/通用（espnow_storage.h / espnow_mem.h / espnow_utils.h）

```c
// storage
esp_err_t espnow_storage_init(void);
esp_err_t espnow_storage_set(const char *key, const void *value, size_t length);
esp_err_t espnow_storage_get(const char *key, void *value, size_t length);
esp_err_t espnow_storage_erase(const char *key);

// mem 宏
ESP_MALLOC(size)  ESP_CALLOC(n,size)  ESP_REALLOC(ptr,size)  ESP_REALLOC_RETRY(ptr,size)  ESP_FREE(ptr)
void espnow_mem_add_record(void *ptr, int size, const char *tag, int line);
void espnow_mem_remove_record(void *ptr, const char *tag, int line);
void espnow_mem_print_record(void);
void espnow_mem_print_heap(void);
void espnow_mem_print_task(void);

// utils
esp_err_t espnow_reboot(TickType_t wait_ticks);
int  espnow_reboot_unbroken_count(void);
int  espnow_reboot_total_count(void);
bool espnow_reboot_is_exception(bool erase_coredump);
esp_err_t espnow_timesync_start(void);     // SNTP 对时（联网）
bool     espnow_timesync_check(void);
esp_err_t espnow_timesync_wait(uint32_t wait_ms);
void     espnow_print_system_info(uint32_t interval_ms);
uint8_t *espnow_mac_str2hex(const char *mac_str, uint8_t *mac_hex);

// 错误处理宏
ESP_PARAM_CHECK(con)
ESP_ERROR_CHECK(err)
ESP_ERROR_RETURN(con, err, fmt, ...)
ESP_ERROR_GOTO(con, label, fmt, ...)
ESP_ERROR_CONTINUE(con, fmt, ...)
ESP_ERROR_BREAK(con, fmt, ...)
ESP_ERROR_ASSERT(err)
```

## 调试：日志/Console/命令（espnow_log.h / espnow_console.h / espnow_cmd.h）

```c
// log
#define ESP_EVENT_ESPNOW_LOG_FLASH_FULL  (ESP_EVENT_ESPNOW_DEBUG_BASE + 1)
typedef esp_err_t (*espnow_log_custom_write_cb)(const char *data, size_t size, const char *tag, esp_log_level_t level);
typedef struct espnow_log_config_s { esp_log_level_t log_level_uart, log_level_flash, log_level_espnow, log_level_custom; espnow_log_custom_write_cb log_custom_write; } espnow_log_config_t;
esp_err_t espnow_log_init(const espnow_log_config_t *config);
esp_err_t espnow_log_deinit(void);
esp_err_t espnow_log_get_config(espnow_log_config_t *config);
esp_err_t espnow_log_set_config(const espnow_log_config_t *config);
esp_err_t espnow_log_flash_read(char *data, size_t *size);
size_t    espnow_log_flash_size(void);

// console
typedef struct espnow_console_config_s { struct { bool uart; bool espnow; } monitor_command; struct { const char *base_path; const char *partition_label; } store_history; } espnow_console_config_t;
esp_err_t espnow_console_init(const espnow_console_config_t *config);
esp_err_t espnow_console_deinit(void);
void     espnow_console_commands_register(void);

// cmd 注册函数
void register_espnow(void); void register_system(void); void register_wifi(void);
void register_peripherals(void); void register_iperf(void); void register_wifi_sniffer(void); void register_sdcard(void);
```

## 示例封装：examples/solution 组件（非组件公开 API）

> 以下函数来自 `examples/solution/components/espnow_device` 与 `examples/solution/components/wifi_prov`，是综合示例内部封装的应用层入口（非 `espressif/esp-now` 组件公开 API），仅在直接基于 `examples/solution` 改造工程时可见。

```c
// components/espnow_device/include/initiator.h
void     app_espnow_initiator_register(void);     // 注册事件、创建 event group
void     app_espnow_initiator(void);              // 启动 sec/console/prov 任务
#ifdef CONFIG_APP_ESPNOW_PROVISION
esp_err_t app_espnow_prov_beacon_start(int32_t sec);   // 启动 sec 秒 ESP-NOW 配网 beacon
#endif
#ifdef CONFIG_APP_ESPNOW_SECURITY
void     app_espnow_initiator_sec_start(void);    // 安全握手任务（scan + initiator_start + result_free）
#endif

// components/espnow_device/include/responder.h
void     app_espnow_responder_register(void);
void     app_espnow_responder(void);              // sec_responder_start + console/log/ota 启动
#ifdef CONFIG_APP_ESPNOW_PROVISION
esp_err_t app_espnow_prov_responder_start(void); // 反向 ESP-NOW 配网 initiator 任务（轮询 espnow_get_key）
#endif

// components/wifi_prov/include/wifi_prov.h  （仅 CONFIG_APP_WIFI_PROVISION / initiator）
void     wifi_prov_init(void);                    // 替代 app_wifi_init：注册 network_prov_mgr
void     wifi_prov(void);                          // 启动 BLE/SoftAP 上层配网（阻塞直至完成）
```

> `app_espnow_prov_beacon_start` 内部调用 `esp_wifi_get_config` 读取本机 STA 配置后通过 `espnow_prov_responder_start` 广播；`app_espnow_prov_responder_start` 实际启动的是 responder 侧的 *ESP-NOW 配网 initiator* 任务（向已联网的 initiator 请求 Wi-Fi 配置）。命名上注意区分“配网 beacon 角色”与“配网请求角色”。

