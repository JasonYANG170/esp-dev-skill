# 迁移旧 mbedtls_* 密码 API 到 PSA

> **适用摘要**: 把基于 Mbed TLS 3.x `mbedtls_*`（AES/SHA/HMAC/CMAC/RSA/ECDSA/ECDH/HKDF/PBKDF2 等）的密码代码迁移到 TF-PSA-Crypto 的 `psa_*` API。涵盖 per-algorithm 旧→新函数对照、RNG 回调移除、配置文件拆分（`MBEDTLS_xxx_C` → `PSA_WANT_*` + `MBEDTLS_PSA_ACCEL_*`）、错误码合并、PK 模块变化与 PAKE 接口更新。

## 触发意图

- "把 mbedtls_aes_crypt / mbedtls_md_hmac / mbedtls_pk_sign 迁移到 PSA"
- "Mbed TLS 3.x 升级到 TF-PSA-Crypto 1.0 / Mbed TLS 4.0"
- "f_rng / p_rng 回调没了怎么办"
- "MBEDTLS_xxx_C 换成 PSA_WANT_*"
- "mbedtls_md_type_t / mbedtls_cipher_type_t 对应哪个 PSA 宏"
- "ECDSA 签名 ASN.1 vs raw 格式"
- "mbedtls_entropy / ctr_drbg 还需要吗"

## 前置条件

| 条件 | 要求 |
|---|---|
| 库状态 | 迁移完成后入口须先调用 `psa_crypto_init()`（替代旧 entropy/ctr_drbg 初始化序列） |
| 头文件 | 新代码只 `#include <psa/crypto.h>`；仅解析/格式化等非密码用途可继续用 `<mbedtls/pk.h>`、`<mbedtls/asn1.h>`、`<mbedtls/nist_kw.h>` |
| 配置项 | 密码机制开关全部迁到 `include/psa/crypto_config.h`（`PSA_WANT_*`），不再用 `MBEDTLS_RSA_C`/`MBEDTLS_SHA256_C` 等旧开关 |
| 参考文档 | `docs/psa-transition.md`（per-API 对照，2406 行）、`docs/1.0-migration-guide.md`（1.0 破坏性变更） |

## 分步说明

### 1. 头文件与库目标重命名（1.0 仓库拆分）

TF-PSA-Crypto 1.0 把 Mbed TLS 拆成两个仓库：密码（本仓库）与 TLS/X.509。文件位置与库名已变（来自 `docs/1.0-migration-guide.md`）：

| 旧（Mbed TLS 3.x） | 新（TF-PSA-Crypto 1.0） |
|---|---|
| `library/*` | `core/`、`drivers/builtin/src/` |
| `include/mbedtls/*`（密码部分） | `include/psa/`、`include/tf-psa-crypto/` |
| `3rdparty/everest`、`3rdparty/p256-m` | `drivers/everest`、`drivers/p256-m` |
| CMake 目标 `mbedcrypto` | `tfpsacrypto` |
| CMake 包 `find_package(MbedTLS)` / `MbedTLS::mbedcrypto` | `find_package(TF-PSA-Crypto)` / `TF-PSA-Crypto::tfpsacrypto` |
| pkg-config `mbedcrypto.pc` | `tfpsacrypto.pc` |
| `MBEDTLS_PSA_CRYPTO_CONFIG_FILE` / `_USER_CONFIG_FILE` | `TF_PSA_CRYPTO_CONFIG_FILE` / `TF_PSA_CRYPTO_USER_CONFIG_FILE` |

```cmake
// 旧（3.x）
target_link_libraries(my_app PRIVATE mbedcrypto)

// 新（1.0）
target_link_libraries(my_app PRIVATE tfpsacrypto)
```

### 2. RNG 回调全部移除（最常见的破坏性变更）

PSA 拥有一个全局 RNG（在 `psa_crypto_init()` 内部播种），所以所有 `f_rng, p_rng` 参数都没了。删除旧 entropy/DRBG 样板代码：

