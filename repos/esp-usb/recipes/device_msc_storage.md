# USB 设备：MSC 大容量存储

> **适用摘要**: 把 ESP 芯片做成 USB 大容量存储（U 盘）设备，存储介质为 SPI-Flash 或 SD 卡，处理挂载/卸载事件。

## 触发意图

- "USB U 盘"
- "USB MSC"
- "大容量存储设备"
- "USB 暴露 SPI flash"
- "USB SD 卡读卡器"
- "tinyusb msc storage"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"` |
| menuconfig | `CONFIG_TINYUSB_MSC_ENABLED=y`；`CONFIG_TINYUSB_MSC_BUFSIZE`（FS 默认 512，HS 默认 8192） |
| 分区表 | SPI-Flash 介质需要一张含 `storage`（DATA/FAT）分区的 `partitions.csv` |
| 参考示例 | `device/esp_tinyusb/test_apps/msc_storage/main/`（storage_common.c、test_msc_storage.c） |

## 分步说明

### 1. 准备存储介质

**SPI-Flash（所有目标可用）**：通过 wear levelling 挂载到 `storage` 分区。

```c
#include "wear_levelling.h"
#include "esp_partition.h"

static void storage_init_spiflash(wl_handle_t *wl_handle) {
    const esp_partition_t *data_partition = esp_partition_find_first(
        ESP_PARTITION_TYPE_DATA, ESP_PARTITION_SUBTYPE_DATA_FAT, "storage");
    assert(data_partition != NULL);
    ESP_ERROR_CHECK(wl_mount(data_partition, wl_handle));
    printf("SPIFLASH WL: sectors=%u, sector_size=%u\n",
           (unsigned)(wl_size(*wl_handle) / wl_sector_size(*wl_handle)),
           (unsigned)wl_sector_size(*wl_handle));
}
```

**SD 卡（仅 S3/P4/S31，`SOC_SDMMC_HOST_SUPPORTED`）**：

```c
#include "driver/sdmmc_host.h"
#include "sdmmc_cmd.h"

static void storage_init_sdmmc(sdmmc_card_t **card) {
    sdmmc_host_t host = SDMMC_HOST_DEFAULT();
    sdmmc_slot_config_t slot_config = SDMMC_SLOT_CONFIG_DEFAULT();
    slot_config.width = 4;
    slot_config.flags |= SDMMC_SLOT_FLAG_INTERNAL_PULLUP;
    // 自定义引脚（按板子）
    // slot_config.clk/cmd/d0..d3 = ...

    sdmmc_card_t *c = calloc(1, sizeof(sdmmc_card_t));
    ESP_ERROR_CHECK(sdmmc_host_init());
    ESP_ERROR_CHECK(sdmmc_host_init_slot(host.slot, &slot_config));
    ESP_ERROR_CHECK(sdmmc_card_init(&host, c));
    *card = c;
}
```

> S2/H4 不支持 SDMMC 主机，只能用 SPI-Flash 介质。

### 2. 创建 MSC 存储实例

```c
#include "tinyusb.h"
#include "tinyusb_msc.h"

void app_main(void) {
    // 1) 安装设备驱动（MSC 的默认配置描述符由 esp_tinyusb 内置，full_speed_config 可填 NULL）
    tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
    tusb_cfg.descriptor.full_speed_config = NULL;   // 用 MSC 内置默认
    ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));

    // 2) 准备介质
    wl_handle_t wl_handle;
    storage_init_spiflash(&wl_handle);

    // 3) 创建 SPI-Flash 介质实例（创建时会自动安装 MSC 驱动）
    tinyusb_msc_storage_handle_t storage_hdl;
    const tinyusb_msc_storage_config_t config_spi = {
        .medium.wl_handle = wl_handle,
        .mount_point = TINYUSB_MSC_STORAGE_MOUNT_USB,   // 先交给 USB 主机
    };
    ESP_ERROR_CHECK(tinyusb_msc_new_storage_spiflash(&config_spi, &storage_hdl));
}
```

SD 卡实例：

