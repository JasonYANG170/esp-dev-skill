# esp-protocols API Reference（按组件分组）

> 全部签名取自仓库 `components/*/include/*.h`。修改 API 时务必回查头文件。

## esp_modem — 生命周期与模式（C API）

头文件：`components/esp_modem/include/esp_modem_c_api_types.h`、`components/esp_modem/include/esp_modem_config.h`、`components/esp_modem/include/esp_modem_dce_config.h`

```c
// 创建 / 销毁 / 模式
esp_modem_dce_t *esp_modem_new(const esp_modem_dte_config_t *dte_config,
                               const esp_modem_dce_config_t *dce_config,
                               esp_netif_t *netif);
esp_modem_dce_t *esp_modem_new_dev(esp_modem_dce_device_t module,
                                   const esp_modem_dte_config_t *dte_config,
                                   const esp_modem_dce_config_t *dce_config,
                                   esp_netif_t *netif);
void    esp_modem_destroy(esp_modem_dce_t *dce);
esp_err_t esp_modem_set_mode(esp_modem_dce_t *dce, esp_modem_dce_mode_t mode);
esp_modem_dce_mode_t esp_modem_get_mode(esp_modem_dce_t *dce);

// 错误回调 / APN / 网络
esp_err_t esp_modem_set_error_cb(esp_modem_dce_t *dce, esp_modem_terminal_error_cbt err_cb);
esp_err_t esp_modem_set_apn(esp_modem_dce_t *dce, const char *apn);
esp_err_t esp_modem_pause_net(esp_modem_dce_t *dce, bool pause);

// 通用 AT 命令执行
esp_err_t esp_modem_command(esp_modem_dce_t *dce, const char *command,
                            esp_err_t(*got_line_cb)(uint8_t *data, size_t len), uint32_t timeout_ms);

// USB DTE（需 CONFIG_EXAMPLE_SERIAL_CONFIG_USB 等条件，来自 USB 相关头）
// esp_modem_new_dev_usb() / esp_modem_dce_device_t / ESP_MODEM_*_USB_CONFIG()
```

### 关键枚举与宏

```c
// 模式
typedef enum {
    ESP_MODEM_MODE_COMMAND, ESP_MODEM_MODE_DATA, ESP_MODEM_MODE_CMUX,
    ESP_MODEM_MODE_CMUX_MANUAL, ESP_MODEM_MODE_CMUX_MANUAL_EXIT,
    ESP_MODEM_MODE_CMUX_MANUAL_SWAP, ESP_MODEM_MODE_CMUX_MANUAL_DATA,
    ESP_MODEM_MODE_CMUX_MANUAL_COMMAND, ESP_MODEM_MODE_DETECT,
    ESP_MODEM_MODE_UNDEF,
} esp_modem_dce_mode_t;

// 设备
typedef enum {
    ESP_MODEM_DCE_GENERIC, ESP_MODEM_DCE_SIM7600, ESP_MODEM_DCE_SIM7070,
    ESP_MODEM_DCE_SIM7000, ESP_MODEM_DCE_BG96, ESP_MODEM_DCE_EC20,
    ESP_MODEM_DCE_SIM800, ESP_MODEM_DCE_SQNGM02S, ESP_MODEM_DCE_CUSTOM,
} esp_modem_dce_device_t;

// SIM PIN 状态
typedef enum {
    ESP_MODEM_SIM_PIN_STATE_UNKNOWN, ESP_MODEM_SIM_PIN_STATE_READY,
    ESP_MODEM_SIM_PIN_STATE_NEED_PIN, ESP_MODEM_SIM_PIN_STATE_NEED_PUK,
    ESP_MODEM_SIM_PIN_STATE_OTHER,
} esp_modem_sim_pin_state_t;

// 终端错误
typedef enum {
    ESP_MODEM_TERMINAL_BUFFER_OVERFLOW, ESP_MODEM_TERMINAL_CHECKSUM_ERROR,
    ESP_MODEM_TERMINAL_UNEXPECTED_CONTROL_FLOW,
    ESP_MODEM_TERMINAL_DEVICE_GONE, ESP_MODEM_TERMINAL_UNKNOWN_ERROR,
} esp_modem_terminal_error_t;

#define ESP_MODEM_C_API_STR_BUF_SIZE CONFIG_ESP_MODEM_C_API_STR_MAX   // 默认 128

// 默认配置宏
#define ESP_MODEM_DTE_DEFAULT_CONFIG()   // 见 esp_modem_config.h（UART_NUM_1, 115200, tx=25, rx=26 ...）
#define ESP_MODEM_DCE_DEFAULT_CONFIG(APN) // { .apn = APN }
```

