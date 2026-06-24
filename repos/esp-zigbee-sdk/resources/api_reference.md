# ESP Zigbee SDK v2.x API 速查

> 全部签名取自 `components/esp-zigbee-lib/include/`（v2.x 主线 `ezbee/` 头文件）。前缀 `ezb_`。仅列出常用 API；完整定义见对应头文件。
> 平台层 API（`esp_zigbee_*`）在 `esp_zigbee.h`；协议栈核心 API（`ezb_*`）在 `ezbee/` 各头。

## 平台层（`esp_zigbee.h`）

```c
// 初始化/启动/主循环
esp_err_t esp_zigbee_init(const esp_zigbee_config_t *config);
esp_err_t esp_zigbee_deinit(void);
esp_err_t esp_zigbee_start(bool autostart);     // false=no-autostart(推荐)
bool      esp_zigbee_is_started(void);
esp_err_t esp_zigbee_launch_mainloop(void);     // 阻塞，不返回

// 线程安全锁（调用任何 ezb_ API 前后，回调内除外）
bool      esp_zigbee_lock_acquire(TickType_t block_ticks);
void      esp_zigbee_lock_release(void);

// 任务队列投递（投递的回调内免锁）
esp_err_t esp_zigbee_task_queue_post(esp_zigbee_callback_t cb, void *ctx);

// 错误转换 / 出厂复位
esp_err_t esp_zigbee_err_to_esp(ezb_err_t error);
void      esp_zigbee_factory_reset(void) __attribute__((__noreturn__));
```

### 配置结构（`esp_zigbee.h`）
```c
typedef enum {
    ESP_ZIGBEE_RADIO_MODE_NATIVE   = 0x0,
    ESP_ZIGBEE_RADIO_MODE_UART_RCP = 0x1,
} esp_zigbee_radio_mode_t;

typedef struct {
    uart_port_t   port;
    uart_config_t uart_config;
    gpio_num_t    rx_pin, tx_pin;
} esp_zigbee_uart_config_t;

typedef struct {
    esp_zigbee_radio_mode_t radio_mode;
    union { esp_zigbee_uart_config_t radio_uart_config; };
} esp_zigbee_radio_config_t;

typedef struct {
    const char             *storage_partition_name;   // 通常 "zb_storage"
    esp_zigbee_radio_config_t radio_config;
} esp_zigbee_platform_config_t;

struct esp_zigbee_zczr_config_s { uint8_t max_children; };
struct esp_zigbee_zed_config_s  { uint8_t ed_timeout; uint32_t keep_alive; }; // ms

typedef struct {
    ezb_nwk_device_type_t device_type;        // EZB_NWK_DEVICE_TYPE_*
    bool install_code_policy;
    union {
        struct esp_zigbee_zczr_config_s zczr_config;
        struct esp_zigbee_zed_config_s  zed_config;
    };
} esp_zigbee_device_config_t;

typedef struct {
    esp_zigbee_device_config_t   device_config;
    esp_zigbee_platform_config_t platform_config;
} esp_zigbee_config_t;
```

## Core（`ezbee/core.h`）

```c
ezb_err_t ezb_core_init(void);
void      ezb_core_deinit(void);
ezb_err_t ezb_dev_start(bool autostart);
bool      ezb_dev_is_started(void);

// 地址
void ezb_set_extended_address(const ezb_extaddr_t *extaddr);
void ezb_get_extended_address(ezb_extaddr_t *extaddr);
void ezb_set_short_address(ezb_shortaddr_t short_addr);
ezb_shortaddr_t ezb_get_short_address(void);

// 网络 ID / 信道 / 功率
void ezb_set_use_extended_panid(const ezb_extpanid_t *extpanid);
void ezb_get_use_extended_panid(ezb_extpanid_t *extpanid);
void ezb_get_extended_panid(ezb_extpanid_t *extpanid);
void      ezb_set_panid(ezb_panid_t pan_id);
ezb_panid_t ezb_get_panid(void);
uint8_t    ezb_get_current_channel(void);
void       ezb_set_tx_power(int8_t power);
void       ezb_get_tx_power(int8_t *power);
ezb_err_t  ezb_set_channel_mask(uint32_t channel_mask);
uint32_t   ezb_get_channel_mask(void);
void       ezb_set_rx_on_when_idle(bool rx_on);
bool       ezb_get_rx_on_when_idle(void);

// 内存配置
ezb_err_t ezb_config_memory(const ezb_mem_config_t *mem_cfg);
```