```c
// 旧（3.x）—— 全部删除
mbedtls_entropy_context entropy;
mbedtls_ctr_drbg_context ctr_drbg;
mbedtls_entropy_init(&entropy);
mbedtls_ctr_drbg_init(&ctr_drbg);
mbedtls_ctr_drbg_seed(&ctr_drbg, mbedtls_entropy_func, &entropy, ...);
mbedtls_pk_sign(&pk, md_alg, hash, hash_len, sig, sig_size, &sig_len,
                mbedtls_ctr_drbg_random, &ctr_drbg);   // <-- f_rng, p_rng
mbedtls_ctr_drbg_free(&ctr_drbg);
mbedtls_entropy_free(&entropy);

// 新（1.0）—— 用 PSA，无 RNG 参数
psa_crypto_init();   // 一次性，内部播种
psa_sign_message(key_id, alg, msg, msg_len, sig, sig_size, &sig_len);
```

受影响函数（来自 `docs/1.0-migration-guide.md` "Function prototype changes"）：`mbedtls_pk_sign`、`mbedtls_pk_sign_restartable`、`mbedtls_pk_sign_ext`、`mbedtls_pk_check_pair`、`mbedtls_pk_parse_key`、`mbedtls_pk_parse_keyfile`、`mbedtls_lms_generate_private_key`、`mbedtls_lms_sign` 等都去掉了 `f_rng, p_rng`。要随机数时直接 `psa_generate_random()`。

> 若必须与仍带 `f_rng` 的第三方/内部代码桥接：`#include <mbedtls/psa_util.h>`，传 `mbedtls_psa_get_random` 作为 `f_rng`、`MBEDTLS_PSA_RANDOM_STATE` 作为 `p_rng`。

### 3. 配置文件拆分：`MBEDTLS_xxx_C` → `PSA_WANT_*`

机制可用性改由 `include/psa/crypto_config.h` 的 `PSA_WANT_*` 控制（来自 `docs/psa-transition.md` "Cryptographic mechanism availability"）。自动翻译旧配置的方法：

```bash
# 用 Mbed TLS 3.6（不是 TF-PSA-Crypto）构建后运行：
programs/test/query_compile_time_config -l | sed -n 's/^\(PSA_WANT_.*\)=1/#define \1/p'
# 把输出粘贴进 include/psa/crypto_config.h，再删除旧的 MBEDTLS_xxx_C 密码行
```

条件编译也要改写：

```c
// 旧（3.x）
#if defined(MBEDTLS_AES_C) && defined(MBEDTLS_CIPHER_MODE_CBC) && defined(MBEDTLS_CIPHER_PADDING_PKCS7)

// 新（1.0）
#if PSA_WANT_KEY_TYPE_AES && PSA_WANT_ALG_CBC_PKCS7
```

### 4. 错误码合并：`MBEDTLS_ERR_*` → `PSA_ERROR_*`

1.0 起错误码空间合并。返回 `int` 的 `mbedtls_*` 函数也可能返回 `PSA_ERROR_*`。常用映射（节选自 `docs/1.0-migration-guide.md` "Simplified legacy error codes"）：

| 旧 `MBEDTLS_ERR_*` | 新 `PSA_ERROR_*` |
|---|---|
| `MBEDTLS_ERR_CIPHER_INVALID_PADDING` | `PSA_ERROR_INVALID_PADDING` |
| `MBEDTLS_ERR_CIPHER_AUTH_FAILED` / `MBEDTLS_ERR_CCM_AUTH_FAILED` / `MBEDTLS_ERR_GCM_AUTH_FAILED` / `MBEDTLS_ERR_CHACHAPOLY_AUTH_FAILED` | `PSA_ERROR_INVALID_SIGNATURE` |
| `MBEDTLS_ERR_ECP_VERIFY_FAILED` / `MBEDTLS_ERR_ECP_SIG_LEN_MISMATCH` | `PSA_ERROR_INVALID_SIGNATURE` |
| `MBEDTLS_ERR_RSA_INVALID_PADDING` | `PSA_ERROR_INVALID_PADDING` |
| `MBEDTLS_ERR_RSA_OUTPUT_TOO_LARGE` / `MBEDTLS_ERR_MPI_BUFFER_TOO_SMALL` / `MBEDTLS_ERR_ECP_BUFFER_TOO_SMALL` | `PSA_ERROR_BUFFER_TOO_SMALL` |
| `MBEDTLS_ERR_PLATFORM_HW_ACCEL_FAILED` | `PSA_ERROR_HARDWARE_FAILURE` |
| `MBEDTLS_ERR_PLATFORM_FEATURE_UNSUPPORTED` / `MBEDTLS_ERR_OID_NOT_FOUND` | `PSA_ERROR_NOT_SUPPORTED` |
| 各 `*_BAD_INPUT_DATA` / `*_ALLOC_FAILED` | `PSA_ERROR_INVALID_ARGUMENT` / `PSA_ERROR_INSUFFICIENT_MEMORY` |

