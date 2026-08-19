# 构建与配置（CMake / crypto_config.h）

> **适用摘要**: 使用 CMake 构建 TF-PSA-Crypto（静态/共享库、子项目、`find_package`），通过 `include/psa/crypto_config.h` 或 `configs/` 预设选择启用的加密机制，以及用 `scripts/config.py` 编程式编辑配置。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/TF-PSA-Crypto/resources/`, source/examples in `repos/TF-PSA-Crypto/`, and this recipe path `repos/TF-PSA-Crypto/recipes/build_and_config.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "怎么编译 TF-PSA-Crypto"
- "CMake 构建 PSA Crypto"
- "crypto_config.h 怎么改"
- "PSA_WANT_* 配置"
- "configs/ 预设"
- "TF_PSA_CRYPTO_CONFIG_FILE"
- "find_package(TF-PSA-Crypto)"
- "build mode / ASan / Check"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工具链 | C 编译器(GCC 5.4+/Clang 3.8+/Arm Compiler 6.21+/MSVC 2019+)、CMake 3.20.2+、Make 或 Ninja |
| Python | 3.8+（生成源码/测试需要；不跑测试可跳过部分） |
| 子模块 | 开发分支需 `git submodule update --init`（拉取 `framework/`） |
| 参考文档 | `README.md`、`configs/README.txt` |

## 分步说明

### 1. 基本构建（独立目录）

来自 `README.md`：

```bash
git clone <TF-PSA-Crypto repo>
cd TF-PSA-Crypto
git submodule update --init            # 开发分支必需；release tarball 已含

mkdir build && cd build
cmake ..
cmake --build .

ctest                                  # 跑测试套件（需 Python）
```

产物：`libtfpsacrypto`（默认静态库）。

> 注意：CMake 的编译器/标志只在**首次**调用时生效。`CC=your_cc make` 之类无效；要改须重新配置：`CC=your_cc cmake ..`。

### 2. 常用 CMake 选项

| 选项 | 作用 |
|---|---|
| `-DENABLE_TESTING=Off` | 不构建测试（无 Python 环境时用） |
| `-DUSE_SHARED_TF_PSA_CRYPTO_LIBRARY=On` | 构建共享库 |
| `-DCMAKE_BUILD_TYPE=Debug/Release/ASan/Check/TSan/...` | 构建模式 |
| `-DTF_PSA_CRYPTO_CONFIG_FILE="/abs/path/my.h"` | 用外部配置文件替代默认 `crypto_config.h` |
| `-LH` | 列出所有可用 CMake 选项 |

构建模式（`README.md`）：`Release`、`Debug`、`ASan`（含 LeakSanitizer）、`ASanDbg`、`MemSan`（实验性，需 clang）、`MemSanDbg`、`Check`（警告即错误）、`TSan`、`TSanDbg`。

### 3. crypto_config.h：机制开关

`include/psa/crypto_config.h` 用 `PSA_WANT_*` 宏控制启用的密钥类型/算法/曲线/组。要启用某机制，取消注释对应行（默认大量已启用）：

```c
/* include/psa/crypto_config.h（节选） */
#define PSA_WANT_ALG_SHA_256             1   /* SHA-256 哈希 */
#define PSA_WANT_ALG_GCM                 1   /* AES-GCM */
#define PSA_WANT_ALG_ECDSA               1   /* ECDSA 签名 */
#define PSA_WANT_KEY_TYPE_AES            1   /* AES 密钥 */
#define PSA_WANT_KEY_TYPE_ECC_KEY_PAIR_GENERATE 1   /* 可生成的 ECC 密钥对 */
#define PSA_WANT_ECC_SECP_R1_256         1   /* secp256r1 曲线 */
```

机制分两类符号（来自 `README.md`）：

- **密钥类型**：`PSA_WANT_KEY_TYPE_*`（如 `PSA_WANT_KEY_TYPE_AES`、`PSA_WANT_KEY_TYPE_RSA_KEY_PAIR_GENERATE`）
- **算法**：`PSA_WANT_ALG_*`（如 `PSA_WANT_ALG_SHA_256`、`PSA_WANT_ALG_GCM`、`PSA_WANT_ALG_HKDF`）
- **曲线/组**：`PSA_WANT_ECC_<FAMILY>_<BITS>`（如 `PSA_WANT_ECC_SECP_R1_256`）、`PSA_WANT_DH_RFC7919_<BITS>`

