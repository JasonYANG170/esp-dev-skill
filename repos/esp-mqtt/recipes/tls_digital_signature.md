# 数字签名外设 TLS 认证（DS peripheral）

> **适用摘要**: 使用 ESP32-S2/S3/C3/C5/C6/H2/P4 内置的 Digital Signature（DS）硬件外设完成 mqtts:// 双向 TLS 认证，私钥永不出硬件；对应 `examples/ssl_ds/`。与 `tls_mutual_auth.md`（PEM 嵌入 cert + key）是两条不同的工作流。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "MQTT 数字签名认证"
- "digital signature MQTT"
- "DS peripheral TLS"
- "esp_secure_cert ds_data"
- "私钥保护在硬件里 / 硬件签名"

## 前置条件

| 条件 | 要求 |
|---|---|
| 目标芯片 | ESP32-S2 / S3 / C3 / C5 / C6 / H2 / P4（含 DS 外设；ESP32 经典款不支持） |
| Kconfig | `CONFIG_MQTT_TRANSPORT_SSL=y`（默认开） |
| 组件依赖 | `espressif/esp_secure_cert_mgr: "^2.0.2"`（`main/idf_component.yml`） |
| 文件 | 服务端 CA 证书（PEM，embed）；设备证书 + RSA 私钥（不在固件里，烧入 `esp_secure_cert` 分区） |
| 工具 | `pip install esp-secure-cert-tool`（provisioning 工具） |
| 参考示例 | `examples/ssl_ds/` |

## 分步说明

### 1. 关键差异：与 mTLS（PEM）工作流对比

`tls_mutual_auth.md` 把 client cert + 私钥 embed 进固件（`.certificate` + `.key`）。DS 外设走完全不同的路径：

- 私钥的明文**从不离开设备** —— DS 外设在硬件里完成 RSA 签名，应用只能通过 efuse 中的 HMAC 密钥间接驱动。
- 设备证书与 DS 上下文存放在独立的 `esp_secure_cert` 分区，运行时用 `esp_secure_cert_get_device_cert()` / `esp_secure_cert_get_ds_ctx()` 读取。
- 客户端配置时 `.key = NULL`，改为 `.ds_data = (void *)ds_data`。

### 2. 生成客户端证书与 RSA 私钥（来自 `examples/ssl_ds/README.md`）

```bash
cd main
openssl genrsa -out client.key
openssl req -out client.csr -key client.key -new
# 把 client.csr 提交到 https://test.mosquitto.org/ssl/index.php 取回 client.crt（作为设备证书）
```

> 注意：CSR 必须包含 Country / Organisation / Common Name，不能用默认值。

### 3. Provisioning：配置 DS 外设并生成 `esp_secure_cert` 分区

安装工具后执行（在项目根目录运行，`--skip_flash` 表示先生成镜像、随后由构建系统烧录）：

```bash
pip install esp-secure-cert-tool

configure_esp_secure_cert.py -p /* Serial port */ \
    --device-cert /* Device cert (client.crt) */ \
    --private-key /* RSA priv key (client.key) */ \
    --target_chip /* esp32s3 | esp32c6 | ... */ \
    --configure_ds \
    --skip_flash
```

执行后会在 `esp_secure_cert_data/esp_secure_cert.bin` 生成分区镜像，构建系统在 `idf.py flash` 时自动按分区名 `esp_secure_cert` 烧录到正确偏移。

### 4. 在顶层 `CMakeLists.txt` 注册分区烧录（来自 `examples/ssl_ds/CMakeLists.txt`）

```cmake
# ...在 project(...) 之后...
# Flash the custom partition named `esp_secure_cert`.
set(partition esp_secure_cert)
idf_build_get_property(project_dir PROJECT_DIR)
set(image_file ${project_dir}/esp_secure_cert_data/${partition}.bin)
partition_table_get_partition_info(offset "--partition-name ${partition}" "offset")
esptool_py_flash_target_image(flash "${partition}" "${offset}" "${image_file}")

# 服务端 CA 用 target_add_binary_data 嵌入（示例用 TEXT 形式）
target_add_binary_data(${CMAKE_PROJECT_NAME}.elf "main/mosquitto.org.crt" TEXT)
```

> 分区表中必须有名为 `esp_secure_cert` 的条目（data 类型）。`esp_secure_cert_mgr` 组件会按该名查找。

### 5. 组件依赖（`main/idf_component.yml`）

```yaml
dependencies:
  espressif/esp_secure_cert_mgr: "^2.0.2"
  espressif/mqtt:
    version: "*"
    override_path: "../../.."   # 仅在 esp-mqtt 仓库内部示例中需要；独立项目用 espressif/mqtt 版本号
```

### 6. 读取设备证书与 DS 上下文（来自 `examples/ssl_ds/main/app_main.c`）

需要 `#include "esp_secure_cert_read.h"`。两个关键调用：

```c
#include "esp_secure_cert_read.h"

/* DS 上下文由 esp_secure_cert 动态分配，不应被释放（esp-tls 持有其句柄） */
esp_ds_data_ctx_t *ds_data = esp_secure_cert_get_ds_ctx();
if (ds_data == NULL) {
    ESP_LOGE(TAG, "Error in reading DS data from NVS");
    vTaskDelete(NULL);
}

char *device_cert = NULL;
uint32_t len = 0;
esp_err_t ret = esp_secure_cert_get_device_cert(&device_cert, &len);
if (ret != ESP_OK) {
    ESP_LOGE(TAG, "Failed to obtain the device certificate");
    vTaskDelete(NULL);
}
```

