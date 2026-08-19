---
name: esp-secure-cert-mgr-skill
description: >-
  AI Skill for developing ESP-IDF firmware with the Espressif esp_secure_cert_mgr component,
  used to read, write, and verify PKI credentials (device cert, CA cert, private key, DS context)
  stored in the esp_secure_cert partition of pre-provisioned ESP modules.
  Trigger words: "esp_secure_cert", "esp_secure_cert_mgr", "secure cert", "pre-provisioning",
  "预置证书", "安全证书", "数字签名 DS", "Digital Signature", "PKI", "TLS 凭据", "esp_secure_cert partition",
  "ECDSA peripheral", "HMAC ECDSA", "secure boot v2"
license: Apache-2.0
metadata:
  author: Community
  version: "1.0.0"
---

# esp_secure_cert_mgr-skill

面向 Espressif `esp_secure_cert_mgr` 组件的 AI Skill。该组件为出厂预置（pre-provisioning）的 ESP 模块提供对 `esp_secure_cert` 分区中 PKI 凭据（设备证书、CA 证书、私钥、DS 上下文）的统一读取、写入与校验接口，分区默认采用 TLV 格式。本 Skill 提供场景化 recipes、完整 API 参考、配置项参考、常见坑点与真实示例索引，全部内容均来自仓库的真实文档与代码。

## Core Principles

1. **绝不臆造 API** — 所有函数、结构体、宏、配置项必须能在 `resources/` 或仓库 `include/` / `docs/` 中找到；找不到就省略，不要发明。
2. **分区默认 TLV 格式** — `esp_secure_cert` 分区使用 TLV 格式（`cust_flash_tlv`）。legacy 格式（`cust_flash`、`nvs`）需通过 `CONFIG_ESP_SECURE_CERT_SUPPORT_LEGACY_FORMATS` 显式启用。
3. **读取 API 必须配对 free** — `esp_secure_cert_get_device_cert()` / `get_ca_cert()` / `get_priv_key()` 返回的指针，NVS 或 HMAC 加密场景会动态分配内存，必须用对应 `esp_secure_cert_free_*()` 释放。
4. **`get_*` 系列 API 仅返回 subtype 0 的条目** — 若分区中同一类型有多条目（不同 subtype），应改用通用 API `esp_secure_cert_get_tlv_info()` 并指定 type + subtype。
5. **DS 外设与裸私钥互斥编译** — 启用 `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL`（依赖 `SOC_DIG_SIGN_SUPPORTED`）时使用 `esp_secure_cert_get_ds_ctx()`；未启用时使用 `esp_secure_cert_get_priv_key()`。头文件用 `#ifdef` 自动切换。
6. **写入 API 要求 IDF >= 5.3** — `esp_secure_cert_write.h` 通过 `ESP_SECURE_CERT_WRITE_SUPPORT` 宏控制可用性；IDF < 5.3 编译时这些函数不可用。
7. **写之前必须先擦除** — 闪存只能从 1 写到 0。`esp_secure_cert_append_tlv()` 默认 `check_erase=true`，未擦除会返回 `ESP_ERR_SECURE_CERT_FLASH_NOT_ERASED`。
8. **TLV 数据 16 字节对齐** — 写入时数据自动填充到 16 字节倍数以满足 Flash Encryption 写对齐要求（`MIN_ALIGNMENT_REQUIRED`）。
9. **`unmap` 后所有指针失效** — `esp_secure_cert_unmap_partition()` / `esp_secure_cert_tlv_set_partition()` 会解除分区映射，之前 read API 返回的指针立即失效，需重新调用。
10. **`esp_secure_cert_tlv_set_partition()` 切换活动分区** — 传 `NULL` 恢复默认主分区；OTA 暂存场景用于切到 staging 分区进行验证。
11. **HMAC 派生 ECDSA 私钥从不落盘** — 使用 `esp_secure_cert_append_tlv_with_hmac_ecdsa_derivation()` 仅写入 salt，私钥读取时通过 PBKDF2-HMAC-SHA256（2048 次迭代）从 eFuse HMAC key + salt 实时派生。需 `SOC_HMAC_SUPPORTED` 与 eFuse 中 `HMAC_UP` 用途的密钥。
12. **签名校验依赖 Secure Boot V2** — `CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION` 依赖 `SECURE_BOOT_V2_RSA_ENABLED` 或 `SECURE_BOOT_V2_ECDSA_ENABLED`，应在启动时调用 `esp_secure_cert_verify_partition_signature(NULL)`。

