# ESP RainMaker API 速查

> 全部签名来自 `components/esp_rainmaker/include/*.h` 真实头文件。按模块分组。

## Core（`esp_rmaker_core.h`）

### 节点
```c
esp_rmaker_node_t *esp_rmaker_node_init(const esp_rmaker_config_t *config, const char *name, const char *type);
esp_err_t esp_rmaker_start(void);
esp_err_t esp_rmaker_stop(void);
esp_err_t esp_rmaker_node_deinit(const esp_rmaker_node_t *node);
esp_rmaker_state_t esp_rmaker_get_state(void);  /* DEINIT/INIT_DONE/STARTING/CONFIG_REPORTED/STARTED/STOP_REQUESTED */
const esp_rmaker_node_t *esp_rmaker_get_node(void);
char *esp_rmaker_get_node_id(void);
esp_rmaker_node_info_t *esp_rmaker_node_get_info(const esp_rmaker_node_t *node);

esp_err_t esp_rmaker_node_add_attribute(const esp_rmaker_node_t *node, const char *attr_name, const char *val);
esp_err_t esp_rmaker_node_edit_attribute(const esp_rmaker_node_t *node, const char *attr_name, const char *val);
esp_err_t esp_rmaker_node_add_fw_version(const esp_rmaker_node_t *node, const char *fw_version);
esp_err_t esp_rmaker_node_add_model(const esp_rmaker_node_t *node, const char *model);
esp_err_t esp_rmaker_node_add_subtype(const esp_rmaker_node_t *node, const char *subtype);
esp_err_t esp_rmaker_node_add_readme(const esp_rmaker_node_t *node, const char *readme);
```

### 设备/服务
```c
esp_rmaker_device_t *esp_rmaker_device_create(const char *dev_name, const char *type, void *priv_data);
esp_rmaker_device_t *esp_rmaker_service_create(const char *serv_name, const char *type, void *priv_data);
esp_err_t esp_rmaker_device_delete(const esp_rmaker_device_t *device);

esp_err_t esp_rmaker_device_add_cb(const esp_rmaker_device_t *device,
                                   esp_rmaker_device_write_cb_t write_cb,
                                   esp_rmaker_device_read_cb_t read_cb);
esp_err_t esp_rmaker_device_add_bulk_cb(const esp_rmaker_device_t *device,
                                        esp_rmaker_device_bulk_write_cb_t write_cb,
                                        esp_rmaker_device_bulk_read_cb_t read_cb);

esp_err_t esp_rmaker_node_add_device(const esp_rmaker_node_t *node, const esp_rmaker_device_t *device);
esp_err_t esp_rmaker_node_remove_device(const esp_rmaker_node_t *node, const esp_rmaker_device_t *device);
esp_rmaker_device_t *esp_rmaker_node_get_device_by_name(const esp_rmaker_node_t *node, const char *device_name);

esp_err_t esp_rmaker_device_add_attribute(const esp_rmaker_device_t *device, const char *attr_name, const char *val);
esp_err_t esp_rmaker_device_add_subtype(const esp_rmaker_device_t *device, const char *subtype);
esp_err_t esp_rmaker_device_add_model(const esp_rmaker_device_t *device, const char *model);
char *esp_rmaker_device_get_name(const esp_rmaker_device_t *device);
void *esp_rmaker_device_get_priv_data(const esp_rmaker_device_t *device);
char *esp_rmaker_device_get_type(const esp_rmaker_device_t *device);
const char *esp_rmaker_device_cb_src_to_str(esp_rmaker_req_src_t src);
```