### DTE 配置结构体（`esp_modem_dte_config_t`）

```c
struct esp_modem_dte_config {
    size_t dte_buffer_size;
    uint32_t task_stack_size;
    unsigned task_priority;
    union {
        struct esp_modem_uart_term_config uart_config;  // port_num, data_bits, stop_bits, parity,
                                                        // flow_control, source_clk, baud_rate,
                                                        // tx_io_num, rx_io_num, rts_io_num, cts_io_num,
                                                        // rx_buffer_size, tx_buffer_size, event_queue_size
        struct esp_modem_vfs_term_config vfs_config;    // fd, deleter, resource
        void *extension_config;
    };
};

typedef enum {
    ESP_MODEM_FLOW_CONTROL_NONE = 0, ESP_MODEM_FLOW_CONTROL_SW, ESP_MODEM_FLOW_CONTROL_HW,
} esp_modem_flow_ctrl_t;

struct esp_modem_dce_config { const char *apn; };   // DCE 配置目前只有 APN
```

## esp_modem — AT 命令函数（C API）

头文件：`components/esp_modem/command/include/esp_modem_api.h`

```c
esp_err_t esp_modem_sync(esp_modem_dce_t *dce);
esp_err_t esp_modem_at(esp_modem_dce_t *dce, const char *cmd, char *out, int timeout);
esp_err_t esp_modem_at_raw(esp_modem_dce_t *dce, const char *cmd, char *out,
                           const char *pass, const char *fail, int timeout);

esp_err_t esp_modem_set_pin(esp_modem_dce_t *dce, const char *pin);
esp_err_t esp_modem_reset_pin(esp_modem_dce_t *dce, const char *puk, const char *pin);
esp_err_t esp_modem_read_pin(esp_modem_dce_t *dce, bool *pin_ok);
esp_err_t esp_modem_read_pin_state(esp_modem_dce_t *dce, esp_modem_sim_pin_state_t *state);
esp_err_t esp_modem_set_echo(esp_modem_dce_t *dce, const bool echo_on);

esp_err_t esp_modem_get_imsi(esp_modem_dce_t *dce, char *imsi);
esp_err_t esp_modem_get_imei(esp_modem_dce_t *dce, char *imei);
esp_err_t esp_modem_get_module_name(esp_modem_dce_t *dce, char *name);
esp_err_t esp_modem_get_operator_name(esp_modem_dce_t *dce, char *name, int *act);
esp_err_t esp_modem_get_signal_quality(esp_modem_dce_t *dce, int *rssi, int *ber);
esp_err_t esp_modem_get_battery_status(esp_modem_dce_t *dce, int *voltage, int *bcs, int *bcl);

esp_err_t esp_modem_sms_txt_mode(esp_modem_dce_t *dce, const bool txt);
esp_err_t esp_modem_sms_character_set(esp_modem_dce_t *dce);
esp_err_t esp_modem_send_sms(esp_modem_dce_t *dce, const char *number, const char *message);

esp_err_t esp_modem_set_data_mode(esp_modem_dce_t *dce);
esp_err_t esp_modem_set_command_mode(esp_modem_dce_t *dce);
esp_err_t esp_modem_set_cmux(esp_modem_dce_t *dce);
esp_err_t esp_modem_resume_data_mode(esp_modem_dce_t *dce);

esp_err_t esp_modem_set_flow_control(esp_modem_dce_t *dce, int dce_flow, int dte_flow);
esp_err_t esp_modem_set_baud(esp_modem_dce_t *dce, int baud);
esp_err_t esp_modem_hang_up(esp_modem_dce_t *dce);
esp_err_t esp_modem_power_down(esp_modem_dce_t *dce);
esp_err_t esp_modem_reset(esp_modem_dce_t *dce);
esp_err_t esp_modem_store_profile(esp_modem_dce_t *dce);

esp_err_t esp_modem_set_operator(esp_modem_dce_t *dce, int mode, int format, const char *oper);
esp_err_t esp_modem_set_network_attachment_state(esp_modem_dce_t *dce, int state);
esp_err_t esp_modem_get_network_attachment_state(esp_modem_dce_t *dce, int *state);
esp_err_t esp_modem_set_radio_state(esp_modem_dce_t *dce, int state);
esp_err_t esp_modem_get_radio_state(esp_modem_dce_t *dce, int *state);
esp_err_t esp_modem_set_network_mode(esp_modem_dce_t *dce, int mode);
esp_err_t esp_modem_set_preferred_mode(esp_modem_dce_t *dce, int mode);
esp_err_t esp_modem_set_network_bands(esp_modem_dce_t *dce, const char *mode, const int *bands, int size);
esp_err_t esp_modem_get_network_system_mode(esp_modem_dce_t *dce, int *mode);
esp_err_t esp_modem_set_gnss_power_mode(esp_modem_dce_t *dce, int mode);
esp_err_t esp_modem_get_gnss_power_mode(esp_modem_dce_t *dce, int *mode);
esp_err_t esp_modem_config_psm(esp_modem_dce_t *dce, int mode, const char *tau, const char *active_time);
esp_err_t esp_modem_config_network_registration_urc(esp_modem_dce_t *dce, int value);
esp_err_t esp_modem_get_network_registration_state(esp_modem_dce_t *dce, int *state);
esp_err_t esp_modem_config_mobile_termination_error(esp_modem_dce_t *dce, int mode);
esp_err_t esp_modem_config_edrx(esp_modem_dce_t *dce, int mode, int access_technology, const char *edrx_value);
esp_err_t esp_modem_set_pdp_context(esp_modem_dce_t *dce, esp_modem_PdpContext_t *pdp);

// URC（需 CONFIG_ESP_MODEM_URC_HANDLER=y）
esp_err_t esp_modem_set_urc(esp_modem_dce_t *dce, esp_err_t(*got_line_cb)(uint8_t *data, size_t len));
```

