# Changelog

本 Skill 版本变更记录。格式参考 [Keep a Changelog](https://keepachangelog.com/)。

## [1.1.0] - 2026-06-18

### Added
- `recipes/tls_digital_signature.md`：数字签名（DS）外设 TLS 认证场景。覆盖 ESP32-S2/S3/C3/C5/C6/H2/P4 硬件 DS 工作流，与 `tls_mutual_auth.md`（PEM 嵌入）互补。来源示例 `examples/ssl_ds/`（README、`main/app_main.c`、`CMakeLists.txt`、`main/idf_component.yml`），补全了仓库 9 个示例中唯一缺独立 recipe 的一例。
  - `esp_secure_cert` 分区 provisioning（`esp-secure-cert-tool` / `configure_esp_secure_cert.py --configure_ds`）。
  - 顶层 `CMakeLists.txt` 注册 `esp_secure_cert` 分区烧录（`esptool_py_flash_target_image`）。
  - `esp_secure_cert_get_ds_ctx()` / `esp_secure_cert_get_device_cert()` 读取设备证书与 DS 上下文，`.key=NULL` + `.ds_data` 配置。
  - `esp_ds_data_ctx_t` 结构体字段（`esp_ds_data` / `efuse_key_id` / `rsa_length_bits`）。
  - MbedTLS v4 与旧版的条件编译头（`psa_crypto_driver_esp_rsa_ds.h` vs `rsa_sign_alt.h`）。

### Changed
- `SKILL.md`：传输与安全 recipe 表新增 `tls_digital_signature.md` 行；`metadata.version` 1.0.0 → 1.1.0。
- `resources/example_list.md`：`examples/ssl_ds/` 行补充 DS 工作流要点（依赖、分区烧录、provisioning、引用文件）。
- `resources/api_reference.md`：新增「数字签名（DS）外设辅助 API」节，含 `esp_secure_cert_read.h` 签名与 `esp_ds_data_ctx_t` 定义及典型配置示例。

### Grounding
- `examples/ssl_ds/main/app_main.c`（`esp_secure_cert_get_ds_ctx` / `esp_secure_cert_get_device_cert` 调用、`.ds_data` 配置、条件编译）。
- `examples/ssl_ds/README.md`（目标芯片、openssl CSR、`esp-secure-cert-tool` provisioning）。
- `examples/ssl_ds/CMakeLists.txt`（`esp_secure_cert` 分区烧录 CMake 片段）。
- `examples/ssl_ds/main/idf_component.yml`（`espressif/esp_secure_cert_mgr: "^2.0.2"`）。
- `include/mqtt_client.h`（`authentication.ds_data` 字段，`void *`，不 copy/free）。
- `docs/en/index.rst`（Authentication 节 `ds_data` 说明）。
- esp_secure_cert_mgr `include/esp_secure_cert_read.h`（函数签名与文档注释）。
- IDF `components/mbedtls/port/psa_driver/include/psa_crypto_driver_esp_rsa_ds_contexts.h`（`esp_ds_data_ctx_t` 定义）。

## [1.0.0] - 2026-06-18

### Added
- 初始发布，面向 Espressif ESP-MQTT 客户端组件的 AI Skill。
- `SKILL.md`：核心原则（12 条）、When to Use、11 个 recipe 索引、Kconfig 速查表、scheme/默认端口表、客户端状态机、13 条关键陷阱（含 WRONG/CORRECT 代码对照）、执行工作流与失败策略。
- `AGENTS.md`：项目上下文、include 约定、证书 embed 方式、标准 app_main / mqtt_app_start 模式、构建工作流、MQTT 代码生成 checklist。
- `recipes/` 11 个场景：TCP 连接、事件处理、发布订阅/QoS、mTLS 双向认证、服务端 CA 单向校验、PSK 认证、WebSocket/WSS、MQTT 5.0、遗嘱消息、outbox 管理、自定义 outbox。
- `resources/api_reference.md`：`esp_mqtt_client_*` / `esp_mqtt5_*` 全部函数签名、枚举、`esp_mqtt_client_config_t` 全字段、MQTT5 结构体、feature 支持宏。
- `resources/config_reference.md`：全部 Kconfig 项与运行时配置字段。
- `resources/pitfalls.md`：30 条高频陷阱汇总。
- `resources/example_list.md`：仓库 9 个真实示例索引及依赖。
- `README.md`：中文介绍、安装方式、支持范围、目录结构。

### Grounding
- 所有 API、结构体、宏、Kconfig、代码片段均取自 esp-mqtt 仓库 `include/mqtt_client.h`、`include/mqtt5_client.h`、`include/mqtt_supported_features.h`、`Kconfig`、`docs/en/index.rst` 及 `examples/*/main/app_main.c`。
