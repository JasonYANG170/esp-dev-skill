# 灯光组件（light_driver：LED/PWM 与 WS2812）

> **适用摘要**: 用 `light_driver_init` 配置 LED(PWM) 或 WS2812，设置通断/亮度/色温/色调/饱和度，并用 blink/breathe 特效做配网指示。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/light_driver.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "控制灯"
- "WS2812 / PWM LED"
- "亮度/色温/色调"
- "配网灯效 blink"
- "light_driver"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `light_driver.h`（带入 `color_format.h`） |
| 组件依赖 | REQUIRES 含 `light` |
| Kconfig | `USE_LIGHT_DEVICE_TYPE_WS2812`（默认）或 `USE_LIGHT_DEVICE_TYPE_LED`（见 `components/light/Kconfig`） |
| 参考产品 | `products/light_cw_pwm`（2CH CW PWM）、`products/light_rgbcw_ws2812`（3CH RGB WS2812）、`products/socket`（WS2812 仅作指示） |

## 分步说明

### 1. 设备/通道/模式枚举（`light_driver.h`）

```c
typedef enum {
    LIGHT_DEVICE_TYPE_LED = 0,   /* PWM LED */
    LIGHT_DEVICE_TYPE_WS2812,
} light_device_type_t;

typedef enum {
    LIGHT_CHANNEL_COMB_INVALID = 0,
    LIGHT_CHANNEL_COMB_1CH_C,    /* 冷白 */
    LIGHT_CHANNEL_COMB_1CH_W,    /* 暖白 */
    LIGHT_CHANNEL_COMB_2CH_CW,   /* 冷暖白 */
    LIGHT_CHANNEL_COMB_3CH_RGB,
    LIGHT_CHANNEL_COMB_5CH_RGBCW,
} light_channel_comb_t;

typedef enum {
    LIGHT_WORK_MODE_INVALID,
    LIGHT_WORK_MODE_COLOR,   /* RGB */
    LIGHT_WORK_MODE_WHITE,   /* CW */
} light_work_mode_t;
```

### 2. 配置结构（union 不可混用）

```c
typedef union {
    struct { gpio_num_t red, green, blue, cold, warm; } led_io;      /* PWM LED */
    struct { gpio_num_t ctrl_io; } ws2812_io;                         /* WS2812 */
} light_io_conf_t;

typedef struct {
    light_device_type_t   device_type;
    light_channel_comb_t  channel_comb;
    light_io_conf_t       io_conf;
    int min_brightness;
    int max_brightness;
} light_driver_config_t;
```

### 3. WS2812（3CH RGB）初始化（取自 `products/socket`/`light_rgbcw_ws2812`）

```cpp
#include <light_driver.h>
#define INDICATOR_GPIO_NUM ((gpio_num_t)8)

light_driver_config_t cfg = {
    .device_type  = LIGHT_DEVICE_TYPE_WS2812,
    .channel_comb = LIGHT_CHANNEL_COMB_3CH_RGB,
    .io_conf = { .ws2812_io = { .ctrl_io = INDICATOR_GPIO_NUM } },
    .min_brightness = 0,
    .max_brightness = 100,
};
light_driver_init(&cfg);
light_driver_set_power(true);
light_driver_set_hue(100);
light_driver_set_saturation(100);
light_driver_set_brightness(100);
```

### 4. PWM LED（2CH CW）初始化（取自 `products/light_cw_pwm`）

```cpp
#define COLD_CHANNEL_IO ((gpio_num_t)4)
#define WARM_CHANNEL_IO ((gpio_num_t)6)

light_driver_config_t cfg = {
    .device_type  = LIGHT_DEVICE_TYPE_LED,
    .channel_comb = LIGHT_CHANNEL_COMB_2CH_CW,
    .io_conf = { .led_io = { .cold = COLD_CHANNEL_IO, .warm = WARM_CHANNEL_IO } },
    .min_brightness = 0,
    .max_brightness = 100,
};
light_driver_init(&cfg);
light_driver_set_temperature(4000);
light_driver_set_brightness(100);
light_driver_set_power(true);
```

### 5. 控制 API

```c
int light_driver_set_power(uint8_t val);            /* 0/1 */
int light_driver_set_brightness(uint8_t val);       /* 0-100 */
int light_driver_set_hue(uint16_t val);             /* 0-360 */
int light_driver_set_saturation(uint8_t val);       /* 0-100 */
int light_driver_set_temperature(uint32_t val);     /* Kelvin */
int light_driver_set_color_mode(uint8_t val);       /* 1=COLOR, 2=WHITE */
```

### 6. Matter 量程换算（取自 `products/light_cw_pwm`/`light_rgbcw_ws2812`）

```cpp
/* 亮度：Matter 0-255 -> driver 0-100 */
int app_driver_set_light_brightness(uint8_t brightness) {
    brightness = brightness * 100 / 255;
    return light_driver_set_brightness(brightness);
}
/* 色温：Matter mireds -> Kelvin */
int app_driver_set_light_temperature(uint16_t temperature) {
    temperature = 1000000 / temperature;
    return light_driver_set_temperature(temperature);
}
```

### 7. 特效（配网指示，取自 `products/socket`）

```c
typedef enum { LIGHT_EFFECT_INVALID, LIGHT_EFFECT_BLINK, LIGHT_EFFECT_BREATHE } light_effect_type_t;

typedef struct {
    light_effect_type_t type;
    light_work_mode_t   mode;            /* COLOR 或 WHITE */
    union { RGB_color_t RGB; uint32_t cct; HS_color_t HS; CW_white_t CW; } color;
    int8_t max_brightness;
    int8_t min_brightness;
} light_effect_config_t;

void light_driver_effect_start(light_effect_config_t *effect, int speed_ms, int total_ms); /* total_ms=-1 无限 */
void light_driver_effect_stop(void);
```

配网开始时闪烁、结束时停止：

```cpp
light_effect_config_t effect_config = {
    .type = LIGHT_EFFECT_BLINK,
    .mode = LIGHT_WORK_MODE_COLOR,   /* WS2812 用 COLOR；PWM CW 用 WHITE */
    .max_brightness = 100,
    .min_brightness = 10
};
/* 在 LOW_CODE_EVENT_SETUP_MODE_START: */
light_driver_effect_start(&effect_config, 2000, 120000);   /* 周期 2s, 持续 120s */
/* 在 LOW_CODE_EVENT_SETUP_MODE_END: */
light_driver_effect_stop();
```

### 8. 颜色结构（`color_format.h`）

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

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 灯不亮 | WS2812 却填了 `led_io` | WS2812 用 `ws2812_io.ctrl_io` |
| 亮度异常 | 没做 0-255→0-100 换算 | `brightness = brightness*100/255` |
| 色温反了 | 直接传 mireds | `temperature = 1000000/temperature` 转 Kelvin |
| 灯效停不掉 | 漏 effect_stop | SETUP_MODE_END 调 `light_driver_effect_stop()` |
| WS2812 闪烁错乱 | 用了 PWM LED 类型 | WS2812 设 `LIGHT_DEVICE_TYPE_WS2812` |
| mode 选错 | 单灯用了 WHITE | WS2812 单指示灯用 `LIGHT_WORK_MODE_COLOR` |

## 参考

- `components/light/light_driver.h`、`components/light/utils/color_format.h`
- `products/light_cw_pwm/main/app_driver.cpp`
- `products/light_rgbcw_ws2812/main/app_driver.cpp`
- `products/socket/main/app_driver.cpp`（WS2812 指示 + 特效）
