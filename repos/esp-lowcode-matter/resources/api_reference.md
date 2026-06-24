# esp-lowcode-matter API 速查

> 所有签名均取自仓库 `components/*/include` 与 `components/*/*.h`，未在仓库出现的 API 不要使用。

## 1. low_code（核心：事件/特性收发）— `components/low_code/low_code.h`

### 类型

```c
/* 特性 ID */
typedef enum {
    LOW_CODE_FEATURE_ID_UNHANDLED = 0,
    LOW_CODE_FEATURE_ID_POWER = 1001,
    LOW_CODE_FEATURE_ID_BRIGHTNESS = 1002,
    LOW_CODE_FEATURE_ID_COLOR_TEMPERATURE = 1003,
    LOW_CODE_FEATURE_ID_HUE = 1004,
    LOW_CODE_FEATURE_ID_SATURATION = 1005,
    LOW_CODE_FEATURE_ID_TEMPERATURE = 4004,
    LOW_CODE_FEATURE_ID_COOLING_SETPOINT = 4005,
    LOW_CODE_FEATURE_ID_HEATING_SETPOINT = 4006,
    LOW_CODE_FEATURE_ID_TEMPERATURE_SENSOR_VALUE = 5001,
    LOW_CODE_FEATURE_ID_OCCUPANCY_SENSOR_VALUE = 6001,
    LOW_CODE_FEATURE_ID_MAX = UINT32_MAX,
} low_code_feature_id_t;

/* 值类型 */
typedef enum {
    LOW_CODE_VALUE_TYPE_INVALID = 0,
    LOW_CODE_VALUE_TYPE_BOOLEAN,
    LOW_CODE_VALUE_TYPE_INTEGER,
    LOW_CODE_VALUE_TYPE_UNSIGNED_INTEGER,
    LOW_CODE_VALUE_TYPE_FLOAT,
    LOW_CODE_VALUE_TYPE_STRING,
    LOW_CODE_VALUE_TYPE_OCTET_STRING,
    LOW_CODE_VALUE_TYPE_ARRAY,
    LOW_CODE_VALUE_TYPE_CUSTOM,
} low_code_feature_value_type_t;

typedef struct low_code_feature_value {
    low_code_feature_value_type_t type;
    int value_len;
    uint8_t *value;
} low_code_feature_value_t;

typedef struct low_code_feature_details {
    uint16_t endpoint_id;
    low_code_feature_id_t feature_id;
    union {
        struct { uint32_t cluster_id; uint32_t attribute_id; uint32_t command_id; } matter;
        struct { char *name; char *type; } rainmaker;
    } low_level;
} low_code_feature_details_t;

typedef struct low_code_feature_data {
    low_code_feature_details_t details;
    low_code_feature_value_t value;
    void *priv_data;
} low_code_feature_data_t;

/* 事件 */
typedef enum {
    LOW_CODE_EVENT_INVALID = 0,
    LOW_CODE_EVENT_SETUP_MODE_START,
    LOW_CODE_EVENT_SETUP_MODE_END,
    LOW_CODE_EVENT_SETUP_DEVICE_CONNECTED,
    LOW_CODE_EVENT_SETUP_STARTED,
    LOW_CODE_EVENT_SETUP_SUCCESSFUL,
    LOW_CODE_EVENT_SETUP_FAILED,
    LOW_CODE_EVENT_NETWORK_CONNECTED,
    LOW_CODE_EVENT_NETWORK_DISCONNECTED,
    LOW_CODE_EVENT_OTA_STARTED,
    LOW_CODE_EVENT_OTA_STOPPED,
    LOW_CODE_EVENT_READY,
    LOW_CODE_EVENT_IDENTIFICATION_START,
    LOW_CODE_EVENT_IDENTIFICATION_STOP,
    LOW_CODE_EVENT_IDENTIFICATION_BLINK,
    LOW_CODE_EVENT_IDENTIFICATION_BREATHE,
    LOW_CODE_EVENT_IDENTIFICATION_OKAY,
    LOW_CODE_EVENT_IDENTIFICATION_CHANNEL_CHANGE,
    LOW_CODE_EVENT_IDENTIFICATION_FINISH_EFFECT,
    LOW_CODE_EVENT_IDENTIFICATION_STOP_EFFECT,
    LOW_CODE_EVENT_TEST_MODE_LOW_CODE,
    LOW_CODE_EVENT_TEST_MODE_COMMON,
    LOW_CODE_EVENT_TEST_MODE_BLE,
    LOW_CODE_EVENT_TEST_MODE_SNIFFER,
    LOW_CODE_EVENT_FACTORY_RESET,
    LOW_CODE_EVENT_FORCED_ROLLBACK,
    LOW_CODE_EVENT_BLE_ADVERTISE,
} low_code_event_type_t;

typedef struct low_code_event {
    low_code_event_type_t event_type;
    int event_data_size;
    void *event_data;
} low_code_event_t;

typedef int (*low_code_event_callback_t)(low_code_event_t *event);
typedef int (*low_code_feature_update_callback_t)(low_code_feature_data_t *data);
```

