# Thread Border Router（ESP32-S3 + ESP32-H2）

> **适用摘要**: 用 ESP Thread Border Router 板（ESP32-S3 主控 + ESP32-H2 作 15.4 RCP）搭一个 Matter Thread Border Router：烧 RCP 固件到 H2、烧 BR 固件到 S3、commission BR 后用 ThreadBorderRouterManagement cluster 配置 Thread 网络，再 commission Thread 终端设备入网。

## 触发意图

- "做 Thread Border Router / 边界路由器"
- "ESP32-H2 当 RCP"
- "ThreadBorderRouterManagement set-active-dataset-request"
- "esp_rcp_update 自动刷 RCP"
- "esp32c5 / esp32c6 / esp32h2 部署 Thread Matter"
- "matter esp ot_cli dataset"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP Thread Border Router 板（集成 ESP32-S3 + ESP32-H2） |
| 参考工程 | `examples/thread_border_router/`（专用）或 `examples/controller/` + `sdkconfig.defaults.otbr`（controller 兼 BR） |
| RCP 固件 | `esp-idf/examples/openthread/ot_rcp`（烧到 ESP32-H2） |
| 关键 Kconfig | `CONFIG_OPENTHREAD_ENABLED=y`、`CONFIG_OPENTHREAD_BORDER_ROUTER=y`、`CONFIG_ENABLE_WIFI_STATION=y`、`CONFIG_USE_MINIMAL_MDNS=n` |
| 工具 | chip-tool（commission BR + Thread 终端） |

## 分步说明

### 1. 烧 RCP 固件到 ESP32-H2

Border Router 板上的 H2 作 15.4 Radio Co-Processor。两种方式（来自 `examples/thread_border_router/README.md`）：

**方式 A — 直接烧 ot_rcp：**

```bash
cd /path/to/esp-idf/examples/openthread/ot_rcp
idf.py set-target esp32h2 build
idf.py -p <H2_PORT> erase-flash flash
```

**方式 B — 启用 `CONFIG_AUTO_UPDATE_RCP` 自动更新：**

先 build ot_rcp（不烧），再启用 BR 侧的 `CONFIG_AUTO_UPDATE_RCP=y`。烧 BR 固件到 S3 时，BR 启动会自动把 RCP 镜像经串口刷到 H2（见 `esp_rcp_update` 组件）。`examples/controller/main/app_main.cpp` 中的相关代码：

```cpp
#if defined(CONFIG_AUTO_UPDATE_RCP)
    esp_vfs_spiffs_conf_t rcp_fw_conf = {
        .base_path = "/rcp_fw", .partition_label = "rcp_fw",
        .max_files = 10, .format_if_mount_failed = false
    };
    esp_vfs_spiffs_register(&rcp_fw_conf);
    esp_rcp_update_config_t rcp_update_config = ESP_OPENTHREAD_RCP_UPDATE_CONFIG();
    esp_rcp_update_init(&rcp_update_config);
    esp_ot_register_rcp_handler();
#endif
```

启动时还会在持锁状态下比对版本并按需更新 RCP，更新前后临时关掉再重开 Thread：

```cpp
esp_matter::lock::ScopedChipStackLock lock(portMAX_DELAY);
using namespace chip::DeviceLayer;
bool thread_was_enabled = ThreadStackMgr().IsThreadEnabled();
if (thread_was_enabled) ThreadStackMgr().SetThreadEnabled(false);
esp_ot_update_rcp_if_different();
if (thread_was_enabled) ThreadStackMgr().SetThreadEnabled(true);
```

### 2. 烧 Border Router 固件到 ESP32-S3

`examples/thread_border_router/`：默认 8MB flash。

```bash
cd /path/to/esp-matter/examples/thread_border_router
idf.py set-target esp32s3
idf.py build
idf.py -p <S3_PORT> erase-flash flash monitor
```

若用 `examples/controller/` 兼做 BR（同时是 controller/commissioner）：

```bash
idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults.otbr" set-target esp32s3 build
idf.py -p <PORT> erase-flash flash monitor
```

> Thread Border Router DevKit Board 用 USB 口。

