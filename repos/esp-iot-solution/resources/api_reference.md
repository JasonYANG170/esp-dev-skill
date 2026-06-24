# ESP-IoT-Solution API 速查（按组件分组）

> 所有签名取自仓库 `components/<name>/include/*.h`。仅收录公开头文件中实际声明的函数/结构体/枚举，未出现的 API 视为不存在。

## i2c_bus

头文件：`components/i2c_bus/include/i2c_bus.h`

```c
#define NULL_I2C_MEM_ADDR        0xFF    // 8 位"无内部地址"占位
#define NULL_I2C_MEM_16BIT_ADDR  0xFFFF  // 16 位"无内部地址"占位
#define NULL_I2C_DEV_ADDR        0xFF    // 非法设备地址
typedef void *i2c_bus_handle_t;
typedef void *i2c_bus_device_handle_t;

// 总线 / 设备生命周期
i2c_bus_handle_t        i2c_bus_create(i2c_port_t port, const i2c_config_t *conf);
esp_err_t               i2c_bus_delete(i2c_bus_handle_t *p_bus_handle);
i2c_bus_device_handle_t i2c_bus_device_create(i2c_bus_handle_t bus_handle, uint8_t dev_addr, uint32_t clk_speed);
esp_err_t               i2c_bus_device_delete(i2c_bus_device_handle_t *p_dev_handle);

// 查询
uint8_t  i2c_bus_scan(i2c_bus_handle_t bus_handle, uint8_t *buf, uint8_t num);
uint32_t i2c_bus_get_current_clk_speed(i2c_bus_handle_t bus_handle);
uint8_t  i2c_bus_get_created_device_num(i2c_bus_handle_t bus_handle);
uint8_t  i2c_bus_device_get_address(i2c_bus_device_handle_t dev_handle);

// 8 位寄存器读写
esp_err_t i2c_bus_read_byte  (i2c_bus_device_handle_t dev, uint8_t mem_address, uint8_t *data);
esp_err_t i2c_bus_read_bytes (i2c_bus_device_handle_t dev, uint8_t mem_address, size_t len, uint8_t *data);
esp_err_t i2c_bus_write_byte (i2c_bus_device_handle_t dev, uint8_t mem_address, uint8_t data);
esp_err_t i2c_bus_write_bytes(i2c_bus_device_handle_t dev, uint8_t mem_address, size_t len, const uint8_t *data);

// 位级读写
esp_err_t i2c_bus_read_bit  (i2c_bus_device_handle_t dev, uint8_t mem_address, uint8_t bit_num, uint8_t *data);
esp_err_t i2c_bus_read_bits (i2c_bus_device_handle_t dev, uint8_t mem_address, uint8_t bit_start, uint8_t length, uint8_t *data);
esp_err_t i2c_bus_write_bit (i2c_bus_device_handle_t dev, uint8_t mem_address, uint8_t bit_num, uint8_t data);
esp_err_t i2c_bus_write_bits(i2c_bus_device_handle_t dev, uint8_t mem_address, uint8_t bit_start, uint8_t length, uint8_t data);

// 16 位寄存器地址
esp_err_t i2c_bus_read_reg16 (i2c_bus_device_handle_t dev, uint16_t mem_address, size_t len, uint8_t *data);
esp_err_t i2c_bus_write_reg16(i2c_bus_device_handle_t dev, uint16_t mem_address, size_t len, const uint8_t *data);

// 底层命令链（仅当高层接口不满足时）
esp_err_t i2c_bus_cmd_begin(i2c_bus_device_handle_t dev_handle, i2c_cmd_handle_t cmd);
```

> IDF >= 5.3 默认走新 `driver/i2c_master`；旧驱动须 menuconfig 启用 `CONFIG_I2C_BUS_BACKWARD_CONFIG`。软件 I2C 须启用 `CONFIG_I2C_BUS_SUPPORT_SOFTWARE` 后传入 `i2c_sw_port_t`（如 `I2C_NUM_SW_0`）。

## spi_bus

头文件：`components/spi_bus/include/spi_bus.h`

```c
#define NULL_SPI_CS_PIN (-1)
typedef void *spi_bus_handle_t;
typedef void *spi_bus_device_handle_t;

typedef struct {
    gpio_num_t miso_io_num;     // -1 不用
    gpio_num_t mosi_io_num;
    gpio_num_t sclk_io_num;
    int max_transfer_sz;        // < 4096 自动取 4096
} spi_config_t;

typedef struct {
    gpio_num_t cs_io_num;       // NULL_SPI_CS_PIN 表示�� CS
    uint8_t mode;               // 0/1/2/3
    int clock_speed_hz;         // SPI_MASTER_FREQ_*
} spi_device_config_t;

spi_bus_handle_t        spi_bus_create        (spi_host_device_t host_id, const spi_config_t *bus_conf);
esp_err_t               spi_bus_delete        (spi_bus_handle_t *p_bus_handle);
spi_bus_device_handle_t spi_bus_device_create (spi_bus_handle_t bus_handle, const spi_device_config_t *device_conf);
esp_err_t               spi_bus_device_delete (spi_bus_device_handle_t *p_dev_handle);

esp_err_t spi_bus_transfer_byte (spi_bus_device_handle_t dev, uint8_t data_out, uint8_t *data_in);
esp_err_t spi_bus_transfer_bytes(spi_bus_device_handle_t dev, const uint8_t *data_out, uint8_t *data_in, uint32_t data_len);
esp_err_t spi_bus_transfer_reg16(spi_bus_device_handle_t dev, uint16_t data_out, uint16_t *data_in);   // MSB 先发
esp_err_t spi_bus_transfer_reg32(spi_bus_device_handle_t dev, uint32_t data_out, uint32_t *data_in);   // MSB 先发

esp_err_t spi_bus_transmit_begin(spi_bus_device_handle_t dev_handle, spi_transaction_t *p_trans);      // polling 底层
```

