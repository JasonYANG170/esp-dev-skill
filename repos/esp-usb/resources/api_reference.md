# ESP-USB API Quick Reference

Real signatures grouped by module. All names come from the repository headers (`device/esp_tinyusb/include*`, `host/usb/include/usb`, `host/class/*/include`). Header paths are relative to the repo root.

---

## USB Device — `esp_tinyusb` (component `espressif/esp_tinyusb`)

Header: `device/esp_tinyusb/include/tinyusb.h`

```c
typedef enum {
    TINYUSB_PORT_FULL_SPEED_0 = 0,
#if (SOC_USB_OTG_PERIPH_NUM > 1)
    TINYUSB_PORT_HIGH_SPEED_0,
#endif
    TINYUSB_PORT_MAX,
} tinyusb_port_t;

typedef struct {
    bool skip_setup;          // bypass internal PHY setup (external PHY)
    bool self_powered;        // enable VBUS monitoring
    int  vbus_monitor_io;     // GPIO for VBUS sense (ignored if !self_powered)
} tinyusb_phy_config_t;

typedef struct {
    size_t  size;             // device task stack size in bytes
    uint8_t priority;         // device task priority
    int     xCoreID;          // core affinity
} tinyusb_task_config_t;

typedef struct {
    const tusb_desc_device_t          *device;
    const tusb_desc_device_qualifier_t *qualifier;
    const char **string;
    int           string_count;
    const uint8_t *full_speed_config;
    const uint8_t *high_speed_config;
} tinyusb_desc_config_t;

typedef enum {
    TINYUSB_EVENT_ATTACHED = 0,
    TINYUSB_EVENT_DETACHED = 1,
#ifdef CONFIG_TINYUSB_SUSPEND_CALLBACK
    TINYUSB_EVENT_SUSPENDED = 2,
#endif
#ifdef CONFIG_TINYUSB_RESUME_CALLBACK
    TINYUSB_EVENT_RESUMED = 3,
#endif
} tinyusb_event_id_t;

typedef struct {
    tinyusb_event_id_t id;
    uint8_t rhport;
    union {
        struct { bool remote_wakeup; } suspended;
    };
} tinyusb_event_t;

typedef void (*tinyusb_event_cb_t)(tinyusb_event_t *event, void *arg);

typedef struct {
    tinyusb_port_t         port;
    tinyusb_phy_config_t   phy;
    tinyusb_task_config_t  task;
    tinyusb_desc_config_t  descriptor;
    tinyusb_event_cb_t     event_cb;
    void                  *event_arg;
} tinyusb_config_t;

esp_err_t tinyusb_driver_install(const tinyusb_config_t *config);
esp_err_t tinyusb_driver_uninstall(void);
esp_err_t tinyusb_remote_wakeup(void);

#define TINYUSB_ESPRESSIF_VID  0x303A
```

Header: `device/esp_tinyusb/include/tinyusb_default_config.h`

```c
// Macro: initialize tinyusb_config_t with target defaults.
//   TINYUSB_DEFAULT_CONFIG()                 // no callback
//   TINYUSB_DEFAULT_CONFIG(event_cb)
//   TINYUSB_DEFAULT_CONFIG(event_cb, arg)
#define TINYUSB_DEFAULT_CONFIG(...)

#define TINYUSB_TASK_DEFAULT()                 // {4096, 5, TINYUSB_DEFAULT_TASK_AFFINITY}
#define TINYUSB_TASK_CUSTOM(size, prio, core)  // custom task cfg
#define TINYUSB_DEFAULT_TASK_SIZE    4096
#define TINYUSB_DEFAULT_TASK_PRIO    5
#if CONFIG_FREERTOS_UNICORE
#define TINYUSB_DEFAULT_TASK_AFFINITY 0
#else
#define TINYUSB_DEFAULT_TASK_AFFINITY 1
#endif
#define TINYUSB_CONFIG_FULL_SPEED(event_hdl, arg)  // FS-only initializer
#if (SOC_USB_OTG_PERIPH_NUM > 1) || CONFIG_IDF_TARGET_ESP32S31
#define TINYUSB_CONFIG_HIGH_SPEED(event_hdl, arg)  // HS-capable initializer
#endif
```

### CDC-ACM device

Header: `device/esp_tinyusb/include/tinyusb_cdc_acm.h`

```c
typedef enum { TINYUSB_CDC_ACM_0 = 0, TINYUSB_CDC_ACM_1, TINYUSB_CDC_ACM_MAX } tinyusb_cdcacm_itf_t;

typedef enum {
    CDC_EVENT_RX,
    CDC_EVENT_RX_WANTED_CHAR,
    CDC_EVENT_LINE_STATE_CHANGED,
    CDC_EVENT_LINE_CODING_CHANGED,
} cdcacm_event_type_t;

typedef struct {
    cdcacm_event_type_t type;
    union {
        cdcacm_event_rx_wanted_char_data_t      rx_wanted_char_data;
        cdcacm_event_line_state_changed_data_t  line_state_changed_data;   // .dtr, .rts
        cdcacm_event_line_coding_changed_data_t line_coding_changed_data;  // .p_line_coding
    };
} cdcacm_event_t;

typedef void (*tusb_cdcacm_callback_t)(int itf, cdcacm_event_t *event);

typedef struct {
    tinyusb_cdcacm_itf_t     cdc_port;
    tusb_cdcacm_callback_t   callback_rx;
    tusb_cdcacm_callback_t   callback_rx_wanted_char;
    tusb_cdcacm_callback_t   callback_line_state_changed;
    tusb_cdcacm_callback_t   callback_line_coding_changed;
} tinyusb_config_cdcacm_t;

esp_err_t tinyusb_cdcacm_init(const tinyusb_config_cdcacm_t *cfg);
esp_err_t tinyusb_cdcacm_deinit(int itf);
esp_err_t tinyusb_cdcacm_register_callback(tinyusb_cdcacm_itf_t itf,
                                           cdcacm_event_type_t event_type,
                                           tusb_cdcacm_callback_t callback);
esp_err_t tinyusb_cdcacm_unregister_callback(tinyusb_cdcacm_itf_t itf, cdcacm_event_type_t event_type);
size_t   tinyusb_cdcacm_write_queue_char(tinyusb_cdcacm_itf_t itf, char ch);
size_t   tinyusb_cdcacm_write_queue(tinyusb_cdcacm_itf_t itf, const uint8_t *in_buf, size_t in_size);
esp_err_t tinyusb_cdcacm_write_flush(tinyusb_cdcacm_itf_t itf, uint32_t timeout_ticks);
esp_err_t tinyusb_cdcacm_read(tinyusb_cdcacm_itf_t itf, uint8_t *out_buf, size_t out_buf_sz, size_t *rx_data_size);
bool     tinyusb_cdcacm_initialized(tinyusb_cdcacm_itf_t itf);
```

