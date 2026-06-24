# Mbed TLS 常见陷阱汇总（4.x）

> 以下陷阱基于 `docs/4.0-migration-guide.md` 与 `programs/` 示例的真实行为。违反这些会导致编译失败、运行崩溃或安全隐患。

## 1. 必须��用 psa_crypto_init()，且在任何加密操作之前

```c
// ❌ 错误：未初始化 PSA 就解析密钥 / 握手
mbedtls_pk_parse_key(&key, ...);   // 可能返回 PSA_ERROR_BAD_STATE
mbedtls_ssl_handshake(&ssl);       // 崩溃或失败

// ✅ 正确：程序入口先初始化
psa_status_t st = psa_crypto_init();
if (st != PSA_SUCCESS) { /* 处理 */ }
```

> 规则：解析密钥、解析证书、生成 CSR/证书、TLS/DTLS 握手、任何签名/加解密——之前都必须已调用 `psa_crypto_init()`。

## 2. 4.x 不再有手动 RNG，不要调用 mbedtls_ssl_conf_rng

```c
// ❌ 错误：3.x 写法，4.x 编译报错（函数已删除）
mbedtls_entropy_context entropy;
mbedtls_ctr_drbg_context ctr_drbg;
mbedtls_entropy_init(&entropy);
mbedtls_ctr_drbg_seed(&ctr_drbg, mbedtls_entropy_func, &entropy, ...);
mbedtls_ssl_conf_rng(&conf, mbedtls_ctr_drbg_random, &ctr_drbg);

// ✅ 正确：删除全部 RNG 代码，改用 PSA 全局 RNG
psa_crypto_init();
// 不调用 conf_rng；TLS 自动用 PSA RNG
```

## 3. 公开函数不再接收 f_rng / p_rng 参数

```c
// ❌ 错误：4.x 这些函数签名已变
mbedtls_pk_parse_key(&key, data, len, f_rng, p_rng);
mbedtls_x509write_crt_pem(&crt, buf, size, f_rng, p_rng);

// ✅ 正确
mbedtls_pk_parse_key(&key, data, len, pwd, pwd_len);   // 后两参是口令
mbedtls_x509write_crt_pem(&crt, buf, size);
```

## 4. mbedtls_x509_crt_parse 的返回值与 buf 结尾

```c
// ❌ 错误：PEM 数据未以 '\0' 结尾
mbedtls_x509_crt_parse(&cacert, pem, strlen(pem));   // 漏掉结尾，解析失败

// ❌ 错误：误判返回值（>0 当成失败）
if (mbedtls_x509_crt_parse(&cacert, pem, len) != 0) abort();

// ✅ 正确：传入含 '\0' 的长度；<0 才是失败，>0 是跳过的证书数
size_t len = strlen(pem) + 1;
int ret = mbedtls_x509_crt_parse(&cacert, (const unsigned char *)pem, len);
if (ret < 0) { /* 真正失败 */ }
```

## 5. DTLS 必须绑定定时器回调，否则握手卡死

```c
// ❌ 错误：DTLS 未设置 timer 回调
mbedtls_ssl_config_defaults(&conf, MBEDTLS_SSL_IS_CLIENT,
                            MBEDTLS_SSL_TRANSPORT_DATAGRAM, ...);
mbedtls_ssl_setup(&ssl, &conf);
mbedtls_ssl_set_bio(&ssl, &fd, mbedtls_net_send, mbedtls_net_recv, NULL);  // 缺 timeout
mbedtls_ssl_handshake(&ssl);   // 卡住，无重传

// ✅ 正确：DTLS 三参数 BIO + 定时器回调
mbedtls_ssl_set_bio(&ssl, &fd, mbedtls_net_send, mbedtls_net_recv,
                    mbedtls_net_recv_timeout);
mbedtls_ssl_set_timer_cb(&ssl, &timer,
                         mbedtls_timing_set_delay, mbedtls_timing_get_delay);
```

## 6. mbedtls_ssl_read/write 必须处理 WANT_READ/WANT_WRITE

```c
// ❌ 错误：把非阻塞返回当成致命错误
ret = mbedtls_ssl_read(&ssl, buf, len);
if (ret != len) { panic(); }

// ✅ 正确：WANT_* 表示暂时不可用，重试
while ((ret = mbedtls_ssl_write(&ssl, buf, len)) <= 0) {
    if (ret != MBEDTLS_ERR_SSL_WANT_READ && ret != MBEDTLS_ERR_SSL_WANT_WRITE) break;
}
```

## 7. 服务器每次循环必须 session_reset，否则第二个客户端握手失败