## When to Use

**Applicable:**
- 在固件中读取预置的设备证书 / CA 证书 / 私钥用于 TLS 连接
- 使用 DS（Digital Signature）外设保护 RSA 私钥（ESP32-S2/S3/C3/C5）
- 使用 ECDSA 外设（eFuse 私钥）签名
- 使用 HMAC 派生 ECDSA 私钥（密钥不落盘）
- 运行时写自定义 TLV 用户数据 / 运行时 provisioning
- 主机端在内存缓冲区中生成 `esp_secure_cert` 分区镜像
- 对 `esp_secure_cert` 分区做完整性校验（SHA256）或签名校验（Secure Boot V2）
- 对 `esp_secure_cert` 分区做 OTA 升级（unallocated space / passive OTA / direct）
- 配置 `partitions.csv`、生成与烧录 `esp_secure_cert` 分区

**Not applicable:**
- 非 ESP-IDF 平台（该组件依赖 ESP-IDF partition / eFuse / mbedTLS / PSA）
- 通用 TLS / mbedTLS 用法问题（不涉及 `esp_secure_cert` 分区）
- 生产 eFuse 烧录策略设计本身（仅涉及读写 `esp_secure_cert` 分区所需的那部分）

---

## Scenario Quick Reference (Recipes)

当用户意图匹配下列场景时，**先读对应 recipe**，里面包含完整调用链、分步说明、常见错误与真实代码。

### 读取凭据

| recipe | scenario |
|---|---|
| `recipes/read_certs_and_key.md` | 读取设备证书 / CA 证书 / 私钥（非 DS 场景）并释放内存 |
| `recipes/use_ds_peripheral.md` | 使用 Digital Signature (DS) 外设获取 `esp_ds_data_ctx_t` 并验证密文 |
| `recipes/use_ecdsa_peripheral.md` | 使用 ECDSA 外设（eFuse 私钥）签名，配合 mbedTLS/PSA |
| `recipes/iterate_tlv_entries.md` | 遍历/列出/通用查询任意 TLV 条目（`esp_secure_cert_get_tlv_info` / `list_tlv_entries`） |

### 写入与 Provisioning

| recipe | scenario |
|---|---|
| `recipes/write_user_data.md` | 运行时写入自定义 TLV 用户数据（flash 模式） |
| `recipes/hmac_ecdsa_derivation.md` | 写入 HMAC 派生 ECDSA 私钥配置（密钥不落盘） |
| `recipes/generate_partition_in_buffer.md` | 主机端在内存缓冲区中生成 `esp_secure_cert` 镜像 |

### 校验与 OTA

| recipe | scenario |
|---|---|
| `recipes/verify_partition.md` | 启动时校验分区完整性（SHA256）与签名（Secure Boot V2） |
| `recipes/ota_update_partition.md` | 对 `esp_secure_cert` 分区做 OTA 升级（staging / passive / direct） |
| `recipes/generate_partition_csv.md` | 用 CSV 与 `configure_esp_secure_cert.py` 生成并烧录分区 |

---

## esp_secure_cert Partition Format Reference

### TLV 条目布局（来自 `docs/write_support.md` 与 `esp_secure_cert_tlv_private.h`）

```
┌────────────────────────────────────────────────���────────────┐
│                    TLV Header (12 bytes)                    │
├─────────┬────────┬────────┬──────────┬─────────┬───────────┤
│  Magic  │ Flags  │  Type  │ Subtype  │ Length  │ Reserved  │
│ 4 bytes │ 1 byte │ 1 byte │  1 byte  │ 2 bytes │  3 bytes  │
├─────────┴────────┴────────┴──────────┴─────────┴───────────┤
│                     Data (variable)                         │
├─────────────────────────────────────��───────────────────────┤
│                 Padding (16-byte aligned)                   │
├─────────────────────────────────────────────────────────────┤
│                   TLV Footer (4 bytes)                      │
│                       CRC32                                 │
└─────────────────────────────────────────────────────────────┘
```

