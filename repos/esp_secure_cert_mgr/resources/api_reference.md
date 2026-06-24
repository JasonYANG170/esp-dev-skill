# esp_secure_cert_mgr API Reference

> 全部签名与枚举取自仓库真实头文件（`include/`）。按模块分组。仅在对应 Kconfig / SoC 能力启用时可见的 API 已注明条件。

## 读取 API — `esp_secure_cert_read.h`

```c
/* 初始化 NVS 分区（仅 nvs 格式需要） */
esp_err_t esp_secure_cert_init_nvs_partition(void);

/* 设备证书（返回 subtype 0 的首条 DEV_CERT） */
esp_err_t esp_secure_cert_get_device_cert(char **buffer, uint32_t *len);
esp_err_t esp_secure_cert_free_device_cert(char *buffer);

/* CA 证书（返回 subtype 0 的首条 CA_CERT） */
esp_err_t esp_secure_cert_get_ca_cert(char **buffer, uint32_t *len);
esp_err_t esp_secure_cert_free_ca_cert(char *buffer);
```

私钥（与 DS 互斥编译）：

```c
#ifndef CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL
/* 非 DS：返回明文/派生私钥指针 */
esp_err_t esp_secure_cert_get_priv_key(char **buffer, uint32_t *len);
esp_err_t esp_secure_cert_free_priv_key(char *buffer);
#else
/* DS：返回 DS 上下文（仅 subtype 0） */
esp_ds_data_ctx_t *esp_secure_cert_get_ds_ctx(void);
void               esp_secure_cert_free_ds_ctx(esp_ds_data_ctx_t *ds_ctx);
#endif
```

私钥元信息（仅 TLV，未启用 legacy 时可用）：

```c
#ifndef CONFIG_ESP_SECURE_CERT_SUPPORT_LEGACY_FORMATS
esp_err_t esp_secure_cert_get_priv_key_type(esp_secure_cert_key_type_t *priv_key_type);
esp_err_t esp_secure_cert_get_priv_key_efuse_id(uint8_t *efuse_block_id);
#endif
```

### `esp_secure_cert_key_type_t`（`esp_secure_cert_read.h`）

| 枚举 | 含义 |
|---|---|
| `ESP_SECURE_CERT_INVALID_KEY` | 非法 |
| `ESP_SECURE_CERT_DEFAULT_FORMAT_KEY` | 默认格式 |
| `ESP_SECURE_CERT_HMAC_ENCRYPTED_KEY` | HMAC 加密 |
| `ESP_SECURE_CERT_HMAC_DERIVED_ECDSA_KEY` | HMAC 派生 ECDSA |
| `ESP_SECURE_CERT_ECDSA_PERIPHERAL_KEY` | ECDSA 外设（eFuse） |

## TLV 通用读取 — `esp_secure_cert_tlv_read.h`

```c
/* 按 type + subtype 查询（自动解密 + CRC 校验） */
esp_err_t esp_secure_cert_get_tlv_info(esp_secure_cert_tlv_config_t *tlv_config,
                                       esp_secure_cert_tlv_info_t *tlv_info);
esp_err_t esp_secure_cert_free_tlv_info(esp_secure_cert_tlv_info_t *tlv_info);

/* 迭代器（零初始化从头开始） */
esp_err_t esp_secure_cert_iterate_to_next_tlv(esp_secure_cert_tlv_iterator_t *tlv_iterator);
esp_err_t esp_secure_cert_get_tlv_info_from_iterator(esp_secure_cert_tlv_iterator_t *tlv_iterator,
                                                     esp_secure_cert_tlv_info_t *tlv_info);

/* 一键打印清单 */
void esp_secure_cert_list_tlv_entries(void);

/* 分区映射管理 */
esp_err_t esp_secure_cert_map_partition(esp_secure_cert_partition_ctx_t **ctx);
void      esp_secure_cert_unmap_partition(void);
esp_err_t esp_secure_cert_tlv_set_partition(const esp_partition_t *partition);

/* 完整性校验 */
esp_err_t esp_secure_cert_verify_partition_integrity(void);
```

### 关键结构体

```c
typedef struct tlv_config {
    esp_secure_cert_tlv_type_t    type;
    esp_secure_cert_tlv_subtype_t subtype;
} esp_secure_cert_tlv_config_t;

typedef struct tlv_info {
    esp_secure_cert_tlv_type_t    type;
    esp_secure_cert_tlv_subtype_t subtype;
    char    *data;
    uint32_t length;
    uint8_t  flags;
} esp_secure_cert_tlv_info_t;

typedef struct esp_secure_cert_partition_ctx {
    const esp_partition_t       *partition;
    const void                  *esp_secure_cert_mapped_addr;
    spi_flash_mmap_handle_t      handle;
#ifdef ESP_SECURE_CERT_WRITE_SUPPORT
    atomic_bool                  write_lock;   /* 写锁（IDF >= 5.3） */
#endif
} esp_secure_cert_partition_ctx_t;

typedef struct tlv_iterator {
    void *iterator;
} esp_secure_cert_tlv_iterator_t;
```

