# I2C + GPIO 中断驱动的电容触摸屏读取（TT21100）

> **适用摘要**：WASM 应用同时打开 I2C 设备 `/dev/i2c/0` 与 GPIO 设备 `/dev/gpio/<ready_pin>`，轮询 READY 引脚电平，当 READY 拉低时用两步 I2C 读取（先读 2 字节长度字，再读该长度字节）解析 TT21100 触点/按键报告。与 `i2c_sensor.md`（BH1750，单外设、定长读取）不同，本配方处理“GPIO 数据就绪门控 + 变长结构化多字节报告”这一真实 HMI 输入模式。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wdf/resources/`, source/examples in `repos/esp-wdf/`, and this recipe path `repos/esp-wdf/recipes/i2c_touch_gpio_irq.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "读触摸屏 / 触控芯片"
- "TT21100 触点 + 按键"
- "GPIO READY 中断门控 I2C 读取"
- "esp-box 触摸面板 wasm"
- "两步 I2C 读变长报告"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `<fcntl.h>`、`<unistd.h>`、`<errno.h>`、`<stdint.h>`、`<stdbool.h>`、`<sys/ioctl.h>`、`"sdkconfig.h"`、`"ioctl/esp_i2c_ioctl.h"`、`"ioctl/esp_gpio_ioctl.h"` |
| Kconfig | `CONFIG_I2C_SDA_PIN_NUM`、`CONFIG_I2C_SCL_PIN_NUM`、`CONFIG_TT21100_READY_PIN_NUM`、`CONFIG_I2C_TEST_COUNT`、可选 `CONFIG_I2C_SDA_SCL_PIN_PULLUP` |
| 参考 | `examples/peripherals/i2c/i2c_tt21100/` |

## 分步说明

### 1. 设备结构与报告布局（取自 examples/peripherals/i2c/i2c_tt21100）

TT21100 每次中断在 I2C 上送出一份变长报告：首 2 字节为本份报告总长度，随后是 `report_hdr_t`（length/id/time_stamp），按 `hdr.id` 区分触点报告（`0x1`）与按键报告（`0x3`）。这些 `__attribute__((packed))` 结构体直接覆盖到读回的字节缓冲上：

```c
#define I2C_TT21100_ADDRESS     0x24
#define TT21100_TOUCH_POINT_NUM 2
#define TT21100_BUTTON_NUM      4
#define TT21100_PACK_ID_TOUCH   0x1
#define TT21100_PACK_ID_BUTTON  0x3

typedef struct touch_record {
    uint8_t  type       : 3;
    uint8_t  reserved_0 : 5;
    uint8_t  touch_id   : 5;
    uint8_t  event_id   : 2;
    uint8_t  tip        : 1;
    uint16_t x;
    uint16_t y;
    uint8_t  pressure;
    uint16_t major_axis_length;
    uint8_t  orientation;
} __attribute__((packed)) touch_record_t;

typedef struct report_hdr {
    uint16_t length;
    uint8_t  id;
    uint16_t time_stamp;
} __attribute__((packed)) report_hdr_t;

typedef struct touch_report {
    report_hdr_t  hdr;
    uint8_t  data_num     : 5;   /* 触点个数 */
    uint8_t  large_object : 1;
    uint8_t  reserved_0   : 2;
    uint8_t  noise_efect    : 3;
    uint8_t  reserved_1     : 3;
    uint8_t  report_counter : 2;
    touch_record_t record[0];   /* 柔性数组，data_num 个触点 */
} __attribute__((packed)) touch_report_t;

typedef struct button_report {
    report_hdr_t  hdr;
    union {
        struct { uint8_t button_1:1; uint8_t button_2:1; uint8_t button_3:1; uint8_t button_4:1; } button_data;
        uint8_t button;
    };
    union {
        struct { uint16_t button_1_signal; uint16_t button_2_signal;
                 uint16_t button_3_signal; uint16_t button_4_signal; } button_signal_data;
        uint16_t button_signal[4];
    };
} __attribute__((packed)) button_report_t;

typedef struct tt21100_device {
    int i2c_fd;
    int gpio_fd;
} tt21100_device_t;
```

### 2. 双外设初始化（I2C 主机 + GPIO 上拉输入）