## mdns

头文件：`components/mdns/include/mdns.h`

```c
// 初始化与 hostname
esp_err_t mdns_init(void);
void     mdns_free(void);
esp_err_t mdns_hostname_set(const char *hostname);
esp_err_t mdns_hostname_get(char *hostname);          // 缓冲 >= MDNS_NAME_BUF_LEN
esp_err_t mdns_instance_name_set(const char *instance_name);

// 委托主机
esp_err_t mdns_delegate_hostname_add(const char *hostname, const mdns_ip_addr_t *address_list);
esp_err_t mdns_delegate_hostname_set_address(const char *hostname, const mdns_ip_addr_t *address_list);
esp_err_t mdns_delegate_hostname_remove(const char *hostname);
bool     mdns_hostname_exists(const char *hostname);

// 服务发布
esp_err_t mdns_service_add(const char *instance_name, const char *service_type, const char *proto,
                           uint16_t port, mdns_txt_item_t txt[], size_t num_items);
esp_err_t mdns_service_add_for_host(const char *instance_name, const char *service_type, const char *proto,
                                    const char *hostname, uint16_t port, mdns_txt_item_t txt[], size_t num_items);
esp_err_t mdns_service_remove(const char *service_type, const char *proto);
esp_err_t mdns_service_remove_for_host(const char *instance, const char *service_type, const char *proto, const char *hostname);
esp_err_t mdns_service_remove_all(void);

// 服务 TXT / 端口 / 子类型
esp_err_t mdns_service_port_set(const char *service_type, const char *proto, uint16_t port);
esp_err_t mdns_service_port_set_for_host(const char *instance, const char *service_type, const char *proto, const char *hostname, uint16_t port);
esp_err_t mdns_service_txt_set(const char *service_type, const char *proto, mdns_txt_item_t txt[], uint8_t num_items);
esp_err_t mdns_service_txt_set_for_host(const char *instance, const char *service_type, const char *proto, const char *hostname, mdns_txt_item_t txt[], uint8_t num_items);
esp_err_t mdns_service_txt_item_set(const char *service_type, const char *proto, const char *key, const char *value);
esp_err_t mdns_service_txt_item_set_with_explicit_value_len(const char *service_type, const char *proto, const char *key, const char *value, uint8_t value_len);
esp_err_t mdns_service_txt_item_set_for_host(const char *instance, const char *service_type, const char *proto, const char *hostname, const char *key, const char *value);
esp_err_t mdns_service_txt_item_set_for_host_with_explicit_value_len(const char *instance, const char *service_type, const char *proto, const char *hostname, const char *key, const char *value, uint8_t value_len);
esp_err_t mdns_service_txt_item_remove(const char *service_type, const char *proto, const char *key);
esp_err_t mdns_service_txt_item_remove_for_host(const char *instance, const char *service_type, const char *proto, const char *hostname, const char *key);
esp_err_t mdns_service_subtype_add_for_host(const char *instance_name, const char *service_type, const char *proto, const char *hostname, const char *subtype);
esp_err_t mdns_service_subtype_remove_for_host(const char *instance_name, const char *service_type, const char *proto, const char *hostname, const char *subtype);

// 查询
esp_err_t mdns_query(const char *name, const char *service_type, const char *proto, uint16_t type, uint32_t timeout, size_t max_results, mdns_result_t **results);
esp_err_t mdns_query_generic(const char *name, const char *service_type, const char *proto, uint16_t type, uint32_t timeout, size_t max_results, mdns_result_t **results);
esp_err_t mdns_query_ptr(const char *service_type, const char *proto, uint32_t timeout, size_t max_results, mdns_result_t **results);
esp_err_t mdns_query_srv(const char *instance_name, const char *service_type, const char *proto, uint32_t timeout, mdns_result_t **result);
esp_err_t mdns_query_txt(const char *instance_name, const char *service_type, const char *proto, uint32_t timeout, mdns_result_t **result);
esp_err_t mdns_query_a(const char *host_name, uint32_t timeout, esp_ip4_addr_t *addr);
esp_err_t mdns_query_aaaa(const char *host_name, uint32_t timeout, esp_ip6_addr_t *addr);
void     mdns_query_results_free(mdns_result_t *results);
```