### 函数

```c
/* 注册应用回调（feature 下发 / event 下发） */
int low_code_register_callbacks(low_code_feature_update_callback_t feature_update_from_system,
                                low_code_event_callback_t event_from_system);

/* 在 loop() 中拉取消息 */
int low_code_get_feature_update_from_system();
int low_code_get_event_from_system();

/* 主动上报 */
int low_code_feature_update_to_system(low_code_feature_data_t *feature);
int low_code_event_to_system(low_code_event_t *event);

/* 传输层回调（高级，应用一般不用） */
int low_code_register_transport_callbacks(low_code_callback_list_t *callbacks);
int low_code_feature_update_from_transport(low_code_feature_data_t *data);
int low_code_event_from_transport(low_code_event_t *event);
```

### 错误码（low_code.h 自定义）

`ESP_OK=0`、`ESP_FAIL=-1`，及 `ESP_ERR_NO_MEM=0x101`、`_INVALID_ARG=0x102`、`_INVALID_STATE=0x103`、`_INVALID_SIZE=0x104`、`_NOT_FOUND=0x105`、`_NOT_SUPPORTED=0x106`、`_TIMEOUT=0x107` 等。

## 2. system（LP Core 系统工具）— `components/system/system.h`

```c
void system_setup(void);
void system_loop(void);
void system_timer_update(void);   /* 由 system_loop 调用 */

void system_sleep(uint32_t seconds);
void system_delay(uint32_t seconds);
void system_delay_ms(uint32_t ms);
void system_delay_us(uint32_t us);
uint32_t system_get_time(void);   /* 启动以来 ms */

typedef void *system_timer_handle_t;
typedef void (*system_timer_cb_t)(system_timer_handle_t timer_handle, void *user_data);
system_timer_handle_t system_timer_create(system_timer_cb_t callback, void *arg, int timeout_ms, bool periodic);
int system_timer_start(system_timer_handle_t handle);
int system_timer_stop(system_timer_handle_t handle);
int system_timer_delete(system_timer_handle_t handle);

void system_enable_software_interrupt(void);

typedef enum { INPUT, OUTPUT } pin_mode_t;
typedef enum { LOW = 0, HIGH } pin_level_t;
void system_set_pin_mode(int gpio_num, pin_mode_t mode);
void system_digital_write(int gpio_num, pin_level_t level);
int  system_digital_read(int gpio_num);
```

## 3. button — `components/button/button_driver.h`

```c
typedef enum {
    BUTTON_PRESS_DOWN = 0, BUTTON_PRESS_UP, BUTTON_SINGLE_CLICK,
    BUTTON_LONG_PRESS_START, BUTTON_LONG_PRESS_UP, BUTTON_EVENT_MAX,
} button_event_t;

typedef struct {
    uint16_t long_press_time;
    uint16_t short_press_time;
    int gpio_num;
    uint8_t pullup_en:1;
    uint8_t pulldown_en:1;
    uint8_t active_level:1;
} button_config_t;

typedef void (*button_cb_t)(void *button_handle, void *usr_data);
typedef void *button_handle_t;

button_handle_t button_driver_create(const button_config_t *config);
int button_driver_delete(button_handle_t btn_handle);
int button_driver_register_cb(button_handle_t btn_handle, button_event_t event, button_cb_t cb, void *usr_data);
int button_driver_unregister_cb(button_handle_t btn_handle, button_event_t event);
```

## 4. relay — `components/relay/relay_driver.h`

```c
void relay_driver_init(int gpio_num);
void relay_driver_set_power(int gpio_num, bool power);
```

## 5. light — `components/light/light_driver.h` + `color_format.h`

