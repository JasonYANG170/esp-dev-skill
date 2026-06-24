---
name: mbedtls-skill
description: >-
  AI Skill for Mbed TLS (Espressif fork, 4.x) SSL/TLS/DTLS library development. Use when
  users need to create TLS/DTLS clients or servers, parse/write X.509 certificates and CSRs,
  configure ciphersuites/protocols, use PSK or session resumption, or migrate code from
  Mbed TLS 3.x to 4.x (PSA Crypto).
  Trigger words: "mbedTLS", "Mbed TLS", "mbedtls", "TLS", "DTLS", "SSL", "X.509", "证书",
  "PSK", "session ticket", "PSA Crypto", "psa_crypto_init", "加密套件", "ciphersuite",
  "HTTPS", "CSR", "CRL"
tags:
  - embedded
  - TLS
  - SSL
  - DTLS
  - cryptography
  - X.509
  - PSA
  - mbedtls
  - network-security
  - Espressif
license: Apache-2.0 OR GPL-2.0-or-later
compatibility: >-
  C library, C99 toolchain (GCC/Clang/MSVC/Armclang), CMake ≥ 3.20.2 build only (4.x removed
  Make/VS). Cross-platform (POSIX hosts, embedded via platform.h port). Cryptography provided
  by the bundled TF-PSA-Crypto submodule (PSA Crypto API).
metadata:
  author: Community
  version: "1.1.0"
---

# mbedtls-skill

AI Skill for Mbed TLS（Espressif fork，分支 `mbedtls-4.1.0-idf`，对应上游 Mbed TLS 4.x）。提供基于真实源码的场景化配方、完整 API 参考、配置指南与常见陷阱。覆盖 TLS/DTLS 客户端与服务器、X.509 证书/CSR 解析与签发、PSK、会话恢复、安全加固，以及从 3.x 到 4.x（PSA Crypto）的迁移。

> 本仓库是 Espressif 对上游 Mbed TLS 的 fork（分支名 `mbedtls-4.1.0-idf`），API 与上游 4.x 一致。4.x 的核心变化：仓库拆分为 Mbed TLS（X.509 + TLS）与 TF-PSA-Crypto（加密）；RNG 统一由 PSA 子系统提供；仅支持 CMake 构建。

## Core Principles