签名约定（`esp_secure_cert_read.h`）：

| 函数 | 返回 | 说明 |
|---|---|---|
| `esp_ds_data_ctx_t *esp_secure_cert_get_ds_ctx(void)` | 指针成功 / NULL 失败 | 仅在 `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL` 下可用；读取 subtype 0 的 DS 上下文 |
| `esp_err_t esp_secure_cert_get_device_cert(char **buffer, uint32_t *len)` | `ESP_OK` / 错误 | NVS 分区会动态分配，需配对 `esp_secure_cert_free_device_cert()`；cust_flash 返回只读 flash 指针 |
| `void esp_secure_cert_free_ds_ctx(esp_ds_data_ctx_t *ds_ctx)` | void | 释放 DS 上下文内存 |

### 7. 客户端配置（`.key = NULL`，`.ds_data = ds_data`）

```c
extern const uint8_t server_cert_pem_start[] asm("_binary_mosquitto_org_crt_start");
extern const uint8_t server_cert_pem_end[]   asm("_binary_mosquitto_org_crt_end");

const esp_mqtt_client_config_t mqtt_cfg = {
    .broker = {
        .address.uri = "mqtts://test.mosquitto.org:8884",
        .verification.certificate = (const char *)server_cert_pem_start,
    },
    .credentials = {
        .authentication = {
            .certificate = (const char *)device_cert,
            .key = NULL,                         /* 私钥不提供 */
            .ds_data = (void *)ds_data           /* 改用 DS 外设 */
        },
    },
};
esp_mqtt_client_handle_t client = esp_mqtt_client_init(&mqtt_cfg);
esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
esp_mqtt_client_start(client);
```

> `mqtt_client.h` 中 `ds_data` 字段类型为 `void *`，注释明确：客户端**不会** copy 或 free 该指针，由用户管理生命周期。事件处理与普通 mTLS 相同（参见 `event_handling.md`）。

### 8. 编译期条件编译（MbedTLS v4 vs 旧版）

示例 `app_main.c` 按构建后端选择签名实现头文件：

```c
#if CONFIG_MBEDTLS_VER_4_X_SUPPORT
#include "psa_crypto_driver_esp_rsa_ds.h"   /* PSA 驱动路径 */
#else
#include "rsa_sign_alt.h"                    /* 传统 alt 路径 */
#endif
```

这两个头提供 DS 外设与 mbedTLS 的桥接。普通使用方通常无需直接调用其中的函数 —— esp-tls 会在 TLS 握手需要签名时通过 `ds_data` 自动驱动外设。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `Error in reading DS data from NVS`（ds_data 为 NULL） | `esp_secure_cert` 分区未烧录 / 无 DS 上下文 TLV | 重新执行 `configure_esp_secure_cert.py --configure_ds`；确认分区名 `esp_secure_cert` 且烧录成功 |
| `Failed to obtain the device certificate` | 分区中缺 device cert TLV | provisioning 时 `--device-cert` 路径错误；重新生成 `esp_secure_cert.bin` |
| TLS 握手在签名阶段失败 | 目标芯片不支持 DS（如 ESP32 经典款） | 仅 ESP32-S2/S3/C3/C5/C6/H2/P4 可用；换芯片或回退到普通 mTLS |
| 编译报 `esp_ds_data_ctx_t` 未定义 | 未依赖 esp_secure_cert_mgr / IDF 过旧 | `idf_component.yml` 加 `espressif/esp_secure_cert_mgr: "^2.0.2"`；需 IDF>=5.x |
| 构建系统烧不到分区镜像 | 顶层 `CMakeLists.txt` 未注册 `esptool_py_flash_target_image` | 按 step 4 复制 CMake 片段；镜像须在 `esp_secure_cert_data/esp_secure_cert.bin` |
| provisioning 报 RSA 长度不支持 | DS 外设 RSA 长度受限 | 使用 2048（或 3072）位 RSA 私钥；`rsa_length_bits` 字段会写入 `esp_ds_data_ctx_t` |
| `ds_data` 被提前释放 | 在 client 运行期间 free 了上下文 | 不要调用 `esp_secure_cert_free_ds_ctx()` 直到 `esp_mqtt_client_destroy()` 之后 |

## 参考项目

- `examples/ssl_ds/main/app_main.c` — 完整 DS 工作流示例
- `examples/ssl_ds/CMakeLists.txt` — `esp_secure_cert` 分区烧录 CMake 片段
- `examples/ssl_ds/main/idf_component.yml` — `esp_secure_cert_mgr` 依赖声明
- `examples/ssl_ds/README.md` — provisioning 工具步骤与目标芯片
- `docs/en/index.rst`（Authentication 节，`ds_data` 字段说明）
- esp-secure-cert-tool: https://github.com/espressif/esp_secure_cert_mgr/tree/main/tools
- ESP-TLS DS 文档: https://docs.espressif.com/projects/esp-idf/en/latest/esp32s2/api-reference/protocols/esp_tls.html#digital-signature-with-esp-tls
