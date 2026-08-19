# PSA 密码处理器驱动开发与 driver-only 构建

> **适用摘要**: 为 TF-PSA-Crypto 编写并集成 PSA cryptoprocessor 驱动（transparent 加速器 / opaque 安全元件），以及配置 driver-only 构建（某机制只由驱动提供、移除内置实现以省代码体积）。涵盖 JSON 驱动描述文件、自动生成 vs 手动集成的入口点、`PSA_WANT_*` + `MBEDTLS_PSA_ACCEL_*` + `MBEDTLS_xxx_C` 三者关系，并以仓库自带的 ESP AES/SHA 硬件加速驱动 JSON 为实例。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/TF-PSA-Crypto/resources/`, source/examples in `repos/TF-PSA-Crypto/`, and this recipe path `repos/TF-PSA-Crypto/recipes/psa_driver_development.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "PSA 驱动 / cryptoprocessor driver / accelerator"
- "transparent 驱动 vs opaque 驱动"
- "怎么把 AES/SHA 卸载到硬件加速器"
- "driver description JSON 怎么写"
- "driver-only build / 移除内置软件实现省 flash"
- "PSA_WANT 与 MBEDTLS_PSA_ACCEL 与 MBEDTLS_xxx_C 的关系"
- "psa_driver_wrapper / DRIVER_PREFIX_ENABLED"

## 前置条件

| 条件 | 要求 |
|---|---|
| 角色 | 驱动开发者 / 平台集成者（需可改构建配置与驱动分发层源码） |
| 接口状态 | PSA Driver Interface 仍为 **work-in-progress**：仅 `import_key`/`export_public_key`（transparent）等少数入口点支持自动生成；其余需手动编辑分发层（见 `docs/psa-driver-example-and-guide.md`） |
| 参考文档 | `docs/psa-driver-example-and-guide.md`（开发与集成实例）、`docs/driver-only-builds.md`（driver-only 构建）、`docs/proposed/psa-driver-developer-guide.md`（交付物）、`docs/proposed/psa-driver-integration-guide.md`（构建接入） |
| 参考驱动 | `scripts/data_files/driver_jsons/esp_aes_driver.json`、`esp_sha_driver.json`、`mbedtls_test_transparent_driver.json`、`mbedtls_test_opaque_driver.json`、`p256_transparent_driver.json` |

## 分步说明

### 1. 两种驱动类型

来自 `docs/proposed/psa-driver-developer-guide.md` 与 `docs/psa-driver-example-and-guide.md`：

| 类型 | 密钥可见性 | 典型用途 | 分发依据 |
|---|---|---|---|
| **transparent**（透明） | 明文传入（每次操作开头） | 硬件**加速器**、可选的替代软件实现 | 按"算法+密钥类型+长度"组合匹配 |
| **opaque**（不透明） | 仅存在于受保护环境（安全元件/HSM/secure enclave） | 隔离私钥 | 按 key **location**（`psa_set_key_lifetime` 的位置字段）匹配 |

应用代码对二者无感知：透明驱动在可用时**替代**默认软件实现；opaque 驱动在该 location 的密钥被使用时被调用。

### 2. 驱动的三个交付物

来自 `docs/proposed/psa-driver-developer-guide.md` "Deliverables for a driver"：

1. **JSON 驱动描述文件** —— 声明驱动实现的入口点与支持的机制。
2. **C 头文件** —— 定义驱动描述引用的类型（文件名在 JSON 的 `headers` 字段声明）。
3. **目标文件**（或源文件） —— 定义 JSON 声明的入口点函数。

### 3. JSON 驱动描述文件结构（真实示例）

来自仓库自带 `scripts/data_files/driver_jsons/`。ESP AES 硬件加速驱动（`esp_aes_driver.json`）：

```json
{
    "prefix":       "esp_aes",
    "type":         "transparent",
    "mbedtls/h_condition":   "defined(ESP_AES_DRIVER_ENABLED)",
    "headers":      ["../../../port/psa_driver/include/psa_crypto_driver_esp_aes.h",
                     "../../../port/psa_driver/include/psa_crypto_driver_esp_aes_gcm.h"],
    "capabilities": [
        {
            "mbedtls/c_condition": "defined(ESP_AES_DRIVER_ENABLED)",
            "_comment": "This driver will support AES operations using the ESP hardware accelerator.",
            "entry_points": ["cipher_encrypt", "cipher_decrypt", "cipher_encrypt_setup",
                             "cipher_decrypt_setup", "cipher_set_iv", "cipher_update",
                             "cipher_finish", "cipher_abort"],
            "algorithms": ["PSA_ALG_AES_CBC", "PSA_ALG_AES_GCM"],
            "fallback": false
        }
    ]
}
```

