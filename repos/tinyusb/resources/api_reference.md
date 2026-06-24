# TinyUSB API 速查（按模块分组）

> 所有签名取自 `src/` 真实头文件。设备 API 前缀 `tud_`，主机 API 前缀 `tuh_`，核心栈前缀 `tusb_`。多实例 API 带 `_n_` 后缀（单实例默认 instance 0）。

## 核心栈（src/tusb.h, src/device/usbd.h）

```c
// 初始化（宏重载，推荐显式形式）
bool tusb_rhport_init(uint8_t rhport, const tusb_rhport_init_t *rh_init);
#define tusb_init(...)   // tusb_init(rhport, &init) 或 tusb_init()（需定义 RHPORT0/1_MODE）
bool tusb_inited(void);
bool tusb_deinit(uint8_t rhport);

// 中断转发（在 USB ISR 里调用）
void tusb_int_handler(uint8_t rhport, bool in_isr);

// 主任务
//   设备：tud_task()（src/device/usbd.h 内 inline）
//   主机：tuh_task()（src/host/usbh.h 内 inline）
//   带超时/ISR 变体：tud_task_ext(timeout_ms, in_isr) / tuh_task_ext(timeout_ms, in_isr)
```

### 角色与速度枚举（tusb.h / tusb_option.h）

```c
// tusb_rhport_init_t.role
TUSB_ROLE_INVALID, TUSB_ROLE_DEVICE, TUSB_ROLE_HOST
// tusb_rhport_init_t.speed
TUSB_SPEED_AUTO, TUSB_SPEED_LOW, TUSB_SPEED_FULL, TUSB_SPEED_HIGH
```

## 设备核心（src/device/usbd.h）

```c
bool tud_connected(void);
bool tud_mounted(void);
bool tud_suspended(void);
bool tud_remote_wakeup(void);
bool tud_control_xfer(uint8_t rhport, tusb_control_request_t const *request,
                      void *buffer, uint16_t len);

// 描述符回调（设备必须实现前三个）
uint8_t const*  tud_descriptor_device_cb(void);
uint8_t const*  tud_descriptor_configuration_cb(uint8_t index);
uint16_t const* tud_descriptor_string_cb(uint8_t index, uint16_t langid);
uint8_t const*  tud_descriptor_bos_cb(void);                       // 可选，BOS/WebUSB
uint8_t const*  tud_descriptor_device_qualifier_cb(void);          // 可选（高速）
uint8_t const*  tud_descriptor_other_speed_configuration_cb(uint8_t index);

// 状态回调
void tud_mount_cb(void);
void tud_umount_cb(void);
void tud_suspend_cb(bool remote_wakeup_en);
void tud_resume_cb(void);

// Vendor 控制传输回调
bool tud_vendor_control_xfer_cb(uint8_t rhport, uint8_t stage,
                                tusb_control_request_t const *request);

// 自定义类驱动注入（不改栈源码）
usbd_class_driver_t const* usbd_app_driver_get_cb(uint8_t *driver_count);

// 描述符辅助宏（usbd.h）
TUD_CONFIG_DESCRIPTOR(cfg, itf_count, str, total_len, attr, max_power)
TUD_CDC_DESCRIPTOR(itfnum, stridx, ep_notif, ep_notif_size, epout, epin, epsize)
TUD_MSC_DESCRIPTOR(itfnum, stridx, epout, epin, epsize)
TUD_HID_DESCRIPTOR(itfnum, stridx, boot_proto, report_desc_len, epin, epsize, ep_interval)
TUD_HID_INOUT_DESCRIPTOR(itfnum, stridx, boot_proto, report_desc_len, epout, epin, epsize, ep_interval)
TUD_MIDI_DESC_HEAD / TUD_MIDI_DESC_HEAD_LEN / ...        // 见 usbd.h 完整定义
TUD_CDC_DESC_LEN / TUD_MSC_DESC_LEN / TUD_HID_DESC_LEN / TUD_HID_INOUT_DESC_LEN
```

## CDC 设备（src/class/cdc/cdc_device.h）