## button

头文件：`components/button/include/iot_button.h`、`button_gpio.h`、`button_adc.h`、`button_matrix.h`、`button_rtc.h`、`button_types.h`

```c
typedef struct button_dev_t *button_handle_t;

// 通用配置（不含 GPIO，仅时间参数）
typedef struct {
    uint16_t long_press_time;   // 0 则用 BUTTON_LONG_PRESS_TIME_MS
    uint16_t short_press_time;  // 0 则用 BUTTON_SHORT_PRESS_TIME_MS
} button_config_t;

// 事件枚举 button_event_t
BUTTON_PRESS_DOWN, BUTTON_PRESS_UP, BUTTON_PRESS_REPEAT, BUTTON_PRESS_REPEAT_DONE,
BUTTON_SINGLE_CLICK, BUTTON_DOUBLE_CLICK, BUTTON_MULTIPLE_CLICK,
BUTTON_LONG_PRESS_START, BUTTON_LONG_PRESS_HOLD, BUTTON_LONG_PRESS_UP,
BUTTON_PRESS_END, BUTTON_EVENT_MAX, BUTTON_NONE_PRESS

// 事件参数（长按/多击）
typedef union {
    struct { uint16_t press_time; } long_press;          // LONG_PRESS_START / LONG_PRESS_UP
    struct { uint16_t clicks; }     multiple_clicks;     // MULTIPLE_CLICK
} button_event_args_t;

// 回调
typedef void (*button_cb_t)(void *button_handle, void *usr_data);

// GPIO 后端
typedef struct {
    int32_t gpio_num;
    uint8_t active_level;
    bool enable_power_save;
    bool disable_pull;
} button_gpio_config_t;
esp_err_t iot_button_new_gpio_device(const button_config_t *btn_cfg, const button_gpio_config_t *gpio_cfg, button_handle_t *ret_button);

// 注册 / 注销 / 查询
esp_err_t      iot_button_register_cb  (button_handle_t h, button_event_t event, button_event_args_t *args, button_cb_t cb, void *usr_data);
esp_err_t      iot_button_unregister_cb(button_handle_t h, button_event_t event, button_event_args_t *args);
size_t         iot_button_count_cb(button_handle_t h);
size_t         iot_button_count_event_cb(button_handle_t h, button_event_t event);
button_event_t iot_button_get_event(button_handle_t h);
const char    *iot_button_get_event_str(button_event_t event);
esp_err_t      iot_button_print_event(button_handle_t h);
uint8_t        iot_button_get_repeat(button_handle_t h);
uint32_t       iot_button_get_pressed_time(button_handle_t h);
uint16_t       iot_button_get_long_press_hold_cnt(button_handle_t h);
esp_err_t      iot_button_set_param(button_handle_t h, button_param_t param, void *value);
uint8_t        iot_button_get_key_level(button_handle_t h);
esp_err_t      iot_button_resume(void);
esp_err_t      iot_button_stop(void);
esp_err_t      iot_button_delete(button_handle_t h);

// 低功耗
typedef void (*button_power_save_cb_t)(void *usr_data);
typedef struct { button_power_save_cb_t enter_power_save_cb; void *usr_data; } button_power_save_config_t;
esp_err_t iot_button_register_power_save_cb(const button_power_save_config_t *config);
void      iot_button_power_save_wakeup_isr(uint32_t gpio_num);
```

## knob

头文件：`components/knob/include/iot_knob.h`、`knob_gpio.h`、`knob_hal.h`、`knob_rtc.h`

```c
// 事件 knob_event_t
KNOB_LEFT = 0, KNOB_RIGHT
knob_handle_t iot_knob_create(const knob_config_t *config);
int           iot_knob_get_count_value(knob_handle_t h);
esp_err_t     iot_knob_register_cb(knob_handle_t h, knob_event_t event, knob_cb_t cb, void *usr_data);
```

## led_indicator

头文件：`components/led/led_indicator/include/led_indicator.h`、`led_indicator_gpio.h`、`led_indicator_ledc.h`、`led_indicator_rgb.h`、`led_indicator_strips.h`、`led_types.h`