```c
typedef enum { LIGHT_DEVICE_TYPE_LED = 0, LIGHT_DEVICE_TYPE_WS2812 } light_device_type_t;
typedef enum {
    LIGHT_CHANNEL_COMB_INVALID = 0,
    LIGHT_CHANNEL_COMB_1CH_C, LIGHT_CHANNEL_COMB_1CH_W,
    LIGHT_CHANNEL_COMB_2CH_CW, LIGHT_CHANNEL_COMB_3CH_RGB, LIGHT_CHANNEL_COMB_5CH_RGBCW,
} light_channel_comb_t;
typedef enum { LIGHT_WORK_MODE_INVALID, LIGHT_WORK_MODE_COLOR, LIGHT_WORK_MODE_WHITE } light_work_mode_t;

typedef union {
    struct { gpio_num_t red, green, blue, cold, warm; } led_io;
    struct { gpio_num_t ctrl_io; } ws2812_io;
} light_io_conf_t;

typedef struct {
    light_device_type_t device_type;
    light_channel_comb_t channel_comb;
    light_io_conf_t io_conf;
    int min_brightness;
    int max_brightness;
} light_driver_config_t;

typedef enum { LIGHT_EFFECT_INVALID, LIGHT_EFFECT_BLINK, LIGHT_EFFECT_BREATHE } light_effect_type_t;
typedef struct {
    light_effect_type_t type;
    light_work_mode_t mode;
    union { RGB_color_t RGB; uint32_t cct; HS_color_t HS; CW_white_t CW; } color;
    int8_t max_brightness;
    int8_t min_brightness;
} light_effect_config_t;

int  light_driver_init(light_driver_config_t *config);
int  light_driver_set_power(uint8_t val);
int  light_driver_set_brightness(uint8_t val);     /* 0-100 */
int  light_driver_set_hue(uint16_t val);           /* 0-360 */
int  light_driver_set_saturation(uint8_t val);     /* 0-100 */
int  light_driver_set_temperature(uint32_t val);   /* Kelvin */
int  light_driver_set_color_mode(uint8_t val);     /* 1=COLOR, 2=WHITE */
void light_driver_effect_start(light_effect_config_t *effect, int speed_ms, int total_ms);
void light_driver_effect_stop(void);
```

color_format.h：

```c
typedef struct { uint16_t hue; uint8_t saturation; } HS_color_t;
typedef struct { uint8_t cold; uint8_t warm; } CW_white_t;
typedef struct { uint8_t red, green, blue; } RGB_color_t;

void temp_to_hs(uint32_t temperature, HS_color_t *HS);
void rgb2hs(RGB_color_t RGB, HS_color_t *HS);
void temp_to_cw(uint32_t temperature, CW_white_t *CW);
void hsv_to_rgb(HS_color_t HS, uint8_t brightness, RGB_color_t *RGB);
void cw_to_temp(CW_white_t CW, uint32_t* temperature);
void cw_to_hsv(CW_white_t CW, HS_color_t* HS);
```

## 6. temperature_sensor_sht30 — `components/temperature_sensor_sht30/temperature_sensor_sht30.h`

```c
int temperature_sensor_sht30_init(int i2c_port);
int temperature_sensor_sht30_get_celsius(int i2c_port, float *temperature);
```

## 7. occupancy_sensor_ld2420 — `components/occupancy_sensor_ld2420/occupancy_sensor_ld2420.h`

```c
typedef void* occupancy_sensor_ld2420_handle_t;
typedef struct { uart_port_t uart_num; int tx_pin; int rx_pin; int ot_pin; } occupancy_sensor_ld2420_cfg_t;
typedef struct { uint8_t occupied; uint16_t range; } occupancy_sensor_ld2420_normal_mode_data_t;
typedef struct { uint8_t occupied; uint16_t target_distance; uint16_t zone_noise_level[16]; } occupancy_sensor_ld2420_report_mode_data_t;

occupancy_sensor_ld2420_handle_t occupancy_sensor_ld2420_init(occupancy_sensor_ld2420_cfg_t *cfg);
int occupancy_sensor_ld2420_get_firmware_version(handle, char *buffer, size_t size);
int occupancy_sensor_ld2420_set_minimum_distance(handle, uint16_t minimum_distance);
int occupancy_sensor_ld2420_set_maximum_distance(handle, uint16_t maximum_distance);
int occupancy_sensor_ld2420_set_absence_report_delay(handle, uint16_t delay_s);
int occupancy_sensor_ld2420_set_gate_trigger_threshold(handle, uint8_t gate_index, uint16_t threshold);
int occupancy_sensor_ld2420_set_gate_hold_threshold(handle, uint8_t gate_index, uint16_t threshold);
int occupancy_sensor_ld2420_enter_normal_mode(handle);
int occupancy_sensor_ld2420_read_normal_data(handle, occupancy_sensor_ld2420_normal_mode_data_t *data);
int occupancy_sensor_ld2420_enter_report_mode(handle);
int occupancy_sensor_ld2420_read_report_data(handle, occupancy_sensor_ld2420_report_mode_data_t *data);
```