### 参数
```c
esp_err_t esp_rmaker_device_add_param(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param);
esp_rmaker_param_t *esp_rmaker_device_get_param_by_type(const esp_rmaker_device_t *device, const char *param_type);
esp_rmaker_param_t *esp_rmaker_device_get_param_by_name(const esp_rmaker_device_t *device, const char *param_name);
esp_err_t esp_rmaker_device_assign_primary_param(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param);

esp_rmaker_param_t *esp_rmaker_param_create(const char *param_name, const char *type,
                                            esp_rmaker_param_val_t val, uint8_t properties);
esp_err_t esp_rmaker_param_add_ui_type(const esp_rmaker_param_t *param, const char *ui_type);
esp_err_t esp_rmaker_param_add_bounds(const esp_rmaker_param_t *param,
                                      esp_rmaker_param_val_t min, esp_rmaker_param_val_t max, esp_rmaker_param_val_t step);
esp_err_t esp_rmaker_param_add_simple_time_series_ttl(const esp_rmaker_param_t *param, uint16_t ttl_days);
esp_err_t esp_rmaker_param_add_valid_str_list(const esp_rmaker_param_t *param, const char *strs[], uint8_t count);
esp_err_t esp_rmaker_param_add_array_max_count(const esp_rmaker_param_t *param, int count);

esp_err_t esp_rmaker_param_update(const esp_rmaker_param_t *param, esp_rmaker_param_val_t val);
esp_err_t esp_rmaker_report_updated_params(void);
esp_err_t esp_rmaker_param_update_and_report(const esp_rmaker_param_t *param, esp_rmaker_param_val_t val);
esp_err_t esp_rmaker_param_update_and_notify(const esp_rmaker_param_t *param, esp_rmaker_param_val_t val);
esp_err_t esp_rmaker_raise_alert(const char *alert_str);   /* 最大 ESP_RMAKER_MAX_ALERT_LEN(100) */
esp_err_t esp_rmaker_param_report_simple_ts_data(const esp_rmaker_param_t *param, esp_rmaker_param_val_t val, int timestamp, uint16_t ttl_days);

char *esp_rmaker_param_get_name(const esp_rmaker_param_t *param);
char *esp_rmaker_param_get_type(const esp_rmaker_param_t *param);
esp_rmaker_param_val_t *esp_rmaker_param_get_val(esp_rmaker_param_t *param);
```

### 值类型构造
```c
esp_rmaker_param_val_t esp_rmaker_bool(bool bval);
esp_rmaker_param_val_t esp_rmaker_int(int ival);
esp_rmaker_param_val_t esp_rmaker_float(float fval);
esp_rmaker_param_val_t esp_rmaker_str(const char *sval);
esp_rmaker_param_val_t esp_rmaker_obj(const char *val);
esp_rmaker_param_val_t esp_rmaker_array(const char *val);
```

### 节点上报 / 内置服务启用
```c
esp_err_t esp_rmaker_report_node_details(void);          /* start 后动态增删设备时用 */
esp_err_t esp_rmaker_timezone_service_enable(void);
esp_err_t esp_rmaker_system_service_enable(esp_rmaker_system_serv_config_t *config);
esp_err_t esp_rmaker_ota_enable_default(void);
```

### 本地控制
```c
bool     esp_rmaker_local_ctrl_service_started(void);
esp_err_t esp_rmaker_local_ctrl_enable(void);
esp_err_t esp_rmaker_local_ctrl_disable(void);
esp_err_t esp_rmaker_local_ctrl_set_pop(const char *pop);
/* 仅 CONFIG_ESP_RMAKER_LOCAL_CTRL_CHAL_RESP_ENABLE 时 */
esp_err_t esp_rmaker_local_ctrl_enable_chal_resp(const char *instance_name);
esp_err_t esp_rmaker_local_ctrl_disable_chal_resp(void);
```

### 直发 / 分组 / AWS 凭据
```c
esp_err_t esp_rmaker_publish_direct(const char *message);
esp_err_t esp_rmaker_store_group_id(const char *group_id);
esp_err_t esp_rmaker_get_stored_group_id(char **group_id);
esp_err_t esp_rmaker_node_auth_sign_msg(const void *challenge, size_t inlen, void **response, size_t *outlen);
char* esp_rmaker_get_aws_region(void);
esp_rmaker_aws_credentials_t* esp_rmaker_get_aws_security_token(const char *role_alias);
void   esp_rmaker_free_aws_credentials(esp_rmaker_aws_credentials_t *credentials);
```

## 关键结构/枚举（`esp_rmaker_core.h`）

