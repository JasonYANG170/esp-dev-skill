# 示例项目索引（programs/）

> 以下示例取自仓库 `programs/` 目录（描述来自 `programs/README.md`）。示例旨在演示单一特性，生产使用前需单独审计。

## SSL/TLS 示例（programs/ssl/）

| 路径 | 说明 |
|---|---|
| `programs/ssl/ssl_client1.c` | 简单 HTTPS 客户端：发送固定请求并显示响应（推荐作为新客户端起点） |
| `programs/ssl/ssl_server.c` | 简单 HTTPS 服务器：单连接、返回固定响应（推荐作为新服务器起点） |
| `programs/ssl/ssl_client2.c` | 客户端特性全集演示：可选 TLS/库特性（套件、PSK、版本、ALPN 等） |
| `programs/ssl/ssl_server2.c` | 服务器特性全集演示：可选 TLS/库特性 |
| `programs/ssl/mini_client.c` | 极简 SSL 客户端（主要用于基准测试） |
| `programs/ssl/dtls_client.c` | 简单 DTLS 客户端：发一个数据报、读一个响应 |
| `programs/ssl/dtls_server.c` | 简单 DTLS 服务器：含 HelloVerify cookie |
| `programs/ssl/ssl_mail_client.c` | SMTP-over-TLS / SMTP-STARTTLS 客户端：发送固定邮件 |
| `programs/ssl/ssl_fork_server.c` | HTTPS 服务器：每客户端一进程（需 POSIX `fork`） |
| `programs/ssl/ssl_pthread_server.c` | HTTPS 服务器：每客户端一线程（需 pthread） |
| `programs/ssl/ssl_context_info.c` | SSL 上下文信息工具 |

辅助：`programs/ssl/ssl_test_lib.{c,h}`、`programs/ssl/ssl_test_common_source.c`（测试公共代码）。

## X.509 证书示例（programs/x509/）

| 路径 | 说明 |
|---|---|
| `programs/x509/cert_app.c` | 连接 TLS 服务器并校验其证书链 |
| `programs/x509/cert_req.c` | 为私钥生成证书签名请求（CSR） |
| `programs/x509/cert_write.c` | 签发 CSR 或自签证书 |
| `programs/x509/crl_app.c` | 加载并 dump 证书吊销列表（CRL） |
| `programs/x509/req_app.c` | 加载并 dump 证书签名请求（CSR） |
| `programs/x509/load_roots.c` | 加载受信任根证书 |

## 工具（programs/util/）

| 路径 | 说明 |
|---|---|
| `programs/util/pem2der.c` | PEM 转 DER 转换器（无 PEM 支持的最小构建可用） |
| `programs/util/strerror.c` | 打印 Mbed TLS 整数错误码对应的描述 |

## 测试程序（programs/test/）

| 路径 | 说明 |
|---|---|
| `programs/test/selftest.c` | 运行各库模块的自测函数 |
| `programs/test/udp_proxy.c` | UDP 代理：可注入延迟/重复/丢包，用于 DTLS 测试 |
| `programs/test/cmake_package/` | 演示 `find_package(MbedTLS)` 消费方式 |
| `programs/test/cmake_package_install/` | 演示安装后通过 `CMAKE_PREFIX_PATH` 消费 |
| `programs/test/cmake_subproject/` | 演示 `add_subdirectory()` 作为子项目 |
| `programs/test/dlopen.c` | 运行时动态加载示例 |

## 其它

| 路径 | 说明 |
|---|---|
| `programs/fuzz/` | 模糊测试入口 |