### mdns 类型与宏

```c
typedef struct { const char *key; const char *value; } mdns_txt_item_t;
typedef struct { const char *subtype; } mdns_subtype_item_t;
typedef struct mdns_ip_addr_s { esp_ip_addr_t addr; struct mdns_ip_addr_s *next; } mdns_ip_addr_t;

typedef enum { MDNS_IP_PROTOCOL_V4, MDNS_IP_PROTOCOL_V6, MDNS_IP_PROTOCOL_MAX } mdns_ip_protocol_t;

#define MDNS_TYPE_A 0x0001
#define MDNS_TYPE_PTR 0x000C
#define MDNS_TYPE_TXT 0x0010
#define MDNS_TYPE_AAAA 0x001C
#define MDNS_TYPE_SRV 0x0021
#define MDNS_TYPE_ANY 0x00FF
#define MDNS_NAME_MAX_LEN 64
#define MDNS_NAME_BUF_LEN (MDNS_NAME_MAX_LEN+1)
```

## esp_websocket_client

头文件：`components/esp_websocket_client/include/esp_websocket_client.h`

```c
typedef struct esp_websocket_client *esp_websocket_client_handle_t;

esp_websocket_client_handle_t esp_websocket_client_init(const esp_websocket_client_config_t *config);
esp_err_t esp_websocket_client_set_uri(esp_websocket_client_handle_t client, const char *uri);
esp_err_t esp_websocket_client_set_headers(esp_websocket_client_handle_t client, const char *headers);
esp_err_t esp_websocket_client_append_header(esp_websocket_client_handle_t client, const char *key, const char *value);
esp_err_t esp_websocket_client_start(esp_websocket_client_handle_t client);
esp_err_t esp_websocket_client_stop(esp_websocket_client_handle_t client);
esp_err_t esp_websocket_client_destroy(esp_websocket_client_handle_t client);
esp_err_t esp_websocket_client_destroy_on_exit(esp_websocket_client_handle_t client);

int esp_websocket_client_send_text(esp_websocket_client_handle_t client, const char *data, int len, TickType_t timeout);
int esp_websocket_client_send_bin(esp_websocket_client_handle_t client, const char *data, int len, TickType_t timeout);
int esp_websocket_client_send_text_partial(esp_websocket_client_handle_t client, const char *data, int len, TickType_t timeout);
int esp_websocket_client_send_bin_partial(esp_websocket_client_handle_t client, const char *data, int len, TickType_t timeout);
int esp_websocket_client_send_cont_msg(esp_websocket_client_handle_t client, const char *data, int len, TickType_t timeout);
int esp_websocket_client_send_fin(esp_websocket_client_handle_t client, TickType_t timeout);
int esp_websocket_client_send_with_opcode(esp_websocket_client_handle_t client, ws_transport_opcodes_t opcode, const uint8_t *data, int len, TickType_t timeout);

esp_err_t esp_websocket_client_close(esp_websocket_client_handle_t client, TickType_t timeout);
esp_err_t esp_websocket_client_close_with_code(esp_websocket_client_handle_t client, int code, const char *data, int len, TickType_t timeout);

bool     esp_websocket_client_is_connected(esp_websocket_client_handle_t client);
size_t   esp_websocket_client_get_ping_interval_sec(esp_websocket_client_handle_t client);
esp_err_t esp_websocket_client_set_ping_interval_sec(esp_websocket_client_handle_t client, size_t ping_interval_sec);
int      esp_websocket_client_get_reconnect_timeout(esp_websocket_client_handle_t client);
esp_err_t esp_websocket_client_set_reconnect_timeout(esp_websocket_client_handle_t client, int reconnect_timeout_ms);

esp_err_t esp_websocket_register_events(esp_websocket_client_handle_t client, esp_websocket_event_id_t event,
                                        esp_event_handler_t event_handler, void *event_handler_arg);
esp_err_t esp_websocket_unregister_events(esp_websocket_client_handle_t client, esp_websocket_event_id_t event,
                                          esp_event_handler_t event_handler);
```

