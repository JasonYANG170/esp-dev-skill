# 库初始化与生命周期管理

> **适用摘要**: 正确初始化 PSA Crypto 库、在程序结束时释放资源。这是使用任何 `psa_*` API 之前必须完成的第一步。

## 触发意图

- "如何初始化 PSA Crypto"
- "psa_crypto_init 怎么用"
- "PSA 库用完怎么释放"
- "psa_crypto_init 失败怎么办"
- "mbedtls_psa_crypto_free"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `#include <psa/crypto.h>` |
| 额外头文件 | 释放时需要 `psa/crypto_extra.h`（声明 `mbedtls_psa_crypto_free`） |
| 构建选项 | `MBEDTLS_PSA_CRYPTO_C` 已启用（默认开启） |
| 参考示例 | `programs/psa/crypto_examples.c`、`programs/psa/hmac_demo.c` |

## 分步说明

### 1. 唯一的头文件

应用代码只需包含一个头文件即可获得所有 PSA 加密机制：

```c
#include <psa/crypto.h>
```

如需版本宏或读取 `crypto_config.h`，再额外包含：

```c
#include "tf-psa-crypto/build_info.h"   /* TF_PSA_CRYPTO_VERSION_STRING == "1.0.0" */
```

### 2. 调用 psa_crypto_init

`psa_crypto_init()` 必须在任何其他 `psa_*` 函数之前调用并成功。返回类型是 `psa_status_t`（`PSA_SUCCESS == 0`），**不是** `int`，因此不能写成 `if (!psa_crypto_init())`。

```c
#include <psa/crypto.h>
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    psa_status_t status = psa_crypto_init();
    if (status != PSA_SUCCESS) {
        printf("psa_crypto_init failed: %d\n", (int) status);
        /* 常见原因: PSA_ERROR_INSUFFICIENT_ENTROPY (RNG 无法播种),
                     PSA_ERROR_HARDWARE_FAILURE */
        return EXIT_FAILURE;
    }

    /* 此后可安全调用任意 psa_* API */
    return EXIT_SUCCESS;
}
```

要点（来自 `include/psa/crypto.h` 中 `psa_crypto_init` 的文档）：

- 可多次调用；首次成功后，后续调用保证返回 `PSA_SUCCESS`。
- 在 `psa_crypto_init()` 之前调用其他 PSA 函数是**未定义行为**，实现通常会返回 `PSA_ERROR_BAD_STATE`。
- 库内部维护一个全局 RNG，`psa_crypto_init` 负责播种它——因此后续 `psa_*` 调用**不需要**再传 RNG 参数。

### 3. 释放资源：mbedtls_psa_crypto_free

需要释放所有与 PSA 加密相关的资源时，调用 `mbedtls_psa_crypto_free()`（声明在 `psa/crypto_extra.h`）：

```c
#include "psa/crypto_extra.h"

mbedtls_psa_crypto_free();   /* 销毁所有易失(volatile)密钥，释放内部状态 */
```

说明：

- `mbedtls_psa_crypto_free()` 销毁所有易失密钥并释放内部缓冲区。**持久密钥**(persistent)保留在存储中。
- 在长期运行的服务中通常不需要调用；但在测试程序或嵌入式系统复位前调用是良好实践。

### 4. 完整生命周期模板

综合 `programs/psa/crypto_examples.c` 与 `programs/psa/hmac_demo.c` 的惯用写法（`PSA_CHECK` + `goto exit`）：

```c
#include <psa/crypto.h>
#include "tf-psa-crypto/build_info.h"
#include "psa/crypto_extra.h"
#include <stdio.h>
#include <stdlib.h>

#define PSA_CHECK(expr)                                       \
    do {                                                      \
        status = (expr);                                      \
        if (status != PSA_SUCCESS) {                          \
            printf("Error %d at line %d: %s\n",               \
                   (int) status, __LINE__, #expr);            \
            goto exit;                                        \
        }                                                     \
    } while (0)

int main(void)
{
    psa_status_t status = PSA_SUCCESS;

    PSA_CHECK(psa_crypto_init());

    /* ...此��执行实际加密操作... */

exit:
    mbedtls_psa_crypto_free();
    return status == PSA_SUCCESS ? EXIT_SUCCESS : EXIT_FAILURE;
}
```

`PSA_CHECK` 宏保证出错时跳转到 `exit`，从而总能执行清理（`*_abort` / `psa_destroy_key` / `mbedtls_psa_crypto_free`）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `PSA_ERROR_BAD_STATE` | 未先调用 `psa_crypto_init` 就用了其他 `psa_*` | 在 `main` 开头调用并检查 `psa_crypto_init()` 的返回值 |
| `PSA_ERROR_INSUFFICIENT_ENTROPY` | RNG 播种失败（缺少熵源/移植层） | 提供正确的熵源或移植 `psa_crypto` 的 RNG 后端 |
| 编译错误：未声明 `mbedtls_psa_crypto_free` | 未包含 `psa/crypto_extra.h` | 加上 `#include "psa/crypto_extra.h"` |
| `if (!psa_crypto_init())` 行为异常 | 把 `psa_status_t` 当成 `int`（成功=0 被当成 false） | 改为 `if (psa_crypto_init() != PSA_SUCCESS)` |
| 程序退出后持久密钥丢失 | 误以为 `mbedtls_psa_crypto_free` 会保留持久密钥 | 持久密钥不受影响；如需删除用 `psa_destroy_key(id)` |

## 参考

- `programs/psa/crypto_examples.c` — `psa_crypto_init()` + `mbedtls_psa_crypto_free()` 的最简示例
- `programs/psa/hmac_demo.c` — `PSA_CHECK` 宏 + `goto exit` 清理模式
- `include/psa/crypto.h` — `psa_crypto_init` 文档（初始化语义、错误码列表）
- `include/psa/crypto_extra.h` — `mbedtls_psa_crypto_free` 声明
- `docs/psa-transition.md` — "General application layout" 一节
