# esp_secure_cert_mgr 示例工程索引

> 全部路径来自仓库 `examples/` 与 `test_apps/` 的真实目录。

| 示例路径 | 类型 | 说明 |
|---|---|---|
| `examples/esp_secure_cert_app/` | 应用 | 读取分区（证书/密钥/DS）、可选写 demo（备份→擦→写→恢复）、签名校验。主程序 `main/app_main.c` |
| `examples/esp_secure_cert_app/main/app_main.c` | 源码 | `test_read_existing_data`（读 + 校验）、`test_ciphertext_validity`（DS 验签）、`test_priv_key_validity`（私钥/ECDSA 验签）、`run_write_demo`（写 TLV） |
| `examples/esp_secure_cert_app/esp_secure_cert_config_examples.csv` | 配置 | 示例 CSV：CA/dev cert + 明文 RSA 私钥（string）+ 自定义数据（string/hex） |
| `examples/esp_secure_cert_app/partitions.csv` | 分区表 | TLV 分区 `esp_secure_cert, 0x3F, , 0xD000, 0x2000, encrypted` |
| `examples/esp_secure_cert_app/sdkconfig.defaults` | 配置 | 自定义分区表（offset 0xc000） |
| `examples/esp_secure_cert_app/sdkconfig.defaults.esp32` | 配置 | ESP32 专用：secure boot + SECURE_VERIFICATION + DS |
| `examples/esp_secure_cert_ota_example/` | 应用 | `esp_secure_cert` 分区 OTA（三种暂存模式 + NVS 恢复） |
| `examples/esp_secure_cert_ota_example/main/app_main.c` | 源码 | `esp_secure_cert_ota_update`、`check_and_recover_staging_partition`、`read_custom_data`、`save/clear_staging_info_to_nvs` |
| `examples/esp_secure_cert_ota_example/main/partition_utils.c` | 源码 | 查找未分配 flash 空间的辅助函数 |
| `examples/esp_secure_cert_ota_example/partitions.csv` | 分区表 | unallocated OTA 模式分区表 |
| `examples/esp_secure_cert_ota_example/partitions_passive_ota.csv` | 分区表 | passive OTA 模式分区表 |
| `examples/esp_secure_cert_ota_example/sdkconfig.ci.direct_ota` | CI 配置 | direct OTA 模式 |
| `examples/esp_secure_cert_ota_example/sdkconfig.ci.passive_ota` | CI 配置 | passive OTA 模式 |
| `examples/esp_secure_cert_ota_example/sdkconfig.ci.unallocated_ota` | CI 配置 | unallocated 模式 |
| `test_apps/` | 测试 | 组件自动化测试 |
| `test_apps/qemu_test/` | 测试 | QEMU 下的签名校验等测试 |

## 关键源码定位

- 读取实现：`srcs/esp_secure_cert_read.c`
- TLV 读取实现：`srcs/esp_secure_cert_tlv_read.c`
- TLV 写入实现：`srcs/esp_secure_cert_tlv_write.c`
- HMAC/派生实现：`srcs/esp_secure_cert_crypto.c`
- 签名校验实现：`srcs/esp_secure_cert_signature_verify.c`
- 工具入口：`tools/configure_esp_secure_cert.py`（依赖 `tools/esp_secure_cert/` 下 `configure_ds.py`、`tlv_format.py`、`tlv_parser.py`、`nvs_format.py`、`custflash_format.py`、`efuse_helper.py` 等）

## 组件引入

- Component Manager：`espressif/esp_secure_cert_mgr`（主页 https://components.espressif.com/component/espressif/esp_secure_cert_mgr ），或工程 `idf_component.yml` 加 `espressif/esp_secure_cert_mgr: "^2.9.2"`。
- Extra component：`git clone` 后在顶层 `CMakeLists.txt` 设 `EXTRA_COMPONENT_DIRS`。