```c
typedef void *led_indicator_handle_t;

// 闪烁步进类型 blink_step_type_t
LED_BLINK_STOP(-1), LED_BLINK_HOLD, LED_BLINK_BREATHE, LED_BLINK_BRIGHTNESS,
LED_BLINK_RGB, LED_BLINK_RGB_RING, LED_BLINK_HSV, LED_BLINK_HSV_RING, LED_BLINK_LOOP

typedef struct {
    blink_step_type_t type;
    uint32_t value;
    uint32_t hold_time_ms;
} blink_step_t;

typedef struct {
    blink_step_t const **blink_lists;
    uint16_t blink_list_num;     // 数组下标即优先级，0 最高
} led_indicator_config_t;

// LED 状态枚举
LED_STATE_OFF(0), LED_STATE_25_PERCENT(64), LED_STATE_50_PERCENT(128),
LED_STATE_75_PERCENT(191), LED_STATE_ON(255)

// GPIO 后端
typedef struct { bool is_active_level_high; int32_t gpio_num; } led_indicator_gpio_config_t;
esp_err_t led_indicator_new_gpio_device(const led_indicator_config_t *led_config, const led_indicator_gpio_config_t *gpio_cfg, led_indicator_handle_t *handle);

// 通用控制
esp_err_t led_indicator_delete(led_indicator_handle_t handle);
esp_err_t led_indicator_start(led_indicator_handle_t handle, int blink_type);
esp_err_t led_indicator_stop (led_indicator_handle_t handle, int blink_type);
esp_err_t led_indicator_preempt_start(led_indicator_handle_t handle, int blink_type);
esp_err_t led_indicator_preempt_stop (led_indicator_handle_t handle, int blink_type);
uint8_t   led_indicator_get_brightness(led_indicator_handle_t handle);
esp_err_t led_indicator_set_on_off(led_indicator_handle_t handle, bool on_off);
esp_err_t led_indicator_set_brightness(led_indicator_handle_t handle, uint32_t brightness);
uint32_t  led_indicator_get_hsv(led_indicator_handle_t handle);
esp_err_t led_indicator_set_hsv(led_indicator_handle_t handle, uint32_t ihsv_value);
uint32_t  led_indicator_get_rgb(led_indicator_handle_t handle);
esp_err_t led_indicator_set_rgb(led_indicator_handle_t handle, uint32_t irgb_value);
esp_err_t led_indicator_set_color_temperature(led_indicator_handle_t handle, const uint32_t temperature);
```

> GPIO 单色灯 duty 应为 `LED_DUTY_1_BIT`；调亮度/颜色用 LEDC/RGB/Strips 后端。

## sensor_hub

头文件：`components/sensors/sensor_hub/include/iot_sensor_hub.h`、`sensor_type.h`、`sensor_event.h`

```c
typedef void *sensor_handle_t;
typedef void *sensor_event_handler_instance_t;
typedef void (*sensor_event_handler_t)(void *arg, sensor_event_base_t base, int32_t event_id, void *event_data);

// 传感器类型 sensor_type_t
NULL_ID, HUMITURE_ID, IMU_ID, LIGHT_SENSOR_ID, SENSOR_TYPE_MAX

// 工作模式 sensor_mode_t：MODE_DEFAULT, MODE_POLLING, MODE_INTERRUPT
// 量程     sensor_range_t：RANGE_DEFAULT, RANGE_MIN, RANGE_MEDIUM, RANGE_MAX

typedef struct {
    bus_handle_t      bus;
    uint8_t           addr;
    sensor_type_t     type;
    sensor_mode_t     mode;
    sensor_range_t    range;
    uint32_t          min_delay;   // 采集间隔 ms
    int               intr_pin;
    int               intr_type;
} sensor_config_t;

esp_err_t iot_sensor_create(const char *sensor_name, const sensor_config_t *config, sensor_handle_t *p_sensor_handle);
esp_err_t iot_sensor_start(sensor_handle_t h);
esp_err_t iot_sensor_stop (sensor_handle_t h);
esp_err_t iot_sensor_delete(sensor_handle_t h);
int       iot_sensor_scan(void);
esp_err_t iot_sensor_handler_register          (sensor_handle_t h, sensor_event_handler_t handler, sensor_event_handler_instance_t *ctx);
esp_err_t iot_sensor_handler_unregister        (sensor_handle_t h, sensor_event_handler_instance_t ctx);
esp_err_t iot_sensor_handler_register_with_type(sensor_type_t type, int32_t event_id, sensor_event_handler_t handler, sensor_event_handler_instance_t *ctx);
esp_err_t iot_sensor_handler_unregister_with_type(sensor_type_t type, int32_t event_id, sensor_event_handler_instance_t ctx);
```

数据类型 `sensor_data_t`（事件 `event_data`）含 `timestamp`、`sensor_name`、`sensor_type`、`sensor_addr` 及联合体（`acce`/`gyro`/`mag`/`temperature`/`humidity`/`baro`/`light`/`rgbw`/`uv`/`proximity`/...）。事件 ID：`SENSOR_STARTED`、`SENSOR_STOPED`、`SENSOR_TEMP_DATA_READY`(13)、`SENSOR_HUMI_DATA_READY`(14)、`SENSOR_ACCE_DATA_READY`(10)、`SENSOR_GYRO_DATA_READY`(11)、`SENSOR_LIGHT_DATA_READY`(16) 等。

## power_measure

头文件：`components/sensors/power_measure/include/power_measure.h`、`power_measure_bl0937.h`、`power_measure_bl0942.h`、`power_measure_ina236.h`

