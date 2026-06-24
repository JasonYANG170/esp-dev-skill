# Mbed TLS API 快速参考（4.x）

> 所有签名均取自 `include/mbedtls/*.h` 与 `tf-psa-crypto/include/{psa,mbedtls}/*.h`。4.x 起 RNG 由 PSA 子系统统一提供（`psa_crypto_init()`），公开函数不再接收 `f_rng/p_rng` 参数。

## PSA Crypto（RNG / 加密入口）

```c
// tf-psa-crypto/include/psa/crypto.h
psa_status_t psa_crypto_init(void);              // 任何加密操作前调用，PSA_SUCCESS 表示成功
void mbedtls_psa_crypto_free(void);              // 释放 PSA 子系统（退出前）

// 通用错误类型 psa_status_t：PSA_SUCCESS=0，负值表示错误（PSA_ERROR_BAD_STATE 等）
```

## SSL 核心（include/mbedtls/ssl.h）

```c
// 生命周期
void mbedtls_ssl_init(mbedtls_ssl_context *ssl);
int  mbedtls_ssl_setup(mbedtls_ssl_context *ssl, const mbedtls_ssl_config *conf);
int  mbedtls_ssl_session_reset(mbedtls_ssl_context *ssl);   // 复位以服务新连接
void mbedtls_ssl_free(mbedtls_ssl_context *ssl);

// 配置
void mbedtls_ssl_config_init(mbedtls_ssl_config *conf);
int  mbedtls_ssl_config_defaults(mbedtls_ssl_config *conf,
                                 int endpoint,      // MBEDTLS_SSL_IS_CLIENT / MBEDTLS_SSL_IS_SERVER
                                 int transport,     // MBEDTLS_SSL_TRANSPORT_STREAM / _DATAGRAM
                                 int preset);       // MBEDTLS_SSL_PRESET_DEFAULT
void mbedtls_ssl_config_free(mbedtls_ssl_config *conf);

// 认证 / 证书
void mbedtls_ssl_conf_authmode(mbedtls_ssl_config *conf, int authmode);
//   MBEDTLS_SSL_VERIFY_NONE / OPTIONAL / REQUIRED
void mbedtls_ssl_conf_ca_chain(mbedtls_ssl_config *conf, mbedtls_x509_crt *ca_chain,
                               mbedtls_x509_crl *ca_crl);
int  mbedtls_ssl_conf_own_cert(mbedtls_ssl_config *conf,
                               const mbedtls_x509_crt *own_cert,
                               const mbedtls_pk_context *pk_key);
void mbedtls_ssl_conf_cert_profile(mbedtls_ssl_config *conf,
                                   const mbedtls_x509_crt_profile *profile);
//   预设：mbedtls_x509_crt_profile_default / _next / _suiteb

// BIO / 计时器
void mbedtls_ssl_set_bio(mbedtls_ssl_context *ssl, void *p_bio,
                         mbedtls_ssl_send_t *f_send,
                         mbedtls_ssl_recv_t *f_recv,
                         mbedtls_ssl_recv_timeout_t *f_recv_timeout);
void mbedtls_ssl_set_timer_cb(mbedtls_ssl_context *ssl, void *p_timer,
                              mbedtls_ssl_set_timer_t *f_set_timer,
                              mbedtls_ssl_get_timer_t *f_get_timer);
int  mbedtls_ssl_set_hostname(mbedtls_ssl_context *ssl, const char *hostname);

// 协议 / 套件
void mbedtls_ssl_conf_min_tls_version(mbedtls_ssl_config *conf, int ver); // MBEDTLS_SSL_VERSION_TLS1_2/_TLS1_3
void mbedtls_ssl_conf_max_tls_version(mbedtls_ssl_config *conf, int ver);
void mbedtls_ssl_conf_ciphersuites(mbedtls_ssl_config *conf, const int *ciphersuites);
void mbedtls_ssl_conf_groups(mbedtls_ssl_config *conf, const uint16_t *group_list);
void mbedtls_ssl_conf_sig_algs(mbedtls_ssl_config *conf, const uint16_t *sig_algs);
int  mbedtls_ssl_conf_alpn_protocols(mbedtls_ssl_config *conf, const char **protos);
const char *mbedtls_ssl_get_alpn_protocol(const mbedtls_ssl_context *ssl);

// 读超时 / DTLS
void mbedtls_ssl_conf_read_timeout(mbedtls_ssl_config *conf, uint32_t timeout_ms);
void mbedtls_ssl_conf_handshake_timeout(mbedtls_ssl_config *conf, uint32_t min, uint32_t max);

// 握手 / 数据
int mbedtls_ssl_handshake(mbedtls_ssl_context *ssl);
int mbedtls_ssl_read(mbedtls_ssl_context *ssl, unsigned char *buf, size_t len);
int mbedtls_ssl_write(mbedtls_ssl_context *ssl, const unsigned char *buf, size_t len);
int mbedtls_ssl_close_notify(mbedtls_ssl_context *ssl);
int mbedtls_ssl_renegotiate(mbedtls_ssl_context *ssl);

// 调试 / 查询
void mbedtls_ssl_conf_dbg(mbedtls_ssl_config *conf,
                          void (*f_dbg)(void*,int,const char*,int,const char*), void *p_dbg);
uint32_t mbedtls_ssl_get_verify_result(const mbedtls_ssl_context *ssl);
const char *mbedtls_ssl_get_ciphersuite(const mbedtls_ssl_context *ssl);
int  mbedtls_ssl_get_session(const mbedtls_ssl_context *ssl, mbedtls_ssl_session *session);
int  mbedtls_ssl_set_session(mbedtls_ssl_context *ssl, const mbedtls_ssl_session *session);

// PSK
int  mbedtls_ssl_conf_psk(mbedtls_ssl_config *conf,
                          const unsigned char *psk, size_t psk_len,
                          const unsigned char *psk_identity, size_t psk_identity_len);
int  mbedtls_ssl_conf_psk_opaque(mbedtls_ssl_config *conf, psa_key_id_t psk,
                                 const unsigned char *psk_identity, size_t psk_identity_len);
void mbedtls_ssl_conf_psk_cb(mbedtls_ssl_config *conf,
                             mbedtls_ssl_psk_external_cb_t f_psk, void *p_psk);

// 会话票据 / ticket
void mbedtls_ssl_conf_session_tickets_cb(mbedtls_ssl_config *conf,
        mbedtls_ssl_ticket_write_t *f_ticket_write,
        mbedtls_ssl_ticket_parse_t *f_ticket_parse, void *p_ticket);
void mbedtls_ssl_conf_new_session_tickets(mbedtls_ssl_config *conf, int new_tickets);

// 会话序列化
int mbedtls_ssl_session_load(mbedtls_ssl_session *session,
                             const unsigned char *buf, size_t len);
int mbedtls_ssl_session_save(const mbedtls_ssl_session *session,
                             unsigned char *buf, size_t buf_size, size_t *olen);

// TLS 1.3 密钥交换模式（仅 MBEDTLS_SSL_PROTO_TLS1_3）
//   kex_modes 为位掩码，取值见下方常量
void mbedtls_ssl_conf_tls13_key_exchange_modes(mbedtls_ssl_config *conf,
                                               const int kex_modes);
//   kex_modes 取值（可位或组合）：
//     MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK             (1u<<0) 纯 PSK
//     MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_EPHEMERAL       (1u<<1) 纯 Ephemeral
//     MBEDTLS_SSL_TLS1_3_KEY_EXCHANGE_MODE_PSK_EPHEMERAL   (1u<<2) PSK+Ephemeral
//   预组合宏：_ALL / _PSK_ALL / _EPHEMERAL_ALL / _NONE(0)
// 查询：握手是否完成（get_early_data_status 的前置条件）
static inline int mbedtls_ssl_is_handshake_over(mbedtls_ssl_context *ssl);  // 1=完成 0=进行中

// TLS 1.3 早期数据 0-RTT（仅 MBEDTLS_SSL_EARLY_DATA）
//   配置（客户端与服务端，默认 DISABLED）
void mbedtls_ssl_conf_early_data(mbedtls_ssl_config *conf, int early_data_enabled);
//     early_data_enabled: MBEDTLS_SSL_EARLY_DATA_DISABLED(0) / _ENABLED(1)
//   仅服务端：NewSessionTicket 中声明的最大早期数据字节数（默认 MBEDTLS_SSL_MAX_EARLY_DATA_SIZE=1024）
void mbedtls_ssl_conf_max_early_data_size(mbedtls_ssl_config *conf,
                                          uint32_t max_early_data_size);
//   仅客户端：在握手首飞写早期数据；返回写入字节数(>=0)，或
//     MBEDTLS_ERR_SSL_CANNOT_WRITE_EARLY_DATA（此后该 ctx 永不能再写早期数据，改用 ssl_write）
int  mbedtls_ssl_write_early_data(mbedtls_ssl_context *ssl,
                                  const unsigned char *buf, size_t len);
//   仅服务端：读取早期数据（仅在 handshake/read/write 返回
//     MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA 后调用）；返回读取字节数(>0)
int  mbedtls_ssl_read_early_data(mbedtls_ssl_context *ssl,
                                 unsigned char *buf, size_t len);
//   仅客户端：查询服务端是否接受早期数据（前置条件：握手已完成）
//     返回 MBEDTLS_SSL_EARLY_DATA_STATUS_NOT_INDICATED / _ACCEPTED / _REJECTED，
//     或 PSA_ERROR_INVALID_ARGUMENT（服务端调用/握手未完成）
int  mbedtls_ssl_get_early_data_status(mbedtls_ssl_context *ssl);
//   相关错误码：
//     MBEDTLS_ERR_SSL_RECEIVED_EARLY_DATA      -0x7C00（服务端：收到早期数据，先 read_early_data）
//     MBEDTLS_ERR_SSL_CANNOT_WRITE_EARLY_DATA  -0x7C80（客户端：无法再写早期数据）
//     MBEDTLS_ERR_SSL_CANNOT_READ_EARLY_DATA   -0x7B80（非 RECEIVED_EARLY_DATA 上下文调用 read）

// DTLS cookie
void mbedtls_ssl_conf_dtls_cookies(mbedtls_ssl_config *conf,
                                   mbedtls_ssl_cookie_write_t *f_cookie_write,
                                   mbedtls_ssl_cookie_check_t *f_cookie_check, void *p_cookie);
int  mbedtls_ssl_set_client_transport_id(mbedtls_ssl_context *ssl,
                                         const unsigned char *info, size_t ilen);
```

