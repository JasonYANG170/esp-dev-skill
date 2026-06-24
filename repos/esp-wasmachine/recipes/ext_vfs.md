# Extended VFS（UART / GPIO / I2C / SPI / LEDC）

> **适用摘要**: 启用 Extended VFS，让 WASM 应用通过 `/dev/uart/x` 与 `ioctl` 访问 UART、GPIO、I2C、SPI、LEDC 外设。

## 触发意图

- "WASM 应用操作 GPIO"
- "WASM 里读 I2C/SPI"
- "/dev/uart 怎么用"
- "ext_vfs ioctl 命令"

## 前置条件

| 条件 | 要求 |
|---|---|
| 主开关 | `CONFIG_WASMACHINE_EXT_VFS=y`（默认 y） |
| UART | `CONFIG_WASMACHINE_EXT_VFS_UART=y`（默认 y），且控制台不在 UART（`!ESP_CONSOLE_UART_DEFAULT && !ESP_CONSOLE_UART_CUSTOM`） |
| GPIO/I2C/SPI/LEDC | esp-iot-solution 的 `CONFIG_EXTENDED_VFS_GPIO/_I2C/_SPI/_LEDC`（参考工程默认 `CONFIG_EXTENDED_VFS_SPI=n`） |
| libc | `CONFIG_WASMACHINE_WASM_EXT_NATIVE_LIBC=y`（ioctl 由 libc 注册） |

## 分步说明

### 1. VFS 初始化

`wm_ext_wasm_vfs_init()`（`components/wasmachine_ext_wasm_vfs/src/wm_ext_vfs.c`）在 `wm_wamr_init()` 内被调用：

```c
void wm_ext_wasm_vfs_init(void)
{
#ifdef CONFIG_WASMACHINE_EXT_VFS_UART
    uart_vfs_dev_register();        /* IDF >= 5.3；<5.3 用 esp_vfs_dev_uart_register() */
#endif
#ifdef CONFIG_EXTENDED_VFS
    ext_vfs_init();                 /* GPIO/I2C/SPI/LEDC 来自 esp-iot-solution */
#endif
}
```

### 2. UART 设备节点

Kconfig 帮助文本（`components/wasmachine_ext_wasm_vfs/Kconfig.wasmachine`）原文要点：
- 启用后注册 `/dev/uart/0`、`/dev/uart/1`、`/dev/uart/2`。
- **UART1、UART2 不会被自动初始化**，所以 `/dev/uart/1`、`/dev/uart/2` 不能直接用——固件需先 `uart_driver_install`。
- 若 UART0/1/2 被用于其他用途（如控制台），对应节点同样不可用。

WASM 应用侧用法：

```c
int fd = open("/dev/uart/1", O_RDWR);
char buf[16];
int n = read(fd, buf, sizeof(buf));
write(fd, "AT\r\n", 4);
close(fd);
```

### 3. 外设 ioctl 命令分发

`wm_ext_wasm_native_libc.c` 的 `ioctl_wrapper` 按 `cmd` 分发（受 `CONFIG_EXTENDED_VFS_*` 守卫）：

| 配置宏 | cmd 值 | 处理函数 |
|---|---|---|
| `CONFIG_EXTENDED_VFS_GPIO` | `GPIOCSCFG` | `wm_ext_wasm_gpio_ioctl` |
| `CONFIG_EXTENDED_VFS_I2C`  | `I2CIOCSCFG`、`I2CIOCRDWR`、`I2CIOCEXCHANGE` | `wm_ext_wasm_i2c_ioctl` |
| `CONFIG_EXTENDED_VFS_SPI`  | `SPIIOCSCFG`、`SPIIOCEXCHANGE` | `wm_ext_wasm_native_spi_ioctl` |
| `CONFIG_EXTENDED_VFS_LEDC` | `LEDCIOCSCFG`、`LEDCIOCSSETFREQ`、`LEDCIOCSSETDUTY`、`LEDCIOCSSETPHASE`、`LEDCIOCSPAUSE`、`LEDCIOCSRESUME` | `wm_ext_wasm_native_ledc_ioctl` |

这些 `cmd` 宏来自 esp-iot-solution 的 `ioctl/esp_*_ioctl.h`（经 `wm_ext_vfs_ioctl.h` include）。

### 4. WASM 应用侧 ioctl（伪代码）

```c
int fd = open("/dev/gpio0", O_RDWR);   /* 节点名取决于 ext_vfs 注册 */
/* 配置/读/写经 ioctl + data_seq 传参（见 recipes/data_sequence.md） */
ioctl(fd, GPIOCSCFG, args);
```

ioctl 的第 3 参数 `va_args` 是 `char *`，内部用 `wm_ext_wasm_native_get_data_seq(exec_env, va_args)` 取出 `data_seq_t` 解包参数（见 `wm_ext_wasm_native_common.h`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `/dev/uart/0` 打不开 | 控制台占用 UART0 | 控制台改 USB/USB-Serial-JTAG，或只用 UART1/2 |
| `/dev/uart/1` 读写无响应 | UART1 未 `uart_driver_install` | 固件先安装 UART1 驱动（Kconfig 帮助明确说明） |
| `ioctl` 报 `EINVAL` | `cmd` 未在守卫分支中 | 开启对应 `CONFIG_EXTENDED_VFS_*` |
| `SPIIOCSCFG` 找不到 | 参考工程默认关 SPI | 改 `CONFIG_EXTENDED_VFS_SPI=y` |
| ioctl 参数乱码 | 未用 data_seq 序列化 | 按 `recipes/data_sequence.md` 打包参数 |

## 参考

- `components/wasmachine_ext_wasm_vfs/src/wm_ext_vfs.c` — `wm_ext_wasm_vfs_init`
- `components/wasmachine_ext_wasm_vfs/include/wm_ext_vfs_ioctl.h` — ioctl 声明
- `components/wasmachine_ext_wasm_vfs/Kconfig.wasmachine` — UART 依赖说明
- `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_libc.c` — `ioctl_wrapper` 分发
- `recipes/native_libc.md` — libc `ioctl` import
- `recipes/data_sequence.md` — ioctl 参数序列化
