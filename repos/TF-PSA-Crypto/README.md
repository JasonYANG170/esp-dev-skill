# TF-PSA-Crypto-skill

针对 **TF-PSA-Crypto**（PSA Cryptography API v1.2 的参考实现，库版本 1.0.0）的 AI 编程技能（Claude Code / Agent Skill）。本技能帮助 AI 代理基于仓库自身的头文件、示例与文档，正确地编写、修改与调试使用 `psa_*` 加密 API 的应用代码——**绝不臆造 API**。

## 这是什么

TF-PSA-Crypto 是 TrustedFirmware 旗下的密码学库，实现了 Arm PSA Cryptography API（`psa/crypto.h` 中的 `psa_*` 函数），覆盖哈希、MAC、对称/AEAD 加密、非对称签名、密钥协商、密钥派生与密钥管理。它是 Mbed TLS 加密能力的继任者。

本技能将仓库的真实知识（API 签名、宏定义、配置开关、示例代码）浓缩为：

- **场景化 recipe**（哈希、MAC、对称加密、AEAD、非对称、密钥派生、密钥管理、构建配置、旧 mbedtls_* 迁移、PSA 驱动开发）
- **完整 API 速查**（按模块分组的真实函数签名）
- **配置参考**（`crypto_config.h` 中全部 `PSA_WANT_*` 开关）
- **常见陷阱合集**（含错误码诊断表）

## 功能特性

- 中文场景说明 + 可直接复制、源自 `programs/psa/*.c` 的真实代码
- 所有函数名/结构体/宏/配置项均来自仓库 `include/psa/` 与 `core/`
- 涵盖 PSA 全部主要操作类：hash / MAC / cipher / AEAD / sign / key agreement / KDF
- 密钥生命周期（volatile / persistent）、属性对象、usage flag 的正确用法
- CMake 构建模式、`configs/` 预设、`TF_PSA_CRYPTO_CONFIG_FILE` 外部配置
- `PSA_ERROR_*` 错误码到根因的快速诊断表

## 安装

将本技能目录放入 Claude Code 的 skills 路径之一：

- **项目级**：`<项目根>/.claude/skills/TF-PSA-Crypto-skill/`
- **用户级**：`~/.claude/skills/TF-PSA-Crypto-skill/`

例如（项目级）：

```bash
mkdir -p .claude/skills
cp -r TF-PSA-Crypto-skill .claude/skills/
```

之后 Claude Code 会自动加载本技能。触发时提及 "PSA Crypto"、"TF-PSA-Crypto"、"psa/crypto.h"、"psa_sign"、"psa_aead"、"crypto_config.h" 等关键词即可。

## 目录结构

```
TF-PSA-Crypto-skill/
├── SKILL.md                    # 核心规则、原则、recipe 索引、陷阱、执行流程
├── AGENTS.md                   # 补充约定（头文件、main 模式、构建、代码检查清单）
├── recipes/                    # 场景化 recipe（含真实代码与错误表）
│   ├── init_and_lifecycle.md
│   ├── key_management.md
│   ├── hashing.md
│   ├── mac.md
│   ├── cipher.md
│   ├── aead.md
│   ├── asymmetric.md
│   ├── key_derivation.md
│   ├── build_and_config.md
│   ├── migration_to_psa.md
│   └── psa_driver_development.md
├── resources/                  # 速查文档（均基于仓库真实内容）
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md                   # 本文件
└── CHANGELOG.md
```

## 支持范围

- **库**：TF-PSA-Crypto 1.0.0（PSA Crypto API 1.2）
- **平台**：可移植 C99（8 位字节、二补码、`int`/`size_t` ≥ 32 位）
- **工具链**：CMake 3.20.2+；GCC 5.4+ / Clang 3.8+ / Arm Compiler 6.21+ / MSVC 2019+；Python 3.8+（生成源码/测试）
- **覆盖操作**：哈希、MAC（HMAC/CMAC）、对称加密、AEAD、非对称签名（ECDSA/RSA/EdDSA）、密钥协商（ECDH/FFDH）、密钥派生（HKDF/PBKDF2/TLS12-PRF）、密钥管理、旧 `mbedtls_*` → PSA 迁移、PSA cryptoprocessor 驱动开发与 driver-only 构建

不在范围内：TLS/X.509 协议层设计、底层 RNG 熵源的板级移植、用旧 `mbedtls_*` 加密 API 作为终点（应迁移到 PSA）。

## 许可证

本技能文档按原样提供。TF-PSA-Crypto 库本身采用 Apache-2.0 OR GPL-2.0-or-later 双许可（见仓库 `LICENSE`）。