## SSL 会话缓存（include/mbedtls/ssl_cache.h）

```c
void mbedtls_ssl_cache_init(mbedtls_ssl_cache_context *cache);
int  mbedtls_ssl_cache_get(void *data, mbedtls_ssl_session *session);
int  mbedtls_ssl_cache_set(void *data, const mbedtls_ssl_session *session);
void mbedtls_ssl_cache_set_timeout(mbedtls_ssl_cache_context *cache, int timeout);
void mbedtls_ssl_cache_set_max_entries(mbedtls_ssl_cache_context *cache, int max);
void mbedtls_ssl_cache_free(mbedtls_ssl_cache_context *cache);
```

## SSL 会话票据（include/mbedtls/ssl_ticket.h）

```c
void mbedtls_ssl_ticket_init(mbedtls_ssl_ticket_context *ctx);
int  mbedtls_ssl_ticket_setup(mbedtls_ssl_ticket_context *ctx);   // 内部生成密钥（依赖 PSA RNG）
int  mbedtls_ssl_ticket_rotate(mbedtls_ssl_ticket_context *ctx, ...);
void mbedtls_ssl_ticket_free(mbedtls_ssl_ticket_context *ctx);
// 作为 conf_session_tickets_cb 回调：mbedtls_ssl_ticket_write / mbedtls_ssl_ticket_read（见头文件）
```

