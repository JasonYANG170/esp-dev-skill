# Thread 边界路由器服务

> **适用摘要**: 用 `esp_rmaker_thread_br_enable(platform_config)` 把一个 RainMaker 节点变成 **Thread Border Router (TBR)**，让 RainMaker-over-Thread 设备经 BR 上的 **NAT64** 会话接入 RainMaker 云。覆盖 ESP32-S3(主) + ESP32-H2(RCP) 分体、RCP 自动更新、dataset/`ThreadCmd` 控制、LwIP IPv6 编译要求。

## 触发意图

- "RainMaker Thread 边界路由器 / TBR"
- "esp_rmaker_thread_br_enable"
- "RainMaker over Thread 子设备入网"
- "Thread BR NAT64"
- "ThreadCmd / ActiveDataset"
- "ESP32-H2 RCP + ESP32-S3"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | ESP Thread Border Router Board（ESP32-S3 主控 + ESP32-H2 RCP，UART 互联）；或 M5Stack CoreS3 + Module Gateway H2 |
| 主控固件 | `esp-rainmaker` 的 `examples/thread_br/`（set-target `esp32s3`） |
| RCP 固件 | ESP-IDF 的 `examples/openthread/ot_rcp`（构建时会自动打包进主控固件） |
| Kconfig | `CONFIG_OPENTHREAD_ENABLED=y`、`CONFIG_OPENTHREAD_BORDER_ROUTER=y`、`CONFIG_LWIP_IPV6_NUM_ADDRESSES` 足够大（README 要求 12，示例 defaults 设 8） |
| 节点 | `esp_rmaker_thread_br_enable()` 须在 `esp_rmaker_node_init()` 之后、`esp_rmaker_start()` 之前调用 |
| 参考示例 | `examples/thread_br/` |

## 与"RainMaker over Thread 叶子节点"的区别

| 维度 | 叶子节点（Thread 设备本身上云） | **本 recipe：Border Router** |
|---|---|---|
| 角色 | Thread 网络里的一个 RainMaker 节点 | 给**其它** Thread 设备提供上云通道的 BR |
| 关键 Kconfig | `CONFIG_ESP_RMAKER_NETWORK_OVER_THREAD=y` | `CONFIG_OPENTHREAD_BORDER_ROUTER=y` + NAT64 |
| 关键 API | 无（叶子用标准 RainMaker 流程） | `esp_rmaker_thread_br_enable(&platform_config)` |
| 设备类型 | 业务设备（switch/light…） | `esp.device.thread-br`（`ESP_RMAKER_DEVICE_THREAD_BR`） |
| 网络 | Thread（SRP/MLE） | Wi-Fi 上云 + Thread 下挂 |

> `claiming_and_provisioning.md` 只列了叶子的 `NETWORK_OVER_THREAD` Kconfig；启用 BR **服务本身**用本 recipe。

## 分步说明

### 1. sdkconfig.defaults（BR 核心项）

来自 `examples/thread_br/sdkconfig.defaults`：

```text
# --- OpenThread BR ---
CONFIG_OPENTHREAD_ENABLED=y
CONFIG_OPENTHREAD_BORDER_ROUTER=y
CONFIG_OPENTHREAD_RADIO_SPINEL_UART=y          # RCP 走 UART spinel
CONFIG_OPENTHREAD_LOG_LEVEL_NOTE=y

# --- LwIP（NAT64 / IPv6 转发）---
CONFIG_LWIP_IPV6_FORWARD=y
CONFIG_LWIP_IPV6_NUM_ADDRESSES=8                # IDF 5.3.1+ 可能要求 12，见下方"常见错误"
CONFIG_LWIP_IPV6_AUTOCONFIG=y
CONFIG_LWIP_HOOK_IP6_ROUTE_DEFAULT=y
CONFIG_LWIP_HOOK_ND6_GET_GW_DEFAULT=y
CONFIG_LWIP_HOOK_IP6_INPUT_CUSTOM=y
CONFIG_LWIP_HOOK_IP6_SELECT_SRC_ADDR_CUSTOM=y
CONFIG_LWIP_MULTICAST_PING=y
CONFIG_LWIP_NETIF_STATUS_CALLBACK=y

# --- mbedTLS（Thread 的 EC-JPAKE / DTLS / CMAC）---
CONFIG_MBEDTLS_CMAC_C=y
CONFIG_MBEDTLS_SSL_PROTO_DTLS=y
CONFIG_MBEDTLS_KEY_EXCHANGE_ECJPAKE=y
CONFIG_MBEDTLS_ECJPAKE_C=y

# --- 生成新 dataset 时需要更大的 MQTT task 栈 ---
CONFIG_MQTT_USE_CUSTOM_CONFIG=y
CONFIG_MQTT_TASK_STACK_SIZE=7168

# --- Secure Local Control ---
CONFIG_ESP_RMAKER_LOCAL_CTRL_AUTO_ENABLE=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_SECURITY_1=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_STACK_SIZE=8096

# --- 开发板标识 ---
CONFIG_ESP_THREAD_BR_BOARD_DEV_KIT=y

# USB 串口控制台（ESP Thread Border Router Board）
CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y
```

