# LP Core GPIO（system_* API）与 Arduino 映射

> **适用摘要**: 在 LP Core 上用 `system_set_pin_mode` / `system_digital_write` / `system_digital_read` 操作 GPIO，并提供 Arduino→LowCode 函数映射。

> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/gpio_system.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "lowcode 怎么控制 GPIO"
- "pinMode / digitalWrite 对应什么"
- "system_digital_write"
- "读输入引脚"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `system.h` |
| 参考文档 | `docs/create_product.md`（Arduino 映射表） |

## 分步说明

### API（`system.h`）

```c
typedef enum { INPUT, OUTPUT } pin_mode_t;
typedef enum { LOW = 0, HIGH } pin_level_t;

void system_set_pin_mode(int gpio_num, pin_mode_t mode);
void system_digital_write(int gpio_num, pin_level_t level);
int  system_digital_read(int gpio_num);   /* 返回 1=HIGH, 0=LOW */
```

### Arduino → LowCode 映射（取自 `docs/create_product.md`）

| Arduino | LowCode | 说明 |
|---|---|---|
| `pinMode(pin, mode)` | `system_set_pin_mode(pin, mode)` | 配置 INPUT/OUTPUT |
| `digitalWrite(pin, value)` | `system_digital_write(pin, level)` | 输出 HIGH/LOW |
| `digitalRead(pin)` | `system_digital_read(pin)` | 读电平 |
| `delay(ms)` | `system_delay_ms(ms)` | 毫秒延时 |
| `serial.println("msg")` | `printf("%s: msg\n", TAG)` | 串口输出 |
| `analogWrite(pin, value)` | TODO（仓库未实现） | 见 light_driver PWM |
| `analogRead(pin)` | TODO（仓库未实现） | — |

### 延时与时间 API

```c
void system_delay(uint32_t seconds);
void system_delay_ms(uint32_t ms);
void system_delay_us(uint32_t us);
void system_sleep(uint32_t seconds);   /* 同 system_delay */
uint32_t system_get_time(void);         /* 启动以来毫秒数 */
```

### 示例：控制一个指示 GPIO

```cpp
#include <system.h>

void indicator_init(void) {
    system_set_pin_mode(7, OUTPUT);
}
void indicator_on(void)  { system_digital_write(7, HIGH); }
void indicator_off(void) { system_digital_write(7, LOW); }
```

### 示例：读一个输入引脚

```cpp
int read_button_raw(void) {
    system_set_pin_mode(9, INPUT);
    return system_digital_read(9);   /* 1=HIGH, 0=LOW */
}
```

> 需要按键去抖/长按/单击识别时，优先用 button 组件（见 `recipes/button_driver.md`），不要自己轮询。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 用了 ESP-IDF `gpio_*` | LP Core 上不可用 | 改用 `system_*` |
| 引脚不工作 | 未 set_pin_mode 就读写 | 先配置模式 |
| 期望 PWM/ADC | system_* 仅数字 IO | PWM 用 `light_driver`（LED）；ADC 仓库未暴露 |
| 按键抖动 | 直接 digital_read 自己判 | 用 button_driver 组件 |

## 参考

- `components/system/system.h`
- `docs/create_product.md`（Arduino 映射表）
- `recipes/button_driver.md`