```c
typedef struct {
    bool enable_time_sync;
} esp_rmaker_config_t;

typedef enum { RMAKER_VAL_TYPE_INVALID, RMAKER_VAL_TYPE_BOOLEAN, RMAKER_VAL_TYPE_INTEGER,
               RMAKER_VAL_TYPE_FLOAT, RMAKER_VAL_TYPE_STRING, RMAKER_VAL_TYPE_OBJECT,
               RMAKER_VAL_TYPE_ARRAY } esp_rmaker_val_type_t;

typedef union { bool b; int i; float f; char *s; } esp_rmaker_val_t;
typedef struct { esp_rmaker_val_type_t type; esp_rmaker_val_t val; } esp_rmaker_param_val_t;

typedef enum {
    PROP_FLAG_WRITE = (1<<0), PROP_FLAG_READ = (1<<1), PROP_FLAG_TIME_SERIES = (1<<2),
    PROP_FLAG_PERSIST = (1<<3), PROP_FLAG_SIMPLE_TIME_SERIES = (1<<4)
} esp_param_property_flags_t;

typedef enum { ESP_RMAKER_STATE_DEINIT=0, ESP_RMAKER_STATE_INIT_DONE,
               ESP_RMAKER_STATE_STARTING, ESP_RMAKER_STATE_CONFIG_REPORTED,
               ESP_RMAKER_STATE_STARTED, ESP_RMAKER_STATE_STOP_REQUESTED } esp_rmaker_state_t;

typedef enum { ESP_RMAKER_REQ_SRC_INIT, ESP_RMAKER_REQ_SRC_CLOUD, ESP_RMAKER_REQ_SRC_SCHEDULE,
               ESP_RMAKER_REQ_SRC_SCENE_ACTIVATE, ESP_RMAKER_REQ_SRC_SCENE_DEACTIVATE,
               ESP_RMAKER_REQ_SRC_LOCAL, ESP_RMAKER_REQ_SRC_CMD_RESP, ESP_RMAKER_REQ_SRC_FIRMWARE,
               ESP_RMAKER_REQ_SRC_BLE_LOCAL, ESP_RMAKER_REQ_SRC_MAX } esp_rmaker_req_src_t;

/* 回调原型 */
typedef esp_err_t (*esp_rmaker_device_write_cb_t)(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
        const esp_rmaker_param_val_t val, void *priv_data, esp_rmaker_write_ctx_t *ctx);
typedef esp_err_t (*esp_rmaker_device_read_cb_t)(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
        void *priv_data, esp_rmaker_read_ctx_t *ctx);
typedef esp_err_t (*esp_rmaker_device_bulk_write_cb_t)(const esp_rmaker_device_t *device,
        const esp_rmaker_param_write_req_t write_req[], uint8_t count, void *priv_data, esp_rmaker_write_ctx_t *ctx);
typedef esp_err_t (*esp_rmaker_device_bulk_read_cb_t)(const esp_rmaker_device_t *device,
        const esp_rmaker_param_t *params[], uint8_t count, void *priv_data, esp_rmaker_read_ctx_t *ctx);

#define ESP_RMAKER_CONFIG_VERSION "2020-03-20"
#define ESP_RMAKER_MAX_ALERT_LEN 100
#define SYSTEM_SERV_FLAG_REBOOT (1<<0)
#define SYSTEM_SERV_FLAG_FACTORY_RESET (1<<1)
#define SYSTEM_SERV_FLAG_WIFI_RESET (1<<2)
#define SYSTEM_SERV_FLAGS_ALL  (SYSTEM_SERV_FLAG_REBOOT|SYSTEM_SERV_FLAG_FACTORY_RESET|SYSTEM_SERV_FLAG_WIFI_RESET)
```

## 事件（`RMAKER_EVENT` / `RMAKER_COMMON_EVENT`）
```c
/* RMAKER_EVENT */
RMAKER_EVENT_INIT_DONE=1, RMAKER_EVENT_CLAIM_STARTED, RMAKER_EVENT_CLAIM_SUCCESSFUL,
RMAKER_EVENT_CLAIM_FAILED, RMAKER_EVENT_USER_NODE_MAPPING_DONE, RMAKER_EVENT_LOCAL_CTRL_STARTED,
RMAKER_EVENT_USER_NODE_MAPPING_RESET, RMAKER_EVENT_LOCAL_CTRL_STOPPED,
RMAKER_EVENT_STARTED, RMAKER_EVENT_CONFIG_REPORTED

/* RMAKER_COMMON_EVENT（来自 esp_rmaker_common_events.h，见 examples） */
RMAKER_EVENT_REBOOT, RMAKER_EVENT_WIFI_RESET, RMAKER_EVENT_FACTORY_RESET,
RMAKER_MQTT_EVENT_CONNECTED, RMAKER_MQTT_EVENT_DISCONNECTED, RMAKER_MQTT_EVENT_PUBLISHED
```

