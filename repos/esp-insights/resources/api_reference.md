# ESP-Insights API Quick Reference

> 所有签名均来自仓库真实头文件（`components/*/include/*.h`）。按组件/头文件分组。未列出的即为不存在，勿臆造。
> 多数 metrics/variables/network 接口受 `#if CONFIG_DIAG_ENABLE_*` 守卫，未启用对应 Kconfig 时这些符号不参与编译。

---

## esp_insights.h（核心 agent，`components/esp_insights/include/esp_insights.h`）

### 类型与事件

```c
/* 配置 */
typedef struct {
    uint32_t      log_type;     /* esp_diag_log_type_t 位或 */
    const char   *node_id;      /* NULL 则用 MAC */
    const char   *auth_key;     /* 仅 HTTPS 有效 */
    bool          alloc_ext_ram;/* 尽量把大 buffer 分到外部 RAM */
} esp_insights_config_t;

/* 事件 base */
ESP_EVENT_DECLARE_BASE(INSIGHTS_EVENT);

typedef enum {
    INSIGHTS_EVENT_TRANSPORT_SEND_SUCCESS,
    INSIGHTS_EVENT_TRANSPORT_SEND_FAILED,
    INSIGHTS_EVENT_TRANSPORT_RECV,
} esp_insights_event_t;

typedef struct {
    uint8_t *data;
    size_t   data_len;
    int      msg_id;
} esp_insights_transport_event_data_t;

/* transport 回调原型 */
typedef esp_err_t (*esp_insights_transport_init_t)(void *userdata);
typedef void      (*esp_insights_transport_deinit_t)(void);
typedef esp_err_t (*esp_insights_transport_connect_t)(void);
typedef void      (*esp_insights_transport_disconnect_t)(void);
typedef int       (*esp_insights_transport_data_send_t)(void *data, size_t len);

typedef struct {
    struct {
        esp_insights_transport_init_t       init;
        esp_insights_transport_deinit_t     deinit;
        esp_insights_transport_connect_t    connect;
        esp_insights_transport_disconnect_t disconnect;
        esp_insights_transport_data_send_t  data_send;
    } callbacks;
    void *userdata;
} esp_insights_transport_config_t;
```

### 函数

```c
esp_err_t    esp_insights_init(esp_insights_config_t *config);
void         esp_insights_deinit(void);

esp_err_t    esp_insights_transport_register(esp_insights_transport_config_t *config);
void         esp_insights_transport_unregister(void);

esp_err_t    esp_insights_send_data(void);

esp_err_t    esp_insights_enable(esp_insights_config_t *config);
void         esp_insights_disable(void);

const char  *esp_insights_get_node_id(void);
bool         esp_insights_is_reporting_enabled(void);
esp_err_t    esp_insights_reporting_enable(void);
esp_err_t    esp_insights_reporting_disable(void);

esp_err_t    esp_insights_test_cmd_handler(void);     /* 仅测试解析器 */
esp_err_t    esp_insights_cmd_resp_enable(void);      /* RainMaker MQTT 命令响应 */
```

---

## esp_diagnostics.h（日志钩子 / 数据类型，`components/esp_diagnostics/include/esp_diagnostics.h`）

### 枚举与结构

