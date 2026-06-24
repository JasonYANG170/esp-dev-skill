# esp_secure_cert_mgr 汇总陷阱 (Pitfalls)

> 与 `SKILL.md` 的 "Critical Pitfalls" 互为补充，按主题归类，附真实代码修正。

## 1. 内存与指针

### 1.1 NVS / HMAC 场景读到的指针是 malloc 的，必须配对 free
- `esp_secure_cert_get_device_cert` / `_ca_cert` / `_priv_key` 在 NVS 或 HMAC 加密下会动态分配；cust_flash 返回只读 flash 指针。
- **修正**：每个 `get_*` 配对 `esp_secure_cert_free_*`（内部仅在确实分配时释放）。

### 1.2 `unmap_partition()` / `tlv_set_partition()` 后旧指针失效
- cust_flash 下读指针指向内存映射地址；unmap 或切换分区后映射被解除。
- **修正**：unmap 前把数据拷走，或 unmap 后重新调用读 API。

### 1.3 切换活动分区后未复位 NULL（OTA 读到 staging）
- **修正**：OTA 复制完成后显式 `esp_secure_cert_tlv_set_partition(NULL)`。

## 2. 编译期可见性

### 2.1 DS 开启时 `get_priv_key` 不存在
- `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL=y` 时 `get_priv_key/free_priv_key` 被 `#ifndef` 排除。
- **修正**：DS 场景用 `esp_secure_cert_get_ds_ctx()` / `esp_secure_cert_free_ds_ctx()`。

### 2.2 `get_priv_key_type` / `get_priv_key_efuse_id` 仅 TLV 可用
- 启用 `CONFIG_ESP_SECURE_CERT_SUPPORT_LEGACY_FORMATS` 时被排除。
- **修正**：确认为 TLV 格式或关闭 legacy 支持。

### 2.3 写 API 需 IDF >= 5.3
- 未定义 `ESP_SECURE_CERT_WRITE_SUPPORT` 时 `esp_secure_cert_write.h` 内函数不可用并产生 `#warning`。
- **修正**：升级 IDF，或用宏守卫 `#ifdef ESP_SECURE_CERT_WRITE_SUPPORT`。

## 3. 写入流程

### 3.1 写前未擦除 → `FLASH_NOT_ERASED`
- flash 模式默认 `check_erase=true`。
- **修正**：先 `esp_secure_cert_unmap_partition()` 再 `esp_secure_cert_erase_partition()`（生产设备先备份）。

### 3.2 同 type+subtype 重复 → `TLV_ALREADY_EXISTS`
- **修正**：换 subtype，或先 erase。

### 3.3 buffer 模式缺字段 → `BUFFER_CONFIG_INVALID`
- **修正**：必填 `buffer` / `buffer_size` / `bytes_written`。

### 3.4 并发写 → `WRITE_IN_PROGRESS`
- 写锁为 `atomic_bool` 非阻塞 fail-fast。
- **修正**：避免多任务并发写；失败重试。

## 4. HMAC / ECDSA 派生

### 4.1 `HMAC_KEY_NOT_FOUND`
- eFuse 无 purpose=`HMAC_UP` 的密钥。
- **修正**：先烧 key，或用 `esp_secure_cert_derive_hmac_ecdsa_key()` 一步生成（仅首次，eFuse 无 key 时）。

### 4.2 `HMAC_KEY_ALREADY_EXISTS`
- eFuse 已有 HMAC_UP 又调 `derive_hmac_ecdsa_key`。
- **修正**：改用 `append_tlv_with_hmac_ecdsa_derivation()`。

### 4.3 误以为派生私钥在 flash
- 仅 salt 落盘，私钥读取时由 PBKDF2-HMAC-SHA256（2048 轮）实时派生为 121 字节 DER。

## 5. 校验

### 5.1 签名校验未启用依赖
- `CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION` 依赖 Secure Boot V2。
- **修正**：同时启用 `CONFIG_SECURE_BOOT` 及 RSA/ECDSA V2。

### 5.2 `verify_partition_signature` 传非 NULL ctx
- `esp_sign_verify_ctx_t` 仅占位。
- **修正**：始终传 `NULL`。

### 5.3 完整性校验 `ESP_ERR_NOT_FOUND`
- 分区无 `ESP_SECURE_CERT_INTEGRITY_TLV`。
- **修正**：用 `configure_esp_secure_cert.py` 重新生成（自动追加）。

## 6. 分区表

### 6.1 开了 flash 加密却没加 `encrypted` 标志
- **修正**：TLV 行末尾加 `encrypted`。

### 6.2 旧设备分区名是 `pre_prov`
- **修正**：`partitions.csv` 用 `pre_prov, 0x3F, , 0xD000, 0x6000,`。

### 6.3 误用 Matter 专用 TLV（201/202）
- 文档明确禁止应用使用。
- **修正**：用 `ESP_SECURE_CERT_USER_DATA_1..5`（51..55）。

## 7. ECDSA 外设签名

### 7.1 直接 `mbedtls_pk_parse_key` 解析 eFuse 私钥
- eFuse 私钥不可读。
- **修正**：`get_priv_key_type` 判断为 `ECDSA_PERIPHERAL_KEY` 后，`get_priv_key_efuse_id` 取 block，用 opaque key（`esp_ecdsa_set_pk_context` 或 PSA `esp_ecdsa_opaque_key_t`）。

### 7.2 mbedTLS / PSA 版本分支
- 不同 IDF/mbedTLS 版本头文件与 API 不同。
- **修正**：严格按示例的 `MBEDTLS_VERSION_NUMBER` / `ESP_IDF_VERSION` 分支。