> IDF v5.3.1+ 编译时若报 `#error CONFIG_LWIP_IPV6_NUM_ADDRESSES should be set to 12`，按 README 把它调到 12 再重新 build。

### 2. 构建 RCP 固件并打包

BR 支持主控在构建时把 RCP 镜像打进主控固件，运行时再烧到 ESP32-H2：

```bash
# 1) 先在 ESP-IDF 里构建 ot_rcp（target esp32h2），menuconfig 里勾 OPENTHREAD_NCP_VENDOR_HOOK
cd $IDF_PATH/examples/openthread/ot_rcp
idf.py set-target esp32h2 build

# 2) 回到 thread_br，主控 target esp32s3
cd <esp-rainmaker>/examples/thread_br
idf.py set-target esp32s3 build
# M5Stack CoreS3 + Module Gateway H2：
# idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.m5stack" set-target esp32s3 build
idf.py -p <PORT> flash monitor
```

### 3. `app_main` 启用 BR 服务（start 之前）

```c
#include <esp_openthread.h>                 /* esp_openthread_platform_config_t + DEFAULT_* 宏 */
#include <esp_rmaker_thread_br.h>
#include <esp_rmaker_core.h>
#include <app_wifi.h>                       /* 该示例用 app_wifi（不是 app_network） */
#include "app_thread_config.h"

#ifdef CONFIG_AUTO_UPDATE_RCP
#include <esp_rcp_update.h>
#include <esp_ot_rcp_update.h>
#include <esp_openthread_spinel.h>
#endif

void app_main(void)
{
    /* NVS（标准流程） */
    /* ... */

    app_wifi_init();                        /* Wi-Fi 上云通道（须在 node_init 之前） */

    esp_rmaker_node_t *node = esp_rmaker_node_init(&cfg, "ESP RainMaker Device", "ThreadBR");

    /* 挂一个 esp.device.thread-br 设备（App 图标） */
    esp_rmaker_device_t *br_dev = esp_rmaker_device_create("ThreadBR", ESP_RMAKER_DEVICE_THREAD_BR, NULL);
    esp_rmaker_device_add_param(br_dev,
        esp_rmaker_name_param_create(ESP_RMAKER_DEF_NAME_PARAM, "ESP-ThreadBR"));
    esp_rmaker_node_add_device(node, br_dev);

    /* OpenThread 平台配置：HOST + PORT + RADIO 三段 */
    esp_openthread_platform_config_t thread_cfg = {
        .host_config  = ESP_OPENTHREAD_DEFAULT_HOST_CONFIG(),
        .port_config  = ESP_OPENTHREAD_DEFAULT_PORT_CONFIG(),
        .radio_config = ESP_OPENTHREAD_DEFAULT_RADIO_CONFIG(),
    };

#ifdef CONFIG_AUTO_UPDATE_RCP
    esp_rcp_update_config_t rcp_update_cfg = ESP_OPENTHREAD_RCP_UPDATE_CONFIG();
    esp_rcp_update_init(&rcp_update_cfg);
    esp_ot_register_rcp_handler();
#endif

    /* 启用 RainMaker BR 服务（须在 node_init 之后、start 之前） */
    esp_rmaker_thread_br_enable(&thread_cfg);

    /* 标准服务 */
    esp_rmaker_ota_enable_default();
    esp_rmaker_timezone_service_enable();
    esp_rmaker_schedule_enable();
    esp_rmaker_scenes_enable();
    app_insights_enable();

    esp_rmaker_start();

#ifdef CONFIG_AUTO_UPDATE_RCP
    esp_ot_update_rcp_if_different();
#endif

    app_wifi_start(POP_TYPE_RANDOM);        /* 走标准 Wi-Fi 配网 */
}
```

