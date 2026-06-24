# SSD1306/SSD1315 OLED 显示（I2C）

> **适用摘要**: 用 `display_ssd1306_i2c_create` 创建 OLED 句柄，用 `display_ssd1306_draw_string` / `display_ssd1306_refresh_gram` / `display_ssd1306_clear_screen` 显示文本，结合事件回调显示配网状态。

## 触发意图

- "加 OLED 显示"
- "SSD1306 / SSD1315"
- "显示温度/状态"
- "display_ssd1306"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `display_ssd1306.h`（带入 `display_ssd1306_fonts.h`） |
| 组件依赖 | REQUIRES 含 `display_ssd1306` |
| I2C | 先 `i2c_master_init` |
| 参考产品 | `products/temperature_sensor_with_display`（用 SSD1315，由 SSD1306 驱动通用支持） |

## 分步说明

### 1. 关键宏与句柄（`display_ssd1306.h`）

```c
#define SSD1306_I2C_ADDRESS ((uint8_t)0x3C)
#define SSD1306_WIDTH  128
#define SSD1306_HEIGHT 64
typedef void *display_ssd1306_handle_t;
```

### 2. API（节选）

```c
display_ssd1306_handle_t display_ssd1306_i2c_create(uint16_t dev_addr, int i2c_port);
esp_err_t ssd1306_init(display_ssd1306_handle_t dev);
void ssd1306_delete(display_ssd1306_handle_t dev);

void display_ssd1306_clear_screen(display_ssd1306_handle_t dev, uint8_t chFill);
void display_ssd1306_draw_string(display_ssd1306_handle_t dev, uint8_t x, uint8_t y,
                                 const uint8_t *pchString, uint8_t chSize, uint8_t chMode);
esp_err_t display_ssd1306_refresh_gram(display_ssd1306_handle_t dev);

void ssd1306_draw_char(...);
void ssd1306_draw_num(...);
void ssd1306_draw_line(...);
void ssd1306_draw_bitmap(...);
void ssd1306_fill_point(...);
void ssd1306_fill_rectangle(...);
```

### 3. 初始化（取自 `products/temperature_sensor_with_display`）

```cpp
#include <i2c_master.h>
#include <display_ssd1306.h>

#define I2C_PORT   I2C_NUM_0
#define I2C_SCL_IO (gpio_num_t)1
#define I2C_SDA_IO (gpio_num_t)2

static display_ssd1306_handle_t ssd1306_handle;

i2c_master_init(I2C_PORT, I2C_SCL_IO, I2C_SDA_IO);
ssd1306_handle = display_ssd1306_i2c_create(SSD1306_I2C_ADDRESS, I2C_PORT);
if (!ssd1306_handle) { printf("%s: ssd1306 create failed\n", TAG); return -1; }
```

### 4. 显示字符串（清屏→绘制→刷新）

```cpp
static void app_driver_update_display(char *msg)
{
    display_ssd1306_clear_screen(ssd1306_handle, 0x00);

    char *newline_ptr = strchr(msg, '\n');
    if (newline_ptr != NULL) {
        size_t len = newline_ptr - msg;
        char line_1[len + 1];
        strncpy(line_1, msg, len);
        line_1[len] = '\0';
        char *line_2 = newline_ptr + 1;
        display_ssd1306_draw_string(ssd1306_handle, 40, 35, (const uint8_t *)line_1, 14, 1);
        if (*line_2 != '\0') {
            display_ssd1306_draw_string(ssd1306_handle, 40, 50, (const uint8_t *)line_2, 14, 1);
        }
    } else {
        display_ssd1306_draw_string(ssd1306_handle, 40, 35, (const uint8_t *)msg, 14, 1);
    }
    display_ssd1306_refresh_gram(ssd1306_handle);   /* 必须 refresh 才显示 */
}
```

### 5. 显示温度（与周期上报回调结合）

```cpp
void app_driver_read_and_report_feature(system_timer_handle_t timer_handle, void *user_data)
{
    float temperature = 0.0;
    char temperature_str[10] = {0};
    temperature_sensor_sht30_get_celsius(I2C_PORT, &temperature);

    sprintf(temperature_str, "%d.%02d C", (int)temperature, ((int)(temperature * 100) % 100));
    app_driver_update_display((char*)temperature_str);
    system_delay_ms(100);
    app_driver_report_temperature(temperature);

    /* 配网时在温度与 "Setup Mode" 间交替显示 */
    if (setup_started) {
        system_delay_ms(1000);
        app_driver_update_display((char*)"Setup\nMode");
    }
}
```

### 6. 配合事件回调显示状态

```cpp
case LOW_CODE_EVENT_SETUP_SUCCESSFUL:
    setup_started = false;
    app_driver_update_display((char*)"Setup\nSuccess");
    system_delay(2);
    break;
case LOW_CODE_EVENT_SETUP_FAILED:
    setup_started = false;
    app_driver_update_display((char*)"Setup\nFailed");
    system_delay(2);
    break;
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 屏幕不显示 | 没调 `refresh_gram` | draw 后必须 refresh |
| 显示旧内容 | 没清屏 | 先 `clear_screen(0x00)` |
| I2C 地址错 | 用了 0x3D | 默认 `SSD1306_I2C_ADDRESS=0x3C` |
| SSD1315 不工作 | 以为不支持 | 该驱动可通用驱动 SSD1315 |
| 字符大小/位置乱 | chSize/坐标不当 | 参考 14 号字、(40,35)/(40,50) 布局 |
| 局部缓冲栈溢出 | VLA 过大 | 控制字符串长度，避免深缓冲 |

## 参考

- `components/display_ssd1306/include/display_ssd1306.h`
- `products/temperature_sensor_with_display/main/app_driver.cpp`
- `recipes/sht30_sensor.md`、`recipes/event_handling.md`
