# UART 输出与文件系统读写

> **适用摘要**：WASM 应用通过 `/dev/uart/0`（或 `/dev/usbserjtag`）`write` 串口数据；通过 `/storage/<file>` 路径用 `open/write/read/lseek/close` 读写 VFS 文件。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-wdf/resources/`, source/examples in `repos/esp-wdf/`, and this recipe path `repos/esp-wdf/recipes/uart_filesystem.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "WASM 串口输出"
- "读写文件"
- "uart device"
- "storage 文件系统"

## 前置条件

| 条件 | 要求 |
|---|---|
| UART 头文件 | 标准 POSIX；Kconfig `CONFIG_UART_DEVICE_UART0` 或 `CONFIG_UART_DEVICE_USB_SERIAL_JTAG_CONTROLLER` |
| 参考 UART | `examples/peripherals/uart/uart_simple/` |
| 参考 FS | `examples/file_system/` |

## 分步说明

### 1. UART 输出（取自 examples/peripherals/uart/uart_simple）

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>

#include "sdkconfig.h"

#ifdef CONFIG_UART_DEVICE_UART0
#define UART_DEVICE "/dev/uart/0"
#elif defined(CONFIG_UART_DEVICE_USB_SERIAL_JTAG_CONTROLLER)
#define UART_DEVICE "/dev/usbserjtag"
#endif

void on_init(void)
{
    int fd, ret;
    const char device[] = UART_DEVICE;
    const char text[] = "Hello World!\n";
    int text_size = sizeof(text);

    fd = open(device, O_RDWR);
    if (fd < 0) { printf("Open failed, errno=%d.\n", errno); return; }

    ret = write(fd, text, text_size);
    if (ret < 0) { printf("Write failed, errno=%d.\n", errno); close(fd); return; }

    close(fd);
}

int main(int argc, char *argv[]) { on_init(); return 0; }
```

### 2. 文件系统读写（取自 examples/file_system）

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>

#define BUFFER_SIZE 32

void on_init(void)
{
    int fd, ret;
    const char file[] = "/storage/hello.txt";
    const char text[] = "Hello World!";
    int text_size = sizeof(text);
    char buffer[BUFFER_SIZE] = { 0 };

    /* 写：O_WRONLY | O_CREAT | O_TRUNC */
    fd = open(file, O_WRONLY | O_CREAT | O_TRUNC);
    if (fd < 0) { printf("Open write failed, errno=%d.\n", errno); return; }
    write(fd, text, text_size);
    close(fd);

    /* 读：O_RDONLY */
    fd = open(file, O_RDONLY);
    if (fd < 0) { printf("Open read failed, errno=%d.\n", errno); return; }
    ret = read(fd, buffer, BUFFER_SIZE - 1);
    if (ret > 0) printf("Read text is \"%s\".\n", buffer);
    close(fd);

    /* 定位文件末尾获取大小 */
    fd = open(file, O_RDONLY);
    if (fd < 0) { printf("Open seek failed, errno=%d.\n", errno); return; }
    ret = lseek(fd, 0, SEEK_END);
    if (ret >= 0) printf("File size is %d.\n", ret);
    close(fd);
}

int main(int argc, char *argv[]) { on_init(); return 0; }
```

> 注意：原 `examples/file_system/main/file_system_main.c` 在读取/定位分支的条件检查写法存在瑕疵（用 `ret` 判 `open` 返回值）。上述代码已修正为检查 `fd`。功能语义与示例一致。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| UART 写失败 | 设备未在 menuconfig 选择 | 选 `CONFIG_UART_DEVICE_UART0` 或 USB_SERIAL_JTAG |
| 文件写失败 errno | `/storage` 未挂载 | 确认宿主 ESP-WASMachine 已挂载文件系统 |
| `read` 返回 0 | 文件为空或已到末尾 | 写入成功后再读 |
| 用了 `fopen/fread` | 未必链接 stdio 文件层 | ESP-WDF 文件示例用底层 `open/read/write` |

## 参考

- `examples/peripherals/uart/uart_simple/main/uart_simple_main.c`
- `examples/file_system/main/file_system_main.c`
- `resources/api_reference.md` —— 第 3.1 节 设备节点路径
