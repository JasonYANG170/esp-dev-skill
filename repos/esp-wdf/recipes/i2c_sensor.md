# I2C 传感器读取（BH1750）

> **适用摘要**：WASM 应用通过 `/dev/i2c/0` 设备节点作 I2C 主机，`ioctl(I2CIOCSCFG)` 配置，用 `I2CIOCRDWR`（两步）或 `I2CIOCEXCHANGE`（一步收发）读取 BH1750 光照传感器。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wdf/resources/`, source/examples in `repos/esp-wdf/`, and this recipe path `repos/esp-wdf/recipes/i2c_sensor.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "读 I2C 传感器"
- "BH1750 光照"
- "i2c master wasm"
- "esp-wdf i2c device"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | 同 GPIO + `"ioctl/esp_i2c_ioctl.h"` |
| Kconfig | `CONFIG_I2C_SDA_PIN_NUM`、`CONFIG_I2C_SCL_PIN_NUM`、`CONFIG_BH1750_ADDR`、`CONFIG_BH1750_OPMODE`、`CONFIG_I2C_TEST_COUNT`、可选 `CONFIG_I2C_READ_BH1750_BY_EXCHANGE` |
| 参考 | `examples/peripherals/i2c/i2c_bh1750/` |

## 分步说明

### 1. 应用代码（取自 examples/peripherals/i2c/i2c_bh1750）

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <stdint.h>
#include <sys/ioctl.h>

#include "sdkconfig.h"
#include "ioctl/esp_i2c_ioctl.h"

#define I2C_DEVICE_BASE     "/dev/i2c/0"
#define I2C_DEVICE_CLOCK    (100 * 1000)
#define I2C_DEVICE_SDA_PIN  CONFIG_I2C_SDA_PIN_NUM
#define I2C_DEVICE_SCL_PIN  CONFIG_I2C_SCL_PIN_NUM
#define I2C_BH1750_ADDRESS  CONFIG_BH1750_ADDR
#define I2C_BH1750_OPMODE   CONFIG_BH1750_OPMODE
#define I2C_BH1750_TIME     (30 * 1000)
#define I2C_TEST_COUNT      CONFIG_I2C_TEST_COUNT

void on_init(void)
{
    int fd, ret;
    i2c_cfg_t cfg;
    const char device[] = I2C_DEVICE_BASE;

    fd = open(device, O_WRONLY);
    if (fd < 0) { printf("Opening failed, errno=%d.\n", errno); return; }

    /* 配置主机模式 + SDA/SCL 上拉 + 引脚 + 时钟 */
    cfg.flags   = I2C_MASTER | I2C_SDA_PULLUP | I2C_SCL_PULLUP;
    cfg.sda_pin = I2C_DEVICE_SDA_PIN;
    cfg.scl_pin = I2C_DEVICE_SCL_PIN;
    cfg.master.clock = I2C_DEVICE_CLOCK;
    ret = ioctl(fd, I2CIOCSCFG, &cfg);
    if (ret < 0) { printf("Configure failed, errno=%d.\n", errno); close(fd); return; }

    for (int i = 0; i < I2C_TEST_COUNT; i++) {
        uint8_t data[2];
#ifdef CONFIG_I2C_READ_BH1750_BY_EXCHANGE
        /* 一步收发：先写命令，延时，再读 */
        i2c_ex_msg_t ex_msg;
        uint8_t opmode = I2C_BH1750_OPMODE;
        ex_msg.flags     = I2C_EX_MSG_DELAY_EN;
        ex_msg.addr      = I2C_BH1750_ADDRESS;
        ex_msg.delay_ms  = I2C_BH1750_TIME / 1000;
        ex_msg.tx_buffer = &opmode;
        ex_msg.tx_size   = 1;
        ex_msg.rx_buffer = data;
        ex_msg.rx_size   = 2;
        ret = ioctl(fd, I2CIOCEXCHANGE, &ex_msg);
        if (ret < 0) { printf("Exchange failed, errno=%d.\n", errno); close(fd); return; }
#else
        /* 两步：先写操作模式，等待测量，再读 */
        i2c_msg_t msg;
        data[0] = I2C_BH1750_OPMODE;
        msg.flags  = I2C_MSG_WRITE | I2C_MSG_CHECK_ACK;
        msg.addr   = I2C_BH1750_ADDRESS;
        msg.buffer = data;
        msg.size   = 1;
        ret = ioctl(fd, I2CIOCRDWR, &msg);
        if (ret < 0) { printf("Write failed, errno=%d.\n", errno); close(fd); return; }

        usleep(I2C_BH1750_TIME);          /* BH1750 测量时间 */

        msg.flags  = 0;                    /* 读 */
        msg.addr   = I2C_BH1750_ADDRESS;
        msg.buffer = data;
        msg.size   = 2;
        ret = ioctl(fd, I2CIOCRDWR, &msg);
        if (ret < 0) { printf("Read failed, errno=%d.\n", errno); close(fd); return; }
#endif
        printf("Sensor val: %.02f [Lux].\n", (data[0] << 8 | data[1]) / 1.2);
        sleep(1);
    }

    close(fd);
}

int main(int argc, char *argv[]) { on_init(); return 0; }
```

### 2. I2C 命令速查

| 命令 | 用途 |
|---|---|
| `I2CIOCSCFG` | 设置 `i2c_cfg_t`（引脚、主机/从机、上拉、时钟、地址） |
| `I2CIOCRDWR` | 单条 `i2c_msg_t` 读或写（`flags` 决定方向） |
| `I2CIOCEXCHANGE` | `i2c_ex_msg_t` 一步收发（可带延时） |

`i2c_msg_t` 标志：`I2C_MSG_WRITE`、`I2C_MSG_CHECK_ACK`、`I2C_MSG_NO_START`、`I2C_MSG_NO_END`。`i2c_ex_msg_t` 标志：`I2C_EX_MSG_READ_FIRST`、`I2C_EX_MSG_CHECK_ACK`、`I2C_EX_MSG_DELAY_EN`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 配置失败 errno | 引脚/时钟未填 | `cfg.sda_pin/scl_pin/master.clock` 全部赋值 |
| 读全 0xFF | 未等待测量 / 地址错 | 写命令后 `usleep`；核对 `CONFIG_BH1750_ADDR` |
| 用了宿主 `i2c_master_*` | WASM 不能直接调宿主驱动 | 走 `ioctl(I2CIOCSCFG/I2CIOCRDWR/I2CIOCEXCHANGE)` |
| 数据方向反 | `I2C_MSG_WRITE` 没置位却要写 | 写消息 `flags = I2C_MSG_WRITE \| I2C_MSG_CHECK_ACK` |

## 参考

- `examples/peripherals/i2c/i2c_bh1750/main/i2c_bh1750_main.c`
- `components/wamr/libc-builtin-extended/include/ioctl/esp_i2c_ioctl.h`
- `resources/api_reference.md` —— 第 3.3 节