```c
// ❌ 错误：复用未复位的 ssl 上下文服务下一个客户端
while (1) {
    mbedtls_net_accept(&listen_fd, &client_fd, NULL, 0, NULL);
    mbedtls_ssl_set_bio(&ssl, &client_fd, ...);
    mbedtls_ssl_handshake(&ssl);   // 第二次失败

// ✅ 正确：accept 前 reset
while (1) {
    mbedtls_ssl_session_reset(&ssl);
    mbedtls_net_accept(&listen_fd, &client_fd, NULL, 0, NULL);
    mbedtls_ssl_set_bio(&ssl, &client_fd, ...);
    mbedtls_ssl_handshake(&ssl);
    // ... reset 前先 mbedtls_net_free(&client_fd) ...
}
```

## 8. DTLS 服务器必须启用 HelloVerify cookie（防 DoS）

```c
// ❌ 错误：生产 DTLS 服务器不校验 cookie，易受放大攻击
// （未调用 conf_dtls_cookies）

// ✅ 正确
mbedtls_ssl_cookie_setup(&cookie_ctx);
mbedtls_ssl_conf_dtls_cookies(&conf, mbedtls_ssl_cookie_write,
                              mbedtls_ssl_cookie_check, &cookie_ctx);
// accept 后：
mbedtls_ssl_set_client_transport_id(&ssl, client_ip, cliip_len);
// 握手可能返回 MBEDTLS_ERR_SSL_HELLO_VERIFY_REQUIRED —— 这是正常的，reset 重来
```

## 9. 生产环境用 MBEDTLS_SSL_VERIFY_REQUIRED，不要用 OPTIONAL/NONE

```c
// ❌ 错误：示例为方便互调用 OPTIONAL，生产沿用即不安全
mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_OPTIONAL);

// ✅ 正确：强制校验
mbedtls_ssl_conf_authmode(&conf, MBEDTLS_SSL_VERIFY_REQUIRED);
mbedtls_ssl_conf_ca_chain(&conf, &cacert, NULL);
// 握手后还可检查 flags：
uint32_t flags = mbedtls_ssl_get_verify_result(&ssl);
```

## 10. 客户端应调用 mbedtls_ssl_set_hostname（SNI + 证书校验）

```c
// ❌ 错误：客户端不设 hostname，SNI 不发送、证书名校验失效
mbedtls_ssl_setup(&ssl, &conf);
mbedtls_ssl_handshake(&ssl);

// ✅ 正确
mbedtls_ssl_setup(&ssl, &conf);
mbedtls_ssl_set_hostname(&ssl, "example.com");
```

## 11. 版本 API 已改名（4.x）

```c
// ❌ 错误：3.x 的 conf_min_version / conf_max_version 已删除
mbedtls_ssl_conf_min_version(&conf, MBEDTLS_SSL_MAJOR_VERSION_3, MBEDTLS_SSL_MINOR_VERSION_3);

// ✅ 正确
mbedtls_ssl_conf_min_tls_version(&conf, MBEDTLS_SSL_VERSION_TLS1_2);
```

## 12. 曲线/签名 API 已改名

```c
// ❌ 错误：conf_curves / conf_sig_hashes 已删除
mbedtls_ssl_conf_curves(&conf, curves);
mbedtls_ssl_conf_sig_hashes(&conf, hashes);

// ✅ 正确
mbedtls_ssl_conf_groups(&conf, groups);
mbedtls_ssl_conf_sig_algs(&conf, sig_algs);
```

## 13. set_serial 已改名为 set_serial_raw

```c
// ❌ 错误：4.x 删除了 set_serial
mbedtls_x509write_crt_set_serial(&crt, &mpi_serial);

// ✅ 正确
mbedtls_x509write_crt_set_serial_raw(&crt, serial_buf, serial_len);
```

## 14. 链接顺序（GNU 链接器）

```sh
# ❌ 错误：顺序颠倒导致未定义引用
-ltfpsacrypto -lmbedx509 -lmbedtls

# ✅ 正确：依赖库在后
-lmbedtls -lmbedx509 -ltfpsacrypto
# （libtfpsacrypto 亦提供旧名 libmbedcrypto）
```

## 15. 4.x 仅支持 CMake，不要找 Makefile / VS 工程

```sh
# ❌ make 已不可用
make

# ✅ CMake
cmake -B build /path/to/mbedtls
cmake --build build
```

## 16. 构建/克隆后必须初始化子模块

```sh
# 4.x 拆分为 Mbed TLS + TF-PSA-Crypto（含嵌套子模块）
git submodule update --init --recursive
# 缺失会导致 crypto 头/源找不到，编译失败
```
