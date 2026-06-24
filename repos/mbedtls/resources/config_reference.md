# Mbed TLS 配置参考（4.x）

> 配置宏来自 `include/mbedtls/mbedtls_config.h`（TLS/X.509）与 `tf-psa-crypto/include/psa/crypto_config.h`（加密能力，`PSA_WANT_*`）。4.x 起加密能力不再用 `MBEDTLS_*_C`（如 `MBEDTLS_RSA_C`）控制，改由 PSA 宏决定。可用 `scripts/config.py --file <path> set/unset <MACRO>` 程序化修改。

## TLS / X.509 配置宏（mbedtls_config.h）

### 网络与平台

| 宏 | 作用 |
|---|---|
| `MBEDTLS_NET_C` | 启用 `mbedtls/net_sockets.h` 的 BSD socket 网络层（`net_connect/bind/accept` 等） |
| `MBEDTLS_TIMING_C` | 启用 `mbedtls/timing.h`（DTLS 定时器回调所需） |
| `MBEDTLS_PLATFORM_C` | 启用平台抽象（`mbedtls_printf`/`exit`/内存分配可移植） |
| `MBEDTLS_FS_IO` | 启用文件 IO（`*_parse_file`、`*_parse_keyfile`） |
| `MBEDTLS_HAVE_TIME` | 证书时间校验依赖 |
| `MBEDTLS_ERROR_C` | 启用 `mbedtls_strerror` |
| `MBEDTLS_DEBUG_C` | 启用 `mbedtls_debug_set_threshold` |

### SSL 协议与角色

| 宏 | 作用 |
|---|---|
| `MBEDTLS_SSL_CLI_C` | TLS/DTLS 客户端支持 |
| `MBEDTLS_SSL_SRV_C` | TLS/DTLS 服务器支持 |
| `MBEDTLS_SSL_PROTO_TLS1_2` | TLS/DTLS 1.2 |
| `MBEDTLS_SSL_PROTO_TLS1_3` | TLS/DTLS 1.3 |
| `MBEDTLS_SSL_PROTO_DTLS` | DTLS（数据报）支持 |
| `MBEDTLS_SSL_DTLS_CONNECTION_ID` | DTLS 1.2 Connection-ID（RFC 9146；旧 `_COMPAT` 已删除） |
| `MBEDTLS_SSL_SESSION_TICKETS` | TLS 1.2 session ticket 支持 |
| `MBEDTLS_SSL_RENEGOTIATION` | TLS 重协商 |
| `MBEDTLS_SSL_ENCRYPT_THEN_MAC` | Encrypt-then-MAC 扩展 |
| `MBEDTLS_SSL_EXTENDED_MASTER_SECRET` | Extended Master Secret 扩展 |
| `MBEDTLS_SSL_MAX_FRAG_LEN` | Max Fragment Length 协商 |

### TLS 1.3 专属（mbedtls_config.h）

> 启用 `MBEDTLS_SSL_PROTO_TLS1_3` 后才生效。硬性依赖：`MBEDTLS_PSA_CRYPTO_C` 与 `MBEDTLS_SSL_KEEP_PEER_CERTIFICATE` 必须保持启用。

| 宏 | 作用 |
|---|---|
| `MBEDTLS_SSL_TLS1_3_COMPATIBILITY_MODE` | 中间箱兼容模式（RFC 8446 D.4，发 dummy ChangeCipherSpec）；默认启用 |
| `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK_ENABLED` | 纯 PSK 密钥交换（无证书/签名代码，体积最小） |
| `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_EPHEMERAL_ENABLED` | 纯 Ephemeral（ECDHE/FFDHE）；需 `X509_CRT_PARSE_C` + 签名算法 |
| `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK_EPHEMERAL_ENABLED` | PSK + Ephemeral 组合（PSK 恢复 + 前向安全，无证书代码） |
| `MBEDTLS_SSL_EARLY_DATA` | TLS 1.3 0-RTT 早期数据；需 `MBEDTLS_SSL_SESSION_TICKETS` + 上面的 PSK 或 PSK-Ephemeral 模式 |
| `MBEDTLS_SSL_MAX_EARLY_DATA_SIZE` | NewSessionTicket 声明的最大早期数据字节数（默认 1024） |

### SSL 辅助模块

