---
name: esp-thread-br-skill
description: >-
  AI Skill for Espressif Thread Border Router (esp-thread-br) SDK firmware development on ESP-IDF.
  Use when creating, configuring, building, or debugging a Thread Border Router based on ESP32 host SoC
  plus ESP32-H2/C6 RCP, including Wi-Fi/Ethernet backbone, IPv6 connectivity, mDNS, NAT64, TREL, RCP
  update, HTTP OTA, Web GUI, and OpenThread CLI extensions.
  Trigger words: "esp-thread-br", "Thread Border Router", "OpenThread", "border router", "RCP", "ESP32-H2",
  "ESP32-S3", "802.15.4", "NAT64", "TREL", "SRP", "Thread", "线程边界路由器", "Thread 边界路由器", "嘉立创", "乐鑫"
tags:
  - embedded
  - esp-idf
  - openthread
  - thread
  - 802.15.4
  - border-router
  - esp32
  - esp32-s3
  - esp32-h2
  - ipv6
  - firmware
license: Apache-2.0
compatibility: ESP-IDF (>=5.1.0, recommended v5.5.4); host SoC ESP32-S3 / ESP32-P4 / ESP32-C5 / ESP32-C61, RCP ESP32-H2 or ESP32-C6
metadata:
  author: Community
  version: "1.1.0"
---

# esp-thread-br-skill

针对乐鑫 (Espressif) 官方 Thread Border Router SDK (esp-thread-br) 的 AI Skill。提供基于仓库真实文档与代码的场景化 Recipe、API/配置参考、常见陷阱与构建工作流。SDK 构建于 ESP-IDF 与 OpenThread 之上，由一颗 ESP32 系列主控 SoC 运行 Border Router 固件，并通过 UART/SPI 连接一颗 ESP32-H2/C6 作为 802.15.4 Radio Co-Processor (RCP)。

## Core Principles

