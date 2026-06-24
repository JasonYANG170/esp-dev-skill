# esp-thread-br 扩展 API 速查

> 仅收录 esp-thread-br 仓库自身提供的 API（`components/`）。OpenThread 栈 API 见 https://openthread.io/reference，ESP-IDF / esp-openthread API 见 ESP-IDF 文档。所有签名取自仓库头文件。

## 1. RCP 更新 — `components/esp_rcp_update/include/esp_rcp_update.h`

```c
typedef enum {
    RCP_TYPE_INVALID = 0,
    RCP_TYPE_UART    = 1,
    RCP_TYPE_MAX     = 2,
} esp_rcp_type_t;

typedef struct {
    esp_rcp_type_t rcp_type;
    int uart_rx_pin;
    int uart_tx_pin;
    int uart_port;
    int uart_baudrate;
    int reset_pin;
    int boot_pin;
    uint32_t update_baudrate;
    char firmware_dir[RCP_FIRMWARE_DIR_SIZE];   // RCP_FIRMWARE_DIR_SIZE = 20
    target_chip_t target_chip;                   // ESP32H2_CHIP / ESP32C6_CHIP
} esp_rcp_update_config_t;

// 默认配置宏（H2 为默认 target_chip）
#define ESP_RCP_UPDATE_DEFAULT_CONFIG()  ...

esp_err_t esp_rcp_update_init(const esp_rcp_update_config_t *update_config);
esp_err_t esp_rcp_update(void);
const char *esp_rcp_get_firmware_dir(void);
int8_t esp_rcp_get_update_seq(void);
int8_t esp_rcp_get_next_update_seq(void);
void esp_rcp_reset(void);
esp_err_t esp_rcp_submit_new_image(void);
esp_err_t esp_rcp_mark_image_verified(bool verified);
esp_err_t esp_rcp_mark_image_unusable(void);
esp_err_t esp_rcp_load_version_in_storage(char *version_str, size_t size);
void esp_rcp_update_deinit(void);
```

## 2. RCP OTA 流式接口 — `components/esp_rcp_update/include/esp_rcp_ota.h`

```c
typedef enum {
    ESP_RCP_OTA_STATE_READ_HEADER = 0,
    ESP_RCP_OTA_STATE_DOWNLOAD_RCP_FW,
    ESP_RCP_OTA_STATE_FINISHED,
    ESP_RCP_OTA_STATE_INVALID,
} esp_rcp_ota_state_t;

typedef uint32_t esp_rcp_ota_handle_t;

esp_err_t esp_rcp_ota_begin(esp_rcp_ota_handle_t *out_handle);
esp_rcp_ota_state_t esp_rcp_ota_get_state(esp_rcp_ota_handle_t handle);
uint32_t esp_rcp_ota_get_subfile_size(esp_rcp_ota_handle_t handle, esp_rcp_filetag_t filetag);
esp_err_t esp_rcp_ota_receive(esp_rcp_ota_handle_t handle, const void *data, size_t size, size_t *received_size);
esp_err_t esp_rcp_ota_end(esp_rcp_ota_handle_t handle);
esp_err_t esp_rcp_ota_abort(esp_rcp_ota_handle_t handle);
```

## 3. RCP 镜像 filetag — `components/esp_rcp_update/include/esp_rcp_firmware.h`

```c
#define MAX_SUBFILE_INFO 7

typedef enum {
    FILETAG_RCP_VERSION        = 0,
    FILETAG_RCP_FLASH_ARGS     = 1,
    FILETAG_RCP_BOOTLOADER     = 2,
    FILETAG_RCP_PARTITION_TABLE= 3,
    FILETAG_RCP_FIRMWARE       = 4,
    FILETAG_HOST_FIRMWARE      = 5,
    FILETAG_IMAGE_HEADER       = 0xff,
} esp_rcp_filetag_t;

struct esp_rcp_subfile_info {
    uint32_t tag;
    uint32_t size;
    uint32_t offset;
} __attribute__((packed));
typedef struct esp_rcp_subfile_info esp_rcp_subfile_info_t;

#define ESP_RCP_IMAGE_FILENAME "rcp_image"
```

## 4. BR HTTP OTA — `components/esp_br_http_ota/include/esp_br_http_ota.h`