ESP SHA 硬件加速驱动（`esp_sha_driver.json`）—— 同结构，`entry_points` 为 `hash_setup/hash_update/hash_finish/hash_abort/hash_compute`，`algorithms` 为 `PSA_ALG_SHA_1/SHA_256/SHA_384/SHA_512`。

字段说明：

| 字段 | 含义 |
|---|---|
| `prefix` | 驱动前缀，所有函数/宏以它开头（如 `esp_aes_`） |
| `type` | `"transparent"` 或 `"opaque"` |
| `mbedtls/h_condition` | 条件包含驱动头文件的预处理表达式（Mbed TLS 扩展） |
| `headers` | 驱动描述所需类型的头文件路径列表 |
| `capabilities[].mbedtls/c_condition` | 条件启用某组分发能力的预处理表达式 |
| `capabilities[].entry_points` | 该能力组实现的入口点（如 `sign_hash`、`cipher_update`） |
| `capabilities[].algorithms` | 支持的算法（决定分发匹配） |
| `capabilities[].fallback` | `true`=驱动不支持时回退到软件；`false`=不回退 |
| `capabilities[].names` | 入口点→自定义函数名重映射（见 `mbedtls_test_transparent_driver.json` 把 `export_public_key` 映射到 `mbedtls_test_transparent_export_public_key`） |

### 4. 哪些入口点支持自动生成 vs 手动集成

来自 `docs/psa-driver-example-and-guide.md`。**仅下表入口点**支持 JSON 自动生成驱动分发层：

| Transparent 驱动 | Opaque 驱动 |
|---|---|
| `import_key` | `import_key` |
| `export_public_key` | `export_public_key` |
| | `export_key` |
| | `copy_key` |
| | `get_builtin_key` |

其余入口点（`cipher_*`、`hash_*`、`aead_*`、`sign_hash`、`verify_hash`、`key_agreement` 等，包括上面 ESP 驱动声明的全部 cipher/hash 入口点）**必须手动编辑分发层**，走下面的 7 步流程。

### 5. 手动集成：7 步流程（来自 psa-driver-example-and-guide.md）

步骤 1、2、3、7 每个驱动做一次；步骤 4、5、6 对**每个**单段入口点或**每个**多段子部分各做一次。

**步骤 1 — 选驱动前缀与启用宏**：例如前缀 `ESP_AES`、宏 `ESP_AES_DRIVER_ENABLED`。构建带驱动时在编译期定义此宏。

**步骤 2 — 在某个驱动头文件里声明驱动存在**：

```c
#if defined(ESP_AES_DRIVER_ENABLED)
#ifndef PSA_CRYPTO_ACCELERATOR_DRIVER_PRESENT
#define PSA_CRYPTO_ACCELERATOR_DRIVER_PRESENT
#endif
// 其它定义
#endif
```

**步骤 3 — 条件包含驱动头文件**：在 `psa_crypto_driver_wrappers.h` 中：

```c
#if defined(ESP_AES_DRIVER_ENABLED)
#include "psa_crypto_driver_esp_aes.h"
#endif
```

**步骤 4 — 定位分发层中的包装函数**：分发层在 `psa_crypto_driver_wrappers.h.jinja` 与 `psa_crypto_driver_wrappers_no_static.c.jinja`。函数名形如 `psa_driver_wrapper_<entry_point>`（例如 `psa_driver_wrapper_sign_hash()`、`psa_driver_wrapper_cipher_encrypt()`）。

**步骤 5 — 编写驱动入口点函数**：命名 `<前缀>_transparent_<entry_point>`（或 `_opaque_`），签名与 wrapper 相同。返回码：`PSA_SUCCESS` 成功；`PSA_ERROR_NOT_SUPPORTED` 表示"输入合法但本驱动不支持"（transparent 返回它可让库回退到下个驱动或软件实现）；其它 `PSA_ERROR_*` 立即上抛。

**步骤 6 — 修改 wrapper 函数**：wrapper 内是按 key location 的 `switch`。transparent 驱动放在 `case PSA_KEY_LOCATION_LOCAL_STORAGE`；opaque 放在对应 location 的独立 `case`。所有驱动调用须包在 `#if defined(PSA_CRYPTO_ACCELERATOR_DRIVER_PRESENT)` 内，单个驱动再包一层 `#if defined(<DRIVER_PREFIX>_ENABLED)`。涉及密钥属性的检查（类型/位数）**必须**在 wrapper 里做（属性标记为 private，库外不可访问）。