## App Signals（`ezbee/app_signals.h`）

```c
typedef void *ezb_app_signal_t;
typedef bool (*ezb_app_signal_handler_t)(const ezb_app_signal_t *app_signal);

ezb_err_t              ezb_app_signal_add_handler(ezb_app_signal_handler_t handler);
void                   ezb_app_signal_remove_handler(ezb_app_signal_handler_t handler);
ezb_app_signal_type_t  ezb_app_signal_get_type(const ezb_app_signal_t *signal);
const void            *ezb_app_signal_get_params(const ezb_app_signal_t *signal);
const char            *ezb_app_signal_to_string(ezb_app_signal_type_t signal);
```

### 主要信号（节选）
- ZDO: `EZB_ZDO_SIGNAL_DEFAULT_START`, `EZB_ZDO_SIGNAL_SKIP_STARTUP`, `EZB_ZDO_SIGNAL_ERROR`, `EZB_ZDO_SIGNAL_LEAVE`, `EZB_ZDO_SIGNAL_LEAVE_INDICATION`, `EZB_ZDO_SIGNAL_DEVICE_ANNCE`, `EZB_ZDO_SIGNAL_DEVICE_UNAVAILABLE`, `EZB_ZDO_SIGNAL_DEVICE_UPDATE`, `EZB_ZDO_SIGNAL_DEVICE_AUTHORIZED`
- BDB: `EZB_BDB_SIGNAL_DEVICE_FIRST_START`, `EZB_BDB_SIGNAL_DEVICE_REBOOT`, `EZB_BDB_SIGNAL_STEERING`, `EZB_BDB_SIGNAL_FORMATION`, `EZB_BDB_SIGNAL_FINDING_AND_BINDING_INITIATOR_FINISHED`, `EZB_BDB_SIGNAL_FINDING_AND_BINDING_TARGET_FINISHED`, `EZB_BDB_SIGNAL_TOUCHLINK_INITIATOR_FINISHED`, `EZB_BDB_SIGNAL_TOUCHLINK_TARGET_FINISHED`
- NWK: `EZB_NWK_SIGNAL_DEVICE_ASSOCIATED`, `EZB_NWK_SIGNAL_NO_ACTIVE_LINKS_LEFT`, `EZB_NWK_SIGNAL_PANID_CONFLICT_DETECTED`, `EZB_NWK_SIGNAL_NETWORK_STATUS`, `EZB_NWK_SIGNAL_PERMIT_JOIN_STATUS`

### 载荷结构
- `ezb_bdb_signal_simple_params_t { uint8_t status; }`
- `ezb_zdo_signal_device_annce_params_t { ezb_shortaddr_t short_addr; ezb_extaddr_t device_addr; uint8_t capability; }`
- `ezb_zdo_signal_leave_params_t { ezb_zdo_leave_type_t leave_type; }`（`EZB_ZDO_LEAVE_TYPE_RESET/REJOIN`）
- `ezb_zdo_signal_leave_indication_params_t { short_addr; device_addr; leave_type; }`
- `ezb_nwk_signal_permit_join_status_params_t { uint8_t duration; }`

## BDB（`ezbee/bdb.h`）