1. **双芯片架构不可省略** — Thread Border Router 必须由主控 SoC（ESP32-S3/P4/C5/C61 等）+ RCP（ESP32-H2 或 ESP32-C6）组成；RCP 镜像由主控通过串口烧写，不存在单芯片方案。
2. **OpenThread API 才是线程网络操作入口** — 形网/入网、dataset、SRP 等"线程网络层"操作使用 OpenThread API（`otDatasetCreateNewNetwork`、`otThreadSetEnabled` 等），见 https://openthread.io/reference；esp-thread-br 仅额外提供 Border Router/RCP/OTA/Web 等 ESP 扩展。
3. **Backbone Netif 必须显式绑定** — 在 `esp_openthread_border_router_init()` 之前必须调用 `esp_openthread_set_backbone_netif(get_example_netif())`，否则 Wi-Fi/Ethernet 与 Thread 之间无法转发。
4. **启动顺序：init → start → set_backbone → auto_start** — `nvs_flash_init` → `esp_netif_init` → `esp_event_loop_create_default` → `esp_openthread_start(config)` → `esp_openthread_set_backbone_netif()` → `esp_openthread_border_router_init()` → `esp_openthread_auto_start(&dataset)`。
5. **RCP 更新依赖 SPIFFS 分区 `rcp_fw`** — 启用 `CONFIG_AUTO_UPDATE_RCP=y` 时必须挂载 `rcp_fw` SPIFFS 分区，并调用 `esp_rcp_update_init(&rcp_update_config)`；构建期 `ot_rcp` 镜像会被 `create_ota_image.py` 自动打包进该分区。
6. **lwIP 关键配置缺一不可** — `LWIP_IPV6_NUM_ADDRESSES`、`LWIP_IPV6_FORWARD`、`LWIP_HOOK_IP6_ROUTE_DEFAULT`、`LWIP_HOOK_ND6_GET_GW_DEFAULT`、`LWIP_FORCE_ROUTER_FORWARDING` 等必须按 `sdkconfig.defaults` 设置，否则双向 IPv6 与路由通告不工作。
7. **`LWIP_IPV6_NUM_ADDRESSES` 与 IDF 版本绑定** — IDF v5.3.1/v5.4 及之后必须为 12，v5.3.0 及更早为 8；Border Router 库内部固定依赖该值。
8. **手动模式下必须按 CLI 序列起网** — 非 `OPENTHREAD_BR_AUTO_START`：`wifi connect` → `dataset init new` → `dataset commit active` → `ifconfig up` → `thread start`，顺序颠倒会导致 `detached` 不收敛。
9. **OTA 镜像是单一打包文件** — `ota_with_rcp_image` 由 `create_ota_image.py` 生成，包含 RCP 版本/flashargs/bootloader/partition table/firmware/host firmware 多个子文件（filetag 0~5），头由 `0xff` 起始；切勿自行拼接。
10. **自签 HTTPS OTA 必须替换信任证书** — OTA 服务器证书要替换 `examples/basic_thread_border_router/server_certs/ca_cert.pem`，并在 `app_main` 中通过 `esp_set_ota_server_cert()` 注入。
11. **Web GUI / REST 兼容 ot-br-posix** — `esp_br_web_start("/spiffs")` 启动的 REST API（`/node`、`/node/dataset/active`、`/available_network`、`/topology` 等）与 ot-br-posix 一致，可直接对接 Home Assistant 的 "OpenThread Border Router" 集成。
12. **RCP 升级需先停 Thread 与 ifconfig** — 手动 `otrcp update` 前应 `thread stop` 且 `ifconfig down`，升级完再 `ifconfig up` → `thread start`。
13. **RF 外部共存只在同频段干扰时有效，且两端 Kconfig 必须同时开** — `CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE` 须在 BR（ESP32-S3）与 ot_rcp（ESP32-H2）镜像中都开启，引脚/线数一致；仅当 Wi-Fi 与 802.15.4 信道频率相近、干扰明显时才有收益，否则无意义（见 `docs/en/dev-guide/build_and_run.rst` 2.1.3.5）。
14. **Credential Sharing 依赖 Commissioner，`OPENTHREAD_BR_START_WEB` 已自动 select** — 开 Web Server 会自动 `select OPENTHREAD_COMMISSIONER + OPENTHREAD_JOINER`，无需也不应手动选 Commissioner；ePSKc 经 `otBorderAgentEphemeralKeyStart` 通告 `meshcop-e`，Commissioner 用其建 DTLS 会话。

## When to Use

**Applicable:**
- 基于 esp-thread-br `basic_thread_border_router` / `m5stack_thread_border_router` 示例创建或定制 Thread Border Router 固件
- 配置 Wi-Fi 或 Ethernet backbone、手动/自动起网、Thread dataset
- 启用并验证双向 IPv6 连通、组播转发、服务发现 (SRP/mDNS)、NAT64、TREL、DHCPv6 PD
- 启用 Thread 1.4 Credential Sharing（ePSKc / Commissioner / meshcop-e）
- 集成 RCP 自动更新、HTTP OTA、Web GUI、Home Assistant
- 使用 ESP 扩展 CLI 命令（`wifi`、`otrcp`、`ota`、`curl`、`dns64server`、`mcast`、`ip`、`br pd` 等）
- 切换通信接口 (UART/SPI)、引脚配置、外部 RF 共存（3/4 线 Wi-Fi↔802.15.4）

**Not applicable:**
- OpenThread 栈本身的内部 API 细节（参考 https://openthread.io/reference）
- 纯 ESP-IDF Wi-Fi/Ethernet 通用问题（参考 ESP-IDF 文档）
- ot-br-posix（Linux 版）部署，除非涉及 `ot_rcp` 作为其 RCP 的兼容性配置
- 非 Thread 的 Zigbee/Matter 协议栈开发

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读取对应 recipe**，其中包含完整调用链、分步说明、常见错误与代码示例。

### 构建与基础配置

