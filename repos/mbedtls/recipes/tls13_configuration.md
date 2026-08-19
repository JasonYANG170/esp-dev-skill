# TLS 1.3 配置与密钥交换模式选择

> **适用摘要**: 启用并裁剪 TLS 1.3——选择四种密钥交换模式（pure-PSK / pure-Ephemeral / PSK-Ephemeral 组合）、开关中间箱兼容模式、满足 TLS 1.3 的硬性前置依赖（`MBEDTLS_PSA_CRYPTO_C`、`MBEDTLS_SSL_KEEP_PEER_CERTIFICATE` 必须保持启用），以及运行时用 `mbedtls_ssl_conf_tls13_key_exchange_modes()` 收窄密钥交换集合。这些是独立于 TLS 1.2 配置的 TLS 1.3 专属配置面。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/mbedtls/resources/`, source/examples in `repos/mbedtls/`, and this recipe path `repos/mbedtls/recipes/tls13_configuration.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "启用 TLS 1.3"
- "TLS 1.3 密钥交换模式 / key exchange mode"
- "只走 PSK 不要证书（减小体积）"
- "tls13_kex_modes"
- "中间箱兼容模式 / middlebox compatibility mode"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_client2.c`、`programs/ssl/ssl_server2.c`（均提供 `tls13_kex_modes=` 命令行选项） |
| 配置宏 | `MBEDTLS_SSL_PROTO_TLS1_3` 必须启用；按下文选择密钥交换模式宏；`MBEDTLS_PSA_CRYPTO_C` 与 `MBEDTLS_SSL_KEEP_PEER_CERTIFICATE` 必须保持启用 |
| 依赖 | `psa_crypto_init()`；Ephemeral 模式还需 `MBEDTLS_X509_CRT_PARSE_C` 及对应签名/密钥交换 PSA 算法 |

## 分步说明

TLS 1.3 可与 1.2 在同一构建中共存（独立启用 `MBEDTLS_SSL_PROTO_TLS1_2` / `_TLS1_3`）。除通用的 TLS 1.2 配置外，TLS 1.3 还提供四个专属构建宏：一个中间箱兼容开关 + 三个密钥交换模式开关。它们直接决定哪些代码被链接进来（按需裁剪体积），以及是否需要证书/签名基础设施。

### 1. 四个 TLS 1.3 专属构建宏（mbedtls_config.h）

| 宏 | 作用 | 何时启用 |
|---|---|---|
| `MBEDTLS_SSL_TLS1_3_COMPATIBILITY_MODE` | 中间箱兼容模式（RFC 8446 D.4）：在 TLS 1.3 连接中发送部分dummy ChangeCipherSpec 记录，使期待 1.2 流量的中间设备不阻断连接 | 默认启用；除非带宽极其敏感且确认无中间盒问题，否则保留 |
| `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_EPHEMERAL_ENABLED` | pure-Ephemeral（ECDHE/FFDHE）密钥交换：需要证书与签名 | 需要基于证书的身份认证时启用 |
| `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK_ENABLED` | pure-PSK 密钥交换：仅 PSK，**不含**任何密钥交换协议、证书或签名代码 | 仅用外部预置 PSK、要最小体积时启用 |
| `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK_EPHEMERAL_ENABLED` | PSK-Ephemeral 组合：PSK + (EC)DHE，提供前向安全；**不含**证书与签名代码 | 用 NewSessionTicket 恢复（默认含 PSK-Ephemeral）或想要 PSK 前向安全时启用 |

### 2. 硬性前置依赖（不可关闭）

TLS 1.3 实现要求以下两个宏**必须**保持默认启用状态：

| 宏 | 原因 |
|---|---|
| `MBEDTLS_PSA_CRYPTO_C` | TLS 1.3 的加密与 RNG 完全依赖 PSA 子系统（4.x 本就强制） |
| `MBEDTLS_SSL_KEEP_PEER_CERTIFICATE` | TLS 1.3 实现需要保留解析出的对端证书 |

关闭任一会导致 TLS 1.3 构建异常。其余 TLS 1.2 配置项大多与 TLS 1.3 兼容，启用 TLS 1.3 时通常无需调整。

### 3. 运行时收窄密钥交换集合

构建时启用某个模式只是"允许"它；运行时可用 `mbedtls_ssl_conf_tls13_key_exchange_modes()` 进一步限制本次连接实际协商的密钥交换。参数是位掩码：

```c
#include "mbedtls/ssl.h"

/* 默认相当于 _ALL（pure-PSK | PSK-Ephemeral | pure-Ephemeral） */
mbedtls_ssl_conf_tls13_key_exchange_modes(&conf,
                                          MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_ALL);