两个外设各自 `open` + `ioctl(...SCFG)`。READY 引脚配 `GPIOC_PULLUP_EN`。初始化末尾先调一次 `tt21100_clear`（见第 4 步）把可能的残留报告读空。

```c
#define I2C_DEVICE              "/dev/i2c/0"
#define I2C_DEVICE_CLOCK        (400 * 1000)
#define I2C_DEVICE_SDA_PIN      CONFIG_I2C_SDA_PIN_NUM
#define I2C_DEVICE_SCL_PIN      CONFIG_I2C_SCL_PIN_NUM

#define _COMBINE(a, b)          a #b
#define COMBINE(a, b)           _COMBINE(a, b)
#define GPIO_PIN_NUM            CONFIG_TT21100_READY_PIN_NUM
#define GPIO_DEVICE_BASE        "/dev/gpio/"
#define GPIO_DEVICE             COMBINE(GPIO_DEVICE_BASE, GPIO_PIN_NUM)

static int tt21100_init(tt21100_device_t *dev)
{
    int ret;
    i2c_cfg_t i2c_cfg;
    gpioc_cfg_t gpio_cfg;

    dev->i2c_fd = open(I2C_DEVICE, O_RDONLY);
    if (dev->i2c_fd < 0) {
        printf("Opening device %s for reading failed, errno=%d.\n", I2C_DEVICE, errno);
        goto errout_open_i2c;
    }

    i2c_cfg.flags   = I2C_MASTER;
#ifdef CONFIG_I2C_SDA_SCL_PIN_PULLUP
    i2c_cfg.flags  |= I2C_SDA_PULLUP | I2C_SCL_PULLUP;
#endif
    i2c_cfg.sda_pin = I2C_DEVICE_SDA_PIN;
    i2c_cfg.scl_pin = I2C_DEVICE_SCL_PIN;
    i2c_cfg.master.clock = I2C_DEVICE_CLOCK;
    ret = ioctl(dev->i2c_fd, I2CIOCSCFG, &i2c_cfg);
    if (ret < 0) {
        printf("Configure I2C failed, errno=%d.\n", errno);
        goto errout_ioctl_i2c;
    }

    dev->gpio_fd = open(GPIO_DEVICE, O_RDONLY);
    if (dev->gpio_fd < 0) {
        printf("Opening device %s for read failed, errno=%d.\n", GPIO_DEVICE, errno);
        goto errout_ioctl_i2c;
    }

    gpio_cfg.flags = GPIOC_PULLUP_EN;
    ret = ioctl(dev->gpio_fd, GPIOCSCFG, &gpio_cfg);
    if (ret < 0) {
        printf("Set GPIO-%d pull-up failed, errno=%d.\n", GPIO_PIN_NUM, errno);
        goto errout_ioctl_gpio;
    }

    ret = tt21100_clear(dev);          /* 读空残留报告，使 READY 回到高 */
    if (ret < 0) {
        goto errout_ioctl_gpio;
    }
    return 0;

errout_ioctl_gpio:
    close(dev->gpio_fd);
errout_ioctl_i2c:
    close(dev->i2c_fd);
errout_open_i2c:
    return -1;
}
```

### 3. 核心读取：GPIO 门控 + 两步变长 I2C 读取

关键模式：先用 `read(gpio_fd, &level, 1)` 读 READY 电平（低有效，`0` 表示有数据）；为低时第一步读 2 字节长度字，第二步按该长度读取报告体，再把缓冲强转为 `report_hdr_t*` 按 `hdr->id` 分派。两步都用 `I2CIOCRDWR`（`flags=0` 表示读）。

