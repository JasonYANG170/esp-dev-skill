# 主机控制协处理器 GPIO（GPIO Expander）

> **适用摘要**: 通过 ESP-Hosted 链路，从 host 远程配置与读写协处理器的 GPIO（输出电平、输入读取、开漏、上下拉、中断类型）。相当于把协处理器的 IO"扩展"给 host 使用。API 平台无关，亦可用于非 ESP host。

## 触发意图

- "host 控制 slave 的 GPIO"
- "ESP-Hosted GPIO expander"
- "esp_hosted_cp_gpio_set_level"
- "远程读协处理器 IO"
- "GPIO expander example"

## 前置条件

| 条件 | 要求 |
|---|---|
| 链路 | 已 `esp_hosted_init()` + `esp_hosted_connect_to_slave()` 成功 |
| 头文件 | `#include "esp_hosted.h"`（含 `esp_hosted_cp_gpio.h`） |
| 参考例程 | `examples/host_gpio_expander/` |

## 分步说明

### 1. 配置结构体与模式宏（来自 `host/api/include/esp_hosted_cp_gpio.h`）

```c
typedef struct {
    uint64_t pin_bit_mask;   // 位掩码，每位对应一个 GPIO
    uint32_t mode;           // H_CP_GPIO_MODE_*
    uint32_t pull_up_en;     // H_CP_GPIO_PULL_UP / 1
    uint32_t pull_down_en;   // H_CP_GPIO_PULL_DOWN / 0
    uint32_t intr_type;      // 中断类型
} esp_hosted_cp_gpio_config_t;

#define H_CP_GPIO_MODE_DISABLE         (0)
#define H_CP_GPIO_MODE_INPUT           (H_BIT0)
#define H_CP_GPIO_MODE_OUTPUT          (H_BIT1)
#define H_CP_GPIO_MODE_OUTPUT_OD       (H_BIT1 | H_BIT2)
#define H_CP_GPIO_MODE_INPUT_OUTPUT_OD (H_BIT0 | H_BIT1 | H_BIT2)
#define H_CP_GPIO_MODE_INPUT_OUTPUT    (H_BIT0 | H_BIT1)

#define H_CP_GPIO_PULL_UP   (1)
#define H_CP_GPIO_PULL_DOWN (0)
```

### 2. API 一览

```c
esp_err_t esp_hosted_cp_gpio_config(const esp_hosted_cp_gpio_config_t *cfg);
esp_err_t esp_hosted_cp_gpio_reset_pin(uint32_t gpio_num);
esp_err_t esp_hosted_cp_gpio_set_level(uint32_t gpio_num, uint32_t level);
esp_err_t esp_hosted_cp_gpio_get_level(uint32_t gpio_num, int *level);
esp_err_t esp_hosted_cp_gpio_set_direction(uint32_t gpio_num, uint32_t mode);
esp_err_t esp_hosted_cp_gpio_input_enable(uint32_t gpio_num);
esp_err_t esp_hosted_cp_gpio_set_pull_mode(uint32_t gpio_num, uint32_t pull_mode);
```

### 3. 完整示例（改编自 `examples/host_gpio_expander`）

```c
#include "esp_log.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_hosted.h"

static const char *TAG = "gpio_expander";
#define SLAVE_GPIO_PIN 2

static esp_err_t configure_and_test(esp_hosted_cp_gpio_config_t *cfg, const char *name)
{
    ESP_LOGW(TAG, "---- %s ----", name);
    esp_err_t ret = esp_hosted_cp_gpio_config(cfg);
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "GPIO config failed: 0x%x", ret);
        return ret;
    }

    if (cfg->mode == H_CP_GPIO_MODE_OUTPUT ||
        cfg->mode == H_CP_GPIO_MODE_OUTPUT_OD) {
        esp_hosted_cp_gpio_set_level(SLAVE_GPIO_PIN, 0);
        vTaskDelay(pdMS_TO_TICKS(500));
        esp_hosted_cp_gpio_set_level(SLAVE_GPIO_PIN, 1);
        vTaskDelay(pdMS_TO_TICKS(500));
        ESP_LOGI(TAG, "GPIO toggled");
    } else {
        int level = 0;
        esp_hosted_cp_gpio_get_level(SLAVE_GPIO_PIN, &level);
        ESP_LOGI(TAG, "GPIO %d level: %d", SLAVE_GPIO_PIN, level);
    }
    return ESP_OK;
}

void app_main(void)
{
    esp_hosted_cp_gpio_config_t gpio_config = {
        .pin_bit_mask = (1ULL << SLAVE_GPIO_PIN),
    };

    ESP_ERROR_CHECK(esp_hosted_init());
    esp_err_t ret = esp_hosted_connect_to_slave();
    if (ret != ESP_OK) {
        ESP_LOGE(TAG, "connect_to_slave failed: %s", esp_err_to_name(ret));
        return;
    }

    // Demo 1: 推挽输出
    gpio_config.mode = H_CP_GPIO_MODE_OUTPUT;
    gpio_config.pull_up_en = 0;
    gpio_config.pull_down_en = 0;
    configure_and_test(&gpio_config, "Standard Output");

    // Demo 2: 开漏输出 + 上拉
    gpio_config.mode = H_CP_GPIO_MODE_OUTPUT_OD;
    gpio_config.pull_up_en = 1;
    gpio_config.pull_down_en = 0;
    configure_and_test(&gpio_config, "Open-Drain Output");

    // Demo 3: 输入读取
    gpio_config.mode = H_CP_GPIO_MODE_INPUT;
    configure_and_test(&gpio_config, "Input Read");
}
```

> 该例程的目标支持芯片在 `examples/host_gpio_expander/README.md` 中给出。`SLAVE_GPIO_PIN` 必须是协处理器上可用的 GPIO。

### 4. 外部共存（相关但独立的 API）

`host/api/include/esp_hosted_cp_ext_coex.h` 提供 1/2/3/4-wire 外部共存配置（`ESP_HOSTED_EXT_COEX_WIRE_1..4`、leader/follower 角色等），用于协处理器与外部 RF 的共存调度。该 API 受 `CONFIG_ESP_HOSTED_CP_EXT_COEX` 控制。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_hosted_cp_gpio_config` 返回非 OK | 链路未就绪 / 引脚号非法 | 先确认 connect_to_slave 成功；引脚需为协处理器可用 GPIO |
| 读到的电平不变 | 配成了输出却按输入读 | 用 `H_CP_GPIO_MODE_INPUT` 或 `INPUT_OUTPUT` |
| 上拉无效果 | 协处理器该引脚无内部上拉 | 外部加上拉，或改用开漏 + 外部上拉 |
| expander 与外设引脚冲突 | 协处理器该 GPIO 已被传输/RF 占用 | 避开传输与 strapping 引脚 |

## 参考

- `host/api/include/esp_hosted_cp_gpio.h`（全部结构与 API）
- `host/api/include/esp_hosted_cp_ext_coex.h`（外部共存）
- `examples/host_gpio_expander/main/host_gpio_expander_example.c`
- `docs/gpio_expander.md`
