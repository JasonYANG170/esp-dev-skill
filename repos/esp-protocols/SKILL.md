---
name: esp-protocols-skill
description: >-
  AI Skill for ESP-IDF networking protocol components (esp-protocols). Used when developing
  cellular modem (esp_modem) command/PPP/CMUX applications, mDNS service discovery, WebSocket
  client, PPP link (eppp_link), DNS over TLS/HTTPS (esp_dns), and other protocol components.
  Trigger words: "esp_modem", "esp-protocols", "mDNS", "WebSocket", "PPPoS", "CMUX", "蜂窝模组",
  "SIM7600", "SIM800", "BG96", "esp_dns", "DoT", "DoH", "eppp_link", "协议组件"
license: Apache-2.0
metadata:
  author: Community
  version: "1.1.0"
---

# esp-protocols-skill

本 Skill 面向 Espressif [esp-protocols](https://github.com/espressif/esp-protocols) 仓库 —— 一组用于 ESP-IDF 的网络协议组件集合（esp_modem 蜂窝模组、mdns、esp_websocket_client、eppp_link、esp_dns 等）。提供场景化 Recipe、真实 API 参考、配置选项、以及必须遵守的陷阱清单。所有 API、结构体、宏、Kconfig 符号、文件路径与代码片段均来自仓库真实文档与源码，未在文档/源码中出现的 API 一律不予编造。

## Core Principles

1. **绝不臆造 API** —— 任何函数名 / 结构体 / 宏 / Kconfig 必须能在 `resources/` 或仓库 `components/*/include` 中找到；找不到即视为不存在。
2. **DCE/DTE/PPP 三件套是 esp_modem 的核心模型** —— `DCE`（模组抽象）由 `DTE`（终端，当前为 UART/USB）、`PPP netif`（网络接口）、`Module`（具体型号命令库）组成；必须先建 `esp_netif_t` 再建 `DCE`。
3. **模式切换有约束** —— `COMMAND ↔ DATA`，以及 `COMMAND ↔ CMUX`；任何模式都可转到 `UNDEF` 再回来。手动 CMUX 模式（`CMUX_MANUAL_*`）仅给需要精细控制通道切换的高级场景使用。
4. **PPPoS 拿 IP 走标准 IP 事件** —— 切到 `ESP_MODEM_MODE_DATA` 后，通过 `IP_EVENT_PPP_GOT_IP` / `IP_EVENT_PPP_LOST_IP` 与 `NETIF_PPP_STATUS` 事件感知连接状态，而非自己轮询模组。
5. **C-API 字符串输出缓冲区至少 `ESP_MODEM_C_API_STR_BUF_SIZE`（默认 128）字节** —— `esp_modem_get_imsi/get_imei/get_operator_name/get_module_name/at/at_raw` 都按此截断，缓冲区过小会被截断。
6. **mDNS 必须先 `mdns_init()`，设置 hostname 后再 `mdns_service_add()`** —— 查询结果链表用完必须 `mdns_query_results_free()` 释放，否则内存泄漏。
7. **WebSocket 事件处理函数中禁止调用 `esp_websocket_client_stop/destroy/close`** —— 这些 API 会自死锁；应在事件回调外（如单独任务/定时器）调用。
8. **组件按需添加** —— esp-protocols 是一组独立 managed component，每个组件有自己的 `idf_component.yml`；在 `idf_component.yml` 中只声明实际用到的组件（如 `espressif/esp_modem`）。
9. **esp_modem 默认 Production Mode** —— 使用预生成的 AT 命令头文件；仅当要修改 `esp_modem_command_declare.inc` 时才设 `CONFIG_ESP_MODEM_ENABLE_DEVELOPMENT_MODE=y`。
10. **UART OTA 不稳定时优先缓解 UART 而非协议** —— 将 UART ISR 放入 IRAM、增大 RX buffer、提高 terminal task 优先级、启用硬件流控（参考 docs/esp_modem/en/README.rst Known issues）。
11. **eppp_link 是双 MCU 串连组网的 PPP 引擎** —— 一端 `eppp_listen()`（server），一端 `eppp_connect()`（client），传输层可选 UART/SPI/SDIO/Ethernet。
12. **CMUX 兼容性因设备而异** —— SIM7000 不支持 CMUX；A76xx 系列用 2 字节 CMUX payload 需关闭 `CONFIG_ESP_MODEM_CMUX_DEFRAGMENT_PAYLOAD`；A7670 退出 CMUX 异常需打补丁。
13. **esp_mqtt_cxx 必须启用 C++ 异常** —— 组件内部用异常做错误处理，`esp_mqtt.hpp` 在 `__cpp_exceptions` 未定义时直接 `#error`；必须 `CONFIG_COMPILER_CXX_EXCEPTIONS=y`，并在 `Client` 派生类构造完成后才 `start()`、在 `on_connected()` 内订阅。
14. **mosquitto 板载 broker 在调用线程阻塞运行** —— `mosq_broker_run()` 不建独立任务，调用任务栈需 ≥ 5 kB；`mosq_broker_stop()` 才能解除阻塞。每客户端约 4 kB 堆，异常断连重连期间占双倍。
15. **NAPT 网关必须开 IP_FORWARD + IPV4_NAPT** —— WiFi AP 转 PPPoS 路由场景需 `CONFIG_LWIP_IP_FORWARD=y` + `CONFIG_LWIP_IPV4_NAPT=y`，再对 AP 接口 IP 调 `ip_napt_enable(addr, 1)`，否则 AP 客户端无法上网。

## When to Use

**Applicable:**
- 使用 esp_modem 与蜂窝模组（SIM800/SIM7600/SIM7000/SIM7070/BG96/EC20/Generic/Custom）通信、拨号上网（PPPoS）
- 使用 CMUX 在数据模式下同时收发 AT 命令
- 添加自定义模组（继承 `GenericModule`，`CONFIG_ESP_MODEM_ADD_CUSTOM_MODULE`）
- 使用 mDNS 发布/发现局域网服务（`_http._tcp` 等）
- 使用 `esp_websocket_client` 建立 ws/wss 连接、收发文本/二进制/分片帧
- 使用 eppp_link 在双 MCU 间用 UART/SPI/SDIO/ETH 建立 PPP 通道
- 使用 esp_dns 实现 DoT/DoH/TCP DNS
- 使用 esp_mqtt_cxx 的 `idf::mqtt::Client` 用 C++ 写 MQTT 客户端（subscribe/publish/QoS/TLS/LastWill/Filter）
- 使用 mosquitto 组件在 ESP32 上跑板载 MQTT broker（basic auth/TLS/消息回调）
- 把 ESP32 当蜂窝路由器：WiFi soft-AP 经 NAPT 转发到 PPPoS（含 minimal DCE 工厂）

**Not applicable:**
- 非 ESP-IDF 平台（本 Skill 假定 ESP-IDF v5.x + CMake 构建）
- 纯 Wi-Fi/BLE 协议栈本身（用 esp-idf 自身组件，而非本仓库）
- PCB / 硬件原理图设计
- 通用 C++ 异步框架问题（asio 仅为移植层，超出本 Skill 范围）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配以下场景时，**先阅读对应 Recipe** —— 内含完整调用链、分步说明、常见错误与真实代码。

### esp_modem 蜂窝模组

| recipe | scenario |
|---|---|
| `recipes/modem_pppos_uart.md` | UART 连接蜂窝模组，拨号 PPPoS 上网（读取信号/SIM、切 DATA 模式、拿 IP） |
| `recipes/modem_custom_module.md` | 自定义模组（继承 `GenericModule` + `custom_module.hpp`），添加私有 AT 命令 |
| `recipes/modem_cmux.md` | CMUX 模式：在 PPP 数据通道的同时收发 AT 命令 |
| `recipes/modem_ap_to_pppos_napt.md` | WiFi soft-AP ↔ PPPoS NAPT 网关：蜂窝流量共享给 AP 客户端（含 minimal DCE 工厂） |

### mDNS 服务发现

| recipe | scenario |
|---|---|
| `recipes/mdns_advertise.md` | mDNS 初始化、设置 hostname、`mdns_service_add` 发布服务与 TXT 记录 |
| `recipes/mdns_query.md` | mDNS 查询服务（PTR/SRV/TXT/A/AAAA）并释放结果 |

### WebSocket 客户端

| recipe | scenario |
|---|---|
| `recipes/websocket_client.md` | `esp_websocket_client` 初始化、事件处理、收发文本/二进制/分片帧、TLS |

### MQTT（C++ 客户端 / 板载 Broker）

| recipe | scenario |
|---|---|
| `recipes/mqtt_cxx_client.md` | `esp_mqtt_cxx` 的 `idf::mqtt::Client` C++ 封装：继承重写事件回调、`BrokerConfiguration`/`ClientCredentials`/`Configuration`、subscribe/publish/QoS/Retain/Filter、TLS（PEM/DER） |
| `recipes/mosquitto_broker.md` | ESP32 板载 Mosquitto broker：`mosq_broker_run`、`mosq_broker_config`（host/port/TLS/basic auth/消息回调）、本地客户端 loopback 自测、serverless 跨 NAT 同步 |

### 其他协议组件

| recipe | scenario |
|---|---|
| `recipes/eppp_link.md` | eppp_link 双 MCU PPP 组网（server `eppp_listen` / client `eppp_connect`） |
| `recipes/esp_dns_secure.md` | esp_dns 使用 DoT / DoH / TCP DNS 解析 |

---

## esp_modem 模式状态机

```
      +---------+    +---------+
      | COMMAND |<-->|   DATA  |   (esp_modem_set_mode: ESP_MODEM_MODE_COMMAND / ESP_MODEM_MODE_DATA)
      +---------+    +---------+
           ^
           |
           v
      +-------+
      | CMUX  |                 (ESP_MODEM_MODE_CMUX)
      +-------+
```

> 从任何模式都可切到 `ESP_MODEM_MODE_UNDEF`，再切回目标模式。`ESP_MODEM_MODE_DETECT` 会自动探测当前模式并尝试恢复。

### esp_modem 支持的模组枚举（`esp_modem_dce_device_t`）

| 枚举值 | 说明 |
|---|---|
| `ESP_MODEM_DCE_GENERIC` | 通用设备（仅用标准 AT） |
| `ESP_MODEM_DCE_SIM7600` | SIMCom SIM7600 系列 |
| `ESP_MODEM_DCE_SIM7070` | SIMCom SIM7070（Cat-M/NB-IoT） |
| `ESP_MODEM_DCE_SIM7000` | SIMCom SIM7000（**不支持 CMUX**） |
| `ESP_MODEM_DCE_BG96` | Quectel BG96 |
| `ESP_MODEM_DCE_EC20` | Quectel EC20 |
| `ESP_MODEM_DCE_SIM800` | SIMCom SIM800（2G） |
| `ESP_MODEM_DCE_SQNGM02S` | Sequans GM02S |
| `ESP_MODEM_DCE_CUSTOM` | 用户自定义（需 `CONFIG_ESP_MODEM_ADD_CUSTOM_MODULE`） |

### 关键 Kconfig（`CONFIG_*`）

| Kconfig | 默认 | 说明 |
|---|---|---|
| `CONFIG_ESP_MODEM_USE_PPP_MODE` | y | 启用 lwip PPP netif（自动 select `LWIP_PPP_SUPPORT`） |
| `CONFIG_ESP_MODEM_CMUX_DEFRAGMENT_PAYLOAD` | y | CMUX 内部重组分片；2 字节 payload 设备（A76xx）需关 |
| `CONFIG_ESP_MODEM_C_API_STR_MAX` | 128 | C-API 文本输出缓冲最小字节数 |
| `CONFIG_ESP_MODEM_ADD_CUSTOM_MODULE` | n | 启用自定义模组 C-API |
| `CONFIG_ESP_MODEM_PPP_ESCAPE_BEFORE_EXIT` | n | PPP→CMD 时发 `+++`（Quectel 友好，SIMCOM 可能出错） |
| `CONFIG_ESP_MODEM_ENABLE_DEVELOPMENT_MODE` | n | 开发模式（直接展开 AT 命令宏） |
| `CONFIG_ESP_MODEM_URC_HANDLER` | n | 启用 `esp_modem_set_urc()` URC 处理 |
| `CONFIG_MDNS_MAX_SERVICES` | 10 | mDNS 最大服务数 |
| `CONFIG_MDNS_MAX_INTERFACES` | 3 | mDNS 最大接口数 |
| `CONFIG_MDNS_TASK_PRIORITY` | 1 | mDNS 任务优先级（勿高于系统任务） |

---

## Critical Pitfalls (Must Read)

以下是最常见的错误，违反任何一条都会导致固件不工作。

### 1. 必须先建 PPP netif，再用它创建 DCE

```c
// ❌ WRONG — netif 还没建就传给 esp_modem_new
esp_modem_dce_t *dce = esp_modem_new(&dte_config, &dce_config, NULL);

// ✅ CORRECT — 先建 PPP netif，再建 DCE
esp_netif_config_t netif_ppp_config = ESP_NETIF_DEFAULT_PPP();
esp_netif_t *esp_netif = esp_netif_new(&netif_ppp_config);
assert(esp_netif);
esp_modem_dce_t *dce = esp_modem_new(&dte_config, &dce_config, esp_netif);
```

### 2. C-API 字符串输出缓冲至少 ESP_MODEM_C_API_STR_BUF_SIZE

```c
// ❌ WRONG — 缓冲过小，IMSI/IMEI 被截断
char imsi[16];
esp_modem_get_imsi(dce, imsi);

// ✅ CORRECT — 使用官方推荐的缓冲尺寸
char imsi[ESP_MODEM_C_API_STR_BUF_SIZE];   // >= CONFIG_ESP_MODEM_C_API_STR_MAX (默认 128)
esp_modem_get_imsi(dce, imsi);
```

### 3. 切 DATA 模式后通过 IP 事件感知，而非轮询模组

```c
// ❌ WRONG — 切到 DATA 模式后还在发 AT 命令（此时 UART 是 PPP 流）
esp_modem_set_mode(dce, ESP_MODEM_MODE_DATA);
esp_modem_get_signal_quality(dce, &rssi, &ber);   // 会失败

// ✅ CORRECT — 注册 IP 事件，等待 GOT_IP
esp_event_handler_register(IP_EVENT, ESP_EVENT_ANY_ID, &on_ip_event, NULL);
esp_event_handler_register(NETIF_PPP_STATUS, ESP_EVENT_ANY_ID, &on_ppp_changed, NULL);
esp_modem_set_mode(dce, ESP_MODEM_MODE_DATA);
// 在 on_ip_event() 中处理 IP_EVENT_PPP_GOT_IP
```

### 4. 硬件流控要在创建 DCE 后调用 esp_modem_set_flow_control

```c
// ❌ WRONG — 直接改 uart_config 但没通知模组侧
dte_config.uart_config.flow_control = ESP_MODEM_FLOW_CONTROL_HW;

// ✅ CORRECT — 配置流控后再调用 set_flow_control(2,2)
dte_config.uart_config.flow_control = ESP_MODEM_FLOW_CONTROL_HW;
esp_modem_dce_t *dce = esp_modem_new_dev(ESP_MODEM_DCE_BG96, &dte_config, &dce_config, esp_netif);
if (dte_config.uart_config.flow_control == ESP_MODEM_FLOW_CONTROL_HW) {
    esp_modem_set_flow_control(dce, 2, 2);   // 2/2 = HW flow control
}
```

### 5. 暂停网络发 AT 命令必须用 esp_modem_pause_net，不要切模式

```c
// ❌ WRONG — 为发一条 AT 把整个 PPP 断掉
esp_modem_set_mode(dce, ESP_MODEM_MODE_COMMAND);
esp_modem_get_signal_quality(dce, &rssi, &ber);
esp_modem_set_mode(dce, ESP_MODEM_MODE_DATA);

// ✅ CORRECT — pause_net 临时挂起 netif，发完 AT 再恢复
esp_modem_pause_net(dce, true);
esp_modem_get_signal_quality(dce, &rssi, &ber);
esp_modem_pause_net(dce, false);
```

### 6. mDNS 查询结果必须释放

```c
// ❌ WRONG — results 链表泄漏
mdns_result_t *results = NULL;
mdns_query_ptr("_http", "_tcp", 3000, 20, &results);
// ... 用完直接 return

// ✅ CORRECT — 用完释放
mdns_result_t *results = NULL;
if (mdns_query_ptr("_http", "_tcp", 3000, 20, &results) == ESP_OK) {
    // ... 处理 results
    mdns_query_results_free(results);
}
```

### 7. mDNS 必须先 init + hostname_set 再 service_add

```c
// ❌ WRONG — 未初始化就加服务
mdns_service_add("srv", "_http", "_tcp", 80, NULL, 0);

// ✅ CORRECT
mdns_init();
mdns_hostname_set("esp32-xxxx");
mdns_instance_name_set("ESP32-WebServer");
mdns_service_add("ESP32-WebServer", "_http", "_tcp", 80, serviceTxtData, 3);
```

### 8. WebSocket 事件回调内禁止 stop/destroy/close

```c
// ❌ WRONG — 在事件处理函数里销毁，自死锁
static void ws_event_handler(...) {
    if (event_id == WEBSOCKET_EVENT_DATA) {
        esp_websocket_client_destroy(client);   // 死锁
    }
}

// ✅ CORRECT — 用信号量通知外部任务，由外部任务销毁
static void ws_event_handler(...) {
    if (event_id == WEBSOCKET_EVENT_DATA) {
        xSemaphoreGive(shutdown_sema);   // 外部任务据此 destroy
    }
}
```

### 9. WebSocket TLS 用证书包要传 crt_bundle_attach

```c
// ❌ WRONG — 想用证书包但没设置
websocket_cfg.uri = "wss://echo.websocket.org";
// websocket_cfg.crt_bundle_attach 缺失 → 无法验证服务器

// ✅ CORRECT
#include "esp_crt_bundle.h"
websocket_cfg.uri = "wss://echo.websocket.org";
websocket_cfg.crt_bundle_attach = esp_crt_bundle_attach;
```

### 10. eppp_link 两端 IP 必须交叉对应

```c
// ❌ WRONG — 两端都把自己当 server IP，PPP 协商失败
// server: EPPP_DEFAULT_CONFIG(EPPP_DEFAULT_SERVER_IP(), EPPP_DEFAULT_CLIENT_IP())
// client: EPPP_DEFAULT_CONFIG(EPPP_DEFAULT_SERVER_IP(), EPPP_DEFAULT_CLIENT_IP())

// ✅ CORRECT — 用官方宏，server/client 的 our/their 已正确交叉
// server:
eppp_config_t cfg = EPPP_DEFAULT_SERVER_CONFIG();   // our=192.168.11.1, their=192.168.11.2
esp_netif_t *netif = eppp_listen(&cfg);
// client:
eppp_config_t cfg = EPPP_DEFAULT_CLIENT_CONFIG();   // our=192.168.11.2, their=192.168.11.1
esp_netif_t *netif = eppp_connect(&cfg);
```

### 11. CMUX 与 SIM7000/A76xx 的兼容性

```c
// ❌ WRONG — 对 SIM7000 切 CMUX（设备根本不支持）
esp_modem_set_mode(dce, ESP_MODEM_MODE_CMUX);   // 永远不成功

// �� CORRECT — SIM7000 用 COMMAND/DATA；A76xx 关掉 payload 重组
// menuconfig: CONFIG_ESP_MODEM_CMUX_DEFRAGMENT_PAYLOAD=n   (A76xx 系列)
// 普通 1 字节 payload 设备保持默认 y
```

### 12. esp_modem_new vs esp_modem_new_dev 的选择

```c
// ❌ WRONG — 用 esp_modem_new() 却期望 SIM7600 专有命令
esp_modem_dce_t *dce = esp_modem_new(&dte_config, &dce_config, esp_netif);

// ✅ CORRECT — 指定具体设备枚举
esp_modem_dce_t *dce = esp_modem_new_dev(ESP_MODEM_DCE_SIM7600, &dte_config, &dce_config, esp_netif);
```

### 13. UART OTA 不稳定优先缓解 UART，而非改协议

```c
// ❌ WRONG — OTA 频繁 buffer overflow 就去改 PPP/HTTP 逻辑

// ✅ CORRECT — 按 docs/esp_modem/en/README.rst Known issues 处理：
//   1) UART ISR 放入 IRAM
//   2) 增大 uart_config.rx_buffer_size
//   3) 提高 dte_config.task_priority
//   4) 启用硬件流控（esp_modem_set_flow_control(dce, 2, 2)）
//   5) 仍不行：参考 test_app/esp_modem/test/target_ota 两步式 OTA
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确需求：用哪个组件（esp_modem/mdns/websocket/eppp_link/esp_dns）、目标芯片、传输层（UART/USB/SPI） |
| 2 | Recipe | 在 `recipes/` 中找匹配场景，按其调用链与分步说明实施 |
| 3 | Query | Recipe 未覆盖的 API，查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | Validate | 校验：函数签名、头文件 include、配置结构体字段、Kconfig 符号均来自仓库 |
| 5 | Confirm | 向用户确认实现计划：组件依赖、netif/DCE 创建顺序、事件注册、模式切换 |
| 6 | Execute | 新项目：参照 `resources/example_list.md` 中最接近的真实 example 改造；已有项目：原地编辑 |
| 7 | Check | 复查：netif→DCE 顺序、缓冲尺寸、事件处理内无阻塞 API、查询结果释放、CMUX 设备兼容性 |
| 8 | Build | `idf.py build`（ESP-IDF v5.x + CMake） |
| 9 | Debug | `idf.py monitor` 观察日志；蜂窝模组可结合 `examples/modem_console` 交互调试 |

### Step 6 Detail — Example 改造策略

**新项目首选改造仓库自带 example**（路径见 `resources/example_list.md`）：

- 蜂窝模组 PPPoS（C）→ `components/esp_modem/examples/pppos_client/`
- 蜂窝模组交互控制台（C++）→ `components/esp_modem/examples/modem_console/`
- CMUX 客户端 → `components/esp_modem/examples/simple_cmux_client/`
- AP 转 PPPoS（NAT 网关）→ `components/esp_modem/examples/ap_to_pppos/`
- mDNS 发布/查询 → `components/mdns/examples/query_advertise/`
- WebSocket 客户端 → `components/esp_websocket_client/examples/target/`
- eppp_link 双 MCU → `components/eppp_link/examples/`（server/client）
- MQTT C++ 客户端（明文/TLS）→ `components/esp_mqtt_cxx/examples/{tcp,ssl}/`
- 板载 Mosquitto broker → `components/mosquitto/examples/broker/`

复制整个 example 目录到目标工程，再按需求修改 `Kconfig.projbuild`、`sdkconfig.defaults`、main 源文件。**不要**重写整个工程。

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/` 中找不到 | 立即停止，告知用户该 API 可能不存在于本组件 |
| `esp_modem_new` 返回 NULL | 检查 netif 是否已创建、内存是否足够、`CONFIG_ESP_MODEM_USE_PPP_MODE` 是否启用 |
| PPP 拿不到 IP | 确认 SIM 已注册网络、APN 正确、信号质量足够；检查 `IP_EVENT_PPP_GOT_IP` 是否注册 |
| CMUX 进入失败 | 确认设备支持 CMUX（SIM7000 不支持）；A76xx 关闭 `ESP_MODEM_CMUX_DEFRAGMENT_PAYLOAD` |
| UART OTA buffer overflow | ISR 放 IRAM、增大 RX buffer、提高任务优先级、启用 HW 流控 |
| mDNS 查询无结果 | 确认两端在同一多播域、`mdns_init()` 成功、hostname 已设置 |
| WebSocket 连接失败 | 检查 URI scheme（ws/wss）、TLS 证书/crt_bundle、事件日志中的 handshake status code |
| eppp_link 协商失败 | 两端 IP 必须交叉（用 `EPPP_DEFAULT_SERVER_CONFIG`/`EPPP_DEFAULT_CLIENT_CONFIG`）；传输层配置一致 |

## References

- 场景 Recipe → `recipes/` 目录
- API 速查（按组件分组） → `resources/api_reference.md`
- 配置选项（Kconfig） → `resources/config_reference.md`
- 常见陷阱汇总 → `resources/pitfalls.md`
- 真实 example 索引 → `resources/example_list.md`
