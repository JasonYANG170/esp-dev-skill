# 写入 HMAC 派生 ECDSA 私钥（私钥不落盘）

> **适用摘要**: 利用硬件 HMAC 外设 + PBKDF2-HMAC-SHA256 实时派生 ECDSA 私钥。分区里只存 salt，私钥永不在 flash 中。演示写入配置与（可选）一步生成并烧录 eFuse 的完整流程。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp_secure_cert_mgr/resources/`, source/examples in `repos/esp_secure_cert_mgr/`, and this recipe path `repos/esp_secure_cert_mgr/recipes/hmac_ecdsa_derivation.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "HMAC 派生 ECDSA 私钥"
- "私钥不落盘"
- "PBKDF2 派生密钥"
- "esp_secure_cert_append_tlv_with_hmac_ecdsa_derivation"
- "esp_secure_cert_derive_hmac_ecdsa_key"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | `SOC_HMAC_SUPPORTED`（ESP32-C3 / S2 / S3 / C6 / H2） |
| eFuse | 已烧入 purpose=`ESP_EFUSE_KEY_PURPOSE_HMAC_UP` 的密钥 block（或用 `derive_hmac_ecdsa_key` 一次性生成并烧录） |
| IDF 版本 | >= 5.3（写入 API） |
| 头文件 | `esp_secure_cert_write.h`、`esp_secure_cert_crypto.h`、`esp_random.h` |
| 参考文档 | `espressif-repos/esp_secure_cert_mgr/docs/write_support.md`、`docs/format.md` |

## 分步说明

### 1. 仅写 salt（已有 HMAC_UP key）

```c
#include "esp_log.h"
#include "esp_random.h"
#include "esp_secure_cert_write.h"
#include "esp_secure_cert_tlv_config.h"

static const char *TAG = "hmac";

esp_err_t setup_derivation(void)
{
    /* 1) 生成随机 salt（P-256 通常 32 字节） */
    uint8_t salt[32];
    esp_fill_random(salt, sizeof(salt));

    /* 2) 先擦除分区 */
    esp_secure_cert_erase_partition();

    /* 3) 写入派生配置：写入 salt TLV + 私钥 marker（设 DERIVATION flag） */
    esp_err_t err = esp_secure_cert_append_tlv_with_hmac_ecdsa_derivation(
        salt, sizeof(salt),
        ESP_SECURE_CERT_SUBTYPE_0,
        NULL);                       // 默认 flash 模式
    if (err != ESP_OK) {
        ESP_LOGE(TAG, "derivation setup failed: %s", esp_err_to_name(err));
    }
    return err;
}
```

读取侧无需特殊处理：`esp_secure_cert_get_priv_key()` 检测到 DERIVATION flag 后，自动用 PBKDF2-HMAC-SHA256（salt + eFuse HMAC key，2048 轮）派生 32 字节原始密钥，并转换为 121 字节 DER（SECP256R1）返回。

派生参数（来自 `esp_secure_cert_tlv_private.h`）：
- `ESP_SECURE_CERT_KEY_DERIVATION_ITERATION_COUNT = 2048`
- `ESP_SECURE_CERT_DERIVED_ECDSA_KEY_SIZE = 32`
- `ESP_SECURE_CERT_ECDSA_DER_KEY_SIZE = 121`

### 2. 一步生成 + 烧录 eFuse（首次，eFuse 无 HMAC_UP key 时）

```c
uint8_t pub_key[65];                 // 未压缩公钥：0x04 || X || Y
size_t pub_key_len = sizeof(pub_key);

esp_err_t err = esp_secure_cert_derive_hmac_ecdsa_key(
    pub_key, &pub_key_len,
    ESP_SECURE_CERT_SUBTYPE_0,
    NULL);
if (err == ESP_OK) {
    /* HMAC key 已永久烧入 eFuse（不可改），salt 已写入分区 */
    /* pub_key 内为可用公钥，可交给服务器签发设备证书 */
}
```

`derive_hmac_ecdsa_key` 内部：生成随机 salt + 随机 HMAC key → 软件 PBKDF2 派生 ECDSA 私钥 → 校验 `1 < d < N` → 烧 eFuse（HMAC_UP）→ 硬件复算校验 → 写 salt TLV。

> 注意：若 eFuse 中已存在 HMAC_UP key，该函数返回 `ESP_ERR_SECURE_CERT_HMAC_KEY_ALREADY_EXISTS`；此时改用第 1 步的 `append_tlv_with_hmac_ecdsa_derivation`。

### 3. HMAC 加密写入（AES-GCM，区别于派生）

若只想加密存储（而非派生），用 `esp_secure_cert_append_tlv_with_hmac_encryption()`：用 HMAC_UP key 派生 AES-GCM 密钥加密 TLV 内容，读时自动解密。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `HMAC_KEY_NOT_FOUND` | eFuse 无 HMAC_UP key | 先烧 key，或用 `derive_hmac_ecdsa_key` |
| `HMAC_KEY_ALREADY_EXISTS` | eFuse 已有 HMAC_UP，又调 `derive_hmac_ecdsa_key` | 改用 `append_tlv_with_hmac_ecdsa_derivation` |
| `ECDSA_KEY_GEN_FAILED` | 派生出的 d 不满足 `1<d<N`（概率极低） | 重试（函数内部已重试） |
| `KEY_VERIFICATION_FAILED` | 硬件复算与软件不一致 | 检查 eFuse 状态 / HMAC 外设配置 |
| 读私钥失败 | salt TLV 缺失或 HMAC key 丢失 | 确认 salt 已写入、eFuse 未被清 |
| 非 HMAC 芯片编译报错 | 无 `SOC_HMAC_SUPPORTED` | 换支持 HMAC 的芯片 |

## 参考

- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_write.h`（`append_tlv_with_hmac_ecdsa_derivation` / `append_tlv_with_hmac_encryption` / `derive_hmac_ecdsa_key`）
- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_crypto.h`（`esp_pbkdf2_hmac_sha256` / `esp_secure_cert_validate_ecdsa_key` / `esp_secure_cert_calc_public_key`）
- `espressif-repos/esp_secure_cert_mgr/private_include/esp_secure_cert_tlv_private.h`（迭代数 / 派生尺寸常量）
- `espressif-repos/esp_secure_cert_mgr/docs/write_support.md`（HMAC-ECDSA 派生流程图）
- `espressif-repos/esp_secure_cert_mgr/docs/format.md`（"TLV storage algorithms"）
