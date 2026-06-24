# 对 esp_secure_cert 分区做 Fail-Safe OTA 升级

> **适用摘要**: 远程更新 `esp_secure_cert` 分区（轮换证书/密钥）。演示三种暂存策略（unallocated space / passive OTA / direct）、用 NVS 记录恢复点实现断电回滚，以及 `esp_secure_cert_tlv_set_partition()` 切换活动分区做校验。

## 触发意图

- "OTA 更新证书分区"
- "远程轮换设备证书"
- "esp_secure_cert 断电保护"
- "esp_secure_cert_tlv_set_partition"
- "staging 分区"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考 | `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_ota_example/`（三种 mode 的完整实现） |
| 网络 | HTTPS OTA 服务器托管新的 `esp_secure_cert.bin` |
| 暂存（推荐） | unallocated flash 空间 ≥ 分区大小，或可用的 passive OTA 分区 |
| NVS | 用于保存 staging 偏移/大小/label/type/subtype 以便恢复 |
| 镜像 | 新镜像由 `configure_esp_secure_cert.py` 生成（自动追加 INTEGRITY TLV） |

## 分步说明

### 1. 三种 OTA 模式（Kconfig 选择）

| 模式 | Kconfig | staging 来源 | 安全性 |
|---|---|---|---|
| 未分配空间（推荐） | `CONFIG_EXAMPLE_ESP_SECURE_CERT_OTA_USE_UNALLOCATED_SPACE` | 分区表后的空洞 | 高，主分区不动 |
| passive OTA 分区 | `CONFIG_EXAMPLE_ESP_SECURE_CERT_USE_PASSIVE_OTA` | 空闲 OTA app 分区 | 高，主分区不动 |
| 直接写（不推荐） | `CONFIG_EXAMPLE_ESP_SECURE_CERT_DIRECT_OTA` | 主分区本身 | 低，断电即损坏 |

### 2. 找到主分区并准备 staging

```c
#include "esp_partition.h"
#include "esp_https_ota.h"
#include "esp_secure_cert_tlv_read.h"
#include "esp_secure_cert_tlv_config.h"

#define ESP_SECURE_CERT_CUST_FLASH_PARTITION_TYPE 0x3F
#define ESP_SECURE_CERT_TLV_PARTITION_NAME        "esp_secure_cert"

const esp_partition_t *primary = esp_partition_find_first(
    ESP_SECURE_CERT_CUST_FLASH_PARTITION_TYPE,
    ESP_PARTITION_SUBTYPE_ANY,
    ESP_SECURE_CERT_TLV_PARTITION_NAME);
```

> unallocated 模式用 `partition_utils_find_unallocated()` 找空洞并 `esp_partition_register_external()` 注册为临时 staging；passive 模式用 `esp_ota_get_next_update_partition(NULL)`。

### 3. 下载到 staging 并校验

```c
/* esp_https_ota_config_t.partition.staging / .final 按模式填好 */
esp_err_t err = esp_https_ota(&ota_config);     // 下载到 staging
if (err != ESP_OK) { /* 清理临时分区 */ return err; }

/* 切到 staging 做校验 */
esp_secure_cert_tlv_set_partition(staging);
err = esp_secure_cert_verify_partition_integrity();
if (err != ESP_OK) {
    esp_secure_cert_tlv_set_partition(NULL);    // 失败复位
    return err;
}
#if CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION
err = esp_secure_cert_verify_partition_signature(NULL);
if (err != ESP_OK) { esp_secure_cert_tlv_set_partition(NULL); return err; }
#endif
```

### 4. 保存恢复点到 NVS（断电保护）

```c
/* 把 staging 的 address/size/label/type/subtype 写入 NVS（namespace "esc"） */
nvs_handle_t h;
nvs_open("esc", NVS_READWRITE, &h);
nvs_set_u32(h, "stg_addr",   staging->address);
nvs_set_u32(h, "stg_size",   staging->size);
nvs_set_str(h, "stg_label",  staging->label);
nvs_set_u8(h,  "stg_type",    staging->type);
nvs_set_u8(h,  "stg_subtype", staging->subtype);
nvs_commit(h); nvs_close(h);
```

### 5. 复制 staging → 主分区，然后清理

```c
ESP_LOGW(TAG, "确保供电稳定！此时断电可能损坏分区");
err = esp_partition_copy(primary, 0, staging, 0, primary->size);
esp_secure_cert_tlv_set_partition(NULL);        // 复位到主分区
clear_staging_info_from_nvs();                  // 清除恢复点
/* 注销临时分区（unallocated 模式） */
esp_partition_deregister_external(staging);
```

### 6. 启动期恢复（处理上次中断）

`app_main()` 开头先 `check_and_recover_staging_partition()`：
1. 从 NVS 读 staging 信息 → `register_partition()` 重建；
2. `esp_secure_cert_tlv_set_partition(recovered)`；
3. `esp_secure_cert_verify_partition_integrity()` 再校验一次；
4. `esp_partition_copy(primary, recovered, ...)` 完成中断的复制；
5. `esp_secure_cert_tlv_set_partition(NULL)` + 清 NVS。

> 若 NVS 无恢复点（`ESP_ERR_NOT_FOUND`），正常启动 OTA 任务。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 复制后读到旧数据 | 未 `tlv_set_partition(NULL)` 复位 | 复制完成后显式复位 |
| 断电后分区损坏 | 用了 direct OTA 或复制中断 | 改用暂存模式 + NVS 恢复点 |
| staging 校验失败 | 下载不完整 / 镜像无 INTEGRITY TLV | 用工具重新生成镜像 |
| `partition_copy` 失败 | 主分区大小 < staging | 保证两者同尺寸 |
| passive 模式无可用分区 | OTA 分区含有效 app | 排查 `esp_ota_get_state_partition` 状态 |
| 恢复后 staging 指针失效 | `set_partition` 内部 unmap 旧分区 | 切换后重新调用读 API |

## 参考

- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_ota_example/main/app_main.c`（`esp_secure_cert_ota_update` / `check_and_recover_staging_partition`）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_ota_example/README.md`（三种模式说明）
- `espressif-repos/esp_secure_cert_mgr/include/esp_secure_cert_tlv_read.h`（`tlv_set_partition` / `verify_partition_integrity`）
- `espressif-repos/esp_secure_cert_mgr/examples/esp_secure_cert_ota_example/partitions.csv`、`partitions_passive_ota.csv`