不再返回两个错误码之和；`mbedtls_low_level_strerr()`/`mbedtls_high_level_strerr()` 已删除；`mbedtls_strerror()` 不再提供（移到 Mbed TLS 侧）。数值化排错用 `programs/psa/psa_constant_names status <n>`。

### 5. 对称加密：`mbedtls_cipher_*` / `mbedtls_aes_*` → `psa_cipher_*`

旧 `cipher.h` 已移除。旧 `mbedtls_cipher_setup`+`setkey`+`set_iv`+`update`+`finish` 的多段流程对应 PSA 的 setup→generate_iv/set_iv→update→finish（见 `recipes/cipher.md`）。

| 旧（3.x） | 新（1.0） |
|---|---|
| `mbedtls_cipher_context_t` + `mbedtls_cipher_init/free` | `psa_cipher_operation_t` = `PSA_CIPHER_OPERATION_INIT`（无 free，出错/结束调 `psa_cipher_abort`） |
| `mbedtls_cipher_setup` + `setkey` + `set_padding_mode` | `psa_cipher_encrypt_setup`/`psa_cipher_decrypt_setup`（密钥改由 `psa_key_id_t` 传入，不再 setkey） |
| `mbedtls_cipher_set_iv` | `psa_cipher_set_iv`（解密）/ `psa_cipher_generate_iv`（加密随机 IV） |
| `mbedtls_cipher_update` | `psa_cipher_update` |
| `mbedtls_cipher_finish` | `psa_cipher_finish` |
| `mbedtls_cipher_crypt`（一次性） | `psa_cipher_encrypt`/`psa_cipher_decrypt`（输出含/要求前置 IV） |
| `mbedtls_cipher_reset` | `psa_cipher_abort`（重用须重新 setup） |

模式选择宏：旧 `MBEDTLS_CIPHER_MODE_CBC`+`MBEDTLS_CIPHER_PADDING_PKCS7` → 新 `PSA_ALG_CBC_PKCS7`；`MBEDTLS_CIPHER_ID_AES` → `PSA_KEY_TYPE_AES`。块密码不再有 `aes.h`/`aria.h`/`camellia.h`/`des.h`（DES 已移除）。

### 6. 哈希与 MAC：`mbedtls_md_*` → `psa_hash_*` / `psa_mac_*`

`md.h` 在 1.x 仅保留哈希薄包装，**不再支持 HMAC** —— 必须迁到 PSA。

哈希算法名对照（来自 `docs/psa-transition.md` "Hash mechanism selection"）：

| 旧 `MBEDTLS_MD_*` | 新 `PSA_ALG_*` |
|---|---|
| `MBEDTLS_MD_MD5` | `PSA_ALG_MD5` |
| `MBEDTLS_MD_SHA1` | `PSA_ALG_SHA_1`（注意下划线） |
| `MBEDTLS_MD_SHA224` / `SHA256` / `SHA384` / `SHA512` | `PSA_ALG_SHA_224` / `_256` / `_384` / `_512` |
| `MBEDTLS_MD_SHA3_224` ... `_512` | `PSA_ALG_SHA3_224` ... `_512` |
| `MBEDTLS_MD_RIPEMD160` | `PSA_ALG_RIPEMD160` |