### websocket 枚举

```c
typedef enum {
    WEBSOCKET_EVENT_ANY = -1, WEBSOCKET_EVENT_ERROR = 0,
    WEBSOCKET_EVENT_CONNECTED, WEBSOCKET_EVENT_DISCONNECTED,
    WEBSOCKET_EVENT_DATA, WEBSOCKET_EVENT_CLOSED,
    WEBSOCKET_EVENT_BEFORE_CONNECT, WEBSOCKET_EVENT_BEGIN, WEBSOCKET_EVENT_FINISH,
    WEBSOCKET_EVENT_MAX
} esp_websocket_event_id_t;
// IDF>=6.0 还有 WEBSOCKET_EVENT_HEADER_RECEIVED（需 WS_TRANSPORT_HEADER_CALLBACK_SUPPORT=1）

typedef enum { WEBSOCKET_TRANSPORT_UNKNOWN = 0x0, WEBSOCKET_TRANSPORT_OVER_TCP, WEBSOCKET_TRANSPORT_OVER_SSL } esp_websocket_transport_t;

typedef enum {
    WEBSOCKET_ERROR_TYPE_NONE, WEBSOCKET_ERROR_TYPE_TCP_TRANSPORT,
    WEBSOCKET_ERROR_TYPE_PONG_TIMEOUT, WEBSOCKET_ERROR_TYPE_HANDSHAKE,
    WEBSOCKET_ERROR_TYPE_SERVER_CLOSE,
} esp_websocket_error_type_t;
```

### websocket 配置结构体关键字段（`esp_websocket_client_config_t`）

`uri`、`host`、`port`、`username`、`password`、`path`、`transport`、`subprotocol`、`user_agent`、`headers`、`cert_pem`/`cert_len`、`client_cert`/`client_cert_len`、`client_key`/`client_key_len`、`client_ds_data`、`use_global_ca_store`、`crt_bundle_attach`、`cert_common_name`、`skip_cert_common_name_check`、`buffer_size`、`task_prio`、`task_stack`、`task_name`、`task_core_id`/`task_core_id_set`、`disable_auto_reconnect`、`enable_close_reconnect`、`ping_interval_sec`、`pingpong_timeout_sec`、`disable_pingpong_discon`、`reconnect_timeout_ms`、`network_timeout_ms`、`keep_alive_enable`/`keep_alive_idle`/`keep_alive_interval`/`keep_alive_count`、`if_name`、`ext_transport`、`user_context`。

## eppp_link

头文件：`components/eppp_link/include/eppp_link.h`

