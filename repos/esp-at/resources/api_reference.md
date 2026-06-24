# ESP-AT 公开 API 速查

> 全部签名来自仓库真实头文件：`components/at/include/esp_at_core.h`、`esp_at.h`、`esp_at_cmd_register.h`、`esp_at_init.h`、`esp_at_types.h`。
> AT 核心库不开源，仅暴露这些 API。部分 API 受 Kconfig 条件编译保护（已标注）。

## 模块与类型（esp_at_core.h）

### 状态枚举

```c
typedef enum {
    ESP_AT_STATUS_NORMAL  = 0x0,  // 命令模式
    ESP_AT_STATUS_TRANSMIT,        // 透传模式
} esp_at_status_t;
```

### 睡眠模式

```c
typedef enum {
    ESP_AT_SLEEP_DISABLE  = 0,
    ESP_AT_SLEEP_MIN_MODEM,   // 最小 modem 睡眠
    ESP_AT_SLEEP_LIGHT,       // 浅睡
    ESP_AT_SLEEP_MAX_MODEM,   // 最大 modem 睡眠
    ESP_AT_SLEEP_MODE_MAX,
} esp_at_sleep_mode_t;
```

### 参数解析返回值

```c
typedef enum {
    ESP_AT_PARA_PARSE_RET_FAIL    = -1,
    ESP_AT_PARA_PARSE_RET_OK      =  0,
    ESP_AT_PARA_PARSE_RET_OMITTED,   // 参数被省略
} esp_at_para_parse_ret_t;
```

> 旧别名（esp_at_legacy.h，已 deprecated）：`ESP_AT_PARA_PARSE_RESULT_FAIL/OK/OMITTED`。新代码用 `_RET_`。

### 结果码（esp_at_rc_t）

```c
typedef enum {
    ESP_AT_RESULT_CODE_OK                  = 0x00,  // OK
    ESP_AT_RESULT_CODE_ERROR               = 0x01,  // ERROR
    ESP_AT_RESULT_CODE_FAIL                = 0x02,  // ERROR
    ESP_AT_RESULT_CODE_SEND_OK             = 0x03,  // SEND OK
    ESP_AT_RESULT_CODE_SEND_FAIL           = 0x04,  // SEND FAIL
    ESP_AT_RESULT_CODE_IGNORE              = 0x05,  // 不输出
    ESP_AT_RESULT_CODE_PROCESS_DONE        = 0x06,
    ESP_AT_RESULT_CODE_OK_AND_INPUT_PROMPT = 0x07,
    ESP_AT_RESULT_CODE_MAX,
} esp_at_rc_t;
```

## 数据结构

### 单条指令描述符

```c
typedef struct {
    char    *cmd_name;                          // 如 "+MYCMD"
    uint8_t (*test_cmd)(uint8_t *cmd_name);     // AT+CMD=?
    uint8_t (*query_cmd)(uint8_t *cmd_name);    // AT+CMD?
    uint8_t (*setup_cmd)(uint8_t para_num);     // AT+CMD=<...>
    uint8_t (*exe_cmd)(uint8_t *cmd_name);      // AT+CMD
} esp_at_cmd_t;
```

### 设备 I/O 回调（传输接口）

```c
typedef struct {
    int32_t (*read_data)(uint8_t *data, int32_t len);
    int32_t (*write_data)(uint8_t *data, int32_t len);
    int32_t (*get_data_length)(void);
    bool    (*wait_write_complete)(int32_t timeout_msec);
} esp_at_intf_ops_t;
```

### socket 透传回调

```c
typedef struct {
    int32_t (*recv_data)(uint8_t *data, int32_t len);
    void (*connect_cb)(void);
    void (*disconnect_cb)(void);
} esp_at_net_ops_t;
```

### BLE 透传回调

```c
typedef struct {
    int32_t (*recv_data)(uint8_t *data, int32_t len);
    void (*connect_cb)(void);
    void (*disconnect_cb)(void);
} esp_at_ble_ops_t;
```

### AT 核心生命周期回调

```c
typedef struct {
    void (*status_callback)(esp_at_status_t status);
    void (*pre_sleep_callback)(esp_at_sleep_mode_t mode);
    void (*pre_wakeup_callback)(void);
    void (*pre_deepsleep_callback)(void);
    void (*pre_restart_callback)(void);
    void (*pre_active_write_data_callback)(int32_t (*write_fn)(uint8_t *data, int32_t len));
} esp_at_custom_ops_t;
```