```c
static int tt21100_read(tt21100_device_t *dev, int *msg_id, tt21100_input_data_t *input_data)
{
    int ret;
    uint8_t gpio_level;

    ret = read(dev->gpio_fd, &gpio_level, 1);
    if (ret == 1 && gpio_level == 0) {
        i2c_msg_t msg;
        uint16_t length;
        uint8_t buffer[256];
        report_hdr_t *hdr;

        /* 第一步：读 2 字节长度字 */
        msg.flags  = 0;
        msg.addr   = I2C_TT21100_ADDRESS;
        msg.buffer = (uint8_t *)&length;
        msg.size   = sizeof(length);
        ret = ioctl(dev->i2c_fd, I2CIOCRDWR, &msg);
        if (ret < 0) {
            printf("Read data from TT21100 failed, errno=%d.\n", errno);
            return -1;
        }

        /* 第二步：按长度读报告体 */
        msg.flags  = 0;
        msg.addr   = I2C_TT21100_ADDRESS;
        msg.buffer = buffer;
        msg.size   = length;
        ret = ioctl(dev->i2c_fd, I2CIOCRDWR, &msg);
        if (ret < 0) {
            printf("Read data from TT21100 failed, errno=%d.\n", errno);
            return -1;
        }

        hdr = (report_hdr_t *)buffer;
        switch (hdr->id) {
            case TT21100_PACK_ID_TOUCH: {
                touch_report_t *report = (touch_report_t *)buffer;
                if (report->data_num) {
                    touch_record_t *record = report->record;
                    for (int i = 0; i < report->data_num; i++) {
                        input_data->touch[i].valid = true;
                        input_data->touch[i].x = record[i].x;
                        input_data->touch[i].y = record[i].y;
                    }
                    *msg_id = TT21100_PACK_ID_TOUCH;
                    return 1;
                }
            } break;
            case TT21100_PACK_ID_BUTTON: {
                button_report_t *report = (button_report_t *)buffer;
                for (int i = 0; i < 4; i++) {
                    if (report->button & (1 << i)) {
                        input_data->button[i].valid  = true;
                        input_data->button[i].signal = report->button_signal[i];
                    }
                }
                *msg_id = TT21100_PACK_ID_BUTTON;
                return 1;
            } break;
            default:
                break;
        }
    }
    return 0;   /* READY 高 / 未知 id：本次无数据 */
}
```

### 4. 读空残留（清 READY）

开机或重读前芯片可能已拉低 READY。`tt21100_clear` 反复“读 READY → 若为低则读走 256 字节”直到 READY 回高，确保后续读到的都是新事件：

```c
static int tt21100_clear(tt21100_device_t *dev)
{
    int ret;
    uint8_t gpio_level;
    while (1) {
        i2c_msg_t msg;
        uint8_t buffer[256];
        ret = read(dev->gpio_fd, &gpio_level, 1);
        if (ret != 1 || gpio_level != 0) {
            break;
        }
        msg.flags  = 0;
        msg.addr   = I2C_TT21100_ADDRESS;
        msg.buffer = buffer;
        msg.size   = sizeof(buffer);
        ret = ioctl(dev->i2c_fd, I2CIOCRDWR, &msg);
        if (ret < 0) {
            printf("Read data from TT21100 failed, errno=%d.\n", errno);
            return -1;
        }
    }
    return 0;
}
```

### 5. 轮询主循环（on_init 形态）

示例用 App Framework 的 `on_init()` 形态（同时保留一个只调用 `on_init()` 的 `main`）。循环里 `usleep(10ms)` 退避以减少空转；每次成功取到报告就 `printf` 坐标/按键信号：

```c
#define I2C_TEST_COUNT          CONFIG_I2C_TEST_COUNT

void on_init(void)
{
    int ret;
    int test_count;
    tt21100_device_t device;

    ret = tt21100_init(&device);
    if (ret < 0) {
        return;
    }

    printf("TT21100 touch screen is initialized, and start reading touch point information:\n");

    test_count = 0;
    while (test_count < I2C_TEST_COUNT) {
        int msg_id;
        tt21100_input_data_t input_data = { 0 };

        ret = tt21100_read(&device, &msg_id, &input_data);
        if (ret == 1) {
            switch (msg_id) {
                case TT21100_PACK_ID_TOUCH:
                    printf("  X0=%d Y0=%d", input_data.touch[0].x, input_data.touch[0].y);
                    for (int i = 1; i < TT21100_TOUCH_POINT_NUM; i++) {
                        if (input_data.touch[i].valid) {
                            printf(" X%d=%d Y%d=%d", i, input_data.touch[i].x,
                                                     i, input_data.touch[i].y);
                        }
                    }
                    printf("\n");
                    break;
                case TT21100_PACK_ID_BUTTON:
                    for (int i = 0; i < TT21100_BUTTON_NUM; i++) {
                        if (input_data.button[i].valid) {
                            printf(" Button%d=%d", i, input_data.button[i].signal);
                        }
                    }
                    printf("\n");
                    break;
                default:
                    break;
            }
            test_count++;
        } else if (ret == 0) {
            usleep(10 * 1000);          /* 无数据退避 */
        } else {
            break;                       /* I2C 出错退出 */
        }
    }

    close(device.gpio_fd);
    close(device.i2c_fd);
    printf("Test done and de-initialize TT21100 touch screen.\n");
}

int main(int argc, char *argv[])
{
    on_init();
    return 0;
}
```