```c
typedef struct power_measure_t *power_measure_handle_t;

// 公共配置（字段以 power_measure.h 为准）：overcurrent、undervoltage、enable_energy_detection 等
typedef struct { /* 见 power_measure.h */ } power_measure_config_t;

// 创建设备（三种驱动后端，均返回 esp_err_t，句柄为出参）
esp_err_t power_measure_new_bl0937_device(const power_measure_config_t *config, const power_measure_bl0937_config_t *bl0937_config, power_measure_handle_t *handle);
esp_err_t power_measure_new_bl0942_device(const power_measure_config_t *config, const power_measure_bl0942_config_t *bl0942_config, power_measure_handle_t *handle);
esp_err_t power_measure_new_ina236_device(const power_measure_config_t *config, const power_measure_ina236_config_t *ina236_config, power_measure_handle_t *handle);

// 读取 / 校准 / 释放
esp_err_t power_measure_get_voltage (power_measure_handle_t h, float *voltage);
esp_err_t power_measure_get_current (power_measure_handle_t h, float *current);
esp_err_t power_measure_get_active_power(power_measure_handle_t h, float *power);
esp_err_t power_measure_get_apparent_power(power_measure_handle_t h, float *apparent_power);
esp_err_t power_measure_get_power_factor(power_measure_handle_t h, float *power_factor);
esp_err_t power_measure_get_energy  (power_measure_handle_t h, float *energy);
esp_err_t power_measure_calibrate_voltage(power_measure_handle_t h, float expected_voltage);
esp_err_t power_measure_calibrate_current(power_measure_handle_t h, float expected_current);
esp_err_t power_measure_calibrate_power(power_measure_handle_t h, float expected_power);
esp_err_t power_measure_reset_energy_calculation(power_measure_handle_t h);
esp_err_t power_measure_delete(power_measure_handle_t handle);
```

> 各后端 config 结构体（含引脚/总线/地址/校准系数）以 `power_measure_bl0937.h` / `power_measure_bl0942.h` / `power_measure_ina236.h` 为准。

> 详细参数与读取函数以各 `power_measure_<chip>.h` 为准。

## usb_stream（UVC + UAC Host）

头文件：`components/usb/usb_stream/include/usb_stream.h`、`libuvc_def.h`（仅 ESP32-S2/S3）

```c
#define FPS2INTERVAL(fps)  (10000000ul / fps)
#define FRAME_RESOLUTION_ANY  __UINT16_MAX__
#define UAC_FREQUENCY_ANY     __UINT32_MAX__
#define UAC_BITS_ANY           __UINT16_MAX__
#define UAC_CH_ANY             0

// 流 id  usb_stream_t：STREAM_UVC, STREAM_UAC_SPK, STREAM_UAC_MIC, STREAM_MAX
// 控制   stream_ctrl_t：CTRL_NONE, CTRL_SUSPEND, CTRL_RESUME, CTRL_UAC_MUTE, CTRL_UAC_VOLUME
// 连接   usb_stream_state_t：STREAM_CONNECTED, STREAM_DISCONNECTED

esp_err_t uvc_streaming_config(const uvc_config_t *config);
esp_err_t uac_streaming_config(const uac_config_t *config);
esp_err_t usb_streaming_start(void);
esp_err_t usb_streaming_stop(void);
esp_err_t usb_streaming_connect_wait(size_t timeout_ms);
esp_err_t usb_streaming_state_register(state_callback_t cb, void *user_ptr);
esp_err_t usb_streaming_control(usb_stream_t stream, stream_ctrl_t ctrl_type, void *ctrl_value);
esp_err_t uac_spk_streaming_write(void *data, size_t data_bytes, size_t timeout_ms);
esp_err_t uac_mic_streaming_read(void *buf, size_t buf_size, size_t *data_bytes, size_t timeout_ms);
esp_err_t uac_frame_size_list_get(usb_stream_t stream, uac_frame_size_t *frame_list, size_t *list_size, size_t *cur_index);
esp_err_t uac_frame_size_reset(usb_stream_t stream, uint8_t ch_num, uint16_t bit_resolution, uint32_t samples_frequence);
esp_err_t uvc_frame_size_list_get(uvc_frame_size_t *frame_list, size_t *list_size, size_t *cur_index);
```

回调签名：`uvc_frame_callback_t`（UVC 帧）、`mic_callback_t(mic_frame_t*, void*)`（mic，禁止阻塞）、`state_callback_t(usb_stream_state_t, void*)`。

## iot_usbh_cdc（USB Host CDC）

头文件：`components/usb/iot_usbh_cdc/include/iot_usbh_cdc.h`、`iot_usbh_cdc_type.h`、`usbh_helper.h`

```c
typedef struct usbh_cdc_port_t *usbh_cdc_port_handle_t;

// 事件 usbh_cdc_device_event_t：CDC_HOST_DEVICE_EVENT_CONNECTED, CDC_HOST_DEVICE_EVENT_DISCONNECTED
// 标志 usbh_cdc_port_flags_t：USBH_CDC_FLAGS_DISABLE_NOTIFICATION (1<<0)

esp_err_t usbh_cdc_driver_install(const usbh_cdc_driver_config_t *config);
esp_err_t usbh_cdc_driver_uninstall(void);
esp_err_t usbh_cdc_register_dev_event_cb  (const usb_device_match_id_t *list, usbh_cdc_device_event_callback_t cb, void *user_data);
esp_err_t usbh_cdc_unregister_dev_event_cb(usbh_cdc_device_event_callback_t cb);
esp_err_t usbh_cdc_port_open (const usbh_cdc_port_config_t *port_config, usbh_cdc_port_handle_t *cdc_port_out);
esp_err_t usbh_cdc_port_close(usbh_cdc_port_handle_t h);
esp_err_t usbh_cdc_write_bytes (usbh_cdc_port_handle_t h, const uint8_t *buf, size_t length, TickType_t ticks_to_wait);
esp_err_t usbh_cdc_read_bytes  (usbh_cdc_port_handle_t h, uint8_t *buf, size_t *length, TickType_t ticks_to_wait);
esp_err_t usbh_cdc_send_custom_request(usbh_cdc_port_handle_t h, uint8_t bmRequestType, uint8_t bRequest,
                                       uint16_t wValue, uint16_t wIndex, uint16_t wLength, uint8_t *data);
```

