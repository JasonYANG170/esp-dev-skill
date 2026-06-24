# Changelog

本 Skill 的变更记录。版本号遵循语义化版本 (SemVer)。

## [1.1.0] - 2026-06-22

### Added
- 补齐 3 个经审计确认的特性空缺（基于仓库真实文档与代码）：
  - `recipes/dhcpv6_pd.md` — DHCPv6 Prefix Delegation 客户端：Kea DHCPv6 服务器 `pd-pools` 配置、`ot br pd enable/state/omrprefix/onlinkprefix` CLI 序列、Thread 设备地址与 ping 验证。来源 `docs/en/codelab/dhcpv6_pd.rst`（3.8）。
  - `recipes/credential_sharing.md` — Thread 1.4 Credential Sharing：M5Stack 触摸屏生成 ePSKc（`otBorderAgentEphemeralKeyStart` + `otVerhoeffChecksumCalculate`）、`meshcop-e` 通告与 Commissioner DTLS 会话、`POST /commission` 与 `/join_network` 的 `credentialType`（`networkKeyType`/`pskdType`）。来源 `examples/m5stack_thread_border_router/`、`examples/common/thread_border_router_m5stack/src/br_m5stack_epskc_page.c`、`components/esp_ot_br_server/src/esp_br_web_api.c`。
  - `recipes/rf_coexistence.md` — RF 外部共存（Wi-Fi↔802.15.4）：3 线/4 线（`tx_line`）差异、两端 Kconfig（`CONFIG_ESP_COEX_EXTERNAL_COEXIST_ENABLE`）必须同开、`ot_external_coexist_init()` 自动初始化、`esp_external_coex_set_work_mode` / `esp_enable_extern_coex_gpio_pin`。来源 `docs/en/dev-guide/build_and_run.rst`（2.1.3.5）、`examples/common/thread_border_router/src/border_router_launch.c`、ESP-IDF `ot_external_coexist.c` / `esp_coexist.h`。
- `SKILL.md`：
  - 新增 Core Principle 13（RF 共存仅同频段干扰有效 + 两端 Kconfig 同开）、14（Credential Sharing 依赖 Commissioner，`OPENTHREAD_BR_START_WEB` 已自动 select）。
  - Scenario Quick Reference 新增 `dhcpv6_pd`、`credential_sharing`（网络特性组）与 `rf_coexistence`（维护与扩展组）。
  - 版本 `1.0.0` → `1.1.0`。
- `resources/api_reference.md`：新增 §11 RF 外部共存（`ot_external_coexist_init` / `external_coex_wire_t` / `esp_external_coex_gpio_set_t` / `esp_extern_coex_work_mode_t`）、§12 Credential Sharing（`otBorderAgentEphemeralKeyStart/Stop` / `otVerhoeffChecksumCalculate` / `esp_openthread_register_meshcop_e_handler` / `/commission`、`/join_network` REST）、§13 DHCPv6 PD CLI（`ot br pd enable/disable/state/omrprefix/onlinkprefix`）。
- `resources/example_list.md`：补 `br_m5stack_epskc_page.c`、`thread_border_router_m5stack/Kconfig.projbuild`、`border_router_launch.c`、`thread_border_router/Kconfig.projbuild`、`docs/en/codelab/dhcpv6_pd.rst`。

### Grounding
- 所有 API、枚举、结构体、Kconfig、CLI 命令均取自 esp-thread-br 仓库（`docs/en/`、`examples/`、`components/`）及 ESP-IDF（`esp_coexist.h`、`ot_external_coexist.c`、`ot_examples_common.h`、`esp_openthread_netif_glue.h`、`ot_rcp/sdkconfig.ci.ext_coex`）。
- OpenThread 栈 API（`otBorderAgentEphemeralKeyStart/Stop`、`otVerhoeffChecksumCalculate`、`ot br pd *` CLI）指向 https://openthread.io/reference，签名经 `br_m5stack_epskc_page.c` 调用点与 codelab 文档交叉验证。

## [1.0.0] - 2026-06-18

### Added
- 初始发布 esp-thread-br-skill，针对 Espressif ESP Thread Border Router SDK (esp-thread-br)。
- `SKILL.md`：12 条核心原则、When to Use、12 个 Recipe 索引、主控/RCP 组合与引脚表、Kconfig 速查、分区表、13 条关键陷阱（含 WRONG/CORRECT 代码对照）、执行工作流、失败策略。
- `AGENTS.md`：工程约定（命名、include、标准结构、`app_main` 模板、构建工作流、代码生成 checklist、Do Not Modify）。
- `recipes/`（12 个场景）：
  - `build_and_run.md`（拉取/构建/烧录）
  - `board_and_interface.md`（板型、UART/SPI、Standalone 接线）
  - `auto_start_mode.md`（AUTO_START / SoftAP / Ethernet）
  - `bidirectional_ipv6.md`（双向 IPv6 + accept_ra）
  - `multicast_forwarding.md`（ff04 组播 ICMP/UDP）
  - `service_discovery.md`（SRP + mDNS 互发）
  - `nat64.md`（DNS64 + curl）
  - `trel.md`（TREL over Wi-Fi）
  - `web_gui.md`（Web GUI / REST / Home Assistant）
  - `rcp_update.md`（自动/手动更新、序列号、回滚）
  - `http_ota.md`（HTTPS OTA bundle）
- `resources/`：
  - `api_reference.md`（esp_rcp_update / esp_rcp_ota / esp_br_http_ota / esp_br_web / esp_br_wifi_config / esp_ot_cli_extension 等真实签名）
  - `config_reference.md`（Kconfig、引脚、lwIP、mbedTLS、分区、Ethernet 等）
  - `pitfalls.md`（补充陷阱 A~G 类）
  - `example_list.md`（examples/ 与 components/ 真实路径清单）
- `README.md`、`CHANGELOG.md`。

### Grounding
- 所有 API、结构体、宏、Kconfig、引脚、CLI 命令、分区与代码片段均取自 esp-thread-br 仓库的 `docs/en/`、`examples/`、`components/` 及 `sdkconfig.defaults`。
- OpenThread 栈 API 不在本 Skill 内复述，指向 https://openthread.io/reference。
