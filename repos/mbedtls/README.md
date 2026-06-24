# mbedtls-skill

面向 **Mbed TLS（Espressif fork，4.x）** 的 AI Skill。为 AI 代理提供基于真实源码的 SSL/TLS/DTLS 与 X.509 开发能力：场景化配方、完整 API 参考、配置指南、常见陷阱与 3.x→4.x 迁移指引。

## 这是什么

本 Skill 让 AI 代理能够正确使用 Mbed TLS 4.x 开发加密通信软件。所有 API、配置宏、代码示例均取自真实仓库（`include/mbedtls/*.h`、`tf-psa-crypto/include/`、`programs/` 示例、`docs/`），不臆造接口。

Mbed TLS 是实现 X.509 证书操作与 TLS/DTLS 协议的 C 库，代码体积小，适合嵌入式系统。本仓库为 Espressif fork（分支 `mbedtls-4.1.0-idf`），API 与上游 4.x 一致。4.x 的核心变化：仓库拆分为 Mbed TLS（X.509 + TLS）与 TF-PSA-Crypto（加密）；RNG 统一由 PSA 子系统提供；仅支持 CMake 构建。

## 功能特性

- **TLS 客户端/服务器**：握手、读写、证书校验、SNI、会话缓存与票据
- **DTLS（UDP）**：客户端/服务器、定时器回调、HelloVerify cookie 防 DoS
- **X.509**：证书/CSR/CRL 的解析、校验、签发与生成
- **安全加固**：协议版本、加密套件、椭圆曲线、签名算法、ALPN、证书 profile
- **PSK**：明文 / opaque（PSA 密钥槽）/ 回调三种无证书 TLS 方式
- **3.x → 4.x 迁移**：移除手动 RNG、改 PSA 配置、API 改名对照
- **构建与裁剪**：CMake 构建、`mbedtls_config.h` 与 PSA `crypto_config.h`、`configs/` 预设、`config.py`

## 文件结构

```
mbedtls-skill/
├── SKILL.md              # 主技能文件：原则、配方索引、陷阱、执行流程
├── AGENTS.md             # 补充约定：头文件、骨架、清单
├── recipes/              # 场景配方（10 个）
│   ├── tls_client.md
│   ├── tls_server.md
│   ├── dtls_client.md
│   ├── dtls_server.md
│   ├── cert_parse_verify.md
│   ├── csr_generation.md
│   ├── cert_signing.md
│   ├── psk_tls.md
│   ├── session_resumption.md
│   ├── hardening_error.md
│   ├── build_config.md
│   └── psa_migration.md
├── resources/            # 快速参考文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md             # 本文件
└── CHANGELOG.md
```

## 安装

将本目录复制到 Claude Code 的 skills 目录即可：

```sh
# 项目级（仅当前项目可用）
cp -r mbedtls-skill /path/to/project/.claude/skills/

# 用户级（所有项目可用）
cp -r mbedtls-skill ~/.claude/skills/
```

复制后重启 Claude Code 会话，Skill 自动注册。AI 在遇到 TLS/DTLS/X.509/PSA/mbedTLS 相关请求时会自动加载。

## 支持范围

| 维度 | 说明 |
|---|---|
| 库版本 | Mbed TLS 4.x（分支 `mbedtls-4.1.0-idf`，Espressif fork） |
| 协议 | TLS 1.2、TLS 1.3、DTLS 1.2 |
| 平台 | POSIX 主机（Linux/macOS/Windows）、嵌入式（需移植 `platform.h` 与 PSA RNG） |
| 构建系统 | CMake ≥ 3.20.2（4.x 仅支持 CMake） |
| 不支持 | 上游 3.6 LTS 的特定行为、纯 TF-PSA-Crypto 加密 API 深度使用、其它 TLS 库 |

## 许可证

随 Mbed TLS 源码采用 Apache-2.0 OR GPL-2.0-or-later 双许可。本 Skill 文档由社区编写。