## iot_servo（LEDC 舵机）

头文件：`components/motor/servo/include/iot_servo.h`

```c
typedef struct {
    gpio_num_t     servo_pin[LEDC_CHANNEL_MAX];
    ledc_channel_t ch[LEDC_CHANNEL_MAX];
} servo_channel_t;

typedef struct {
    uint16_t      max_angle;       // 最大角度
    uint16_t      min_width_us;    // 最小角度脉宽（典型 500）
    uint16_t      max_width_us;    // 最大角度脉宽（典型 2500）
    uint32_t      freq;            // PWM 频率（典型 50）
    ledc_timer_t  timer_number;
    servo_channel_t channels;
    uint8_t       channel_number;  // 须等于实际填入的通道项数
} servo_config_t;

esp_err_t iot_servo_init      (ledc_mode_t speed_mode, const servo_config_t *config);
esp_err_t iot_servo_deinit    (ledc_mode_t speed_mode);
esp_err_t iot_servo_write_angle(ledc_mode_t speed_mode, uint8_t channel, float angle);   // 非线程安全
esp_err_t iot_servo_read_angle(ledc_mode_t speed_mode, uint8_t channel, float *angle);
```

## touch_button_sensor / touch_button（触摸按键）

头文件：`components/touch/touch_button_sensor/include/touch_button_sensor.h`、`components/touch/touch_button/touch_button.h`

```c
// 状态 touch_state_t：TOUCH_STATE_INACTIVE(0), TOUCH_STATE_ACTIVE
typedef struct touch_button_sensor_t *touch_button_handle_t;
typedef void (*touch_cb_t)(touch_button_handle_t handle, uint32_t channel, touch_state_t state, void *cb_arg);

typedef struct {
    uint32_t  channel_num;
    uint32_t *channel_list;
    float    *channel_threshold;     // 0.0~1.0
    uint32_t *channel_gold_value;    // 可选
    uint32_t  debounce_times;
    bool      skip_lowlevel_init;
} touch_button_config_t;

esp_err_t touch_button_sensor_create(touch_button_config_t *config, touch_button_handle_t *handle, touch_cb_t cb, void *cb_arg);
esp_err_t touch_button_sensor_delete(touch_button_handle_t handle);
esp_err_t touch_button_sensor_get_data      (touch_button_handle_t h, uint32_t channel, uint32_t channel_alt, uint32_t *data);
esp_err_t touch_button_sensor_get_state     (touch_button_handle_t h, uint32_t channel, touch_state_t *state);
esp_err_t touch_button_sensor_get_state_bitmap(touch_button_handle_t h, uint32_t channel, uint32_t *bitmap);
esp_err_t touch_button_sensor_handle_events (touch_button_handle_t h);   // 须周期调用

// iot_button 集成（touch_button 组件）
typedef struct {
    int32_t touch_channel;
    float   channel_threshold;       // 0.0~1.0
    bool    skip_lowlevel_init;
} button_touch_config_t;
esp_err_t iot_button_new_touch_button_device(const button_config_t *button_config, const button_touch_config_t *touch_config, button_handle_t *ret_button);
```

> 触摸组件需 IDF >= v5.3；ESP32/S2/S3 抗干扰有限，仅供测试/演示。

## ble_conn_mgr（BLE 连接管理）

头文件：`components/bluetooth/ble_conn_mgr/include/esp_ble_conn_mgr.h`（基于 NimBLE）