```c
typedef enum {
    ESP_DIAG_LOG_TYPE_ERROR   = 1 << 0,
    ESP_DIAG_LOG_TYPE_WARNING = 1 << 1,
    ESP_DIAG_LOG_TYPE_EVENT   = 1 << 2,
} esp_diag_log_type_t;

typedef enum { /* 日志参数类型（TLV 用） */
    ARG_TYPE_CHAR, ARG_TYPE_SHORT, ARG_TYPE_INT, ARG_TYPE_L, ARG_TYPE_LL,
    ARG_TYPE_INTMAX, ARG_TYPE_PTRDIFF, ARG_TYPE_UCHAR, ARG_TYPE_USHORT,
    ARG_TYPE_UINT, ARG_TYPE_UL, ARG_TYPE_ULL, ARG_TYPE_UINTMAX, ARG_TYPE_SIZE,
    ARG_TYPE_DOUBLE, ARG_TYPE_LDOUBLE, ARG_TYPE_STR, ARG_TYPE_INVALID,
} esp_diag_arg_type_t;

typedef enum {
    ESP_DIAG_DATA_PT_METRICS, ESP_DIAG_DATA_PT_VARIABLE,
} esp_diag_data_pt_type_t;

typedef enum {
    ESP_DIAG_DATA_TYPE_BOOL, ESP_DIAG_DATA_TYPE_INT, ESP_DIAG_DATA_TYPE_UINT,
    ESP_DIAG_DATA_TYPE_FLOAT, ESP_DIAG_DATA_TYPE_STR, ESP_DIAG_DATA_TYPE_IPv4,
    ESP_DIAG_DATA_TYPE_MAC, ESP_DIAG_DATA_TYPE_NULL, ESP_DIAG_DATA_TYPE_MAX,
} esp_diag_data_type_t;

typedef struct {
    esp_diag_log_type_t type;
    uint32_t pc;
    uint64_t timestamp;
    char tag[16];
    void *msg_ptr;
    uint8_t msg_args[CONFIG_DIAG_LOG_MSG_ARG_MAX_SIZE];
    uint8_t msg_args_len;
    char task_name[CONFIG_FREERTOS_MAX_TASK_NAME_LEN];
} esp_diag_log_data_t;

#define DIAG_HEX_SHA_SIZE 16
#define DIAG_SHA_SIZE     (DIAG_HEX_SHA_SIZE / 2)
typedef struct {
    uint32_t chip_model, chip_rev, reset_reason;
    char app_version[32], project_name[32];
    char app_elf_sha256[DIAG_HEX_SHA_SIZE + 1];
} esp_diag_device_info_t;

typedef struct { uint32_t bt[16]; uint32_t depth; bool corrupted; } esp_diag_task_bt_t;
typedef struct {
    char name[CONFIG_FREERTOS_MAX_TASK_NAME_LEN];
    uint32_t state, high_watermark;
#ifndef CONFIG_IDF_TARGET_ARCH_RISCV
    esp_diag_task_bt_t bt_info;   /* 仅 Xtensa */
#endif
} esp_diag_task_info_t;

typedef struct {
    uint16_t type, data_type;
#ifndef CONFIG_ESP_INSIGHTS_META_VERSION_10
    char tag[16];
#endif
    char key[16]; uint64_t ts;
    union { bool b; int32_t i; uint32_t u; float f; uint32_t ipv4; uint8_t mac[6]; } value;
} esp_diag_data_pt_t;

typedef struct { /* 字符串数据点 */
    uint16_t type, data_type;
#ifndef CONFIG_ESP_INSIGHTS_META_VERSION_10
    char tag[16];
#endif
    char key[16]; uint64_t ts;
    union { char str[32]; } value;
} esp_diag_str_data_pt_t;

typedef struct {
    esp_diag_log_write_cb_t write_cb;
    void *cb_arg;
} esp_diag_log_config_t;
```

### 函数 / 宏

```c
esp_err_t esp_diag_log_hook_init(esp_diag_log_config_t *config);
void      esp_diag_log_hook_enable(uint32_t type);
void      esp_diag_log_hook_disable(uint32_t type);

esp_err_t esp_diag_log_event(const char *tag, const char *format, ...)
          __attribute__((format(printf, 2, 3)));

#define ESP_DIAG_EVENT(tag, format, ...) \
{ \
    esp_diag_log_event(tag, "EV (%" PRIu32 ") %s: " format, esp_log_timestamp(), tag, ##__VA_ARGS__); \
    ESP_LOGI(tag, format, ##__VA_ARGS__); \
}

esp_err_t esp_diag_device_info_get(esp_diag_device_info_t *device_info);
uint64_t  esp_diag_timestamp_get(void);
uint32_t  esp_diag_task_snapshot_get(esp_diag_task_info_t *tasks, size_t size);
void      esp_diag_task_snapshot_dump(void);
uint32_t  esp_diag_meta_crc_get(void);
uint32_t  esp_diag_data_size_get_crc(void);

/* 外部包装日志时用（CONFIG_DIAG_USE_EXTERNAL_LOG_WRAP） */
void esp_diag_log_writev(esp_log_level_t level, const char *tag, const char *format, va_list v);
void esp_diag_log_write (esp_log_level_t level, const char *tag, const char *format, va_list v);
```

---

## esp_diagnostics_metrics.h（需 `CONFIG_DIAG_ENABLE_METRICS`）

