# 构建与配置（CMake + mbedtls_config.h）

> **适用摘要**: 用 CMake 构建 Mbed TLS（4.x 仅支持 CMake），通过 `mbedtls_config.h` 与 PSA `crypto_config.h` 裁剪库，用 `scripts/config.py` 程序化修改配置，选用 `configs/` 预设。4.x 起不再支持 Make 与 Visual Studio 工程。

## 触发意图

- "怎么编译 mbedTLS"
- "裁剪 TLS 配置"
- "config.py 用法"
- "减小代码体积"
- "CMake 选项"

## 前置条件

| 条件 | 要求 |
|---|---|
| 工具链 | CMake ≥ 3.20.2、C99 编译器（GCC/Clang/MSVC/Armclang）、Python ≥ 3.8 |
| 子模块 | 构建前 `git submodule update --init --recursive`（含 TF-PSA-Crypto） |

## 分步说明

### 1. 基础 CMake 构建

```sh
# 4.x 仅支持 CMake（Make / VS 工程已移除）
mkdir build && cd build
cmake /path/to/mbedtls_source
cmake --build .

# 产物：libmbedtls、libmbedx509、libtfpsacrypto（亦提供旧名 libmbedcrypto）
# 依赖顺序（GNU 链接器）：-lmbedtls -lmbedx509 -ltfpsacrypto

# 构建测试需要 Python；如不需要：
cmake -DENABLE_TESTING=Off /path/to/mbedtls_source

# 构建动态库：
cmake -DUSE_SHARED_MBEDTLS_LIBRARY=On /path/to/mbedtls_source

# 调试构建：
cmake -DCMAKE_BUILD_TYPE=Debug /path/to/mbedtls_source
# 可选类型：Release/Debug/Coverage/ASan/ASanDbg/MemSan/TSan/Check
```

### 2. 作为依赖引入（find_package）

```cmake
# 安装或指定构建目录后，父项目：
find_package(MbedTLS REQUIRED)
add_executable(myapp main.c)
target_link_libraries(myapp PUBLIC
    MbedTLS::mbedtls
    MbedTLS::tfpsacrypto
    MbedTLS::mbedx509)
# 指定查找路径：set(MbedTLS_DIR ${BUILD_DIR}/cmake)
```

### 3. 配置文件结构（4.x 拆分）

4.x 起配置拆分为两个文件：

| 配置项 | 文件 | 内容 |
|---|---|---|
| TLS / X.509 | `include/mbedtls/mbedtls_config.h` | `MBEDTLS_SSL_*`、`MBEDTLS_X509_*`、`MBEDTLS_NET_C` 等 |
| 加密（PSA） | `tf-psa-crypto/include/psa/crypto_config.h` | `PSA_WANT_ALG_*`、`PSA_WANT_KEY_TYPE_*` 等加密能力 |

```sh
# 用 config.py 程序化修改（替代手动编辑）
python3 scripts/config.py --file include/mbedtls/mbedtls_config.h set MBEDTLS_SSL_PROTO_TLS1_3
python3 scripts/config.py --file include/mbedtls/mbedtls_config.h unset MBEDTLS_DEBUG_C
python3 scripts/config.py --help
```

### 4. 使用 configs/ 预设（针对特定场景裁剪）

| 预设 | 适用场景 |
|---|---|
| `configs/config-ccm-psk-tls1_2.h` | 仅 TLS 1.2 + CCM + PSK，极小体积 |
| `configs/config-ccm-psk-dtls1_2.h` | 仅 DTLS 1.2 + CCM + PSK |
| `configs/config-suite-b.h` | Suite B（美国 NSA）合规配置 |
| `configs/config-symmetric-only.h` | 仅对称加密（无 TLS/非对称） |
| `configs/config-thread.h` | Thread 网络协议栈所需子集 |
| `configs/config-tfm.h` | TF-M（Trusted Firmware-M）配置 |

### 5. 嵌入式移植注意

- `mbedtls_config.h` 中关闭依赖文件系统的宏（`MBEDTLS_FS_IO`、`MBEDTLS_ENTROPY_NV_SEED`）以适配无 FS 平台。
- 通过 `platform.h` 移植 `mbedtls_printf`、`mbedtls_exit`、内存分配、时间等。
- 4.x RNG 由 PSA 提供；嵌入式需在 PSA 层接入硬件 TRNG（见 TF-PSA-Crypto 集成文档）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `make` 不可用 | 4.x 移除了 Make | 改用 CMake |
| 子模块缺失编译错 | 未初始化 TF-PSA-Crypto | `git submodule update --init --recursive` |
| 链接顺序错乱 | 库有依赖关系 | GNU 顺序：`-lmbedtls -lmbedx509 -ltfpsacrypto` |
| 配置改了不生效 | 改错文件（加密改到 mbedtls_config） | 加密能力改 `psa/crypto_config.h` |
| 代码体积过大 | 默认配置启用全部 | 用 `configs/` 预设或 `config.py` 裁剪 |

## 参考

- 文档: `README.md`（CMake / 工具版本 / 消费方式）、`docs/4.0-migration-guide.md`（CMake-only）
- 预设: `configs/README.txt`、`configs/*.h`
- 脚本: `scripts/config.py --help`
- CMake 目标见 `CMakeLists.txt`（`ENABLE_TESTING`、`USE_SHARED_MBEDTLS_LIBRARY`、`MBEDTLS_FATAL_WARNINGS`、`GEN_FILES`）