```c
bool     tud_cdc_configure(const tud_cdc_configure_t *driver_cfg);
bool     tud_cdc_n_ready(uint8_t itf);
bool     tud_cdc_n_connected(uint8_t itf);
uint8_t  tud_cdc_n_get_line_state(uint8_t itf);
void     tud_cdc_n_get_line_coding(uint8_t itf, cdc_line_coding_t *coding);
void     tud_cdc_n_set_wanted_char(uint8_t itf, char wanted);

uint32_t tud_cdc_n_available(uint8_t itf);
uint32_t tud_cdc_n_read(uint8_t itf, void *buffer, uint32_t bufsize);
void     tud_cdc_n_read_flush(uint8_t itf);
bool     tud_cdc_n_peek(uint8_t itf, uint8_t *ui8);

uint32_t tud_cdc_n_write(uint8_t itf, void const *buffer, uint32_t bufsize);
uint32_t tud_cdc_n_write_flush(uint8_t itf);
uint32_t tud_cdc_n_write_available(uint8_t itf);
bool     tud_cdc_n_write_clear(uint8_t itf);

bool     tud_cdc_n_notify_uart_state(uint8_t itf, const cdc_notify_uart_state_t *state);
bool     tud_cdc_n_notify_conn_speed_change(uint8_t itf, const cdc_notify_conn_speed_change_t *cs);

// 单实例（instance 0）inline 简写：tud_cdc_read / write / available / write_flush / connected / ...

// 应用回调
void tud_cdc_rx_cb(uint8_t itf);
void tud_cdc_rx_wanted_cb(uint8_t itf, char wanted_char);
void tud_cdc_tx_complete_cb(uint8_t itf);
void tud_cdc_notify_complete_cb(uint8_t itf);
void tud_cdc_line_state_cb(uint8_t itf, bool dtr, bool rts);
void tud_cdc_line_coding_cb(uint8_t itf, cdc_line_coding_t const *p_line_coding);
void tud_cdc_send_break_cb(uint8_t itf, uint16_t duration_ms);
```

## HID 设备（src/class/hid/hid_device.h）

```c
bool     tud_hid_n_ready(uint8_t instance);
uint8_t  tud_hid_n_interface_protocol(uint8_t instance);   // HID_ITF_PROTOCOL_*
uint8_t  tud_hid_n_get_protocol(uint8_t instance);         // 0=boot 1=report
bool     tud_hid_n_report(uint8_t instance, uint8_t report_id, void const *report, uint16_t len);
bool     tud_hid_n_keyboard_report(uint8_t instance, uint8_t report_id, uint8_t modifier, const uint8_t keycode[6]);
bool     tud_hid_n_mouse_report(uint8_t instance, uint8_t report_id, uint8_t buttons,
                                int8_t x, int8_t y, int8_t vertical, int8_t horizontal);
bool     tud_hid_n_abs_mouse_report(uint8_t instance, uint8_t report_id, uint8_t buttons,
                                    int16_t x, int16_t y, int8_t vertical, int8_t horizontal);
bool     tud_hid_n_gamepad_report(uint8_t instance, uint8_t report_id, int8_t x, int8_t y, int8_t z,
                                  int8_t rz, int8_t rx, int8_t ry, uint8_t hat, uint32_t buttons);
bool     tud_hid_n_stylus_report(uint8_t instance, uint8_t report_id, uint8_t attrs, uint16_t x, uint16_t y);
// 单实例 inline 简写：tud_hid_ready / tud_hid_report / tud_hid_keyboard_report / ...

// 回调
uint8_t const* tud_hid_descriptor_report_cb(uint8_t instance);   // 必须
uint16_t tud_hid_get_report_cb(uint8_t instance, uint8_t report_id,
                               hid_report_type_t report_type, uint8_t *buffer, uint16_t reqlen);
void     tud_hid_set_report_cb(uint8_t instance, uint8_t report_id, hid_report_type_t report_type,
                               uint8_t const *buffer, uint16_t bufsize);
void     tud_hid_set_protocol_cb(uint8_t instance, uint8_t protocol);
bool     tud_hid_set_idle_cb(uint8_t instance, uint8_t idle_rate);
void     tud_hid_report_complete_cb(uint8_t instance, uint8_t const *report, uint16_t len);
void     tud_hid_report_failed_cb(uint8_t instance, hid_report_type_t report_type,
                                  uint8_t const *report, uint16_t xferred_bytes);
```