```c
ezb_err_t ezb_bdb_set_primary_channel_set(uint32_t channel_mask);
uint32_t  ezb_bdb_get_primary_channel_set(void);
ezb_err_t ezb_bdb_set_secondary_channel_set(uint32_t channel_mask);
uint32_t  ezb_bdb_get_secondary_channel_set(void);
void      ezb_bdb_set_scan_duration(uint8_t duration);
bool      ezb_bdb_dev_joined(void);
void      ezb_bdb_set_commissioning_mode(ezb_bdb_comm_mode_mask_t commissioning_mode);
ezb_bdb_comm_status_t ezb_bdb_get_commissioning_status(void);
ezb_err_t ezb_bdb_start_top_level_commissioning(ezb_bdb_comm_mode_mask_t mode_mask);
ezb_err_t ezb_bdb_cancel_steering(void);
ezb_err_t ezb_bdb_cancel_formation(void);
ezb_err_t ezb_bdb_cancel_touchlink_target(void);
ezb_err_t ezb_bdb_open_network(uint8_t permit_duration);   // 秒
ezb_err_t ezb_bdb_close_network(void);
void      ezb_bdb_reset_via_local_action(void);
bool      ezb_bdb_is_factory_new(void);
```

### 模式 / 状态（`ezbee/bdb.h`）
- 模式 `ezb_bdb_comm_mode_t`: `EZB_BDB_MODE_INITIALIZATION`(0x01), `EZB_BDB_MODE_TOUCHLINK_INITIATOR`(0x02), `EZB_BDB_MODE_NETWORK_STEERING`(0x04), `EZB_BDB_MODE_NETWORK_FORMATION`(0x08), `EZB_BDB_MODE_FINDING_N_BINDING`(0x10), `EZB_BDB_MODE_TOUCHLINK_TARGET`(0x20)
- 状态 `EZB_BDB_STATUS_*`: SUCCESS, IN_PROGRESS, NOT_AA_CAPABLE, NO_NETWORK, TARGET_FAILURE, FORMATION_FAILURE, NO_IDENTIFY_QUERY_RESPONSE, BINDING_TABLE_FULL, NO_SCAN_RESPONSE, NOT_PERMITTED, TCLK_EX_FAILURE, NOT_ON_A_NETWORK, ON_A_NETWORK, CANCELLED, DEV_ANNCE_SEND_FAILURE

## AF / 数据模型（`ezbee/af.h`）

```c
ezb_af_device_desc_t ezb_af_create_device_desc(void);
void      ezb_af_free_device_desc(ezb_af_device_desc_t dev_desc);
ezb_err_t ezb_af_device_add_endpoint_desc(ezb_af_device_desc_t dev_desc, ezb_af_ep_desc_t ep_desc);
ezb_af_ep_desc_t ezb_af_device_remove_endpoint_desc(ezb_af_device_desc_t dev_desc, uint8_t ep_id);
ezb_af_ep_desc_t ezb_af_device_get_endpoint_desc(ezb_af_device_desc_t dev_desc, uint8_t ep_id);
ezb_err_t ezb_af_device_desc_register(ezb_af_device_desc_t dev_desc);

ezb_af_ep_desc_t ezb_af_create_endpoint_desc(const ezb_af_ep_config_t *ep_config);
ezb_af_ep_desc_t ezb_af_create_gateway_endpoint(const ezb_af_ep_config_t *ep_config);
ezb_err_t ezb_af_endpoint_add_cluster_desc(ezb_af_ep_desc_t ep_desc, ezb_zcl_cluster_desc_t cluster_desc);
ezb_zcl_cluster_desc_t ezb_af_endpoint_get_cluster_desc(const ezb_af_ep_desc_t ep_desc, uint16_t cluster_id, uint8_t role);
ezb_af_ep_desc_t ezb_af_get_ep_desc(uint8_t ep_id);

// 描述符查询
const ezb_af_node_desc_t       *ezb_af_get_node_desc(void);
const ezb_af_node_power_desc_t *ezb_af_get_node_power_desc(void);
const ezb_af_simple_desc_t     *ezb_af_get_simple_desc(uint8_t ep_id);
void ezb_af_node_desc_set_manuf_code(uint16_t manuf_code);
ezb_err_t ezb_af_set_node_power_desc(const ezb_af_node_power_desc_t *desc);
```

