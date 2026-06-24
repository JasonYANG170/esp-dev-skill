# esp-matter 配置项速查（真实 Kconfig）

> 全部取自 `components/esp_matter/Kconfig`。`menuconfig` 路径：`Component config → ESP Matter`。

## 核心数据模型 / 容量

| Kconfig | 默认 | 含义 |
|---|---|---|
| `CONFIG_ESP_MATTER_ENABLE_DATA_MODEL` | y | 使用 esp-matter 数据模型（zap 示例关掉） |
| `CONFIG_ESP_MATTER_ENABLE_GENERATED_DATA_MODEL` | n | 实验性：用脚本生成的 data model 替代 legacy |
| `CONFIG_ESP_MATTER_ENABLE_MATTER_SERVER` | y | start() 时启动 Matter server（client-only 示例关掉） |
| `CONFIG_ESP_MATTER_ENABLE_OPENTHREAD` | y | start() 时初始化并启动 Thread 栈 |
| `CONFIG_ESP_MATTER_ENABLE_OPTIONAL_ATTRIBUTES` | n | 启用可选属性 helper API（仅测试） |
| `CONFIG_ESP_MATTER_MAX_DEVICE_TYPE_COUNT` | 16 | 每 endpoint 最大 device type 数 |
| `CONFIG_ESP_MATTER_ATTRIBUTE_BUFFER_LARGEST` | 259 | 属性读写缓冲区最大尺寸 |
| `CONFIG_ESP_MATTER_NVS_PART_NAME` | "nvs" | Matter 存非易失属性的 NVS 分区名 |
| `CONFIG_ESP_MATTER_DEFERRED_ATTR_PERSISTENCE_TIME_MS` | 3000 | deferred 属性合并写盘的间隔 |
| `CONFIG_ESP_MATTER_MAX_DYNAMIC_ENDPOINT_COUNT` | 16 | 动态 endpoint 上限（aggregator/bridge） |
| `CONFIG_ESP_MATTER_SCENES_TABLE_SIZE` | 16 | Scenes 表大小 |
| `CONFIG_ESP_MATTER_BINDING_TABLE_SIZE` | 10 | Binding 表大小 |
| `CONFIG_ESP_MATTER_MODE_SELECT_CLUSTER_ENDPOINT_COUNT` | 0 | 用 mode_select 的 endpoint 数 |
| `CONFIG_ESP_MATTER_TEMPERATURE_CONTROL_CLUSTER_ENDPOINT_COUNT` | 0 | 用 temperature_control 的 endpoint 数 |
| `CONFIG_CUSTOM_NETWORK_CONFIG` | n | 跳过 Network Commissioning cluster（带外配网） |

## 内存分配策略

`choice ESP_MATTER_MEM_ALLOC_MODE`（数据模型内存来源）：

| 选项 | 含义 |
|---|---|
| `CONFIG_ESP_MATTER_MEM_ALLOC_MODE_INTERNAL`（默认） | 仅内部 DRAM |
| `CONFIG_ESP_MATTER_MEM_ALLOC_MODE_EXTERNAL` | 仅外部 SPIRAM（需 `SPIRAM_USE_CAPS_ALLOC`/`SPIRAM_USE_MALLOC`） |
| `CONFIG_ESP_MATTER_MEM_ALLOC_MODE_DEFAULT` | 走默认 malloc |
| `CONFIG_ESP_MATTER_MEM_ALLOC_MODE_IRAM_8BIT` | 优先 IRAM（需 `ESP32_IRAM_AS_8BIT_ACCESSIBLE_MEMORY`） |

## 四类 Factory Data Provider（每组是 `choice`）

| Provider | Test | Factory | Secure Cert | Custom |
|---|---|---|---|---|
| DAC | `EXAMPLE_DAC_PROVIDER` | `FACTORY_PARTITION_DAC_PROVIDER` | `SEC_CERT_DAC_PROVIDER` | `CUSTOM_DAC_PROVIDER` |
| Commissionable Data | `EXAMPLE_COMMISSIONABLE_DATA_PROVIDER` | `FACTORY_COMMISSIONABLE_DATA_PROVIDER` | `SEC_CERT_COMMISSIONABLE_DATA_PROVIDER` | `CUSTOM_COMMISSIONABLE_DATA_PROVIDER` |
| Device Instance Info | `EXAMPLE_DEVICE_INSTANCE_INFO_PROVIDER` | `FACTORY_DEVICE_INSTANCE_INFO_PROVIDER` | `SEC_CERT_DEVICE_INSTANCE_INFO_PROVIDER` | `CUSTOM_DEVICE_INSTANCE_INFO_PROVIDER` |
| Device Info | `NONE_DEVICE_INFO_PROVIDER` | `FACTORY_DEVICE_INFO_PROVIDER` | — | `CUSTOM_DEVICE_INFO_PROVIDER` |

启用 Factory/Secure Cert 的总开关：

