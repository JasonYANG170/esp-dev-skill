# USB 设备：控制台重定向与 VFS

> **适用摘要**: 把标准输入/输出（`printf`/stdin）重定向到 USB CDC，或将 CDC 接口注册进 VFS 以便用 `fopen/fread/fwrite` 读写。

## 触发意图

- "USB 打印日志"
- "printf 重定向到 USB"
- "USB 控制台"
- "USB CDC 当文件读写"
- "tinyusb console"
- "vfs_tinyusb"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"` |
| menuconfig | `CONFIG_TINYUSB_CDC_ENABLED=y` |
| 参考示例 | `device/esp_tinyusb/test_apps/cdc/main/test_cdc.c`（CDC1 注册到 VFS） |

## 分步说明

### 1. 安装设备驱动 + CDC 接口

```c
#include "tinyusb_default_config.h"
#include "tinyusb.h"
#include "tinyusb_cdc_acm.h"

app_main(void) {
    tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));

    const tinyusb_config_cdcacm_t acm_cfg = {
        .cdc_port = TINYUSB_CDC_ACM_0,
        .callback_rx = NULL,
        .callback_rx_wanted_char = NULL,
        .callback_line_state_changed = NULL,
        .callback_line_coding_changed = NULL,
    };
    ESP_ERROR_CHECK(tinyusb_cdcacm_init(&acm_cfg));
}
```

### 2. 控制台重定向（stdin/stdout/stderr → USB）

`tinyusb_console_init(cdc_intf)` 把标准流重定向到指定 CDC 接口；`tinyusb_console_deinit` 恢复到 UART。

```c
#include "tinyusb_console.h"

// 把 CDC0 当作系统控制台
ESP_ERROR_CHECK(tinyusb_console_init(0));

printf("This text goes out over USB CDC0\n");   // 直接走 USB
char line[64];
if (fgets(line, sizeof(line), stdin)) {          // 从 USB 读一行
    printf("echo: %s", line);
}
```

> 注：控制台需要主机一侧已打开该串口（DTR 置位）才会有数据流动。

### 3. 把 CDC 注册进 VFS（文件方式访问）

`esp_vfs_tusb_cdc_register` 把 CDC 接口挂到一个 VFS 路径，之后可用标准 `fopen/fread/fwrite/fclose` 读写。同一时刻只能注册一个 CDC 接口。

```c
#include "vfs_tinyusb.h"

#define VFS_PATH "/dev/usb-cdc1"

ESP_ERROR_CHECK(esp_vfs_tusb_cdc_register(TINYUSB_CDC_ACM_1, VFS_PATH));
esp_vfs_tusb_cdc_set_rx_line_endings(ESP_LINE_ENDINGS_CRLF);  // 收到的 CRLF 转 LF
esp_vfs_tusb_cdc_set_tx_line_endings(ESP_LINE_ENDINGS_LF);    // 发送不转换

FILE *cdc = fopen(VFS_PATH, "r+");
assert(cdc != NULL);

uint8_t buf[256];
size_t n = fread(buf, 1, sizeof(buf), cdc);
if (n > 0) {
    fwrite(buf, 1, n, cdc);   // 回显
}
fclose(cdc);

// 卸载
esp_vfs_tusb_cdc_unregister(VFS_PATH);
```

> 默认路径宏 `VFS_TUSB_PATH_DEFAULT = "/dev/tusb_cdc"`，最大路径长度 `VFS_TUSB_MAX_PATH = 16`（含 `\0`）。

### 4. 行尾转换模式

| 模式 | 发送（TX）行为 | 接收（RX）行为 |
|---|---|---|
| `ESP_LINE_ENDINGS_CRLF` | LF → CRLF | CRLF → LF |
| `ESP_LINE_ENDINGS_CR` | LF → CR | CR → LF |
| `ESP_LINE_ENDINGS_LF` | 不转换 | 不转换 |

- TX 由 `esp_vfs_tusb_cdc_set_tx_line_endings` 控制
- RX 由 `esp_vfs_tusb_cdc_set_rx_line_endings` 控制

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `printf` 没有输出 | 未调 `tinyusb_console_init`，或主机未打开串口 | 先 `tinyusb_console_init(0)`；主机端打开对应 COM 口 |
| `esp_vfs_tusb_cdc_register` 返回 `ESP_ERR_INVALID_STATE` | 该 CDC 接口未初始化 | 先 `tinyusb_cdcacm_init` 再注册 VFS |
| `ESP_ERR_INVALID_ARG` 路径 | 路径长度超过 `VFS_TUSB_MAX_PATH`(16) | 缩短路径或用 `NULL` 取默认 |
| 同时注册两个 CDC 到 VFS 失败 | 同一时刻只允许一个 TinyUSB CDC 注册到 VFS | 只注册一个；另一个用 `tinyusb_cdcacm_read` 直接读 |
| 收发数据带 `\r` | 行尾转换模式不匹配 | 按上表设置 RX/TX 行尾模式 |

## 参考

- `device/esp_tinyusb/include/tinyusb_console.h` — `tinyusb_console_init/deinit`
- `device/esp_tinyusb/include/vfs_tinyusb.h` — `esp_vfs_tusb_cdc_register` 等
- `device/esp_tinyusb/test_apps/cdc/main/test_cdc.c` — VFS 注册 + 回显示例
- ESP-IDF 示例（仓库文档引用）：`peripherals/usb/device/tusb_console`