- Profile ID: `EZB_AF_HA_PROFILE_ID`(0x0104), `EZB_AF_ZDP_PROFILE_ID`(0x0000), `EZB_AF_SE_PROFILE_ID`(0x0109), `EZB_AF_TL_PROFILE_ID`(0xC05E), `EZB_AF_GP_PROFILE_ID`(0xA1E0)

## ZHA 设备创建（`ezbee/zha.h`）

```c
ezb_af_ep_desc_t ezb_zha_create_on_off_light(uint8_t ep_id, const ezb_zha_on_off_light_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_dimmable_light(uint8_t ep_id, const ezb_zha_dimmable_light_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_color_dimmable_light(uint8_t ep_id, const ezb_zha_color_dimmable_light_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_on_off_switch(uint8_t ep_id, const ezb_zha_on_off_switch_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_dimmer_switch(uint8_t ep_id, const ezb_zha_dimmer_switch_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_color_dimmer_switch(uint8_t ep_id, const ezb_zha_color_dimmer_switch_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_mains_power_outlet(uint8_t ep_id, const ezb_zha_mains_power_outlet_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_temperature_sensor(uint8_t ep_id, const ezb_zha_temperature_sensor_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_light_sensor(uint8_t ep_id, const ezb_zha_light_sensor_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_thermostat(uint8_t ep_id, const ezb_zha_thermostat_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_configuration_tool(uint8_t ep_id, const ezb_zha_configuration_tool_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_shade(uint8_t ep_id, const ezb_zha_shade_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_shade_controller(uint8_t ep_id, const ezb_zha_shade_controller_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_window_covering(uint8_t ep_id, const ezb_zha_window_covering_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_window_covering_controller(uint8_t ep_id, const ezb_zha_window_covering_controller_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_door_lock(uint8_t ep_id, const ezb_zha_door_lock_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_door_lock_controller(uint8_t ep_id, const ezb_zha_door_lock_controller_config_t *cfg);
ezb_af_ep_desc_t ezb_zha_create_custom_gateway(uint8_t ep_id, const ezb_zha_custom_gateway_config_t *cfg);
```
配套配置宏：`EZB_ZHA_<DEVICE>_CONFIG()`（如 `EZB_ZHA_ON_OFF_LIGHT_CONFIG()`、`EZB_ZHA_TEMPERATURE_SENSOR_CONFIG()`、`EZB_ZHA_CUSTOM_GATEWAY_CONFIG()`）。

## NWK（`ezbee/nwk.h`）

```c
// 设备类型
enum { EZB_NWK_DEVICE_TYPE_COORDINATOR=0x0, EZB_NWK_DEVICE_TYPE_ROUTER=0x1,
       EZB_NWK_DEVICE_TYPE_END_DEVICE=0x2,  EZB_NWK_DEVICE_TYPE_NONE=0x3 };

// ED timeout
enum { EZB_NWK_ED_TIMEOUT_10SEC=0, EZB_NWK_ED_TIMEOUT_2MIN, ..., EZB_NWK_ED_TIMEOUT_16384MIN=14 };

void            ezb_nwk_set_rx_on_when_idle(bool rx_on);
bool            ezb_nwk_get_rx_on_when_idle(void);
void            ezb_nwk_get_extended_address(ezb_extaddr_t *extaddr);
ezb_shortaddr_t ezb_nwk_get_short_address(void);
ezb_panid_t     ezb_nwk_get_panid(void);
uint8_t         ezb_nwk_get_current_channel(void);
void            ezb_nwk_get_extended_panid(ezb_extpanid_t *extpanid);
ezb_err_t       ezb_address_extended_by_short(ezb_shortaddr_t shortaddr, ezb_extaddr_t *extaddr);
```

## APS（`ezbee/aps.h`）

```c
void ezb_aps_secur_enable_distributed_security(bool enable);
bool ezb_aps_secur_is_distributed_security(void);
```

## Security（`ezbee/secur.h`）

