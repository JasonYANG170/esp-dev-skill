# 协处理器蓝牙控制器初始化（v2.5.2+）

> **适用摘要**: 自 ESP-Hosted-MCU v2.5.2 起，协处理器上的蓝牙控制器默认关闭，以便在使能前设置 BT MAC。本配方演示如何在 host 上经 `esp_hosted_bt_controller_init/enable` 启用协处理器控制器、读写 BT MAC，以及与 NimBLE / BlueDroid host 栈的衔接（Hosted HCI 与标准 HCI 两种路径）。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-hosted-mcu/resources/`, source/examples in `repos/esp-hosted-mcu/`, and this recipe path `repos/esp-hosted-mcu/recipes/host_bt_controller.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ESP-Hosted 蓝牙不工作"
- "esp_hosted_bt_controller_init"
- "设置 BT MAC 地址"
- "Hosted HCI vs 标准 HCI"
- "NimBLE / BlueDroid over ESP-Hosted"

## 前置条件

| 条件 | 要求 |
|---|---|
| 版本 | ESP-Hosted-MCU >= v2.5.2（更早版本控制器默认开启，无需此流程；见 `docs/migration_guide.md`） |
| 头文件 | `#include "esp_hosted.h"`（含 `esp_hosted_misc.h`） |
| 参考文档 | `docs/bluetooth_design.md` |

## 分步说明

### 1. 两条 HCI 路径（来自 `docs/bluetooth_design.md`）

- **Hosted HCI**：标准 HCI 外加 ESP-Hosted 头，复用 SPI/SDIO 传输，无需额外 GPIO。调试友好。
- **标准 HCI（专用）**：透明 HCI，需独立的 UART（额外 2 或 4 GPIO 含流控）。可移植到任意带 BT 控制器的协处理器。

> 两者互斥：启用 Hosted HCI 时必须禁用标准 HCI over UART，反之亦然。

### 2. 在 host 上启用协处理器控制器（必须在 BT host 栈之前）

```c
#include "esp_hosted.h"
#include "esp_mac.h"

void app_main(void)
{
    // ... netif/event loop/esp_hosted_init ...
    esp_hosted_connect_to_slave();

    // 1)（可选）读取 BT MAC 长度与当前值
    size_t mac_len = esp_hosted_iface_mac_addr_len_get(ESP_MAC_BT);  // 一般 6
    uint8_t mac[6] = {0};
    esp_hosted_iface_mac_addr_get(mac, sizeof(mac), ESP_MAC_BT);

    // 2)（可选）设置自定义 BT MAC——必须在 enable 之前
    uint8_t custom_mac[6] = {0x11,0x22,0x33,0x44,0x55,0x66};
    esp_hosted_iface_mac_addr_set(custom_mac, sizeof(custom_mac), ESP_MAC_BT);
    // 注意：该 MAC 设置是临时的，设备重启后恢复

    // 3) 初始化并使能协处理器 BT 控制器
    esp_hosted_bt_controller_init();
    esp_hosted_bt_controller_enable();

    // 4) 在此之后初始化你的 BT host 栈（NimBLE 或 BlueDroid）
}
```

相关 API（`host/esp_hosted_misc.h`）：

```c
esp_err_t esp_hosted_bt_controller_init(void);
esp_err_t esp_hosted_bt_controller_enable(void);
esp_err_t esp_hosted_bt_controller_disable(void);
esp_err_t esp_hosted_bt_controller_deinit(bool mem_release);

esp_err_t esp_hosted_iface_mac_addr_set(uint8_t *mac, size_t mac_len, esp_mac_type_t type);
esp_err_t esp_hosted_iface_mac_addr_get(uint8_t *mac, size_t mac_len, esp_mac_type_t type);
size_t    esp_hosted_iface_mac_addr_len_get(esp_mac_type_t type);   // 0=不支持, 6=MAC-48, 8=EUI-64
```

### 3. BlueDroid + Hosted HCI（节选要点）

`host/esp_hosted_bluedroid.h` 提供 BlueDroid 的 HCI 衔接：

```c
void     hosted_hci_bluedroid_open(void);
void     hosted_hci_bluedroid_close(void);
void     hosted_hci_bluedroid_send(uint8_t *data, uint16_t len);
bool     hosted_hci_bluedroid_check_send_available(void);
esp_err_t hosted_hci_bluedroid_register_host_callback(const esp_bluedroid_hci_driver_callbacks_t *callback);
```

启用 Hosted HCI 时，在 host menuconfig 选择蓝牙走 Hosted HCI；在协处理器侧确保 HCI Over SPI/SDIO（即 Hosted HCI）已开启。具体衔接步骤见 `docs/bluetooth_design.md` 第 5 节。

### 4. 选哪个 host 栈

| 场景 | 推荐 | 说明 |
|---|---|---|
| 仅 BLE | NimBLE | 代码与运行内存占用更小 |
| Classic BT + BLE | BlueDroid | 唯一支持 Classic BT 的栈 |

> 经典 ESP32 协处理器只支持 BT v4.2；若用 ESP32 作协处理器，host 的 BT 栈也必须是 v4.2。其他 ESP 协处理器仅支持 BLE。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| BT host 栈初始化失败 | 控制器未先 init/enable（v2.5.2+） | 在 host 栈前调用 `esp_hosted_bt_controller_init/enable` |
| 改了 MAC 没生效 | 在 enable 之后才 set | MAC set 必须在 enable 之前 |
| 同时开了 Hosted HCI 和标准 HCI | 两者互斥 | 只留其一 |
| ESP32 协处理器 BT 版本不匹配 | v4.2 only | host 栈配置为 v4.2 |
| Classic BT 不可用 | 协处理器非 ESP32 | 仅 ESP32 支持 Classic BT + BLE 4.2 |

## 参考

- `host/esp_hosted_misc.h`（控制器与 MAC API）
- `host/esp_hosted_bluedroid.h`、`host/esp_hosted_bt.h`（BlueDroid HCI 衔接）
- `docs/bluetooth_design.md`（Hosted HCI 与标准 HCI 完整初始化/收发说明）
- `docs/migration_guide.md`（v2.5.2 控制器默认关闭的迁移说明）
- `examples/host_bluedroid_host_only/`、`examples/host_nimble_bleprph_host_only_vhci/`