### Console / VFS

Header: `device/esp_tinyusb/include/tinyusb_console.h`

```c
esp_err_t tinyusb_console_init(int cdc_intf);     // redirect stdin/stdout/stderr to CDC
esp_err_t tinyusb_console_deinit(int cdc_intf);   // restore to UART
```

Header: `device/esp_tinyusb/include/vfs_tinyusb.h`

```c
#define VFS_TUSB_MAX_PATH      16
#define VFS_TUSB_PATH_DEFAULT  "/dev/tusb_cdc"

esp_err_t esp_vfs_tusb_cdc_register(int cdc_intf, char const *path);
esp_err_t esp_vfs_tusb_cdc_unregister(char const *path);
void     esp_vfs_tusb_cdc_set_tx_line_endings(esp_line_endings_t mode);
void     esp_vfs_tusb_cdc_set_rx_line_endings(esp_line_endings_t mode);
```

### MSC device

Header: `device/esp_tinyusb/include/tinyusb_msc.h`

```c
typedef struct tinyusb_msc_storage_s *tinyusb_msc_storage_handle_t;

typedef enum {
    TINYUSB_MSC_STORAGE_MOUNT_USB = 0,   // exposed to USB host
    TINYUSB_MSC_STORAGE_MOUNT_APP,       // for local app use
} tinyusb_msc_mount_point_t;

typedef enum {
    TINYUSB_MSC_EVENT_MOUNT_START,
    TINYUSB_MSC_EVENT_MOUNT_COMPLETE,
    TINYUSB_MSC_EVENT_MOUNT_FAILED,
    TINYUSB_MSC_EVENT_FORMAT_REQUIRED,
    TINYUSB_MSC_EVENT_FORMAT_FAILED,
} tinyusb_msc_event_id_t;

typedef struct {
    tinyusb_msc_event_id_t id;
    tinyusb_msc_mount_point_t mount_point;
    union {
        struct {} event_data;
        struct { bool is_mounted; } mount_changed_data __attribute__((deprecated));
    };
} tinyusb_msc_event_t;

typedef struct {
    char *base_path;
    esp_vfs_fat_mount_config_t config;
    bool  do_not_format;
    BYTE  format_flags;
} tinyusb_msc_fatfs_config_t;

typedef void (*tusb_msc_callback_t)(tinyusb_msc_storage_handle_t handle,
                                    tinyusb_msc_event_t *event, void *arg);

typedef struct {
    union {
        wl_handle_t wl_handle;                 // SPI flash
#if (SOC_SDMMC_HOST_SUPPORTED)
        sdmmc_card_t *card;                    // SD/MMC
#endif
    } medium;
    tinyusb_msc_fatfs_config_t  fat_fs;
    tinyusb_msc_mount_point_t   mount_point;
} tinyusb_msc_storage_config_t;

typedef struct {
    union {
        struct { uint16_t auto_mount_off:1; uint16_t reserved15:15; };
        uint16_t val;
    } user_flags;
    tusb_msc_callback_t callback;
    void                *callback_arg;
} tinyusb_msc_driver_config_t;

esp_err_t tinyusb_msc_install_driver(const tinyusb_msc_driver_config_t *config);
esp_err_t tinyusb_msc_uninstall_driver(void);
esp_err_t tinyusb_msc_new_storage_spiflash(const tinyusb_msc_storage_config_t *config,
                                           tinyusb_msc_storage_handle_t *handle);
#if (SOC_SDMMC_HOST_SUPPORTED)
esp_err_t tinyusb_msc_new_storage_sdmmc(const tinyusb_msc_storage_config_t *config,
                                        tinyusb_msc_storage_handle_t *handle);
#endif
esp_err_t tinyusb_msc_delete_storage(tinyusb_msc_storage_handle_t handle);
esp_err_t tinyusb_msc_set_storage_callback(tusb_msc_callback_t callback, void *arg);
esp_err_t tinyusb_msc_format_storage(tinyusb_msc_storage_handle_t handle);
esp_err_t tinyusb_msc_config_storage_fat_fs(tinyusb_msc_storage_handle_t handle,
                                            tinyusb_msc_fatfs_config_t *fatfs_config);
esp_err_t tinyusb_msc_set_storage_mount_point(tinyusb_msc_storage_handle_t handle,
                                              tinyusb_msc_mount_point_t mount_point);
esp_err_t tinyusb_msc_get_storage_capacity(tinyusb_msc_storage_handle_t handle, uint32_t *sector_count);
esp_err_t tinyusb_msc_get_storage_sector_size(tinyusb_msc_storage_handle_t handle, uint32_t *sector_size);
esp_err_t tinyusb_msc_get_storage_mount_point(tinyusb_msc_storage_handle_t handle,
                                              tinyusb_msc_mount_point_t *mount_point);
```

