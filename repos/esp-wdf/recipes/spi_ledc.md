# SPI 主机收发与 LEDC 占空比

> **适用摘要**：WASM 应用通过 `/dev/spi/2` 作 SPI 主机收发（`SPIIOCSCFG` + `SPIIOCEXCHANGE`），或通过 `/dev/ledc/0` 控制 LEDC 通道占空比（`LEDCIOCSCFG` + `LEDCIOCSSETDUTY`）。

## 触发意图

- "SPI 主机通信"
- "LEDC 调光 / PWM"
- "esp-wdf spi device"
- "ledc duty"

## 前置条件

| 条件 | 要求 |
|---|---|
| SPI 头文件 | `"ioctl/esp_spi_ioctl.h"`；Kconfig `CONFIG_SPI_DEVICE_SPI2/SPI3`、`CONFIG_SPI_CS/SCLK/MOSI/MISO_PIN_NUM` |
| LEDC 头文件 | `"ioctl/esp_ledc_ioctl.h"`；Kconfig `CONFIG_LEDC_DEVICE_LEDC0/1/2`、`CONFIG_LEDC_FREQUENCY`、`CONFIG_LEDC_CHANNEL_OUPUT_PIN` |
| 参考 | `examples/peripherals/spi/spi_master_simple/`、`examples/peripherals/ledc/ledc_simple/` |

## 分步说明

### 1. SPI 主机（取自 examples/peripherals/spi/spi_master_simple）

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <stdint.h>
#include <sys/ioctl.h>

#include "sdkconfig.h"
#include "ioctl/esp_spi_ioctl.h"

#ifdef CONFIG_SPI_DEVICE_SPI2
#define SPI_DEVICE "/dev/spi/2"
#elif defined(CONFIG_SPI_DEVICE_SPI3)
#define SPI_DEVICE "/dev/spi/3"
#endif

#define SPI_DEVICE_CLOCK        (1000 * 1000)
#define SPI_DEVICE_CS_PIN       CONFIG_SPI_CS_PIN_NUM
#define SPI_DEVICE_SCLK_PIN     CONFIG_SPI_SCLK_PIN_NUM
#define SPI_DEVICE_MOSI_PIN     CONFIG_SPI_MOSI_PIN_NUM
#define SPI_DEVICE_MISO_PIN     CONFIG_SPI_MISO_PIN_NUM
#define SPI_DEVICE_TEST_COUNT   (10)