```c
ESP_EVENT_DECLARE_BASE(BLE_CONN_MGR_EVENTS);
#define MAX_BLE_DEVNAME_LEN  29
#define BLE_CONN_HANDLE_INVALID  0xFFFF
#define BLE_CONN_MGR_ADDR_STR  "%02x:%02x:%02x:%02x:%02x:%02x"
#define BLE_CONN_MGR_ADDR_HEX(addr) ((addr)[5]),...((addr)[0])

// 配置（device_name/remote_name/broadcast_data/extended_adv/periodic_adv/服务 UUID）
typedef struct {
    uint8_t device_name[MAX_BLE_DEVNAME_LEN];
    uint8_t remote_name[MAX_BLE_DEVNAME_LEN];
    uint8_t broadcast_data[15];                  // BROADCAST_PARAM_LEN
    uint16_t extended_adv_len, periodic_adv_len, extended_adv_rsp_len;
    const char *extended_adv_data, *periodic_adv_data, *extended_adv_rsp_data;
    uint16_t include_service_uuid : 1;
    esp_ble_conn_uuid_type_t adv_uuid_type;      // BLE_CONN_UUID_TYPE_16/32/128
    union { uint16_t adv_uuid16; uint8_t adv_uuid128[16]; };
} esp_ble_conn_config_t;

// 生命周期
esp_err_t esp_ble_conn_init (esp_ble_conn_config_t *config);
esp_err_t esp_ble_conn_start(void);
esp_err_t esp_ble_conn_stop (void);
esp_err_t esp_ble_conn_deinit(void);

// 连接管理（默认 + 多连接 by_handle 系列）
esp_err_t esp_ble_conn_connect   (void);
esp_err_t esp_ble_conn_disconnect(void);
esp_err_t esp_ble_conn_connect_to_addr   (const uint8_t peer_addr[6], uint8_t peer_addr_type);
esp_err_t esp_ble_conn_disconnect_by_handle(uint16_t conn_handle);
esp_err_t esp_ble_conn_get_conn_handle    (uint16_t *out);
esp_err_t esp_ble_conn_get_conn_handle_by_addr(const uint8_t addr[6], uint8_t type, uint16_t *out);
esp_err_t esp_ble_conn_get_mtu(uint16_t *out); esp_err_t esp_ble_conn_mtu_update(uint16_t conn_handle, uint16_t mtu);

// 广播 / 扫描（peripheral / central）
esp_err_t esp_ble_conn_adv_params_set   (const esp_ble_conn_adv_params_t *params);
esp_err_t esp_ble_conn_adv_data_set      (const uint8_t *data, uint16_t len);
esp_err_t esp_ble_conn_periodic_adv_data_set(const uint8_t *data, uint16_t len);
esp_err_t esp_ble_conn_adv_start(void); esp_err_t esp_ble_conn_adv_stop(void);
esp_err_t esp_ble_conn_scan_params_set(const esp_ble_conn_scan_params_t *params);
esp_err_t esp_ble_conn_scan_start(void); esp_err_t esp_ble_conn_scan_stop(void);
esp_err_t esp_ble_conn_register_scan_callback(esp_ble_conn_scan_cb_t cb, void *arg);
esp_err_t esp_ble_conn_parse_adv_data(const uint8_t *adv, uint8_t len, uint8_t ad_type,
                                      const uint8_t **out_data, uint8_t *out_len);

// GATT 服务 / 数据（默认 + by_handle）
esp_err_t esp_ble_conn_add_svc   (const esp_ble_conn_svc_t *svc);
esp_err_t esp_ble_conn_remove_svc(const esp_ble_conn_svc_t *svc);
esp_err_t esp_ble_conn_notify (const esp_ble_conn_data_t *inbuff);
esp_err_t esp_ble_conn_read   (esp_ble_conn_data_t *outbuf);
esp_err_t esp_ble_conn_write  (const esp_ble_conn_data_t *inbuff);
esp_err_t esp_ble_conn_subscribe(esp_ble_conn_desc_t desc, const esp_ble_conn_data_t *inbuff);
// by_handle 变体：notify / read / write / subscribe 多连接版（首参 conn_handle）

// L2CAP CoC
esp_err_t esp_ble_conn_l2cap_coc_mem_init(void);
esp_err_t esp_ble_conn_l2cap_coc_create_server(uint16_t psm, uint16_t mtu, esp_ble_conn_l2cap_coc_event_cb_t cb, void *arg);
esp_err_t esp_ble_conn_l2cap_coc_connect(uint16_t conn_handle, uint16_t psm, uint16_t mtu, uint16_t sdu_size, esp_ble_conn_l2cap_coc_event_cb_t cb, void *arg);
esp_err_t esp_ble_conn_l2cap_coc_send(esp_ble_conn_l2cap_coc_chan_t chan, const esp_ble_conn_l2cap_coc_sdu_t *sdu);
esp_err_t esp_ble_conn_l2cap_coc_accept     (uint16_t conn_handle, uint16_t peer_sdu_size, esp_ble_conn_l2cap_coc_chan_t chan);
esp_err_t esp_ble_conn_l2cap_coc_recv_ready (esp_ble_conn_l2cap_coc_chan_t chan, uint16_t sdu_size);
esp_err_t esp_ble_conn_l2cap_coc_disconnect (esp_ble_conn_l2cap_coc_chan_t chan);
```

事件 `esp_ble_conn_event_t`：`STARTED`/`STOPPED`/`CONNECTED`/`DISCONNECTED`/`DATA_RECEIVE`/`DISC_COMPLETE`/`PERIODIC_REPORT`/`PERIODIC_SYNC_LOST`/`PERIODIC_SYNC`/`CCCD_UPDATE`/`MTU`/`CONN_PARAM_UPDATE`/`SCAN_RESULT`/`ENC_CHANGE`/`PASSKEY_ACTION`。event_data 为 `esp_ble_conn_event_data_t *`（联合体按 id 取成员）。`DATA_RECEIVE.data` 是堆分配的，处理完须 `free()`。

## ble_profiles — esp_ble_ota_raw（OTA 固件升级）

头文件：`components/bluetooth/ble_profiles/esp/ble_ota_raw/include/esp_ble_ota_raw.h`（依赖 ble_conn_mgr，服务 UUID `0x8018`）

```c
typedef void (*esp_ble_ota_raw_recv_fw_cb_t)(uint8_t *buf, uint32_t length);  // 一个扇区(4096B)通过 CRC 后回调
typedef esp_err_t (*esp_ble_ota_raw_ota_begin_cb_t)(uint32_t image_size_bytes);

esp_err_t esp_ble_ota_raw_init(void);                              // 注册 OTA + DIS 服务
esp_err_t esp_ble_ota_raw_deinit(void);
esp_err_t esp_ble_ota_raw_recv_fw_data_callback(esp_ble_ota_raw_recv_fw_cb_t cb);
void esp_ble_ota_raw_set_ota_begin_cb(esp_ble_ota_raw_ota_begin_cb_t cb);
void esp_ble_ota_raw_set_sector_send_window_for_ringbuf(uint32_t ringbuf_capacity_bytes);
uint32_t esp_ble_ota_raw_get_fw_length(void);                      // START 前返回 UINT32_MAX
```

