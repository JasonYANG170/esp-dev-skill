# Changelog

所有重要变更记录于此文件。格式基于 [Keep a Changelog](https://keepachangelog.com/)，版本遵循 [语义化版本](https://semver.org/)。

## [1.1.0] - 2026-06-18

### Added

- 新增 4 篇场景配方（recipes/），填补审计确认的 4 个缺口；所有 API/宏/示例路径均取自真实仓库：
  - `recipes/tls13_early_data.md` — TLS 1.3 早期数据（0-RTT）：客户端 `mbedtls_ssl_write_early_data` + `mbedtls_ssl_get_early_data_status`（拒绝时作为普通 `ssl_write` 重发）、服务端 `MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA` → `mbedtls_ssl_read_early_data`。源文档 `docs/tls13-early-data.md`；示例 `programs/ssl/ssl_client2.c`、`ssl_server2.c`。
  - `recipes/tls13_configuration.md` — TLS 1.3 构建配置与密钥交换模式选择：四种 `_KEY_EXCHANGE_MODE_*_ENABLED` 宏、`_COMPATIBILITY_MODE`、硬性依赖（`MBEDTLS_PSA_CRYPTO_C` / `MBEDTLS_SSL_KEEP_PEER_CERTIFICATE`）、运行时 `mbedtls_ssl_conf_tls13_key_exchange_modes` 收窄。源文档 `docs/architecture/tls13-support.md`；示例 `ssl_client2.c` / `ssl_server2.c`（`tls13_kex_modes=`）。
  - `recipes/smtp_starttls_client.md` — SMTP over TLS / STARTTLS 就地升级：明文 SMTP（banner/EHLO/STARTTLS）→ 同一 socket 升级 TLS → TLS 内继续 SMTP（AUTH LOGIN base64、MAIL FROM/RCPT TO/DATA）。示例 `programs/ssl/ssl_mail_client.c`。
  - `recipes/tls_fork_server.md` — 多进程 `fork()` 并发 TLS 服务器（每客户端一进程、`SIGCHLD` 回收、父关 client_fd / 子关 listen_fd、子进程内 `ssl_setup`）。示例 `programs/ssl/ssl_fork_server.c`（仅 POSIX）。

### Changed

- `SKILL.md`：版本 `1.0.0` → `1.1.0`；新增核心原则 13（早期数据安全属性与处理模式）；Scenario Quick Reference 新增 "TLS 1.3" 分组（2 篇）并在 "TLS（基于 TCP）" 增列 `tls_fork_server.md`、`smtp_starttls_client.md`；关键宏速查表新增 `MBEDTLS_SSL_EARLY_DATA`、`_TLS1_3_KEY_EXCHANGE_MODE_*`、`_TLS1_3_COMPATIBILITY_MODE`；Step 6 示例起点表新增 fork 服务器、SMTP、TLS 1.3 早期数据/kex 三行；Failure Strategies 新增 STARTTLS、`CANNOT_WRITE_EARLY_DATA`、TLS 1.3 构建异常三行。
- `resources/api_reference.md`：新增 TLS 1.3 早期数据与密钥交换模式签名（`mbedtls_ssl_conf_tls13_key_exchange_modes`、`_conf_early_data`、`_conf_max_early_data_size`、`_write_early_data`、`_read_early_data`、`_get_early_data_status`、`_is_handshake_over` 及相关错误码/状态常量）。
- `resources/config_reference.md`：新增 "TLS 1.3 专属" 配置宏小节（`_TLS1_3_*` 模式宏、`_EARLY_DATA`、`_MAX_EARLY_DATA_SIZE`、`_COMPATIBILITY_MODE` 及硬性依赖说明）。

## [1.0.0] - 2026-06-18

### Added

- 初始发布：面向 Mbed TLS（Espressif fork，4.x）的 AI Skill。
- `SKILL.md`：12 条核心原则、配方索引表、4.x 配置速查、3.x→4.x 改名对照、12 条带正/误代码对照的关键陷阱、执行工作流、失败策略。
- `AGENTS.md`：头文件包含规范、TLS 客户端骨架、调试/错误模板、init/free 配对表、构建流程、代码生成清单。
- 配方（recipes/）12 篇：TLS 客户端、TLS 服务器、DTLS 客户端、DTLS 服务器（含 cookie）、证书解析校验、CSR 生成、证书签发、PSK TLS、会话恢复与票据、加固与错误诊断、构建与配置、3.x→4.x PSA 迁移。每篇均含真实调用链、可复制代码、常见错误表与参考示例路径。
- 资源文档（resources/）4 篇：
  - `api_reference.md` — 按模块分组的真实函数签名（SSL/缓存/票据/cookie/网络/计时/X.509/CSR/PK/平台/错误）。
  - `config_reference.md` — `mbedtls_config.h` 与 PSA `crypto_config.h` 的真实宏、CMake 选项、`configs/` 预设。
  - `pitfalls.md` — 16 条 4.x 常见陷阱（PSA 初始化、RNG 删除、DTLS 定时器、session_reset、cookie、返回值语义、改名 API、链接顺序等）。
  - `example_list.md` — `programs/` 真实示例路径与描述索引。
- 所有 API、宏、配置项、代码示例均取自真实仓库头文件、`programs/` 示例与 `docs/`，无臆造内容。