```c
// 安全级别
enum { EZB_SECUR_SECLEVEL_NONE=0x00, ..., EZB_SECUR_SECLEVEL_ENC_MIC32=0x05,
       EZB_SECUR_SECLEVEL_ENC_MIC64=0x06, EZB_SECUR_SECLEVEL_ENC_MIC128=0x07 };

// Install code 类型
enum { EZB_SECUR_IC_TYPE_48=0x00, EZB_SECUR_IC_TYPE_64, EZB_SECUR_IC_TYPE_96, EZB_SECUR_IC_TYPE_128 };

ezb_err_t ezb_secur_set_ic_required(bool required);
ezb_err_t ezb_secur_ic_add(const ezb_extaddr_t *address, ezb_secur_ic_type_t ic_type, const uint8_t *ic);
ezb_err_t ezb_secur_ic_remove(const ezb_extaddr_t *address);
ezb_err_t ezb_secur_ic_remove_all(void);
ezb_err_t ezb_secur_ic_set(ezb_secur_ic_type_t ic_type, const uint8_t *ic);
ezb_err_t ezb_secur_ic_get(uint8_t *ic, ezb_secur_ic_type_t *ic_type);
ezb_err_t ezb_secur_set_tclk_exchange_required(bool required);
void      ezb_secur_set_global_link_key(const uint8_t *key);
ezb_err_t ezb_secur_set_security_level(ezb_secur_seclevel_t level);
ezb_secur_seclevel_t ezb_secur_get_security_level(void);
ezb_err_t ezb_secur_set_network_key(const uint8_t *key);
ezb_err_t ezb_secur_get_network_key(uint8_t *key);
```

## ZDO（`ezbee/zdo.h` 及 `ezbee/zdo/`）

```c
// 发现与绑定
ezb_err_t ezb_zdo_match_desc_req(const ezb_zdo_match_desc_req_t *req);  // 回调 ezb_zdo_match_desc_req_result_t
ezb_err_t ezb_zdo_bind_req(const ezb_zdo_bind_req_t *req);              // 回调 ezb_zdp_bind_req_result_t
```
- 状态：`EZB_ZDP_STATUS_SUCCESS` 等
- 结构体字段见 `ezbee/zdo/zdo_type.h`、`zdo_bind_mgmt.h`、`zdo_dev_srv_disc.h`、`zdo_nwk_mgmt.h`

## ZCL 通用（`ezbee/zcl/`）

```c
// 核心 action 回调（zcl_core.h）
typedef void (*ezb_zcl_core_action_callback_t)(ezb_zcl_core_action_callback_id_t callback_id, void *message);
ezb_err_t ezb_zcl_core_action_handler_register(ezb_zcl_core_action_callback_t handler);
// callback_id: EZB_ZCL_CORE_SET_ATTR_VALUE_CB_ID, _DEFAULT_RSP_CB_ID, _REPORT_ATTR_CB_ID, ...

// 属性操作
ezb_err_t ezb_zcl_set_attr_value(uint8_t ep_id, uint16_t cluster_id, uint8_t cluster_role,
                                 uint16_t attr_id, uint16_t manuf_code,
                                 uint8_t *value, bool force);
ezb_zcl_attr_desc_t *ezb_zcl_get_attr_desc(uint8_t ep_id, uint16_t cluster_id, uint8_t cluster_role,
                                           uint16_t attr_id, uint16_t manuf_code);
void ezb_zcl_attr_desc_get_value(ezb_zcl_attr_desc_t *desc, void *value);

// 通用命令（zcl_general_cmd.h）
ezb_err_t ezb_zcl_read_attr_cmd_req(const ezb_zcl_read_attr_cmd_t *cmd_req);
ezb_err_t ezb_zcl_write_attr_cmd_req(const ezb_zcl_write_attr_cmd_t *cmd_req);
ezb_err_t ezb_zcl_config_report_cmd_req(const ezb_zcl_config_report_cmd_t *cmd_req);
ezb_err_t ezb_zcl_report_attr_cmd_req(const ezb_zcl_report_attr_cmd_t *cmd_req);

// 上报管理（zcl_reporting.h）
ezb_zcl_reporting_info_t ezb_zcl_reporting_info_find(uint8_t ep_id, uint16_t cluster_id, uint8_t role,
                                                     uint16_t attr_id, uint16_t manuf_code);
ezb_err_t ezb_zcl_reporting_start_attr_report(ezb_zcl_reporting_info_t info);
ezb_err_t ezb_zcl_reporting_stop_attr_report(ezb_zcl_reporting_info_t info);

// 角色 / 方向
// EZB_ZCL_CLUSTER_SERVER / EZB_ZCL_CLUSTER_CLIENT
// EZB_ZCL_CMD_DIRECTION_TO_SRV / EZB_ZCL_CMD_DIRECTION_TO_CLI
```