1. **PSA Crypto 是 4.x 的加密与 RNG 入口** — 程序入口必须先调用 `psa_crypto_init()`；任何加密操作（解析密钥、解析证书、TLS 握手、签名）之前都依赖它。失败返回负的 `psa_status_t`，成功为 `PSA_SUCCESS`(0)。
2. **不再有手动 RNG** — 4.x 删除了 `mbedtls_entropy_*` / `mbedtls_ctr_drbg_*` / `mbedtls_ssl_conf_rng`。所有需要随机数的操作统一使用 PSA 全局 RNG，公开函数不再接收 `f_rng/p_rng` 参数。
3. **配置已拆分为两个文件** — TLS/X.509 配置在 `include/mbedtls/mbedtls_config.h`（`MBEDTLS_SSL_*`、`MBEDTLS_X509_*` 等）；加密能力在 `tf-psa-crypto/include/psa/crypto_config.h`（`PSA_WANT_*`）。旧的 `MBEDTLS_RSA_C` / `MBEDTLS_AES_C` 等加密 `_C` 宏已废弃。
4. **标准 TLS 客户端调用链（顺序固定）** — `psa_crypto_init` → `*_init` → 加载 CA → `net_connect` → `config_defaults(CLIENT, STREAM/ DATAGRAM, DEFAULT)` → 配置认证/CA → `ssl_setup` → `set_hostname` → `set_bio`(DTLS 还需 `set_timer_cb`) → `handshake`(处理 WANT_READ/WRITE) → 校验 → read/write → `close_notify` → `*_free` + `psa_crypto_free`。
5. **非阻塞 I/O 必须重试 WANT_READ/WANT_WRITE** — `mbedtls_ssl_read/write/handshake` 可能返回这两个码，表示底层暂时不可用，必须重试同一操作，不能当致命错误。
6. **服务器每客户端必须 session_reset** — `net_accept` 后、`set_bio` 前调用 `mbedtls_ssl_session_reset(&ssl)` 复位上下文，否则第二个客户端握手失败；复位前先 `net_free` 旧 client_fd。
7. **DTLS 三大必需项** — transport 设 `DATAGRAM`；BIO 必须提供 `recv_timeout` 回调（`mbedtls_net_recv_timeout`）；必须 `set_timer_cb` 绑定 `mbedtls_timing_set_delay/get_delay`，否则握手重传状态机无法工作。
8. **DTLS 服务器必须启用 HelloVerify cookie** — 通过 `mbedtls_ssl_conf_dtls_cookies` 注册 `mbedtls_ssl_cookie_write/check`，accept 后用 `mbedtls_ssl_set_client_transport_id` 设置客户端地址标识；握手返回 `MBEDTLS_ERR_SSL_HELLO_VERIFY_REQUIRED` 是正常的中间态，reset 重来。
9. **生产环境强制 MBEDTLS_SSL_VERIFY_REQUIRED** — 示例为方便互调用 `OPTIONAL`，生产必须用 `REQUIRED` 并提供完整 CA 链；客户端应调用 `set_hostname` 做证书名校验。
10. **X.509 parse 返回值语义** — `mbedtls_x509_crt_parse`：`<0` 失败，`>0` 成功但跳过的证书数，`0` 完全成功。PEM 输入 buf 必须以 `\0` 结尾（长度含结尾）。
11. **4.x 仅 CMake 构建** — Make 与 Visual Studio 工程已移除；库依赖关系 `mbedtls → mbedx509 → tfpsacrypto`，GNU 链接顺序须为 `-lmbedtls -lmbedx509 -ltfpsacrypto`。`libtfpsacrypto` 亦提供旧名 `libmbedcrypto`。
12. **已删除/改名的 API 必须用新名** — `conf_min/max_version`→`conf_min/max_tls_version`；`conf_curves`→`conf_groups`；`conf_sig_hashes`→`conf_sig_algs`；`x509write_crt_set_serial`→`set_serial_raw`；(D)TLS 1.2 DHE 已移除（仅 ECDHE）。
13. **TLS 1.3 早期数据（0-RTT）非前向安全且无重放保护** — 只用于幂等请求；Mbed TLS 未实现 RFC 8446 第 8 节防重放。客户端 `write_early_data` 即便返回正值也不代表服务端接受，必须握手完成后用 `get_early_data_status` 确认，`REJECTED` 时把已发数据作为普通 `ssl_write` 重发。服务端启用 `conf_early_data` 后，`handshake/read/write` 可能返回 `MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA`，须先 `read_early_data` 再继续。TLS 1.3 还要求 `MBEDTLS_PSA_CRYPTO_C` 与 `MBEDTLS_SSL_KEEP_PEER_CERTIFICATE` 保持启用。

## When to Use

**Applicable:**
- 编写 TLS（TCP）或 DTLS（UDP）客户端/服务器
- 解析、校验、生成 X.509 证书 / CSR / CRL
- 配置加密套件、协议版本、椭圆曲线、签名算法、ALPN、证书 profile
- 使用 PSK（明文 / opaque / 回调）做无证书 TLS
- 启用会话缓存或会话票据加速重连
- 将基于 Mbed TLS 3.x 的代码迁移到 4.x（移除 RNG、改 PSA 配置）
- 裁剪库体积、选用 `configs/` 预设、移植到嵌入式平台

**Not applicable:**
- 上游 TF-PSA-Crypto 纯加密 API 的深度使用（应参考其自身文档/技能）
- 非 Mbed TLS 的其它 TLS 库（OpenSSL、wolfSSL、BoringSSL）
- 硬件安全模块（HSM）或独立 PSA 驱动移植（参考 TF-PSA-Crypto driver 集成文档）
- 上游 3.6 LTS 分支的特定行为（本技能针对 4.x）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应配方**——其中包含完整调用链、分步说明、常见错误与代码示例。

### TLS（基于 TCP）

| recipe | 场景 |
|---|---|
| `recipes/tls_client.md` | TLS 客户端：握手、读写、校验服务器证书（HTTPS / MQTT over TLS） |
| `recipes/tls_server.md` | TLS 服务器：监听、装载证书与私钥、会话缓存、服务多客户端 |
| `recipes/tls_fork_server.md` | 多进程 `fork()` 并发 TLS 服务器（每客户端一进程、SIGCHLD 回收；仅 POSIX） |
| `recipes/smtp_starttls_client.md` | SMTP over TLS / STARTTLS 就地升级（明文 SMTP → 同一 socket 升级 TLS） |
| `recipes/session_resumption.md` | 会话恢复：服务端缓存（ssl_cache）/ 票据（ssl_ticket）、客户端 save/load |

