# esp_secure_cert_mgr Configuration Reference

> 全部取自仓库 `Kconfig`、`idf_component.yml`、头文件宏与 `docs/format.md`。

## Kconfig 选项（`Kconfig` → "ESP Secure Cert Manager" 菜单）

| 选项 | 类型 | 默认 | 依赖 | 作用 |
|---|---|---|---|---|
| `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL` | bool | y | `SOC_DIG_SIGN_SUPPORTED` | 启用 DS 外设支持；`select MBEDTLS_HARDWARE_RSA_DS_PERIPHERAL`。启用后 `get_priv_key` 不可见，改用 `get_ds_ctx` |
| `CONFIG_ESP_SECURE_CERT_SUPPORT_LEGACY_FORMATS` | bool | n | — | 启用旧格式 `cust_flash` / `nvs` 支持；当前格式为 `cust_flash_tlv` |
| `CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION` | bool | n | `SECURE_BOOT_V2_RSA_ENABLED` 或 `SECURE_BOOT_V2_ECDSA_ENABLED` | 启用分区签名校验（依赖 Secure Boot V2） |
| `CONFIG_ESP_SECURE_CERT_WRITE_ENABLE_LOGGING` | bool | n | — | 写操作的详细日志（info/debug/warning）；生产构建建议关闭以减小体积 |

## 组件依赖（`idf_component.yml`）

```yaml
version: "2.9.2"
description: "ESP Secure Cert Manager"
url: https://github.com/espressif/esp_secure_cert_mgr
dependencies:
  idf:
    version: ">=4.3"
```

> 写入类 API 由 `esp_secure_cert_tlv_read.h` 在编译期检测：`ESP_IDF_VERSION >= 5.3.0` 才定义 `ESP_SECURE_CERT_WRITE_SUPPORT`。

## SoC 能力依赖（决定 API 可见性）

| 能力宏 | 影响的 API / 特性 |
|---|---|
| `SOC_DIG_SIGN_SUPPORTED` | `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL` 可选；DS 相关 API |
| `SOC_HMAC_SUPPORTED` | `esp_secure_cert_append_tlv_with_hmac_encryption` / `_with_hmac_ecdsa_derivation` / `derive_hmac_ecdsa_key` / `esp_secure_cert_crypto.h` |
| `SOC_ECDSA_SUPPORTED` | ECDSA 外设签名路径 |
| `SOC_ECDSA_SUPPORT_DETERMINISTIC_MODE` | 选择 `PSA_ALG_DETERMINISTIC_ECDSA` |

## partitions.csv 配置（来自 `docs/format.md`）

| 格式 | 行 | 大小 |
|---|---|---|
| TLV（默认 `cust_flash_tlv`） | `esp_secure_cert, 0x3F, , 0xD000, 0x2000, encrypted` | 8 KiB |
| legacy `cust_flash` | `esp_secure_cert, 0x3F, , 0xD000, 0x6000, encrypted` | 24 KiB |
| legacy `nvs` | `esp_secure_cert, data, nvs, 0xD000, 0x6000,` | 24 KiB |
| 旧设备 `pre_prov`（cust_flash） | `pre_prov, 0x3F, , 0xD000, 0x6000,` | 24 KiB |

要点：
- 自定义分区类型 `0x3F`（`ESP_SECURE_CERT_TLV_PARTITION_TYPE` / `ESP_SECURE_CERT_CUST_FLASH_PARTITION_TYPE`）。
- 分区名常量 `ESP_SECURE_CERT_TLV_PARTITION_NAME = "esp_secure_cert"`；旧设备为 `"pre_prov"`。
- `encrypted` flag：开启 flash 加密时强制加密本分区；未开启时被忽略。
- OTA 示例使用更靠后偏移（如 `0x20000`，大小 `0x10000`）。

## 示例工程 sdkconfig 片段（来自 examples）

`esp_secure_cert_app/sdkconfig.defaults`：
```ini
CONFIG_PARTITION_TABLE_CUSTOM=y
CONFIG_PARTITION_TABLE_OFFSET=0xc000
CONFIG_PARTITION_TABLE_CUSTOM_FILENAME="partitions.csv"
```

`esp_secure_cert_app/sdkconfig.defaults.esp32`：
```ini
CONFIG_ESP32_REV_MIN_3=y
CONFIG_ESP32_REV_MIN=3
CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION=y
CONFIG_SECURE_BOOT=y
CONFIG_EXAMPLE_ESP_SECURE_CERT_ENABLE_CI=y
CONFIG_ESP_SECURE_CERT_SUPPORT_LEGACY_FORMATS=n
CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL=y
```

OTA 示例 mode Kconfig（`esp_secure_cert_ota_example/main/Kconfig.projbuild`）：
- `CONFIG_EXAMPLE_ESP_SECURE_CERT_OTA_USE_UNALLOCATED_SPACE`（默认）
- `CONFIG_EXAMPLE_ESP_SECURE_CERT_USE_PASSIVE_OTA`
- `CONFIG_EXAMPLE_ESP_SECURE_CERT_DIRECT_OTA`

`esp_secure_cert_app` 写 demo 开关：`CONFIG_EXAMPLE_ESP_SECURE_CERT_WRITE_DEMO`（Example Configuration 菜单）。

## TLV 写配置默认值

`esp_secure_cert_write_config_init()`：
- `FLASH` 模式：`check_erase = true`，`auto_erase = false`。
- `BUFFER` 模式：需手动填 `buffer` / `buffer_size` / `bytes_written`。

## 派生 / 加密相关常量

| 宏（`esp_secure_cert_tlv_private.h`） | 值 | 含义 |
|---|---|---|
| `ESP_SECURE_CERT_HMAC_KEY_ID` | 0 | HMAC-ECDSA 派生用的 hmac_key_id |
| `ESP_SECURE_CERT_DERIVED_ECDSA_KEY_SIZE` | 32 | 派生 ECDSA 原始密钥字节数 |
| `ESP_SECURE_CERT_KEY_DERIVATION_ITERATION_COUNT` | 2048 | PBKDF2 迭代次数 |
| `ESP_SECURE_CERT_ECDSA_DER_KEY_SIZE` | 121 | 派生密钥转 DER 后字节数 |
| `MIN_ALIGNMENT_REQUIRED` | 16 | flash 加密写对齐 |
| `HMAC_ENCRYPTION_IV_LEN` | 16 | HMAC-AES-GCM IV 长度 |
| `HMAC_ENCRYPTION_TAG_LEN` | 16 | GCM tag 长度 |
| `HMAC_ENCRYPTION_AES_GCM_KEY_LEN` | 32 | AES-GCM 密钥长度 |