### `tusb_config.h` derived macros (set by Kconfig)

Header: `device/esp_tinyusb/include/tusb_config.h`

```c
CFG_TUD_ENABLED                1
// CFG_TUD_MAX_SPEED: OPT_MODE_HIGH_SPEED (P4/S31), else OPT_MODE_FULL_SPEED
CFG_TUD_CDC                    CONFIG_TINYUSB_CDC_COUNT
CFG_TUD_MSC                    CONFIG_TINYUSB_MSC_ENABLED
CFG_TUD_HID                    CONFIG_TINYUSB_HID_COUNT
CFG_TUD_MIDI                   CONFIG_TINYUSB_MIDI_COUNT
CFG_TUD_VENDOR                 CONFIG_TINYUSB_VENDOR_COUNT
CFG_TUD_ECM_RNDIS              CONFIG_TINYUSB_NET_MODE_ECM_RNDIS
CFG_TUD_NCM                    CONFIG_TINYUSB_NET_MODE_NCM
CFG_TUD_DFU                    CONFIG_TINYUSB_DFU_MODE_DFU
CFG_TUD_DFU_RUNTIME            CONFIG_TINYUSB_DFU_MODE_DFU_RUNTIME
CFG_TUD_BTH                    CONFIG_TINYUSB_BTH_ENABLED
CFG_TUD_CDC_RX_BUFSIZE         CONFIG_TINYUSB_CDC_RX_BUFSIZE
CFG_TUD_CDC_TX_BUFSIZE         CONFIG_TINYUSB_CDC_TX_BUFSIZE
CFG_TUD_CDC_EP_BUFSIZE         CONFIG_TINYUSB_CDC_EP_BUFSIZE
CFG_TUD_MSC_BUFSIZE            CONFIG_TINYUSB_MSC_BUFSIZE
CFG_TUD_ENDPOINT0_SIZE         64
```

---

## USB Host Library — `usb` (component `espressif/usb`)

Header: `host/usb/include/usb/usb_host.h` (umbrella; also pulls `usb_helpers.h`, `usb_types_ch9.h`, `usb_types_stack.h`)