## MSC 设备（src/class/msc/msc_device.h）

```c
bool    tud_msc_set_sense(uint8_t lun, uint8_t sense_key, uint8_t add_sense_code, uint8_t add_sense_qualifier);
bool    tud_msc_async_io_done(int32_t bytes_io, bool in_isr);

// 后端回调
int32_t tud_msc_read10_cb(uint8_t lun, uint32_t lba, uint32_t offset, void *buffer, uint32_t bufsize);
int32_t tud_msc_write10_cb(uint8_t lun, uint32_t lba, uint32_t offset, uint8_t *buffer, uint32_t bufsize);
void    tud_msc_inquiry_cb(uint8_t lun, uint8_t vendor_id[8], uint8_t product_id[16], uint8_t product_rev[4]);
uint32_t tud_msc_inquiry2_cb(uint8_t lun, scsi_inquiry_resp_t *inquiry_resp, uint32_t bufsize);
bool    tud_msc_test_unit_ready_cb(uint8_t lun);
void    tud_msc_capacity_cb(uint8_t lun, uint32_t *block_count, uint16_t *block_size);
int32_t tud_msc_scsi_cb(uint8_t lun, uint8_t const scsi_cmd[16], void *buffer, uint16_t bufsize);
uint8_t tud_msc_get_maxlun_cb(void);
bool    tud_msc_start_stop_cb(uint8_t lun, uint8_t power_condition, bool start, bool load_eject);
bool    tud_msc_prevent_allow_medium_removal_cb(uint8_t lun, uint8_t prohibit_removal, uint8_t control);
int32_t tud_msc_request_sense_cb(uint8_t lun, void *buffer, uint16_t bufsize);
void    tud_msc_read10_complete_cb(uint8_t lun);
void    tud_msc_write10_complete_cb(uint8_t lun);
void    tud_msc_scsi_complete_cb(uint8_t lun, uint8_t const scsi_cmd[16]);
bool    tud_msc_is_writable_cb(uint8_t lun);
```

## Vendor 设备（src/class/vendor/vendor_device.h）

```c
// 缓冲模式（CFG_TUD_VENDOR_RX_BUFSIZE > 0）
uint32_t tud_vendor_available(void);
uint32_t tud_vendor_read(void *buffer, uint32_t bufsize);
uint32_t tud_vendor_write(void const *buffer, uint32_t bufsize);
uint32_t tud_vendor_write_flush(void);
uint32_t tud_vendor_write_available(void);
// 多实例 tud_vendor_n_*

// 零缓冲直通模式（RX_BUFSIZE = 0）
void tud_vendor_rx_cb(uint8_t itf, uint8_t const *buffer, uint16_t bufsize);

// 配置描述符宏
TUD_VENDOR_DESCRIPTOR(itfnum, stridx, epout, epin, epsize)
```

## 主机核心（src/host/usbh.h）

```c
void tuh_task_ext(uint32_t timeout_ms, bool in_isr);   // tuh_task() 为 inline 简写
void tuh_mount_cb(uint8_t dev_addr);
void tuh_umount_cb(uint8_t dev_addr);
usbd_class_driver_t const* usbh_app_driver_get_cb(uint8_t *driver_count);  // 自定��主机类

// MAX3421E（SPI 外置主机控制器）
bool tuh_max3421_reg_write(uint8_t rhport, uint8_t reg, uint8_t data, bool in_isr);
uint8_t tuh_max3421_reg_read(uint8_t rhport, uint8_t reg, bool in_isr);
```

## CDC 主机（src/class/cdc/cdc_host.h）

