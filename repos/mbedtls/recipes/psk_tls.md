# PSK（预共享密钥）TLS/DTLS

> **适用摘要**: 在 TLS/DTLS 中使用预共享密钥（PSK）做认证，无需证书。适用于资源受限、双方已共享密钥的嵌入式设备加密通信。

## 触发意图

- "PSK 加密"
- "预共享密钥 TLS"
- "无证书 TLS"
- "psk 回调"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `programs/ssl/ssl_client2.c`（含 PSK 选项）、`programs/ssl/ssl_server2.c` |
| 配置宏 | 需启用 PSK 相关支持；TLS 仍需 `MBEDTLS_SSL_CLI_C` / `MBEDTLS_SSL_SRV_C` |
| 依赖 | 需 `psa_crypto_init()` |

## 分步说明

Mbed TLS 提供两种使用 PSK 的方式：(1) 直接配置明文 PSK（`mbedtls_ssl_conf_psk`，适合简单场景）；(2) 配置存放在 PSA 密钥槽中的 opaque PSK（`mbedtls_ssl_conf_psk_opaque`，密钥不暴露在内存，更安全）；(3) 注册 PSK 回调（`mbedtls_ssl_conf_psk_cb`），由服务器根据客户端身份动态查找密钥。

### 1. 直接配置明文 PSK（客户端 + 服务器）

```c
unsigned char psk[] = { /* 预共享密钥字节 */ };
size_t psk_len = sizeof(psk);
const unsigned char psk_identity[] = "Client_identity";

psa_crypto_init();
/* ... config_defaults / setup ... */

/* 客户端与服务端都调用同一函数配置 PSK 及其 identity */
ret = mbedtls_ssl_conf_psk(&conf, psk, psk_len,
                           psk_identity, sizeof(psz_identity) - 1);
if (ret != 0) { /* 失败处理 */ }

/* 不再调用 conf_own_cert / conf_ca_chain —— 用 PSK 替代证书 */
```

### 2. 使用 opaque PSK（PSA 密钥槽，更安全）

```c
/* 先用 PSA API 把 PSK 导入到密钥槽（见 TF-PSA-Crypto 文档），得到 slot / key_id */
psa_key_id_t slot = /* ... psa_import_key(...) ... */;

ret = mbedtls_ssl_conf_psk_opaque(&conf, slot,
                                  psk_identity, sizeof(psz_identity) - 1);
```

### 3. 服务器端动态 PSK 回调

```c
/* 回调签名：根据客户端 identity 返回对应 PSK */
static int psk_callback(void *p_ctx, int ssl,
                        const unsigned char *name, size_t name_len,
                        unsigned char *psk_buf, size_t psk_buf_len)
{
    /* 根据 name 查找并填充 psk_buf，返回 PSK 长度；找不到返回负值 */
    return (int) psk_len;
}

mbedtls_ssl_conf_psk_cb(&conf, psk_callback, &my_context);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 握手 `NO_CIPHERSUITE` | 双方 PSK 或算法不匹配 | 确认 PSK、identity、加密套件一致 |
| PSK identity 不符 | 客户端/服务器 identity 字符串不同 | 两端使用相同 identity |
| 明文 PSK 内存泄露 | 密钥留在 RAM | 优先用 `conf_psk_opaque`，密钥存 PSA 槽 |
| 回调返回值错误 | 把长度当负值返回 | 成功返回 PSK 字节数（>0），失败返回负错误码 |
| TLS1_3 不支持明文 PSK？ | TLS 1.3 PSK 语义不同 | 注意 TLS 1.3 PSK/会话票据机制，参考 `docs/architecture/tls13-support.md` |

## 参考

- 示例: `programs/ssl/ssl_client2.c`（PSK / PSK opaque 选项）
- 示例: `programs/ssl/ssl_server2.c`
- 头文件: `include/mbedtls/ssl.h`（`mbedtls_ssl_conf_psk`、`mbedtls_ssl_conf_psk_opaque`、`mbedtls_ssl_conf_psk_cb`）