| Kconfig | 含义 |
|---|---|
| `CONFIG_ENABLE_ESP32_FACTORY_DATA_PROVIDER` | 才能让 Factory 选项可见 |
| `CONFIG_ENABLE_ESP32_DEVICE_INSTANCE_INFO_PROVIDER` | 才能让 Factory Device Instance Info 可见 |
| `CONFIG_ENABLE_TEST_SETUP_PARAMS` | 是否保留测试 passcode/discriminator（默认 y；量产关） |

> Factory / Secure Cert / Custom 任一启用时，必须关 `CONFIG_ENABLE_TEST_SETUP_PARAMS`。

## esp_secure_cert 相关

| Kconfig | 含义 |
|---|---|
| `CONFIG_SEC_CERT_DAC_PROVIDER` | 从 esp_secure_cert 分区读 DAC |
| `CONFIG_ESP_SECURE_CERT_DS_PERIPHERAL` | 用 ECDSA/DS 外设（ESP32-H2/P4/C5 支持）；不用 DS 设 n |
| `CONFIG_ENABLE_SET_CERT_DECLARATION_API` | 启用 `SetCertificationDeclaration()` 运行时设 CD |

## Cluster 裁剪（`Select Supported Matter Clusters`）

`CONFIG_SUPPORT_<CLUSTER>_CLUSTER=n` 可排除不用的 cluster 以省 flash。`examples/light/sdkconfig.defaults` 给出完整样板，常见项：

```
CONFIG_SUPPORT_DOOR_LOCK_CLUSTER=n
CONFIG_SUPPORT_WINDOW_COVERING_CLUSTER=n
CONFIG_SUPPORT_THERMOSTAT_CLUSTER=n
CONFIG_SUPPORT_FAN_CONTROL_CLUSTER=n
CONFIG_SUPPORT_OCCUPANCY_SENSING_CLUSTER=n
CONFIG_SUPPORT_MODE_SELECT_CLUSTER=n
CONFIG_SUPPORT_POWER_SOURCE_CLUSTER=n
CONFIG_SUPPORT_AIR_QUALITY_CLUSTER=n
CONFIG_SUPPORT_TEMPERATURE_CONTROL_CLUSTER=n
CONFIG_SUPPORT_ICD_MANAGEMENT_CLUSTER=n
CONFIG_SUPPORT_TIME_SYNCHRONIZATION_CLUSTER=n
# ...（共 80+ 个 CONFIG_SUPPORT_*）
```

> 通用工程建议从 `examples/light/sdkconfig.defaults` 复制裁剪模板，再按需打开用到的 cluster。

## 网络 / 无线（多模芯片）

| Kconfig | 含义 |
|---|---|
| `CONFIG_OPENTHREAD_ENABLED` | 启用 Thread（802.15.4） |
| `CONFIG_ENABLE_WIFI_STATION` | 启用 Wi-Fi station |
| `CONFIG_ENABLE_WIFI_AP` | 启用 Wi-Fi AP（默认 n） |
| `CONFIG_USE_MINIMAL_MDNS` | Thread 用 n，Wi-Fi 用 y |
| `CONFIG_ESP_WIFI_SOFTAP_SUPPORT` | 软 AP 支持（默认 n） |

ESP32-C5/C6 二选一对照表：

| 模式 | `OPENTHREAD_ENABLED` | `ENABLE_WIFI_STATION` | `USE_MINIMAL_MDNS` |
|---|---|---|---|
| Thread | y | n | n |
| Wi-Fi | n | y | y |

## shell / 蓝牙 / 其它

| Kconfig | 含义 |
|---|---|
| `CONFIG_ENABLE_CHIP_SHELL` | 设备端 `matter` 控制台（生产可关省 ~54KB） |
| `CONFIG_OPENTHREAD_CLI` | `matter esp ot_cli` 命令 |
| `CONFIG_BT_ENABLED` / `CONFIG_BT_NIMBLE_ENABLED` | BLE 用于入网 |
| `CONFIG_BT_NIMBLE_ENABLE_CONN_REATTEMPT` | BLE 重连尝试（示例关） |
| `CONFIG_LWIP_IPV6_NUM_ADDRESSES` | 设为 `MAX_FABRIC+1`（fabric 各占一个本地 IPv6） |
| `CONFIG_MBEDTLS_HKDF_C` | Matter 需 HKDF |
| `CONFIG_BUTTON_PERIOD_TIME_MS` / `CONFIG_BUTTON_LONG_PRESS_TIME_MS` | 按键驱动参数 |

## OTA

| Kconfig | 含义 |
|---|---|
| `CONFIG_ENABLE_OTA_REQUESTOR` | 启用 Matter OTA Requestor |
| `CONFIG_ENABLE_ENCRYPTED_OTA` | 启用加密 OTA（需 RSA-3072 私钥） |

## 构建优化

| 环境变量 | 含义 |
|---|---|
| `IDF_CCACHE_ENABLE=1` | 启用 ccache（Matter 构建慢，强烈建议） |
| `ESP_MATTER_DEVICE_PATH` | 指定非默认板子的 device_hal 路径 |