## 8. display_ssd1306 — `components/display_ssd1306/include/display_ssd1306.h`

```c
#define SSD1306_I2C_ADDRESS ((uint8_t)0x3C)
#define SSD1306_WIDTH  128
#define SSD1306_HEIGHT 64
typedef void *display_ssd1306_handle_t;

display_ssd1306_handle_t display_ssd1306_i2c_create(uint16_t dev_addr, int i2c_port);
esp_err_t ssd1306_init(display_ssd1306_handle_t dev);
void ssd1306_delete(display_ssd1306_handle_t dev);

void display_ssd1306_clear_screen(display_ssd1306_handle_t dev, uint8_t chFill);
void display_ssd1306_draw_string(display_ssd1306_handle_t dev, uint8_t x, uint8_t y, const uint8_t *str, uint8_t chSize, uint8_t chMode);
esp_err_t display_ssd1306_refresh_gram(display_ssd1306_handle_t dev);

void ssd1306_draw_char(display_ssd1306_handle_t dev, uint8_t x, uint8_t y, uint8_t ch, uint8_t size, uint8_t mode);
void ssd1306_draw_num(display_ssd1306_handle_t dev, uint8_t x, uint8_t y, uint32_t num, uint8_t len, uint8_t size);
void ssd1306_draw_1616char(...);
void ssd1306_draw_3216char(...);
void ssd1306_draw_bitmap(display_ssd1306_handle_t dev, uint8_t x, uint8_t y, const uint8_t *bmp, uint8_t w, uint8_t h);
void ssd1306_draw_line(display_ssd1306_handle_t dev, int16_t x1, int16_t y1, int16_t x2, int16_t y2);
void ssd1306_fill_point(display_ssd1306_handle_t dev, uint8_t x, uint8_t y, uint8_t point);
void ssd1306_fill_rectangle(display_ssd1306_handle_t dev, uint8_t x1, uint8_t y1, uint8_t x2, uint8_t y2, uint8_t dot);
```

## 9. sw_timer（LP Core 软件定时器，备用）— `components/sw_timer/sw_timer.h`

> 产品代码多直接用 `system_timer_*`（见 §2）。sw_timer 提供等价的周期/单次定时器，`sw_timer_handle_t`、`sw_timer_cb_t(callback, user_data)`。

## 10. 外设驱动（drivers/）

| 驱动 | 头文件示例 | 用途 |
|---|---|---|
| i2c | `i2c_master.h`（`i2c_master_init(port, scl, sda)`） | SHT30 / SSD1306 |
| rmt | （由 light ws2812 内部使用） | WS2812 |
| uart | `uart.h`（`uart_port_t` 等） | LD2420 |

## 11. 温控器（Thermostat）应用层模式 — `products/thermostat/main/app_priv.h`

> 温控器无独立组件，全部 API 复用 `low_code`（§1）的 `low_code_feature_data_t` / `low_code_feature_update_to_system`。以下为 `products/thermostat/main/app_priv.h` 的应用层驱动原型——温控器是**单 endpoint（id=1）多 feature** 模式：`LOW_CODE_FEATURE_ID_TEMPERATURE`(4004)、`LOW_CODE_FEATURE_ID_COOLING_SETPOINT`(4005)、`LOW_CODE_FEATURE_ID_HEATING_SETPOINT`(4006)，值均为带符号 `int16_t`（°C×100，可负，必须用 `LOW_CODE_VALUE_TYPE_INTEGER`）。详见 `recipes/thermostat.md`。

```c
/* 应用层驱动（仓库提供占位实现，需自行补硬件逻辑） */
int app_driver_init();
int app_driver_set_temperature(int16_t temperature);          /* LocalTemperature 下发 */
int app_driver_set_cooling_setpoint(int16_t cooling_setpoint); /* OccupiedCoolingSetpoint 下发 */
int app_driver_set_heating_setpoint(int16_t heating_setpoint); /* OccupiedHeatingSetpoint 下发 */
int app_driver_event_handler(low_code_event_t *event);         /* 全 LOW_CODE_EVENT_* switch */
```

数据模型要点（`products/thermostat/configuration/data_model_wifi.zap`）：endpoint 1 的 endpointType `deviceTypeRef.code = 769`（MA-thermostat），Thermostat cluster code `513`，`FeatureMap` 默认 `3`（Heating + Cooling）；温度属性 `LocalTemperature`(0)、`OccupiedCoolingSetpoint`(17)、`OccupiedHeatingSetpoint`(18)；`product_info.json` 中 `"device_type_id": 769`。