| 宏 | 作用 |
|---|---|
| `MBEDTLS_SSL_CACHE_C` | 会话缓存（服务端 session ID 恢复） |
| `MBEDTLS_SSL_TICKET_C` | 会话票据加密/解密 |
| `MBEDTLS_SSL_COOKIE_C` | DTLS HelloVerify cookie |

### X.509

| 宏 | 作用 |
|---|---|
| `MBEDTLS_X509_USE_C` | X.509 基础解析 |
| `MBEDTLS_X509_CRT_PARSE_C` | 证书解析（`x509_crt_parse`） |
| `MBEDTLS_X509_CRT_WRITE_C` | 证书签发（`x509write_crt_*`） |
| `MBEDTLS_X509_CSR_PARSE_C` | CSR 解析 |
| `MBEDTLS_X509_CSR_WRITE_C` | CSR 生成（`x509write_csr_*`） |
| `MBEDTLS_X509_CRL_PARSE_C` | CRL 解析 |
| `MBEDTLS_PEM_PARSE_C` | 解析 PEM 输入 |
| `MBEDTLS_PEM_WRITE_C` | 输出 PEM |

## 加密能力宏（tf-psa-crypto/include/psa/crypto_config.h）

> 4.x 加密能力一律由 `PSA_WANT_*` 控制。示例（非全量）：

| 宏 | 作用 |
|---|---|
| `PSA_WANT_ALG_SHA_256` / `PSA_WANT_ALG_SHA_384` | 哈希算法 |
| `PSA_WANT_ALG_RSA_PKCS1V15_SIGN` / `PSA_WANT_ALG_RSA_PSS` | RSA 签名 |
| `PSA_WANT_ALG_ECDSA` / `PSA_WANT_ALG_DETERMINISTIC_ECDSA` | ECDSA |
| `PSA_WANT_ALG_HKDF` / `PSA_WANT_ALG_HMAC` | 密钥派生/MAC |
| `PSA_WANT_KEY_TYPE_RSA_KEY_PAIR` / `_PUBLIC_KEY` | RSA 密钥类型 |
| `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR` / `_PUBLIC_KEY` | ECC 密钥类型 |
| `PSA_WANT_KEY_TYPE_AES` / `PSA_WANT_ALG_CTR` / `PSA_WANT_ALG_GCM` | 对称加密 |

## 4.x 已删除 / 改名的配置与符号

| 3.x | 4.x 处理 |
|---|---|
| `MBEDTLS_SSL_DTLS_CONNECTION_ID_COMPAT` | 已删除（仅保留 RFC 9146 版） |
| `MBEDTLS_RSA_C` / `MBEDTLS_AES_C` / `MBEDTLS_SHA256_C` 等加密 `_C` 宏 | 移至 PSA `PSA_WANT_*` |
| `compat-2.x.h` | 已删除 |
| (D)TLS 1.2 的 DHE 密钥交换 | 已移除（仅保留 ECDHE） |
| 多个旧加密套件 | 已移除（见迁移指南完整列表） |

## CMake 构建选项（CMakeLists.txt）

| 选项 | 默认 | 作用 |
|---|---|---|
| `ENABLE_TESTING` | ON（子项目 OFF） | 构建 tests/ |
| `USE_SHARED_MBEDTLS_LIBRARY` | OFF | 构建动态库 |
| `MBEDTLS_FATAL_WARNINGS` | ON | 警告视为错误 |
| `GEN_FILES` | 视情况 | 重新生成派生源文件 |
| `CMAKE_BUILD_TYPE` | Release | Release/Debug/Coverage/ASan/MemSan/TSan/Check |
| `MBEDTLS_CONFIG_FILE` | mbedtls_config.h | 指定自定义 TLS 配置头 |

## 预设配置（configs/）

| 文件 | 场景 |
|---|---|
| `configs/config-ccm-psk-tls1_2.h` + `crypto-config-ccm-psk-tls1_2.h` | 仅 TLS 1.2 + CCM + PSK |
| `configs/config-ccm-psk-dtls1_2.h` | 仅 DTLS 1.2 + CCM + PSK |
| `configs/config-suite-b.h` + `crypto-config-suite-b.h` | Suite B 合规 |
| `configs/config-symmetric-only.h` | 仅对称加密 |
| `configs/config-thread.h` + `crypto-config-thread.h` | Thread 协议栈 |
| `configs/config-tfm.h` | TF-M（Trusted Firmware-M） |
