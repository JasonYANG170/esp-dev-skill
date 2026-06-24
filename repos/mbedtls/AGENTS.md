# AGENTS.md — 补充约定

> 核心规则、配方索引、陷阱、执行工作流均在 `SKILL.md`。
> 本文件仅覆盖 `SKILL.md` 未涉及的约定与工具指导，不重复内容。

## Project Context

**Language**: C (C99) · **Target**: 跨平台（POSIX 主机、嵌入式）· **Toolchain**: CMake ≥ 3.20.2，C99 编译器（GCC/Clang/MSVC/Armclang）· **Repo**: Mbed TLS（Espressif fork `mbedtls-4.1.0-idf`，上游 4.x）

> 4.x 关键事实：仓库 = Mbed TLS（X.509 + TLS）+ TF-PSA-Crypto（加密，作为子模块）；RNG 由 PSA 子系统统一提供；仅 CMake 构建。

## Code Generation Conventions

### 头文件包含

```c
/* 任何 Mbed TLS 源文件的标准起点 */
#include "mbedtls/build_info.h"     /* 必须最先：拉入 mbedtls_config.h 与编译期信息 */
#include "mbedtls/platform.h"       /* 平台抽象（printf/exit/内存），第二 */

/* 按需引入模块 */
#include "mbedtls/ssl.h"            /* TLS/DTLS */
#include "mbedtls/net_sockets.h"    /* BSD socket 网络层 */
#include "mbedtls/x509_crt.h"       /* 证书 */
#include "mbedtls/x509_csr.h"       /* CSR */
#include "mbedtls/x509_crl.h"       /* CRL */
#include "mbedtls/ssl_cache.h"      /* 会话缓存 */
#include "mbedtls/ssl_ticket.h"     /* 会话票据 */
#include "mbedtls/ssl_cookie.h"     /* DTLS cookie */
#include "mbedtls/timing.h"         /* DTLS 定时器回调 */
#include "mbedtls/error.h"          /* mbedtls_strerror */
#include "mbedtls/debug.h"          /* 调试日志 */

/* 公钥（位于 TF-PSA-Crypto） */
#include "mbedtls/pk.h"

/* PSA Crypto 入口（4.x RNG/加密） */
#include "psa/crypto.h"             /* psa_crypto_init() */
```

> 4.x 某些写证书/测试程序需要在 `build_info.h` 之前定义 `MBEDTLS_DECLARE_PRIVATE_IDENTIFIERS`（见 `programs/x509/cert_write.c`）。

### 标准 TLS 客户端骨架

```c
#include "mbedtls/build_info.h"
#include "mbedtls/platform.h"
#include "mbedtls/net_sockets.h"
#include "mbedtls/ssl.h"
#include "mbedtls/error.h"

int main(void) {
    mbedtls_net_context server_fd;
    mbedtls_ssl_context ssl;
    mbedtls_ssl_config conf;
    mbedtls_x509_crt cacert;

    mbedtls_net_init(&server_fd);
    mbedtls_ssl_init(&ssl);
    mbedtls_ssl_config_init(&conf);
    mbedtls_x509_crt_init(&cacert);

    if (psa_crypto_init() != PSA_SUCCESS) { return 1; }   /* 4.x 必需 */

    mbedtls_x509_crt_parse(&cacert, ca_pem, ca_pem_len);
    mbedtls_net_connect(&server_fd, host, port, MBEDTLS_NET_PROTO_TCP);

    mbedtls_ssl_config_defaults(&conf, MBEDTLS_SSL_IS_CLIENT,
                                MBEDTLS_SSL_TRANSPORT_STREAM,
                                MBEDTLS_SSL_PRESET_DEFAULT);
    mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
    mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);
    mbedtls_ssl_setup(&ssl, &conf);
    mbedtls_ssl_set_hostname(&ssl, host);
    mbedtls_ssl_set_bio(&ssl, &server_fd, mbedtls_net_send, mbedtls_net_recv, NULL);

    /* handshake / read / write，处理 WANT_READ/WANT_WRITE ... */

    mbedtls_ssl_close_notify(&ssl);
    mbedtls_net_free(&server_fd);
    mbedtls_x509_crt_free(&cacert);
    mbedtls_ssl_free(&ssl);
    mbedtls_ssl_config_free(&conf);
    mbedtls_psa_crypto_free();
    return 0;
}
```

### 调试回调模板