```c
typedef enum { EPPP_SERVER, EPPP_CLIENT } eppp_type_t;
typedef enum { EPPP_TRANSPORT_UART, EPPP_TRANSPORT_SPI, EPPP_TRANSPORT_SDIO, EPPP_TRANSPORT_ETHERNET } eppp_transport_t;

esp_netif_t *eppp_connect(eppp_config_t *config);                 // client 建链
esp_netif_t *eppp_listen(eppp_config_t *config);                  // server 监听
esp_netif_t *eppp_init(eppp_type_t role, eppp_config_t *config);
esp_netif_t *eppp_open(eppp_type_t role, eppp_config_t *config, int connect_timeout_ms);
esp_netif_t *eppp_netif_init(eppp_type_t role, eppp_transport_handle_t h, eppp_config_t *eppp_config);
esp_err_t    eppp_netif_start(esp_netif_t *netif);
esp_err_t    eppp_netif_stop(esp_netif_t *netif, int stop_timeout_ms);
esp_err_t    eppp_perform(esp_netif_t *netif);
esp_err_t    eppp_add_channels(esp_netif_t *netif, eppp_channel_fn_t *tx, const eppp_channel_fn_t rx, void *context);

#define EPPP_DEFAULT_SERVER_IP()  ESP_IP4TOADDR(192,168,11,1)
#define EPPP_DEFAULT_CLIENT_IP()  ESP_IP4TOADDR(192,168,11,2)
#define EPPP_DEFAULT_CONFIG(our_ip, their_ip) { ... }
#define EPPP_DEFAULT_SERVER_CONFIG()  // our=server_ip, their=client_ip
#define EPPP_DEFAULT_CLIENT_CONFIG()  // our=client_ip,  their=server_ip
#define EPPP_DEFAULT_UART_CONFIG() / EPPP_DEFAULT_SPI_CONFIG() / EPPP_DEFAULT_SDIO_CONFIG() / EPPP_DEFAULT_ETH_CONFIG()
```

## esp_dns

头文件：`components/esp_dns/include/esp_dns.h`

```c
typedef enum { ESP_DNS_PROTOCOL_UDP, ESP_DNS_PROTOCOL_TCP, ESP_DNS_PROTOCOL_DOT, ESP_DNS_PROTOCOL_DOH } esp_dns_protocol_type_t;
typedef struct esp_dns_handle *esp_dns_handle_t;

esp_dns_handle_t esp_dns_init_doh(esp_dns_config_t *config);
esp_dns_handle_t esp_dns_init_dot(esp_dns_config_t *config);
esp_dns_handle_t esp_dns_init_tcp(esp_dns_config_t *config);
esp_dns_handle_t esp_dns_init_udp(esp_dns_config_t *config);

int esp_dns_cleanup_doh(esp_dns_handle_t handle);
int esp_dns_cleanup_dot(esp_dns_handle_t handle);
int esp_dns_cleanup_tcp(esp_dns_handle_t handle);
int esp_dns_cleanup_udp(esp_dns_handle_t handle);

#define ESP_DNS_DEFAULT_TCP_PORT 53
#define ESP_DNS_DEFAULT_DOT_PORT 853
#define ESP_DNS_DEFAULT_DOH_PORT 443
#define ESP_DNS_DEFAULT_TIMEOUT_MS 10000
```

### esp_dns 配置结构体（`esp_dns_config_t`）

```c
typedef struct {
    esp_dns_protocol_type_t protocol;
    const char *dns_server;
    uint16_t port;
    uint32_t timeout_ms;
    struct { const char *cert_pem; esp_err_t (*crt_bundle_attach)(void *conf); } tls_config;
    union { struct { const char *url_path; } doh_config; } protocol_config;
} esp_dns_config_t;
```

## esp_mqtt_cxx — C++ MQTT 客户端（`idf::mqtt::Client`）

头文件：`components/esp_mqtt_cxx/include/esp_mqtt.hpp`、`components/esp_mqtt_cxx/include/esp_mqtt_client_config.hpp`

> 编译前提：`CONFIG_COMPILER_CXX_EXCEPTIONS=y`（否则 `esp_mqtt.hpp` 直接 `#error`）。命名空间 `idf::mqtt`。

