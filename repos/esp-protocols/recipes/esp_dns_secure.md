# esp_dns 安全 DNS（DoT / DoH / TCP）

> **适用摘要**: 用 esp_dns 组件建立 DNS over TLS（DoT）、DNS over HTTPS（DoH）或 TCP DNS 解析，绕过明文 UDP DNS、提升隐私与可靠性。

## 触发意图

- "DNS over TLS / DoT"
- "DNS over HTTPS / DoH"
- "加密 DNS"
- "esp_dns"
- "自定义 DNS 服务器"

## 前置条件

| 条件 | 要求 |
|---|---|
| 网络 | 已连外网 |
| 组件依赖 | `espressif/esp_dns` |
| TLS（DoT/DoH） | 需服务器 CA 证书或证书包 |
| 参考头文件 | `components/esp_dns/include/esp_dns.h` |

## 分步说明

### 1. 默认端口与协议枚举（来自头文件）

```c
#include "esp_dns.h"

// 默认端口
ESP_DNS_DEFAULT_TCP_PORT   // 53
ESP_DNS_DEFAULT_DOT_PORT   // 853
ESP_DNS_DEFAULT_DOH_PORT   // 443
ESP_DNS_DEFAULT_TIMEOUT_MS // 10000

// 协议类型
// ESP_DNS_PROTOCOL_UDP  传统 UDP DNS
// ESP_DNS_PROTOCOL_TCP  TCP DNS
// ESP_DNS_PROTOCOL_DOT  DNS over TLS
// ESP_DNS_PROTOCOL_DOH  DNS over HTTPS
```

### 2. 初始化 DoT（DNS over TLS）

```c
esp_dns_config_t cfg = {
    .protocol = ESP_DNS_PROTOCOL_DOT,
    .dns_server = "1.1.1.1",          // 支持 Cloudflare/Google 等 DoT 服务器
    .port = ESP_DNS_DEFAULT_DOT_PORT, // 853
    .timeout_ms = ESP_DNS_DEFAULT_TIMEOUT_MS,
    .tls_config = {
        .cert_pem = dot_ca_pem,        // 服务器 CA（PEM 字符串）
        // 或用证书包：
        // .crt_bundle_attach = esp_crt_bundle_attach,
    },
};
esp_dns_handle_t handle = esp_dns_init_dot(&cfg);
assert(handle);
// ... 使用 handle 进行 DNS 解析 ...
esp_dns_cleanup_dot(handle);
```

### 3. 初始化 DoH（DNS over HTTPS）

```c
esp_dns_config_t cfg = {
    .protocol = ESP_DNS_PROTOCOL_DOH,
    .dns_server = "dns.google",
    .port = ESP_DNS_DEFAULT_DOH_PORT,   // 443
    .timeout_ms = ESP_DNS_DEFAULT_TIMEOUT_MS,
    .tls_config = {
        .crt_bundle_attach = esp_crt_bundle_attach,
    },
    .protocol_config.doh_config.url_path = "/dns-query",
};
esp_dns_handle_t handle = esp_dns_init_doh(&cfg);
assert(handle);
// ... 解析 ...
esp_dns_cleanup_doh(handle);
```

### 4. 初始化 TCP / UDP DNS

```c
// TCP DNS（端口 53）
esp_dns_config_t cfg = {
    .protocol = ESP_DNS_PROTOCOL_TCP,
    .dns_server = "8.8.8.8",
    .port = ESP_DNS_DEFAULT_TCP_PORT,
    .timeout_ms = ESP_DNS_DEFAULT_TIMEOUT_MS,
};
esp_dns_handle_t h = esp_dns_init_tcp(&cfg);
// ...
esp_dns_cleanup_tcp(h);

// UDP DNS
esp_dns_handle_t hu = esp_dns_init_udp(&cfg);   // .protocol = ESP_DNS_PROTOCOL_UDP
esp_dns_cleanup_udp(hu);
```

### 5. 配置结构体字段速查（`esp_dns_config_t`）

| 字段 | 说明 |
|---|---|
| `protocol` | `esp_dns_protocol_type_t` |
| `dns_server` | DNS 服务器 IP 或主机名 |
| `port` | 自定义端口（不用默认时） |
| `timeout_ms` | 查询超时 |
| `tls_config.cert_pem` | PEM 证书（DoT/DoH） |
| `tls_config.crt_bundle_attach` | 证书包 attach 函数（DoT/DoH） |
| `protocol_config.doh_config.url_path` | DoH 的 URL 路径（如 `/dns-query`） |

> 清理函数命名规则：`esp_dns_cleanup_<protocol>`（`doh`/`dot`/`tcp`/`udp`），与 init 一一对应。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| DoT/DoH 握手失败 | 无 CA 证书或证书包 | 设置 `tls_config.cert_pem` 或 `crt_bundle_attach` |
| `esp_dns_init_*` 返回 NULL | 参数非法 / 内存不足 | 校验 `dns_server` 非空、protocol 与函数匹配 |
| 用错 cleanup 函数 | init/cleanup 协议不匹配 | `esp_dns_init_doh` → `esp_dns_cleanup_doh`，依此类推 |
| DoH 解析失败 | url_path 错误 | 服务器要求 `/dns-query` 等特定路径 |
| 端口被防火墙挡 | 用了非标准端口 | 确认网络放通 853/443/53 |

## 参考

- `components/esp_dns/include/esp_dns.h` — `esp_dns_config_t`、`esp_dns_init_doh/dot/tcp/udp` 与对应 cleanup
- `components/esp_dns/README.md` — 组件简介