| recipe | scenario |
|---|---|
| `recipes/build_and_run.md` | 拉取仓库、设置 IDF、构建 RCP 镜像、配置并烧录 basic_thread_border_router |
| `recipes/board_and_interface.md` | 板型选择、UART/SPI 接口与 RCP/Reset/Boot 引脚配置、Standalone 接线 |
| `recipes/auto_start_mode.md` | 启用自动起网：Wi-Fi 连接 + dataset 自动建网 + BR 角色 |

### 网络特性

| recipe | scenario |
|---|---|
| `recipes/bidirectional_ipv6.md` | 双向 IPv6 连通：Linux 主机 accept_ra 配置 + ping 验证 |
| `recipes/multicast_forwarding.md` | 组播转发：admin-local (ff04) 组、ICMP/UDP 跨网段 |
| `recipes/service_discovery.md` | 服务发现：SRP 注册 + mDNS 互发 (avahi) |
| `recipes/nat64.md` | NAT64：DNS64 server + curl 访问 IPv4 互联网 |
| `recipes/trel.md` | TREL：Wi-Fi 设备经 TREL 加入 Thread 网络 |
| `recipes/dhcpv6_pd.md` | DHCPv6 Prefix Delegation：Kea 服务器 + `ot br pd enable/state/omrprefix` 下发前缀 |
| `recipes/credential_sharing.md` | Credential Sharing：ePSKc / meshcop-e / Commissioner DTLS 共享凭据（M5Stack） |

### 维护与扩展

| recipe | scenario |
|---|---|
| `recipes/web_gui.md` | Web GUI 与 REST API：`/node`、`/available_network`、`/topology`、Home Assistant |
| `recipes/rcp_update.md` | RCP 更新机制：自动/手动 (`otrcp update`)、序列号与回滚 |
| `recipes/http_ota.md` | HTTPS OTA：本地 openssl 服务器、自签证书、`ota download` |
| `recipes/rf_coexistence.md` | RF 外部共存：3/4 线 Wi-Fi↔802.15.4、`ot_external_coexist_init`、两端 Kconfig |

---

## 主控 / RCP 组合与引脚参考

仓库 `README.md` 与 `examples/basic_thread_border_router/README_standalone_RCP.md` 给出的推荐组合与默认引脚（来源：`Kconfig.projbuild`、`esp_ot_config.h`）。

### 推荐主控 + RCP 组合

| 主控 SoC | RCP | 适用场景 |
|---|---|---|
| ESP32-P4 | ESP32-C6 | 高性能 / AI/ML |
| ESP32-C5 | ESP32-H2 | 双频 Wi-Fi (2.4G + 5G) |
| ESP32-C61 | ESP32-H2 | Wi-Fi 6 (802.11ax) |
| ESP32-S3 | ESP32-H2 | 通用 / 带屏带摄像头 (官方 BR 板默认) |

### 官方 BR 开发板 (ESP_BR_BOARD_DEV_KIT) 默认引脚

| 配置项 | 默认 GPIO | 含义 |
|---|---|---|
| `CONFIG_PIN_TO_RCP_TX` | 17 | 主控 UART RX ← RCP TX |
| `CONFIG_PIN_TO_RCP_RX` | 18 | 主控 UART TX → RCP RX |
| `CONFIG_PIN_TO_RCP_RESET` | 7 | RCP RESET |
| `CONFIG_PIN_TO_RCP_BOOT` | 8 | RCP BOOT (ESP32-H2 GPIO8) |
| `CONFIG_PIN_TO_RCP_CS` | 10 | SPI CS（仅 SPI 模式） |
| `CONFIG_PIN_TO_RCP_SCLK` | 12 | SPI SCLK |
| `CONFIG_PIN_TO_RCP_MISO` | 13 | SPI MISO |
| `CONFIG_PIN_TO_RCP_MOSI` | 11 | SPI MOSI |

### Standalone 接线 (ESP_BR_BOARD_STANDALONE, UART)