```c
uint8_t  tuh_cdc_itf_get_index(uint8_t daddr, uint8_t itf_num);
bool     tuh_cdc_itf_get_info(uint8_t idx, tuh_itf_info_t *info);
bool     tuh_cdc_mounted(uint8_t idx);
bool     tuh_cdc_get_control_line_state_local(uint8_t idx, uint16_t *line_state);
bool     tuh_cdc_get_line_coding_local(uint8_t idx, cdc_line_coding_t *line_coding);

uint32_t tuh_cdc_write_available(uint8_t idx);
uint32_t tuh_cdc_write(uint8_t idx, void const *buffer, uint32_t bufsize);
uint32_t tuh_cdc_write_flush(uint8_t idx);
bool     tuh_cdc_write_clear(uint8_t idx);
uint32_t tuh_cdc_read_available(uint8_t idx);
uint32_t tuh_cdc_read(uint8_t idx, void *buffer, uint32_t bufsize);
bool     tuh_cdc_peek(uint8_t idx, uint8_t *ch);
bool     tuh_cdc_read_clear(uint8_t idx);

// 异步控制请求（完成回调 tuh_xfer_cb_t）
bool tuh_cdc_set_control_line_state(uint8_t idx, uint16_t line_state, tuh_xfer_cb_t complete_cb, uintptr_t user_data);
bool tuh_cdc_set_baudrate(uint8_t idx, uint32_t baudrate, tuh_xfer_cb_t complete_cb, uintptr_t user_data);
bool tuh_cdc_set_data_format(uint8_t idx, uint8_t stop_bits, uint8_t parity, uint8_t data_bits, tuh_xfer_cb_t complete_cb, uintptr_t user_data);
bool tuh_cdc_set_line_coding(uint8_t idx, cdc_line_coding_t const *line_coding, tuh_xfer_cb_t complete_cb, uintptr_t user_data);
```

## MSC 主机（src/class/msc/msc_host.h）

```c
bool     tuh_msc_mounted(uint8_t dev_addr);
bool     tuh_msc_ready(uint8_t dev_addr);
uint8_t  tuh_msc_get_maxlun(uint8_t dev_addr);
uint32_t tuh_msc_get_block_count(uint8_t dev_addr, uint8_t lun);
uint32_t tuh_msc_get_block_size(uint8_t dev_addr, uint8_t lun);

// 异步操作（回调 tuh_msc_complete_cb_t）
bool tuh_msc_scsi_command(uint8_t daddr, msc_cbw_t const *cbw, void *data, tuh_msc_complete_cb_t complete_cb, uintptr_t arg);
bool tuh_msc_inquiry(uint8_t dev_addr, uint8_t lun, scsi_inquiry_resp_t *response, tuh_msc_complete_cb_t complete_cb, uintptr_t arg);
bool tuh_msc_test_unit_ready(uint8_t dev_addr, uint8_t lun, tuh_msc_complete_cb_t complete_cb, uintptr_t arg);
bool tuh_msc_request_sense(uint8_t dev_addr, uint8_t lun, void *response, tuh_msc_complete_cb_t complete_cb, uintptr_t arg);
bool tuh_msc_read10(uint8_t dev_addr, uint8_t lun, void *buffer, uint32_t lba, uint16_t block_count, tuh_msc_complete_cb_t complete_cb, uintptr_t arg);
bool tuh_msc_write10(uint8_t dev_addr, uint8_t lun, void const *buffer, uint32_t lba, uint16_t block_count, tuh_msc_complete_cb_t complete_cb, uintptr_t arg);
bool tuh_msc_read_capacity(uint8_t dev_addr, uint8_t lun, scsi_read_capacity10_resp_t *response, tuh_msc_complete_cb_t complete_cb, uintptr_t arg);

void tuh_msc_mount_cb(uint8_t dev_addr);
void tuh_msc_umount_cb(uint8_t dev_addr);
```

## HID 主机（src/class/hid/hid_host.h）