函数对照：

| 旧（3.x） | 新（1.0） |
|---|---|
| `mbedtls_md`（一次性） | `psa_hash_compute`（验签用 `psa_hash_compare`，替代手动 `memcmp`/`mbedtls_ct_memcmp`） |
| `mbedtls_md_setup`(`hmac=0`)+`starts`+`update`+`finish` | `psa_hash_setup`+`update`+`finish`（操作对象 `psa_hash_operation_t`） |
| `mbedtls_md_clone` | `psa_hash_clone` |
| `mbedtls_md_hmac_starts`+`update`+`finish` + `mbedtls_ct_memcmp` | `psa_mac_sign_setup`+`update`+`sign_finish`（或验签 `psa_mac_verify_setup`+`update`+`verify_finish`，内建常量时间比较） |
| `mbedtls_cipher_cmac_*` | `psa_mac_*`（`PSA_ALG_CMAC` + `PSA_KEY_TYPE_AES` 等） |
| `MBEDTLS_MD_MAX_SIZE` | `PSA_HASH_MAX_SIZE`；动态用 `PSA_HASH_LENGTH(alg)` |

`mbedtls_md_psa_alg_from_type()`/`mbedtls_md_type_from_psa_alg()`（`<mbedtls/psa_util.h>`）可在过渡期做两套枚举互转。

### 7. 非对称签名：`mbedtls_pk_sign` / `mbedtls_rsa_*` / `mbedtls_ecdsa_*` → `psa_sign_*`

| 旧（3.x） | 新（1.0） |
|---|---|
| `mbedtls_pk_sign` / `mbedtls_pk_verify`（已计算 hash） | `psa_sign_hash` / `psa_verify_hash` |
| `mbedtls_pk_sign` / `mbedtls_pk_verify`（对原文） | `psa_sign_message` / `psa_verify_message`（内部 hash+签） |
| `mbedtls_pk_sign_ext`(PSS) / `mbedtls_rsa_rsassa_pss_sign` | `PSA_ALG_RSA_PSS(hash)` |
| `mbedtls_rsa_rsassa_pkcs1_v15_sign` | `PSA_ALG_RSA_PKCS1V15_SIGN(hash)` |
| `mbedtls_ecdsa_sign` / `mbedtls_ecdsa_write_signature` | `PSA_ALG_ECDSA(hash)`（随机）或 `PSA_ALG_DETERMINISTIC_ECDSA(hash)` |
| `mbedtls_pk_parse_key`（PEM/DER） | 解析仍用 PK，再 `mbedtls_pk_get_psa_attributes()` + `mbedtls_pk_import_into_psa()` 得到 `psa_key_id_t` |

**ECDSA 签名格式差异（重要）**：PSA 用 **raw 固定长度** 格式，旧 API 用 **ASN.1 DER**。互转用 `<mbedtls/psa_util.h>` 的 `mbedtls_ecdsa_raw_to_der()` / `mbedtls_ecdsa_der_to_raw()`。

非对称加解密：`mbedtls_pk_encrypt`/`mbedtls_rsa_pkcs1_encrypt` → `psa_asymmetric_encrypt`（`PSA_ALG_RSA_PKCS1V15_CRYPT` 或 `PSA_ALG_RSA_OAEP(hash)`）；解密对应 `psa_asymmetric_decrypt`。

### 8. 密钥协商：`mbedtls_ecdh_*` / `mbedtls_dhm_*` → `psa_raw_key_agreement`

无 `mbedtls_ecdh_context`/`mbedtls_dhm_context` 对应物 —— 改用 `psa_key_id_t` 持有本方私钥：