### 地址模式与构造宏（`ezbee/core_types.h`）
```c
enum { EZB_ADDR_MODE_NONE=0, EZB_ADDR_MODE_GROUP=1, EZB_ADDR_MODE_SHORT=2, EZB_ADDR_MODE_EXT=3 };
EZB_ADDR_SHORT(short_addr)   // 构造 short 地址
EZB_ADDR_EXT(extended_addr)  // 构造 ext 地址
EZB_ADDR_NONE()              // 绑定表寻址
EZB_ADDR_GROUP(group_addr)
```

## ZCL 各 cluster 命令（节选，详见对应头）

```c
// on_off.h
ezb_err_t ezb_zcl_on_off_on_cmd_req(const ezb_zcl_on_off_cmd_t *cmd_req);
ezb_err_t ezb_zcl_on_off_off_cmd_req(const ezb_zcl_on_off_cmd_t *cmd_req);
ezb_err_t ezb_zcl_on_off_toggle_cmd_req(const ezb_zcl_on_off_cmd_t *cmd_req);
ezb_err_t ezb_zcl_on_off_off_with_effect_cmd_req(const ezb_zcl_on_off_off_with_effect_cmd_t *cmd_req);
ezb_err_t ezb_zcl_on_off_on_with_recall_global_scene_cmd_req(const ezb_zcl_on_off_cmd_t *cmd_req);
ezb_err_t ezb_zcl_on_off_on_with_timed_off_cmd_req(const ezb_zcl_on_off_on_with_timed_off_cmd_t *cmd_req);

// level.h
ezb_err_t ezb_zcl_level_move_to_level_cmd_req(const ezb_zcl_level_move_to_level_cmd_t *cmd_req);
ezb_err_t ezb_zcl_level_move_to_level_with_on_off_cmd_req(const ezb_zcl_level_move_to_level_cmd_t *cmd_req);

// color_control.h —— 多个 move/step 命令（hue/sat/color_temp/color_loop），签名以头文件为准

// custom.h
ezb_err_t ezb_zcl_custom_cluster_cmd_req(const ezb_zcl_custom_cluster_cmd_t *cmd_req);
// 注册 handlers：ezb_zcl_custom_cluster_handlers_t（以 custom.h 实际声明为准）
```

### 常用 cluster ID
`EZB_ZCL_CLUSTER_ID_BASIC`(0x0000), `_IDENTIFY`(0x0003), `_GROUPS`(0x0004), `_SCENES`(0x0005), `_ON_OFF`(0x0006), `_LEVEL_CONTROL`(0x0008), `_OTA`(0x0019), `_DOOR_LOCK`(0x0101), `_WINDOW_COVERING`(0x0102), `_THERMOSTAT`(0x0201), `_COLOR_CONTROL`(0x0300), `_TEMPERATURE_MEASUREMENT`(0x0402), `_ILLUMINANCE_MEASUREMENT`(0x0400), `_IAS_ZONE`(0x0500), `_ELECTRICAL_MEASUREMENT`(0x0B04)

## Touchlink（`ezbee/touchlink.h`）

