# GPIO 数字 I/O 合理性测试

> **适用摘要**: 用 `bist_gpio_output_test()` 验证引脚能被可靠拉高/拉低并回读；用 `bist_gpio_input_test()` 验证输入引脚能读到预期外部电平（IEC 60730 组件 7.1）。包含无效 GPIO 号与各 SoC 引脚映射。

## 触发意图

- "GPIO 自检 / IO 测试"
- "数字 IO 合理性"
- "stuck-at 引脚检测"
- "IEC 60730 7.1"
- "invalid GPIO 处理"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `tests/digital_io_test/main.c` |
| 头文件 | `bist_esp.h`（含 `bist_gpio.h`、`gpio.h`） |
| 配置 | `CONFIG_ESP_BIST_GPIO_TEST=y` |
| 硬件 | 输出测试：引脚悬空或接 LED；输入测试：引脚接已知电平（按键接 GND = 0，接 VDD = 1） |

## 分步说明

### 1. 两个测试函数（来自 `bist_gpio.h`）

```c
// 输出：复位引脚 → 配为输出 → 拉 0 回读 → 拉 1 回读 → 复位
bist_esp_err_t bist_gpio_output_test(gpio_num_t gpio_num);

// 输入：复位引脚 → 配为输入 → 读电平 → 与 expected_level 比对 → 复位
bist_esp_err_t bist_gpio_input_test(gpio_num_t gpio_num, bool expected_level);
```

两者失败均返回 `BIST_ESP_IO_TEST_ERR`；非法 GPIO 号同样返回 `BIST_ESP_IO_TEST_ERR`。

> 测试会**重新配置**引脚，因此必须在应用配置 GPIO **之前**运行（见 `samples/standalone/main.c` 顺序）。

### 2. SoC 引脚映射（来自 `tests/digital_io_test/main.c`）

```c
#define BIST_TEST_GPIO_EXT_OUT_IO   (2)
#define BIST_TEST_GPIO_EXT_IN_IO    (3)
#define BIST_TEST_GPIO_SIGNAL_IDX   (SIG_IN_FUNC97_IDX)

#ifdef SOC_TARGET_ESP32C6
#define BIST_TEST_GPIO_INPUT_IO     (8)
#elif SOC_TARGET_ESP32C5
#define BIST_TEST_GPIO_INPUT_IO     (8)
#elif SOC_TARGET_ESP32C3
#define BIST_TEST_GPIO_INPUT_IO     (9)
#elif SOC_TARGET_ESP32H2
#define BIST_TEST_GPIO_INPUT_IO     (9)
#endif
#define BIST_TEST_GPIO_INPUT_LEVEL  (1)
```

### 3. 单测：正常 + 非法 GPIO（来自 `tests/digital_io_test/main.c`）

```c
#include "bist_esp.h"
#include "unity.h"
#include "hal/gpio_ll.h"

void setUp(void) {}
void tearDown(void) {}

void test_BIST_IO_INVALID_GPIO(void) {
    bist_esp_err_t ret;

    ret = bist_gpio_output_test(SOC_GPIO_PIN_COUNT);     // 越界
    TEST_ASSERT_EQUAL(BIST_ESP_IO_TEST_ERR, ret);
    ret = bist_gpio_input_test(SOC_GPIO_PIN_COUNT, 0);
    TEST_ASSERT_EQUAL(BIST_ESP_IO_TEST_ERR, ret);
    ret = bist_gpio_output_test(-1);                      // 负数
    TEST_ASSERT_EQUAL(BIST_ESP_IO_TEST_ERR, ret);
    ret = bist_gpio_input_test(-1, 0);
    TEST_ASSERT_EQUAL(BIST_ESP_IO_TEST_ERR, ret);
}

void test_BIST_IO_OUTPUT_GPIO(void) {
    TEST_ASSERT_EQUAL(BIST_ESP_OK, bist_gpio_output_test(BIST_TEST_GPIO_EXT_OUT_IO));
}

void test_BIST_IO_INPUT_GPIO(void) {
    TEST_ASSERT_EQUAL(BIST_ESP_OK,
        bist_gpio_input_test(BIST_TEST_GPIO_INPUT_IO, BIST_TEST_GPIO_INPUT_LEVEL));
}

int main(void) {
    UNITY_BEGIN();
    RUN_TEST(test_BIST_IO_INVALID_GPIO);
    RUN_TEST(test_BIST_IO_OUTPUT_GPIO);
    RUN_TEST(test_BIST_IO_INPUT_GPIO);
    return UNITY_END();
}
```

### 4. 集成进 post-boot 自检（来自 `samples/standalone/main.c`）

```c
#define LED_GPIO 7
#define BTN_GPIO 9

static void post_boot_tests(void)
{
    bist_esp_err_t err;

    err = bist_gpio_output_test(LED_GPIO);
    if (err == BIST_ESP_IO_TEST_ERR) {
        ESP_LOGE(TAG, "GPIO output test failed for LED GPIO");
        fail_safe_exit();
    }

    err = bist_gpio_input_test(BTN_GPIO, 1);   // 按键硬件接法决定 expected
    if (err == BIST_ESP_IO_TEST_ERR) {
        ESP_LOGE(TAG, "GPIO input test failed for Button GPIO");
        fail_safe_exit();
    }
}

// 之后再配置应用 GPIO
static void configure_led(void)    { gpio_reset_pin(LED_GPIO); gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT); }
static void configure_button(void) { gpio_set_direction(BTN_GPIO, GPIO_MODE_INPUT); }
```

### 5. `bist.conf`（来自 `tests/digital_io_test/bist.conf`）

```
CONFIG_ESP_BIST_CPU_REG_TEST=n
CONFIG_ESP_BIST_CPU_CSR_REG_TEST=n
CONFIG_ESP_BIST_MEMORY_RAM_TEST=n
CONFIG_ESP_BIST_MEMORY_FLASH_TEST=n
CONFIG_ESP_BIST_STACK_TEST=n
CONFIG_ESP_BIST_CLOCK_TEST=n
CONFIG_ESP_BIST_GPIO_TEST=y
CONFIG_ESP_BIST_PROGRAM_COUNTER_TEST=n
CONFIG_ESP_BIST_WATCHDOG_TEST=n
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 输出测试读回不符 | 引脚被外部强拉（如接按键到 GND 又配输出高） | 测试时引脚悬空或仅接弱负载 |
| 输入测试失败 | 外部电平与 `expected_level` 不符 | 确认硬件：按键接 GND 则 expected=0 |
| 在应用 GPIO 配置后跑测试 | 引脚已被重配 | 测试必须在 `configure_led/button()` 之前 |
| `bist_gpio_output_test(SOC_GPIO_PIN_COUNT)` 不报错？ | 实际会报错 | 库对越界/负数均返回 `BIST_ESP_IO_TEST_ERR` |
| C6 上用 GPIO9 输入失败 | 该引脚已被 strapping/USB 占用 | 按表选 `BIST_TEST_GPIO_INPUT_IO=8` |

## 参考

- `tests/digital_io_test/main.c`、`tests/digital_io_test/bist.conf`
- `src/bist/core/io/include/bist_gpio.h`
- `src/bist/drivers/include/gpio.h`（`gpio_reset_pin` / `gpio_set_direction` / `gpio_set_level` / `gpio_get_level`）
- `src/bist/README.md`（IO Tests 节）
- `samples/standalone/main.c`（调用顺序：测试 → 应用配置）
