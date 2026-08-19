# 开关设备与 Binding（On/Off Light Switch）

> **适用摘要**: 创建一个 On/Off Light Switch（client 端 OnOff），用 Binding cluster 把它绑定到远端灯，绑定后通过按键发送 OnOff 命令控制远端灯，并可选订阅灯的状态以同步本机指示灯。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-matter/resources/`, source/examples in `repos/esp-matter/`, and this recipe path `repos/esp-matter/recipes/switches_binding.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "做一个 Matter 开关"
- "light_switch 怎么绑定灯"
- "Binding cluster"
- "on_off_light_switch::create"
- "switch 控制另一个 matter 灯"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/light_switch/` |
| 组件 | `app/clusters/bindings/`（来自 connectedhomeip） |
| 可选 | `CONFIG_SUBSCRIBE_TO_ON_OFF_SERVER_AFTER_BINDING`（订阅远端 OnOff） |

## 分步说明

### 1. 创建 On/Off Light Switch endpoint

来自 `examples/light_switch/main/app_main.cpp`。Switch 是 OnOff 的 **client**：

```cpp
using namespace esp_matter::endpoint;
using namespace chip::app::Clusters;

on_off_light_switch::config_t switch_config;
endpoint_t *endpoint = on_off_light_switch::create(node, &switch_config,
                                                   ENDPOINT_FLAG_NONE, switch_handle);
switch_endpoint_id = endpoint::get_id(endpoint);
```

### 2. 追加 Groups cluster（client + server）

switch 示例在 switch endpoint 上同时加 Groups 的 server 和 client：

```cpp
cluster::groups::config_t groups_config;
cluster::groups::create(endpoint, &groups_config, CLUSTER_FLAG_SERVER | CLUSTER_FLAG_CLIENT);
```

### 3. 绑定的本质：写 Binding cluster 属性

绑定不是 API 调用，而是由 commissioner（chip-tool）向 switch 的 Binding cluster 写入一条 `Binding` 结构（含远端 node_id / endpoint / cluster / fabric）。写入后 connectedhomeip 的 `Binding::Table` 会缓存。

绑定示例（chip-tool，把 switch 绑定到灯 node `0x7283` endpoint 1）：

```text
binding write binding 0 '{ "0:STRUCT": [ { "1:U8": 0, "2:U16": 0, "3:U32": 3, "4:NODE": 29265, "5:ENDPOINT": 1, "6:CLUSTER": 6 } ] }' 0x7284 1
```

字段含义（TagNumber）：`1:BindingType`(0=Unicast)、`4:Node`、`5:Endpoint`、`6:Cluster`(6=OnOff)。

### 4. 绑定变化时建立 CASE 并（可选）订阅

来自 `examples/light_switch/main/app_main.cpp` 的 `BindingsChanged` 处理：当绑定表变化时，遍历 `Binding::Table`，对 Unicast 绑定用 `client::connect()` 建立 CASE 会话，并订阅远端 OnOff 属性：

```cpp
if (do_subscribe && event->Type == ...) {
    for (const auto &binding : chip::app::Clusters::Binding::Table::GetInstance()) {
        if (binding.type == chip::app::Clusters::Binding::MATTER_UNICAST_BINDING
            && event->BindingsChanged.fabricIndex == binding.fabricIndex) {
            ReadClientHandle req_handle;
            uint32_t attribute_id = chip::app::Clusters::OnOff::Attributes::OnOff::Id;
            req_handle.attribute_path = {binding.remote, binding.clusterId.value(), attribute_id};
            client::connect(server.GetCASESessionManager(), binding.fabricIndex,
                            binding.nodeId, &req_handle);
        }
    }
    do_subscribe = false;
}
```

### 5. 按键下发 OnOff 命令

switch 的按键回调里，对绑定的远端 endpoint 调用 OnOff 命令（invoke）。示例通过 `InvokeCallback` / `CommandSender` 完成，见 `examples/light_switch/main/app_driver.cpp` 的 `app_driver_button_toggle_cb`。

### 6. `app_attribute_update_cb` 处理本机指示灯

如果订阅了远端 OnOff，远端状态变化会回写到 switch 的本地缓存属性；在回调里联动本机 LED 指示灯即可同步显示远端灯的开/关。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 绑定后命令发不出 | Binding 属性没写成功 / fabric 不一致 | 用 chip-tool 确认 `binding read binding 0 <switch_node> 1` 能读到 |
| 订阅不上 | `do_subscribe` 只触发一次 | 重启 switch 或重新写 binding 触发 `BindingsChanged` |
| switch 端缺 OnOff client | 用了 `on_off_light` 而非 `on_off_light_switch` | 用 `on_off_light_switch::create` |
| 命令 fabric 错 | node_id / endpoint 不在 ACL 中 | 把 switch 加入灯的 AccessControlList |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/examples/light_switch/main/app_main.cpp`（`on_off_light_switch::create`、`BindingsChanged` 处理、`client::connect`）
- `D:/esp-skill/espressif-repos/esp-matter/examples/light_switch/README.md`（Bind light to switch 步骤）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h`（`on_off_light_switch` / `dimmer_switch` / `color_dimmer_switch`）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst`（Cluster Control）