- Magic = `0xBA5EBA11`（`ESP_SECURE_CERT_TLV_MAGIC`）
- Header 结构 `esp_secure_cert_tlv_header_t`（packed，sizeof == 12），Footer `esp_secure_cert_tlv_footer_t`（sizeof == 4）
- 生成分区时会在末尾自动追加 `ESP_SECURE_CERT_INTEGRITY_TLV`（SHA256）

### partitions.csv 配置（来自 `docs/format.md`）

| 格式 | partitions.csv 行 | 分区大小 |
|---|---|---|
| TLV（默认 `cust_flash_tlv`） | `esp_secure_cert, 0x3F, , 0xD000, 0x2000, encrypted` | 8 KiB |
| legacy `cust_flash` | `esp_secure_cert, 0x3F, , 0xD000, 0x6000, encrypted` | 24 KiB |
| legacy `nvs` | `esp_secure_cert, data, nvs, 0xD000, 0x6000,` | 24 KiB |

> 自定义分区类型 `0x3F`，分区名常量 `ESP_SECURE_CERT_TLV_PARTITION_NAME = "esp_secure_cert"`，类型常量 `ESP_SECURE_CERT_TLV_PARTITION_TYPE = 0x3F`。

### TLV Type 枚举（来自 `esp_secure_cert_tlv_config.h`）

| 枚举 | 值 | 说明 |
|---|---|---|
| `ESP_SECURE_CERT_CA_CERT_TLV` | 0 | CA 证书 |
| `ESP_SECURE_CERT_DEV_CERT_TLV` | 1 | 设备证书 |
| `ESP_SECURE_CERT_PRIV_KEY_TLV` | 2 | 私钥 |
| `ESP_SECURE_CERT_DS_DATA_TLV` | 3 | DS 数据 |
| `ESP_SECURE_CERT_DS_CONTEXT_TLV` | 4 | DS 上下文 |
| `ESP_SECURE_CERT_HMAC_ECDSA_KEY_SALT` | 5 | HMAC-ECDSA 派生用的 salt |
| `ESP_SECURE_CERT_TLV_SEC_CFG` | 6 | 安全配置（含 eFuse block id） |
| `ESP_SECURE_CERT_TEE_SEC_STORAGE_KEY_TLV` | 7 | TEE 安全存储密钥 |
| `ESP_SECURE_CERT_SIGNATURE_BLOCK_TLV` | 8 | 签名块（Secure Boot V2） |
| `ESP_SECURE_CERT_INTEGRITY_TLV` | 9 | 完整性 SHA256 |
| `ESP_SECURE_CERT_TLV_END` | 50 | 末尾标记（用于取结束地址） |
| `ESP_SECURE_CERT_USER_DATA_1` … `USER_DATA_5` | 51 … 55 | 用户自定义数据槽 |
| `ESP_SECURE_CERT_MATTER_TLV_1` / `_2` | 201 / 202 | Matter 项目专用（不要用于应用） |
| `ESP_SECURE_CERT_TLV_MAX` / `_INVALID` | 254 / 255 | 边界 |

> Subtype 枚举 `ESP_SECURE_CERT_SUBTYPE_0` … `ESP_SECURE_CERT_SUBTYPE_16`（0–16），`ESP_SECURE_CERT_SUBTYPE_MAX=254`。

### 私钥类型枚举（来自 `esp_secure_cert_read.h`）

| `esp_secure_cert_key_type_t` | 说明 |
|---|---|
| `ESP_SECURE_CERT_INVALID_KEY` | 非法 |
| `ESP_SECURE_CERT_DEFAULT_FORMAT_KEY` | 默认格式 |
| `ESP_SECURE_CERT_HMAC_ENCRYPTED_KEY` | HMAC 加密 |
| `ESP_SECURE_CERT_HMAC_DERIVED_ECDSA_KEY` | HMAC 派生 ECDSA |
| `ESP_SECURE_CERT_ECDSA_PERIPHERAL_KEY` | ECDSA 外设（eFuse） |

### TLV Flags（来自 `esp_secure_cert_tlv_private.h`）

| 宏 | 值 | 含义 |
|---|---|---|
| `ESP_SECURE_CERT_TLV_FLAG_HMAC_ENCRYPTION` | `2 << 6` | 需用 HMAC-AES-GCM 解密 |
| `ESP_SECURE_CERT_TLV_FLAG_HMAC_ECDSA_KEY_DERIVATION` | `1 << 6` | 启用 HMAC 派生 ECDSA 私钥 |
| `ESP_SECURE_CERT_TLV_FLAG_KEY_ECDSA_PERIPHERAL` | `1 << 3` | ECDSA 密钥存于 eFuse |

