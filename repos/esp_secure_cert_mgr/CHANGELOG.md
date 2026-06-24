# Changelog

## [1.0.0] - 2026-06-18

初始版本。面向 ESP-IDF `esp_secure_cert_mgr` 组件的 AI Skill，全部内容基于仓库 `D:/esp-skill/espressif-repos/esp_secure_cert_mgr/` 的真实文档与代码。

### Added
- `SKILL.md`：核心原则（12 条）、When to Use、10 个 recipe 索引、分区格式/TLV 类型/partitions.csv 参考表、13 条 Critical Pitfalls（含 WRONG/CORRECT 代码）、执行工作流、失败策略。
- `AGENTS.md`：工程约定（语言/目标/工具链）、组件引入方式、头文件包含顺序、标准项目结构、`app_main` 初始化模式、构建流程、codegen 清单、Do Not Modify 说明。
- 10 个场景配方（`recipes/`）：
  - `read_certs_and_key.md` — 读取设备/CA 证书与私钥并正确释放
  - `use_ds_peripheral.md` — DS 外设获取 `esp_ds_data_ctx_t` 与验签
  - `use_ecdsa_peripheral.md` — ECDSA 外设（eFuse 私钥）签名
  - `iterate_tlv_entries.md` — TLV 通用查询/迭代/列举
  - `write_user_data.md` — 运行时写自定义 TLV（flash 模式）
  - `hmac_ecdsa_derivation.md` — HMAC 派生 ECDSA 私钥（不落盘）
  - `generate_partition_in_buffer.md` — 主机端 buffer 模式生成镜像
  - `verify_partition.md` — 完整性 + 签名校验
  - `ota_update_partition.md` — fail-safe 分区 OTA（三种暂存 + NVS 恢复）
  - `generate_partition_csv.md` — `configure_esp_secure_cert.py` 生成/签名/解析
- 4 个资源文档（`resources/`）：
  - `api_reference.md` — 按模块分组的真实 API/结构体/枚举/错误码
  - `config_reference.md` — Kconfig、SoC 能力、partitions.csv、派生常量
  - `pitfalls.md` — 按主题归类的陷阱汇总
  - `example_list.md` — 仓库 `examples/` / `test_apps/` 真实路径索引
- `README.md`：中文介绍、特性、安装、目录结构、数据来源。
- `CHANGELOG.md`：本文件。

### Grounding
- 所有函数名、结构体、宏、Kconfig 符号、文件路径、代码片段均取自仓库 `include/`、`private_include/`、`docs/`、`examples/`、`tools/`；未在仓库中找到的内容一律省略，未做任何臆造。
