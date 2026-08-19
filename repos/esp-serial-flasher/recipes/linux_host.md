# Linux 主机烧录（PC / 树莓派，DTR/RTS 或 libgpiod 复位）

> **适用摘要**: 在 Linux 主机（PC、树莓派 4/5、BeagleBone 等）上用 `linux_port` 经 `/dev/ttyUSB*` 或 `/dev/ttyACM*` 烧录 ESP 目标。复位/BOOT 可经 USB-UART 桥的 DTR/RTS 自动复位，或经 libgpiod 控制 GPIO。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/linux_host.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Linux 烧录 ESP"
- "树莓派��录 ESP"
- "PC 烧录 ESP（不用 esptool）"
- "libgpiod 复位"
- "DTR/RTS 自动复位"

## 前置条件

| 条件 | 要求 |
|---|---|
| 系统 | Debian/Ubuntu 或任意 Linux；CMake ≥3.22、gcc/clang |
| 串口权限 | 用户在 `dialout` 组（`/dev/ttyUSB*`）|
| GPIO（可选） | libgpiod ≥2.0（仅 `LINUX_PORT_GPIO=ON` 时） |
| 参考示例 | `examples/linux_example/` |

## 分步说明

### 1. 构建

```bash
cd examples/linux_example
mkdir -p build && cd build

# 方式 A：USB 连接（USB-UART 桥 DTR/RTS 自动复位，无需 libgpiod）
cmake ..

# 方式 B：SBC + libgpiod 控制 reset/boot
cmake -DLINUX_PORT_GPIO=ON ..
make
```

```bash
# 权限
sudo usermod -aG dialout $USER     # 串口访问
sudo usermod -aG gpio $USER        # libgpiod（可选）
# 重新登录生效
```

### 2. 构造 linux_port（DTR/RTS 模式）

```c
#include "esp_loader.h"
#include "linux_port.h"

linux_port_t port = {
    .port.ops   = &linux_uart_ops,
    .device     = "/dev/ttyUSB0",
    .baudrate   = 115200,
    .gpio_mode  = LINUX_GPIO_DTR_RTS,   // USB-UART 桥自动复位
};
```

`linux_gpio_mode_t`（来自 `linux_port.h`）：
- `LINUX_GPIO_NONE` — 手动进下载模式
- `LINUX_GPIO_GPIOD` — libgpiod 字符设备（任意 SBC），填 `gpio_chip_path` / `reset_pin` / `boot_pin`
- `LINUX_GPIO_DTR_RTS` — 标准 esptool 自动复位电路（CP2102/CH340/FT232）；USB JTAG Serial 设备（C3/S3/C6/H2/P4 经内置 USB，呈现为 `/dev/ttyACM*`）会经 sysfs VID/PID 自动检测并透明处理重枚举

### 3. libgpiod 模式（树莓派 GPIO）

```c
linux_port_t port = {
    .port.ops       = &linux_uart_ops,
    .device         = "/dev/serial0",
    .baudrate       = 115200,
    .gpio_mode      = LINUX_GPIO_GPIOD,
    .gpio_chip_path = "/dev/gpiochip0",
    .reset_pin      = 5,     // BCM 引脚号
    .boot_pin       = 6,
};
```

### 4. init_serial + 连接 + 烧录

```c
esp_loader_t loader;
if (esp_loader_init_serial(&loader, &port.port) != ESP_LOADER_SUCCESS) {
    return;
}

esp_loader_connect_args_t args = ESP_LOADER_CONNECT_DEFAULT();
if (esp_loader_connect(&loader, &args) != ESP_LOADER_SUCCESS) {
    return;
}
if (esp_loader_get_target(&loader) != ESP8266_CHIP) {
    esp_loader_change_transmission_rate(&loader, 921600);
}

// 烧录（镜像来源：文件、网络等，只要已知 size）
flash_binary(&loader, bin_data, bin_size, 0x10000);
esp_loader_reset_target(&loader);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 打不开 `/dev/ttyUSB0` | 不在 `dialout` 组 | `usermod -aG dialout` 后重新登录 |
| `LINUX_PORT_GPIO=ON` 链接失败 | 缺 libgpiod ≥2.0 | `apt install libgpiod-dev`（Debian13/Ubuntu24.04+），或源码编译 |
| `/dev/ttyACM*` 复位后消失 | USB JTAG Serial 重枚举 | `LINUX_GPIO_DTR_RTS` 模式自动处理重枚举 |
| GPIO 引脚号错 | 用了物理引脚而非 BCM | libgpiod 用 GPIO chip line 号，核对硬件 |
| 改速率失败（ESP8266） | 8266 不支持 | 跳过 change_transmission_rate |

## 参考

- `examples/linux_example/` — Linux 主机完整示例（含运行参数、接线图、树莓派步骤）
- `port/linux_port.h` — `linux_port_t`、`linux_uart_ops`、`linux_gpio_mode_t`
- `docs/platform-setup.md` — Linux 构建变量（`PORT`、`LINUX_PORT_GPIO`）
- `docs/hardware-connections.md` — UART 接线
