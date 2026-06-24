# GPIO 外设（VFS + ioctl）

> **适用摘要**：WASM 应用通过设备节点 `/dev/gpio/<pin>` 访问 GPIO：`open` 拿到 fd，`ioctl(GPIOCSCFG)` 配置上下拉，`write` 翻转电平，`close` 释放。

## 触发意图

- "控制 GPIO"
- "WASM 点灯 / 翻转引脚"
- "gpio device node"
- "esp-wdf 外设"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `<fcntl.h>`、`<unistd.h>`、`<errno.h>`、`<stdint.h>`、`<sys/ioctl.h>`、`"ioctl/esp_gpio_ioctl.h"`、`"sdkconfig.h"` |
| Kconfig | `CONFIG_GPIO_SIMPLE_GPIO_PIN_NUM`（目标引脚号） |
| 参考 | `examples/peripherals/gpio/gpio_simple/` |

## 分步说明

### 1. 应用代码（取自 examples/peripherals/gpio/gpio_simple）

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <stdint.h>
#include <sys/ioctl.h>

#include "sdkconfig.h"
#include "ioctl/esp_gpio_ioctl.h"

/* 把宏拼成设备路径 "/dev/gpio/<pin>" */
#define _COMBINE(a, b)      a #b
#define COMBINE(a, b)       _COMBINE(a, b)

#define GPIO_PIN_NUM        CONFIG_GPIO_SIMPLE_GPIO_PIN_NUM
#define GPIO_DEVICE_BASE    "/dev/gpio/"
#define GPIO_DEVICE         COMBINE(GPIO_DEVICE_BASE, GPIO_PIN_NUM)

#define GPIO_TEST_COUNT     10

void on_init(void)
{
    int fd;
    int ret;
    gpioc_cfg_t cfg;
    uint8_t state = 0;
    const char device[] = GPIO_DEVICE;

    fd = open(device, O_WRONLY);
    if (fd < 0) {
        printf("Opening device %s for writing failed, errno=%d.\n", device, errno);
        return;
    }
    printf("Opening device %s for writing OK, fd=%d.\n", device, fd);

    /* 配置上拉 */
    cfg.flags = GPIOC_PULLUP_EN;
    ret = ioctl(fd, GPIOCSCFG, &cfg);
    if (ret < 0) {
        printf("Set GPIO-%d pull-up failed, errno=%d.\n", GPIO_PIN_NUM, errno);
        close(fd);
        return;
    }

    /* 翻转 GPIO_TEST_COUNT * 2 次，1↔0 算一次 */
    for (int i = 0; i < GPIO_TEST_COUNT * 2; i++) {
        state = !state;
        ret = write(fd, &state, 1);          /* 写 1 字节电平 */
        if (ret < 0) {
            printf("Set GPIO-%d to be %d failed, errno=%d.\n", GPIO_PIN_NUM, state, errno);
            close(fd);
            return;
        }
        printf("Set GPIO-%d to be %d OK.\n", GPIO_PIN_NUM, state);
        sleep(1);
    }

    close(fd);
}

int main(int argc, char *argv[])
{
    on_init();
    return 0;
}
```

### 2. GPIO ioctl 命令与标志

| 命令/标志 | 值 | 用途 |
|---|---|---|
| `GPIOCSCFG` | `_GPIOC(0x0001)` | 设置 GPIO 配置 |
| `GPIOC_PULLDOWN_EN` | `1 << 0` | 下拉 |
| `GPIOC_PULLUP_EN` | `1 << 1` | 上拉 |
| `GPIOC_OPENDRAIN_EN` | `1 << 2` | 开漏 |

`gpioc_cfg_t` 含联合体 `flags` / `flags_data`（位域：`pulldown_en`/`pullup_en`/`opendrain_en`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `open` 返回负、errno 报错 | 引脚号未配置 / 节点不存在 | menuconfig 设 `CONFIG_GPIO_SIMPLE_GPIO_PIN_NUM` |
| 调用了 `gpio_set_level` 等宿主 API | WASM 应用不能直接调宿主驱动 | 一律走 `open/ioctl/write/close` |
| 电平不动 | 没配上下拉/未 `write` | 先 `ioctl(GPIOCSCFG)` 再 `write(fd,&state,1)` |
| 漏 `close` | fd 泄漏 | 用完 `close(fd)` |

## 参考

- `examples/peripherals/gpio/gpio_simple/main/gpio_simple_main.c`
- `components/wamr/libc-builtin-extended/include/ioctl/esp_gpio_ioctl.h`
- `resources/api_reference.md` —— 第 3.1、3.2 节
