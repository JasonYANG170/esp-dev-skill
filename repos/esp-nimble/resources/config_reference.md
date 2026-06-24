# NimBLE 配置项参考

> 本文档区分两种配置来源：
> - **syscfg (`BLE_*`)**：NimBLE 原生 Mynewt 配置，定义在 `nimble/syscfg.yml` 与 `nimble/host/mesh/syscfg.yml`。Mynewt / 自带移植平台直接使用。
> - **Kconfig (`CONFIG_BT_NIMBLE_*`)**：ESP-IDF 集成下的 menuconfig 选项，由 ESP-IDF 的 NimBLE 组件定义。仓库代码（如 `nimble/transport/esp_ipc/`）中引用 `CONFIG_BT_NIMBLE_ROLE_*` 等。具体取值与默认值以 ESP-IDF 版本的 `Component config → Bluetooth → NimBLE` 为准，本文不臆造默认值，仅列出仓库中明确引用的符号。

## GAP 角色（syscfg）

| 符号 | 含义 |
|---|---|
| `BLE_ROLE_CENTRAL` | 启用 Central（中心）角色 |
| `BLE_ROLE_PERIPHERAL` | 启用 Peripheral（外设）角色 |
| `BLE_ROLE_BROADCASTER` | 启用 Broadcaster（广播者）角色 |
| `BLE_ROLE_OBSERVER` | 启用 Observer（观察者）角色 |

> 默认全部启用（值=1）。`nimble/syscfg.yml`。

## 连接与容量（syscfg）

| 符号 | 含义 |
|---|---|
| `BLE_MAX_CONNECTIONS` | 最大并发连接数（默认 1） |
| `BLE_MAX_PERIODIC_SYNCS` | 最大并发周期同步数（默认 1） |
| `BLE_WHITELIST` | 启用白名单（默认 1） |

## 广播 / 扫描特性（syscfg）

| 符号 | 含义 |
|---|---|
| `BLE_EXT_ADV` | 启用扩展广播（默认 0，需 BLE 5.0+ 控制器） |
| `BLE_PERIODIC_ADV` | 启用周期广播 |
| `BLE_MULTI_ADV_INSTANCES` | 额外多广播实例数（总实例数 = 该值 + 1） |
| `BLE_PERIODIC_ADV_SYNC_TRANSFER` | 周期广播同步传输 |
| `BLE_PERIODIC_ADV_SYNC_BIGINFO_REPORTS` | BIGINFO 报告 |
| `BLE_EXT_ADV_MAX_SIZE` | 扩展广播数据最大字节数 |

## 版本

| 符号 | 含义 |
|---|---|
| `BLE_VERSION` | 控制器支持的 Bluetooth 版本（如 50 = BT 5.0） |

## Mesh 配置（syscfg，`nimble/host/mesh/syscfg.yml` 节选）

| 符号 | 含义 |
|---|---|
| `BLE_MESH_PROV` | 启用 provisioning |
| `BLE_MESH_PROV_DEVICE` | 设备端 provisioning（被配网） |
| `BLE_MESH_PROVISIONER` | 启用 Provisioner 角色 |
| `BLE_MESH_PB_ADV` | PB-ADV 承载 |
| `BLE_MESH_PB_GATT` | PB-GATT 承载 |
| `BLE_MESH_GATT` / `BLE_MESH_GATT_SERVER` | Mesh GATT 服务端 |
| `BLE_MESH_PROXY` | GATT Proxy |
| `BLE_MESH_GATT_PROXY` | Proxy 功能 |
| `BLE_MESH_CDB` | 配置数据库（Composition Data） |
| `BLE_MESH_CDB_NODE_COUNT` | CDB 节点数 |
| `BLE_MESH_CDB_SUBNET_COUNT` | CDB 子网数 |
| `BLE_MESH_CDB_APP_KEY_COUNT` | CDB 应用密钥数 |
| `BLE_MESH_SUBNET_COUNT` | 设备子网数 |
| `BLE_MESH_APP_KEY_COUNT` | 设备应用密钥数 |
| `BLE_MESH_MODEL_KEY_COUNT` | 模型密钥数 |
| `BLE_MESH_MODEL_GROUP_COUNT` | 模型组地址数 |
| `BLE_MESH_LABEL_COUNT` | 虚拟标签地址数 |
| `BLE_MESH_CRPL` | 重放保护列表大小 |
| `BLE_MESH_DEFAULT_TTL` | 默认 TTL |
| `BLE_MESH_ADV` | Mesh 广播承载 |
| `BLE_MESH_ADV_BUF_COUNT` | 广播缓冲数 |

> 完整列表见 `nimble/host/mesh/syscfg.yml`。仓库示例 `apps/blemesh` 中使用 `CID_VENDOR 0x05C3`。

## ESP-IDF Kconfig（`CONFIG_BT_NIMBLE_*`）

> 仓库 `nimble/transport/esp_ipc/src/hci_esp_ipc.c` 与 `nimble/transport/esp_ipc_btdm/src/hci_esp_ipc.c` 中明确引用以下符号，由 ESP-IDF 的 NimBLE 组件在 `menuconfig`（`Component config → Bluetooth → NimBLE`）中定义：

| 符号 | 含义 |
|---|---|
| `CONFIG_BT_NIMBLE_ENABLED` | 启用 NimBLE（必须配合 `CONFIG_BT_ENABLED=y`） |
| `CONFIG_BT_NIMBLE_ROLE_PERIPHERAL` | 启用 Peripheral 角色 |
| `CONFIG_BT_NIMBLE_ROLE_CENTRAL` | 启用 Central 角色 |
| `CONFIG_BT_NIMBLE_ROLE_BROADCASTER` | 启用 Broadcaster 角色 |
| `CONFIG_BT_NIMBLE_ROLE_OBSERVER` | 启用 Observer 角色 |
| `CONFIG_BT_NIMBLE_SMSC_ENABLE` | 启用 Secure Connections（LE SC） |
| `CONFIG_BT_NIMBLE_MESH` | 启用 Mesh |
| `CONFIG_BT_NIMBLE_EXT_ADV` | 启用扩展广播 |
| `CONFIG_BT_NIMBLE_MAX_EXT_ADV_INSTANCES` | 扩展广播实例数 |
| `CONFIG_BT_NIMBLE_MAX_CONNECTIONS` | 最大并发连接数 |

> 其它 `CONFIG_BT_NIMBLE_*`（如 MSYS/ACL 缓冲、controller 内存、日志等级等）请直接在 `idf.py menuconfig` 中查看当前 ESP-IDF 版本提供的选项，本 Skill 不臆造默认值。

## 控制器公共地址（syscfg / 地址配置）

来自 `docs/ble_setup/ble_addr.rst`：

```yaml
# Mynewt syscfg.vals 中硬编码公共地址（小端数组）
BLE_PUBLIC_DEV_ADDR: '(uint8_t[6]){0x66, 0x55, 0x44, 0x33, 0x22, 0x11}'
```

运行期随机地址通过 `ble_hs_id_gen_rnd` + `ble_hs_id_set_rnd` 设置（见 `recipes/address_setup.md`）。

## 配置生效说明

- Mynewt 平台：syscfg 在 `sysinit` 阶段生效，构建时由 `newt` 生成 `syscfg.h`。
- ESP-IDF 平台：`CONFIG_BT_NIMBLE_*` 在 `menuconfig` 配置后写入 `sdkconfig`，`idf.py build` 时生成 `sdkconfig.h`；NimBLE 内部的 syscfg 选项由 ESP-IDF 组件根据 Kconfig 映射。修改后需重新 `idf.py build`。