---

## Critical Pitfalls (Must Read)

以下是最常见的错误，违反任一条都会导致固件不工作或数据损坏。

### 1. 读到的指针不释放（NVS / HMAC 场景泄漏内存）

```c
// ❌ WRONG — NVS 或 HMAC 加密场景下 get_* 会 malloc，从不 free
char *cert = NULL; uint32_t len = 0;
esp_secure_cert_get_device_cert(&cert, &len);
// 用完直接丢弃 cert → 内存泄漏

// ✅ CORRECT — 用对应的 free API（仅当确实分配了内存才会真正释放）
char *cert = NULL; uint32_t len = 0;
if (esp_secure_cert_get_device_cert(&cert, &len) == ESP_OK) {
    // 使用 cert ...
    esp_secure_cert_free_device_cert(cert);
}
```

### 2. 同类型多条目却用便捷 API（只拿到 subtype 0）

```c
// ❌ WRONG — 分区里 CA 证书有 subtype 0/1，只读到第一条
char *ca = NULL; uint32_t len = 0;
esp_secure_cert_get_ca_cert(&ca, &len);  // 永远只返回 subtype 0

// ✅ CORRECT — 用通用 API 指定 type + subtype
esp_secure_cert_tlv_config_t cfg = {
    .type = ESP_SECURE_CERT_CA_CERT_TLV,
    .subtype = ESP_SECURE_CERT_SUBTYPE_1,
};
esp_secure_cert_tlv_info_t info = {};
if (esp_secure_cert_get_tlv_info(&cfg, &info) == ESP_OK) {
    // 使用 info.data / info.length ...
    esp_secure_cert_free_tlv_info(&info);
}
```

### 3. 启用 DS 却调用了 get_priv_key（API 编译期互斥）

```c
// ❌ WRONG — CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL=y 时 get_priv_key 不存在
char *key = NULL; uint32_t len = 0;
esp_secure_cert_get_priv_key(&key, &len);  // 链接错误

// ✅ CORRECT — DS 场景用 get_ds_ctx，并配对 free_ds_ctx
esp_ds_data_ctx_t *ds = esp_secure_cert_get_ds_ctx();
if (ds) {
    // 使用 ds->esp_ds_data, ds->rsa_length_bits, ds->efuse_key_id ...
    esp_secure_cert_free_ds_ctx(ds);
}
```

### 4. 写入前未擦除（默认 check_erase 为 true）

```c
// ❌ WRONG — 直接 append，旧数据未擦 → ESP_ERR_SECURE_CERT_FLASH_NOT_ERASED
esp_secure_cert_tlv_info_t tlv = { .type = ESP_SECURE_CERT_USER_DATA_1,
                                   .subtype = ESP_SECURE_CERT_SUBTYPE_0,
                                   .data = buf, .length = n, .flags = 0 };
esp_secure_cert_append_tlv(&tlv, NULL);  // 返回 FLASH_NOT_ERASED

// ✅ CORRECT — 先擦除（生产设备慎用，会清空所有凭据）
esp_secure_cert_erase_partition();
esp_secure_cert_append_tlv(&tlv, NULL);
```

### 5. IDF 版本低于 5.3 却调用写入 API

```c
// ❌ WRONG — IDF < 5.3 时 ESP_SECURE_CERT_WRITE_SUPPORT 未定义，链接不到符号
esp_secure_cert_append_tlv(&tlv, NULL);

// ✅ CORRECT — 用宏守卫，或升级 IDF >= 5.3
#ifdef ESP_SECURE_CERT_WRITE_SUPPORT
    esp_secure_cert_append_tlv(&tlv, NULL);
#else
    ESP_LOGE(TAG, "write APIs need ESP-IDF >= 5.3");
#endif
```

### 6. 误用已释放指针 / unmap 后悬空

```c
// ❌ WRONG — unmap 后继续用之前 get 返回的 flash 指针
esp_secure_cert_get_device_cert(&cert, &len);
esp_secure_cert_unmap_partition();
printf("%s", cert);  // 悬空，行为未定义

// ✅ CORRECT — unmap 前把数据拷走，或 unmap 后重新 get
char *cert = NULL; uint32_t len = 0;
esp_secure_cert_get_device_cert(&cert, &len);
char *copy = malloc(len); memcpy(copy, cert, len);
esp_secure_cert_free_device_cert(cert);
esp_secure_cert_unmap_partition();
// 之后用 copy
```