```c
// 旧（3.x）contextless ECDH
mbedtls_ecp_group grp; mbedtls_mpi our_priv, z; mbedtls_ecp_point our_pub, their_pub;
mbedtls_ecp_group_load(&grp, MBEDTLS_ECP_DP_SECP256R1);
mbedtls_ecdh_gen_public(&grp, &our_priv, &our_pub, f_rng, p_rng);
mbedtls_ecdh_compute_shared(&grp, &z, &their_pub, &our_priv, f_rng, p_rng);

// 新（1.0）
psa_key_attributes_t attr = PSA_KEY_ATTRIBUTES_INIT;
psa_set_key_type(&attr, PSA_KEY_TYPE_ECC_KEY_PAIR(PSA_ECC_FAMILY_SECP_R1));
psa_set_key_bits(&attr, 256);
psa_set_key_algorithm(&attr, PSA_ALG_ECDH);
psa_set_key_usage_flags(&attr, PSA_KEY_USAGE_DERIVE);
psa_key_id_t our_key;
psa_generate_key(&attr, &our_key);
psa_export_public_key(our_key, our_pub, sizeof(our_pub), &our_pub_len);   // 发给对端
psa_raw_key_agreement(PSA_ALG_ECDH, our_key,
                      their_pub, their_pub_len,
                      shared, sizeof(shared), &shared_len);
psa_destroy_key(our_key);
```

曲线对照（节选）：`MBEDTLS_ECP_DP_SECP256R1` → `PSA_ECC_FAMILY_SECP_R1` + 256；`MBEDTLS_ECP_DP_CURVE25519` → `PSA_ECC_FAMILY_MONTGOMERY` + 255；`MBEDTLS_ECP_DP_SECP256K1` → `PSA_ECC_FAMILY_SECP_K1` + 256。FFDH：`MBEDTLS_DHM_RFC7919_FFDHE2048_P_BIN` → `PSA_DH_FAMILY_RFC7919` + 2048，算法 `PSA_ALG_FFDH`。secp192/224 系列已不再支持。

### 9. 密钥派生：HKDF / PBKDF2

`mbedtls_hkdf` / `mbedtls_pkcs5_pbkdf2_hmac` 改走通用 KDF 接口（见 `recipes/key_derivation.md`）：`psa_key_derivation_setup(PSA_ALG_HKDF(PSA_ALG_SHA_256))` → `input_bytes(SALT/SECRET/INFO)` → `output_bytes`/`output_key`。PBKDF2 用 `PSA_ALG_PBKDF2_HMAC(hash)`，并用 `psa_key_derivation_input_cost` 设迭代次数。

### 10. PK 模块与 PAKE 接口变化

PK 仍在（仅做密钥解析/格式化），但（来自 `docs/1.0-migration-guide.md`）：

- `mbedtls_pk_type_t` 已删除；区分 padding 用 `mbedtls_pk_sigalg_t`（`MBEDTLS_PK_SIGALG_RSA_PKCS1V15`/`_RSA_PSS`/`_ECDSA`）。
- `mbedtls_pk_setup_opaque()` 改名 `mbedtls_pk_wrap_psa()`；`MBEDTLS_PK_RSA_ALT` 移除（改用 opaque 驱动）。
- `mbedtls_pk_setup`/`mbedtls_pk_rsa`/`mbedtls_pk_ec` 移除（不再暴露底层 RSA/ECC context）。
- `mbedtls_pk_can_do()` → `mbedtls_pk_can_do_psa()`。
- NIST-KW 的 `mbedtls_nist_kw_wrap/unwrap` 现接收 `mbedtls_svc_key_id_t` 而非自定义 context。

PAKE（EC-JPAKE）接口升级到 PSA 1.2 final（`docs/1.0-migration-guide.md` "Changes to the PAKE interface"）：

```c
// 旧（3.6 beta）
psa_pake_setup(&op, &suite);
psa_pake_set_password_key(&op, key);
psa_pake_cs_set_algorithm(&suite, PSA_ALG_JPAKE);
psa_pake_cs_set_hash(&suite, PSA_ALG_SHA_256);
psa_pake_get_implicit_key(&op, &derivation);

// 新（1.0 final）
psa_pake_setup(&op, key, &suite);                      // key 并入 setup
psa_pake_cs_set_algorithm(&suite, PSA_ALG_JPAKE(PSA_ALG_SHA_256));  // hash 并入算法宏
psa_pake_get_shared_key(&op, &shared_attr, &shared_id); // 替代 get_implicit_key，得到新密钥
```