## 标准参数 helper（`esp_rmaker_standard_params.h`）
```c
esp_rmaker_param_t *esp_rmaker_name_param_create(const char *param_name, const char *val);
esp_rmaker_param_t *esp_rmaker_power_param_create(const char *param_name, bool val);
esp_rmaker_param_t *esp_rmaker_brightness_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_hue_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_saturation_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_intensity_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_cct_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_direction_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_speed_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_temperature_param_create(const char *param_name, float val);
esp_rmaker_param_t *esp_rmaker_ota_status_param_create(const char *param_name);
esp_rmaker_param_t *esp_rmaker_ota_info_param_create(const char *param_name);
esp_rmaker_param_t *esp_rmaker_ota_url_param_create(const char *param_name);
esp_rmaker_param_t *esp_rmaker_timezone_param_create(const char *param_name, const char *val);
esp_rmaker_param_t *esp_rmaker_timezone_posix_param_create(const char *param_name, const char *val);
esp_rmaker_param_t *esp_rmaker_timestamp_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_schedules_param_create(const char *param_name, int max_schedules);
esp_rmaker_param_t *esp_rmaker_scenes_param_create(const char *param_name, int max_scenes);
esp_rmaker_param_t *esp_rmaker_reboot_param_create(const char *param_name);
esp_rmaker_param_t *esp_rmaker_factory_reset_param_create(const char *param_name);
esp_rmaker_param_t *esp_rmaker_wifi_reset_param_create(const char *param_name);
esp_rmaker_param_t *esp_rmaker_local_control_pop_param_create(const char *param_name, const char *val);
esp_rmaker_param_t *esp_rmaker_local_control_type_param_create(const char *param_name, int val);
esp_rmaker_param_t *esp_rmaker_user_token_param_create(const char *param_name);
esp_rmaker_param_t *esp_rmaker_base_url_param_create(const char *param_name);
esp_rmaker_param_t *esp_rmaker_group_id_param_create(const char *param_name, const char *val);
esp_rmaker_param_t *esp_rmaker_user_token_status_param_create(const char *param_name, int val);
```

## 标准设备 helper（`esp_rmaker_standard_devices.h`）
```c
esp_rmaker_device_t *esp_rmaker_switch_device_create(const char *dev_name, void *priv_data, bool power);
esp_rmaker_device_t *esp_rmaker_lightbulb_device_create(const char *dev_name, void *priv_data, bool power);
esp_rmaker_device_t *esp_rmaker_fan_device_create(const char *dev_name, void *priv_data, bool power);
esp_rmaker_device_t *esp_rmaker_temp_sensor_device_create(const char *dev_name, void *priv_data, float temperature);
```

## 标准服务 helper（`esp_rmaker_standard_services.h`）
```c
esp_rmaker_device_t *esp_rmaker_ota_service_create(const char *serv_name, void *priv_data);
esp_rmaker_device_t *esp_rmaker_time_service_create(const char *serv_name, const char *timezone,
                                                    const char *timezone_posix, void *priv_data);
esp_rmaker_device_t *esp_rmaker_create_schedule_service(const char *serv_name,
                                                    esp_rmaker_device_write_cb_t write_cb,
                                                    esp_rmaker_device_read_cb_t read_cb,
                                                    int max_schedules, void *priv_data);
esp_rmaker_device_t *esp_rmaker_create_scenes_service(const char *serv_name,
                                                    esp_rmaker_device_write_cb_t write_cb,
                                                    esp_rmaker_device_read_cb_t read_cb,
                                                    int max_scenes, bool deactivation_support, void *priv_data);
esp_rmaker_device_t *esp_rmaker_create_system_service(const char *serv_name, void *priv_data);
esp_rmaker_device_t *esp_rmaker_create_local_control_service(const char *serv_name, const char *pop,
                                                    int sec_type, void *priv_data);
esp_rmaker_device_t *esp_rmaker_create_user_auth_service(const char *serv_name,
                                                    esp_rmaker_device_bulk_write_cb_t bulk_write_cb,
                                                    esp_rmaker_device_bulk_read_cb_t read_cb, void *priv_data);
esp_rmaker_device_t *esp_rmaker_create_groups_service(const char *serv_name,
                                                    esp_rmaker_device_bulk_write_cb_t write_cb,
                                                    char *group_id, void *priv_data);
```

