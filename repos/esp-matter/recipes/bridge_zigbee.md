# Bridge 设备（Zigbee / BLE Mesh / ESP-NOW 桥接）

> **适用摘要**: 用 Aggregator + 动态 bridged_node endpoint 把非 Matter 设备（Zigbee、BLE Mesh、ESP-NOW、RainMaker）桥接进 Matter 网络。说明 aggregator 创建、动态 endpoint 恢复、以及 `app_bridge_initialize` 回调机制。

## 触发意图

- "把 Zigbee 设备桥接进 Matter"
- "Matter bridge / aggregator"
- "动态 endpoint"
- "bridged_node::create"
- "matter esp bridge add"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/bridge_apps/zigbee_bridge/`、`blemesh_bridge/`、`esp-now_bridge_light/`、`esp_rainmaker_bridge/`、`bridge_cli/` |
| 组件 | `examples/common/app_bridge/` |

## 分步说明

### 1. 创建 aggregator 作为桥接父端点

Aggregator 是动态 bridged endpoint 的容器。来自 `examples/bridge_apps/zigbee_bridge/main/app_main.cpp`：

```cpp
using namespace esp_matter::endpoint;

aggregator::config_t aggregator_config;
endpoint_t *aggregator = aggregator::create(node, &aggregator_config, ENDPOINT_FLAG_NONE, NULL);
aggregator_endpoint_id = endpoint::get_id(aggregator);
```

> Root Node（endpoint 0）也可作为父端点；但多设备桥接推荐用专门的 aggregator。

### 2. 用 app_bridge 框架管理动态端点

`examples/common/app_bridge/` 提供了统一回调接口。初始化时注册三个回调：

```cpp
err = app_bridge_initialize(node,
                            create_bridge_devices,        // 给定 device_type_id 创建 endpoint
                            create_zigbee_bridged_device, // 拿到 Zigbee 设备时建对应 bridged endpoint
                            free_zigbee_bridged_device);  // 释放
```

- `create_bridge_devices(endpoint_t *ep, uint32_t device_type_id, void *priv_data)`：在动态 endpoint 上按 `device_type_id` 调用对应 `::<device>::create()`（或 `endpoint::add`）。
- `create_zigbee_bridged_device(node_t *node, uint16_t endpoint)`：发现一个 Zigbee 设备时，创建一个 bridged_node endpoint 并关联。

### 3. 动态添加 bridged endpoint

每个被桥接的设备对应一个 `bridged_node` endpoint，其 parent 指向 aggregator。运行时通过设备控制台或桥接逻辑触发：

```text
matter esp bridge add <parent_endpoint_id> <device_type_id>
```

`parent_endpoint_id` 必须是 aggregator 的 endpoint id。代码层面由 `app_bridge` 通过 `endpoint::resume()` / 动态注册完成。

### 4. 属性联动

每个 bridged endpoint 的 `priv_data` 指向桥接层维护的 `app_bridged_device_t`。当 Matter 客户端写该 endpoint 属性时，`app_attribute_update_cb` 里 cast 回 `priv_data`，再把命令翻译成 Zigbee/BLE Mesh/ESP-NOW 的实际下发。

### 5. Zigbee 桥接的额外依赖

`zigbee_bridge` 还需要 `esp-zigbee-lib`（ESP-Zigbee-SDK）。BLE Mesh 桥接需要 `examples/common/blemesh_platform/`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `bridge add` 报错 | parent 不是 aggregator | 先 `aggregator::create`，再用其 endpoint id |
| 客户端发现不到桥接设备 | 动态 endpoint 没注册成功 | 检查 `app_bridge_initialize` 返回值与回调 |
| 属性写进去没下发到 Zigbee | `priv_data` cast 错 / 没翻译协议 | 在 `app_attribute_update_cb` 里正确 cast 并调用 Zigbee 栈 API |
| Zigbee 桥接编译失败 | 缺 esp-zigbee-lib | 按 `zigbee_bridge/README.md` 加依赖 |
| 重启后桥接设备丢失 | 动态 endpoint 没持久化恢复 | `app_bridge_initialize` 已处理恢复；确认 fctry/nvs 分区正常 |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/examples/bridge_apps/zigbee_bridge/main/app_main.cpp`（`aggregator::create`、`app_bridge_initialize`）
- `D:/esp-skill/espressif-repos/esp-matter/examples/common/app_bridge/`（桥接框架）
- `D:/esp-skill/espressif-repos/esp-matter/examples/bridge_apps/blemesh_bridge/`、`esp-now_bridge_light/`、`esp_rainmaker_bridge/`、`bridge_cli/`
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Device console（`matter esp bridge`）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h`（`aggregator` / `bridged_node`）