## 写入 API — `esp_secure_cert_write.h`（需 `ESP_SECURE_CERT_WRITE_SUPPORT`，即 IDF >= 5.3）

```c
esp_err_t esp_secure_cert_erase_partition(void);
esp_err_t esp_secure_cert_check_flash_erased(size_t offset, size_t size, bool *is_erased);

esp_err_t esp_secure_cert_append_tlv(esp_secure_cert_tlv_info_t *tlv_info,
                                     const esp_secure_cert_write_config_t *write_config);
esp_err_t esp_secure_cert_append_tlv_batch(esp_secure_cert_tlv_info_t *tlv_entries,
                                           size_t num_entries,
                                           const esp_secure_cert_write_config_t *write_config);

#if SOC_HMAC_SUPPORTED
esp_err_t esp_secure_cert_append_tlv_with_hmac_encryption(esp_secure_cert_tlv_info_t *tlv_info,
                                                          const esp_secure_cert_write_config_t *write_config);
esp_err_t esp_secure_cert_append_tlv_with_hmac_ecdsa_derivation(const uint8_t *salt, size_t salt_len,
                                                                 esp_secure_cert_tlv_subtype_t subtype,
                                                                 const esp_secure_cert_write_config_t *write_config);
esp_err_t esp_secure_cert_derive_hmac_ecdsa_key(uint8_t *pub_key_buf, size_t *pub_key_len,
                                                  esp_secure_cert_tlv_subtype_t subtype,
                                                  const esp_secure_cert_write_config_t *write_config);
#endif

/* 内联初始化器（头文件内 static inline） */
static inline void esp_secure_cert_write_config_init(esp_secure_cert_write_config_t *config,
                                                     esp_secure_cert_write_mode_t mode);
```

### 写配置结构体

```c
typedef enum {
    ESP_SECURE_CERT_WRITE_MODE_FLASH  = 0,
    ESP_SECURE_CERT_WRITE_MODE_BUFFER = 1,
} esp_secure_cert_write_mode_t;

typedef struct {
    esp_secure_cert_write_mode_t mode;
    union {
        struct { bool check_erase; bool auto_erase; } flash;
        struct { uint8_t *buffer; size_t buffer_size; size_t *bytes_written; } buffer;
    };
    uint32_t reserved[4];
} esp_secure_cert_write_config_t;
```

## 写错误码 — `esp_secure_cert_write_errors.h`

base = `0x7000`。完整列表：

| 错误码 | 值 |
|---|---|
| `ESP_ERR_SECURE_CERT_TLV_INVALID_TYPE` | +1 |
| `ESP_ERR_SECURE_CERT_TLV_INVALID_LENGTH` | +2 |
| `ESP_ERR_SECURE_CERT_TLV_INVALID_DATA` | +3 |
| `ESP_ERR_SECURE_CERT_TLV_BUFFER_TOO_SMALL` | +4 |
| `ESP_ERR_SECURE_CERT_TLV_ALREADY_EXISTS` | +5 |
| `ESP_ERR_SECURE_CERT_PARTITION_NOT_FOUND` | +10 |
| `ESP_ERR_SECURE_CERT_PARTITION_ACCESS_FAILED` | +11 |
| `ESP_ERR_SECURE_CERT_FLASH_NOT_ERASED` | +12 |
| `ESP_ERR_SECURE_CERT_FLASH_WRITE_FAILED` | +13 |
| `ESP_ERR_SECURE_CERT_FLASH_READ_FAILED` | +14 |
| `ESP_ERR_SECURE_CERT_ERASE_FAILED` | +15 |
| `ESP_ERR_SECURE_CERT_INVALID_WRITE_MODE` | +20 |
| `ESP_ERR_SECURE_CERT_BUFFER_CONFIG_INVALID` | +21 |
| `ESP_ERR_SECURE_CERT_BUFFER_OVERFLOW` | +22 |
| `ESP_ERR_SECURE_CERT_WRITE_OFFSET_INVALID` | +23 |
| `ESP_ERR_SECURE_CERT_HMAC_KEY_NOT_FOUND` | +30 |
| `ESP_ERR_SECURE_CERT_HMAC_ENCRYPTION_FAILED` | +31 |
| `ESP_ERR_SECURE_CERT_HMAC_IV_GENERATION_FAILED` | +32 |
| `ESP_ERR_SECURE_CERT_WRITE_NO_MEMORY` | +40 |
| `ESP_ERR_SECURE_CERT_ERASE_CHECK_NO_MEMORY` | +41 |
| `ESP_ERR_SECURE_CERT_WRITE_IN_PROGRESS` | +50 |
| `ESP_ERR_SECURE_CERT_ECDSA_KEY_INVALID` | +60 |
| `ESP_ERR_SECURE_CERT_ECDSA_KEY_GEN_FAILED` | +61 |
| `ESP_ERR_SECURE_CERT_EFUSE_WRITE_FAILED` | +62 |
| `ESP_ERR_SECURE_CERT_HMAC_KEY_ALREADY_EXISTS` | +63 |
| `ESP_ERR_SECURE_CERT_KEY_VERIFICATION_FAILED` | +64 |
| `ESP_ERR_SECURE_CERT_PBKDF2_FAILED` | +65 |