## 注册 API

```c
// 注册自定义指令数组
bool esp_at_custom_cmd_array_register(const esp_at_cmd_t *custom_at_cmd_array, uint32_t cmd_num);

// 注册传输接口 I/O
void esp_at_device_ops_register(esp_at_intf_ops_t *ops);

// 注册 socket 透传回调（需 esp_at_module_init 之后调用）
bool esp_at_custom_net_ops_register(int32_t link_id, esp_at_net_ops_t *ops);

// 注册 BLE 透传回调
bool esp_at_custom_ble_ops_register(int32_t conn_index, esp_at_ble_ops_t *ops);

// 注册生命周期回调
void esp_at_custom_ops_register(esp_at_custom_ops_t *ops);
```

> 底层符号名（库内）：`esp_at_custom_cmd_array_regist`、`esp_at_device_ops_regist` 等（旧别名见 esp_at_legacy.h）。

## 命令上下文

```c
const uint8_t *esp_at_get_current_cmd_name(void);  // 仅在 handler 执行期间有效

// 在二级分区表(at_customize.csv)中查找分区
const esp_partition_t *esp_at_custom_partition_find(esp_partition_type_t type,
                                                    esp_partition_subtype_t subtype,
                                                    const char *label);
```

## 参数解析

```c
esp_at_para_parse_ret_t esp_at_get_para_as_digit(int32_t para_index, int32_t *value);
esp_at_para_parse_ret_t esp_at_get_para_as_float(int32_t para_index, float *value);
esp_at_para_parse_ret_t esp_at_get_para_as_str(int32_t para_index, uint8_t **result);
// 返回的字符串指针指向 AT 核心内部缓冲，不可 free/修改
```

## 端口 I/O

```c
bool    esp_at_port_recv_data_notify(int32_t len, uint32_t msec);
int32_t esp_at_port_read_data(uint8_t *data, int32_t len);

// 写数据四变体（过滤 × 唤醒 MCU 两两组合）：
int32_t esp_at_port_write_data(uint8_t *data, int32_t len);                          // 过滤, 不唤醒
int32_t esp_at_port_write_data_without_filter(uint8_t *data, int32_t len);           // 不过滤, 不唤醒
int32_t esp_at_port_active_write_data(uint8_t *data, int32_t len);                   // 过滤, 唤醒 MCU
int32_t esp_at_port_active_write_data_without_filter(uint8_t *data, int32_t len);    // 不过滤, 唤醒 MCU

bool    esp_at_port_wait_write_complete(int32_t timeout_msec);
int32_t esp_at_port_get_data_length(void);

// 进入/退出特殊接收模式（接收原始数据）
void esp_at_port_enter_specific(esp_at_port_specific_callback_t callback);
void esp_at_port_exit_specific(void);
```

回调类型：

```c
typedef void (*esp_at_port_specific_callback_t)(void);
```

## 命令响应

```c
// 仅输出结果串，不改接收任务状态
void esp_at_write_result(uint8_t result_code);   // 底层符号 esp_at_response_result

// 输出结果串并把端口恢复到 ready（用于指令结束）
void esp_at_dispatch_result(esp_at_rc_t code, void *pbuf);  // 底层符号 at_handle_result_code
```

## 系统控制

```c
void esp_at_restart(void);       // 硬件看门狗兜底 + esp_restart()
void esp_at_restart_async(void); // 先回 OK 再重启（响应 AT 指令用）
```

## 工具

```c
// MAC 字符串 "XX:XX:XX:XX:XX:XX" → 6 字节数组
bool esp_at_str_2_mac(const char *str, uint8_t mac[6]);
```

## Wi-Fi（CONFIG_AT_WIFI_COMMAND_SUPPORT）

```c
esp_err_t esp_at_wifi_connect(void);                              // 替代 esp_wifi_connect
esp_err_t esp_at_wifi_disconnect(void);                           // 替代 esp_wifi_disconnect
void     esp_at_wifi_reconnect_init(bool force);                  // force=true 无条件重连
void     esp_at_wifi_reconnect_stop(void);
esp_err_t esp_at_wifi_scan_start(const wifi_scan_config_t *config, bool block);
```

## TCP/IP（CONFIG_AT_NET_COMMAND_SUPPORT）

