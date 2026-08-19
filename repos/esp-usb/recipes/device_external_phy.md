# USB 设备：外部 PHY（ESP32-S3）

> **适用摘要**: 在 ESP32-S3 上使用外部 USB PHY（SP5301/TUSB1106/STUSB03E），让 USB-OTG 与 USB-Serial-JTAG 同时工作。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-usb/resources/`, source/examples in `repos/esp-usb/`, and this recipe path `repos/esp-usb/recipes/device_external_phy.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "外部 PHY"
- "external PHY"
- "SP5301"
- "TUSB1106"
- "STUSB03E"
- "USB-OTG 和 USB-Serial-JTAG 同时用"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标 | ESP32-S3（含 USB-OTG 与 USB-Serial-JTAG，二者共享一个 PHY） |
| 组件依赖 | `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"`（device）；host 用 `espressif/usb` |
| menuconfig | 按需启用对应类（`CONFIG_TINYUSB_CDC_ENABLED` 等） |
| 文档来源 | `docs/en/usb_device.rst`（External PHY Configuration 段） |

## 分步说明

ESP32-S3 的 USB-OTG 与 USB-Serial-JTAG **共用同一个 PHY**，只能二选一。若要在 USB-Serial-JTAG 仍工作（用于调试/烧录）时同时使用 USB 设备/主机功能，必须接外部 PHY。

### 1. 配置外部 PHY 的 GPIO 映射

未使用的引脚必须设为 `-1`（如 `suspend_n_io_num` 当前不支持）。`fs_edge_sel_io_num` 仅在需要切换低速/全速时连接。

```c
#include "usb/usb_phy.h"

// GPIO 映射（按你的板子原理图调整）
const usb_phy_ext_io_conf_t ext_io_conf = {
    .vp_io_num        = 8,
    .vm_io_num        = 5,
    .rcv_io_num       = 11,
    .oen_io_num       = 17,
    .vpo_io_num       = 4,
    .vmo_io_num       = 46,
    .suspend_n_io_num = -1,    // 不支持，必须 -1
    .fs_edge_sel_io_num = -1,  // 可选
};
```

### 2. 创建并安装外部 PHY（设备模式）

```c
#include "usb/usb_phy.h"
#include "tinyusb_default_config.h"
#include "tinyusb.h"

const usb_phy_config_t phy_config = {
    .controller = USB_PHY_CTRL_OTG,
    .target     = USB_PHY_TARGET_EXT,
    .otg_mode   = USB_OTG_MODE_DEVICE,
    .otg_speed  = USB_PHY_SPEED_FULL,
    .ext_io_conf = &ext_io_conf,
};

usb_phy_handle_t phy_hdl;
ESP_ERROR_CHECK(usb_new_phy(&phy_config, &phy_hdl));

// 关键：跳过 esp_tinyusb 内部的 PHY 初始化，复用上面创建的外部 PHY
tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
tusb_cfg.phy.skip_setup = true;
ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
```

### 3. 主机模式使用外部 PHY

```c
#include "usb/usb_phy.h"
#include "usb/usb_host.h"

const usb_phy_config_t phy_config = {
    .controller = USB_PHY_CTRL_OTG,
    .target     = USB_PHY_TARGET_EXT,
    .otg_mode   = USB_OTG_MODE_HOST,
    .otg_speed  = USB_PHY_SPEED_FULL,
    .ext_io_conf = &ext_io_conf,
};

usb_phy_handle_t phy_hdl;
ESP_ERROR_CHECK(usb_new_phy(&phy_config, &phy_hdl));

// 关键：跳过 Host Library 的 PHY 初始化
usb_host_config_t host_config = {
    .skip_phy_setup = true,
};
ESP_ERROR_CHECK(usb_host_install(&host_config));
```

### 4. 已测试的外部 PHY IC

| IC | 说明 |
|---|---|
| **SP5301** | 直接支持，按官方接线即可 |
| **TUSB1106** | 直接支持，按其数据手册接线（D+/D- 串阻、供电） |
| **STUSB03E** | 需要模拟开关做信号路由（见 `docs/_static/usb_device/ext_phy_schematic_stusb03e.png`） |

> 替代方案：也可不用外部 PHY，而是烧 eFuse 把调试接口从 USB-Serial-JTAG 切到普通 JTAG，从而释放 PHY 给 USB-OTG。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 外部 PHY 配好后仍用内部 PHY | 未设 `skip_setup=true`（device）/`skip_phy_setup=true`（host） | 在 `tinyusb_driver_install`/`usb_host_install` 前置位 |
| 枚举失败 | GPIO 映射与原理图不符 | 核对每个 `*_io_num`；未用引脚设 `-1` |
| `suspend_n_io_num` 连了引脚但无效 | 该引脚当前不支持 | 必须设为 `-1` |
| 设备/主机与 USB-Serial-JTAG 抢 PHY | 仍只用内部 PHY | 接外部 PHY 并 `skip_setup`，或烧 eFuse 切 JTAG |
| 自供电设备未接 VBUSDET | 外部 PHY 模式下也要 VBUS 监测 | 自供电时设 `self_powered=true` + `vbus_monitor_io`，并把 PHY 的 VBUSDET 接到 ESP32-S3 |

## 参考

- `docs/en/usb_device.rst` — External PHY Configuration（device）
- `docs/en/usb_host.rst` — External PHY Configuration（host）
- `docs/_static/usb_device/ext_phy_schematic_stusb03e.png` — STUSB03E 接线示例
- ESP-IDF `usb/usb_phy.h` — `usb_phy_config_t`、`usb_new_phy`