## DTLS Cookie（include/mbedtls/ssl_cookie.h）

```c
void mbedtls_ssl_cookie_init(mbedtls_ssl_cookie_ctx *ctx);
int  mbedtls_ssl_cookie_setup(mbedtls_ssl_cookie_ctx *ctx);
void mbedtls_ssl_cookie_set_timeout(mbedtls_ssl_cookie_ctx *ctx, unsigned long delay);
void mbedtls_ssl_cookie_free(mbedtls_ssl_cookie_ctx *ctx);
// 作为 conf_dtls_cookies 回调：mbedtls_ssl_cookie_write / mbedtls_ssl_cookie_check
```

## 网络（include/mbedtls/net_sockets.h）

```c
typedef struct mbedtls_net_context { int fd; } mbedtls_net_context;

void mbedtls_net_init(mbedtls_net_context *ctx);
int  mbedtls_net_connect(mbedtls_net_context *ctx, const char *host, const char *port, int proto);
//   proto: MBEDTLS_NET_PROTO_TCP / MBEDTLS_NET_PROTO_UDP
int  mbedtls_net_bind(mbedtls_net_context *ctx, const char *bind_ip, const char *port, int proto);
int  mbedtls_net_accept(mbedtls_net_context *bind_ctx, mbedtls_net_context *client_ctx,
                        void *client_ip, size_t buf_size, size_t *client_ip_len);
int  mbedtls_net_poll(mbedtls_net_context *ctx, uint32_t rw, uint32_t timeout);
int  mbedtls_net_set_block(mbedtls_net_context *ctx);
int  mbedtls_net_set_nonblock(mbedtls_net_context *ctx);
// BIO 回调（用于 mbedtls_ssl_set_bio）
int  mbedtls_net_recv(void *ctx, unsigned char *buf, size_t len);
int  mbedtls_net_send(void *ctx, const unsigned char *buf, size_t len);
int  mbedtls_net_recv_timeout(void *ctx, unsigned char *buf, size_t len, uint32_t timeout);
void mbedtls_net_close(mbedtls_net_context *ctx);
void mbedtls_net_free(mbedtls_net_context *ctx);
```

