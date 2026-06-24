# Zephyr 主机烧录（west module / device tree / prj.conf / esf shell）

> **适用摘要**: 把 `esp-serial-flasher` 作为 Zephyr **west module** 集成，用 device-tree 驱动的 `espressif,esp-loader` 节点描述 UART/复位/BOOT 引脚与波特率，应用代码经 `esp_loader_from_device()` 取 loader，可选启用交互式 `esf` shell。与其它 port 的核心差别：**不手填 port 结构体**——一切配置来自 DTS overlay，连接参数也由 driver 提供。

## 触发意图

- "Zephyr 烧录 ESP"
- "west module esp-serial-flasher"
- "esp_loader_from_device"
- "device tree esp-loader"
- "esf shell"
- "zephyr,esp-loader chosen"

## 前置条件

| 条件 | 要求 |
|---|---|
| 主机 | Zephyr RTOS v4.4.0 + Zephyr SDK v1.0.1（平台 README 标注的测试版本） |
| 工具 | `west`（Zephyr 元工具）、CMake ≥3.22 |
| 目标 | 任意支持的 ESP SoC（经 UART） |
| 参考示例 | `examples/zephyr_example/`（含 README、prj.conf、boards/ overlay、sample.yaml） |
| 参考头 | `port/zephyr_port.h`、`docs/platform-setup.md` |

## 分步说明

### 1. 声明 west module（submanifest）

按 `docs/platform-setup.md` 的 Zephyr Setup 章节，在 `zephyr/submanifest/esf.yaml` 加入：

```yaml
manifest:
  projects:
    - name: esp-serial-flasher
      url: https://github.com/espressif/esp-serial-flasher
      revision: master
      path: modules/lib/esp_serial_flasher # 按需调整
```

然后更新模块：

```bash
west update                    # 全量更新所有模块
# 或只取 esp-serial-flasher：
west update esp-serial-flasher
```

### 2. Device Tree overlay（定义 esp-loader 节点）

Zephyr 的 port 是 **DTS 驱动** 的（`compatible = "espressif,esp-loader"`）。在 board overlay（如 `boards/esp32s3_devkitc_procpu.overlay`）里声明节点，并用 `chosen` 选定活动实例。以下为仓库 overlay 的真实结构：

```dts
/ {
    esp_loader0: esp_loader {
        compatible = "espressif,esp-loader";
        uart             = <&uart1>;
        default-baudrate = <115200>;
        higher-baudrate  = <230400>;
        reset-gpios      = <&gpio0 7 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
        boot-gpios       = <&gpio0 8 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
        sync-timeout-ms  = <100>;
        num-trials       = <10>;
        status = "okay";
    };

    chosen {
        zephyr,esp-loader = &esp_loader0;
    };
};

&uart1 {
    /* UART 引脚映射、current-speed = <115200>; status = "okay"; */
};
```

绑定（`zephyr/dts/bindings/misc/esp-loader/espressif,esp-loader.yaml`）的必填属性：`uart`（phandle）、`reset-gpios`、`boot-gpios`、`default-baudrate`、`higher-baudrate`、`num-trials`、`sync-timeout-ms`。`higher-baudrate = <0>` 表示连接后不提速。

### 3. prj.conf

示例 `prj.conf`（最小集）：

```kconfig
CONFIG_ESP_SERIAL_FLASHER=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=2
CONFIG_GPIO=y
CONFIG_SERIAL=y
CONFIG_CONSOLE=y
CONFIG_CONSOLE_GETCHAR=y
```

库自身 Kconfig（`zephyr/Kconfig`）还提供：
- `CONFIG_ESP_SERIAL_FLASHER_UART_BUFSIZE`（默认 512，UART TX/RX 缓冲）
- `CONFIG_SERIAL_FLASHER_RESET_HOLD_TIME_MS`（默认 100）、`..._BOOT_HOLD_TIME_MS`（默认 50）
- `CONFIG_SERIAL_FLASHER_WRITE_BLOCK_RETRIES`（默认 3）
- `SERIAL_FLASHER_LOG_LEVEL_*` choice（NONE/ERROR/WARN/INFO/DEBUG）

### 4. 应用代码（经 device 取 loader）

Zephyr port 的独特 API（`port/zephyr_port.h`）——**不手动声明 port 结构体**，从 `const struct device *` 取 loader / config / 连接参数：

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <esp_loader.h>
#include <esp_loader_io.h>
#include <zephyr_port.h>

const struct device *dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_esp_loader));