判断宏改 `PSA_ALG_IS_JPAKE(alg)`。仍仅支持 secp256r1 + SHA-256。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `'mbedtls_aes_init' undefined` 等 | `aes.h`/`sha256.h`/`cipher.h` 等头文件在 1.0 已整体移除 | 改用 `#include <psa/crypto.h>` 与对应 `psa_*` 函数 |
| `psa_crypto_init()` 返回 `PSA_ERROR_INSUFFICIENT_ENTROPY` | 旧 `MBEDTLS_NO_PLATFORM_ENTROPY` 配置未迁移 | 嵌入式平台改用 `MBEDTLS_PSA_DRIVER_GET_ENTROPY`（提供 `mbedtls_platform_get_entropy`）或保持 `MBEDTLS_PSA_BUILTIN_GET_ENTROPY` |
| `mbedtls_pk_sign` 签名结果与对端验签不符 | 旧 ASN.1 DER 格式 vs PSA raw 格式（ECDSA） | 用 `mbedtls_ecdsa_raw_to_der()`/`mbedtls_ecdsa_der_to_raw()` 转换 |
| `PSA_ERROR_NOT_SUPPORTED` 但旧 `MBEDTLS_xxx_C` 明明开着 | `MBEDTLS_xxx_C` 在 1.0 不再控制 PSA 机制可用性 | 在 `crypto_config.h` 启用对应 `PSA_WANT_KEY_TYPE_*`/`PSA_WANT_ALG_*` |
| `f_rng`/`p_rng` 参数编译报错 | 1.0 公开函数不再接受 RNG 回调 | 删除这两个参数；确保已调 `psa_crypto_init()` |
| `TF_PSA_CRYPTO_CONFIG_FILE` 不生效 | 误用了旧宏 `MBEDTLS_PSA_CRYPTO_CONFIG_FILE` | 改用新宏名；删 build 缓存重配 |
| RSA 签名从 PSS 退化为 PKCS1v1.5 | PK 不再携带 padding 策略；`mbedtls_pk_sign` 默认 v1.5 | 显式调 `mbedtls_pk_sign_ext(MBEDTLS_PK_SIGALG_RSA_PSS, ...)` 或直接用 `psa_sign_hash` + `PSA_ALG_RSA_PSS(hash)` |
| 持久化 JPAKE 策略读回异常 | 旧 `PSA_ALG_JPAKE` 编码与新 `PSA_ALG_JPAKE(hash)` 不同 | 查询时显示为 `PSA_ALG_JPAKE_BETA`，仍允许任意 hash 的 JPAKE cipher suite |

## 参考

- `docs/psa-transition.md` — per-algorithm 旧→PSA 翻译总表（对称/hash/MAC/KDF/RSA/ECC/ECDH/FFDH/EC-JPAKE），每个机制含完整多段流程对照（2406 行）
- `docs/1.0-migration-guide.md` — 1.0 破坏性变更：仓库拆分、配置文件拆分、RNG 配置重写、统一错误码表、PK 模块变化、PAKE 接口更新、ASN.1/NIST-KW 函数原型变化
- `programs/psa/crypto_examples.c` — AES-CBC/CTR 分段加解密，演示 `psa_cipher_*_setup`/`generate_iv`/`update`/`finish`/`abort` 的标准 PSA 流程（替代旧 `mbedtls_cipher_*`）
- `programs/psa/key_ladder_demo.c` — HKDF-SHA256 密钥阶梯 + AEAD 包裹，演示 `psa_key_derivation_*` 与一次性 `psa_aead_encrypt`/`psa_aead_decrypt`（替代旧 `mbedtls_hkdf`）
- `programs/psa/aead_demo.c` — AES-GCM/ChaCha20-Poly1305 多段 AEAD（替代旧 `mbedtls_gcm_*`/`mbedtls_chachapoly_*`）
