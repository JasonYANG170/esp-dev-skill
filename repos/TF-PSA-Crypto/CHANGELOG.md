# Changelog

所有显著变更记录于此。版本号遵循语义化版本（SemVer）。

## [1.1.0] - 2026-06-18

新增两个 recipe，填补 SKILL.md "When to Use" 与 "Not applicable" 中已声明但缺少分步指南的两个高频场景。

### 新增

- **recipes/migration_to_psa.md** — 把旧 `mbedtls_*` 密码 API（AES/SHA/HMAC/CMAC/RSA/ECDSA/ECDH/HKDF/PBKDF2）迁移到 `psa_*`。涵盖 per-algorithm 旧→新函数对照表、RNG 回调移除、`MBEDTLS_xxx_C` → `PSA_WANT_*` 配置拆分、错误码合并映射、PK 模块与 PAKE 接口变化、ECDSA raw vs ASN.1 格式。
  - 来源文档：`docs/psa-transition.md`（per-API 翻译总表，2406 行）、`docs/1.0-migration-guide.md`（1.0 破坏性变更）
  - 来源示例：`programs/psa/crypto_examples.c`、`programs/psa/key_ladder_demo.c`、`programs/psa/aead_demo.c`
- **recipes/psa_driver_development.md** — PSA cryptoprocessor 驱动开发与 driver-only 构建。涵盖 transparent（加速器）vs opaque（安全元件）驱动类型、JSON 驱动描述文件结构、自动生成 vs 手动集成的入口点表、手动集成 7 步流程（`DRIVER_PREFIX_ENABLED` / `PSA_CRYPTO_ACCELERATOR_DRIVER_PRESENT` / wrapper 编辑）、driver-only 构建的 `PSA_WANT_*` + `MBEDTLS_PSA_ACCEL_*` + `MBEDTLS_xxx_C` 三步配置。
  - 来源文档：`docs/psa-driver-example-and-guide.md`、`docs/driver-only-builds.md`、`docs/proposed/psa-driver-developer-guide.md`、`docs/proposed/psa-driver-integration-guide.md`
  - 来源驱动描述：`scripts/data_files/driver_jsons/esp_aes_driver.json`、`esp_sha_driver.json`、`mbedtls_test_transparent_driver.json`、`mbedtls_test_opaque_driver.json`、`p256_transparent_driver.json`
  - 来源示例：`programs/test/which_aes.c`（AES 实现诊断）
- **SKILL.md**：新增 "Migration & Driver Integration" 场景分组（含两 recipe 索引）；Step 7 示例选择策略补充迁移与驱动指引；"Not applicable" 中驱动条目改为指向新 recipe；metadata.version 1.0.0 → 1.1.0。
- **resources/api_reference.md**：新增第 13 节（迁移桥接函数：`mbedtls_psa_get_random`/`MBEDTLS_PSA_RANDOM_STATE`、`mbedtls_md_psa_alg_from_type`、`mbedtls_ecdsa_raw_to_der`/`der_to_raw`、PK↔PSA 桥 `mbedtls_pk_wrap_psa`/`can_do_psa`/`get_psa_attributes`/`import_into_psa`/`copy_from_psa`/`copy_public_from_psa`）与第 14 节（驱动接口：transparent/opaque 分发、自动生成入口点表、返回码、driver-only 配置三元组）。所有签名取自 `include/mbedtls/psa_util.h`、`include/mbedtls/pk.h`。
- **resources/example_list.md**：新增 "Driver description files" 分组（`scripts/data_files/driver_jsons/` 下 7 个文件，含 ESP AES/SHA 硬件加速驱动 JSON）；扩充 `programs/test/which_aes.c` 描述以关联驱动诊断。

### 约束（Grounding）

- 两 recipe 的每个函数名/结构体/宏/配置项/文件路径与代码片段均取自仓库 `docs/`、`include/`、`programs/`、`scripts/data_files/driver_jsons/`。
- 驱动接口明确标注为 work-in-progress（依据 `docs/proposed/psa-driver-developer-guide.md` 与 `docs/proposed/psa-driver-integration-guide.md` 顶部声明），recipe 仅描述当前已实现部分。

## [1.0.0] - 2026-06-18

首个发布版本。基于 TF-PSA-Crypto 1.0.0（PSA Cryptography API v1.2）仓库的真实头文件、示例程序与文档构建。

### 新增

- **SKILL.md**：核心原则（12 条）、When to Use、recipe 索引（9 类场景）、密钥类型/算法/usage flag/错误码/派生步骤速查表、关键陷阱（14 条，含错误/正确代码对比）、执行流程、失败策略。
- **AGENTS.md**：头文件包含约定、标准 `main()` 模板（含 `PSA_CHECK` + `goto exit`）、变量命名惯例、CMake 构建工作流、代码生成检查清单、Do-Not-Modify 说明。
- **recipes/**：9 个场景化 recipe，均含适用摘要、触发意图、前置条件表、源自 `programs/psa/*.c` 的真实代码、常见错误表、参考链接。
  - `init_and_lifecycle.md` — `psa_crypto_init` / `mbedtls_psa_crypto_free`
  - `key_management.md` — 生成/导入/导出/销毁、volatile vs persistent
  - `hashing.md` — `psa_hash_compute` 与分段 + clone
  - `mac.md` — HMAC / AES-CMAC 一次性与分段、sign/verify
  - `cipher.md` — AES-CBC/CTR/ChaCha20 一次性与分段、IV 管理
  - `aead.md` — GCM/CCM/ChaCha20-Poly1305、AD、短标签
  - `asymmetric.md` — ECDSA/RSA 签名验签、`psa_export_public_key`、ECDH
  - `key_derivation.md` — HKDF 密钥阶梯、`output_key`、TLS12-PRF、KDF+密钥协商
  - `build_and_config.md` — CMake 构建、`crypto_config.h`、`configs/` 预设、`find_package`
- **resources/**：4 份速查文档
  - `api_reference.md` — 全部 `psa_*` 公共函数签名（按模块分组）
  - `config_reference.md` — `crypto_config.h` 全部 `PSA_WANT_*` 开关（算法/密钥类型/曲线/DH 组）+ 预设 + CMake 选项
  - `pitfalls.md` — 合并版陷阱（19 条）+ 错误码快速诊断表
  - `example_list.md` — `programs/psa/`、`programs/test/`、`programs/fuzz/`、`configs/`、`docs/` 真实路径索引
- **README.md** / **CHANGELOG.md**：中文介绍与版本记录。

### 约束（Grounding）

- 所有函数名、结构体、宏、配置项、文件路径与代码片段均取自仓库 `include/psa/`、`include/tf-psa-crypto/`、`programs/psa/*.c`、`configs/`、`docs/` 与 `README.md`。
- 未在仓库中找到的内容一律省略，未做臆造。

### 已知薄点（Gaps）

- 驱动开发仅提供入口指引（指向 `docs/psa-driver-example-and-guide.md` 与 `docs/proposed/`），未展开完整驱动编写 recipe（仓库内驱动接口尚标记为 work-in-progress）。
- PAKE（EC-JPAKE）与 LMS 等扩展 API 未单独成 recipe（属扩展/不完整支持，仓库 README 指出 PAKE 仍有小范围非合规）。
- 部分高级 KDF（`*_custom` 变体、`verify_bytes`/`verify_key`）仅在 `api_reference.md` 列出签名，未做独立 recipe。
- 测试套件（`tests/suites/`）未纳入，因其面向库开发者而非应用开发者。