```c
uint8_t tuh_hid_itf_get_count(uint8_t dev_addr);
uint8_t tuh_hid_itf_get_total_count(void);
bool    tuh_hid_itf_get_info(uint8_t daddr, uint8_t idx, tuh_itf_info_t *itf_info);
uint8_t tuh_hid_itf_get_index(uint8_t daddr, uint8_t itf_num);
uint8_t tuh_hid_interface_protocol(uint8_t dev_addr, uint8_t idx);
bool    tuh_hid_mounted(uint8_t dev_addr, uint8_t idx);
uint8_t tuh_hid_get_protocol(uint8_t dev_addr, uint8_t idx);
void    tuh_hid_set_default_protocol(uint8_t protocol);
bool    tuh_hid_set_protocol(uint8_t dev_addr, uint8_t idx, uint8_t protocol);
bool    tuh_hid_get_report(uint8_t dev_addr, uint8_t idx, uint8_t report_id, uint8_t report_type, void *report, uint16_t len);
bool    tuh_hid_set_report(uint8_t dev_addr, uint8_t idx, uint8_t report_id, uint8_t report_type, void *report, uint16_t len);
bool    tuh_hid_receive_ready(uint8_t dev_addr, uint8_t idx);
bool    tuh_hid_receive_report(uint8_t dev_addr, uint8_t idx);
bool    tuh_hid_receive_abort(uint8_t dev_addr, uint8_t idx);
bool    tuh_hid_send_ready(uint8_t dev_addr, uint8_t idx);
bool    tuh_hid_send_report(uint8_t dev_addr, uint8_t idx, uint8_t report_id, const void *report, uint16_t len);

void tuh_hid_mount_cb(uint8_t dev_addr, uint8_t idx, uint8_t const *report_desc, uint16_t desc_len);
void tuh_hid_umount_cb(uint8_t dev_addr, uint8_t idx);
void tuh_hid_report_received_cb(uint8_t dev_addr, uint8_t idx, uint8_t const *report, uint16_t len);
```

## Audio 设备（src/class/audio/audio_device.h）

> TX = 设备→主机（麦克风，IN 端点）；RX = 主机→设备（扬声器，OUT 端点）。所有 `_n_` 多实例版有对应单实例 inline 简写（默认 func_id=0）。各 API 受 `CFG_TUD_AUDIO_ENABLE_EP_IN/OUT/FEEDBACK_EP/INTERRUPT_EP` 条件编译控制。

```c
bool     tud_audio_n_mounted(uint8_t func_id);
uint8_t  tud_audio_n_version(uint8_t func_id);                       // 1=UAC1, 2=UAC2

#if CFG_TUD_AUDIO_ENABLE_EP_OUT   // RX（扬声器方向）
uint16_t   tud_audio_n_available       (uint8_t func_id);
uint16_t   tud_audio_n_read            (uint8_t func_id, void *buffer, uint16_t bufsize);
bool       tud_audio_n_clear_ep_out_ff (uint8_t func_id);
tu_fifo_t* tud_audio_n_get_ep_out_ff   (uint8_t func_id);
#endif

#if CFG_TUD_AUDIO_ENABLE_EP_IN     // TX（麦克风方向）
uint16_t   tud_audio_n_write                 (uint8_t func_id, const void *data, uint16_t len);
bool       tud_audio_n_clear_ep_in_ff        (uint8_t func_id);
tu_fifo_t* tud_audio_n_get_ep_in_ff          (uint8_t func_id);
uint16_t   tud_audio_n_get_ep_in_fifo_threshold(uint8_t func_id);
void       tud_audio_n_set_ep_in_fifo_threshold(uint8_t func_id, uint16_t threshold);
#endif

// 把应答拷进控制缓冲并触发 EP0 发送（持久缓冲可改用 tud_control_xfer 省拷贝）
bool tud_audio_buffer_and_schedule_control_xfer(uint8_t rhport, tusb_control_request_t const *p_request,
                                                void *data, uint16_t len);

#if CFG_TUD_AUDIO_ENABLE_EP_OUT && CFG_TUD_AUDIO_ENABLE_FEEDBACK_EP
bool      tud_audio_n_fb_set(uint8_t func_id, uint32_t feedback);          // 16.16 格式
uint32_t  tud_audio_feedback_update(uint8_t func_id, uint32_t cycles);     // SOF ISR 里按主时钟周期数算
void      tud_audio_feedback_params_cb(uint8_t func_id, uint8_t alt_itf, audio_feedback_params_t *feedback_param);
TU_ATTR_FAST_FUNC void tud_audio_feedback_interval_isr(uint8_t func_id, uint32_t frame_number, uint8_t interval_shift);
// 反馈法枚举：AUDIO_FEEDBACK_METHOD_DISABLED/FIFO_COUNT/FREQUENCY_FIXED/FREQUENCY_FLOAT
#endif

// 接口/控制回调
bool tud_audio_set_itf_cb(uint8_t rhport, tusb_control_request_t const *p_request);
bool tud_audio_set_itf_close_ep_cb(uint8_t rhport, tusb_control_request_t const *p_request);
bool tud_audio_set_req_ep_cb(uint8_t rhport, tusb_control_request_t const *p_request, uint8_t *pBuff);
bool tud_audio_set_req_itf_cb(uint8_t rhport, tusb_control_request_t const *p_request, uint8_t *pBuff);
bool tud_audio_set_req_entity_cb(uint8_t rhport, tusb_control_request_t const *p_request, uint8_t *pBuff);
bool tud_audio_get_req_ep_cb(uint8_t rhport, tusb_control_request_t const *p_request);
bool tud_audio_get_req_itf_cb(uint8_t rhport, tusb_control_request_t const *p_request);
bool tud_audio_get_req_entity_cb(uint8_t rhport, tusb_control_request_t const *p_request);

// 端点尺寸宏（src/device/usbd.h）：TUD_AUDIO_EP_SIZE(is_hs, sample_rate, bytes_per_sample, n_channels)
```