### TLS 1.3

| recipe | 场景 |
|---|---|
| `recipes/tls13_configuration.md` | TLS 1.3 构建配置：四种密钥交换模式（PSK/Ephemeral/PSK-Ephemeral）、中间箱兼容模式、运行时 `conf_tls13_key_exchange_modes` 收窄、硬性依赖（PSA/KEEP_PEER_CERTIFICATE） |
| `recipes/tls13_early_data.md` | TLS 1.3 早期数据（0-RTT）：客户端 `write_early_data` + `get_early_data_status` 拒绝重发、服务端 `RECEIVED_EARLY_DATA` → `read_early_data` |

### DTLS（基于 UDP）

| recipe | 场景 |
|---|---|
| `recipes/dtls_client.md` | DTLS 客户端：定时器回调、recv_timeout、超时重试 |
| `recipes/dtls_server.md` | DTLS 服务器：HelloVerify cookie 防 DoS、客户端地址标识 |

### X.509 证书

| recipe | 场景 |
|---|---|
| `recipes/cert_parse_verify.md` | 解析/校验证书链、打印信息、CRL、密钥用途 |
| `recipes/csr_generation.md` | 生成 CSR（DER/PEM）：主题名、密钥用途、算法 |
| `recipes/cert_signing.md` | 用 CA 签发或自签证书：扩展、SKI/AKI、basic constraints |

### 安全与配置

| recipe | 场景 |
|---|---|
| `recipes/hardening_error.md` | 协议/套件/曲线/签名/profile 收紧、ALPN、错误码诊断 |
| `recipes/psk_tls.md` | PSK（明文/opaque/回调）无证书 TLS/DTLS |
| `recipes/build_config.md` | CMake 构建、配置裁剪、configs/ 预设、config.py |
| `recipes/psa_migration.md` | 3.x → 4.x 迁移：删除 RNG、改 PSA 配置、API 改名 |

---

## 4.x 关键配置速查

### 两个配置文件

| 配置文件 | 路径 | 控制内容 |
|---|---|---|
| TLS / X.509 | `include/mbedtls/mbedtls_config.h` | `MBEDTLS_SSL_*`、`MBEDTLS_X509_*`、`MBEDTLS_NET_C`、`MBEDTLS_TIMING_C` 等 |
| 加密（PSA） | `tf-psa-crypto/include/psa/crypto_config.h` | `PSA_WANT_ALG_*`、`PSA_WANT_KEY_TYPE_*` 等加密能力 |

### 关键 TLS/X.509 宏（mbedtls_config.h）

| 宏 | 作用 |
|---|---|
| `MBEDTLS_SSL_CLI_C` / `MBEDTLS_SSL_SRV_C` | 客户端 / 服务器支持 |
| `MBEDTLS_SSL_PROTO_TLS1_2` / `_TLS1_3` | 协议版本 |
| `MBEDTLS_SSL_PROTO_DTLS` | DTLS 支持 |
| `MBEDTLS_SSL_CACHE_C` / `_TICKET_C` / `_COOKIE_C` | 会话缓存 / 票据 / cookie |
| `MBEDTLS_SSL_EARLY_DATA` | TLS 1.3 0-RTT 早期数据（需 `_PROTO_TLS1_3` + 票据 + PSK/PSK-ephemeral 模式） |
| `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_*_ENABLED` | TLS 1.3 密钥交换模式（PSK / Ephemeral / PSK-Ephemeral） |
| `MBEDTLS_SSL_TLS1_3_COMPATIBILITY_MODE` | TLS 1.3 中间箱兼容模式（默认启用） |
| `MBEDTLS_NET_C` | 网络层（BSD socket） |
| `MBEDTLS_TIMING_C` | 计时（DTLS 定时器回调所需） |
| `MBEDTLS_X509_CRT_PARSE_C` / `_WRITE_C` | 证书解析 / 签发 |
| `MBEDTLS_PEM_PARSE_C` / `_WRITE_C` | PEM 解析 / 输出 |
| `MBEDTLS_ERROR_C` / `MBEDTLS_DEBUG_C` | 错误串 / 调试日志 |