| 主控 | RCP (H2/C6) |
|---|---|
| GND | G |
| 5V | 5V |
| GPIO4 (UART RX) | TX |
| GPIO5 (UART TX) | RX |
| GPIO7 | RST |
| GPIO8 | GPIO9 (BOOT) |

> 注意：ESP32-S3 的 GPIO17/GPIO18 驱动电流不同，作 host 时不建议用作 UART RX/TX，推荐 GPIO4/GPIO5（见 `docs/en/qa.rst` 5.2）。

---

## 关键 Kconfig 速查

| Kconfig 选项 | 作用 | 备注 |
|---|---|---|
| `CONFIG_OPENTHREAD_ENABLED` | 启用 OpenThread | 必选 |
| `CONFIG_OPENTHREAD_BORDER_ROUTER` | Border Router 角色 | 必选 |
| `CONFIG_OPENTHREAD_RADIO_SPINEL_UART` / `..._SPI` | 主控↔RCP 通信接口 | 默认 UART |
| `CONFIG_AUTO_UPDATE_RCP` | 启动时自动更新 RCP | 需挂载 `rcp_fw` 分区 |
| `CONFIG_CREATE_OTA_IMAGE_WITH_RCP_FW` | 构建期打包 `ota_with_rcp_image` | OTA 必备 |
| `CONFIG_OPENTHREAD_RCP_COMMAND` | 启用 `otrcp` CLI | 手动 RCP 更新 |
| `CONFIG_OPENTHREAD_CLI_OTA` | 启用 `ota` CLI + `esp_set_ota_server_cert` | HTTP OTA |
| `CONFIG_OPENTHREAD_CLI_WIFI` | 启用 `wifi` CLI + NVS 配置 | 手动/自动 Wi-Fi |
| `CONFIG_OPENTHREAD_DNS64_CLIENT` | 启用 `dns64server` CLI | NAT64 |
| `CONFIG_OPENTHREAD_NVS_DIAG` | 启用 `nvsdiag` CLI | 调试 |
| `CONFIG_OPENTHREAD_BR_LIB_CHECK` | 启用 `brlibcheck` CLI | 库兼容性检查 |
| `CONFIG_OPENTHREAD_BR_AUTO_START` | 自动起网 | 与手动互斥 |
| `CONFIG_OPENTHREAD_BR_START_WEB` | 启动 Web Server | `select OPENTHREAD_COMMISSIONER/JOINER` |
| `CONFIG_OPENTHREAD_BR_SOFTAP_SETUP` | SoftAP 配网模式 | 首次启动 AP `ESP-ThreadBR-XXXX` |
| `CONFIG_OPENTHREAD_RADIO_TREL` | 启用 TREL | eventfd +1 |
| `CONFIG_LWIP_IPV6_NUM_ADDRESSES` | 每网口 IPv6 地址数 | IDF v5.3.1+ 必须 12 |
| `CONFIG_ESP_BR_BOARD_DEV_KIT` / `_STANDALONE` / `_M5STACK_CORES3` | 板型 | 决定默认引脚 |
| `CONFIG_ESP_BR_H2_TARGET` / `CONFIG_ESP_BR_C6_TARGET` | RCP 目标芯片 | 决定 `ESP_BR_RCP_TARGET_ID` |

---

## 分区表 (partitions.csv, 官方 BR 板)

```
nvs,        data, nvs,      , 0x6000,
otadata,    data, ota,      , 0x2000,
phy_init,   data, phy,      , 0x1000,
ota_0,      app,  ota_0,    , 2M,
ota_1,      app,  ota_1,    , 2M,
web_storage,data, spiffs,   , 200K,
rcp_fw,     data, spiffs,   , 640K,
```

> 4MB Flash 变体需把 `ota_0`/`ota_1` 改为 1500K（见 `examples/basic_thread_border_router/README.md` Troubleshooting）。

---

## Critical Pitfalls (Must Read)