### 7. 切换活动分区后未复位 NULL（OTA 校验完仍读 staging）

```c
// ❌ WRONG — 验证完 staging 后忘记切回主分区
esp_secure_cert_tlv_set_partition(staging);
esp_secure_cert_verify_partition_integrity();
// 后续读取仍走 staging，主分区没更新

// ✅ CORRECT — 校验/复制完成后显式复位
esp_secure_cert_tlv_set_partition(staging);
esp_secure_cert_verify_partition_integrity();
// ... copy staging → primary ...
esp_secure_cert_tlv_set_partition(NULL);  // 切回主分区
```

### 8. 用 cust_flash/nvs 分区却按 TLV 调用

```c
// ❌ WRONG — legacy 格式未启用 SUPPORT_LEGACY_FORMATS，但分区是 nvs/cust_flash
// 头文件里 get_tlv_info / get_priv_key_type 在 legacy 下不可用

// ✅ CORRECT — legacy 分区需启用 CONFIG_ESP_SECURE_CERT_SUPPORT_LEGACY_FORMATS=y
// 并继续用 get_device_cert / get_ca_cert（NVS 自动动态分配）
```

### 9. partitions.csv 没加 encrypted 标志（开了 Flash Encryption 会读到密文）

```ini
# ❌ WRONG — 启用 Flash Encryption 后分区内容会被加密存储，未标记导致读出密文
esp_secure_cert, 0x3F, , 0xD000, 0x2000,

# ✅ CORRECT — TLV 格式加上 encrypted 标志
# Name, Type, SubType, Offset, Size, Flags
esp_secure_cert, 0x3F, , 0xD000, 0x2000, encrypted
```

### 10. ECDSA 外设签名未先取 efuse block id

```c
// ❌ WRONG — 直接 mbedtls_pk_parse_key 解析 eFuse 里的 ECDSA 私钥（根本读不到）
mbedtls_pk_parse_key(&pk, priv_key, priv_key_len, NULL, 0);

// ✅ CORRECT — 先判断 key_type，ECDSA_PERIPHERAL 时用 efuse_block 配 opaque key
esp_secure_cert_key_type_t kt;
esp_secure_cert_get_priv_key_type(&kt);
if (kt == ESP_SECURE_CERT_ECDSA_PERIPHERAL_KEY) {
    uint8_t efuse_blk;
    esp_secure_cert_get_priv_key_efuse_id(&efuse_blk);
    esp_ecdsa_pk_conf_t pk_conf = { .grp_id = MBEDTLS_ECP_DP_SECP256R1,
                                    .efuse_block = efuse_blk };
    esp_ecdsa_set_pk_context(&pk, &pk_conf);  // IDF < 6.0 / mbedTLS < 4.x 路径
}
```

### 11. HMAC 派生 ECDSA 误以为私钥在分区里

```c
// ❌ WRONG — 以为能从分区读出明文 ECDSA 私钥
esp_secure_cert_get_priv_key(&key, &len);  // 派生场景返回的是 121 字节 DER，且实时算出

// ✅ CORRECT — salt 落盘，私钥读取时由 PBKDF2-HMAC-SHA256(salt, hmac_key, 2048) 派生
// 写入侧：
uint8_t salt[32]; esp_fill_random(salt, sizeof(salt));
esp_secure_cert_erase_partition();
esp_secure_cert_append_tlv_with_hmac_ecdsa_derivation(
    salt, sizeof(salt), ESP_SECURE_CERT_SUBTYPE_0, NULL);
// 读取侧：esp_secure_cert_get_priv_key() 会自动派生 121 字节 DER 私钥
```

### 12. 签名校验未启用 Secure Boot 依赖

```c
// ❌ WRONG — 未开 CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION 就调用
esp_secure_cert_verify_partition_signature(NULL);  // 链接错误或语义无效

// ✅ CORRECT — Kconfig 依赖 SECURE_BOOT_V2_RSA_ENABLED 或 SECURE_BOOT_V2_ECDSA_ENABLED
// sdkconfig.defaults:
//   CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION=y
#if CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION
    if (esp_secure_cert_verify_partition_signature(NULL) != ESP_OK) {
        ESP_LOGE(TAG, "signature verification FAILED");
    }
#endif
```

