# 通过 USB CDC-ACM 主机端口烧录（USB Host）

> **适用摘要**: 用 ESP32 主机的 USB OTG（USB Host）经 CDC-ACM 类烧录目标（目标的 USB Serial/JTAG 或 USB OTG 外设）。无需额外 TX/RX/BOOT 线，单根 USB 即可。port 可在断开后经 `esp_loader_init_serial()` 重连。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-serial-flasher/resources/`, source/examples in `repos/esp-serial-flasher/`, and this recipe path `repos/esp-serial-flasher/recipes/usb_cdc_acm.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB 烧录 ESP"
- "CDC-ACM 烧录"
- "USB Host 烧录目标"
- "esp32_usb_cdc_acm_ops"

## 前置条件

| 条件 | 要求 |
|---|---|
| host | 具 USB OTG 的 ESP 芯片（S2/S3/P4）；ESP-IDF v5.5+ |
| port 编译 | `CONFIG_SERIAL_FLASHER_PORT_USB_CDC_ACM=y`（依赖 `SOC_USB_OTG_SUPPORTED`） |
| 依赖组件 | `espressif/usb_host_cdc_acm` ^2（idf_component.yml 自动按 target 拉取） |
| 目标 | ESP32-S3 / C3 / H2 / C6 / C5 / P4 / C61（具 USB Serial/JTAG 或 OTG） |
| 参考示例 | `examples/esp32_usb_cdc_acm_example/` |

## 分步说明

### 1. 安装 USB Host 与 CDC-ACM 驱动（仅一次）

```c
#include "usb/usb_host.h"
#include "usb/cdc_acm_host.h"

const usb_host_config_t host_config = {
    .skip_phy_setup = false,
    .intr_flags     = ESP_INTR_FLAG_LEVEL1,
};
ESP_ERROR_CHECK(usb_host_install(&host_config));

// USB Host 事件处理任务（必须）
xTaskCreate(usb_lib_task, "usb_lib", 4096, NULL, 20, NULL);

ESP_ERROR_CHECK(cdc_acm_host_install(NULL));
```

### 2. 构造 CDC-ACM port（自动检测 VID/PID）

```c
#include "esp32_usb_cdc_acm_port.h"

esp32_usb_cdc_acm_port_t port = {
    .port.ops                     = &esp32_usb_cdc_acm_ops,
    .device_vid                   = USB_VID_PID_AUTO_DETECT,  // 0 = 自动
    .device_pid                   = USB_VID_PID_AUTO_DETECT,
    .connection_timeout_ms        = 1000,
    .out_buffer_size              = 4096,                      // 须 > 最大 USB 包
    .device_disconnected_callback = device_disconnected_callback,
};
```

支持的目标 VID/PID 定义在 `esp32_usb_cdc_acm_port.h`：`ESPRESSIF_VID`(0x303a)、`ESP_SERIAL_JTAG_PID`(0x1001)，以及 CP210x / CH340/CH341 系列。

### 3. init_serial + 连接（速率参数传 0）

```c
esp_loader_t loader;
while (true) {
    if (esp_loader_init_serial(&loader, &port.port) != ESP_LOADER_SUCCESS) {
        continue;   // 设备未就绪，重试
    }
    // USB CDC 忽略 line coding，速率参数传 0
    if (connect_to_target(&loader, 0) == ESP_LOADER_SUCCESS) {
        break;
    }
}
```

### 4. 烧录 + 等待断开（支持热插拔重连）

```c
target_chip_t chip = esp_loader_get_target(&loader);
flash_binary(&loader, bootloader_bin, bootloader_bin_size, get_bootloader_address(chip));
flash_binary(&loader, partition_table_bin, partition_table_bin_size, 0x8000);
flash_binary(&loader, app_bin, app_bin_size, 0x10000);

// 断开回调触发后，回到循环顶部 esp_loader_init_serial 重开设备
xSemaphoreTake(device_disconnected_sem, portMAX_DELAY);
```

> port 可在设备断开后再次调用 `esp_loader_init_serial()` 复用（重新打开设备）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| init 失败 / 一直 retry | 未先 `usb_host_install` + `cdc_acm_host_install` | 按 Step 1 先装驱动 + 事件任务 |
| 改速率报错 | USB CDC 不支持改速率 | 速率参数始终传 0 |
| 目标需手动进下载模式 | 目标固件占用 USB OTG | 手动置 boot 模式再连 |
| 供电不足 | USB 口供电不足以驱动目标 | 目标独立供电 |
| 断开后无法重连 | 未重新 `init_serial` | port 可复用，断开后再次 `esp_loader_init_serial` |

## 参考

- `examples/esp32_usb_cdc_acm_example/main/main.c` — USB Host + CDC-ACM 烧录完整示例（含断开重连）
- `port/esp32_usb_cdc_acm_port.h` — `esp32_usb_cdc_acm_port_t`、`esp32_usb_cdc_acm_ops`、VID/PID 定义
- `idf_component.yml` — `usb_host_cdc_acm` ^2 依赖规则
- `docs/hardware-connections.md` — USB CDC-ACM 接线与供电注意