以下是最常见错误，违反任一条都会导致固件不工作。

### 1. 忘记绑定 Backbone Netif

```c
// ❌ WRONG — 不绑定 backbone，Thread 与 Wi-Fi 互不可达
ESP_ERROR_CHECK(esp_openthread_border_router_init());
esp_openthread_auto_start(&dataset);

// ✅ CORRECT — 先设 backbone 再 init
esp_openthread_lock_acquire(portMAX_DELAY);
esp_openthread_set_backbone_netif(get_example_netif());
ESP_ERROR_CHECK(esp_openthread_border_router_init());
ESP_ERROR_CHECK(esp_openthread_auto_start(&dataset));
esp_openthread_lock_release();
```

### 2. 起网顺序颠倒

```bash
# ❌ WRONG — 先 thread start 再 ifconfig up，永远 detached
> thread start
> ifconfig up

# ✅ CORRECT — dataset → ifconfig up → thread start
> dataset init new
> dataset commit active
> ifconfig up
> thread start
```

### 3. LWIP_IPV6_NUM_ADDRESSES 与 IDF 版本不符

```bash
# ❌ WRONG — IDF v5.4 仍用 8
CONFIG_LWIP_IPV6_NUM_ADDRESSES=8

# ✅ CORRECT — v5.3.1 / v5.4 及之后必须 12
CONFIG_LWIP_IPV6_NUM_ADDRESSES=12
```

### 4. 缺少 lwIP 路由相关 hook

```bash
# ❌ WRONG — 只开 IPv6 forwarding，不设 hook，RA/路由不生效
CONFIG_LWIP_IPV6_FORWARD=y

# ✅ CORRECT — 必须配套（见 sdkconfig.defaults）
CONFIG_LWIP_IPV6_FORWARD=y
CONFIG_LWIP_HOOK_IP6_ROUTE_DEFAULT=y
CONFIG_LWIP_HOOK_ND6_GET_GW_DEFAULT=y
CONFIG_LWIP_HOOK_IP6_INPUT_CUSTOM=y
CONFIG_LWIP_IPV6_AUTOCONFIG=y
CONFIG_LWIP_HOOK_IP6_SELECT_SRC_ADDR_CUSTOM=y
CONFIG_LWIP_FORCE_ROUTER_FORWARDING=y
```

### 5. RCP 自动更新未挂载 SPIFFS

```c
// ❌ WRONG — CONFIG_AUTO_UPDATE_RCP=y 但没挂 rcp_fw，update 找不到镜像
ESP_ERROR_CHECK(esp_rcp_update_init(&rcp_update_config));

// ✅ CORRECT — 先 init_spiffs 注册 rcp_fw 分区，再 init update
static esp_err_t init_spiffs(void) {
#if CONFIG_AUTO_UPDATE_RCP
    esp_vfs_spiffs_conf_t rcp_fw_conf = {
        .base_path = "/" CONFIG_RCP_PARTITION_NAME,
        .partition_label = CONFIG_RCP_PARTITION_NAME,
        .max_files = 10, .format_if_mount_failed = false};
    ESP_RETURN_ON_ERROR(esp_vfs_spiffs_register(&rcp_fw_conf), TAG, "mount rcp fw");
#endif
    return ESP_OK;
}
// app_main:
ESP_ERROR_CHECK(init_spiffs());
ESP_ERROR_CHECK(esp_rcp_update_init(&rcp_update_config));
```

### 6. eventfd 数量不足

```c
// ❌ WRONG — 固定 3 个 eventfd，启用 SPI/TREL 后崩溃
esp_vfs_eventfd_config_t eventfd_config = { .max_fds = 3 };

// ✅ CORRECT — 按需累加
size_t max_eventfd = 3;
#if CONFIG_OPENTHREAD_RADIO_SPINEL_SPI
    max_eventfd++;   // SpiSpinelInterface
#endif
#if CONFIG_OPENTHREAD_RADIO_TREL
    max_eventfd++;   // TREL reception
#endif
esp_vfs_eventfd_config_t eventfd_config = { .max_fds = max_eventfd };
ESP_ERROR_CHECK(esp_vfs_eventfd_register(&eventfd_config));
```