```cpp
// 枚举
enum class QoS { AtMostOnce = 0, AtLeastOnce = 1, ExactlyOnce = 2 };
enum class Retain : bool { NotRetained = false, Retained = true };
enum class MessageID : int {};

// 异常（继承 esp_exception 的 ESPException）
struct MQTTException : ESPException { using ESPException::ESPException; };

// 消息模板（Container 须为连续内存类型）
template <typename T> struct Message {
    T data;
    QoS qos = QoS::AtLeastOnce;
    Retain retain = Retain::NotRetained;
};
using StringMessage = Message<std::string>;

// 主题过滤器（构造非法抛 std::domain_error）
class Filter {
public:
    explicit Filter(std::string user_filter);
    const std::string &get();
    [[nodiscard]] bool match(std::string::const_iterator begin, std::string::const_iterator end) const noexcept;
    [[nodiscard]] bool match(const std::string &topic) const noexcept;
    [[nodiscard]] bool match(char *begin, int size) const noexcept;
};

// 客户端基类（必须继承，on_connected/on_data 为纯虚）
class Client {
public:
    Client(const BrokerConfiguration &broker, const ClientCredentials &credentials, const Configuration &config);
    Client(const esp_mqtt_client_config_t &config);   // 备选：直接传 C 配置
    void start();                                       // 必须在派生类构造完成后调用
    [[nodiscard]] bool is_started() const noexcept;
    std::optional<MessageID> subscribe(const std::string &topic_filter, QoS qos = QoS::AtLeastOnce);

    template <class Container>
    std::optional<MessageID> publish(const std::string &topic, const Message<Container> &message);
    template <class InputIt>
    std::optional<MessageID> publish(const std::string &topic, InputIt first, InputIt last,
                                     QoS qos = QoS::AtLeastOnce, Retain retain = Retain::NotRetained);

    virtual ~Client() = default;
protected:
    using ClientHandler = std::unique_ptr<esp_mqtt_client, MqttClientDeleter>;
    ClientHandler handler;   // 底层 esp_mqtt 句柄（unique_ptr，析构自动 destroy）
    virtual void on_error(const esp_mqtt_event_handle_t event);
    virtual void on_disconnected(const esp_mqtt_event_handle_t event);
    virtual void on_subscribed(const esp_mqtt_event_handle_t event);
    virtual void on_unsubscribed(const esp_mqtt_event_handle_t event);
    virtual void on_published(const esp_mqtt_event_handle_t event);
    virtual void on_before_connect(const esp_mqtt_event_handle_t event);
    virtual void on_connected(const esp_mqtt_event_handle_t event) = 0;   // 必须重写
    virtual void on_data(const esp_mqtt_event_handle_t event) = 0;        // 必须重写
};
```

### esp_mqtt_cxx 配置结构体（`esp_mqtt_client_config.hpp`）

```cpp
struct Host    { std::string address; std::string path; esp_mqtt_transport_t transport; };
struct URI     { std::string address; };
struct BrokerAddress { std::variant<Host, URI> address; uint32_t port = 0; };

struct PEM { const char *data; };
struct DER { const char *data; size_t len; };
using CryptographicInformation = std::variant<PEM, DER>;
struct Insecure {};
struct GlobalCAStore {};
struct PSK { const struct psk_key_hint *hint_key; };   // esp_tls.h 的 PSK 结构
using BrokerAuthentication = std::variant<Insecure, GlobalCAStore, CryptographicInformation, PSK>;

struct BrokerConfiguration { BrokerAddress address; BrokerAuthentication security; };

struct Password { std::string data; };
struct ClientCertificate {
    CryptographicInformation certificate;
    CryptographicInformation key;
    std::optional<Password> key_password = std::nullopt;
};
struct SecureElement {};
struct DigitalSignatureData { void *ds_data; };
using AuthenticationFactor = std::variant<Password, ClientCertificate, SecureElement>;

struct ClientCredentials {
    std::optional<std::string> username;
    AuthenticationFactor authentication;
    std::vector<std::string> alpn_protos;
    std::optional<std::string> client_id = std::nullopt;   // 默认 ESP32_<CHIPID>
};

struct LastWill {
    const char *lwt_topic; const char *lwt_msg;
    int lwt_qos; int lwt_retain; int lwt_msg_len;
};
struct Session {
    LastWill last_will;
    int disable_clean_session;
    int keepalive;                 // 默认 120 秒
    bool disable_keepalive;
    esp_mqtt_protocol_ver_t protocol_ver;
};
struct Task { int task_prio; int task_stack; };   // 默认 5 / 6144
struct Connection {
    esp_mqtt_transport_t transport;
    int reconnect_timeout_ms;      // 默认 10000
    int network_timeout_ms;        // 默认 10000
    int refresh_connection_after_ms;
    bool disable_auto_reconnect;
};
struct Configuration {
    Task task; Session session; Connection connection;
    void *user_context;
    int buffer_size;               // 默认 1024
    int out_buffer_size;
};
```