### 库依赖与链接

```
libmbedtls  →  依赖  →  libmbedx509  →  依赖  →  libtfpsacrypto (= libmbedcrypto)
GNU 链接顺序：-lmbedtls -lmbedx509 -ltfpsacrypto
```

---

## 4.x 移除/改名对照

| 3.x | 4.x |
|---|---|
| `mbedtls_ssl_conf_rng` + entropy/ctr_drbg | 删除，改用 `psa_crypto_init()` |
| `mbedtls_ssl_conf_min/max_version` | `mbedtls_ssl_conf_min/max_tls_version`（用 `MBEDTLS_SSL_VERSION_TLS1_2/_TLS1_3`） |
| `mbedtls_ssl_conf_curves` | `mbedtls_ssl_conf_groups` |
| `mbedtls_ssl_conf_sig_hashes` | `mbedtls_ssl_conf_sig_algs` |
| `mbedtls_x509write_crt_set_serial` | `mbedtls_x509write_crt_set_serial_raw` |
| `MBEDTLS_RSA_C` / `MBEDTLS_AES_C` 等加密 `_C` | 移至 PSA `PSA_WANT_*` |
| `MBEDTLS_SSL_DTLS_CONNECTION_ID_COMPAT` | 删除（仅留 RFC 9146 版） |
| `compat-2.x.h` | 删除 |
| (D)TLS 1.2 DHE 密钥交换 | 移除（仅 ECDHE） |
| Make / VS 工程 | 仅 CMake |

---

## Critical Pitfalls (Must Read)

以下是最常见的错误，违反任何一条都会导致编译失败、运行崩溃或安全隐患。完整列表见 `resources/pitfalls.md`。

### 1. 必须调用 psa_crypto_init()，且在任何加密操作之前

```c
// ❌ WRONG — 未初始化 PSA 就解析密钥 / 握手
mbedtls_pk_parse_key(&key, data, len, NULL, 0);   // 可能返回 PSA_ERROR_BAD_STATE
mbedtls_ssl_handshake(&ssl);

// ✅ CORRECT
psa_status_t st = psa_crypto_init();
if (st != PSA_SUCCESS) { /* handle */ }
mbedtls_pk_parse_key(&key, data, len, NULL, 0);
```

### 2. 4.x 不再有手动 RNG，删除 mbedtls_ssl_conf_rng

```c
// ❌ WRONG — 3.x 写法，4.x 编译报错
mbedtls_entropy_context entropy;
mbedtls_ctr_drbg_context ctr_drbg;
mbedtls_entropy_init(&entropy);
mbedtls_ssl_conf_rng(&conf, mbedtls_ctr_drbg_random, &ctr_drbg);

// ✅ CORRECT — 删除全部 RNG 代码
psa_crypto_init();
// 不调用 conf_rng；TLS 自动用 PSA RNG
```

### 3. DTLS 必须绑定定时器回调 + recv_timeout

```c
// ❌ WRONG — DTLS 缺 timer/timeout，握手卡死
mbedtls_ssl_set_bio(&ssl, &fd, mbedtls_net_send, mbedtls_net_recv, NULL);
mbedtls_ssl_handshake(&ssl);

// ✅ CORRECT
mbedtls_ssl_set_bio(&ssl, &fd, mbedtls_net_send, mbedtls_net_recv,
                    mbedtls_net_recv_timeout);
mbedtls_ssl_set_timer_cb(&ssl, &timer,
                         mbedtls_timing_set_delay, mbedtls_timing_get_delay);
```

### 4. 服务器每次循环必须 session_reset

```c
// ❌ WRONG — 第二个客户端握手失败
while (1) {
    mbedtls_net_accept(&listen_fd, &client_fd, NULL, 0, NULL);
    mbedtls_ssl_set_bio(&ssl, &client_fd, ...);
    mbedtls_ssl_handshake(&ssl);

// ✅ CORRECT
while (1) {
    mbedtls_net_free(&client_fd);
    mbedtls_ssl_session_reset(&ssl);
    mbedtls_net_accept(&listen_fd, &client_fd, NULL, 0, NULL);
    mbedtls_ssl_set_bio(&ssl, &client_fd, ...);
    mbedtls_ssl_handshake(&ssl);
}
```