## OTA（`esp_rmaker_ota.h`）
```c
esp_err_t esp_rmaker_ota_enable_default(void);
esp_err_t esp_rmaker_ota_enable(esp_rmaker_ota_config_t *ota_config, esp_rmaker_ota_type_t type);
esp_err_t esp_rmaker_ota_report_status(esp_rmaker_ota_handle_t ota_handle, ota_status_t status, char *additional_info);
esp_err_t esp_rmaker_ota_default_cb(esp_rmaker_ota_handle_t handle, esp_rmaker_ota_data_t *ota_data);
esp_err_t esp_rmaker_ota_https_cb(esp_rmaker_ota_handle_t handle, esp_rmaker_ota_data_t *ota_data);
esp_err_t esp_rmaker_ota_mqtt_cb(esp_rmaker_ota_handle_t handle, esp_rmaker_ota_data_t *ota_data);
esp_err_t esp_rmaker_ota_fetch(void);
esp_err_t esp_rmaker_ota_fetch_with_delay(int time);
esp_err_t esp_rmaker_ota_mark_valid(void);
esp_err_t esp_rmaker_ota_mark_invalid(void);

extern const char *ESP_RMAKER_OTA_DEFAULT_SERVER_CERT;

typedef enum { OTA_USING_PARAMS=1, OTA_USING_TOPICS } esp_rmaker_ota_type_t;
typedef enum { OTA_STATUS_IN_PROGRESS=1, OTA_STATUS_SUCCESS, OTA_STATUS_FAILED,
               OTA_STATUS_DELAYED, OTA_STATUS_REJECTED } ota_status_t;
typedef enum { RMAKER_OTA_EVENT_INVALID=0, RMAKER_OTA_EVENT_STARTING, RMAKER_OTA_EVENT_IN_PROGRESS,
               RMAKER_OTA_EVENT_SUCCESSFUL, RMAKER_OTA_EVENT_FAILED, RMAKER_OTA_EVENT_REJECTED,
               RMAKER_OTA_EVENT_DELAYED, RMAKER_OTA_EVENT_REQ_FOR_REBOOT } esp_rmaker_ota_event_t;
```

## MQTT（`esp_rmaker_mqtt.h`）
```c
esp_err_t esp_rmaker_mqtt_init(esp_rmaker_mqtt_conn_params_t *conn_params);
void     esp_rmaker_mqtt_deinit(void);
esp_err_t esp_rmaker_mqtt_connect(void);
esp_err_t esp_rmaker_mqtt_disconnect(void);
esp_err_t esp_rmaker_mqtt_publish(const char *topic, void *data, size_t data_len, uint8_t qos, int *msg_id);
esp_err_t esp_rmaker_mqtt_subscribe(const char *topic, esp_rmaker_mqtt_subscribe_cb_t cb, uint8_t qos, void *priv_data);
esp_err_t esp_rmaker_mqtt_unsubscribe(const char *topic);
bool     esp_rmaker_mqtt_is_budget_available(void);
bool     esp_rmaker_is_mqtt_connected(void);
void     esp_rmaker_create_mqtt_topic(char *buf, size_t buf_size, const char *topic_suffix, const char *rule);
```

