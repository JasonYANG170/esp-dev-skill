# esp-matter 示例工程索引（真实路径）

> 路径均相对 `D:/esp-skill/espressif-repos/esp-matter/examples/`。描述取自各示例的 `README.md` 首行。

| 示例路径 | 说明 |
|---|---|
| `examples/light/` | 用 esp-matter 数据模型创建 Extended Color Light 设备（灯具参考工程）；`optimizations.md` 的基准测量对象 |
| `examples/zap_light/` | 用 ZAP 工具生成数据模型的 Color Temperature Light |
| `examples/light_switch/` | On/Off Light Switch，含 Binding cluster 控制远端灯 |
| `examples/generic_switch/` | Generic Switch 设备，演示 Descriptor cluster 的 TagList feature |
| `examples/door_lock/` | Door Lock 设备 |
| `examples/sensors/` | 温度（SHTC3）+ 湿度 + 占用（PIR）传感器，多 endpoint 异步上报 |
| `examples/refrigerator/` | Refrigerator + TemperatureControlledCabinet 设备；见 `recipes/hvac_appliances.md` |
| `examples/room_air_conditioner/` | Room Air Conditioner（OnOff + Thermostat）设备；见 `recipes/hvac_appliances.md` |
| `examples/icd_app/` | 间歇连接��备（ICD，SIT/LIT），仅 ESP32-H2 / ESP32-C6；见 `recipes/icd_device.md` |
| `examples/multiple_on_off_plugin_units/` | 多个 On/Off Plugin Unit，多 endpoint 映射到 GPIO；`optimizations.md` 多 endpoint 内存参考 |
| `examples/all_device_types_app/` | 覆盖所有 esp-matter 支持的 device type 的测试 App；`recipes/hvac_appliances.md` 综合参考 |
| `examples/controller/` | 基础 Matter Commissioner / Controller（ESP32 上运行）；见 `recipes/controller_ondevice.md`，含 `sdkconfig.defaults.otbr`（兼 BR）与 `sdkconfig.defaults.ram_optimization`（SPIRAM，`optimizations.md` 实战） |
| `examples/ota_provider/` | host 端 Matter OTA Provider，配合 OTA Requestor 使用 |
| `examples/thread_border_router/` | Thread Border Router（ESP32-S3 + ESP32-H2 板子）；见 `recipes/thread_border_router.md` |
| `examples/camera/` | Matter Camera，双芯片分割架构 |
| `examples/bridge_apps/zigbee_bridge/` | 把 Zigbee 设备桥接进 Matter（aggregator + 动态 bridged endpoint） |
| `examples/bridge_apps/blemesh_bridge/` | BLE Mesh 设备桥接进 Matter |
| `examples/bridge_apps/esp-now_bridge_light/` | ESP-NOW 灯桥接进 Matter |
| `examples/bridge_apps/esp_rainmaker_bridge/` | RainMaker 设备桥接进 Matter |
| `examples/bridge_apps/bridge_cli/` | 桥接设备的命令行测试工具 |
| `examples/managed_component_light/` | 用 Component Registry 下载 esp_matter 组件（无需全量 esp-matter 环境） |
| `examples/mfg_test_app/` | 制造测试 App，验证模组预配置（pre-provisioning） |
| `examples/light_network_prov/` | Light 网络配网示例（默认指向 esp-rainmaker 仓库） |
| `examples/rainmaker/` | RainMaker Matter 示例集合（指向 esp-rainmaker 仓库） |
| `examples/demo/` | 演示工程 |
| `examples/test_apps/` | 集成测试 App |
| `examples/unit_test_app/` | esp-matter 组件的 Unity 单元测试 |
| `examples/common/` | 示例公共代码：`app_bridge/`、`app_reset/`、`blemesh_platform/`、`external_platform/`、`manufacturing_data_provider/`、`utils/` 等 |

## 选择建议

- 新建灯具 / 开关 / 传感器 / 门锁 → 直接复制对应示例再改
- 多设备桥接 → 从 `examples/bridge_apps/<name>/` 起步
- 生产凭据 / 测试 → 参考 `examples/light/main/app_main.cpp` 的 CD/Provider 代码 + `examples/mfg_test_app/`
- 不想 clone 全量 esp-matter → 用 `examples/managed_component_light/` + `idf.py add-dependency "espressif/esp_matter^1.4.0"`
- 学习所有 device type → `examples/all_device_types_app/`