```c
#define USB_HOST_LIB_EVENT_FLAGS_NO_CLIENTS    0x01
#define USB_HOST_LIB_EVENT_FLAGS_ALL_FREE      0x02
#define USB_HOST_LIB_EVENT_FLAGS_AUTO_SUSPEND  0x04

typedef enum {
    USB_HOST_CLIENT_EVENT_NEW_DEV,
    USB_HOST_CLIENT_EVENT_DEV_GONE,
    USB_HOST_CLIENT_EVENT_DEV_SUSPENDED,
    USB_HOST_CLIENT_EVENT_DEV_RESUMED,
    USB_HOST_CLIENT_EVENT_DEV_REMOVED,
} usb_host_client_event_t;

typedef enum {
    USB_HOST_LIB_AUTO_SUSPEND_ONE_SHOT,
    USB_HOST_LIB_AUTO_SUSPEND_PERIODIC,
} usb_host_lib_auto_suspend_tmr_t;

typedef struct usb_host_client_handle_s *usb_host_client_handle_t;   // from usb_types_stack.h

typedef struct {
    usb_host_client_event_t event;
    union {
        struct { uint8_t address; }              new_dev;
        struct { usb_device_handle_t dev_hdl; }  dev_gone;
        struct { uint8_t address; }              dev_removed;
        struct { usb_device_handle_t dev_hdl; }  dev_suspend_resume;
    };
} usb_host_client_event_msg_t;

typedef void (*usb_host_client_event_cb_t)(const usb_host_client_event_msg_t *event_msg, void *arg);
typedef void (*usb_transfer_cb_t)(usb_transfer_t *transfer);
typedef bool (*usb_host_enum_filter_cb_t)(uint16_t vid, uint16_t pid, uint8_t bcdDevice,
                                          const uint8_t *device_desc, size_t device_desc_len);

typedef struct {
    bool skip_phy_setup;
    bool root_port_unpowered;
    int  intr_flags;
    usb_host_enum_filter_cb_t enum_filter_cb;     // needs CONFIG_USB_HOST_ENABLE_ENUM_FILTER_CALLBACK
    struct {
        uint32_t nptx_fifo_lines;   // >0 to enable custom
        uint32_t ptx_fifo_lines;
        uint32_t rx_fifo_lines;     // >0 to enable custom
    } fifo_settings_custom;
    unsigned peripheral_map;        // 0=default; HS default on HS-capable; bit i = peripheral i
} usb_host_config_t;

typedef struct {
    bool is_synchronous;            // set false
    int  max_num_event_msg;
    struct {
        uint32_t notify_dev_removed:1;
        uint32_t reserved31:31;
    } flags;
    union {
        struct {
            usb_host_client_event_cb_t client_event_callback;
            void                      *callback_arg;
        } async;
    };
} usb_host_client_config_t;

typedef struct {
    int  num_devices;
    int  num_clients;
    bool root_port_suspended;
} usb_host_lib_info_t;

// ---- Library functions ----
esp_err_t usb_host_install(const usb_host_config_t *config);
esp_err_t usb_host_uninstall(void);
esp_err_t usb_host_lib_handle_events(TickType_t timeout_ticks, uint32_t *event_flags_ret);
esp_err_t usb_host_lib_unblock(void);
esp_err_t usb_host_lib_info(usb_host_lib_info_t *info_ret);
esp_err_t usb_host_lib_set_root_port_power(bool enable);
esp_err_t usb_host_lib_root_port_suspend(void);
esp_err_t usb_host_lib_root_port_resume(void);
esp_err_t usb_host_lib_set_auto_suspend(usb_host_lib_auto_suspend_tmr_t timer_type,
                                        size_t timer_interval_ms);

// ---- Client functions ----
esp_err_t usb_host_client_register(const usb_host_client_config_t *client_config,
                                   usb_host_client_handle_t *client_hdl_ret);
esp_err_t usb_host_client_deregister(usb_host_client_handle_t client_hdl);
esp_err_t usb_host_client_handle_events(usb_host_client_handle_t client_hdl, TickType_t timeout_ticks);
esp_err_t usb_host_client_unblock(usb_host_client_handle_t client_hdl);

// ---- Device functions ----
esp_err_t usb_host_device_open(usb_host_client_handle_t client_hdl, uint8_t dev_addr,
                               usb_device_handle_t *dev_hdl_ret);
esp_err_t usb_host_device_close(usb_host_client_handle_t client_hdl, usb_device_handle_t dev_hdl);
esp_err_t usb_host_device_free_all(void);
esp_err_t usb_host_device_addr_list_fill(int list_len, uint8_t *dev_addr_list, int *num_dev_ret);
esp_err_t usb_host_device_info(usb_device_handle_t dev_hdl, usb_device_info_t *dev_info);
esp_err_t usb_host_get_device_descriptor(usb_device_handle_t dev_hdl, const usb_device_desc_t **device_desc);
esp_err_t usb_host_get_active_config_descriptor(usb_device_handle_t dev_hdl, const usb_config_desc_t **config_desc);
esp_err_t usb_host_get_config_desc(usb_host_client_handle_t client_hdl, usb_device_handle_t dev_hdl,
                                   uint8_t bConfigurationValue, const usb_config_desc_t **config_desc_ret);
esp_err_t usb_host_free_config_desc(const usb_config_desc_t *config_desc);

// ---- Interface / endpoint ----
esp_err_t usb_host_interface_claim(usb_host_client_handle_t client_hdl, usb_device_handle_t dev_hdl,
                                   uint8_t bInterfaceNumber, uint8_t bAlternateSetting);
esp_err_t usb_host_interface_release(usb_host_client_handle_t client_hdl, usb_device_handle_t dev_hdl,
                                     uint8_t bInterfaceNumber);
esp_err_t usb_host_endpoint_halt(usb_device_handle_t dev_hdl, uint8_t bEndpointAddress);
esp_err_t usb_host_endpoint_flush(usb_device_handle_t dev_hdl, uint8_t bEndpointAddress);
esp_err_t usb_host_endpoint_clear(usb_device_handle_t dev_hdl, uint8_t bEndpointAddress);

// ---- Transfers ----
esp_err_t usb_host_transfer_alloc(size_t data_buffer_size, int num_isoc_packets, usb_transfer_t **transfer);
esp_err_t usb_host_transfer_free(usb_transfer_t *transfer);
esp_err_t usb_host_transfer_submit(usb_transfer_t *transfer);
esp_err_t usb_host_transfer_submit_control(usb_host_client_handle_t client_hdl, usb_transfer_t *transfer);

#define REMOTE_WAKE_HAL_SUPPORTED (1)
```

---

## Host class driver — CDC-ACM (component `espressif/usb_host_cdc_acm`)

Headers: `host/class/cdc/usb_host_cdc_acm/include/usb/cdc_acm_host.h`, `cdc_host_types.h`

