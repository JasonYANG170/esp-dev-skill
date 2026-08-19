# 产品配置与数据模型定制

> **适用摘要**: 编辑 `product_info.json`（vendor/product/chip/connection 等）与 `data_model.zap`（endpoint/cluster/attribute），改后重跑 Upload Configuration 生成并烧录 `data_model.bin`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-lowcode-matter/resources/`, source/examples in `repos/esp-lowcode-matter/`, and this recipe path `repos/esp-lowcode-matter/recipes/product_configuration.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "改 vendor/product 名"
- "Matter device type id"
- "改数据模型"
- "加 endpoint/cluster"
- "Wi-Fi / Thread 切换"

## 前置条件

| 条件 | 要求 |
|---|---|
| 文件 | `products/<name>/configuration/product_info.json`、`product_config.json`、`data_model_wifi.zap` / `data_model_thread.zap` |
| 工具 | ZAP 编辑器（可选，编辑 zap） |
| 参考文档 | `docs/product_configuration.md` |

## 分步说明

### 1. `product_info.json`（取自 `products/socket`）

```json
{
    "config_version": 3,
    "vendor_id": 65521,
    "product_id": 32768,
    "origin_vendor_id": 65521,
    "origin_product_id": 32768,
    "device_type_id": 266,
    "vendor_name": "Espressif",
    "product_name": "Matter Product",
    "hw_ver": 1,
    "hw_ver_str": "1",
    "chip": "esp32c6",
    "connection_type": "wifi",
    "module": "ESP32-C6-MINI-1",
    "flash_size": "4MB",
    "secure_boot": "enabled",
    "product_type": "socket",
    "solution_type": "low_code"
}
```

字段说明（见 `docs/product_configuration.md`）：
- `vendor_id` / `product_id`：Matter Vendor/Product ID（测试可用 65521/32768 等测试范围）
- `origin_vendor_id` / `origin_product_id`：原始厂商/产品 ID
- `device_type_id`：Matter 设备类型 ID（如插座 266）
- `vendor_name` / `product_name`：在生态 App 中展示给用户
- `chip`：当前必须 `esp32c6`
- `connection_type`：`wifi`（对应 `data_model_wifi.zap`）或 `thread`（对应 `data_model_thread.zap`）
- `module` / `flash_size` / `secure_boot`：硬件与安全配置

### 2. `product_config.json`（测试模式等，取自 `products/socket`）

```json
{
    "config_version": 3,
    "test_mode": [
        { "type": "ezc.test_mode.common", "subtype": 1 },
        { "type": "ezc.test_mode.ble",    "subtype": 1 },
        { "type": "ezc.test_mode.sniffer","subtype": 1, "trigger": 3 },
        { "type": "ezc.test_mode.low_code","subtype": 1, "ssid": "test_low_code_1" }
    ],
    "device_management": true
}
```

`test_mode[].subtype` 会通过 `LOW_CODE_EVENT_TEST_MODE_LOW_CODE` 等事件的 `event_data` 传给应用（见 `recipes/event_handling.md`）。

### 3. 数据模型 `data_model.zap`

- endpoint 0 必须为 root node，应用节点放其它 endpoint
- 用 [ZAP 编辑器](https://github.com/project-chip/zap) 编辑；模板产品的 zap 是好起点
- 启用所需 cluster/attribute 并更新 **feature_map**
- 可启用可选特性，或加自定义 cluster/attribute
- 新增内容会**增加内存**占用，需测试
- 改完更新应用代码（`feature_update_from_system` 的 endpoint/feature 分发）

### 4. 重跑 Upload Configuration

`product_info.json` / `product_config.json` / `*.zap` 任一改动后，必须重新生成 `data_model.bin` 与证书并烧录：

```sh
cd $LOW_CODE_PATH/tools/mfg
export MAC_ADDRESS=<0123456789ABCDEF>
./mfg_low_code.sh $LOW_CODE_PATH/products/$SELECTED_PRODUCT esp32c6 $MAC_ADDRESS
esptool.py write_flash 0xD000  .../output/$MAC_ADDRESS/${MAC_ADDRESS}_esp_secure_cert.bin \
                       0x1F2000 .../output/$MAC_ADDRESS/${MAC_ADDRESS}_fctry.bin
```

QR 码：`.../output/$MAC_ADDRESS/qr_code.png`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| App 显示旧名 | 只改了 json 没重烧 | 重跑 Upload Configuration |
| 新 endpoint 不工作 | app 未按新 endpoint 分发 | 更新 `feature_update_from_system` |
| 数据模型不合规 | 违反 Matter 规范 | 遵守规范：root node 在 endpoint 0，feature_map 正确 |
| 内存不足 | 加了太多 cluster | 精简并测试内存 |
| Thread/Wi-Fi 切换失效 | connection_type 与 zap 不一致 | `connection_type` 与对应 `data_model_*.zap` 匹配 |

## 参考

- `docs/product_configuration.md`
- `products/socket/configuration/product_info.json`、`product_config.json`
- `recipes/create_product.md`、`recipes/getting_started.md`
