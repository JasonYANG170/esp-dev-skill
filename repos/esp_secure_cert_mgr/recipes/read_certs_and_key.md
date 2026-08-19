# 读取设备证书 / CA 证书 / 私钥

> **适用摘要**: 在固件中通过便捷 API 读取 `esp_secure_cert` 分区里的设备证书、CA 证书与（非 DS 场景下的）私钥，并正确释放内存。适用于 TLS 客户端加载凭据等场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp_secure_cert_mgr/resources/`, source/examples in `repos/esp_secure_cert_mgr/`, and this recipe path `repos/esp_secure_cert_mgr/recipes/read_certs_and_key.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "读取设备证书"
- "读 CA 证书"
- "从 esp_secure_cert 拿私钥"
- "给 esp-tls / mbedTLS 提供证书和密钥"
- "pre-provisioning 凭据怎么读"

## 前置条件

| 条件 | 要求 |
|---|---|
| 分区 | `partitions.csv` 含 `esp_secure_cert, 0x3F, , 0xD000, 0x2000, encrypted`（TLV 默认） |
| 烧录 | 已用 `configure_esp_secure_cert.py` 生成并烧录 `esp_secure_cert.bin` |
| Kconfig | DS 场景 `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL` 未启用（否则用 `get_priv_key` 编译不可见） |
| 参考示例 | `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c` |

## 分步说明

### 1. 头文件包含

```c
#include "esp_log.h"
#include "esp_secure_cert_read.h"
```

### 2. 读取设备证书 / CA 证书并释放

```c
static const char *TAG = "app";

void read_certs(void)
{
    char *addr = NULL;
    uint32_t len = 0;

    /* 设备证书 */
    if (esp_secure_cert_get_device_cert(&addr, &len) == ESP_OK) {
        ESP_LOGI(TAG, "Device Cert length=%" PRIu32, len);
        ESP_LOG_BUFFER_HEX(TAG, addr, len > 64 ? 64 : len);
        esp_secure_cert_free_device_cert(addr);   // NVS/HMAC 场景会真正 free
    } else {
        ESP_LOGE(TAG, "Failed to get device cert");
    }

    /* CA 证书 */
    addr = NULL; len = 0;
    if (esp_secure_cert_get_ca_cert(&addr, &len) == ESP_OK) {
        ESP_LOGI(TAG, "CA Cert length=%" PRIu32, len);
        esp_secure_cert_free_ca_cert(addr);
    }
}
```

> 说明：cust_flash 分区返回的是只读 flash 指针；NVS / HMAC 加密场景会动态分配，必须 `free`。

### 3. 读取私钥（仅 `!CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL`）

```c
char *priv_key = NULL;
uint32_t key_len = 0;
if (esp_secure_cert_get_priv_key(&priv_key, &key_len) == ESP_OK) {
    ESP_LOGI(TAG, "Private key length=%" PRIu32, key_len);
    /* 交给 mbedTLS / esp-tls 使用 ... */
    esp_secure_cert_free_priv_key(priv_key);
}
```

### 4. 与 esp-tls 配合（典型用法）

把读到的指针直接填入 `esp_tls_cfg_t`（指针在连接期间需保持有效）：

```c
#include "esp_tls.h"

char *dev_cert = NULL, *ca_cert = NULL; uint32_t cl = 0, al = 0;
esp_secure_cert_get_device_cert(&dev_cert, &cl);
esp_secure_cert_get_ca_cert(&ca_cert, &al);

esp_tls_cfg_t cfg = {
    .cacert_buf = (const unsigned char *)ca_cert,
    .cacert_bytes = al,
    .clientcert_buf = (const unsigned char *)dev_cert,
    .clientcert_bytes = cl,
    /* 非 DS 场景可填 clientkey_buf / clientkey_bytes */
};
esp_tls_t *tls = esp_tls_conn_http_new("https://example.com", &cfg);
/* 用完后 free */
```

> 注意：cust_flash 场景返回的是 flash 只读指针，连接期间一直有效；NVS/HMAC 场景是堆内存，连接结束后再 `free`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `get_*` 返回 ESP_FAIL | 分区未生成/未烧录，或 `partitions.csv` 格式错 | 用 `configure_esp_secure_cert.py` 生成烧录；核对分区表 |
| 内存泄漏 | NVS/HMAC 场景 `malloc` 后未 free | 每个 `get_*` 配对 `free_*` |
| 拿不到第二条同类型证书 | 便捷 API 只返回 subtype 0 | 改用 `esp_secure_cert_get_tlv_info()` 指定 type+subtype |
| 链接报 `undefined reference to esp_secure_cert_get_priv_key` | 启用了 DS 外设，API 编译期被排除 | DS 场景改用 `esp_secure_cert_get_ds_ctx()` |
| Flash Encryption 后读到密文 | `partitions.csv` 未加 `encrypted` 标志 | TLV 行末尾加 `encrypted` |

## 参考

- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_read.h`
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（`test_read_existing_data`）
- `espressif-repos/esp_secure_cert_mgr/README.md`