```c
typedef struct cdc_dev_s *cdc_acm_dev_hdl_t;

typedef enum {
    CDC_ACM_HOST_ERROR,
    CDC_ACM_HOST_SERIAL_STATE,
    CDC_ACM_HOST_NETWORK_CONNECTION,
    CDC_ACM_HOST_DEVICE_DISCONNECTED,
#ifdef CDC_HOST_SUSPEND_RESUME_API_SUPPORTED
    CDC_ACM_HOST_DEVICE_SUSPENDED,
    CDC_ACM_HOST_DEVICE_RESUMED,
#endif
} cdc_acm_host_dev_event_t;

typedef bool   (*cdc_acm_data_callback_t)(const uint8_t *data, size_t data_len, void *user_arg);
typedef void   (*cdc_acm_host_dev_callback_t)(const cdc_acm_host_dev_event_data_t *event, void *user_ctx);
typedef void   (*cdc_acm_new_dev_callback_t)(usb_device_handle_t usb_dev);

typedef struct {
    uint32_t connection_timeout_ms;
    size_t   out_buffer_size;
    size_t   in_buffer_size;
    cdc_acm_host_dev_callback_t event_cb;
    cdc_acm_data_callback_t     data_cb;
    void    *user_arg;
    uint8_t  dev_addr;          // CDC_HOST_ANY_DEV_ADDR to match any
} cdc_acm_host_device_config_t;     // (Form 2)

typedef struct {
    uint16_t vid;               // CDC_HOST_ANY_VID
    uint16_t pid;               // CDC_HOST_ANY_PID
    uint8_t  interface_idx;
    uint8_t  dev_addr;          // CDC_HOST_ANY_DEV_ADDR
    uint32_t connection_timeout_ms;
    size_t   out_buffer_size;
    size_t   in_buffer_size;
    cdc_acm_host_dev_callback_t event_cb;
    cdc_acm_data_callback_t     data_cb;
    void    *user_arg;
} cdc_acm_host_open_config_t;      // (Form 1)

typedef struct {
    size_t                     driver_task_stack_size;
    unsigned                   driver_task_priority;
    int                        xCoreID;
    cdc_acm_new_dev_callback_t new_dev_cb;
} cdc_acm_host_driver_config_t;

esp_err_t cdc_acm_host_install(const cdc_acm_host_driver_config_t *driver_config);
esp_err_t cdc_acm_host_uninstall(void);
esp_err_t cdc_acm_host_register_new_dev_callback(cdc_acm_new_dev_callback_t new_dev_cb);

// cdc_acm_host_open() is a dispatch macro (Form 1 vs Form 2):
esp_err_t cdc_acm_host_open_v2(const cdc_acm_host_open_config_t *open_config, cdc_acm_dev_hdl_t *cdc_hdl_ret);
esp_err_t cdc_acm_host_open_v1_dispatch(uint16_t vid, uint16_t pid, uint8_t interface_idx,
                                        const cdc_acm_host_device_config_t *dev_config,
                                        cdc_acm_dev_hdl_t *cdc_hdl_ret);
#define cdc_acm_host_open_vendor_specific(vid, pid, interface_num, dev_config, cdc_hdl_ret) \
        cdc_acm_host_open_v1_dispatch(vid, pid, interface_num, dev_config, cdc_hdl_ret)

esp_err_t cdc_acm_host_close(cdc_acm_dev_hdl_t cdc_hdl);
esp_err_t cdc_acm_host_data_tx_blocking(cdc_acm_dev_hdl_t cdc_hdl, const uint8_t *data,
                                        size_t data_len, uint32_t timeout_ms);
void     cdc_acm_host_desc_print(cdc_acm_dev_hdl_t cdc_hdl);
esp_err_t cdc_acm_host_protocols_get(cdc_acm_dev_hdl_t cdc_hdl, cdc_comm_protocol_t *comm, cdc_data_protocol_t *data);
esp_err_t cdc_acm_host_cdc_desc_get(cdc_acm_dev_hdl_t cdc_hdl, cdc_desc_subtype_t desc_type,
                                    const usb_standard_desc_t **cdc_desc_ret);
esp_err_t cdc_acm_host_send_custom_request(cdc_acm_dev_hdl_t cdc_hdl, uint8_t bmRequestType,
                                           uint8_t bRequest, uint16_t wValue, uint16_t wIndex,
                                           uint16_t wLength, uint8_t *buffer);
esp_err_t cdc_acm_host_enable_remote_wakeup(cdc_acm_dev_hdl_t cdc_hdl, bool enable);
```

---

## Host class driver — MSC (component `espressif/usb_host_msc`)

Headers: `host/class/msc/usb_host_msc/include/usb/msc_host.h`, `msc_host_vfs.h`

```c
typedef struct msc_host_device *msc_host_device_handle_t;
typedef struct msc_host_vfs    *msc_host_vfs_handle_t;

typedef struct {
    enum { MSC_DEVICE_CONNECTED, MSC_DEVICE_DISCONNECTED,
#ifdef MSC_HOST_SUSPEND_RESUME_API_SUPPORTED
           MSC_DEVICE_SUSPENDED, MSC_DEVICE_RESUMED,
#endif
    } event;
    union { uint8_t address; msc_host_device_handle_t handle; } device;
} msc_host_event_t;
typedef void (*msc_host_event_cb_t)(const msc_host_event_t *event, void *arg);

typedef struct {
    bool                 create_backround_task;   // (note: misspelled in source)
    size_t               task_priority;
    size_t               stack_size;
    BaseType_t           core_id;
    msc_host_event_cb_t  callback;
    void                *callback_arg;
} msc_host_driver_config_t;

typedef struct {
    uint32_t sector_count;
    uint32_t sector_size;
    uint16_t idProduct, idVendor;
    wchar_t  iManufacturer[MSC_STR_DESC_SIZE];
    wchar_t  iProduct[MSC_STR_DESC_SIZE];
    wchar_t  iSerialNumber[MSC_STR_DESC_SIZE];
} msc_host_device_info_t;

esp_err_t msc_host_install(const msc_host_driver_config_t *config);
esp_err_t msc_host_uninstall(void);
esp_err_t msc_host_install_device(uint8_t device_address, msc_host_device_handle_t *device);
esp_err_t msc_host_uninstall_device(msc_host_device_handle_t device);
esp_err_t msc_host_read_sector(msc_host_device_handle_t device, size_t sector, void *data, size_t size);
esp_err_t msc_host_write_sector(msc_host_device_handle_t device, size_t sector, const void *data, size_t size);
esp_err_t msc_host_handle_events(TickType_t timeout);
esp_err_t msc_host_get_device_info(msc_host_device_handle_t device, msc_host_device_info_t *info);
esp_err_t msc_host_print_descriptors(msc_host_device_handle_t device);
esp_err_t msc_host_reset_recovery(msc_host_device_handle_t device);

// VFS
esp_err_t msc_host_vfs_format(msc_host_device_handle_t device, const char *base_path,
                              const esp_vfs_fat_mount_config_t *mount_config, msc_host_vfs_handle_t *vfs_handle);
esp_err_t msc_host_vfs_register(msc_host_device_handle_t device, const char *base_path,
                                const esp_vfs_fat_mount_config_t *mount_config, msc_host_vfs_handle_t *vfs_handle);
esp_err_t msc_host_vfs_unregister(msc_host_vfs_handle_t vfs_handle);
```

---

## Host class driver — HID (component `espressif/usb_host_hid`)

Headers: `host/class/hid/usb_host_hid/include/usb/hid_host.h`, `hid.h`, `hid_usage_keyboard.h`, `hid_usage_mouse.h`