esp_loader_t                    *loader = esp_loader_from_device(dev);
const esp_loader_config_t       *conf   = esp_loader_config_from_device(dev);
const esp_loader_connect_args_t *carg   = esp_loader_connect_args_from_device(dev);

esp_loader_connect(loader, (esp_loader_connect_args_t *)carg);

if (conf->baud_rate_high) {
    esp_loader_change_transmission_rate(loader, conf->baud_rate_high);
}

/* 烧录：esp_loader_flash_start / write / finish ... */

/* 完成后恢复主机默认波特率（可选） */
esp_loader_host_baudrate(loader, conf->baud_rate);
esp_loader_reset_target(loader);
```

`esp_loader_config_t` 字段（来自 DTS）：`uart_dev`、`enable_spec`、`boot_spec`、`baud_rate`、`baud_rate_high`。

### 5. 启用交互式 esf shell

示例 `Kconfig` 定义了 `CONFIG_ESP_SERIAL_FLASHER_SHELL`（`select SHELL`）。在 prj.conf 加一行或命令行传入即可：

```bash
west build -p -b <board> path/to/esp-serial-flasher/examples/zephyr_example \
    -DCONFIG_ESP_SERIAL_FLASHER_SHELL=y
```

启用后控制台出现 `esf` 命令组（来自 README 输出）：

```text
esf - ESP Serial Flasher commands
Subcommands:
  reset     : Reset target
  info      : Show target info
  images    : List available images
  connect   : Connect target
  speed     : Change transmission speed
  flash     : Flash operations
  register  : Register operations
```

shell 模式下 `main.c` 的自动烧录流程被 `#ifndef CONFIG_ESP_SERIAL_FLASHER_SHELL` 跳过，改为等用户在 shell 里交互。

### 6. 构建与烧录

```bash
west init                       # 首次
west update                     # 拉 Zephyr + esp-serial-flasher 模块

# 用 examples/zephyr_example/boards/ 下提供的 board overlay
west build -p -b <supported board from the boards folder> \
    path/to/esp-serial-flasher/examples/zephyr_example

west flash
west espressif monitor          # 或 west build -t guiconfig 调 prj.conf
```

ESP ThreadBR 板（ESP32-S3 主机 + ESP32-H2 目标）有专用 overlay：

```bash
west build -p -b esp_threadbr/esp32s3/procpu \
    ../modules/lib/esp_serial_flasher/examples/zephyr_example
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `DT_CHOSEN(zephyr_esp_loader)` 节点找不到 | overlay 没加 `chosen { zephyr,esp-loader = &esp_loader0; }` | overlay 里加 chosen 节点并 `status = "okay"` |
| 节点 `status = "disabled"` | overlay 模板默认 disabled | `&esp_loader0 { status = "okay"; };` 启用 |
| `esp_loader_from_device` 返回 NULL | device 没 ready / binding 不匹配 | 确认 `compatible = "espressif,esp-loader"` 与绑定文件 |
| west 找不到模块 | submanifest 没加 / 没 `west update` | 加 `zephyr/submanifest/esf.yaml` 后 `west update esp-serial-flasher` |
| `CONFIG_ESP_SERIAL_FLASHER_SHELL` 无效 | 用了示例 Kconfig 之外的名字 | shell 选项在示例 `Kconfig`（非库 `zephyr/Kconfig`）里定义 |
| 连接超时 | DTS 里 UART 引脚/baud 与硬件不符 | 核对 overlay 的 `uart = <&uartN>` 与 `current-speed`、reset/boot-gpios |
| 提速失败（ESP8266） | 8266 不支持改波特率 | `higher-baudrate = <0>` 跳过 `esp_loader_change_transmission_rate` |

## 参考项目

- `examples/zephyr_example/` — 完整 Zephyr 示例（README、prj.conf、Kconfig、boards/ overlay���src/main.c、src/shell.c、sample.yaml）
- `port/zephyr_port.h` — `zephyr_port_t`、`esp_loader_config_t`、`esp_loader_from_device` / `_config_from_device` / `_connect_args_from_device` / `esp_loader_host_baudrate`
- `zephyr/dts/bindings/misc/esp-loader/espressif,esp-loader.yaml` — DTS 绑定（必填属性）
- `zephyr/Kconfig` — 库 Kconfig（`ESP_SERIAL_FLASHER`、`_UART_BUFSIZE`、reset/boot hold time、log level）
- `docs/platform-setup.md` — Zephyr Setup（submanifest、west update）
