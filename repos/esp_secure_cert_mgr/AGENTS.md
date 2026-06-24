# AGENTS.md — Supplementary Agent Guide

> Core principles、recipe 索引、pitfalls、执行工作流都在 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未涉及的工程约定与工具指引，不重复内容。

## Project Context

- **Language**: C（C++ 可用，头文件有 `extern "C"` 守卫）
- **Target**: Espressif ESP32 系列芯片（ESP32 / ESP32-S2 / ESP32-S3 / ESP32-C2/C3/C5/C6 / ESP32-H2 / ESP32-P4 等）
- **Framework**: ESP-IDF（`idf_component.yml` 声明 `idf >= 4.3`；写入类 API 需 `>= 5.3`）
- **Toolchain**: ESP-IDF 工具链（`idf.py`），CMake 构建系统
- **依赖**: mbedTLS / PSA（DS、ECDSA、PKI 解析），`esp_partition`，`esp_efuse`，`esp_hw_support`（HMAC）

## Component 引入方式

两种方式（来自 `README.md`）：

1. **IDF Component Manager**（推荐）：在工程根 `idf_component.yml` 添加依赖
   ```yaml
   dependencies:
     espressif/esp_secure_cert_mgr: "^2.9.2"
   ```
   组件主页：https://components.espressif.com/component/espressif/esp_secure_cert_mgr

2. **Extra component**：克隆仓库后设置 `EXTRA_COMPONENT_DIRS`
   ```cmake
   # 顶层 CMakeLists.txt
   set(EXTRA_COMPONENT_DIRS path/to/esp_secure_cert_mgr)
   ```

## File Naming & Include Patterns

### 公共头（来自 `include/`）

| 头文件 | 用途 |
|---|---|
| `esp_secure_cert_read.h` | 读取 API（设备/CA 证书、私钥、DS 上下文、key type） |
| `esp_secure_cert_write.h` | 写入 API（append_tlv、batch、HMAC 加密/派生、erase） |
| `esp_secure_cert_write_errors.h` | 写入相关错误码（`0x7000` 段） |
| `esp_secure_cert_tlv_read.h` | TLV 通用读取/迭代/分区映射/完整性校验 |
| `esp_secure_cert_tlv_config.h` | TLV type/subtype 枚举 |
| `esp_secure_cert_crypto.h` | PBKDF2-HMAC-SHA256、ECDSA 校验/公钥计算（HMAC 外设） |
| `esp_secure_cert_signature_verify.h` | 签名校验（Secure Boot V2） |

### 私有头（来自 `private_include/`，应用一般不直接用）

- `esp_secure_cert_config.h`、`esp_secure_cert_tlv_private.h`（含 TLV header/footer 结构、magic、flags 宏、分区类型常量）

### 标准包含顺序

```c
#include "esp_secure_cert_read.h"        // 读
#include "esp_secure_cert_tlv_read.h"    // TLV 通用读取
#ifdef ESP_SECURE_CERT_WRITE_SUPPORT
#include "esp_secure_cert_write.h"       // 写（需 IDF >= 5.3）
#endif
#if CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION
#include "esp_secure_cert_signature_verify.h"
#endif
```

> 写入 API 头文件内含 `#warning` 提示：未定义 `ESP_SECURE_CERT_WRITE_SUPPORT` 时函数不可用。

## Standard Project Structure

```
my_secure_cert_project/
├── CMakeLists.txt
├── partitions.csv                # 含 esp_secure_cert 行
├── sdkconfig.defaults            # CONFIG_ESP_SECURE_CERT_* 等
├── main/
│   ├── CMakeLists.txt
│   └── app_main.c
└── managed_components/           # Component Manager 方式引入时自动生成
    └── espressif__esp_secure_cert_mgr/
```

## Canonical Entry / Init Pattern

`app_main()` 启动时读取凭据的典型顺序（参照 `examples/esp_secure_cert_app/main/app_main.c`）：

```c
#include "esp_log.h"
#include "esp_secure_cert_read.h"
#include "esp_secure_cert_tlv_read.h"
#if CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION
#include "esp_secure_cert_signature_verify.h"
#endif

static const char *TAG = "app";

void app_main(void)
{
#if CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION
    /* 1) 启动时先做签名校验 */
    if (esp_secure_cert_verify_partition_signature(NULL) != ESP_OK) {
        ESP_LOGE(TAG, "secure cert signature FAILED");
    }
#endif

    /* 2) 读取凭据 */
    char *dev_cert = NULL; uint32_t dev_len = 0;
    if (esp_secure_cert_get_device_cert(&dev_cert, &dev_len) == ESP_OK) {
        ESP_LOGI(TAG, "device cert len=%" PRIu32, dev_len);
        esp_secure_cert_free_device_cert(dev_cert);
    }

    /* 3) TLV 列举（调试用） */
    esp_secure_cert_list_tlv_entries();
}
```

> DS 场景用 `esp_secure_cert_get_ds_ctx()` / `esp_secure_cert_free_ds_ctx()`；裸私钥用 `esp_secure_cert_get_priv_key()` / `esp_secure_cert_free_priv_key()`。

## Build Workflow

1. 设置目标：`idf.py set-target <chip>`（如 `esp32c3`、`esp32s3`）
2. 配置：`idf.py menuconfig` → `Component config` → `ESP Secure Cert Manager`
3. 生成并烧录 `esp_secure_cert` 分区：`python tools/configure_esp_secure_cert.py ...`
4. 构建：`idf.py build`
5. 烧录与监视：`idf.py -p PORT flash monitor`

## Codegen Checklist

- [ ] `partitions.csv` 含正确格式的 `esp_secure_cert` 行（TLV 用 `0x3F` + `0x2000` + `encrypted`）
- [ ] 包含 `esp_secure_cert_read.h`；写场景包含 `esp_secure_cert_write.h` 并用 `ESP_SECURE_CERT_WRITE_SUPPORT` 宏守卫
- [ ] 每个 `get_*` 都有对应 `free_*`（device_cert / ca_cert / priv_key / tlv_info）
- [ ] DS 场景：`CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL=y` 且芯片 `SOC_DIG_SIGN_SUPPORTED`
- [ ] ECDSA 外设：先 `esp_secure_cert_get_priv_key_type()` 判断，再 `get_priv_key_efuse_id()` 取 block
- [ ] HMAC 派生：芯片 `SOC_HMAC_SUPPORTED`，eFuse 已烧 `HMAC_UP` 用途密钥
- [ ] 写入前 `esp_secure_cert_erase_partition()`；生产设备先备份再擦
- [ ] `unmap_partition()` / `tlv_set_partition()` 调用后，所有旧指针视为失效
- [ ] 签名校验：`CONFIG_ESP_SECURE_CERT_SECURE_VERIFICATION=y`（依赖 Secure Boot V2）
- [ ] OTA 升级：复制完成后 `esp_secure_cert_tlv_set_partition(NULL)` 复位

## Do Not Modify

- `espressif-repos/esp_secure_cert_mgr/` 仓库源码（`include/`、`private_include/`、`srcs/`、`tools/`）——只读引用
- `SKILL.md` 的 YAML frontmatter —— Skill 元数据