```c
typedef struct hid_interface *hid_host_device_handle_t;

typedef enum { HID_HOST_DRIVER_EVENT_CONNECTED, HID_HOST_DRIVER_EVENT_DISCONNECTED } hid_host_driver_event_t;
typedef enum { HID_HOST_INTERFACE_EVENT_INPUT_REPORT,
               HID_HOST_INTERFACE_EVENT_DISCONNECTED,
               HID_HOST_INTERFACE_EVENT_TRANSFER_ERROR } hid_host_interface_event_t;

typedef struct { /* iface index, device_type, ... */ } hid_host_dev_info_t;
typedef struct { /* device_type, iface_index, ... */ }  hid_host_dev_params_t;

typedef void (*hid_host_driver_event_cb_t)(hid_host_device_handle_t hid_device_handle,
                                           const hid_host_driver_event_t event, void *arg);
typedef void (*hid_host_interface_event_cb_t)(hid_host_device_handle_t hid_device_handle,
                                              const hid_host_interface_event_t event, void *arg);

typedef struct {
    bool                          create_background_task;
    size_t                        task_priority;
    size_t                        stack_size;
    BaseType_t                    core_id;
    hid_host_driver_event_cb_t    callback;
    void                         *callback_arg;
} hid_host_driver_config_t;

typedef struct {
    hid_host_interface_event_cb_t callback;
    void                         *callback_arg;
} hid_host_device_config_t;

esp_err_t hid_host_install(const hid_host_driver_config_t *config);
esp_err_t hid_host_uninstall(void);
esp_err_t hid_host_device_open(hid_host_device_handle_t hid_dev_handle, const hid_host_device_config_t *config);
esp_err_t hid_host_device_close(hid_host_device_handle_t hid_dev_handle);
esp_err_t hid_host_handle_events(uint32_t timeout);
esp_err_t hid_host_device_get_params(hid_host_device_handle_t hid_dev_handle, hid_host_dev_params_t *dev_params);
esp_err_t hid_host_device_get_raw_input_report_data(hid_host_device_handle_t hid_dev_handle,
                                                     uint8_t *data, uint16_t *data_len);
esp_err_t hid_host_enable_remote_wakeup(hid_host_device_handle_t hid_dev_handle, bool enable);
esp_err_t hid_host_device_start(hid_host_device_handle_t hid_dev_handle);
esp_err_t hid_host_device_stop(hid_host_device_handle_t hid_dev_handle);
uint8_t  *hid_host_get_report_descriptor(hid_host_device_handle_t hid_dev_handle, uint16_t *desc_len);
esp_err_t hid_host_get_device_info(hid_host_device_handle_t hid_dev_handle, hid_host_dev_info_t *hid_dev_info);
esp_err_t hid_class_request_get_report(hid_host_device_handle_t hid_dev_handle,
                                       uint8_t report_type, uint8_t report_id, uint8_t *report, uint16_t *report_len);
esp_err_t hid_class_request_get_idle(hid_host_device_handle_t hid_dev_handle, uint8_t report_id, uint8_t *idle_rate);
esp_err_t hid_class_request_get_protocol(hid_host_device_handle_t hid_dev_handle, uint8_t *protocol);
```

---

## Host class driver — UVC (component `espressif/usb_host_uvc`)

Header: `host/class/uvc/usb_host_uvc/include/usb/uvc_host.h`