## mosquitto — 板载 MQTT Broker

头文件：`components/mosquitto/port/include/mosq_broker.h`

```c
// 消息回调：mosquitto 处理消息时调用
typedef void (*mosq_message_cb_t)(char *client, char *topic, char *data, int len, int qos, int retain);

// 连接回调：客户端尝试连接时调用。返回 0 接受，非 0 拒绝
typedef int (*mosq_connect_cb_t)(const char *client_id, const char *username,
                                 const char *password, int password_len);

// 配置结构（ESP 移植仅支持以下字段）
struct mosq_broker_config {
    const char *host;                // 监听地址，如 "0.0.0.0"
    int port;                        // 监听端口，如 1883
    esp_tls_cfg_server_t *tls_cfg;   // TLS 配置（NULL 则明文 TCP）
    void (*handle_message_cb)(char *client, char *topic, char *data, int len, int qos, int retain);
    mosq_connect_cb_t handle_connect_cb;   // basic auth 校验回调
};

// 启动 broker（在调用线程阻塞运行，不建独立任务）。返回 int：0 成功
int  mosq_broker_run(struct mosq_broker_config *config);

// 停止 broker（调用后 mosq_broker_run() 解除阻塞并返回）
void mosq_broker_stop(void);
```

> 内存占用（README）：约 60 kB Flash；启动约 2 kB 堆；每客户端约 4 kB 堆；异常断连重连期间保留旧连接信息。推荐任务栈 ≥ 5 kB。

## esp_modem NAPT 网关相关（ap_to_pppos example）

> 以下 API 来自 ESP-IDF（lwip / dhcpserver / esp_wifi）与 example 自带的 `network_dce.h`/`network_dce.cpp`，非 esp_modem 组件本身的导出。

```c
// lwip NAPT（需 CONFIG_LWIP_IP_FORWARD=y + CONFIG_LWIP_IPV4_NAPT=y）
//   对 soft-AP 接口 IP 开启 NAPT，使 AP 客户端流量经 PPP 出口转发
err_t ip_napt_enable(ip4_addr_t addr, u8_t enable);   // lwip/lwip_napt.h

// example 提供的 C-API 包装（network_dce.{c,cpp}，单例 DCE）
esp_err_t modem_init_network(esp_netif_t *netif);   // 创建 DCE 并绑定 PPP netif
bool      modem_start_network();                    // set_mode(DATA_MODE)
bool      modem_stop_network();                     // set_mode(COMMAND_MODE)
bool      modem_check_sync();                       // esp_modem_sync
bool      modem_check_signal();                     // rssi!=99 && rssi>5
void      modem_reset();                            // esp_modem_reset
void      modem_deinit_network();                   // 销毁 DCE
```

### Minimal DCE（C++，`network_dce.cpp`，需 `CONFIG_EXAMPLE_USE_MINIMAL_DCE=y`）

```cpp
namespace esp_modem {
class ModuleIf {
public:
    virtual bool setup_data_mode() = 0;     // 设 PDP context
    virtual bool set_mode(modem_mode mode) = 0;   // DATA_MODE / COMMAND_MODE
    // ... 其他虚函数
};
namespace dce_factory {
class Factory {
protected:
    template <typename Module, typename ...Args>
    static DCE_T<Module> *build_generic_DCE(const config *cfg, Args &&... args);
    template <typename Module, typename ...Args>
    static std::shared_ptr<Module> build_shared_module(const config *cfg, Args &&... args);
};
}   // dce_factory
}   // esp_modem

// example 自定义工厂与模块
class NetDCE_Factory: public esp_modem::dce_factory::Factory { /* create / create_module 模板 */ };
class NetModule: public esp_modem::ModuleIf {
    explicit NetModule(std::shared_ptr<esp_modem::DTE> dte, const esp_modem_dce_config *cfg);
    bool setup_data_mode() override;   // PdpContext(apn) + set_pdp_context
    bool set_mode(esp_modem::modem_mode mode) override;   // set_data_mode/resume_data_mode/set_command_mode
};
```