## 计时（include/mbedtls/timing.h，DTLS 定时器回调）

```c
typedef struct mbedtls_timing_delay_context mbedtls_timing_delay_context;
void mbedtls_timing_set_delay(void *data, uint32_t int_ms, uint32_t fin_ms);
int  mbedtls_timing_get_delay(void *data);   // -1 未到期/0 中间/1 最终到期
```

## X.509 证书（include/mbedtls/x509_crt.h）

```c
// 解析
int mbedtls_x509_crt_parse(mbedtls_x509_crt *chain, const unsigned char *buf, size_t buflen);
int mbedtls_x509_crt_parse_file(mbedtls_x509_crt *chain, const char *path);
int mbedtls_x509_crt_parse_path(mbedtls_x509_crt *chain, const char *path);
// 校验
int mbedtls_x509_crt_verify(mbedtls_x509_crt *crt, mbedtls_x509_crt *trust_ca,
                            mbedtls_x509_crl *ca_crl, const char *cn, uint32_t *flags,
                            int (*f_vrfy)(void*, mbedtls_x509_crt*, int, uint32_t*), void *p_vrfy);
int mbedtls_x509_crt_check_key_usage(const mbedtls_x509_crt *crt, unsigned int usage);
int mbedtls_x509_crt_check_extended_key_usage(const mbedtls_x509_crt *crt, const char *usage_oid);
int mbedtls_x509_crt_is_revoked(const mbedtls_x509_crt *crt, const mbedtls_x509_crl *crl);
// 信息
int mbedtls_x509_crt_info(char *buf, size_t size, const char *prefix, const mbedtls_x509_crt *crt);
int mbedtls_x509_crt_verify_info(char *buf, size_t size, const char *prefix, uint32_t flags);
// 生命周期
void mbedtls_x509_crt_init(mbedtls_x509_crt *crt);
void mbedtls_x509_crt_free(mbedtls_x509_crt *crt);
// 签发（write）
void mbedtls_x509write_crt_init(mbedtls_x509write_cert *ctx);
void mbedtls_x509write_crt_set_version(mbedtls_x509write_cert *ctx, int version);
int  mbedtls_x509write_crt_set_serial_raw(mbedtls_x509write_cert *ctx, const unsigned char *serial, size_t serial_len);
int  mbedtls_x509write_crt_set_validity(mbedtls_x509write_cert *ctx, const char *not_before, const char *not_after);
int  mbedtls_x509write_crt_set_issuer_name(mbedtls_x509write_cert *ctx, const char *issuer_name);
int  mbedtls_x509write_crt_set_subject_name(mbedtls_x509write_cert *ctx, const char *subject_name);
void mbedtls_x509write_crt_set_subject_key(mbedtls_x509write_cert *ctx, mbedtls_pk_context *key);
void mbedtls_x509write_crt_set_issuer_key(mbedtls_x509write_cert *ctx, mbedtls_pk_context *key);
void mbedtls_x509write_crt_set_md_alg(mbedtls_x509write_cert *ctx, mbedtls_md_type_t md_alg);
int  mbedtls_x509write_crt_set_basic_constraints(mbedtls_x509write_cert *ctx, int is_ca, int max_pathlen);
int  mbedtls_x509write_crt_set_subject_key_identifier(mbedtls_x509write_cert *ctx);
int  mbedtls_x509write_crt_set_authority_key_identifier(mbedtls_x509write_cert *ctx);
int  mbedtls_x509write_crt_set_key_usage(mbedtls_x509write_cert *ctx, unsigned int key_usage);
int  mbedtls_x509write_crt_set_ext_key_usage(mbedtls_x509write_cert *ctx, const char *ext_key_usage);
int  mbedtls_x509write_crt_set_ns_cert_type(mbedtls_x509write_cert *ctx, unsigned char ns_cert_type);
int  mbedtls_x509write_crt_der(mbedtls_x509write_cert *ctx, unsigned char *buf, size_t size);  // 返回长度
int  mbedtls_x509write_crt_pem(mbedtls_x509write_cert *ctx, unsigned char *buf, size_t size);  // 返回 0
void mbedtls_x509write_crt_free(mbedtls_x509write_cert *ctx);
```