```c
typedef esp_err_t (*esp_diag_metrics_write_cb_t)(const char *tag, void *data, size_t len, void *cb_arg);
typedef struct { esp_diag_metrics_write_cb_t write_cb; void *cb_arg; } esp_diag_metrics_config_t;
typedef struct {
    const char *tag, *key, *label, *path, *unit;
    esp_diag_data_type_t type;
} esp_diag_metrics_meta_t;

esp_err_t esp_diag_metrics_init(esp_diag_metrics_config_t *config);
esp_err_t esp_diag_metrics_deinit(void);
esp_err_t esp_diag_metrics_register(const char *tag, const char *key, const char *label,
                                    const char *path, esp_diag_data_type_t type);
esp_err_t esp_diag_metrics_unregister_all(void);
const esp_diag_metrics_meta_t *esp_diag_metrics_meta_get_all(uint32_t *len);
void      esp_diag_metrics_meta_print_all(void);

#ifndef CONFIG_ESP_INSIGHTS_META_VERSION_10   /* ---- 2.0 ---- */
esp_err_t esp_diag_metrics_unregister(const char *tag, const char *key);
esp_err_t esp_diag_metrics_add_unit(const char *tag, const char *key, const char *unit);
esp_err_t esp_diag_metrics_report(esp_diag_data_type_t data_type, const char *tag,
                                  const char *key, const void *val, size_t val_sz, uint64_t ts);
esp_err_t esp_diag_metrics_report_bool (const char *tag, const char *key, bool b);
esp_err_t esp_diag_metrics_report_int  (const char *tag, const char *key, int32_t i);
esp_err_t esp_diag_metrics_report_uint (const char *tag, const char *key, uint32_t u);
esp_err_t esp_diag_metrics_report_float(const char *tag, const char *key, float f);
esp_err_t esp_diag_metrics_report_ipv4 (const char *tag, const char *key, uint32_t ip);
esp_err_t esp_diag_metrics_report_mac  (const char *tag, const char *key, uint8_t *mac);
esp_err_t esp_diag_metrics_report_str  (const char *tag, const char *key, const char *str);
#else                                        /* ---- 1.0（仅 key） ---- */
esp_err_t esp_diag_metrics_unregister(const char *key);
esp_err_t esp_diag_metrics_add_unit(const char *key, const char *unit);
esp_err_t esp_diag_metrics_add(esp_diag_data_type_t data_type, const char *key,
                               const void *val, size_t val_sz, uint64_t ts);
esp_err_t esp_diag_metrics_add_bool (const char *key, bool b);
esp_err_t esp_diag_metrics_add_int  (const char *key, int32_t i);
esp_err_t esp_diag_metrics_add_uint (const char *key, uint32_t u);
esp_err_t esp_diag_metrics_add_float(const char *key, float f);
esp_err_t esp_diag_metrics_add_ipv4 (const char *key, uint32_t ip);
esp_err_t esp_diag_metrics_add_mac  (const char *key, uint8_t *mac);
esp_err_t esp_diag_metrics_add_str  (const char *key, const char *str);
#endif
```

---

## esp_diagnostics_variables.h（需 `CONFIG_DIAG_ENABLE_VARIABLES`）

