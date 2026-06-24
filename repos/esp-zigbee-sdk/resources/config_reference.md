# 配置参考（Kconfig / sdkconfig / 分区 / idf_component.yml）

> 全部取自仓库 `examples/` 与 `docs/en/developing.rst`、`faq.rst`。示例私有宏集中在各 `main/<example>.h`。

## 1. `idf_component.yml`（依赖声明）

来自 `examples/home_automation_devices/on_off_light/main/idf_component.yml`：

```yaml
## IDF Component Manager Manifest File
dependencies:
  idf:
    version: ">=5.2"
  espressif/esp-zigbee-lib:
    version: ">=2.0.0"
  # 按需引入示例公共组件（相对路径）：
  light_driver:
    path: "../../../utils/light_driver"
  alarm_timer:
    path: "../../../utils/alarm_timer"
```

新项目添加依赖命令：
```bash
idf.py add-dependency "espressif/esp-zigbee-lib^2.0.0"
```

## 2. 关键 Kconfig / sdkconfig 选项

### IDF 目标
```text
CONFIG_IDF_TARGET="esp32h2"     # 或 esp32c6 / esp32c3 / esp32s3 / esp32p4
CONFIG_SOC_IEEE802154_SUPPORTED=y   # H2/C6 为 y；C3/S3/P4 为未定义
```

> `CONFIG_SOC_IEEE802154_SUPPORTED` 决定能否用 `ESP_ZIGBEE_RADIO_MODE_NATIVE`。各 `main/<example>.h` 用 `#if CONFIG_SOC_IEEE802154_SUPPORTED` 切换 NATIVE / UART_RCP 分支。

### 调试
```text
CONFIG_ZB_DEBUG_MODE=y          # 使用 debug 版本库，输出更多日志（体积更大，需加大 factory 分区）
```

### 低功耗（sleepy 终端，来自 `docs/en/faq.rst`）
```text
CONFIG_PM_ENABLE=y
CONFIG_FREERTOS_USE_TICKLESS_IDLE=y
# 可选：
CONFIG_PM_LIGHT_SLEEP_CALLBACKS=y
CONFIG_ESP_SLEEP_DEBUG=y
```

### 网关（`examples/zigbee_gateway/sdkconfig.defaults*`）
```text
CONFIG_EXAMPLE_CONNECT_WIFI=y            # 或 CONFIG_EXAMPLE_CONNECT_ETHERNET=y
CONFIG_ESP_COEX_SW_COEXIST_ENABLE=y      # 15.4 + Wi-Fi 软件共存
CONFIG_ZIGBEE_GW_RCP_CHIP_ESP32H2=y      # 或 ESP32C6（RCP 目标芯片）
CONFIG_ZIGBEE_GW_AUTO_UPDATE_RCP=y       # 可选：自动升级 RCP 固件
CONFIG_PIN_TO_RCP_TX=...
CONFIG_PIN_TO_RCP_RX=...
CONFIG_PIN_TO_RCP_RESET=...
CONFIG_PIN_TO_RCP_BOOT=...
```

### 按键 / 引脚（示例常用）
```text
CONFIG_GPIO_BOOT_ON_DEVKIT=9      # BOOT 按键 GPIO（按板子调整）
CONFIG_GPIO_EXT1_WAKEUP_SOURCE=9  # 休眠唤醒源
```

### OTA
```text
CONFIG_ZB_DELTA_OTA=y   # 启用 delta OTA（用 esp_delta_ota_ops）
```

## 3. 分区表 `partitions.csv`

### 标准设备（来自 `examples/home_automation_devices/on_off_light/partitions.csv`）
```text
# Name,   Type, SubType, Offset, Size, Flags
nvs,        data, nvs,      ,        0x6000,
otadata,    data, ota,      ,        0x2000,
phy_init,   data, phy,      ,        0x1000,
factory,    app,  factory,  ,        940K,
zb_storage, data, nvs,      ,        16K,
zb_fct,     data, fat,      ,        1K,
```

### 开启 `ZB_DEBUG_MODE` 时（来自 `docs/en/developing.rst`）
```text
factory,    app,  factory,  , 1200K,   # 加大 factory 容纳 debug 库
zb_storage, data, nvs,      , 16K,
zb_fct,     data, fat,      , 1K,
```

> `zb_storage`（NVS，16K）保存 Zigbee 持久化数据；`zb_fct`（FAT，1K）为出厂配置。两者不可省略，且须在 `app_main` 调用 `nvs_flash_init_partition("zb_storage")`。

## 4. 示例私有配置宏（`main/<example>.h`）

每个示例自带一组宏（命名一致，值按设备调整）：

| 宏 | 含义 | 典型值 |
|---|---|---|
| `ESP_ZIGBEE_PRIMARY_CHANNEL_MASK` | BDB 主信道掩码 | `(1U << 13)` |
| `ESP_ZIGBEE_SECONDARY_CHANNEL_MASK` | 次信道掩码 | `0x07FFF800`（11–26 全部） |
| `ESP_ZIGBEE_HA_*_EP_ID` | 设备 endpoint ID | light=10, switch=1, gateway=10 |
| `ESP_ZIGBEE_STORAGE_PARTITION_NAME` | 存储分区名 | `"zb_storage"` |
| `ESP_MANUFACTURER_NAME` | Basic 厂商名（首字节长度） | `"\x09""ESPRESSIF"` |
| `ESP_MODEL_IDENTIFIER` | Basic 型号（首字节长度） | `"\x07"CONFIG_IDF_TARGET` |
| `ESP_ZIGBEE_ZC_CONFIG()` | ZC 设备配置（COORDINATOR + max_children） | `.max_children=10` |
| `ESP_ZIGBEE_ZR_CONFIG()` | ZR 设备配置（ROUTER） | `.max_children=10` |
| `ESP_ZIGBEE_ZED_CONFIG()` | ZED 设备配置（END_DEVICE + ed_timeout + keep_alive） | `EZB_NWK_ED_TIMEOUT_64MIN`, `4000`ms |
| `ESP_ZIGBEE_PLATFORM_CONFIG()` | 平台/无线配置（NATIVE 或 UART_RCP） | 见网关 recipe |
| `ESP_ZIGBEE_DEFAULT_CONFIG()` | 整体 config（device + platform） | — |
| `ESP_ZIGBEE_RCP_CONFIG()` | RCP 升级参数（仅网关） | reset/boot 引脚、波特率 |

## 5. 内存配置（`ezbee/core.h`，可选）

`ezb_config_memory(&mem_cfg)`（`ezb_core_init` 之后）可调整各表容量，0 表示用默认：

```c
typedef struct {
    uint16_t buffer_pool_size;          // 缓冲池
    uint16_t address_table_size;        // 地址表
    uint16_t neighbor_table_size;       // 邻居表
    uint16_t route_table_size;          // 路由表
    uint16_t route_discovery_table_size;
    uint16_t route_record_table_size;
    uint16_t aps_key_pair_set_size;
    uint16_t aps_bind_table_src_size;   // 绑定表源端
    uint16_t aps_bind_table_dst_size;   // 绑定表目的端
} ezb_mem_config_t;
```

## 6. 抓包密钥（`docs/en/developing.rst`）

默认 TC link key（Wireshark 预配置密钥）：
```
5A:69:67:42:65:65:41:6C:6C:69:61:6E:63:65:30:39   (= "ZigbeeAlliance09")
```