```c
/* psa_driver_wrapper_sign_hash() 内，示意（结构来自 psa-driver-example-and-guide.md） */
case PSA_KEY_LOCATION_LOCAL_STORAGE:    /* transparent */
#if defined(PSA_CRYPTO_ACCELERATOR_DRIVER_PRESENT)
#if defined(ESP_AES_DRIVER_ENABLED)     /* 仅作结构示例：AES 无 sign_hash */
    if (/* 能力匹配：算法/密钥类型/长度 */) {
        status = esp_aes_transparent_sign_hash(attributes, key_buffer,
                                               key_buffer_size, alg, hash,
                                               hash_length, signature,
                                               signature_size, signature_length);
        if (status != PSA_ERROR_NOT_SUPPORTED) return status;
    }
#endif
#endif
    break;
```

> 仓库内 p256-m 是完整的"手动集成软件加速器"实例：前缀 `P256`/`p256`，启用宏 `MBEDTLS_PSA_P256M_DRIVER_ENABLED`，入口点 `import_key/export_public_key/generate_key/key_agreement/sign_hash/verify_hash`，入口点函数在 `p256m_driver_entrypoints.[hc]`，错误码翻译 `p256_to_psa_error()`。

**步骤 7 — 构建**：两种方式定义启用宏。

```bash
# 方式 A：编译期 CFLAGS（来自 psa-driver-example-and-guide.md）
cmake -S . -B build -DCMAKE_C_FLAGS="-DESP_AES_DRIVER_ENABLED"
# 或 config.py
python3 scripts/config.py set ESP_AES_DRIVER_ENABLED
```

```bash
# 方式 B：用户配置文件（与默认差异大时）
# 在自定义头文件里 #define ESP_AES_DRIVER_ENABLED，然后：
python3 scripts/config.py set TF_PSA_CRYPTO_USER_CONFIG_FILE "/abs/path/my_config.h"
```

### 6. driver-only 构建：移除内置实现省体积

来自 `docs/driver-only-builds.md`。对每个要"只由驱动提供"的机制，三步配置：

1. 在 `crypto_config.h` **定义**对应 `PSA_WANT_*`（机制在 PSA API 可用）。
2. **定义**对应 `MBEDTLS_PSA_ACCEL_*`（告知 PSA 有加速器）。
3. **注释掉**对应 `MBEDTLS_xxx_C`（不编入内置实现）。

```c
/* 例：SHA-256 只走驱动，不编软件 SHA-256（来自 driver-only-builds.md） */
#define PSA_WANT_ALG_SHA_256              1   // 步骤 1：PSA 侧可用
#define MBEDTLS_PSA_ACCEL_ALG_SHA_256     1   // 步骤 2：声明加速器
// #define MBEDTLS_SHA256_C               // 步骤 3：移除内置
```

> 关键：三宏必须成套。缺 `MBEDTLS_PSA_ACCEL_*` 而注释掉 `MBEDTLS_xxx_C` 会链接失败；缺 `PSA_WANT_*` 则运行时返回 `PSA_ERROR_NOT_SUPPORTED`。

driver-only 支持的机制族（来自 `docs/driver-only-builds.md` "Mechanisms covered"）：哈希（SHA-1/2/3、MD5、RIPEMD160）、ECC（ECDH/ECDSA/EC-JPAKE + 密钥导入导出生成）、FFDH、RSA（PKCS#1 v1.5/v2.1 签名与加密）、AEAD（GCM/CCM + AES/ARIA/Camellia、ChaCha20-Poly1305）、非认证 cipher（ECB/CBC/CTR/CFB/OFB/XTS + AES/ARIA/Camellia）。

ECC 的额外约束（`docs/driver-only-builds.md` "Elliptic-curve cryptography"）：要完全移除 `MBEDTLS_ECP_C`，需所有启用曲线都有对应 `MBEDTLS_PSA_ACCEL_ECC_*` 且所有启用的 `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_*` 都有对应 `MBEDTLS_PSA_ACCEL_KEY_TYPE_ECC_KEY_PAIR_*`；`MBEDTLS_PK_PARSE_EC_COMPRESSED`/`_EXTENDED` 或 `PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_DERIVE` 仍会拖入 `ecp.c` 子集。

### 7. 验证：哪个 AES 实现正在运行

`programs/test/which_aes.c` 打印当前 AES 实现（软件 / AESCE / AESNI 汇编 / AESNI 内建）：

```c
/* programs/test/which_aes.c（节选） */
#define MBEDTLS_DECLARE_PRIVATE_IDENTIFIERS    /* 访问私有标识符 */
#include "mbedtls/private/aes.h"

mbedtls_aes_implementation aes_imp = mbedtls_aes_get_implementation();
switch (aes_imp) {
    case MBEDTLS_AES_IMP_SOFTWARE:       /* 纯软件 */
    case MBEDTLS_AES_IMP_AESCE:          /* Arm Crypto Extension */
    case MBEDTLS_AES_IMP_AESNI_ASM:      /* x86 AESNI 汇编 */
    case MBEDTLS_AES_IMP_AESNI_INTRINSICS: /* x86 AESNI 内建 */
}
```