```c
ezb_err_t ezb_touchlink_action_permission_handler_register(ezb_touchlink_action_permission_callback_t cb);
ezb_err_t ezb_touchlink_identify_handler_register(ezb_touchlink_identify_callback_t cb);
ezb_err_t ezb_touchlink_set_master_key(const uint8_t *key);
ezb_err_t ezb_touchlink_get_master_key(uint8_t *key);
// Touchlink 流程通过 ezb_bdb_start_top_level_commissioning(EZB_BDB_MODE_TOUCHLINK_INITIATOR/TARGET)
```

## 版本（`esp_zigbee_version.h`）

```c
#define ESP_ZIGBEE_VER_MAJOR 2
#define ESP_ZIGBEE_VER_MINOR 0
#define ESP_ZIGBEE_VER_PATCH 1
```

## Deep Sleep 电源管理（ESP-IDF `esp_sleep.h`，用于 deep_sleep_end_device）

> Zigbee SDK 本身不实现 deep sleep——由应用直接调用 ESP-IDF 电源 API。deep sleep 唤醒从 reset 重启，协议栈每次重新 init + rejoin。以下为 deep_sleep_end_device 示例实际使用的签名。

```c
// 进入 deep sleep（整芯片断电，不返回；由唤醒源触发 reset 重启）
void        esp_deep_sleep_start(void) __attribute__((__noreturn__);

// 配置唤醒源（app_main 最先调用，须早于任何 Zigbee 调用）
esp_err_t   esp_sleep_enable_timer_wakeup(uint64_t time_in_us);          // RTC 定时器，单位微秒
esp_err_t   esp_sleep_enable_ext1_wakeup(uint64_t mask, esp_ext1_wakeup_mode_t mode);
// mode: ESP_EXT1_WAKEUP_ALL_LOW / ESP_EXT1_WAKEUP_ANY_LOW

// 查询本次唤醒原因（bitmap，多位或运算）
uint32_t    esp_sleep_get_wakeup_causes(void);
// 位定义: ESP_SLEEP_WAKEUP_TIMER, ESP_SLEEP_WAKEUP_EXT1, ESP_SLEEP_WAKEUP_UNDEFINED, ...

// GPIO 唤醒使能（配合 EXT1，light/deep 通用）
esp_err_t   gpio_wakeup_enable(gpio_num_t gpio_num, gpio_int_type_t intr_type);

// RTC 域 GPIO 上下拉（ESP32-H2 等 SOC_RTCIO_INPUT_OUTPUT_SUPPORTED=1 时）
void        rtc_gpio_pullup_en(gpio_num_t gpio_num);
void        rtc_gpio_pulldown_dis(gpio_num_t gpio_num);
```

### 跨睡眠数据
```c
static RTC_DATA_ATTR struct timeval s_sleep_time;   // 放入 RTC 域，深睡不断电保留
```

### 配套 sdkconfig（deep sleep 关键项）
- `CONFIG_ZB_ENABLED=y` + `CONFIG_ZB_ZED=y`
- `CONFIG_PARTITION_TABLE_CUSTOM=y`（自定义 partitions.csv，含 `zb_storage`/`zb_fct`）
- `CONFIG_NEWLIB_TIME_SYSCALL_USE_RTC_HRT=y`（`gettimeofday` 走 RTC，深睡后时间连续）
- `CONFIG_RTC_CLK_SRC_INT_RC=y`
- `CONFIG_BOOTLOADER_SKIP_VALIDATE_IN_DEEP_SLEEP=y`（唤醒跳过 bootloader flash 校验，加速启动）

> `CONFIG_GPIO_EXT1_WAKEUP_SOURCE` 由 `examples/utils/switch_driver/Kconfig` 定义（ESP32-H2 默认 9、ESP32-C6 默认 7，即 BOOT 按键）。
> deep sleep **不需要** `CONFIG_PM_ENABLE` / `CONFIG_FREERTOS_USE_TICKLESS_IDLE`（那是 light sleep 的配置）。