```c
sdmmc_card_t *card = NULL;
storage_init_sdmmc(&card);
tinyusb_msc_storage_handle_t storage_hdl;
const tinyusb_msc_storage_config_t config_sd = {
    .medium.card = card,
    .mount_point = TINYUSB_MSC_STORAGE_MOUNT_USB,
};
ESP_ERROR_CHECK(tinyusb_msc_new_storage_sdmmc(&config_sd, &storage_hdl));
```

### 3. 挂载点切换（USB ↔ APP）

存储可在 USB 主机和本地应用之间切换归属：

```c
// 交给本地应用读写文件
tinyusb_msc_set_storage_mount_point(storage_hdl, TINYUSB_MSC_STORAGE_MOUNT_APP);
// ... 用 fopen/fread/fwrite 访问 FAT ...
// 再交回 USB 主机
tinyusb_msc_set_storage_mount_point(storage_hdl, TINYUSB_MSC_STORAGE_MOUNT_USB);
```

### 4. MSC 事件回调

```c
static void msc_event_cb(tinyusb_msc_storage_handle_t h,
                         tinyusb_msc_event_t *event, void *arg) {
    switch (event->id) {
    case TINYUSB_MSC_EVENT_MOUNT_COMPLETE:
        printf("MSC mount complete (point=%d)\n", event->mount_point);
        break;
    case TINYUSB_MSC_EVENT_MOUNT_FAILED:
        printf("MSC mount failed\n");
        break;
    case TINYUSB_MSC_EVENT_FORMAT_REQUIRED:
        printf("MSC needs format\n");
        break;
    default:
        break;
    }
}

// 安装驱动后注册
ESP_ERROR_CHECK(tinyusb_msc_install_driver(&(const tinyusb_msc_driver_config_t){
    .user_flags.val = 0,
    .callback = msc_event_cb,
    .callback_arg = NULL,
}));
```

> 也可在创建实例后用 `tinyusb_msc_set_storage_callback(cb, arg)` 注册。需要格式化时调用 `tinyusb_msc_format_storage(storage_hdl)`（需先把介质挂到 `TINYUSB_MSC_STORAGE_MOUNT_APP`）。

### 5. 性能调优

| FIFO (`CONFIG_TINYUSB_MSC_BUFSIZE`) | ESP32-S3 读/写 | ESP32-P4 读/写 |
|---|---|---|
| 512 B | 0.566 / 0.236 MB/s | 1.174 / 0.238 MB/s |
| 8192 B | 0.925 / 0.928 MB/s | 4.744 / 2.157 MB/s |
| 32768 B | — | 5.998 / 4.485 MB/s |

内部 SPI-Flash 在写时会暂停程序执行，吞吐受限；持续吞吐请用 SD 卡（S3/P4/S31）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `tinyusb_msc_new_storage_sdmmc` 链接失败 | 目标不支持 SDMMC（S2/H4） | 改用 `tinyusb_msc_new_storage_spiflash` |
| 主机无法识别 U 盘 / 提示未格式化 | 介质无文件系统 | 监听 `TINYUSB_MSC_EVENT_FORMAT_REQUIRED`，挂到 APP 后调 `tinyusb_msc_format_storage` |
| `ESP_ERR_NOT_SUPPORTED` MSC buffer smaller than WL sector | `CONFIG_TINYUSB_MSC_BUFSIZE` 太小 | 提高到 ≥ WL 扇区大小（通常 4096） |
| 写入很慢 | 内部 SPI-Flash + 小 FIFO | 用 SD 卡；把 `CONFIG_TINYUSB_MSC_BUFSIZE` 调到 8192/32768 |
| 应用与 USB 主机同时写 | 同一时刻介质只能归一方 | 用 `mount_point` 切换归属，不要并发访问 |

## 参考

- `device/esp_tinyusb/include/tinyusb_msc.h` — MSC 设备 API 与事件
- `device/esp_tinyusb/include_private/storage_spiflash.h` / `storage_sdmmc.h`
- `device/esp_tinyusb/test_apps/msc_storage/main/storage_common.c` — SPI-Flash/SD 卡初始化
- `device/esp_tinyusb/test_apps/msc_storage/main/test_msc_storage.c`
- ESP-IDF 示例（仓库文档引用）：`peripherals/usb/device/tusb_msc`