## DFU 设备（src/class/dfu/dfu_device.h + dfu_rt_device.h）

> DFU 模式（`CFG_TUD_DFU`）：实际收发固件。DFU Runtime（`CFG_TUD_DFU_RUNTIME`）：应用运行时声明可进 DFU，仅响应 DETACH。

```c
// ===== DFU 模式（dfu_device.h）=====
// 烧写完成后必须调用（download_cb/manifest_cb 内或异步完成处）
void tud_dfu_finish_flashing(uint8_t status);     // DFU_STATUS_OK 或 errWRITE/errVERIFY 等

// 应用回调（alt = 分区号）
uint32_t tud_dfu_get_timeout_cb(uint8_t alt, uint8_t state);                    // bwPollTimeout(ms)，state 取自 dfu_state_t
void    tud_dfu_download_cb(uint8_t alt, uint16_t block_num, uint8_t const *data, uint16_t length);
void    tud_dfu_manifest_cb(uint8_t alt);
uint16_t tud_dfu_upload_cb(uint8_t alt, uint16_t block_num, uint8_t *data, uint16_t length);
void    tud_dfu_abort_cb(uint8_t alt);
void    tud_dfu_detach_cb(void);

// ===== DFU Runtime（dfu_rt_device.h）=====
void tud_dfu_runtime_reboot_to_dfu_cb(void);   // 收到 DETACH：真实产品里复位进 bootloader
```

DFU 枚举（`dfu.h`）：协议 `DFU_PROTOCOL_RT`(0x01)/`DFU_PROTOCOL_DFU`(0x02)；属性 `DFU_ATTR_CAN_DOWNLOAD/CAN_UPLOAD/MANIFESTATION_TOLERANT/WILL_DETACH`；状态 `dfu_state_t`（`DFU_DNBUSY`/`DFU_MANIFEST` 等）；状态码 `DFU_STATUS_*`。
描述符宏（`usbd.h`）：`TUD_DFU_DESCRIPTOR(itfnum, alt_count, stridx, attr, timeout, xfer_size)`、`TUD_DFU_RT_DESCRIPTOR(itfnum, stridx, attr, timeout, xfer_size)`。

## MIDI 设备（src/class/midi/midi_device.h）

> USB-MIDI 事件包为 4 字节：`[cable<<4 | CIN, status, data1, data2]`。stream API 自动算 CIN，packet API 需调用方提供完整 4 字节。