> 应用须自做 `esp_ble_conn_init` / `start`；OTA 写 flash ��示例辅助 API（`ble_ota_raw_ringbuf_init` / `ble_ota_raw_task_init`，见 `examples/bluetooth/ble_profiles/ble_ota/main/include/ble_ota_raw.h`）配合 `esp_ota_*` 完成。需自定义 `partitions.csv` 含 `ota_0`/`ota_1`/`otadata`，`CONFIG_BLE_OTA_RAW_PROFILE=y`，`CONFIG_BT_NIMBLE_ATT_PREFERRED_MTU=498`。

## ble_profiles — esp_ble_htp（健康体温计）

头文件：`components/bluetooth/ble_profiles/std/ble_htp/include/esp_htp.h`（服务 UUID `0x1809`）

```c
ESP_EVENT_DECLARE_BASE(BLE_HTP_EVENTS);
#define BLE_HTP_UUID16                              0x1809
#define BLE_HTP_CHR_UUID16_TEMPERATURE_MEASUREMENT  0x2A1C
#define BLE_HTP_CHR_UUID16_INTERMEDIATE_TEMPERATURE 0x2A1E
#define BLE_HTP_CHR_UUID16_MEASUREMENT_INTERVAL     0x2A21

typedef struct {
    struct { uint8_t temperature_unit:1, time_stamp:1, temperature_type:1, reserved:5; } flags;
    union { uint32_t celsius; uint32_t fahrenheit; } temperature;
    struct { uint16_t year; uint8_t month,day,hours,minutes,seconds; } __attribute__((packed)) timestamp;
    uint8_t location;
} __attribute__((packed)) esp_ble_htp_data_t;

esp_err_t esp_ble_htp_init(void);
esp_err_t esp_ble_htp_deinit(void);
esp_err_t esp_ble_htp_get_temp_type(uint8_t *temp_type);
esp_err_t esp_ble_htp_get_measurement_interval(uint16_t *interval_val);
esp_err_t esp_ble_htp_set_measurement_interval(uint16_t interval_val);
```

事件 id 用 `BLE_HTP_CHR_UUID16_TEMPERATURE_MEASUREMENT`(0x2A1C) / `BLE_HTP_CHR_UUID16_INTERMEDIATE_TEMPERATURE`(0x2A1E)，event_data 为 `esp_ble_htp_data_t *`。

## bthome_v2（BTHome 协议，Home Assistant 集成）

头文件：`components/bluetooth/ble_adv/bthome/include/bthome_v2.h`

```c
typedef struct bthome_t *bthome_handle_t;
typedef void (*bthome_store_func_t)(bthome_handle_t, const char *key, const uint8_t *data, uint8_t len);
typedef void (*bthome_load_func_t )(bthome_handle_t, const char *key, uint8_t *data, uint8_t len);
typedef struct { bthome_store_func_t store; bthome_load_func_t load; } bthome_callbacks_t;
typedef struct { uint8_t id, len; uint8_t *data; } bthome_report_t;
typedef struct { uint8_t num_reports; bthome_report_t report[BTHOME_REPORTS_MAX]; } bthome_reports_t;   // BTHOME_REPORTS_MAX=10
typedef union { struct __attribute__((packed)) { uint8_t encryption_flag:1, reserved1:1, trigger_based_flag:1, reserved2:2, bthome_version:3; } bit; uint8_t all; } bthome_device_info_t;

esp_err_t bthome_create  (bthome_handle_t *handle);
esp_err_t bthome_delete  (bthome_handle_t handle);
esp_err_t bthome_register_callbacks(bthome_handle_t handle, bthome_callbacks_t *callbacks);
esp_err_t bthome_set_encrypt_key  (bthome_handle_t handle, const uint8_t *key);   // 16 字节
esp_err_t bthome_set_local_mac_addr(bthome_handle_t handle, uint8_t *mac);
esp_err_t bthome_set_peer_mac_addr(bthome_handle_t handle, const uint8_t *mac);
esp_err_t bthome_load_params(bthome_handle_t handle);
void      bthome_free_reports(bthome_reports_t *reports);

// 广播 payload 拼装
uint8_t bthome_payload_add_sensor_data    (uint8_t *buf, uint8_t off, bthome_sensor_id_t     obj_id, uint8_t *data, uint8_t data_len);
uint8_t bthome_payload_adv_add_bin_sensor_data(uint8_t *buf, uint8_t off, bthome_bin_sensor_id_t obj_id, uint8_t data);
uint8_t bthome_payload_adv_add_evt_data  (uint8_t *buf, uint8_t off, bthome_event_id_t      obj_id, uint8_t *evt, uint8_t evt_size);
uint8_t bthome_make_adv_data(bthome_handle_t handle, uint8_t *buffer, uint8_t *name, uint8_t name_len, bthome_device_info_t info, uint8_t *payload, uint8_t payload_len);

// 接收端解析
bthome_reports_t *bthome_parse_adv_data(bthome_handle_t handle, uint8_t *adv, uint8_t len);
```

