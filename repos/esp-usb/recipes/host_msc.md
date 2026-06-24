# USB 主机：MSC 主机驱动（U 盘读写）

> **适用摘要**: 用 MSC 主机驱动读写 USB U 盘/移动存储，支持扇区读写、VFS 注册和设备信息查询。

## 触发意图

- "USB 读 U 盘"
- "USB flash drive host"
- "MSC 主机"
- "读写 USB 存储扇区"
- "msc_host"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/usb_host_msc^1.2.0"`（会带 `usb`） |
| ESP-IDF | `>= 5.5.3` |
| 参考头文件 | `host/class/msc/usb_host_msc/include/usb/msc_host.h`、`msc_host_vfs.h` |

## 分步说明

### 1. 安装 MSC 主机驱动

驱动可自建后台任务处理事件，或由应用轮询 `msc_host_handle_events`。

```c
#include "usb/msc_host.h"

static void msc_event_cb(const msc_host_event_t *event, void *arg) {
    switch (event->event) {
    case MSC_DEVICE_CONNECTED:
        printf("MSC device connected, addr=%u\n", event->device.address);
        // 记录地址，供后续 msc_host_install_device 使用
        break;
    case MSC_DEVICE_DISCONNECTED:
        printf("MSC device disconnected\n");
        break;
    default:
        break;
    }
}

const msc_host_driver_config_t drv_cfg = {
    .create_backround_task = true,   // true: 驱动自建任务；false: 应用轮询
    .task_priority = 10,
    .stack_size    = 4096,
    .core_id       = tskNO_AFFINITY,
    .callback      = msc_event_cb,
    .callback_arg  = NULL,
};
ESP_ERROR_CHECK(msc_host_install(&drv_cfg));
```

> 若 `create_backround_task = false`，应用必须周期性调用 `msc_host_handle_events(timeout)`。

### 2. 安装设备实例

设备连入事件给的是 USB 地址，需 `msc_host_install_device` 拿到设备句柄。

```c
msc_host_device_handle_t msc_dev;
ESP_ERROR_CHECK(msc_host_install_device(device_address, &msc_dev));
```

### 3. 查询设备信息

```c
msc_host_device_info_t info;
ESP_ERROR_CHECK(msc_host_get_device_info(msc_dev, &info));
printf("sectors=%u, sector_size=%u, VID=%04x PID=%04x\n",
       (unsigned)info.sector_count, (unsigned)info.sector_size,
       info.idVendor, info.idProduct);
// 调试：msc_host_print_descriptors(msc_dev);
```

### 4. 扇区读写

```c
const size_t sector_size = info.sector_size;   // 通常 512
uint8_t *buf = heap_caps_malloc(sector_size, MALLOC_CAP_DMA);
assert(buf);

// 读 0 号扇区
ESP_ERROR_CHECK(msc_host_read_sector(msc_dev, 0, buf, sector_size));
ESP_LOG_BUFFER_HEX("msc", buf, 32, ESP_LOG_INFO);

// 写 0 号扇区
memset(buf, 0x55, sector_size);
ESP_ERROR_CHECK(msc_host_write_sector(msc_dev, 0, buf, sector_size));

free(buf);
```

### 5. 通过 VFS 访问（文件系统）

`msc_host_vfs.h` 把 U 盘注册到 VFS，可用标准文件 API。

```c
#include "usb/msc_host_vfs.h"

msc_host_vfs_handle_t vfs_hdl;
// 注册到默认路径（base_path）并挂载
ESP_ERROR_CHECK(msc_host_vfs_register(msc_dev, "/usb", NULL, &vfs_hdl));

FILE *f = fopen("/usb/test.txt", "r");
if (f) {
    char line[128];
    while (fgets(line, sizeof(line), f)) {
        printf("%s", line);
    }
    fclose(f);
}

// 需要格式化时
// msc_host_vfs_format(msc_dev, "/usb", NULL, &vfs_hdl);

// 注销
msc_host_vfs_unregister(vfs_hdl);
```

### 6. 卸载设备与驱动

```c
ESP_ERROR_CHECK(msc_host_uninstall_device(msc_dev));   // 先卸设备
ESP_ERROR_CHECK(msc_host_uninstall());                  // 再卸驱动
```

设备出错时可调 `msc_host_reset_recovery(msc_dev)` 尝试恢复。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 一直没收到 `MSC_DEVICE_CONNECTED` | 驱动未处理事件（后台任务没建/没轮询） | `create_backround_task=true` 或周期调 `msc_host_handle_events` |
| `msc_host_install_device` 失败 | 地址来自过期事件 | 用最新 CONNECTED 事件里的地址 |
| 扇区读写失败 | 缓冲非 DMA 可用内存 / 大小不是扇区整数倍 | 用 `MALLOC_CAP_DMA`，长度 = N × sector_size |
| VFS 注册失败 | 未先 `msc_host_install_device` | 先拿设备句柄再注册 VFS |
| 卸载驱动失败 | 仍有设备句柄未卸 | 先 `msc_host_uninstall_device` 再 `msc_host_uninstall` |
| 大容量 U 盘部分扇区读不出 | 控制器通道数/缓冲受限 | 单次读写量适配，必要时 `msc_host_reset_recovery` |

## 参考

- `host/class/msc/usb_host_msc/include/usb/msc_host.h` — MSC 主机 API
- `host/class/msc/usb_host_msc/include/usb/msc_host_vfs.h` — VFS 注册/格式化
- `host/class/msc/usb_host_msc/include/esp_private/msc_scsi_bot.h` — SCSI/BOT 内部
- ESP-IDF 示例（仓库文档引用）：`peripherals/usb/host/msc`