```c
bool     tud_midi_n_mounted(uint8_t itf);
uint32_t tud_midi_n_available(uint8_t itf, uint8_t cable_num);
uint32_t tud_midi_n_stream_read (uint8_t itf, uint8_t cable_num, void *buffer, uint32_t bufsize);
uint32_t tud_midi_n_stream_write(uint8_t itf, uint8_t cable_num, const uint8_t *buffer, uint32_t bufsize);
bool     tud_midi_n_packet_read (uint8_t itf, uint8_t packet[4]);
bool     tud_midi_n_packet_write(uint8_t itf, const uint8_t packet[4]);
uint32_t tud_midi_n_packet_read_n (uint8_t itf, uint8_t packets[], uint32_t max_packets);
uint32_t tud_midi_n_packet_write_n(uint8_t itf, const uint8_t packets[], uint32_t n_packets);

// 单实例 inline 简写（itf=0）：tud_midi_mounted / available / stream_read / stream_write(cable_num,...) /
//   packet_read / packet_write / packet_read_n / packet_write_n

void tud_midi_rx_cb(uint8_t itf);
```

Code Index Number 枚举（`midi.h`）：`MIDI_CIN_NOTE_OFF`(8)/`NOTE_ON`(9)/`CONTROL_CHANGE`(11)/`PROGRAM_CHANGE`(12)/`PITCH_BEND_CHANGE`(14)/`SYSEX_START`(4)/`SYSEX_END_1BYTE`(5)...`_3BYTE`(7)。
描述符宏（`usbd.h`）：`TUD_MIDI_DESCRIPTOR(itfnum, stridx, epout, epin, epsize)`、`TUD_MIDI_DESC_LEN`。

## USBTMC 设备（src/class/usbtmc/usbtmc_device.h）

> USBTMC/USB488（SCPI/VISA 仪器）。`CFG_TUD_USBTMC_ENABLE_488=1` 时部分回调签名带 488 扩展。处理完接收类回调后必须调 `tud_usbtmc_start_bus_read()` 准备下一包。

```c
// 应用发送（Bulk-IN）：缓冲引用持有到 msgBulkIn_complete_cb，期间不可改
bool tud_usbtmc_transmit_dev_msg_data(const void *data, size_t len, bool endOfMessage, bool usingTermChar);
// 中断端点通知（SRQ 等，首字节是 bNotify1）
bool tud_usbtmc_transmit_notification_data(const void *data, size_t len);
// 准备接收下一包 Bulk-OUT（每个接收/完成回调末尾必调）
bool tud_usbtmc_start_bus_read(void);

// 回调
#if CFG_TUD_USBTMC_ENABLE_488
usbtmc_response_capabilities_488_t const * tud_usbtmc_get_capabilities_cb(void);
#else
usbtmc_response_capabilities_t const * tud_usbtmc_get_capabilities_cb(void);
#endif
void tud_usbtmc_open_cb(uint8_t interface_id);
bool tud_usbtmc_msgBulkOut_start_cb(usbtmc_msg_request_dev_dep_out const *msgHeader);
bool tud_usbtmc_msg_data_cb(void *data, size_t len, bool transfer_complete);
bool tud_usbtmc_msgBulkIn_request_cb(usbtmc_msg_request_dev_dep_in const *request);
bool tud_usbtmc_msgBulkIn_complete_cb(void);
bool tud_usbtmc_initiate_abort_bulk_in_cb(uint8_t *tmcResult);
bool tud_usbtmc_initiate_abort_bulk_out_cb(uint8_t *tmcResult);
bool tud_usbtmc_initiate_clear_cb(uint8_t *tmcResult);
bool tud_usbtmc_check_abort_bulk_in_cb(usbtmc_check_abort_bulk_rsp_t *rsp);
bool tud_usbtmc_check_abort_bulk_out_cb(usbtmc_check_abort_bulk_rsp_t *rsp);
bool tud_usbtmc_check_clear_cb(usbtmc_get_clear_status_rsp_t *rsp);
bool tud_usbtmc_notification_complete_cb(void);
bool tud_usbtmc_indicator_pulse_cb(tusb_control_request_t const *msg, uint8_t *tmcResult);
void tud_usbtmc_bulkIn_clearFeature_cb(void);
void tud_usbtmc_bulkOut_clearFeature_cb(void);
#if CFG_TUD_USBTMC_ENABLE_488
uint8_t tud_usbtmc_get_stb_cb(uint8_t *tmcResult);                 // 返回 IEEE-488.2 状态字节
bool   tud_usbtmc_msg_trigger_cb(usbtmc_msg_generic_t *msg);       // *TRG / ASSERT TRIGGER
#endif
```