### 7. OTA 信任证书未替换 / 未注入

```c
// ❌ WRONG — 用仓库默认 ca_cert.pem 连自建服务器，握手失败
> ota download https://my-host:8070/ota_with_rcp_image

// ✅ CORRECT — 用生成证书替换 server_certs/ca_cert.pem，重新 fullclean build
// 并在 app_main 注入：
#if CONFIG_OPENTHREAD_CLI_OTA
    esp_set_ota_server_cert((char *)server_cert_pem_start);  // _binary_ca_cert_pem_start
#endif
```

### 8. 手动 RCP 更新未停 Thread

```bash
# ❌ WRONG — Thread 仍在跑时直接 update，RCP 复位导致网络崩溃
> otrcp update

# ✅ CORRECT — 先停线程再 ifconfig down
> thread stop
> ifconfig down
> otrcp update
> ifconfig up
> thread start
```

### 9. ESP32-S3 用错 UART 引脚

```c
// ❌ WRONG — ESP32-S3 上用 GPIO17/18 做 RCP UART，驱动电流不一致，通信超时
//   警告：W(2499) OPENTHREAD:[W] P-SpinelDrive-: Wait for response timeout

// ✅ CORRECT — ESP32-S3 host 推荐 GPIO4(RX)/GPIO5(TX)，并在 esp_ot_config.h 与 ot_rcp 端同步修改
.radio_uart_config = {
    .port = 1,
    .uart_config = { .baud_rate = 460800, ... },
    .rx_pin = 4,   // CONFIG_PIN_TO_RCP_TX
    .tx_pin = 5,   // CONFIG_PIN_TO_RCP_RX
},
```

### 10. Ethernet 模式未开 AUTO_START

```c
// ❌ WRONG — 关 OPENTHREAD_BR_AUTO_START 又用 Ethernet
//   app_main 里 #error: 目前不支持手动连 Ethernet

// ✅ CORRECT — Ethernet backbone 必须启用 OPENTHREAD_BR_AUTO_START
//   menuconfig: OPENTHREAD_BR_AUTO_START=y
//   且 EXAMPLE_CONNECT_ETHERNET=y, EXAMPLE_CONNECT_WIFI=n
```

### 11. SPI 模式未同步启用 ot_rcp 端选项

```bash
# ❌ WRONG — 只在 BR 端开 SPI，ot_rcp 端仍 UART，两边握手失败
#   BR: OPENTHREAD_RADIO_SPINEL_SPI=y

# ✅ CORRECT — 两端都改 SPI
#   ot_rcp:    OPENTHREAD_RCP_SPI=y
#   BR:        OPENTHREAD_RADIO_SPINEL_SPI=y
#   并在 esp_ot_config.h / ot_rcp esp_ot_config.h 同步 GPIO
```

### 12. OTA 镜像被当普通二进制直接烧

```bash
# ❌ WRONG — 把 ota_with_rcp_image 当作 app 直接 esptool 烧写
esptool.py write_flash 0x0 ota_with_rcp_image

# ✅ CORRECT — 它是 OTA bundle（含 filetag 0~5 子文件），只能通过运行时下载触发：
#   > ota download https://host:8070/ota_with_rcp_image
#   解析后先更新 BR app 分区，再按需更新 RCP（经 rcp_fw SPIFFS）
```

### 13. Wi-Fi ping 看似丢包（实际是 lwIP 吞了回包）

