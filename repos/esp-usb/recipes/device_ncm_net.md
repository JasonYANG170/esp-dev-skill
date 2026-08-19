# USB 设备：NCM / RNDIS 以太网（Ethernet-over-USB）

> **适用摘要**: 用 `tinyusb_net` 把 ESP32 暴露为一个 USB 网络接口（CDC-NCM 或 ECM/RNDIS），通过 `tinyusb_net_init` 注册收包/TX 释放回调，并用 `tinyusb_net_send_sync` / `tinyusb_net_send_async` 收发以太网帧。常用于 USB tethering、嵌入式 USB 以太网。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-usb/resources/`, source/examples in `repos/esp-usb/`, and this recipe path `repos/esp-usb/recipes/device_ncm_net.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "USB 以太网"
- "NCM / RNDIS 设备"
- "Ethernet-over-USB"
- "USB tethering"
- "tinyusb_net"
- "把 ESP32 当网卡"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件依赖 | `idf.py add-dependency "espressif/esp_tinyusb^2.2.0"` |
| ESP-IDF | `>= 5.0` |
| menuconfig | 网络 mode 选 NCM 或 ECM/RNDIS（见步骤 1） |
| 参考头文件 | `device/esp_tinyusb/include/tinyusb_net.h` |

> 这是**设备**侧网络。`tinyusb_net.h` 的全部声明都在 `#if (CONFIG_TINYUSB_NET_MODE_NONE != 1)` 内——若 menuconfig 选了 “None”，头文件为空，链接会失败。

## 分步说明

### 1. menuconfig 选网络模式

在 `Component config → TinyUSB Stack → Network driver (ECM/NCM/RNDIS)` 选其一：

| Kconfig | 含义 | 宿主支持 |
|---|---|---|
| `CONFIG_TINYUSB_NET_MODE_NCM` | CDC-NCM | Linux / Windows 10+ / macOS |
| `CONFIG_TINYUSB_NET_MODE_ECM_RNDIS` | CDC-ECM + RNDIS（复合） | Windows（RNDIS）/ Linux（ECM） |
| `CONFIG_TINYUSB_NET_MODE_NONE` | 关闭（默认） | — |

NCM 模式下还可调（默认值见 Kconfig）：

- `CONFIG_TINYUSB_NCM_OUT_NTB_BUFFS_COUNT`（默认 3，范围 1–6）：接收侧 NTB 缓冲数。
- `CONFIG_TINYUSB_NCM_IN_NTB_BUFFS_COUNT`（默认 3，范围 1–6）：发送侧 NTB 缓冲数。
- `CONFIG_TINYUSB_NCM_OUT_NTB_BUFF_MAX_SIZE`（默认 3200，范围 1600–10240）：接收 NTB 字节，须显著大于 MTU(1500) 且为 4 的倍数。
- `CONFIG_TINYUSB_NCM_IN_NTB_BUFF_MAX_SIZE`（默认 3200）：发送 NTB 字节。

> 出现 `tud_network_can_xmit: request blocked` 告警时，增大上述缓冲数/尺寸。

也可在 `sdkconfig.defaults` 直接写：

```
CONFIG_TINYUSB_NET_MODE_NCM=y
```

### 2. 实现 RX 与 TX 释放回调

```c
#include "tinyusb_net.h"

// 收到一帧以太网包（buffer/len）。返回值当前被忽略。
static esp_err_t on_recv(void *buffer, uint16_t len, void *ctx) {
    // 在此把帧交给 lwip / 你的网络栈
    return ESP_OK;
}

// 应用此前通过 send 传入的 buffer 用完了，可在此释放
// buff_free_arg 是 send 时传入的用户令牌（通常是缓冲指针）
static void free_tx_buffer(void *buff_free_arg, void *ctx) {
    free(buff_free_arg);
}

// 可选：TinyUSB 内部在 tud_network_init_cb 时回调
static void on_net_init(void *ctx) {
    // 初始化你的网络栈
}
```

### 3. 安装 tinyusb_net 与设备驱动

`tinyusb_net_init()` 必须在 `tinyusb_driver_install()` **之前**调用，以便网络类回调先就位。MAC 地址由应用提供（6 字节）。