### 13. 误用 Matter 专用 TLV 类型

```c
// ❌ WRONG — 用 201/202 存自己的数据
.type = ESP_SECURE_CERT_MATTER_TLV_1,  // 文档明确禁止应用使用

// ✅ CORRECT — 用 ESP_SECURE_CERT_USER_DATA_1..5（51..55）
.type = ESP_SECURE_CERT_USER_DATA_1,
```

---

## Execution Workflow

| Step | Name | Description |
|------|------|-------------|
| 1 | Plan | 明确需求：只读凭据？DS 外设？ECDSA 外设？HMAC 派生？运行时写？OTA 升级？校验签名？ |
| 2 | Recipe | 在 `recipes/` 中匹配场景，先读对应 recipe 获取完整调用链 |
| 3 | Query | recipe 未覆盖的 API 查 `resources/api_reference.md`；配置项查 `resources/config_reference.md` |
| 4 | Validate | 校验头文件包含、Kconfig 依赖（DS/HMAC/ECDSA/SecureVerification）、IDF 版本（写需 >= 5.3）、`partitions.csv` 格式与 encrypted 标志 |
| 5 | Confirm | 向用户确认：include 列表、分区表行、init/读取/释放顺序、写前的擦除与备份策略 |
| 6 | Execute | **新工程**：参照 `resources/example_list.md` 选最近示例（`examples/esp_secure_cert_app` 或 `esp_secure_cert_ota_example`）改造；**已有工程**：原地编辑 |
| 7 | Check | 核对：get/free 配对、DS vs 裸私钥的编译分支、unmap/set_partition 后指针失效、写前擦除 |
| 8 | Build | `idf.py set-target <chip>` → `idf.py build` |
| 9 | Provision | 用 `configure_esp_secure_cert.py`（CSV 或命令行）生成并烧录 `esp_secure_cert.bin` 到 0xD000 |
| 10 | Verify | 串口 `idf.py monitor` 看日志，签名/完整性校验 PASSED |

---

## Failure Strategies

| Situation | Action |
|---|---|
| API 在 `resources/` 找不到 | 立即停止，告知用户该 API 不存在 |
| `get_*` 返回 ESP_FAIL | 检查分区是否已生成并烧录、`partitions.csv` 是否正确、是否启用了 legacy 支持 |
| 写入返回 `FLASH_NOT_ERASED` | 先 `esp_secure_cert_erase_partition()`；生产设备务必先备份 |
| 写入返回 `TLV_ALREADY_EXISTS` | 同 type+subtype 已存在，换 subtype 或先擦除 |
| 写入返回 `HMAC_KEY_NOT_FOUND` | eFuse 未烧 `HMAC_UP` 用途密钥，先烧 eFuse |
| 写入符号链接失败 | IDF < 5.3，升级或用宏守卫 |
| DS 相关 API 不可见 | 检查 `SOC_DIG_SIGN_SUPPORTED` 与 `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL` |
| ECDSA 外设签名失败 | 确认 `SOC_ECDSA_SUPPORTED`、efuse block id 正确、mbedTLS/PSA 版本分支 |
| OTA 后读到旧数据 | 确认复制完成后 `esp_secure_cert_tlv_set_partition(NULL)` 复位 |

## References

- 场景 recipes → `recipes/` 目录
- 完整 API 参考 → `resources/api_reference.md`
- 配置项参考 → `resources/config_reference.md`
- 常见坑点汇总 → `resources/pitfalls.md`
- 示例工程索引 → `resources/example_list.md`
- 仓库真实文档：
  - `espressif-repos/esp_secure_cert_mgr/README.md`
  - `espressif-repos/esp_secure_cert_mgr/docs/format.md`
  - `espressif-repos/esp_secure_cert_mgr/docs/write_support.md`
  - `espressif-repos/esp_secure_cert_mgr/docs/esp_secure_cert_tools/secure_verification.md`
  - `espressif-repos/esp_secure_cert_mgr/docs/esp_secure_cert_tools/configure_esp_secure_cert_csv.md`
  - `espressif-repos/esp_secure_cert_mgr/tools/README.md`