> `esp_rmaker_thread_br_enable()` 内部会启动 Thread 协议栈、配置 NAT64、暴露 `TBRService` 设备（含 `ThreadCmd` / `ActiveDataset` 参数）给云端。

### 4. 配网后用 CLI 设 dataset / 起 Thread 网络

手机 App 完成标准 RainMaker 配网后，BR 还**没**形成 Thread 网络。用 `esp-rainmaker-cli` 写 `TBRService`：

**生成随机 dataset 并启动**（最简）：

```bash
esp-rainmaker-cli setparams --data '{"TBRService":{"ThreadCmd": 1}}' <br_node_id>
```

**或指定 Active dataset 并启动**（hex 字符串）：

```bash
esp-rainmaker-cli setparams \
  --data '{"TBRService":{"ActiveDataset": "0E080000000000010000..."}}' \
  <br_node_id>
```

`<br_node_id>` 是 BR 节点的 node id（如 `3485187E7F68`）。Thread 网络起来后，RainMaker-over-Thread 叶子设备即可经 BR 的 NAT64 上云。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `#error CONFIG_LWIP_IPV6_NUM_ADDRESSES should be set to 12` | IDF 5.3.1+ 校验 BR 的 IPv6 地址数 | menuconfig 把 `LWIP_IPV6_NUM_ADDRESSES` 调到 12，重新 build |
| `esp_rmaker_thread_br_enable` 报错 | 在 `esp_rmaker_start()` 之后调用 | 移到 start 之前（头文件 `@note` 明确要求） |
| Thread 网络起不来 | 未设 dataset | 配网后用 CLI 写 `TBRService.ThreadCmd=1` 或 `ActiveDataset` |
| RCP 无响应 | RCP 固件未烧 / UART 接反 / 版本不匹配 | 先单独 build `ot_rcp`（esp32h2），勾 `OPENTHREAD_NCP_VENDOR_HOOK`；检查主控与 H2 的 TX/RX/波特率 |
| BR 启动后子设备无法上云 | NAT64 未生效 / Wi-Fi 未连 | 确认 `LWIP_IPV6_FORWARD=y`、`LWIP_IPV6_AUTOCONFIG=y`；BR 须先成功连 Wi-Fi |
| 生成 dataset 时崩溃 / MQTT 异常 | MQTT task 栈不足 | `CONFIG_MQTT_TASK_STACK_SIZE=7168`（见示例 defaults） |
| EC-JPAKE / DTLS 编译失败 | mbedTLS 选项未全开 | 开 `MBEDTLS_CMAC_C` / `SSL_PROTO_DTLS` / `KEY_EXCHANGE_ECJPAKE` / `ECJPAKE_C` |
| 把 BR 与叶子混淆 | 用了 `NETWORK_OVER_THREAD` 当 BR | BR 用 `OPENTHREAD_BORDER_ROUTER`；`NETWORK_OVER_THREAD` 是叶子上云（见 `claiming_and_provisioning.md`） |

## 参考项目

- `examples/thread_br/` — Thread Border Router 完整示例
  - `examples/thread_br/main/app_main.c` — `esp_openthread_platform_config_t` 三段配置 + `esp_rmaker_thread_br_enable` + 可选 RCP 自动更新
  - `examples/thread_br/main/app_thread_config.h` — BR 板级配置
  - `examples/thread_br/sdkconfig.defaults` — OpenThread / LwIP / mbedTLS / MQTT 栈 / 板级选项
  - `examples/thread_br/README.md` — 硬件、RCP 构建打包、`ThreadCmd` / `ActiveDataset` CLI 用法
- `components/esp_rainmaker/include/esp_rmaker_thread_br.h` — `esp_rmaker_thread_br_enable(esp_openthread_platform_config_t *)`
- `components/esp_rainmaker/include/esp_rmaker_standard_types.h` — `ESP_RMAKER_DEVICE_THREAD_BR`（`"esp.device.thread-br"`）
- ESP-IDF `examples/openthread/ot_rcp` — RCP 固件来源
- `recipes/claiming_and_provisioning.md` — `CONFIG_ESP_RMAKER_NETWORK_OVER_THREAD`（叶子节点上云，对比参照）