### 5. read/write 必须处理 WANT_READ/WANT_WRITE

```c
// ❌ WRONG — 把非阻塞返回当致命错误
ret = mbedtls_ssl_write(&ssl, buf, len);
if (ret != len) { abort(); }

// ✅ CORRECT — WANT_* 表示暂时不可用，重试
while ((ret = mbedtls_ssl_write(&ssl, buf, len)) <= 0) {
    if (ret != MBEDTLS_ERR_SSL_WANT_READ && ret != MBEDTLS_ERR_SSL_WANT_WRITE) break;
}
```

### 6. DTLS 服务器必须启用 HelloVerify cookie

```c
// ❌ WRONG — 生产 DTLS 服务器不校验 cookie，易受放大攻击

// ✅ CORRECT
mbedtls_ssl_cookie_setup(&cookie_ctx);
mbedtls_ssl_conf_dtls_cookies(&conf, mbedtls_ssl_cookie_write,
                              mbedtls_ssl_cookie_check, &cookie_ctx);
// accept 后：
mbedtls_ssl_set_client_transport_id(&ssl, client_ip, cliip_len);
// 握手可能返回 MBEDTLS_ERR_SSL_HELLO_VERIFY_REQUIRED — 正常，reset 重来
```

### 7. x509_crt_parse 返回值与 buf 结尾

```c
// ❌ WRONG — PEM 未以 '\0' 结尾，且误判 >0 为失败
mbedtls_x509_crt_parse(&cacert, pem, strlen(pem));
if (mbedtls_x509_crt_parse(&cacert, pem, len) != 0) abort();

// ✅ CORRECT — 长度含 '\0'；<0 才失败，>0 是跳过的证书数
size_t len = strlen(pem) + 1;
int ret = mbedtls_x509_crt_parse(&cacert, (const unsigned char *)pem, len);
if (ret < 0) { /* real failure */ }
```

### 8. 生产环境强制 VERIFY_REQUIRED

```c
// ❌ WRONG — 示例为方便互调用 OPTIONAL，生产沿用即不安全
mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_OPTIONAL);

// ✅ CORRECT
mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);
```

### 9. 公开函数不再接收 f_rng/p_rng

```c
// ❌ WRONG — 4.x 签名已变
mbedtls_pk_parse_key(&key, data, len, f_rng, p_rng);
mbedtls_x509write_crt_pem(&crt, buf, size, f_rng, p_rng);

// ✅ CORRECT
mbedtls_pk_parse_key(&key, data, len, pwd, pwd_len);
mbedtls_x509write_crt_pem(&crt, buf, size);
```

### 10. 版本/曲线 API 已改名

```c
// ❌ WRONG — conf_min_version / conf_curves 已删除
mbedtls_ssl_conf_min_version(&conf, MBEDTLS_SSL_MAJOR_VERSION_3, MBEDTLS_SSL_MINOR_VERSION_3);
mbedtls_ssl_conf_curves(&conf, curves);

// ✅ CORRECT
mbedtls_ssl_conf_min_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_2);
mbedtls_ssl_conf_groups(&conf, groups);
```

### 11. set_serial 已改名

```c
// ❌ WRONG — 4.x 删除了 set_serial
mbedtls_x509write_crt_set_serial(&crt, &mpi);

// ✅ CORRECT
mbedtls_x509write_crt_set_serial_raw(&crt, serial_buf, serial_len);
```

### 12. 4.x 仅 CMake，链接顺序固定