### 6. Kconfig（示例自带 `main/Kconfig.projbuild`）

| 配置项 | 类型 | 默认 | 含义 |
|---|---|---|---|
| `CONFIG_I2C_SDA_PIN_NUM` | int | 8 | I2C SDA 引脚 |
| `CONFIG_I2C_SCL_PIN_NUM` | int | 18 | I2C SCL 引脚 |
| `CONFIG_TT21100_READY_PIN_NUM` | int | 3 | TT21100 READY/IRQ 引脚（低有效） |
| `CONFIG_I2C_TEST_COUNT` | int | 100 | 读取并打印的次数 |
| `CONFIG_I2C_SDA_SCL_PIN_PULLUP` | bool | n | 是否在 I2C 配置里置 `I2C_SDA_PULLUP\|I2C_SCL_PULLUP` |

### 7. ioctl 命令与标志速查

| 命令/标志 | 值 | 用途 |
|---|---|---|
| `I2CIOCSCFG` | `_I2CC(0x0001)` | 设置 `i2c_cfg_t`（引脚/主机/上拉/时钟） |
| `I2CIOCRDWR` | `_I2CC(0x0002)` | 按 `i2c_msg_t` 读（`flags=0`）或写（`I2C_MSG_WRITE`） |
| `GPIOCSCFG` | `_GPIOC(0x0001)` | 设置 `gpioc_cfg_t`（上下拉/开漏） |
| `I2C_MASTER` | `1 << 0` | 主机模式 |
| `I2C_SDA_PULLUP` / `I2C_SCL_PULLUP` | `1<<1` / `1<<2` | SDA/SCL 内部上拉 |
| `GPIOC_PULLUP_EN` | `1 << 1` | GPIO 上拉（用于 READY 引脚） |

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `tt21100_read` 永不返回 1 | READY 引脚号/接线错，或未配 `GPIOC_PULLUP_EN` | menuconfig 核对 `CONFIG_TT21100_READY_PIN_NUM`；初始化里 `ioctl(gpio_fd, GPIOCSCFG, &cfg)` |
| 只读到长度字、第二步读失败 | 第一步读到的 `length` 异常大（>256） | 检查 I2C 地址（`0x24`）与时钟（示例用 400kHz）；缓冲固定 256 字节，异常 length 多为线路干扰 |
| 读到的坐标/按键乱跳 | 残留报告未清空 | 初始化与重读前先 `tt21100_clear` 把 READY 读回高 |
| 配置失败 errno | `i2c_cfg` 字段未填全 | `sda_pin/scl_pin/master.clock` 与 `flags=I2C_MASTER` 必须齐备 |
| READY 一直为低、循环不退 | 轮询间隔过密 / 硬件未接上拉 | 无数据时 `usleep(10*1000)` 退避；确认 READY 引脚物理上拉 |
| 调用宿主 `i2c_master_*` / `gpio_set_level` | WASM 沙箱不能直接调宿主驱动 | 一律走 `open` + `ioctl(I2CIOCSCFG/I2CIOCRDWR)` + `read/write` |

## 参考

- `examples/peripherals/i2c/i2c_tt21100/main/i2c_tt21100_main.c`
- `examples/peripherals/i2c/i2c_tt21100/main/Kconfig.projbuild`
- `components/wamr/libc-builtin-extended/include/ioctl/esp_i2c_ioctl.h`
- `components/wamr/libc-builtin-extended/include/ioctl/esp_gpio_ioctl.h`
- `resources/api_reference.md` —— 第 3.1、3.2、3.3 节