## 调度 / 场景 / 连接性 / 分组 / 用户映射
```c
/* esp_rmaker_schedule.h */
esp_err_t esp_rmaker_schedule_enable(void);
/* esp_rmaker_scenes.h */
esp_err_t esp_rmaker_scenes_enable(void);
/* esp_rmaker_connectivity.h */
esp_err_t esp_rmaker_connectivity_enable(void);
esp_err_t esp_rmaker_connectivity_update_lwt(const char *group_id);
bool     esp_rmaker_connectivity_is_enabled(void);
/* esp_rmaker_groups.h */
esp_err_t esp_rmaker_groups_service_enable(void);
/* esp_rmaker_user_mapping.h */
esp_rmaker_user_mapping_state_t esp_rmaker_user_node_mapping_get_state(void);
esp_err_t esp_rmaker_user_mapping_endpoint_create(void);
esp_err_t esp_rmaker_user_mapping_endpoint_register(void);
esp_err_t esp_rmaker_start_user_node_mapping(char *user_id, char *secret_key);
/* esp_rmaker_console.h */
esp_err_t esp_rmaker_console_init(void);
void     esp_rmaker_register_commands(void);
```

## Controller（`esp_rmaker_controller.h`）
```c
typedef esp_err_t (*esp_rmaker_controller_cb_t)(const char *node_id, const char *data,
                                                size_t data_size, void *priv_data);

typedef struct {
    bool report_node_details;
    esp_rmaker_controller_cb_t cb;
    void *priv_data;
} esp_rmaker_controller_config_t;

/* 须在 esp_rmaker_node_init() 之后、esp_rmaker_start() 之前调用 */
esp_err_t esp_rmaker_controller_enable(esp_rmaker_controller_config_t *config);
esp_err_t esp_rmaker_controller_disable(void);
esp_err_t esp_rmaker_controller_get_active_group_id(char **group_id);   /* 调用者 free */
```

## Thread Border Router（`esp_rmaker_thread_br.h`）
```c
#include <esp_openthread.h>
/* 须在 esp_rmaker_node_init() 之后、esp_rmaker_start() 之前调用 */
esp_err_t esp_rmaker_thread_br_enable(const esp_openthread_platform_config_t *platform_config);
/* platform_config 三段（ESP_OPENTHREAD_DEFAULT_*_CONFIG() 宏）：host_config / port_config / radio_config */
```

## Auth Service（`esp_rmaker_auth_service.h`）— 控制器节点用
```c
esp_err_t esp_rmaker_auth_service_enable(void);
esp_err_t esp_rmaker_auth_service_get_user_token(char **user_token);    /* 调用者 free */
esp_err_t esp_rmaker_auth_service_get_base_url(char **base_url);        /* 调用者 free */
esp_err_t esp_rmaker_user_auth_service_token_status_update(esp_rmaker_user_auth_service_token_status_t status);

typedef enum {
    ESP_RMAKER_USER_AUTH_SERVICE_TOKEN_STATUS_NONE = 0,
    ESP_RMAKER_USER_AUTH_SERVICE_TOKEN_STATUS_NOT_VERIFIED = 1,
    ESP_RMAKER_USER_AUTH_SERVICE_TOKEN_STATUS_VERIFIED = 2,
    ESP_RMAKER_USER_AUTH_SERVICE_TOKEN_STATUS_EXPIRED_OR_INVALID = 3,
    ESP_RMAKER_USER_AUTH_SERVICE_TOKEN_STATUS_MAX,
} esp_rmaker_user_auth_service_token_status_t;
```

## User API（`examples/common/rmaker_user_api/`，示例组件而非组件库头文件）
> core 层 `include/app_rmaker_user_api.h`；helper 层 `include/app_rmaker_user_helper_api.h`（需 `CONFIG_ENABLE_RM_USER_HELPER_API=y`）。返回的 `char *` 均由调用者 `free()`。