```c
#include "esp_http_client.h"

esp_err_t esp_br_http_ota(esp_http_client_config_t *http_config);
#define OTA_MAX_WRITE_SIZE 16
```

## 5. Web Server — `components/esp_ot_br_server/include/esp_br_web.h`

```c
void esp_br_web_start(char *base_path);   // base_path = web SPIFFS 挂载点，如 "/spiffs"
```

## 6. SoftAP 配网 — `components/esp_ot_br_server/include/esp_br_wifi_config.h`

```c
esp_err_t esp_br_wifi_config_start(void);
esp_err_t esp_br_wifi_config_get_configured_wifi(char *ssid, size_t ssid_len,
                                                 char *password, size_t password_len,
                                                 uint32_t timeout_ms);  // 0 = 永久等待
esp_err_t esp_br_wifi_config_stop(void);
bool esp_br_wifi_config_is_active(void);
esp_err_t esp_br_wifi_config_get_softap_info(char *ssid, size_t ssid_len,
                                             char *ip_addr, size_t ip_addr_len);
```

## 7. OpenThread RCP 事件处理 — `components/esp_rcp_update/include/esp_ot_rcp_update.h`

```c
void esp_ot_try_update_rcp(const char *running_rcp_version);  // NULL 表示强制更新
void esp_ot_register_rcp_handler(void);
void esp_ot_update_rcp_if_different(void);
```

## 8. CLI 扩展注册 — `components/esp_ot_cli_extension/include/esp_ot_cli_extension.h`

```c
typedef enum {
    WIFI_ADDRESS_EVENT_ADD_IP6,
    WIFI_ADDRESS_EVENT_REMOVE_IP6,
    WIFI_ADDRESS_EVENT_MULTICAST_GROUP_JOIN,
    WIFI_ADDRESS_EVENT_MULTICAST_GROUP_LEAVE,
} esp_wifi_address_event_t;

void esp_cli_custom_command_init(void);
#define OT_EXT_CLI_TAG "ot_ext_cli"
```

注册的命令（`components/esp_ot_cli_extension/src/esp_ot_cli_extension.c`）：

| 命令 | 处理函数 | 启用宏 |
|---|---|---|
| `curl` | `esp_openthread_process_curl` | 默认 |
| `dns64server` | `esp_openthread_process_dns64_server` | `CONFIG_OPENTHREAD_DNS64_CLIENT` |
| `heapdiag` | `esp_ot_process_heap_diag` | 默认 |
| `ip` | `esp_ot_process_ip` | 默认 |
| `loglevel` | `esp_ot_process_logset` | 默认 |
| `mcast` | `esp_ot_process_mcast_group` | 默认 |
| `nvsdiag` | `esp_ot_process_nvs_diag` | `CONFIG_OPENTHREAD_NVS_DIAG` |
| `ota` | `esp_openthread_process_ota_command` | `CONFIG_OPENTHREAD_CLI_OTA` |
| `otrcp` | `esp_openthread_process_rcp_command` | `CONFIG_OPENTHREAD_RCP_COMMAND` |
| `tcpsockclient` / `tcpsockserver` | `esp_ot_process_tcp_client` / `_server` | 默认 |
| `udpsockclient` / `udpsockserver` | `esp_ot_process_udp_client` / `_server` | 默认 |
| `wifi` | `esp_ot_process_wifi_cmd` | `CONFIG_OPENTHREAD_CLI_WIFI` |
| `brlibcheck` | `esp_openthread_process_br_lib_compatibility_check` | `CONFIG_OPENTHREAD_BR_LIB_CHECK` |

## 9. CLI 扩展子模块头

### `esp_ot_wifi_cmd.h`
```c
typedef enum { OT_WIFI_DISCONNECTED, OT_WIFI_CONNECTED, OT_WIFI_RECONNECTING } esp_ot_wifi_state_t;

otError esp_ot_process_wifi_cmd(void *aContext, uint8_t aArgsLength, char *aArgs[]);
void esp_ot_wifi_border_router_init_flag_set(bool initialized);
esp_err_t esp_ot_wifi_connect(const char *ssid, const char *password);
esp_err_t esp_ot_wifi_disconnect(void);
esp_err_t esp_ot_wifi_config_init(void);
esp_err_t esp_ot_wifi_config_set_ssid(const char *ssid);
esp_err_t esp_ot_wifi_config_get_ssid(char *ssid);
esp_err_t esp_ot_wifi_config_set_password(const char *password);
esp_err_t esp_ot_wifi_config_get_password(char *password);
esp_err_t esp_ot_wifi_config_clear(void);
esp_ot_wifi_state_t esp_ot_wifi_state_get(void);
```

