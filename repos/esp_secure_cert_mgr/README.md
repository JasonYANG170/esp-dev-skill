# esp_secure_cert_mgr-skill

面向 Espressif **esp_secure_cert_mgr** 组件的 Claude Code / Agent Skill。该组件为出厂预置（pre-provisioning）的 ESP 模块提供对 `esp_secure_cert` 分区中 PKI 凭证（设备证书、CA 证书、私钥、DS 上下文、HMAC 派生 ECDSA 密钥、自定义 TLV 数据）的统一读取、写入与校验接口，并支持分区签名校验、完整性校验与 fail-safe OTA。

本 Skill 让 AI Agent 能够**完全基于仓库真实文档与代码**（`include/`、`docs/`、`examples/`、`tools/`）正确地进行固件开发，杜绝臆造 API。

## 功能特性

- **场景化配方（10 个 recipes）**：读取证书/密钥、DS 外设、ECDSA 外设、TLV 遍历、写自定义数据、HMAC 派生密钥、buffer 模式生成镜像、完整性/签名校验、fail-safe OTA、`configure_esp_secure_cert.py` 工具用法。
- **完整 API 参考**：按模块分组的真实函数签名、结构体、枚举、错误码（源自 `include/*.h`）。
- **配置参考**：Kconfig 选项、SoC 能力依赖、`partitions.csv` 各格式、派生常量。
- **陷阱汇总**：内存/指针、编译期可见性、写入流程、HMAC/ECDSA、校验、分区表等高频坑点。
- **示例索引**：仓库 `examples/`、`test_apps/` 真实路径与说明。

## 支持范围

- 目标：ESP32 系列芯片（ESP32 / S2 / S3 / C2/C3/C5/C6 / H2 / P4 等）。
- 框架：ESP-IDF（`idf >= 4.3`；写入类 API 需 `>= 5.3`）。
- 能力：DS（`SOC_DIG_SIGN_SUPPORTED`）、HMAC（`SOC_HMAC_SUPPORTED`）、ECDSA（`SOC_ECDSA_SUPPORTED`）按芯片启用。
- 构建：`idf.py` + CMake。

## 安装

将该 Skill 克隆/复制到 Claude Code 的 skills 目录：

- 项目级：`<project>/.claude/skills/esp_secure_cert_mgr-skill/`
- 用户级：`~/.claude/skills/esp_secure_cert_mgr-skill/`

```bash
git clone <this-repo> /path/to/esp_secure_cert_mgr-skill
# 或直接复制本目录
```

随后在对话中提及预置证书、`esp_secure_cert`、DS 外设、分区 OTA 等关键词即可触发。

## 目录结构

```
esp_secure_cert_mgr-skill/
├── SKILL.md                  # 核心规则、配方索引、陷阱、执行工作流
├── AGENTS.md                 # 工程约定、包含顺序、构建流程、codegen 清单
├── recipes/                  # 10 个场景配方
│   ├── read_certs_and_key.md
│   ├── use_ds_peripheral.md
│   ├── use_ecdsa_peripheral.md
│   ├── iterate_tlv_entries.md
│   ├── write_user_data.md
│   ├── hmac_ecdsa_derivation.md
│   ├── generate_partition_in_buffer.md
│   ├── verify_partition.md
│   ├── ota_update_partition.md
│   └── generate_partition_csv.md
├── resources/                # 快速参考文档
│   ├── api_reference.md
│   ├── config_reference.md
│   ├── pitfalls.md
│   └── example_list.md
├── README.md
└── CHANGELOG.md
```

## 数据来源

全部内容来自 `D:/esp-skill/espressif-repos/esp_secure_cert_mgr/`：
- `README.md`、`docs/format.md`、`docs/write_support.md`、`docs/esp_secure_cert_tools/*.md`
- `include/*.h`（API 真相源）、`private_include/*.h`（结构/常量）
- `Kconfig`、`idf_component.yml`
- `examples/esp_secure_cert_app/`、`examples/esp_secure_cert_ota_example/`
- `tools/README.md`、`tools/configure_esp_secure_cert.py`

> 仓库源码为只读引用，请勿修改。
