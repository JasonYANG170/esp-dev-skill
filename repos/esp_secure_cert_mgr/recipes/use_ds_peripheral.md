# 使用 Digital Signature (DS) 外设获取私钥上下文

> **适用摘要**: 当设备预置时启用 RSA Digital Signature (DS) 外设（ESP32-S2/S3/C3/C5 等），私钥以密文形式存于 `esp_secure_cert` 分区。本配方演示如何在固件中获取 `esp_ds_data_ctx_t` 并喂给 TLS / mbedTLS / PSA 完成签名，同时校验密文有效性。

## 触发意图

- "DS 外设怎么用"
- "拿到 esp_ds_data_ctx_t"
- "RSA 私钥被 DS 加密了怎么签名"
- "预置设备的私钥在哪里"
- "esp_secure_cert_get_ds_ctx"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | `SOC_DIG_SIGN_SUPPORTED`（ESP32-S2 / S3 / C3 / C5 等） |
| Kconfig | `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL=y`（默认开，依赖该 SoC 能力，会 `select MBEDTLS_HARDWARE_RSA_DS_PERIPHERAL`） |
| eFuse | DS 相关密钥已在预置/工具流程中烧入对应 block |
| 分区 | 含 `ESP_SECURE_CERT_DS_DATA_TLV`(3) 与 `ESP_SECURE_CERT_DS_CONTEXT_TLV`(4)，subtype 0 |
| 参考示例 | `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（`test_ciphertext_validity`） |

## 分步说明

### 1. 头文件包含

```c
#include "esp_log.h"
#include "esp_secure_cert_read.h"   // DS 启用时此头暴露 get_ds_ctx / free_ds_ctx
#include "soc/soc_caps.h"
```

> 启用 DS 时 `esp_secure_cert_read.h` 内部会 `#include "esp_ds.h"`（或 PSA 版 `psa_crypto_driver_esp_rsa_ds.h`）。

### 2. 获取 DS 上下文

```c
static const char *TAG = "ds_app";

void use_ds(void)
{
    esp_ds_data_ctx_t *ds_data = esp_secure_cert_get_ds_ctx();
    if (ds_data == NULL) {
        ESP_LOGE(TAG, "Failed to obtain DS context");
        return;
    }

    ESP_LOGI(TAG, "rsa_length_bits=%d efuse_key_id=%d",
             ds_data->rsa_length_bits, ds_data->efuse_key_id);
    /* ds_data->esp_ds_data 含密文 c (ESP_DS_C_LEN 字节) 与 iv (ESP_DS_IV_LEN 字节) */

    /* ... 把 ds_data 交给 esp-tls 或自行签名 ... */

    esp_secure_cert_free_ds_ctx(ds_data);   // 释放上下文内存
}
```

> 该 API 仅生成 subtype 0 的 DS 上下文（首个 DS 条目）。

### 3. 交给 esp-tls（推荐路径）

`esp_tls` 支持 DS：把 `ds_data` 填入 `esp_tls_cfg_t` 的 `clientkey_ds` 字段（具体字段名以所用 ESP-IDF 版本的 `esp_tls.h` 为准）。组件本身不在此处提供封装，仅负责返回上下文。

### 4. 自行用 mbedTLS/PSA 验证密文（来自示例）

`examples/esp_secure_cert_app/main/app_main.c` 的 `test_ciphertext_validity()` 给出了完整流程：用 DS 私钥对一段 hash 签名，再用设备证书的公钥验签。关键片段（IDF >= 6.0 / mbedTLS 4.x 的 PSA 路径）：

```c
#include "psa/crypto.h"
#include "mbedtls/x509.h"
#include "psa_crypto_driver_esp_rsa_ds.h"

/* 导入 DS opaque key */
esp_rsa_ds_opaque_key_t rsa_ds_opaque_key = {0};
rsa_ds_opaque_key.ds_data_ctx = ds_data;

psa_key_attributes_t attributes = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attributes, PSA_KEY_TYPE_RSA_KEY_PAIR);
psa_set_key_bits(&attributes, ds_data->rsa_length_bits);
psa_set_key_usage_flags(&attributes, PSA_KEY_USAGE_SIGN_HASH);
psa_set_key_algorithm(&attributes, PSA_ALG_RSA_PKCS1V15_SIGN(PSA_ALG_SHA_256));
psa_set_key_lifetime(&attributes, PSA_KEY_LIFETIME_ESP_RSA_DS_VOLATILE);

psa_key_id_t key_id = PSA_KEY_ID_NULL;
psa_import_key(&attributes, (const uint8_t *)&rsa_ds_opaque_key,
               sizeof(esp_rsa_ds_opaque_key_t), &key_id);

/* 签名 */
uint32_t hash[8] = {[0 ... 7] = 0xAABBCCDD};
unsigned char sig[1000]; size_t sig_len = 0;
psa_sign_hash(key_id, PSA_ALG_RSA_PKCS1V15_SIGN(PSA_ALG_SHA_256),
              (const unsigned char *)hash, sizeof(hash), sig, sizeof(sig), &sig_len);
psa_destroy_key(key_id);

/* 用设备证书公钥验签 */
mbedtls_x509_crt crt; mbedtls_x509_crt_init(&crt);
mbedtls_x509_crt_parse(&crt, (const unsigned char *)dev_cert, dev_cert_len);
mbedtls_pk_verify(&crt.pk, MBEDTLS_MD_SHA256,
                  (const unsigned char *)hash, 0, sig, sig_len);
```

> IDF < 6.0 走 `esp_ds_init_data_ctx()` + `esp_ds_rsa_sign()` 旧路径（见示例中 `#elif ESP_IDF_VERSION < ESP_IDF_VERSION_VAL(6, 0, 0)` 分支）。务必参照示例按 IDF 版本分支。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `undefined reference to esp_secure_cert_get_ds_ctx` | 未启用 `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL` | menuconfig 勾选，或芯片无 DS 能力 |
| `get_ds_ctx` 返回 NULL | 分区缺 DS_DATA/DS_CONTEXT 条目（subtype 0） | 用工具重新生成分区，确保 DS 配置 |
| DS 签名失败 | eFuse DS key 未烧 / block id 不匹配 | 核对预置时 `--efuse_key_id` 与烧录 |
| 切到新 IDF 后编译报错 | mbedTLS 4.x / PSA API 变更 | 严格按示例的 `MBEDTLS_VERSION_NUMBER` / `ESP_IDF_VERSION` 分支 |
| 释放后仍用 ds_data | 未配对 `free_ds_ctx` 或释放后访问 | 用完立即 `esp_secure_cert_free_ds_ctx` |

## 参考

- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_read.h`（`#else /* DS_PERIPHERAL */` 分支）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（`test_ciphertext_validity`）
- `espressif-repos/esp_secure_cert_mgr/README.md`（"What is Pre-Provisioning?" / DS 章节）
- `espressif-repos/esp_secure_cert_mgr/Kconfig`（`ESP_SECURE_CERT_DS_PERIPHERAL`）