常量（`usbtmc.h`）：`USBTMC_VERSION`/`USBTMC_488_VERSION`(0x0100)、状态码 `USBTMC_STATUS_SUCCESS`(0x01)/`PENDING`(0x02)/`FAILED`(0x80)...。
描述符宏（`usbd.h`）：`TUD_USBTMC_DESC(itfnum, bulk_epsize)`、协议 `TUD_USBTMC_PROTOCOL_STD`(0x00)/`TUD_USBTMC_PROTOCOL_USB488`(0x01)。

## Type-C / Power Delivery（src/typec/usbc.h + tcd.h + pd_types.h）

> 独立于 `tud_`/`tuh_` 数据栈。`tuc_` = 应用 API，`tcd_` = 控制器驱动移植层。需 `CFG_TUC_ENABLED=1`（默认 0）。**WIP，仅 STM32 G4**。

```c
// ===== 应用 API（usbc.h）=====
bool tuc_init(uint8_t rhport, uint32_t port_type);     // port_type: TUSB_TYPEC_PORT_SRC/SNK/DRP
bool tuc_inited(uint8_t rhport);
void tuc_task_ext(uint32_t timeout_ms, bool in_isr);
void tuc_task(void);                                   // inline 简写（= tuc_task_ext(UINT32_MAX,false)）
#define tuc_int_handler  tcd_int_handler               // ISR 转发别名
bool tuc_msg_request(uint8_t rhport, void const *rdo); // sink 发 RDO 回复 Source Cap

// PD 消息回调
bool tuc_pd_data_received_cb   (uint8_t rhport, pd_header_t const *header, uint8_t const *dobj, uint8_t const *p_end);
bool tuc_pd_control_received_cb(uint8_t rhport, pd_header_t const *header);

// ===== 控制器驱动移植层（tcd.h，MCU/板卡实现）=====
bool  tcd_init(uint8_t rhport, uint32_t port_type);
void  tcd_int_enable(uint8_t rhport);
void  tcd_int_disable(uint8_t rhport);
void  tcd_int_handler(uint8_t rhport);
bool  tcd_msg_receive(uint8_t rhport, uint8_t *buffer, uint16_t total_bytes);
bool  tcd_msg_send(uint8_t rhport, uint8_t const *buffer, uint16_t total_bytes);
void  tcd_event_handler(tcd_event_t const *event, bool in_isr);   // 事件: TCD_EVENT_CC_CHANGED/RX_COMPLETE/TX_COMPLETE
```

关键类型（`pd_types.h`）：端口角色 `TUSB_TYPEC_PORT_SRC/SNK/DRP`；`pd_header_t`（`msg_type`/`n_data_obj`/`power_role`/`data_role`）；PDO 类型 `PD_PDO_TYPE_FIXED/BATTERY/VARIABLE/APDO`，结构 `pd_pdo_fixed_t`（`voltage_50mv`/`current_max_10ma`）；RDO `pd_rdo_fixed_variable_t`（`object_position`/`current_operate_10ma`/`current_extremum_10ma`）；消息类型 `PD_DATA_SOURCE_CAP`/`PD_CTRL_ACCEPT`/`PD_CTRL_REJECT`/`PD_CTRL_PS_READY`。

## 板级 API（hw/bsp/board_api.h，示例统一用）

```c
void     board_init(void);
void     board_init_after_tusb(void);
uint32_t board_millis(void);
void     board_led_write(bool state);
uint32_t board_button_read(void);
uint8_t  board_uart_read(void);          // 见 board_api.h 完整签名
// 注：board_api.h 由各 BSP 实现，具体以 hw/bsp/board_api.h 为准
```