### `esp_ot_ota_commands.h`
```c
otError esp_openthread_process_ota_command(void *aContext, uint8_t aArgsLength, char *aArgs[]);
void esp_set_ota_server_cert(const char *cert);
```

### `esp_ot_rcp_commands.h`
```c
otError esp_openthread_process_rcp_command(void *aContext, uint8_t aArgsLength, char *aArgs[]);
```

### `esp_ot_dns64.h`
```c
otError esp_openthread_process_dns64_server(void *aContext, uint8_t aArgsLength, char *aArgs[]);
```

### `esp_ot_curl.h`
```c
otError esp_openthread_process_curl(void *aContext, uint8_t aArgsLength, char *aArgs[]);
```

### `esp_ot_ip.h`
```c
typedef enum { UNICAST_DEL, UNICAST_ADD, MULTICAST_DEL, MULTICAST_ADD } action_type;
otError esp_ot_process_ip(void *aContext, uint8_t aArgsLength, char *aArgs[]);
```

### `esp_ot_udp_socket.h`
```c
otError esp_ot_process_mcast_group(void *aContext, uint8_t aArgsLength, char *aArgs[]);
otError esp_ot_process_udp_server(void *aContext, uint8_t aArgsLength, char *aArgs[]);
otError esp_ot_process_udp_client(void *aContext, uint8_t aArgsLength, char *aArgs[]);

esp_err_t socket_get_netif_impl_name(char *name_input, struct ifreq *ifr);
esp_err_t socket_bind_interface(int sock, struct ifreq *ifr);
```

### `esp_ot_tcp_socket.h`
```c
otError esp_ot_process_tcp_server(void *aContext, uint8_t aArgsLength, char *aArgs[]);
otError esp_ot_process_tcp_client(void *aContext, uint8_t aArgsLength, char *aArgs[]);
```

### `esp_ot_heap_diag.h`
```c
otError esp_ot_process_heap_diag(void *aContext, uint8_t aArgsLength, char *aArgs[]);
esp_err_t esp_ot_heap_diag_init(void);
```

### `esp_ot_loglevel.h`
```c
otError esp_ot_process_logset(void *aContext, uint8_t aArgsLength, char *aArgs[]);
```

## 10. 启动封装 — `examples/common/thread_border_router/include/border_router_launch.h`

```c
void launch_openthread_border_router(const esp_openthread_config_t *config,
                                     const esp_rcp_update_config_t *update_config);
```

> 该函数完成：`ot_console_start` → 外部共存 init（可选）→ `esp_rcp_update_init` + `esp_ot_register_rcp_handler`（AUTO_UPDATE_RCP）→ `esp_openthread_start` → `esp_ot_update_rcp_if_different` → `esp_cli_custom_command_init` + `ot_register_external_commands` → `xTaskCreate(ot_br_init)`（AUTO_START）。

## 11. RF 外部共存 — ESP-IDF `esp_coexist.h` + `ot_external_coexist_init`

BR 启动期由 `border_router_launch.c` 在 `CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE` 时自动调用：

```c
// esp-idf/examples/openthread/ot_common_components/ot_examples_common/include/ot_examples_common.h
void ot_external_coexist_init(void);
```

`ot_external_coexist_init()` 内部使用以下 ESP-IDF API（`esp-idf/components/esp_coex/include/esp_coexist.h`）：

