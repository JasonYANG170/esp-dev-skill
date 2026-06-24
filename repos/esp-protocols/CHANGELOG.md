# Changelog

本文件记录 esp-protocols-skill 的版本变更。版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [1.1.0] - 2026-06-18

填补 3 个已确认的高价值缺口（esp_mqtt_cxx / mosquitto / ap_to_pppos NAPT），全部基于仓库真实文档与 example。

### 新增 Recipes

- **`recipes/mqtt_cxx_client.md`** — `esp_mqtt_cxx` 的 `idf::mqtt::Client` C++ 封装：继承重写 `on_connected`/`on_data`、`BrokerConfiguration`/`ClientCredentials`/`Configuration` 三件套、`subscribe`/`publish`、`QoS`/`Retain`/`Filter`、TLS（PEM/DER）、LastWill/用户名密码、`CONFIG_COMPILER_CXX_EXCEPTIONS=y` 前提。来源 example：`components/esp_mqtt_cxx/examples/tcp/`、`components/esp_mqtt_cxx/examples/ssl/`。
- **`recipes/mosquitto_broker.md`** — ESP32 板载 Mosquitto broker：`mosq_broker_run`/`mosq_broker_stop`、`mosq_broker_config`（host/port/`esp_tls_cfg_server_t` TLS/`handle_connect_cb` basic auth/`handle_message_cb` 消息回调）、本地客户端 loopback 自测、serverless 跨 NAT 同步。来源 example：`components/mosquitto/examples/broker/`、`components/mosquitto/examples/serverless_mqtt/`。
- **`recipes/modem_ap_to_pppos_napt.md`** — WiFi soft-AP ↔ PPPoS NAPT 网关：`CONFIG_LWIP_IP_FORWARD`+`CONFIG_LWIP_IPV4_NAPT`、PPP netif→DCE 顺序、`ip_napt_enable`、DHCP 下发 PPP 的 DNS、断线恢复循环、可选 Minimal DCE（`NetDCE_Factory` + `NetModule : ModuleIf`）。来源 example：`components/esp_modem/examples/ap_to_pppos/`。

### SKILL.md 变更

- metadata.version: `1.0.0` → `1.1.0`。
- 新增 Core Principle 13/14/15：esp_mqtt_cxx 异常前提、mosquitto 阻塞运行与内存占用、NAPT 必备 Kconfig。
- When to Use 的 Applicable 新增 3 项；Not applicable 修正：移除「esp_mqtt_cxx 之外」的错误归类（esp_mqtt_cxx 现已纳入覆盖）。
- Scenario Quick Reference：esp_modem 表新增 `modem_ap_to_pppos_napt.md`；新增「MQTT（C++ 客户端 / 板载 Broker）」分组（含 `mqtt_cxx_client.md`、`mosquitto_broker.md`）。
- Execution Workflow Step 6 新增 esp_mqtt_cxx 与 mosquitto example 改造指引。

### resources/ 变更

- `example_list.md`：新增 `mosquitto/examples/broker/`、`mosquitto/examples/serverless_mqtt/`；扩充 esp_mqtt_cxx 两个 example 的说明；修正尾部「聚焦」清单（esp_mqtt_cxx / mosquitto 从排除改为纳入）。
- `api_reference.md`：新增 `esp_mqtt_cxx`（`Client`/`Filter`/`Message`/`QoS`/`Retain` 及全部配置结构体）、`mosquitto`（`mosq_broker_run/stop`、`mosq_broker_config`、`mosq_connect_cb_t`、`mosq_message_cb_t`）、`esp_modem NAPT 网关相关`（`ip_napt_enable`、example 的 `modem_*` C-API、minimal DCE 的 `NetDCE_Factory`/`NetModule`）三节。

### 数据来源

- `components/esp_mqtt_cxx/include/esp_mqtt.hpp`、`esp_mqtt_client_config.hpp`、`docs/esp_mqtt_cxx/en/index.rst`
- `components/mosquitto/port/include/mosq_broker.h`、`components/mosquitto/api.md`、`components/mosquitto/README.md`
- `components/esp_modem/examples/ap_to_pppos/`（`ap_to_pppos.c`、`network_dce.{c,cpp,h}`、`Kconfig.projbuild`、`sdkconfig.defaults`、`README.md`）

## [1.0.0] - 2026-06-18

首个正式版本。

### 新增

- **SKILL.md**：核心原则（12 条）、When to Use、Recipe 索引、esp_modem 模式状态机与模组枚举、关键 Kconfig 表、13 条 Critical Pitfalls（含 WRONG/CORRECT 代码对照）、Execution Workflow、Failure Strategies、References。
- **AGENTS.md**：组件清单、头文件 include 约定、managed component 工程结构、标准入口/初始化模式、构建步骤、代码生成 checklist、Do Not Modify 说明。
- **recipes/**（8 个场景）：
  - `modem_pppos_uart.md` — UART + PPPoS 拨号上网（读 SIM/信号、切 DATA、拿 IP、pause_net）
  - `modem_custom_module.md` — 继承 `GenericModule` 自定义模组与私有 AT 命令
  - `modem_cmux.md` — CMUX 数据/命令双通道，含手动 CMUX 与设备兼容性
  - `mdns_advertise.md` — mDNS 发布服务、TXT、subtype、delegate host
  - `mdns_query.md` — PTR/SRV/TXT/A/AAAA 查询与结果释放
  - `websocket_client.md` — ws/wss 客户端、事件处理、文本/二进制/分片帧、双向认证
  - `eppp_link.md` — 双 MCU PPP 组网（server listen / client connect）
  - `esp_dns_secure.md` — DoT/DoH/TCP/UDP DNS 解析
- **resources/**：
  - `api_reference.md` — esp_modem/mdns/websocket/eppp_link/esp_dns 真实 API（取自头文件）
  - `config_reference.md` — esp_modem/mdns/eppp_link 真实 Kconfig 与组件版本
  - `pitfalls.md` — 按组件组织的陷阱速查
  - `example_list.md` — 仓库内真实 example 路径索引表
- **README.md**：中文介绍、安装方式、支持范围、目录结构、使用建议。
- **CHANGELOG.md**：本文件。

### 数据来源

- 仓库：`D:/esp-skill/espressif-repos/esp-protocols`
- 文档：`docs/esp_modem/en/`（README.rst、api_docs.rst、index.rst）、`docs/mdns/en/`、`docs/esp_websocket_client/en/`
- 头文件：`components/{esp_modem,mdns,esp_websocket_client,eppp_link,esp_dns}/include/*.h`
- Kconfig：`components/{esp_modem,mdns,eppp_link}/Kconfig`
- Example：`components/*/examples/` 与顶层 `examples/`