```c
typedef enum {
    ESP_AT_IPPROTO_UNSPEC = 0,
    ESP_AT_IPPROTO_IPV4_IPV6,
    ESP_AT_IPPROTO_IPV4_ONLY,
    ESP_AT_IPPROTO_IPV6_ONLY,
} esp_at_ip_proto_t;

typedef enum {
    ESP_AT_NETIF_NONE = 0,
    ESP_AT_NETIF_STA,
    ESP_AT_NETIF_AP,
    ESP_AT_NETIF_ETH,
    ESP_AT_NETIF_MAX,
} esp_at_netif_t;

int          esp_at_connect(int fd, const struct sockaddr *name, socklen_t namelen, int timeout_ms);
int32_t      esp_at_get_socket_by_link_id(uint8_t link_id);
esp_at_netif_t esp_at_get_netif_by_socket(int fd);
esp_err_t    esp_at_hostname_to_ipaddr(const char *hostname, esp_at_ip_proto_t ip_proto,
                                       ip_addr_t *target_addr, uint32_t timeout_ms);
esp_err_t    esp_at_hostname_to_addrinfo(const char *hostname, esp_at_ip_proto_t ip_proto,
                                         ip_addr_t *target_addr, struct addrinfo **res, uint32_t timeout_ms);
```

## HTTP Client（CONFIG_AT_HTTP_COMMAND_SUPPORT）

```c
esp_err_t esp_at_http_set_pki_if_config(esp_http_client_config_t *config);   // 应用 AT+HTTPCFG 的 PKI
void     esp_at_http_free_pki_if_config(esp_http_client_config_t *config);
esp_err_t esp_at_http_set_header_if_config(esp_http_client_handle_t handle); // 应用 AT+HTTPCHEAD
esp_err_t esp_at_http_clear_header(void);
```

## WebSocket（CONFIG_AT_WS_COMMAND_SUPPORT）

```c
esp_websocket_client_handle_t esp_at_get_ws_client_handle_by_link_id(uint8_t link_id);  // link_id 0-2
int esp_at_websocket_client_send_by_opcode_fin(esp_websocket_client_handle_t handle,
                                               ws_transport_opcodes_t opcode, bool fin,
                                               const char *data, int length, TickType_t timeout);
```

## 应用层 API（esp_at.h）

```c
const char *esp_at_get_current_module_name(void);
void        esp_at_ready_before(void);                 // weak，AT 就绪前钩子

#ifdef CONFIG_AT_SELF_COMMAND_SUPPORT
esp_err_t esp_at_exe_cmd(const char *cmd, const char *expected_response, uint32_t timeout_ms);
// 不可在 AT handler 内调用，会死锁
#endif

// NVS 弱符号（可覆盖以加密）
esp_err_t esp_at_nvs_set_str(nvs_handle_t handle, const char *key, const char *value);
esp_err_t esp_at_nvs_get_str(nvs_handle_t handle, const char *key, char *out_value, size_t *length);
esp_err_t esp_at_nvs_set_blob(nvs_handle_t handle, const char *key, const void *value, size_t length);
esp_err_t esp_at_nvs_get_blob(nvs_handle_t handle, const char *key, void *out_value, size_t *length);

// 让出 CPU（防止饿死 idle task 触发看门狗）
void esp_at_yield_if_idle_timeout(uint32_t idle_timeout_ms, uint32_t yield_ticks);

// 文件系统
const char    *esp_at_fs_get_mount_point(void);   // 如 "/littlefs"
bool           esp_at_fs_mount(void);
bool           esp_at_fs_unmount(void);
esp_err_t      esp_at_get_fs_info(uint32_t *out_total_bytes, uint32_t *out_free_bytes);
esp_at_fs_type_t esp_at_fs_get_type(void);
```

## 文件系统类型（esp_at_types.h）

```c
typedef enum {
    ESP_AT_FS_FATFS = 0,
    ESP_AT_FS_LITTLEFS,
    ESP_AT_FS_TYPE_MAX,
} esp_at_fs_type_t;

// 挂载点与分区标签宏（按 Kconfig）
// CONFIG_AT_FS_LITTLEFS → AT_FS_MOUNT_POINT="/littlefs", AT_FS_PARTITION_LABEL="fs_storage"
// CONFIG_AT_FS_FATFS    → "/fatfs", "fs_storage"
// CONFIG_AT_FS_FATFS_LEGACY → "/fatfs", "fatfs"
```

## 日志宏（esp_at.h）