### 3. BR 必备 Kconfig（`sdkconfig.defaults.otbr`）

```text
# OpenThread Border Router
CONFIG_OPENTHREAD_ENABLED=y
CONFIG_OPENTHREAD_BORDER_ROUTER=y
CONFIG_OPENTHREAD_SRP_CLIENT=n
CONFIG_OPENTHREAD_DNS_CLIENT=n
CONFIG_THREAD_TASK_STACK_SIZE=8192

# BR 是 Wi-Fi STA + Thread 双栈
CONFIG_ENABLE_WIFI_STATION=y
CONFIG_ENABLE_WIFI_AP=n
CONFIG_USE_MINIMAL_MDNS=n        # BR 必须用完整 mdns
CONFIG_ENABLE_EXTENDED_DISCOVERY=y

# LwIP 路由 / 转发
CONFIG_LWIP_IPV6_AUTOCONFIG=y
CONFIG_LWIP_IPV6_NUM_ADDRESSES=12
CONFIG_LWIP_MULTICAST_PING=y
CONFIG_LWIP_IPV6_FORWARD=y
CONFIG_LWIP_HOOK_IP6_ROUTE_DEFAULT=y
CONFIG_LWIP_HOOK_ND6_GET_GW_DEFAULT=y

# 关掉 Matter 自带 route hook（OTBR 已初始化）
CONFIG_ENABLE_ROUTE_HOOK=n

# BR 用自定义分区表（partitions_br.csv 含 rcp_fw / paa_cert 等）
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions_br.csv"
```

### 4. 在 `app_event_cb` 里初始化 BR

S3 拿到 Wi-Fi IP 后再初始化 OpenThread BR backbone（来自 `examples/controller/main/app_main.cpp`）：

```cpp
static void app_event_cb(const ChipDeviceEvent *event, intptr_t arg) {
    if (event->Type == chip::DeviceLayer::DeviceEventType::kESPSystemEvent
        && event->Platform.ESPSystemEvent.Base == IP_EVENT
        && event->Platform.ESPSystemEvent.Id == IP_EVENT_STA_GOT_IP) {
#if CONFIG_OPENTHREAD_BORDER_ROUTER
        static bool sThreadBRInitialized = false;
        if (!sThreadBRInitialized) {
            esp_openthread_set_backbone_netif(esp_netif_get_handle_from_ifkey("WIFI_STA_DEF"));
            esp_openthread_lock_acquire(portMAX_DELAY);
            esp_openthread_border_router_init();
            esp_openthread_lock_release();
            sThreadBRInitialized = true;
        }
#endif
    }
}
```

OpenThread 平台配置（持锁前调用）：

```cpp
#ifdef CONFIG_OPENTHREAD_BORDER_ROUTER
    esp_openthread_platform_config_t config = {
        .radio_config = ESP_OPENTHREAD_DEFAULT_RADIO_CONFIG(),
        .host_config  = ESP_OPENTHREAD_DEFAULT_HOST_CONFIG(),
        .port_config  = ESP_OPENTHREAD_DEFAULT_PORT_CONFIG(),
    };
    set_openthread_platform_config(&config);
#endif
```

### 5. Commission BR（chip-tool）

先用 chip-tool 把 BR 当普通 Wi-Fi 设备 commission 入 fabric（默认 passcode/discriminator 见 `recipes/commissioning_chiptool.md`）：

```text
pairing ble-wifi 0x7283 <ssid> <passphrase> 20202021 3840
```

### 6. 用 ThreadBorderRouterManagement cluster 配置 Thread 网络

Commission 完后通过 cluster 命令把 Thread Active Dataset 推给 BR（来自 `examples/thread_border_router/README.md`）：

```bash
./chip-tool generalcommissioning arm-fail-safe 180 1 0x7283 0
./chip-tool threadborderroutermanagement set-active-dataset-request hex:<thread-dataset-tlvs> 0x7283 1
./chip-tool generalcommissioning commissioning-complete 0x7283 0
```

三步：arm fail-safe（180s）→ 写 Active Dataset → commissioning-complete。BR 收到后 form/join Thread 网络。