```c
/* core */
typedef enum { APP_RMAKER_USER_API_TYPE_GET, APP_RMAKER_USER_API_TYPE_POST,
               APP_RMAKER_USER_API_TYPE_PUT, APP_RMAKER_USER_API_TYPE_DELETE } app_rmaker_user_api_type_t;

typedef struct {
    bool reuse_session;
    bool no_need_authorize;
    bool payload_is_json;
    app_rmaker_user_api_type_t api_type;
    const char *api_name;           /* 如 "user/nodes"、"user/nodes/mapping" */
    const char *api_version;        /* 如 "v1" */
    const char *api_query_params;   /* 如 "node_details=true&status=true" */
    const char *api_payload;        /* JSON 或 form 字符串 */
} app_rmaker_user_api_request_config_t;

typedef struct {
    char *base_url;
    char *refresh_token;            /* 或 username/password */
} app_rmaker_user_api_config_t;

esp_err_t app_rmaker_user_api_init(app_rmaker_user_api_config_t *config);
esp_err_t app_rmaker_user_api_deinit(void);
esp_err_t app_rmaker_user_api_generic(app_rmaker_user_api_request_config_t *req,
                                      int *status_code, char **response_data);
void app_rmaker_user_api_register_login_failure_callback(void (*cb)(int code, const char *reason));
void app_rmaker_user_api_register_login_success_callback(void (*cb)(void));

/* helper（CONFIG_ENABLE_RM_USER_HELPER_API=y）*/
esp_err_t app_rmaker_user_helper_api_get_user_id(char **user_id);
esp_err_t app_rmaker_user_helper_api_get_nodes_list(char **nodes_list, uint16_t *nodes_count);
esp_err_t app_rmaker_user_helper_api_get_node_config(const char *node_id, char **node_config);
esp_err_t app_rmaker_user_helper_api_get_node_params(const char *node_id, char **node_params);
esp_err_t app_rmaker_user_helper_api_set_node_params(const char *node_id, const char *payload, char **response_data);
esp_err_t app_rmaker_user_helper_api_get_node_connection_status(const char *node_id, bool *connection_status);
esp_err_t app_rmaker_user_helper_api_get_groups(const char *group_id, char **groups);
/* 另有 set_node_mapping / get_node_mapping_status / create_group / delete_group / operate_node_to_group
 * 完整列表见 examples/common/rmaker_user_api/README.md */
```

## On-network Challenge-Response（以太网配网用）
> 两种实现互斥（共用 protocomm_httpd 单例）：`CONFIG_ESP_RMAKER_ON_NETWORK_CHAL_RESP_ENABLE`（独立 HTTP 服务）或 `CONFIG_ESP_RMAKER_LOCAL_CTRL_CHAL_RESP_ENABLE`（复用 local control）。后者 API 在 `esp_rmaker_core.h`。

```c
/* 独立 HTTP 服务（esp_rmaker_on_network_chal_resp.h）*/
esp_rmaker_on_network_chal_resp_config_t cfg = ESP_RMAKER_ON_NETWORK_CHAL_RESP_DEFAULT_CONFIG();
/* cfg.port / cfg.sec_ver / cfg.pop 可覆盖 */
esp_err_t esp_rmaker_on_network_chal_resp_start(esp_rmaker_on_network_chal_resp_config_t *config);

/* 复用 local control（esp_rmaker_core.h）*/
bool     esp_rmaker_local_ctrl_service_started(void);
esp_err_t esp_rmaker_local_ctrl_enable_chal_resp(const char *instance_name);  /* 如 "PROV_xxyyzz" */
esp_err_t esp_rmaker_local_ctrl_disable_chal_resp(void);
```

## 网关/控制器/BR 相关标准类型宏（`esp_rmaker_standard_types.h` / `esp_rmaker_standard_params.h`）
```c
#define ESP_RMAKER_DEVICE_ZIGBEE_GATEWAY   "esp.device.zigbee_gateway"
#define ESP_RMAKER_DEVICE_THREAD_BR        "esp.device.thread-br"
/* 注：esp.device.controller 在仓库中是字面量字符串，未定义为宏 */

#define ESP_RMAKER_PARAM_ADD_ZIGBEE_DEVICE "esp.param.add_zigbee_device"
#define ESP_RMAKER_DEF_ADD_ZIGBEE_DEVICE   "Add_zigbee_device"   /* 参数名 (esp_rmaker_standard_params.h) */

#define ESP_RMAKER_UI_TOGGLE    "esp.ui.toggle"
#define ESP_RMAKER_UI_TRIGGER   "esp.ui.trigger"
#define ESP_RMAKER_UI_QR_SCAN   "esp.ui.qr-scan"
```