```c
#define EXTERNAL_COEXIST_WIRE_1 0   // request
#define EXTERNAL_COEXIST_WIRE_2 1   // request + grant
#define EXTERNAL_COEXIST_WIRE_3 2   // request + priority + grant
#define EXTERNAL_COEXIST_WIRE_4 3   // 3-wire + tx_line

typedef enum {
    EXTERN_COEX_WIRE_1 = EXTERNAL_COEXIST_WIRE_1,
    EXTERN_COEX_WIRE_2 = EXTERNAL_COEXIST_WIRE_2,
    EXTERN_COEX_WIRE_3 = EXTERNAL_COEXIST_WIRE_3,
    EXTERN_COEX_WIRE_4 = EXTERNAL_COEXIST_WIRE_4,
    EXTERN_COEX_WIRE_NUM,
} external_coex_wire_t;

typedef struct {
    gpio_num_t request;    // follower → leader
    gpio_num_t priority;   // follower → leader
    gpio_num_t grant;      // leader → follower
    gpio_num_t tx_line;    // leader → follower（4-wire 才有）
} esp_external_coex_gpio_set_t;

typedef enum {
    EXTERNAL_COEX_LEADER_ROLE = 0,
    EXTERNAL_COEX_FOLLOWER_ROLE = 2,
    EXTERNAL_COEX_UNKNOWN_ROLE,
} esp_extern_coex_work_mode_t;

esp_err_t esp_external_coex_set_work_mode(esp_extern_coex_work_mode_t work_mode);
esp_err_t esp_enable_extern_coex_gpio_pin(external_coex_wire_t wire_type,
                                          esp_external_coex_gpio_set_t gpio_pin);
```

相关 Kconfig（`ot_examples_common/Kconfig.projbuild`）：`CONFIG_EXTERNAL_COEX_WIRE_TYPE`（0~3）、`CONFIG_EXTERNAL_COEX_REQUEST_PIN/GRANT_PIN/PRIORITY_PIN/TX_LINE_PIN`、`CONFIG_EXTERNAL_COEX_WORK_MODE_LEADER/FOLLOWER/UNKNOWN`。官方 BR 板：ESP32-S3 = Leader，ESP32-H2 = Follower。

## 12. Credential Sharing — OpenThread Border Agent ePSKc（栈 API，见 https://openthread.io/reference）

由 `examples/common/thread_border_router_m5stack/src/br_m5stack_epskc_page.c` 实际调用：

```c
// openthread/border_agent.h
otError otBorderAgentEphemeralKeyStart(otInstance *aInstance, const char *aKeyString,
                                       uint32_t aTimerMilli, uint16_t aPort);
otError otBorderAgentEphemeralKeyStop(otInstance *aInstance);

// openthread/verhoeff_checksum.h
otError otVerhoeffChecksumCalculate(const char *aDigits, char *aChecksum);
```

meshcop-e 服务事件回调注册（ESP-IDF `esp_openthread_netif_glue.h`）：

```c
void esp_openthread_register_meshcop_e_handler(esp_event_handler_t handler, bool for_publish);
// for_publish=true → publish 事件；false → remove 事件
```

M5Stack 专用 Kconfig（`thread_border_router_m5stack/Kconfig.projbuild`）：
- `CONFIG_OPENTHREAD_EPHEMERALKEY_LIFE_TIME`（默认 100 秒）
- `CONFIG_OPENTHREAD_EPHEMERALKEY_PORT`（默认 49180）

Web GUI 凭据 REST 端点（`components/esp_ot_br_server/`）：
- `POST /commission` — body `{"pskd":"..."}`，内部 `otCommissionerStart` → `otCommissionerAddJoiner(ins, NULL, pskd, 120)`
- `POST /join_network` — `credentialType` ∈ {`networkKeyType`, `pskdType`}（常量见 `esp_br_web_base.h`：`CREDENTIAL_TYPE_NETWORK_KEY` / `CREDENTIAL_TYPE_PSKD`）

## 13. DHCPv6 PD — OpenThread Border Router CLI（栈内置，见 https://openthread.io/reference）

DHCPv6 Prefix Delegation 由 OpenThread Border Routing 模块实现，BR 默认编译已含，运行时通过 CLI 控制（无需额外 Kconfig）：

| CLI | 作用 |
|---|---|
| `ot br pd enable` | 启用 DHCPv6 PD 客户端，开始向 DHCPv6 服务器请求前缀 |
| `ot br pd disable` | 关闭 PD 客户端 |
| `ot br pd state` | 查询状态（`running` 表示已拿到并下发前缀） |
| `ot br pd omrprefix` | 查询下发的 On-Mesh Route Prefix（`<prefix>/<len> lifetime:.. preferred:..`） |
| `ot br pd onlinkprefix` | 查询通告的 on-link prefix |

> DHCPv6 服务器侧（Kea）配置见 `docs/en/codelab/dhcpv6_pd.rst`（3.8）。
