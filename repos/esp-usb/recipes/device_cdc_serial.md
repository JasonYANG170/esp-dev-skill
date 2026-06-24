# USB 设备：CDC-ACM 串口

> **适用摘要**: 把 ESP 芯片做成一个 USB CDC-ACM 串口设备（虚拟串口），支持发送、接收、回调与双串口。

## 触发意图

- "USB 串口"
- "USB CDC"
- "虚拟串口设备"
- "USB 转 serial"
- "tinyusb cdcacm"
- "双串口 / double CDC"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"` |
| menuconfig | `CONFIG_TINYUSB_CDC_ENABLED=y`；CDC 数量 `CONFIG_TINYUSB_CDC_COUNT`（1 或 2） |
| 参考示例 | `device/esp_tinyusb/test_apps/cdc/main/test_cdc.c`（双 CDC + VFS 回显） |

## 分步说明

### 1. 安装 TinyUSB 设备驱动

使用 `TINYUSB_DEFAULT_CONFIG()` 取默认值，再按需覆盖。宏在 `tinyusb_default_config.h` 中定义，必须 include 该头才能使用宏。

```c
#include "tinyusb_default_config.h"
#include "tinyusb.h"
#include "tinyusb_cdc_acm.h"

void app_main(void) {
    tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
    ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
    // ... 接下来初始化 CDC 接口
}
```

> 说明：`TINYUSB_DEFAULT_CONFIG()` 在 ESP32-P4/ESP32-S31 上默认选 `TINYUSB_PORT_HIGH_SPEED_0`，其他目标选 `TINYUSB_PORT_FULL_SPEED_0`。默认任务栈 `TINYUSB_DEFAULT_TASK_SIZE=4096`、优先级 `TINYUSB_DEFAULT_TASK_PRIO=5`。

### 2. 自定义设备/配置描述符（可选）

若要自定义 VID/PID 或接口布局，提供 `tusb_desc_device_t` 与配置描述符；不提供（设为 `NULL`）则使用 menuconfig 默认值。CDC-ACM 默认就有内置配置描述符。

```c
// 双 CDC 设备描述符（composite，必须用 IAD）
static const tusb_desc_device_t cdc_device_descriptor = {
    .bLength            = sizeof(cdc_device_descriptor),
    .bDescriptorType    = TUSB_DESC_DEVICE,
    .bcdUSB             = 0x0200,
    .bDeviceClass       = TUSB_CLASS_MISC,
    .bDeviceSubClass    = MISC_SUBCLASS_COMMON,
    .bDeviceProtocol    = MISC_PROTOCOL_IAD,
    .bMaxPacketSize0    = CFG_TUD_ENDPOINT0_SIZE,
    .idVendor           = TINYUSB_ESPRESSIF_VID,
    .idProduct          = 0x4002,
    .bcdDevice          = 0x0100,
    .iManufacturer      = 0x01,
    .iProduct           = 0x02,
    .iSerialNumber      = 0x03,
    .bNumConfigurations = 0x01,
};

// 双 CDC 配置描述符（用 TinyUSB 提供的 TUD_CDC_DESCRIPTOR 宏）
static const uint8_t cdc_desc_configuration[] = {
    TUD_CONFIG_DESCRIPTOR(1, 4, 0,
        TUD_CONFIG_DESC_LEN + CFG_TUD_CDC * TUD_CDC_DESC_LEN,
        TUSB_DESC_CONFIG_ATT_REMOTE_WAKEUP, 100),
    TUD_CDC_DESCRIPTOR(0, 4, 0x81, 8, 0x02, 0x82, (TUD_OPT_HIGH_SPEED ? 512 : 64)),
    TUD_CDC_DESCRIPTOR(2, 4, 0x83, 8, 0x04, 0x84, (TUD_OPT_HIGH_SPEED ? 512 : 64)),
};

// 安装时注入
tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.descriptor.device = &cdc_device_descriptor;
tusb_cfg.descriptor.full_speed_config = cdc_desc_configuration;
#if (TUD_OPT_HIGH_SPEED)   // HS 目标还要补 qualifier 和 high_speed_config
tusb_cfg.descriptor.qualifier = &device_qualifier;
tusb_cfg.descriptor.high_speed_config = cdc_desc_configuration;
#endif
ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
```

### 3. 初始化 CDC-ACM 接口（带回调）