```sh
# ❌ WRONG — make 不可用；链接顺序颠倒
make
-ltfpsacrypto -lmbedx509 -lmbedtls

# ✅ CORRECT
cmake -B build /path/to/mbedtls && cmake --build build
-lmbedtls -lmbedx509 -ltfpsacrypto
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确需求：TLS/DTLS？客户端/服务器？是否需证书/PSK/会话恢复？目标平台？ |
| 2 | Recipe | 查 `recipes/` 是否有匹配配方；若有，遵循其调用链 |
| 3 | Query | 配方未覆盖的 API，查 `resources/api_reference.md`；配置查 `config_reference.md` |
| 4 | Validate | 校验所有函数签名、头文件包含、配置宏、4.x 改名项（见 pitfalls） |
| 5 | Confirm | 向用户确认实现计划：PSA 初始化、BIO/timer 配置、认证模式、配置裁剪 |
| 6 | Execute | 新代码：以最接近的 `programs/` 示例为模板改写；已有项目：原地编辑 |
| 7 | Check | 检查：`psa_crypto_init` 调用、WANT_READ/WRITE 处理、服务器 reset、DTLS cookie/timer、链接顺序 |
| 8 | Build | CMake 构建（4.x 唯一）；嵌入式需移植 `platform.h` 与 PSA RNG 源 |
| 9 | Debug | 用 `mbedtls_debug_set_threshold` + `mbedtls_strerror` 排障；用 `ssl_get_verify_result` 查证书问题 |

### Step 6 Detail — 以示例为起点

| 用户需求 | 推荐起点示例 |
|---|---|
| 简单 TLS 客户端 | `programs/ssl/ssl_client1.c` |
| 简单 TLS 服务器 | `programs/ssl/ssl_server.c` |
| 多选项客户端/服务器 | `programs/ssl/ssl_client2.c` / `ssl_server2.c` |
| DTLS 客户端/服务器 | `programs/ssl/dtls_client.c` / `dtls_server.c` |
| 多线程服务器 | `programs/ssl/ssl_pthread_server.c` |
| 多进程（fork）服务器 | `programs/ssl/ssl_fork_server.c`（仅 POSIX） |
| SMTP over TLS / STARTTLS | `programs/ssl/ssl_mail_client.c` |
| TLS 1.3 early data / kex modes | `programs/ssl/ssl_client2.c` / `ssl_server2.c`（`early_data=`、`tls13_kex_modes=`） |
| 证书签发 | `programs/x509/cert_write.c` |
| CSR 生成 | `programs/x509/cert_req.c` |
| 证书校验 | `programs/x509/cert_app.c` |
| 错误码查询 | `programs/util/strerror.c` |

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/` 中查不到 | 立即停止，告知用户该 API 可能不存在或属 TF-PSA-Crypto |
| 编译报 `mbedtls_ssl_conf_rng` 未定义 | 这是 4.x；删除该调用，改用 `psa_crypto_init()`（见 `recipes/psa_migration.md`） |
| 握手卡死（DTLS） | 检查 `set_timer_cb` 与 `recv_timeout` BIO 是否已设置 |
| `PSA_ERROR_BAD_STATE` | 在任何加密操作前补调 `psa_crypto_init()` |
| 第二个客户端握手失败 | accept 前补 `mbedtls_ssl_session_reset` |
| `NO_CIPHERSUITE` | 双方套件/曲线/算法/版本无交集，对齐配置 |
| 证书校验 flags 非零 | 用 `mbedtls_x509_crt_verify_info` 输出详情 |
| 链接未定义引用 | 调整链接顺序为 `-lmbedtls -lmbedx509 -ltfpsacrypto` |
| 子模块缺失编译错 | `git submodule update --init --recursive` |
| 不确定 3.x vs 4.x 行为 | 查 `docs/4.0-migration-guide.md` |
| STARTTLS 后握手失败 | 确保读完 `220` 响应再 `set_bio`+握手；清空明文缓冲区（见 `recipes/smtp_starttls_client.md`） |
| `CANNOT_WRITE_EARLY_DATA` | 未 `conf_early_data` 或未配允许 early data 的 PSK；或已到上限/被拒——改用普通 `ssl_write`（见 `recipes/tls13_early_data.md`） |
| TLS 1.3 启用后构建异常 | 确认 `MBEDTLS_PSA_CRYPTO_C` 与 `MBEDTLS_SSL_KEEP_PEER_CERTIFICATE` 均启用（见 `recipes/tls13_configuration.md`） |

## References

- 场景配方 → `recipes/` 目录
- API 快速参考 → `resources/api_reference.md`
- 配置参考 → `resources/config_reference.md`
- 陷阱汇总 → `resources/pitfalls.md`
- 示例项目索引 → `resources/example_list.md`
- 示例源码 → 上游仓库 `programs/`（SSL/TLS、X.509、工具、测试）
- 迁移指南 → 上游仓库 `docs/4.0-migration-guide.md`
- TLS 1.3 支持 → 上游仓库 `docs/architecture/tls13-support.md`
- 配置预设 → 上游仓库 `configs/`
- 配置脚本 → 上游仓库 `scripts/config.py`
