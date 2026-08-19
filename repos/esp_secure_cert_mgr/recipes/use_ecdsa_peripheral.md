# 使用 ECDSA 外设（eFuse 私钥）签名

> **适用摘要**: 当私钥类型为 `ESP_SECURE_CERT_ECDSA_PERIPHERAL_KEY`（私钥存于 eFuse block，由硬件 ECDSA 外设使用），演示如何判断类型、取 efuse block id，并用 mbedTLS/PSA 完成签名与验签。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp_secure_cert_mgr/resources/`, source/examples in `repos/esp_secure_cert_mgr/`, and this recipe path `repos/esp_secure_cert_mgr/recipes/use_ecdsa_peripheral.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "ECDSA 外设怎么签名"
- "私钥在 eFuse 里"
- "esp_secure_cert_get_priv_key_type"
- "esp_secure_cert_get_priv_key_efuse_id"
- "SECP256R1 签名"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | `SOC_ECDSA_SUPPORTED` |
| 私钥类型 | 分区中私钥 TLV flags 标记为 `ESP_SECURE_CERT_TLV_FLAG_KEY_ECDSA_PERIPHERAL` |
| 格式 | TLV（`get_priv_key_type` 仅 TLV 可用，未启用 `SUPPORT_LEGACY_FORMATS`） |
| 参考示例 | `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（`test_priv_key_validity`） |

## 分步说明

### 1. 判断私钥类型并取 efuse block id

```c
#include "esp_log.h"
#include "esp_secure_cert_read.h"
#include "esp_secure_cert_tlv_read.h"
#include "soc/soc_caps.h"

esp_secure_cert_key_type_t key_type = ESP_SECURE_CERT_DEFAULT_FORMAT_KEY;
#ifndef CONFIG_ESP_SECURE_CERT_SUPPORT_LEGACY_FORMATS
    if (esp_secure_cert_get_priv_key_type(&key_type) != ESP_OK) {
        ESP_LOGE(TAG, "Failed to obtain priv key type");
    }
#endif
```

> 枚举值（`esp_secure_cert_read.h`）：`ESP_SECURE_CERT_DEFAULT_FORMAT_KEY`、`ESP_SECURE_CERT_HMAC_ENCRYPTED_KEY`、`ESP_SECURE_CERT_HMAC_DERIVED_ECDSA_KEY`、`ESP_SECURE_CERT_ECDSA_PERIPHERAL_KEY`。

### 2. ECDSA 外设分支：取 efuse block 并设置 opaque key

IDF < 6.0（mbedTLS < 4.x）路径（来自示例）：

```c
#if SOC_ECDSA_SUPPORTED
#include "ecdsa/ecdsa_alt.h"   // esp_ecdsa_pk_conf_t / esp_ecdsa_set_pk_context

if (key_type == ESP_SECURE_CERT_ECDSA_PERIPHERAL_KEY) {
    uint8_t efuse_block_id;
    if (esp_secure_cert_get_priv_key_efuse_id(&efuse_block_id) != ESP_OK) {
        ESP_LOGE(TAG, "Failed to obtain efuse key id");
        return;
    }
    ESP_LOGI(TAG, "Using key from eFuse block %d", efuse_block_id);

    esp_ecdsa_pk_conf_t pk_conf = {
        .grp_id = MBEDTLS_ECP_DP_SECP256R1,
        .efuse_block = efuse_block_id,
    };
    if (esp_ecdsa_set_pk_context(&pk, &pk_conf) != 0) {
        ESP_LOGE(TAG, "Failed to set ECDSA context");
    }
}
#endif
```

IDF >= 6.0（mbedTLS 4.x / PSA）路径：

```c
#include "psa/crypto.h"
#include "psa_crypto_driver_esp_ecdsa.h"   // esp_ecdsa_opaque_key_t / ESP_ECDSA_CURVE_SECP256R1

psa_key_attributes_t key_attr = PSA_KEY_ATTRIBUTES_INIT;
psa_algorithm_t sign_alg =
#if CONFIG_MBEDTLS_ECDSA_DETERMINISTIC && SOC_ECDSA_SUPPORT_DETERMINISTIC_MODE
    PSA_ALG_DETERMINISTIC_ECDSA(PSA_ALG_SHA_256);
#else
    PSA_ALG_ECDSA(PSA_ALG_SHA_256);
#endif
psa_set_key_type(&key_attr, PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
psa_set_key_bits(&key_attr, 256);
psa_set_key_usage_flags(&key_attr, PSA_KEY_USAGE_SIGN_HASH);
psa_set_key_algorithm(&key_attr, sign_alg);
psa_set_key_lifetime(&key_attr, PSA_KEY_LIFETIME_ESP_ECDSA_VOLATILE);

esp_ecdsa_opaque_key_t opaque_key = {
    .curve = ESP_ECDSA_CURVE_SECP256R1,
    .efuse_block = efuse_block_id,
    .use_km_key = false,
};

psa_key_id_t priv_key_id = 0;
psa_import_key(&key_attr, (uint8_t *)&opaque_key, sizeof(opaque_key), &priv_key_id);
psa_reset_key_attributes(&key_attr);
```

### 3. 签名并用设备证书公钥验签

```c
static uint32_t hash[8] = {[0 ... 7] = 0xAABBCCDD};
unsigned char sig[1024]; size_t sig_len = 0;
psa_sign_hash(priv_key_id, sign_alg, (const unsigned char *)hash, sizeof(hash),
              sig, sizeof(sig), &sig_len);
psa_destroy_key(priv_key_id);

/* 用设备证书里的公钥验签（见示例 verify 段） */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `get_priv_key_type` 链接失败 | 启用了 legacy 格式（API 被 `#ifndef SUPPORT_LEGACY_FORMATS` 排除） | 关闭 legacy，或分区确实为 TLV |
| `esp_ecdsa_set_pk_context` 失败 | efuse block id 错误 / 芯片无 ECDSA | 核对 `SOC_ECDSA_SUPPORTED` 与 efuse |
| 编译报缺 `ecdsa_alt.h` | mbedTLS 版本不匹配 | 按 `MBEDTLS_VERSION_NUMBER` 分支切换头文件 |
| 拿不到私钥内容 | ECDSA 外设私钥永远不可读，只能签名 | 不要尝试 `get_priv_key`，直接走 opaque key |
| DER/格式错误 | 曲线与 eFuse key 不一致 | 统一用 SECP256R1（P-256） |

## 参考

- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_read.h`（`key_type_t` / `get_priv_key_type` / `get_priv_key_efuse_id`）
- `espressif-repos/esp_secure_cert_mgr/private_include/esp_secure_cert_tlv_private.h`（`ESP_SECURE_CERT_TLV_FLAG_KEY_ECDSA_PERIPHERAL`）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（`test_priv_key_validity`，含 IDF/PSA 分支）