```c
#include "tinyusb.h"
#include "tinyusb_default_config.h"
#include "tinyusb_net.h"

uint8_t mac[6] = {0x02, 0x00, 0x00, 0x12, 0x34, 0x56};  // locally administered

const tinyusb_net_config_t net_config = {
    .mac_addr         = { mac[0], mac[1], mac[2], mac[3], mac[4], mac[5] },
    .on_recv_callback = on_recv,
    .free_tx_buffer   = free_tx_buffer,
    .on_init_callback = on_net_init,
    .user_context     = NULL,
};
ESP_ERROR_CHECK(tinyusb_net_init(&net_config));

tinyusb_config_t tusb_cfg = TINYUSB_DEFAULT_CONFIG();
ESP_ERROR_CHECK(tinyusb_driver_install(&tusb_cfg));
```

> **v2 API 变更**：旧原型 `tinyusb_net_init(TINYUSB_USBDEV_0, &cfg)` 已废弃。`TINYUSB_USBDEV_0` 被移除，现在只接受一个参数 `&cfg`。若编译报 `'TINYUSB_USBDEV_0' undeclared` 或 `too many arguments to function 'tinyusb_net_init'`，删掉第一参数即可。

### 4. 发送以太网帧

同步与异步可混用。同步首次发送会分配同步原语，略增堆占用。

```c
// 同步发送（阻塞到被 TinyUSB 接受或超时）
uint8_t *frame = malloc(frame_len);   // 用 heap 才能用 free_tx_buffer 释放
memcpy(frame, my_eth_frame, frame_len);
esp_err_t ret = tinyusb_net_send_sync(frame, frame_len, frame, pdMS_TO_TICKS(1000));
// 第 3 个参数 buff_free_arg 会传给 free_tx_buffer

// 异步发送（仅入队，不保证 USB 真发出）
tinyusb_net_send_async(frame, frame_len, frame);
```

> `ESP_OK`（异步）只表示已入队到 TinyUSB 任务，不代表宿主已收到。用异步时务必通过 `free_tx_buffer` 释放缓冲。

### 5. 反初始化与卸载

```c
tinyusb_net_deinit();
tinyusb_driver_uninstall();
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `tinyusb_net.h` 内容为空 / 链接失败 | menuconfig 选了 `NET_MODE_NONE` | 改选 `CONFIG_TINYUSB_NET_MODE_NCM=y` 或 `ECM_RNDIS` |
| `'TINYUSB_USBDEV_0' undeclared` / 参数过多 | 用了旧 v1 API | 删掉 `TINYUSB_USBDEV_0`，改 `tinyusb_net_init(&cfg)` |
| `tud_network_can_xmit: request blocked` | NTB 缓冲不足 | 增大 `CONFIG_TINYUSB_NCM_IN/OUT_NTB_BUFFS_COUNT` 与 `..._NTB_BUFF_MAX_SIZE` |
| `ESP_ERR_INVALID_STATE` send 失败 | 设备未枚举/未 mount | 等 `TINYUSB_EVENT_ATTACHED` 后再发；先 `tinyusb_driver_install` |
| 宿主不识别网卡 | NCM 在老 Windows 无驱动 | 改用 `ECM_RNDIS`（Windows 走 RNDIS），或宿主装 NCM 驱动 |
| NTB 尺寸报错 | 不是 4 的倍数或 < 2048 | `CONFIG_TINYUSB_NCM_*_NTB_BUFF_MAX_SIZE` 设为 ≥ 2048 且为 4 的倍数 |
| 同步发送超时 | 宿主未及时读端点 | 增大 `timeout`，或确认宿主网卡已 up |

## 参考

- `device/esp_tinyusb/include/tinyusb_net.h` — `tinyusb_net_config_t`、`tinyusb_net_init`、`tinyusb_net_send_sync`/`_send_async`、回调类型
- `device/esp_tinyusb/Kconfig` — `CONFIG_TINYUSB_NET_MODE_*` 选择与 `CONFIG_TINYUSB_NCM_*` 缓冲调优
- `docs/device/migration-guides/v2/tinyusb_ncm.md` — v2 API 变更（去掉 `TINYUSB_USBDEV_0`）
- 测试应用：`device/esp_tinyusb/test_apps/ncm/`（`main/test_ncm.c`：init → `TINYUSB_DEFAULT_CONFIG` → `tinyusb_driver_install` → 等待 mount → deinit；`sdkconfig.defaults` 设 `CONFIG_TINYUSB_NET_MODE_NCM=y`）
- 相关（ESP-IDF）：`examples/peripherals/usb/device/tusb_ncm`（Wi-Fi 透传的完整应用示例）