```c
static void cdc_rx_callback(int itf, cdcacm_event_t *event) {
    uint8_t buf[CONFIG_TINYUSB_CDC_RX_BUFSIZE];
    size_t rx_size = 0;
    tinyusb_cdcacm_read(itf, buf, sizeof(buf), &rx_size);
    if (rx_size > 0) {
        // 回显
        tinyusb_cdcacm_write_queue(itf, buf, rx_size);
        tinyusb_cdcacm_write_flush(itf, 0);   // 回调内不要用带超时的 flush
    }
}

const tinyusb_config_cdcacm_t acm_cfg = {
    .cdc_port = TINYUSB_CDC_ACM_0,
    .callback_rx              = cdc_rx_callback,
    .callback_rx_wanted_char  = NULL,
    .callback_line_state_changed = NULL,
    .callback_line_coding_changed = NULL,
};
ESP_ERROR_CHECK(tinyusb_cdcacm_init(&acm_cfg));
```

第二个 CDC 接口：

```c
const tinyusb_config_cdcacm_t acm_cfg1 = {
    .cdc_port = TINYUSB_CDC_ACM_1,
    .callback_rx = NULL,   // 通过 VFS 读，见下文
    .callback_rx_wanted_char = NULL,
    .callback_line_state_changed = NULL,
    .callback_line_coding_changed = NULL,
};
ESP_ERROR_CHECK(tinyusb_cdcacm_init(&acm_cfg1));
```

### 4. 轮询读取（任务循环）

不用回调时，可在任务中轮询：

```c
uint8_t buf[CONFIG_TINYUSB_CDC_RX_BUFSIZE + 1];
while (true) {
    size_t rx_size = 0;
    int itf = 0;   // TINYUSB_CDC_ACM_0
    if (tinyusb_cdcacm_read(itf, buf, CONFIG_TINYUSB_CDC_RX_BUFSIZE, &rx_size) == ESP_OK
        && rx_size > 0) {
        tinyusb_cdcacm_write_queue(itf, buf, rx_size);
        tinyusb_cdcacm_write_flush(itf, 0);
    }
    vTaskDelay(pdMS_TO_TICKS(10));
}
```

### 5. CDC 事件类型

回调里用 `event->type` 区分：

| 事件 | 触发时机 | 关联数据 |
|---|---|---|
| `CDC_EVENT_RX` | 收到数据 | （由 `tinyusb_cdcacm_read` 取） |
| `CDC_EVENT_RX_WANTED_CHAR` | 收到指定字符 | `rx_wanted_char_data.wanted_char` |
| `CDC_EVENT_LINE_STATE_CHANGED` | DTR/RTS 变化 | `line_state_changed_data.dtr` / `.rts` |
| `CDC_EVENT_LINE_CODING_CHANGED` | 波特率等变化 | `line_coding_changed_data.p_line_coding` |

可用 `tinyusb_cdcacm_register_callback` 在初始化后再注册/替换回调。

### 6. 关键 Kconfig（menuconfig）

- `CONFIG_TINYUSB_CDC_ENABLED`：必须打开，否则 `#include "tinyusb_cdc_acm.h"` 直接 `#error`。
- `CONFIG_TINYUSB_CDC_COUNT`：1 或 2。
- `CONFIG_TINYUSB_CDC_RX_BUFSIZE` / `CONFIG_TINYUSB_CDC_TX_BUFSIZE`：FIFO，默认 512。
- `CONFIG_TINYUSB_CDC_EP_BUFSIZE`：底层端点缓冲，性能影响最大（建议 8192）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `#error "TinyUSB CDC driver must be enabled"` | 未在 menuconfig 启用 CDC | 打开 `CONFIG_TINYUSB_CDC_ENABLED` |
| 回调里 `write_flush` 长时间阻塞 | TinyUSB 延迟到回调结束后才刷新端点 | 回调内用 `timeout=0`，或在别的任务 flush |
| 主机识别不出双串口 | 描述符未用 IAD 复合设备类 | `bDeviceClass=TUSB_CLASS_MISC`、`MISC_SUBCLASS_COMMON`、`MISC_PROTOCOL_IAD` |
| HS 目标主机枚举异常 | 只给了 `full_speed_config`，缺 qualifier/HS 描述符 | 同时给 `high_speed_config` 和 `qualifier` |
| 收不到数据 | 未注册 `callback_rx` 也未轮询 `read` | 二选一；轮询要在循环里调 `tinyusb_cdcacm_read` |

## 参考

- `device/esp_tinyusb/include/tinyusb_cdc_acm.h` — CDC-ACM 设备 API
- `device/esp_tinyusb/include/tinyusb.h` — `tinyusb_config_t` / `tinyusb_driver_install`
- `device/esp_tinyusb/test_apps/cdc/main/test_cdc.c` — 双 CDC 回显示例
- ESP-IDF 示例（仓库文档引用）：`peripherals/usb/device/tusb_serial_device`
