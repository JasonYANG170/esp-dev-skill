# ESP-WDF 配置项参考（Kconfig）

> 所有配置项均来自 `esp-wdf/Kconfig`、`esp-wdf/components/extended_wasm_app/Kconfig`、`esp-wdf/CMakeLists.txt`、各示例 `sdkconfig.defaults`、`Kconfig.projbuild`。配置通过 `idf.py menuconfig` 修改，或在示例的 `sdkconfig.defaults` 中预置。

## 一、编译器选项（Kconfig → "Compiler options"）

| 配置项 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `COMPILER_OPTIMIZATION_DEFAULT` | choice | 选中 | `-Og` 调试优化 |
| `COMPILER_OPTIMIZATION_SIZE` | choice | | `-Os` 优化体积 |
| `COMPILER_OPTIMIZATION_PERF` | choice | | `-O3` 性能优化 |
| `COMPILER_OPTIMIZATION_NONE` | choice | | `-O0` 不优化 |
| `COMPILER_WASI_NO_USE_STDLIB` | bool | `y` | 链接 `-nostdlib`，需 WAMR libc-builtin；sockets 示例设为 `n`（用 libc-wasi） |
| `COMPILER_WASI_STACK_SIZE` | int | `8192` | 辅助栈大小（线性内存内），须小于初始内存 |
| `COMPILER_WASI_USE_SHARED_MEMORY` | bool | `n` | 启用共享线性内存；sockets 示例设为 `y` |
| `COMPILER_WASI_INITIAL_MEMORY` | int | `65536` | 线性内存初始大小，须为 65536 的整数倍 |
| `COMPILER_WASI_MAX_MEMORY` | int | `65536` | 线性内存最大大小，须为 65536 的整数倍 |

## 二、WAMR App Framework（构建系统根据以下项导出符号，见 CMakeLists.txt）

| 配置项 | 导出符号 | 说明 |
|---|---|---|
| `CONFIG_WAMR_APP_FRAMEWORK=y` | `on_init`、`on_destroy` | 启用 App Framework，应用实现这两个回调 |
| `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_REQUEST_RESPONSE=y` | `on_request`、`on_response` | 启用请求/响应原生回调导出 |
| `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER=y` | `on_timer_callback` | 启用定时器原生回调导出（timer 示例需要） |

> 示例 `examples/simple/timer/sdkconfig.defaults` 仅含两行：`CONFIG_WAMR_APP_FRAMEWORK=y` 与 `CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER=y`。

## 三、扩展 WASM 应用（components/extended_wasm_app/Kconfig）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `WDF_EXT_WASM_APP_MQTT` | `y` | 启用 ESP-MQTT 适配 |
| `WDF_EXT_WASM_APP_LVGL` | `y` | 启用 LVGL 适配 |
| `USING_CUSTOMER_LV_CONF_H` | `n` | 启用后使用用户自定义 `lv_conf.h`，否则用框架内置 |
| `LV_COLOR_16_SWAP` | `y` | RGB565 字节序交换（esp-box 启用；esp32_p4_function_ev_board 关闭） |
| `WDF_EXT_WASM_APP_HTTP_CLIENT` | `y` | 启用 ESP HTTP Client 适配 |
| `WDF_EXT_WASM_APP_WIFI_PROVISIONING` | `y` | 启用 Wi-Fi 配网适配 |
| `WDF_EXT_WASM_APP_RMAKER` | `y` | 启用 ESP RainMaker 适配 |

## 四、外设示例 Kconfig.projbuild（每示例自定义配置项）

### GPIO（examples/peripherals/gpio/gpio_simple）
- `CONFIG_GPIO_SIMPLE_GPIO_PIN_NUM` —— 目标 GPIO 引脚号。

### UART（examples/peripherals/uart/uart_simple）
- `CONFIG_UART_DEVICE_UART0` —— 使用 `/dev/uart/0`。
- `CONFIG_UART_DEVICE_USB_SERIAL_JTAG_CONTROLLER` —— 使用 `/dev/usbserjtag`。

### I2C（examples/peripherals/i2c/i2c_bh1750）
- `CONFIG_I2C_SDA_PIN_NUM`、`CONFIG_I2C_SCL_PIN_NUM` —— SDA/SCL 引脚。
- `CONFIG_BH1750_ADDR` —— BH1750 器件地址。
- `CONFIG_BH1750_OPMODE` —— BH1750 操作模式。
- `CONFIG_I2C_TEST_COUNT` —— 测试循环次数。
- `CONFIG_I2C_READ_BH1750_BY_EXCHANGE` —— 使用 `I2CIOCEXCHANGE` 读取（否则用 `I2CIOCRDWR` 两步）。

### SPI（examples/peripherals/spi/spi_master_simple）
- `CONFIG_SPI_DEVICE_SPI2` → `/dev/spi/2`；`CONFIG_SPI_DEVICE_SPI3` → `/dev/spi/3`。
- `CONFIG_SPI_CS_PIN_NUM`、`CONFIG_SPI_SCLK_PIN_NUM`、`CONFIG_SPI_MOSI_PIN_NUM`、`CONFIG_SPI_MISO_PIN_NUM`。

### LEDC（examples/peripherals/ledc/ledc_simple）
- `CONFIG_LEDC_DEVICE_LEDC0/1/2` → `/dev/ledc/0/1/2`。
- `CONFIG_LEDC_FREQUENCY`、`CONFIG_LEDC_CHANNEL_OUPUT_PIN`。

## 五、典型 sdkconfig.defaults 模式

各示例通过 `sdkconfig.defaults` 预置关键开关，复制示例后通常无需 menuconfig：

| 示例 | sdkconfig.defaults 内容（要点） |
|---|---|
| `examples/simple/timer` | `CONFIG_WAMR_APP_FRAMEWORK=y`、`CONFIG_WAMR_APP_FRAMEWORK_EXPORT_TIMER=y` |
| `examples/simple/event_*` | `CONFIG_WAMR_APP_FRAMEWORK=y`（+ 对应 EXPORT 项） |
| `examples/simple/request_*` | `CONFIG_WAMR_APP_FRAMEWORK=y`（+ 对应 EXPORT 项） |
| `examples/protocols/sockets/tcp_client` | `CONFIG_WAMR_APP_FRAMEWORK=y`、`CONFIG_COMPILER_WASI_NO_USE_STDLIB=n`、`CONFIG_COMPILER_WASI_USE_SHARED_MEMORY=y` |

> 注意：网络/sockets 类示例必须关闭 `COMPILER_WASI_NO_USE_STDLIB`（设为 `n`）并打开 `COMPILER_WASI_USE_SHARED_MEMORY`（设为 `y`），否则使用 libc-wasi 时 socket 符号缺失。