```bash
# ❌ WRONG — 用 OT 层 `ot ping` ping BR 自己的 Wi-Fi 地址，必显示 100% loss
> ot ping <br-wifi-global-addr>

# ✅ CORRECT — 这是设计行为（lwIP 收到回包因目的=自己不再上送 OT 层）
#   见 docs/en/qa.rst 5.1；要验证请在 lwIP 层 ping，或 ping 网络中其他设备
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 确认主控/RCP 型号、backbone (Wi-Fi/Ethernet)、是否自动起网、是否 Web/OTA/RCP 自动更新 |
| 2 | Recipe | 匹配 `recipes/` 场景，按其调用链与 CLI 序列执行 |
| 3 | Query | API/配置不在 recipe 内，查 `resources/api_reference.md`、`config_reference.md` |
| 4 | Validate | 校验 Kconfig、引脚、分区表、lwIP hook、eventfd 数量与 IDF 版本匹配 |
| 5 | Confirm | 向用户确认：板型、接口、起网模式、OTA 证书 |
| 6 | Execute | **新工程**：以 `examples/basic_thread_border_router` 或 `examples/m5stack_thread_border_router` 为模板复制修改；**已有工程**：原地编辑 |
| 7 | Build | 先构建 `ot_rcp`（自动打包），再 `idf.py set-target <host>` + `idf.py build` |
| 8 | Flash | `idf.py -p PORT flash monitor` |
| 9 | Verify | 看 RCP reset/API Version、`OPENTHREAD: OpenThread attached to netif`、`Role detached -> leader` |

### Step 6 Detail — 工程创建策略

**目标目录无现有工程（首次创建）：**

1. 按需求选最贴近的示例：
   - 通用 BR / 官方开发板 → `examples/basic_thread_border_router`
   - M5Stack CoreS3 带屏 / 凭证共享 → `examples/m5stack_thread_border_router`
   - Standalone 模组（ESP32-C5/P4/C61 + 外接 RCP）→ `examples/basic_thread_border_router` + `README_standalone_RCP.md`
2. 复制整个示例目录，保留 `main/`、`examples/common/thread_border_router`、`server_certs/`、`partitions.csv`、`sdkconfig.defaults*` 结构。
3. 按目标修改：`esp_ot_config.h`（引脚/接口）、`sdkconfig.defaults`（板型/目标/特性）、`server_certs/ca_cert.pem`（OTA）。
4. 说明复制与修改了什么。

**目标目录已有工程：** 原地编辑，除非用户要求否则不覆盖。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 不在 esp-thread-br 仓库内 | 该 API 多半属 OpenThread 栈或 ESP-IDF，转 https://openthread.io/reference 或 ESP-IDF 文档 |
| `Wait for response timeout` (Spinel) | 检查 RCP UART/SPI 引脚与波特率（默认 460800），ESP32-S3 避开 GPIO17/18 |
| 起网后一直 `detached` | 核对 dataset 是否 commit、`ifconfig up` 在 `thread start` 之前 |
| Linux 主机 ping 不到 Thread 设备 | `accept_ra=2`、`accept_ra_rt_info_max_plen=128` |
| NAT64 curl 失败 | 先 `ot dns64server <v4-dns>`；确认 backbone 联网 |
| OTA 握手失败 | 替换 `server_certs/ca_cert.pem`，`fullclean` 后重建 |
| RCP 自动更新找不到镜像 | 确认 `CONFIG_AUTO_UPDATE_RCP=y` 且 `rcp_fw` SPIFFS 已挂载 |
| Web GUI 打不开 | `CONFIG_OPENTHREAD_BR_START_WEB=y`，浏览器访问 `http://<BR-IPv4>:80/index.html` |
| Ethernet 不可用手动模式 | Ethernet backbone 必须 `OPENTHREAD_BR_AUTO_START=y` |

## References

- 场景 recipes → `recipes/` 目录
- ESP 扩展 API 参考 → `resources/api_reference.md`
- Kconfig / sdkconfig 参考 → `resources/config_reference.md`
- 陷阱合集 → `resources/pitfalls.md`
- 示例清单 → `resources/example_list.md`
- 仓库内权威文档 → `docs/en/`（hardware_platforms / dev-guide / codelab / api-reference / qa）