```c
static void my_debug(void *ctx, int level, const char *file, int line, const char *str) {
    (void)level;
    mbedtls_fprintf((FILE *)ctx, "%s:%04d: %s", file, line, str);
    fflush((FILE *)ctx);
}

/* 配置时注册 + 设阈值 */
mbedtls_ssl_conf_dbg(&conf, my_debug, stdout);
mbedtls_debug_set_threshold(2);   /* 0=关，越大越详细 */
```

### 错误码处理模板

```c
/* Mbed TLS 错误码为负；打印可读串 */
if (ret < 0) {
    char buf[128];
    mbedtls_strerror(ret, buf, sizeof(buf));
    mbedtls_printf("err -0x%04x: %s\n", (unsigned)-ret, buf);
}
```

### 资源释放配对（必须成对调用）

| init / setup | free / reset |
|---|---|
| `mbedtls_net_init` | `mbedtls_net_free` |
| `mbedtls_ssl_init` + `ssl_setup` | `mbedtls_ssl_free`（每连接前 `ssl_session_reset`） |
| `mbedtls_ssl_config_init` | `mbedtls_ssl_config_free` |
| `mbedtls_x509_crt_init` | `mbedtls_x509_crt_free` |
| `mbedtls_pk_init` | `mbedtls_pk_free` |
| `mbedtls_ssl_cache_init` | `mbedtls_ssl_cache_free` |
| `mbedtls_ssl_ticket_init` | `mbedtls_ssl_ticket_free` |
| `mbedtls_ssl_cookie_init` | `mbedtls_ssl_cookie_free` |
| `mbedtls_x509write_crt_init` | `mbedtls_x509write_crt_free` |
| `mbedtls_x509write_csr_init` | `mbedtls_x509write_csr_free` |
| `psa_crypto_init` | `mbedtls_psa_crypto_free` |

## Build Workflow

```sh
# 1. 初始化子模块（4.x 必需，含 TF-PSA-Crypto）
git submodule update --init --recursive

# 2. CMake 配置 + 构建（4.x 仅 CMake）
cmake -B build /path/to/mbedtls
cmake --build build

# 3. 运行示例（构建产物在 build/programs/）
./build/programs/ssl/ssl_server        # 监听 4433
./build/programs/ssl/ssl_client1       # 连 localhost:4433

# 4. 运行测试（需 Python + ENABLE_TESTING）
ctest --test-dir build
```

### 常用 CMake 选项

```sh
cmake -B build /path/to/mbedtls \
  -DENABLE_TESTING=Off \
  -DUSE_SHARED_MBEDTLS_LIBRARY=On \
  -DCMAKE_BUILD_TYPE=Debug
```

## Code Generation Checklist

- [ ] `psa_crypto_init()` 在任何加密操作前调用（解析密钥/证书、握手、签名）
- [ ] 不存在 `mbedtls_entropy_*` / `mbedtls_ctr_drbg_*` / `mbedtls_ssl_conf_rng`（4.x 已删）
- [ ] 公开函数调用未传 `f_rng/p_rng`（4.x 已去掉该参数）
- [ ] TLS 客户端调用了 `mbedtls_ssl_set_hostname`
- [ ] 生产用 `MBEDTLS_SSL_VERIFY_REQUIRED`，提供完整 CA 链
- [ ] `ssl_read/write/handshake` 都处理了 `WANT_READ/WANT_WRITE`
- [ ] 服务器循环中 accept 前 `mbedtls_ssl_session_reset`
- [ ] DTLS：设了 `recv_timeout` BIO 与 `set_timer_cb`
- [ ] DTLS 服务器：注册了 cookie 回调 + `set_client_transport_id`
- [ ] 用了 4.x 新 API（`conf_min/max_tls_version`、`conf_groups`、`conf_sig_algs`、`set_serial_raw`）
- [ ] PEM parse 传入的长度含结尾 `\0`
- [ ] 所有 init 都有配对的 free；退出前 `mbedtls_psa_crypto_free`
- [ ] 链接顺序 `-lmbedtls -lmbedx509 -ltfpsacrypto`

## Do Not Modify

- `resources/` — API/配置文档源（基于上游头文件）
- 上游仓库 `include/`、`library/`、`tf-psa-crypto/` 源码 — 改动应提交到上游，不在应用层打补丁
- `SKILL.md` frontmatter — Skill 元数据