枚举：`bthome_sensor_id_t`（TEMPERATURE=0x45、HUMIDITY=0x2E、BATTERY=0x01、CO2=0x12、...）、`bthome_bin_sensor_id_t`（MOTION=0x21、DOOR=0x1A、OCCUPANCY=0x23、...）、`bthome_event_id_t`（BUTTON=0x3A、DIMMER=0x3C）。常量：`MAX_HUE=360`、`MAX_SATURATION=255`、`MAX_BRIGHTNESS=255`、`MAX_INDEX=127`。底层 BLE 收发配套 `ble_hci` 组件（`ble_hci_init`/`ble_hci_set_scan_param`/`ble_hci_set_adv_param` 等）。

## esp_simplefoc（FOC 无刷电机，C++）

头文件：`esp_simplefoc.h`（API 与 Arduino-FOC 一致；依赖 `iqmath` 做闭环加速）

```cpp
BLDCMotor     motor(pp);                       // pp=极对数（极数/2）
BLDCDriver3PWM driver(phA_gpio, phB_gpio, phC_gpio);
BLDCDriver6PWM driver(phA_h, phA_l, phB_h, phB_l, phC_h, phC_l);   // 半桥
AS5600  as5600(i2c_port, sda_gpio, scl_gpio);  // 或 MT6701 / AS5048A

driver.voltage_power_supply = 12;
driver.voltage_limit        = 11;
driver.init(0);                  // MCPWM 芯片：传 timer id
driver.init({ch_r, ch_g, ch_b}); // LEDC 芯片：传 3 个 LEDC channel
motor.linkDriver(&driver);
motor.linkSensor(&as5600);       // 闭环必需

motor.controller = MotionControlType::velocity;   // torque/velocity/angle/velocity_openloop/angle_openloop
motor.PID_velocity.P = 0.9f; motor.PID_velocity.I = 2.2f;
motor.LPF_velocity.Tf = 0.05;
motor.velocity_limit = 200; motor.voltage_limit = 11; motor.voltage_sensor_align = 2;

motor.init();
motor.initFOC();                 // 闭环对齐传感器（开环模式跳过）
while (1) { motor.loopFOC(); motor.move(target_value); vTaskDelay(1); }
```

> `menuconfig` 设 `CONFIG_FREERTOS_HZ=1000`。`CONFIG_SOC_MCPWM_SUPPORTED` 决定走 MCPWM 还是 LEDC。

## led_indicator（RGB / Strips 后端）

头文件：`components/led/led_indicator/include/led_indicator_rgb.h`、`led_indicator_strips.h`、`led_convert.h`、`led_types.h`

```c
// 颜色宏（led_convert.h）
#define MAX_HUE 360
#define MAX_SATURATION 255
#define MAX_BRIGHTNESS 255
#define MAX_INDEX 127                                     // 127 = 所有灯
#define SET_RGB(r,g,b)           ((((r)&0xFF)<<16) | (((g)&0xFF)<<8) | ((b)&0xFF))
#define SET_HSV(h,s,v)           ((((h>360?360:h))&0x1FF)<<16 | ((s)&0xFF)<<8 | ((v)&0xFF))
#define SET_IRGB(index,r,g,b)    ((((index)&0x7F)<<25) | SET_RGB(r,g,b))
#define SET_IHSV(index,h,s,v)    ((((index)&0x7F)<<25) | SET_HSV(h,s,v))
#define INSERT_INDEX(index,brightness) ((((index)&0x7F)<<25) | ((brightness)&0xFF))

// RGB 后端（三通道 PWM LED）
typedef struct {
    bool is_active_level_high; bool timer_inited; ledc_timer_t timer_num;
    int32_t red_gpio_num, green_gpio_num, blue_gpio_num;
    ledc_channel_t red_channel, green_channel, blue_channel;
} led_indicator_rgb_config_t;
esp_err_t led_indicator_new_rgb_device(const led_indicator_config_t *led_config,
                                       const led_indicator_rgb_config_t *rgb_cfg, led_indicator_handle_t *handle);

// Strips 后端（WS2812 等，RMT/SPI 寻址灯带）
typedef enum { LED_STRIP_RMT, LED_STRIP_SPI, LED_STRIP_MAX } led_strip_driver_t;
typedef struct {
    led_strip_config_t led_strip_cfg;          // 来自 espressif/led_strip
    led_strip_driver_t led_strip_driver;
    union { led_strip_rmt_config_t led_strip_rmt_cfg; led_strip_spi_config_t led_strip_spi_cfg; };
} led_indicator_strips_config_t;
esp_err_t led_indicator_new_strips_device(const led_indicator_config_t *led_config,
                                          const led_indicator_strips_config_t *strips_cfg, led_indicator_handle_t *handle);

// 运行时颜色（仅 RGB/Strips 有效）
uint32_t led_indicator_get_hsv(led_indicator_handle_t h);
esp_err_t led_indicator_set_hsv(led_indicator_handle_t h, uint32_t ihsv_value);
uint32_t led_indicator_get_rgb(led_indicator_handle_t h);
esp_err_t led_indicator_set_rgb(led_indicator_handle_t h, uint32_t irgb_value);
```

`blink_step_type_t` 颜色相关动作：`LED_BLINK_RGB`（设 RGB 色）、`LED_BLINK_RGB_RING`（RGB 渐变）、`LED_BLINK_HSV`（设 HSV 色）、`LED_BLINK_HSV_RING`（HSV 渐变）。GPIO 后端无颜色/亮度；LEDC 单通道有亮度无颜色；RGB 三通道有颜色；Strips 还支持 `SET_I*`/`INSERT_INDEX` 控制 index。