int main(void)
{
    int fd, ret, count = 0;
    spi_cfg_t cfg;
    const char device[] = SPI_DEVICE;
    char text[32];
    int text_size = sizeof(text);

    fd = open(device, O_RDWR);
    if (fd < 0) { printf("Open failed, errno=%d.\n", errno); return -1; }

    /* 配置主机 + 模式0 + 引脚 + 时钟 */
    cfg.flags = SPI_MASTER | SPI_MODE_0;
    cfg.cs_pin   = SPI_DEVICE_CS_PIN;
    cfg.sclk_pin = SPI_DEVICE_SCLK_PIN;
    cfg.mosi_pin = SPI_DEVICE_MOSI_PIN;
    cfg.miso_pin = SPI_DEVICE_MISO_PIN;
    cfg.master.clock = SPI_DEVICE_CLOCK;
    ret = ioctl(fd, SPIIOCSCFG, &cfg);
    if (ret < 0) { printf("Configure failed, errno=%d.\n", errno); goto exit; }

    for (int i = 0; i < SPI_DEVICE_TEST_COUNT; i++) {
        spi_ex_msg_t msg;
        snprintf(text, text_size, "SPI Tx %d", count++);

        msg.rx_buffer = NULL;
        msg.tx_buffer = text;
        msg.size = text_size;
        ret = ioctl(fd, SPIIOCEXCHANGE, &msg);   /* 主机收发 */
        if (ret < 0) { printf("Exchange failed, errno=%d.\n", errno); goto exit; }
        usleep(100 * 1000);                       /* 100ms */
    }

exit:
    ret = close(fd);
    return 0;
}
```

### 2. LEDC 调光（取自 examples/peripherals/ledc/ledc_simple）

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <stdint.h>
#include <sys/ioctl.h>

#include "sdkconfig.h"
#include "ioctl/esp_ledc_ioctl.h"

#ifdef CONFIG_LEDC_DEVICE_LEDC0
#define LEDC_DEVICE "/dev/ledc/0"
#elif defined(CONFIG_LEDC_DEVICE_LEDC1)
#define LEDC_DEVICE "/dev/ledc/1"
#elif defined(CONFIG_LEDC_DEVICE_LEDC2)
#define LEDC_DEVICE "/dev/ledc/2"
#endif

#define LEDC_FREQUENCY          CONFIG_LEDC_FREQUENCY
#define LEDC_CHANNEL_NUM        1
#define LEDC_CHANNEL_OUPUT_PIN  CONFIG_LEDC_CHANNEL_OUPUT_PIN
#define LEDC_SIMPLE_TIME        10
#define LEDC_SIMPLE_DUTY_STEP   20

int main(void)
{
    int fd, ret;
    ledc_cfg_t cfg;
    const ledc_channel_cfg_t channel_cfg[LEDC_CHANNEL_NUM] = {
        { .output_pin = LEDC_CHANNEL_OUPUT_PIN, .duty = 0, .phase = 0 }
    };
    const char device[] = LEDC_DEVICE;

    fd = open(device, O_RDWR);
    if (fd < 0) { printf("Open failed, errno=%d.\n", errno); return -1; }

    cfg.frequency = LEDC_FREQUENCY;
    cfg.channel_num = LEDC_CHANNEL_NUM;
    cfg.channel_cfg = channel_cfg;
    ret = ioctl(fd, LEDCIOCSCFG, &cfg);          /* 配置频率与通道 */
    if (ret < 0) { printf("Configure failed, errno=%d.\n", errno); close(fd); return -1; }

    for (int i = 0; i < LEDC_SIMPLE_TIME; i++) {
        uint32_t duty_value = (channel_cfg[0].duty + i * LEDC_SIMPLE_DUTY_STEP) % 100;
        ledc_duty_cfg_t duty = { .channel = 0, .duty = duty_value };
        ret = ioctl(fd, LEDCIOCSSETDUTY, &duty);  /* 设置占空比 */
        if (ret < 0) { printf("Set duty failed, errno=%d.\n", errno); close(fd); return -1; }
        printf("Set LEDC duty to be %d\n", (int)duty_value);
        sleep(1);
    }

    close(fd);
    return 0;
}
```

### 3. 命令速查

| 模块 | 配置命令 | 数据命令 |
|---|---|---|
| SPI | `SPIIOCSCFG`（`spi_cfg_t`：引脚/模式/时钟） | `SPIIOCEXCHANGE`（`spi_ex_msg_t`：tx/rx/size） |
| LEDC | `LEDCIOCSCFG`（`ledc_cfg_t`：频率/通道数组） | `LEDCIOCSSETDUTY`/`LEDCIOCSSETPHASE`/`LEDCIOCSSETFREQ`/`LEDCIOCSPAUSE`/`LEDCIOCSRESUME` |

SPI 配置标志：`SPI_MASTER`、`SPI_MODE_0..3`、`SPI_RX_LSB`、`SPI_TX_LSB`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| SPI 配置失败 | 引脚/模式/时钟未填 | `cfg.flags` 与所有 pin、`master.clock` 赋值 |
| 收不到数据 | `rx_buffer` 为 NULL 或 `size` 为 0 | 读时设 `rx_buffer` 并填 `size` |
| LEDC 不亮 | 占空比为 0 / 引脚错 | 从非 0 占空比起步，核对 `CONFIG_LEDC_CHANNEL_OUPUT_PIN` |
| 调了宿主 `spi_device_*`/`ledc_set_duty` | WASM 不能直接调宿主驱动 | 一律走 `ioctl` |

## 参考

- `examples/peripherals/spi/spi_master_simple/main/spi_master_simple_main.c`
- `examples/peripherals/ledc/ledc_simple/main/ledc_simple_main.c`
- `components/wamr/libc-builtin-extended/include/ioctl/esp_spi_ioctl.h`
- `components/wamr/libc-builtin-extended/include/ioctl/esp_ledc_ioctl.h`
- `resources/api_reference.md` —— 第 3.4、3.5 节