## 签名校验 — `esp_secure_cert_signature_verify.h`

```c
typedef struct esp_sign_verify_ctx { /* 占位，当前未使用 */ } esp_sign_verify_ctx_t;

/* 校验分区签名（Secure Boot V2），ctx 必须传 NULL */
esp_err_t esp_secure_cert_verify_partition_signature(esp_sign_verify_ctx_t *ctx);
```

> 启用条件：`CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION=y`（依赖 `SECURE_BOOT_V2_RSA_ENABLED` 或 `SECURE_BOOT_V2_ECDSA_ENABLED`）。

## HMAC / 派生辅助 — `esp_secure_cert_crypto.h`（`SOC_HMAC_SUPPORTED`）

```c
int      esp_pbkdf2_hmac_sha256(hmac_key_id_t hmac_key_id, const unsigned char *salt,
                                size_t salt_len, size_t iteration_count,
                                size_t key_length, unsigned char *output);
esp_err_t esp_secure_cert_validate_ecdsa_key(const uint8_t *key_buf, size_t key_len);
esp_err_t esp_secure_cert_sw_pbkdf2_hmac_sha256(const uint8_t *hmac_key, size_t hmac_key_len,
                                                  const uint8_t *salt, size_t salt_len,
                                                  uint32_t iterations,
                                                  uint8_t *output, size_t output_len);
esp_err_t esp_secure_cert_calc_public_key(const uint8_t *priv_key_buf, size_t priv_key_len,
                                           uint8_t *pub_key_buf, size_t *pub_key_len);
```

## TLV 类型 / 子类型 — `esp_secure_cert_tlv_config.h`

见 `SKILL.md` 的 "TLV 类型" 参考表。要点：subtype `0..16`，`ESP_SECURE_CERT_SUBTYPE_MAX=254`（取最高 subtype）。

## 私有头常量（`private_include/`，应用一般不直接用）

`esp_secure_cert_tlv_private.h`：

```c
#define ESP_SECURE_CERT_TLV_PARTITION_TYPE   0x3F
#define ESP_SECURE_CERT_TLV_PARTITION_NAME   "esp_secure_cert"
#define ESP_SECURE_CERT_TLV_MAGIC            0xBA5EBA11
#define ESP_SECURE_CERT_HMAC_KEY_ID          (0)
#define ESP_SECURE_CERT_DERIVED_ECDSA_KEY_SIZE   (32)
#define ESP_SECURE_CERT_KEY_DERIVATION_ITERATION_COUNT  (2048)
#define ESP_SECURE_CERT_ECDSA_DER_KEY_SIZE   121
#define MIN_ALIGNMENT_REQUIRED               16

/* TLV flags */
#define ESP_SECURE_CERT_TLV_FLAG_HMAC_ENCRYPTION            (2 << 6)
#define ESP_SECURE_CERT_TLV_FLAG_HMAC_ECDSA_KEY_DERIVATION  (1 << 6)
#define ESP_SECURE_CERT_TLV_FLAG_KEY_ECDSA_PERIPHERAL       (1 << 3)

typedef struct __attribute__((packed)) esp_secure_cert_tlv_header {
    uint32_t magic; uint8_t flags; uint8_t reserved[3];
    uint8_t type; uint8_t subtype; uint16_t length; uint8_t value[0];
} esp_secure_cert_tlv_header_t;   /* sizeof == 12 */

typedef struct esp_secure_cert_tlv_footer { uint32_t crc; } esp_secure_cert_tlv_footer_t; /* sizeof == 4 */
```

`esp_secure_cert_config.h`：定义 NVS 命名空间键名（`priv_key`/`dev_cert`/`ca_cert`/`cipher_c`/`rsa_len`/`ds_key_id`/`iv`）、cust_flash 偏移布局、magic 字节（`0xC1`/`0xC2`/`0xC3`）、分区类型常量（`0x3F`）。