/* 仅 PSK 类（pure-PSK + PSK-Ephemeral），排除纯 Ephemeral —— 适合纯 PSK 设备 */
mbedtls_ssl_conf_tls13_key_exchange_modes(&conf,
                                          MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK_ALL);

/* 仅纯 Ephemeral（基于证书），强制身份认证 */
mbedtls_ssl_conf_tls13_key_exchange_modes(&conf,
                                          MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_EPHEMERAL_ALL);

/* 也可位或自定义组合 */
mbedtls_ssl_conf_tls13_key_exchange_modes(&conf,
                                          MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK |
                                          MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_EPHEMERAL);
```

> 注意：运行时收窄的前提是**构建时已启用**对应模式宏。`conf_tls13_key_exchange_modes` 不能启用构建期未编译进来的模式。若选 PSK 类模式，还需通过 `mbedtls_ssl_conf_psk` / `_psk_opaque` / `_psk_cb` 配置 PSK；若选纯 Ephemeral，服务端还需 `mbedtls_ssl_conf_own_cert` 提供证书。

### 4. 典型裁剪场景

| 场景 | 启用的密钥交换模式宏 | 备注 |
|---|---|---|
| 全功能 TLS 1.3（证书 + PSK + 恢复） | 三者全开（默认） | 体积最大，链接证书/签名代码 |
| 仅 PSK 设备（无证书、极小体积） | 仅 `_PSK_ENABLED` | 链接进来的代码不含密钥交换协议/证书/签名 |
| PSK 恢复 + 前向安全，无证书 | `_PSK_ENABLED` + `_PSK_EPHEMERAL_ENABLED` | 用 NewSessionTicket 恢复并保持 PFS |
| 仅基于证书的 Ephemeral | 仅 `_EPHEMERAL_ENABLED` | 需 `X509_CRT_PARSE_C` + 签名算法 |

### 5. 用 config.py 程序化修改

```sh
# 启用 TLS 1.3
python3 scripts/config.py --file include/mbedtls/mbedtls_config.h set MBEDTLS_SSL_PROTO_TLS1_3

# 关闭中间箱兼容模式（节省少量带宽）
python3 scripts/config.py --file include/mbedtls/mbedtls_config.h unset MBEDTLS_SSL_TLS1_3_COMPATIBILITY_MODE

# 仅保留 PSK 类密钥交换（去掉 Ephemeral 以减小体积）
python3 scripts/config.py --file include/mbedtls/mbedtls_config.h unset MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_EPHEMERAL_ENABLED
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 启用 TLS 1.3 后构建异常 | 关闭了 `MBEDTLS_PSA_CRYPTO_C` 或 `MBEDTLS_SSL_KEEP_PEER_CERTIFICATE` | 两者必须保持启用（TLS 1.3 硬性依赖） |
| `conf_tls13_key_exchange_modes` 选了某模式但握手 `NO_CIPHERSUITE` | 构建期未启用对应 `_ENABLED` 宏 | 在 `mbedtls_config.h` 启用对应密钥交换模式宏 |
| 选 pure-Ephemeral 但握手失败 | 服务端未提供证书 | 调 `mbedtls_ssl_conf_own_cert` 装载证书与私钥 |
| 选 PSK 类但握手失败 | 未配置 PSK | 调 `mbedtls_ssl_conf_psk` / `_psk_opaque` / `_psk_cb` 配置 PSK 与 identity |
| 中间设备（旧防火墙/代理）阻断 TLS 1.3 连接 | 关闭了兼容模式 | 保留 `MBEDTLS_SSL_TLS1_3_COMPATIBILITY_MODE` 启用 |
| `MBEDTLS_SSL_PROTO_TLS1_3` 不生效 | 关闭后相关 `_KEY_EXCHANGE_MODE_*` 宏无效果 | 这些 TLS 1.3 宏只在 `_PROTO_TLS1_3` 启用时才有意义 |

## 参考

- 文档: `docs/architecture/tls13-support.md` — TLS 1.3 能力清单、ClientHello 扩展支持矩阵、TLS 1.3 专属构建选项说明、硬性依赖（PSA / KEEP_PEER_CERTIFICATE）
- 示例: `programs/ssl/ssl_client2.c`、`programs/ssl/ssl_server2.c`（`tls13_kex_modes=` 命令行选项取值 `psk`/`psk_ephemeral`/`ephemeral`/`ephemeral_all`/`psk_all`/`all`）
- 头文件: `include/mbedtls/ssl.h`（`mbedtls_ssl_conf_tls13_key_exchange_modes` 及 `MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_*` 位掩码常量）
- 配置头: `include/mbedtls/mbedtls_config.h`（`MBEDTLS_SSL_TLS1_3_*` 构建宏）
- 脚本: `scripts/config.py`