> `thread-dataset-tlvs` 可由已有 Thread 网络的 `ot-cli dataset active -x` 产生。

### 7. （备选）用 controller console 直接配 Thread

如果 BR 本身是 controller（`examples/controller` + otbr 配置），可在 device console 上用 `ot_cli` 直接建 Thread 网络（来自 `examples/controller/README.md`）：

```text
matter esp wifi connect <ssid> <password>
matter esp ot_cli dataset init new
matter esp ot_cli dataset commit active
matter esp ot_cli dataset active -x      # 打印 <dataset_tlvs>
matter esp ot_cli ifconfig up
matter esp ot_cli thread start
```

### 8. Commission Thread 终端设备

Thread 网络起来后，用上一步的 `dataset_tlvs` 把 Thread 终端（esp32h2 / esp32c6 / esp32c5）commission 入网：

```bash
./chip-tool pairing ble-thread 0x7384 hex:<thread-dataset-tlvs> 20202021 3840
```

或在 controller console 上：

```text
matter esp controller pairing ble-thread 1234 <dataset_tlvs> 20202021 3840
matter esp controller invoke-cmd 1234 1 6 2   # OnOff Toggle
```

### 9. RIO（路由信息选项）注意事项

Wi-Fi 产品配合 Thread BR 时，TC-SC-4.9 测试要求 Wi-Fi 设备能处理 RA 消息中的 RIO 并维护到 Thread 网络的路由（见 `docs/en/certification.rst`）。要点：

- 需要用能转发 RA 的 Wi-Fi 路由器（部分家用路由器会丢 RA）。
- ESP-IDF v5.4.3+ / v5.5.2+ / v6.x 的 LwIP 已内置 RIO 支持，此时可关掉 Matter 的 route hook 和 LwIP 两个 hook：
  ```text
  CONFIG_ENABLE_ROUTE_HOOK=n
  CONFIG_LWIP_HOOK_IP6_ROUTE_DEFAULT=n
  CONFIG_LWIP_HOOK_ND6_GET_GW_DEFAULT=n
  ```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| H2 不响应 15.4 | 没烧 ot_rcp / RCP 版本不匹配 | 直接烧 ot_rcp，或开 `CONFIG_AUTO_UPDATE_RCP` 自动更新 |
| Thread 起不来 | Active Dataset 没写 / fail-safe 没 arm | 按 arm-fail-safe → set-active-dataset-request → commissioning-complete 顺序 |
| 终端 commission 失败 | `dataset_tlvs` 与 BR 当前网络不一致 | 用 BR 上 `ot_cli dataset active -x` 的输出 |
| RA 转发不到 Wi-Fi 设备 | Wi-Fi 路由器丢了 RA | 换能转发 RA 的路由器；确认 IDF 版本支持 RIO |
| mdns 发现异常 | 用了 minimal mdns | BR 必须 `CONFIG_USE_MINIMAL_MDNS=n` |
| 双栈冲突 | 同时开 Wi-Fi AP | `CONFIG_ENABLE_WIFI_AP=n`，只用 STA |
| `route hook` 重复初始化 | Matter 和 OTBR 都装了 route hook | `CONFIG_ENABLE_ROUTE_HOOK=n`（OTBR 已初始化） |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/examples/thread_border_router/README.md` — 硬件、刷 RCP、commission、ThreadBorderRouterManagement 三步
- `D:/esp-skill/espressif-repos/esp-matter/examples/thread_border_router/`（host SoC 工程）
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/README.md`（OTBR 章节：`ot_cli dataset` / `ifconfig up` / `thread start` / `pairing ble-thread`）
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/sdkconfig.defaults.otbr`（OTBR 完整 Kconfig）
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/main/app_main.cpp`（`esp_openthread_border_router_init()` / `set_openthread_platform_config()` / `esp_ot_update_rcp_if_different()`）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/controller.rst` — OTBR pairing ble-thread 流程
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/certification.rst` — RIO / TC-SC-4.9 说明
- ESP-IDF `examples/openthread/ot_rcp`（H2 RCP 固件源）
- esp-thread-br `components/esp_rcp_update`（RCP 自动更新组件）