```c
#define ESP_AT_LOGE(tag, fmt, ...)   // ERROR 级
#define ESP_AT_LOGW(tag, fmt, ...)
#define ESP_AT_LOGI(tag, fmt, ...)
#define ESP_AT_LOGD(tag, fmt, ...)
#define ESP_AT_LOGV(tag, fmt, ...)
#define ESP_AT_LOG_BUFFER_HEXDUMP(tag, buffer, len, level)

#define ESP_AT_BUF_ON_STACK_SIZE  128
#define ESP_AT_PORT_TX_WAIT_MS_MAX 3000

void esp_at_log_write(esp_log_level_t level, const char *tag, const char *format, ...);  // weak
```

## 错误码编码（esp_at_core.h）

```c
#define ESP_AT_ERROR_NO(subcategory, extension) \
    ((ESP_AT_MODULE_NUM << 24) | ((subcategory) << 16) | (extension))

// 子类（esp_at_errno_t）：
ESP_AT_SUB_OK / _COMMON_ERROR / _NO_TERMINATOR / _NO_AT / _PARA_LENGTH_MISMATCH /
_PARA_TYPE_MISMATCH / _PARA_NUM_MISMATCH / _PARA_INVALID / _PARA_PARSE_FAIL /
_UNSUPPORT_CMD / _CMD_EXEC_FAIL / _CMD_PROCESSING / _CMD_OP_ERROR

// 预定义错误码宏（节选）：
ESP_AT_CMD_ERROR_OK
ESP_AT_CMD_ERROR_NON_FINISH
ESP_AT_CMD_ERROR_NOT_FOUND_AT
ESP_AT_CMD_ERROR_PARA_LENGTH(which_para)
ESP_AT_CMD_ERROR_PARA_TYPE(which_para)
ESP_AT_CMD_ERROR_PARA_NUM(need, given)
ESP_AT_CMD_ERROR_PARA_INVALID(which_para)
ESP_AT_CMD_ERROR_PARA_PARSE_FAIL(which_para)
ESP_AT_CMD_ERROR_CMD_UNSUPPORT
ESP_AT_CMD_ERROR_CMD_EXEC_FAIL(result)
ESP_AT_CMD_ERROR_CMD_PROCESSING
ESP_AT_CMD_ERROR_CMD_OP_ERROR
```

## 指令集初始化宏（esp_at_cmd_register.h）

```c
// 内部指令集（esp-at 项目内）：
ESP_AT_CMD_SET_FIRST_INIT_FN(f, priority)

// 外部/自定义指令集（推荐）：
ESP_AT_CMD_SET_INIT_FN(f, priority)

// 最后阶段：
ESP_AT_CMD_SET_LAST_INIT_FN(f, priority)
// priority 越大执行越晚；函数必须返回 true
```

仓库内部已注册的指令集函数（部分）：

```c
bool esp_at_base_cmd_register(void);
bool esp_at_wifi_cmd_register(void);
bool esp_at_smartconfig_cmd_register(void);
bool esp_at_wps_cmd_register(void);
bool esp_at_eap_cmd_register(void);
bool esp_at_mdns_cmd_register(void);
bool esp_at_net_cmd_register(void);
bool esp_at_ping_cmd_register(void);
bool esp_at_mqtt_cmd_register(void);
bool esp_at_http_cmd_register(void);
bool esp_at_ws_cmd_register(void);
bool esp_at_ble_cmd_register(void);
bool esp_at_ble_hid_cmd_register(void);
bool esp_at_blufi_cmd_register(void);
bool esp_at_ble_ota_cmd_register(void);
bool esp_at_bt_cmd_register(void);
bool esp_at_bt_spp_cmd_register(void);
bool esp_at_bt_a2dp_cmd_register(void);
bool esp_at_fs_cmd_register(void);
bool esp_at_driver_cmd_register(void);
bool esp_at_eth_cmd_register(void);
bool esp_at_fact_cmd_register(void);
bool esp_at_ota_cmd_register(void);
bool esp_at_uart_cmd_register(void);
bool esp_at_user_cmd_register(void);
bool esp_at_web_server_cmd_register(void);
bool esp_at_rainmaker_cmd_register(void);
```

## 入口（esp_at_init.h / main/app_main.c）

```c
void      esp_at_init(void);            // AT 主初始化，由 app_main 调用
esp_err_t esp_at_netif_init(void);      // netif 初始化
void      esp_at_cmd_set_register(void);// 注册所有指令集（内部）
// app_main 调用链：esp_at_main_preprocess() → nvs_flash_init() → esp_at_netif_init()
//                   → esp_event_loop_create_default() → esp_at_init()
```