## X.509 CSR（include/mbedtls/x509_csr.h）

```c
int  mbedtls_x509_csr_parse(mbedtls_x509_csr *csr, const unsigned char *buf, size_t buflen);
int  mbedtls_x509_csr_parse_file(mbedtls_x509_csr *csr, const char *path);
int  mbedtls_x509_csr_info(char *buf, size_t size, const char *prefix, const mbedtls_x509_csr *csr);
void mbedtls_x509_csr_init(mbedtls_x509_csr *csr);
void mbedtls_x509_csr_free(mbedtls_x509_csr *csr);
// 生成
void mbedtls_x509write_csr_init(mbedtls_x509write_csr *ctx);
int  mbedtls_x509write_csr_set_subject_name(mbedtls_x509write_csr *ctx, const char *subject_name);
void mbedtls_x509write_csr_set_key(mbedtls_x509write_csr *ctx, mbedtls_pk_context *key);
void mbedtls_x509write_csr_set_md_alg(mbedtls_x509write_csr *ctx, mbedtls_md_type_t md_alg);
int  mbedtls_x509write_csr_set_key_usage(mbedtls_x509write_csr *ctx, unsigned char key_usage);
int  mbedtls_x509write_csr_set_ns_cert_type(mbedtls_x509write_csr *ctx, unsigned char ns_cert_type);
int  mbedtls_x509write_csr_der(mbedtls_x509write_csr *ctx, unsigned char *buf, size_t size);
int  mbedtls_x509write_csr_pem(mbedtls_x509write_csr *ctx, unsigned char *buf, size_t size);
void mbedtls_x509write_csr_free(mbedtls_x509write_csr *ctx);
```

## 公钥（tf-psa-crypto/include/mbedtls/pk.h）

```c
void mbedtls_pk_init(mbedtls_pk_context *ctx);
void mbedtls_pk_free(mbedtls_pk_context *ctx);
int  mbedtls_pk_parse_key(mbedtls_pk_context *ctx, const unsigned char *key, size_t keylen,
                          const unsigned char *pwd, size_t pwdlen);
int  mbedtls_pk_parse_keyfile(mbedtls_pk_context *ctx, const char *path, const char *password);
int  mbedtls_pk_parse_public_key(mbedtls_pk_context *ctx, const unsigned char *key, size_t keylen);
size_t mbedtls_pk_get_bitlen(const mbedtls_pk_context *ctx);
int  mbedtls_pk_check_pair(const mbedtls_pk_context *pub, const mbedtls_pk_context *prv);
int  mbedtls_pk_verify(mbedtls_pk_context *ctx, mbedtls_md_type_t md_alg,
                       const unsigned char *hash, size_t hash_len,
                       const unsigned char *sig, size_t sig_len);
int  mbedtls_pk_sign(mbedtls_pk_context *ctx, mbedtls_md_type_t md_alg,
                     const unsigned char *hash, size_t hash_len,
                     unsigned char *sig, size_t sig_size, size_t *sig_len);
int  mbedtls_pk_write_key_pem(const mbedtls_pk_context *ctx, unsigned char *buf, size_t size);
int  mbedtls_pk_write_pubkey_pem(const mbedtls_pk_context *ctx, unsigned char *buf, size_t size);
```

## 平台 / 错误 / 调试

```c
// include/mbedtls/platform.h
extern int (*mbedtls_printf)(const char *format, ...);
extern void (*mbedtls_exit)(int status);
int mbedtls_platform_set_calloc_free(void *(*calloc_func)(size_t,size_t), void (*free_func)(void*));

// include/mbedtls/error.h
void mbedtls_strerror(int errnum, char *buffer, size_t buflen);

// include/mbedtls/debug.h
void mbedtls_debug_set_threshold(int threshold);
```