```c
#define UVC_HOST_ANY_VID      (0)
#define UVC_HOST_ANY_PID      (0)
#define UVC_HOST_ANY_DEV_ADDR (0)

typedef struct uvc_host_stream_s *uvc_host_stream_hdl_t;

enum uvc_host_driver_event {
    UVC_HOST_DRIVER_EVENT_DEVICE_CONNECTED = 0x0,
};

enum uvc_host_stream_format {
    UVC_VS_FORMAT_DEFAULT = 0,
    UVC_VS_FORMAT_MJPEG,
    UVC_VS_FORMAT_YUY2,
    UVC_VS_FORMAT_H264,
    UVC_VS_FORMAT_H265,
    UVC_VS_FORMAT_NV12,
};

enum uvc_host_dev_event {
    UVC_HOST_TRANSFER_ERROR,
    UVC_HOST_DEVICE_DISCONNECTED,
    UVC_HOST_FRAME_BUFFER_OVERFLOW,
    UVC_HOST_FRAME_BUFFER_UNDERFLOW,
#ifdef UVC_HOST_SUSPEND_RESUME_API_SUPPORTED   // needs USB_HOST_LIB_EVENT_FLAGS_AUTO_SUSPEND
    UVC_HOST_DEVICE_SUSPENDED,
    UVC_HOST_DEVICE_RESUMED,
#endif
};

typedef struct {
    enum uvc_host_driver_event type;
    union {
        struct { uint8_t dev_addr; uint8_t uvc_stream_index; size_t frame_info_num; } device_connected;
    };
} uvc_host_driver_event_data_t;

typedef struct {
    enum uvc_host_dev_event type;
    union {
        struct { esp_err_t error; }                  transfer_error;
        struct { uvc_host_stream_hdl_t stream_hdl; } device_disconnected;
        struct {}                                    frame_overflow;
        struct {}                                    frame_underflow;
    };
} uvc_host_stream_event_data_t;

typedef struct { unsigned h_res; unsigned v_res; float fps; enum uvc_host_stream_format format; } uvc_host_stream_format_t;

typedef struct {
    const uvc_host_stream_format_t vs_format;
    size_t data_buffer_len;
    size_t data_len;
    uint8_t *data;
} uvc_host_frame_t;

typedef struct {
    enum uvc_host_stream_format format;
    unsigned h_res;
    unsigned v_res;
    uint32_t default_interval;        // units of 100 ns
    uint8_t interval_type;            // 0 = continuous, else discrete count
    union {
        struct { uint32_t interval_min, interval_max, interval_step; };
        uint32_t interval[CONFIG_UVC_INTERVAL_ARRAY_SIZE];
    };
} uvc_host_frame_info_t;

typedef void (*uvc_host_driver_event_callback_t)(const uvc_host_driver_event_data_t *event, void *user_ctx);
typedef void (*uvc_host_stream_callback_t)(const uvc_host_stream_event_data_t *event, void *user_ctx);
typedef bool (*uvc_host_frame_callback_t)(const uvc_host_frame_t *frame, void *user_ctx);  // true=return to driver now; false=keep, later uvc_host_frame_return()

typedef struct {
    size_t driver_task_stack_size;
    unsigned driver_task_priority;
    int xCoreID;
    bool create_background_task;       // false => app polls uvc_host_handle_events()
    uvc_host_driver_event_callback_t event_cb;
    void *user_ctx;
} uvc_host_driver_config_t;

typedef struct {
    uvc_host_stream_callback_t event_cb;
    uvc_host_frame_callback_t  frame_cb;
    void *user_ctx;
    struct {
        uint8_t dev_addr;              // 0 = any
        uint16_t vid;                  // 0 = any
        uint16_t pid;                  // 0 = any
        uint8_t uvc_stream_index;      // 0 = first function
    } usb;
    uvc_host_stream_format_t vs_format;
    struct {
        int number_of_frame_buffers;
        size_t frame_size;             // 0 = use dwMaxVideoFrameSize from negotiation
        uint32_t frame_heap_caps;      // passed to heap_caps_malloc()
        int number_of_urbs;            // triple buffering recommended
        size_t urb_size;               // 0 = default 4x MPS
        uint8_t **user_frame_buffers;  // NULL => driver allocates (since v2.4.0)
    } advanced;
} uvc_host_stream_config_t;

esp_err_t uvc_host_install(const uvc_host_driver_config_t *driver_config);
esp_err_t uvc_host_uninstall(void);
esp_err_t uvc_host_handle_events(unsigned long timeout);
esp_err_t uvc_host_stream_open(const uvc_host_stream_config_t *stream_config, int timeout, uvc_host_stream_hdl_t *stream_hdl_ret);
esp_err_t uvc_host_stream_start(uvc_host_stream_hdl_t stream_hdl);
esp_err_t uvc_host_stream_stop(uvc_host_stream_hdl_t stream_hdl);
esp_err_t uvc_host_stream_close(uvc_host_stream_hdl_t stream_hdl);
esp_err_t uvc_host_stream_format_select(uvc_host_stream_hdl_t stream_hdl, uvc_host_stream_format_t *format);
esp_err_t uvc_host_stream_format_get(uvc_host_stream_hdl_t stream_hdl, uvc_host_stream_format_t *format);
esp_err_t uvc_host_frame_return(uvc_host_stream_hdl_t stream_hdl, uvc_host_frame_t *frame);
void      uvc_host_desc_print(uvc_host_stream_hdl_t stream_hdl);
esp_err_t uvc_host_get_frame_list(uint8_t dev_addr, uint8_t uvc_stream_index,
                                  uvc_host_frame_info_t (*frame_info_list)[], size_t *list_size);
```

---

## Host class driver — UAC (component `espressif/usb_host_uac`)

Header: `host/class/uac/usb_host_uac/include/usb/uac_host.h` (also `uac.h`)

