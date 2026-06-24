# 启动期校验分区完整性与签名

> **适用摘要**: 设备启动时对 `esp_secure_cert` 分区做两层校验：完整性 TLV（SHA256，工具生成时自动追加）与签名块（Secure Boot V2，RSA/ECDSA）。校验应在解析分区内容之前进行。

## 触发意图

- "校验 esp_secure_cert 分区"
- "启动时验签"
- "esp_secure_cert_verify_partition_integrity"
- "esp_secure_cert_verify_partition_signature"
- "防止分区被篡改"

## 前置条件

| 条件 | 要求 |
|---|---|
| 完整性校验 | 分区含 `ESP_SECURE_CERT_INTEGRITY_TLV`（工具生成时自动追加） |
| 签名校验 | `CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION=y`，依赖 `SECURE_BOOT_V2_RSA_ENABLED` 或 `SECURE_BOOT_V2_ECDSA_ENABLED` |
| 头文件 | `esp_secure_cert_tlv_read.h`（完整性）、`esp_secure_cert_signature_verify.h`（签名） |
| 参考文档 | `espressif-repos/esp_secure_cert_mgr/docs/esp_secure_cert_tools/secure_verification.md` |

## 分步说明

### 1. 完整性校验（SHA256）

```c
#include "esp_log.h"
#include "esp_secure_cert_tlv_read.h"

static const char *TAG = "verify";

void check_integrity(void)
{
    /* 查找最高 subtype 的 INTEGRITY TLV，取其 SHA256，
       与除该 TLV 外的整个分区计算结果比对 */
    esp_err_t err = esp_secure_cert_verify_partition_integrity();
    if (err == ESP_OK) {
        ESP_LOGI(TAG, "Integrity OK");
    } else if (err == ESP_ERR_NOT_FOUND) {
        ESP_LOGW(TAG, "No integrity TLV present");
    } else {
        ESP_LOGE(TAG, "Integrity verification FAILED");
    }
}
```

### 2. 签名校验（Secure Boot V2）

```c
#include "esp_secure_cert_signature_verify.h"

void app_main(void)
{
#if CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION
    ESP_LOGI(TAG, "Starting signature verification...");
    /* ctx 仅供兼容，当前实现未使用，必须传 NULL */
    esp_err_t sig_ret = esp_secure_cert_verify_partition_signature(NULL);
    if (sig_ret == ESP_OK) {
        ESP_LOGI(TAG, "Signature PASSED");
    } else {
        ESP_LOGE(TAG, "Signature FAILED");
    }
#endif

    /* 继续正常逻辑（读证书等）... */
}
```

签名块机制（来自 `secure_verification.md`）：
- 对除 `ESP_SECURE_CERT_SIGNATURE_BLOCK_TLV` 外的所有 TLV 计算 SHA256/SHA384；
- 签名块作为最后一个 TLV 追加，含签名与签名公钥；
- 支持多个签名块（subtype 0/1/2），任一通过即成功（便于密钥轮换）；
- RSA 块 1216 字节原样存储；ECDSA 仅存必要字段，验签时在 RAM 重建 1216 字节结构。

### 3. sdkconfig.defaults 配置

```ini
# 完整性校验无需额外选项（依赖工具生成的 INTEGRITY TLV）
# 签名校验：
CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION=y
# 并启用对应 Secure Boot V2（RSA 或 ECDSA），由 bootloader 链支撑
```

### 4. 在 OTA 流程中校验 staging 分区

OTA 下载到 staging 后，先 `esp_secure_cert_tlv_set_partition(staging)` 切换活动分区，再调用上述两个校验 API，通过后才复制到主分区（见 `recipes/ota_update_partition.md`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `verify_partition_integrity` 返回 `ESP_ERR_NOT_FOUND` | 分区无 INTEGRITY TLV | 用 `configure_esp_secure_cert.py` 重新生成（自动追加） |
| 完整性 FAIL | 分区被篡改 / 写入未刷新映射 | 写后 `unmap_partition()` 再校验 |
| 签名函数链接失败 | 未启用 `CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION` | menuconfig 勾选并满足 Secure Boot 依赖 |
| 签名 FAIL | 签名密钥与 secure boot 不一致 / 签名块缺失 | 用 `--secure-sign --signing-key-file` 重新签名 |
| 传非 NULL ctx | 结构体仅占位 | 始终传 `NULL` |

## 参考

- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_tlv_read.h`（`verify_partition_integrity`）
- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_signature_verify.h`（`verify_partition_signature` / `esp_sign_verify_ctx_t`）
- `espressif-repos/esp_secure_cert_mgr/docs/esp_secure_cert_tools/secure_verification.md`（签名块、多签名、命令行签名）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_app/main/app_main.c`（启动期签名校验段）
- `espressif-repos/esp_secure_cert_mgr/Kconfig`（`ESP_SECURE_CERT_SECURE_VERIFICATION`）