```c
typedef esp_err_t (*esp_diag_variable_write_cb_t)(const char *tag, void *data, size_t len, void *cb_arg);
typedef struct { esp_diag_variable_write_cb_t write_cb; void *cb_arg; } esp_diag_variable_config_t;
typedef struct {
    const char *tag, *key, *label, *path, *unit;
    esp_diag_data_type_t type;
} esp_diag_variable_meta_t;

esp_err_t esp_diag_variable_init(esp_diag_variable_config_t *config);
esp_err_t esp_diag_variables_deinit(void);
esp_err_t esp_diag_variable_register(const char *tag, const char *key, const char *label,
                                     const char *path, esp_diag_data_type_t type);
esp_err_t esp_diag_variable_unregister_all(void);
const esp_diag_variable_meta_t *esp_diag_variable_meta_get_all(uint32_t *len);
void      esp_diag_variable_meta_print_all(void);

#ifndef CONFIG_ESP_INSIGHTS_META_VERSION_10   /* ---- 2.0 ---- */
esp_err_t esp_diag_variable_unregister(const char *tag, const char *key);
esp_err_t esp_diag_variable_add_unit(const char *tag, const char *key, const char *unit);
esp_err_t esp_diag_variable_report(esp_diag_data_type_t data_type, const char *tag,
                                   const char *key, const void *val, size_t val_sz, uint64_t ts);
esp_err_t esp_diag_variable_report_bool (const char *tag, const char *key, bool b);
esp_err_t esp_diag_variable_report_int  (const char *tag, const char *key, int32_t i);
esp_err_t esp_diag_variable_report_uint (const char *tag, const char *key, uint32_t u);
esp_err_t esp_diag_variable_report_float(const char *tag, const char *key, float f);
esp_err_t esp_diag_variable_report_ipv4 (const char *tag, const char *key, uint32_t ip);
esp_err_t esp_diag_variable_report_mac  (const char *tag, const char *key, uint8_t *mac);
esp_err_t esp_diag_variable_report_str  (const char *tag, const char *key, const char *str);
#else                                        /* ---- 1.0 ---- */
esp_err_t esp_diag_variable_unregister(const char *key);
esp_err_t esp_diag_variable_add_unit(const char *key, const char *unit);
esp_err_t esp_diag_variable_add(esp_diag_data_type_t data_type, const char *key,
                                const void *val, size_t val_sz, uint64_t ts);
esp_err_t esp_diag_variable_add_bool (const char *key, bool b);
esp_err_t esp_diag_variable_add_int  (const char *key, int32_t i);
esp_err_t esp_diag_variable_add_uint (const char *key, uint32_t u);
esp_err_t esp_diag_variable_add_float(const char *key, float f);
esp_err_t esp_diag_variable_add_ipv4 (const char *key, uint32_t ip);
esp_err_t esp_diag_variable_add_mac  (const char *key, uint8_t *mac);
esp_err_t esp_diag_variable_add_str  (const char *key, const char *str);
#endif
```

---

## esp_diagnostics_system_metrics.h

```c
#if CONFIG_DIAG_ENABLE_HEAP_METRICS
esp_err_t esp_diag_heap_metrics_init(void);
esp_err_t esp_diag_heap_metrics_deinit(void);
void      esp_diag_heap_metrics_reset_interval(uint32_t period);  /* 秒；0=停 */
esp_err_t esp_diag_heap_metrics_dump(void);
#endif

#if CONFIG_DIAG_ENABLE_WIFI_METRICS
esp_err_t esp_diag_wifi_metrics_init(void);
esp_err_t esp_diag_wifi_metrics_deinit(void);
esp_err_t esp_diag_wifi_metrics_dump(void);
void      esp_diag_wifi_metrics_reset_interval(uint32_t period);  /* 秒；0=停 */
#endif
```

---

## esp_diagnostics_network_variables.h（需 `CONFIG_DIAG_ENABLE_NETWORK_VARIABLES`）

```c
esp_err_t esp_diag_network_variables_init(void);
esp_err_t esp_diag_network_variables_deinit(void);
```

---

## esp_diag_data_store.h（数据存储抽象，`components/esp_diag_data_store/include/esp_diag_data_store.h`）

```c
ESP_EVENT_DECLARE_BASE(ESP_DIAG_DATA_STORE_EVENT);
typedef enum {
    ESP_DIAG_DATA_STORE_EVENT_CRITICAL_DATA_WRITE_FAIL,
    ESP_DIAG_DATA_STORE_EVENT_NON_CRITICAL_DATA_WRITE_FAIL,
    ESP_DIAG_DATA_STORE_EVENT_CRITICAL_DATA_LOW_MEM,
    ESP_DIAG_DATA_STORE_EVENT_NON_CRITICAL_DATA_LOW_MEM,
} esp_diag_data_store_events_t;

esp_err_t esp_diag_data_store_critical_write(void *data, size_t len);
esp_err_t esp_diag_data_store_non_critical_write(const char *dg, void *data, size_t len);
int      esp_diag_data_store_critical_read(uint8_t *buf, size_t size);
int      esp_diag_data_store_non_critical_read(uint8_t *buf, size_t size);
esp_err_t esp_diag_data_store_critical_release(size_t size);
esp_err_t esp_diag_data_store_non_critical_release(size_t size);
esp_err_t esp_diag_data_store_init(void);
void      esp_diag_data_store_deinit(void);
uint32_t  esp_diag_data_store_get_crc(void);
esp_err_t esp_diag_data_discard_data(void);   /* init 之后调用 */
```