```c
#define UAC_STR_DESC_MAX_LENGTH           (32)
#define FLAG_STREAM_SUSPEND_AFTER_START   (1 << 0)   // claim iface without starting transfers

typedef struct uac_interface *uac_host_device_handle_t;

typedef enum {
    UAC_HOST_DRIVER_EVENT_RX_CONNECTED = 0x00,    // microphone found
    UAC_HOST_DRIVER_EVENT_TX_CONNECTED,           // speaker found
} uac_host_driver_event_t;

typedef enum {
    UAC_HOST_DEVICE_EVENT_RX_DONE = 0x00,         // RX ring past buffer_threshold
    UAC_HOST_DEVICE_EVENT_TX_DONE,                // TX ring below threshold
    UAC_HOST_DEVICE_EVENT_TRANSFER_ERROR,
    UAC_HOST_DRIVER_EVENT_DISCONNECTED,
} uac_host_device_event_t;

typedef enum { UAC_STREAM_TX = 0, UAC_STREAM_RX, UAC_STREAM_MAX } uac_host_stream_t;

typedef void (*uac_host_driver_event_cb_t)(uint8_t addr, uint8_t iface_num,
                                           const uac_host_driver_event_t event, void *arg);
typedef void (*uac_host_device_event_cb_t)(uac_host_device_handle_t uac_device_handle,
                                           const uac_host_device_event_t event, void *arg);

typedef struct {
    uac_host_stream_t type;  uint8_t iface_num, iface_alt_num, addr;
    uint16_t VID, PID;
    wchar_t iManufacturer[UAC_STR_DESC_MAX_LENGTH], iProduct[UAC_STR_DESC_MAX_LENGTH], iSerialNumber[UAC_STR_DESC_MAX_LENGTH];
} uac_host_dev_info_t;

typedef struct {
    uint8_t format;             // 1 = PCM
    uint8_t channels;
    uint8_t subframe_size;
    uint8_t bit_resolution;
    uint8_t sample_freq_type;   // 0 = continuous, else discrete count
    union {
        uint32_t sample_freq[UAC_FREQ_NUM_MAX];   // discrete (first UAC_FREQ_NUM_MAX)
        struct { uint32_t sample_freq_lower, sample_freq_upper; };  // continuous
    };
} uac_host_dev_alt_param_t;

typedef struct {
    bool create_background_task;
    size_t task_priority;
    size_t stack_size;
    BaseType_t core_id;
    uac_host_driver_event_cb_t callback;   // must not be NULL
    void *callback_arg;
} uac_host_driver_config_t;

typedef struct {
    uint8_t addr;                  // from driver event
    uint8_t iface_num;             // from driver event
    uint32_t buffer_size;          // internal ring buffer bytes
    uint32_t buffer_threshold;     // water level for RX_DONE / TX_DONE
    uac_host_device_event_cb_t callback;
    void *callback_arg;
} uac_host_device_config_t;

typedef struct {
    uint8_t channels;
    uint8_t bit_resolution;
    uint32_t sample_freq;
    uint16_t flags;                // bit0 = FLAG_STREAM_SUSPEND_AFTER_START
} uac_host_stream_config_t;

esp_err_t uac_host_install(const uac_host_driver_config_t *config);
esp_err_t uac_host_uninstall(void);
esp_err_t uac_host_device_open(const uac_host_device_config_t *config, uac_host_device_handle_t *uac_dev_handle);
esp_err_t uac_host_device_open_with_vid_pid(uint16_t vid, uint16_t pid, const uac_host_device_config_t *config,
                                            uac_host_device_handle_t *uac_dev_handle);
esp_err_t uac_host_device_close(uac_host_device_handle_t uac_dev_handle);
esp_err_t uac_host_get_device_info(uac_host_device_handle_t uac_dev_handle, uac_host_dev_info_t *uac_dev_info);
esp_err_t uac_host_get_device_alt_param(uac_host_device_handle_t uac_dev_handle, uint8_t iface_alt, uac_host_dev_alt_param_t *uac_alt_param);
esp_err_t uac_host_printf_device_param(uac_host_device_handle_t uac_dev_handle);
esp_err_t uac_host_handle_events(TickType_t timeout);
esp_err_t uac_host_device_start(uac_host_device_handle_t uac_dev_handle, const uac_host_stream_config_t *stream_config);
esp_err_t uac_host_device_suspend(uac_host_device_handle_t uac_dev_handle);
esp_err_t uac_host_device_resume(uac_host_device_handle_t uac_dev_handle);
esp_err_t uac_host_device_stop(uac_host_device_handle_t uac_dev_handle);
esp_err_t uac_host_device_read(uac_host_device_handle_t uac_dev_handle, uint8_t *data, uint32_t size, uint32_t *bytes_read, uint32_t timeout);
esp_err_t uac_host_device_write(uac_host_device_handle_t uac_dev_handle, uint8_t *data, uint32_t size, uint32_t timeout);
esp_err_t uac_host_device_set_mute(uac_host_device_handle_t uac_dev_handle, bool mute);
esp_err_t uac_host_device_get_mute(uac_host_device_handle_t uac_dev_handle, bool *mute);
esp_err_t uac_host_device_set_volume(uac_host_device_handle_t uac_dev_handle, uint8_t volume);   // 0..100
esp_err_t uac_host_device_get_volume(uac_host_device_handle_t uac_dev_handle, uint8_t *volume);
esp_err_t uac_host_device_set_volume_db(uac_host_device_handle_t uac_dev_handle, int16_t volume_db);  // units of 1/256 dB
esp_err_t uac_host_device_get_volume_db(uac_host_device_handle_t uac_dev_handle, int16_t *volume_db);
```

---

## USB Device — NET / NCM (component `espressif/esp_tinyusb`)

Header: `device/esp_tinyusb/include/tinyusb_net.h` (entire header is inside `#if (CONFIG_TINYUSB_NET_MODE_NONE != 1)`)

```c
typedef esp_err_t (*tusb_net_rx_cb_t)(void *buffer, uint16_t len, void *ctx);
typedef void      (*tusb_net_free_tx_cb_t)(void *buffer, void *ctx);
typedef void      (*tusb_net_init_cb_t)(void *ctx);

typedef struct {
    uint8_t mac_addr[6];
    tusb_net_rx_cb_t     on_recv_callback;
    tusb_net_free_tx_cb_t free_tx_buffer;
    tusb_net_init_cb_t   on_init_callback;
    void *user_context;
} tinyusb_net_config_t;

esp_err_t tinyusb_net_init(const tinyusb_net_config_t *cfg);   // call BEFORE tinyusb_driver_install
void     tinyusb_net_deinit(void);
esp_err_t tinyusb_net_send_sync(void *buffer, uint16_t len, void *buff_free_arg, TickType_t timeout);
esp_err_t tinyusb_net_send_async(void *buffer, uint16_t len, void *buff_free_arg);
```

> Network mode is chosen in menuconfig: `CONFIG_TINYUSB_NET_MODE_NCM` (NCM),
> `CONFIG_TINYUSB_NET_MODE_ECM_RNDIS` (ECM+RNDIS composite), or
> `CONFIG_TINYUSB_NET_MODE_NONE` (default, disables the header).
> NCM tuning: `CONFIG_TINYUSB_NCM_{IN,OUT}_NTB_BUFFS_COUNT` (default 3, 1–6),
> `CONFIG_TINYUSB_NCM_{IN,OUT}_NTB_BUFF_MAX_SIZE` (default 3200, 1600–10240, must be ≥ MTU and multiple of 4).
>
> v2 API change: the obsolete first argument `TINYUSB_USBDEV_0` was removed;
> `tinyusb_net_init(&cfg)` now takes one argument.