> 注意该程序顶部的 `MBEDTLS_DECLARE_PRIVATE_IDENTIFIERS`：这是私有访问标记，仅示例程序可用，**应用代码不应定义它**（见 `docs/1.0-migration-guide.md` "Private declarations"，未来 minor 版本可能移除）。它用于诊断 AES 选用了哪条路径，不用于生产。通用诊断仍用 `PSA_WANT_*` 做编译期检查。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 链接报 `undefined reference to esp_aes_transparent_xxx` | JSON 声明了入口点但没提供实现/没链接 | 实现入口点函数并链接其目标文件/源文件；确认 `headers` 路径正确 |
| 驱动从不被调用 | 没定义启用宏（`ESP_AES_DRIVER_ENABLED` 等） | 步骤 7：用 CFLAGS 或 config.py 定义；确认 `PSA_CRYPTO_ACCELERATOR_DRIVER_PRESENT` 被步骤 2 触发 |
| 驱动调用后无回退、报 `PSA_ERROR_NOT_SUPPORTED` | `fallback: false` 且驱动返回了 NOT_SUPPORTED，或能力 `algorithms` 不含当前算法 | 按需把 `fallback` 改 `true`；或在 JSON `algorithms` 里补齐算法 |
| 移除 `MBEDTLS_xxx_C` 后构建失败 | 缺对应 `MBEDTLS_PSA_ACCEL_*`，或被 `MBEDTLS_NIST_KW_C`/`MBEDTLS_CMAC_C` 等依赖拖住 | 成套定义 `PSA_WANT_*` + `MBEDTLS_PSA_ACCEL_*`；NIST-KW/CMAC 依赖内置 AES，需一并禁用或也加速 |
| opaque 驱动不触发 | 密钥 location 没设到驱动注册的值 | 创建密钥时 `psa_set_key_lifetime(..., PSA_KEY_LIFETIME_FROM_PERSISTENCE_AND_LOCATION(PSA_KEY_PERSISTENCE_VOLATILE, MY_DRIVER_LOCATION))` |
| wrapper 里访问 `attributes` 编译报 private | 密钥属性检查写在驱动入口点而非 wrapper | 把 `psa_get_key_type`/`psa_get_key_bits` 等属性访问移进 wrapper（库内） |
| ECDSA 中断式（restartable）失效 | driver-only ECC 不支持 interruptible 操作 | 如需 `psa_sign_hash_start`+`complete`，保留 `MBEDTLS_ECDSA_C`（见 driver-only-builds.md 限制） |

## 参考

- `docs/psa-driver-example-and-guide.md` — 驱动分发层工作原理、自动生成 vs 手动集成的入口点表、手动集成 7 步流程、p256-m 完整集成实例（含 wrapper 改动代码）
- `docs/driver-only-builds.md` — driver-only 构建总览：`PSA_WANT`+`MBEDTLS_PSA_ACCEL`+`MBEDTLS_xxx_C` 三步法、各机制族（哈希/ECC/FFDH/RSA/cipher/AEAD）支持范围与限制、HMAC/CTR-DRBG 加速、`MBEDTLS_CIPHER_C` 禁用条件
- `docs/proposed/psa-driver-developer-guide.md` — 驱动交付物（JSON+头文件+目标文件）、transparent/opaque 定义、Mbed TLS 扩展字段（`mbedtls/h_condition`、`mbedtls/c_condition`）
- `docs/proposed/psa-driver-integration-guide.md` — 把驱动构建接入 Mbed TLS（`PSA_DRIVERS` 变量、链接驱动对象）
- `scripts/data_files/driver_jsons/esp_aes_driver.json` — ESP AES 硬件加速 transparent 驱动描述（CBC+GCM，8 个 cipher 入口点，`fallback:false`）
- `scripts/data_files/driver_jsons/esp_sha_driver.json` — ESP SHA 硬件加速 transparent 驱动描述（SHA-1/256/384/512，5 个 hash 入口点）
- `scripts/data_files/driver_jsons/mbedtls_test_transparent_driver.json` — 测试用 transparent 驱动（`import_key`/`export_public_key`，含 `names` 重映射与 `fallback:true`）
- `scripts/data_files/driver_jsons/mbedtls_test_opaque_driver.json` — 测试用 opaque 驱动（含 `export_key`/`copy_key`/`get_builtin_key`）
- `scripts/data_files/driver_jsons/p256_transparent_driver.json` — p256-m 软件加速 transparent 驱动描述
- `programs/test/which_aes.c` — 诊断当前 AES 实现（software/AESCE/AESNI asm/AESNI intrinsics）的程序