> 运行时遇到 `PSA_ERROR_NOT_SUPPORTED` 几乎都是这里没启用对应机制。

### 4. 用 configs/ 预设

`configs/` 提供面向特定场景的精简配置（`configs/README.txt`）：

- `configs/crypto-config-ccm-aes-sha256.h` — 仅 AES-CCM + SHA-256
- `configs/crypto-config-symmetric-only.h` — 仅对称密码

用法（外部配置，不污染源码树）：

```bash
cmake -DTF_PSA_CRYPTO_CONFIG_FILE="configs/crypto-config-ccm-aes-sha256.h" \
      -B build-ccmonly -S .
cmake --build build-ccmonly
```

或直接把预设文件覆盖 `include/psa/crypto_config.h`。

### 5. 用 scripts/config.py 编程式编辑

`scripts/config.py`（README 推荐）可 set/get/unset `PSA_WANT_*`：

```bash
python3 scripts/config.py --help
python3 scripts/config.py set PSA_WANT_ALG_SHA3_256
python3 scripts/config.py get  PSA_WANT_KEY_TYPE_AES
```

### 6. 作为子项目 / find_package 消费

**add_subdirectory**（来自 `README.md`）：父 CMake 项目里：

```cmake
add_subdirectory(third_party/TF-PSA-Crypto)
target_link_libraries(my_app PRIVATE tfpsacrypto)
```

**find_package**（来自 `README.md`，构建产物提供 `cmake/` 配置）：

```cmake
# 帮助 CMake 找到包：
set(TF-PSA-Crypto_DIR "${TF_PSA_CRYPTO_BUILD_DIR}/cmake")
find_package(TF-PSA-Crypto REQUIRED)
add_executable(my_app src/main.c)
target_link_libraries(my_app PRIVATE TF-PSA-Crypto::tfpsacrypto)
```

参考示例：`programs/test/cmake_package/CMakeLists.txt`、`programs/test/cmake_package_install/CMakeLists.txt`。

### 7. 生成 API 文档

```bash
cmake -B build -S .
cmake --build build --target tfpsacrypto-apidoc   # 需 Doxygen >= 1.8.14
# 打开 build/apidoc/index.html
```

### 8. 生成的源文件缺失（开发分支）

开发分支的某些文件由脚本生成（release tarball 已含）。生成方法：

```bash
python3 -m pip install --user -r scripts/basic.requirements.txt
python3 framework/scripts/make_generated_files.py
```

或在非 Windows 非交叉编译时，CMake 会自动生成。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `framework/` 为空 | 未初始化子模块 | `git submodule update --init` |
| `PSA_ERROR_NOT_SUPPORTED` | 机制未在 `crypto_config.h` 启用 | 启用对应 `PSA_WANT_*`，重新编译 |
| 改了 `CC`/`CFLAGS` 不生效 | CMake 首次调用后缓存了编译器 | 删除 build 目录重新 `cmake`，或重配 |
| `crypto_config.h` 改了不生效 | 用了 `TF_PSA_CRYPTO_CONFIG_FILE` 指向旧文件 | 确认指向的文件路径；删除 build 缓存重配 |
| 测试构建失败 | 缺 Python 3.8+ 或依赖 | `pip install -r scripts/basic.requirements.txt`，或 `-DENABLE_TESTING=Off` 跳过 |
| 找不到 `TF-PSA-Crypto::tfpsacrypto` | 未设 `TF-PSA-Crypto_DIR` 或 `CMAKE_PREFIX_PATH` | 设 `TF-PSA-Crypto_DIR` 指向构建目录的 `cmake/` |
| 重复定义 `MBEDTLS_xxx` 与 `PSA_WANT_xxx` 冲突 | 在 Mbed TLS 配置里改了 TF-PSA-Crypto 消费的选项 | 只在 `crypto_config.h` 改；Mbed TLS 侧不重复设 |

## 参考

- `README.md` — 构建、工具版本、CMake 用法、Visual Studio 说明
- `configs/README.txt` — 预设配置用法
- `include/psa/crypto_config.h` — 所有 `PSA_WANT_*` 开关（含分组注释）
- `programs/test/cmake_package/CMakeLists.txt`、`programs/test/cmake_package_install/CMakeLists.txt` — `find_package` 消费示例
- `docs/driver-only-builds.md` — 仅驱动构建说明
- `docs/1.0-migration-guide.md` — 从旧版迁移的配置变化
