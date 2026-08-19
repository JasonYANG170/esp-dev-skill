# 数据模型：Node / Endpoint / Cluster / Attribute / Command

> **适用摘要**: 用 esp-matter 的命名空间 API 从零搭出一个设备数据模型 —— 创建 node、标准 device type endpoint、为 endpoint 追加 cluster、为 cluster 追加 attribute 与 command，并把 `priv_data` 传给回调。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-matter/resources/`, source/examples in `repos/esp-matter/`, and this recipe path `repos/esp-matter/recipes/device_data_model.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "怎么创建一个 Matter 设备数据模型"
- "endpoint / cluster / attribute 怎么加"
- "自定义 command"
- "node::create 怎么用"
- "on_off_light::create"

## 前置条件

| 条件 | 要求 |
|---|---|
| 示例参考 | `examples/light/main/app_main.cpp`、`docs/en/developing.rst` |
| 头文件 | `components/esp_matter/data_model/esp_matter_data_model.h`（node/endpoint/cluster/attribute/command） |

## 分步说明

### 1. 创建 Node（endpoint 0 自带 Root Node）

```cpp
#include <esp_matter.h>
using namespace esp_matter;
using namespace esp_matter::endpoint;

node::config_t node_config;
node_t *node = node::create(&node_config, app_attribute_update_cb, app_identification_cb);
// 第 4 个参数 priv_data 可选；node::create 在 endpoint_impl.h 中签名:
// node_t *create(node_t::config_t *config, attribute::callback_t attribute_update_cb,
//                identification::callback_t identification_cb, void *priv_data = nullptr);
```

### 2. 创建标准 Device Type Endpoint

每个标准 device type 在 `endpoint::<name>` 命名空间下都有 `create()`，签名统一为
`(node_t *node, <name>::config_t *config, uint8_t flags, void *priv_data) -> endpoint_t *`。

```cpp
// On/Off Light
on_off_light::config_t light_cfg;
light_cfg.on_off.on_off = DEFAULT_POWER;
endpoint_t *ep = on_off_light::create(node, &light_cfg, ENDPOINT_FLAG_NONE, /*priv_data*/ nullptr);

// 扩展彩色灯（来自 examples/light）
extended_color_light::config_t ecl_cfg;
ecl_cfg.on_off.on_off = DEFAULT_POWER;
ecl_cfg.on_off_lighting.start_up_on_off = nullptr;
ecl_cfg.level_control.current_level = DEFAULT_BRIGHTNESS;
ecl_cfg.color_control.color_mode = (uint8_t)chip::app::Clusters::ColorControl::ColorMode::kColorTemperature;
endpoint_t *ep = extended_color_light::create(node, &ecl_cfg, ENDPOINT_FLAG_NONE, light_handle);

uint16_t endpoint_id = endpoint::get_id(ep);
```

可选的 device type 命名空间见 `SKILL.md` 的"Standard Device-Type"表与 `esp_matter_endpoint_impl.h`。带构造参数的：

```cpp
// window_covering: 构造函数指定 EndProductType
window_covering::config_t wc_cfg(static_cast<uint8_t>(
    chip::app::Clusters::WindowCovering::EndProductType::kTiltOnlyInteriorBlind));
endpoint_t *ep = window_covering_device::create(node, &wc_cfg, ENDPOINT_FLAG_NONE, nullptr);

// pump: 构造函数指定 max pressure/speed/flow
pump::config_t pump_cfg(1, 10, 20);
endpoint_t *ep = pump::create(node, &pump_cfg, ENDPOINT_FLAG_NONE, nullptr);
```

### 3. 给 Endpoint 追加 Cluster

cluster 命名空间在 `esp_matter_cluster_impl.h`。常用：

```cpp
using namespace esp_matter::cluster;

// on_off（server 端）
on_off::config_t on_off_cfg;
cluster_t *cluster = on_off::create(endpoint, &on_off_cfg, CLUSTER_FLAG_SERVER);

// temperature_measurement
temperature_measurement::config_t tm_cfg;
cluster_t *cluster = temperature_measurement::create(endpoint, &tm_cfg, CLUSTER_FLAG_SERVER);

// 带特性位的 thermostat（O.a+ 一致性）
thermostat::config_t th_cfg;
th_cfg.features.heating.occupied_heating_setpoint = 2200;
th_cfg.feature_flags = thermostat::feature::heating::get_id();
cluster::thermostat::create(endpoint, &th_cfg, CLUSTER_FLAG_SERVER);
```

### 4. 给 Cluster 追加 Attribute / Command

具体的 attribute 创建函数在 `esp_matter_attribute_impl.h`，例如 `cluster::on_off::attribute::create_on_off(cluster, value)`。低层 API：

```cpp
// 追加一个已有标准属性
bool default_global_scene_control = true;
attribute_t *attr = on_off::attribute::create_global_scene_control(cluster, default_global_scene_control);

// 追加一个标准 command
command_t *cmd = on_off::command::create_toggle(cluster);

// 追加 level_control 的 move_to_level 命令
command_t *cmd2 = level_control::command::create_move_to_level(cluster);
```

### 5. （可选）追加自定义 attribute / command（厂商扩展）

```cpp
// 自定义 attribute
uint32_t custom_attr_id = 0x0;
uint16_t default_val = 100;
attribute_t *attr = attribute::create(cluster, custom_attr_id, ATTRIBUTE_FLAG_NONE,
                                      esp_matter_uint16(default_val));

// 自定义 command + 回调
static esp_err_t command_callback(const ConcreteCommandPath &path, TLVReader &tlv_data, void *opaque_ptr) {
    ESP_LOGI(TAG, "Custom command callback");
    return ESP_OK;
}
uint32_t custom_cmd_id = 0x0;
command_t *cmd = command::create(cluster, custom_cmd_id, COMMAND_FLAG_ACCEPTED, command_callback);
```

> 真正的厂商自定义 cluster（带 ZAP codegen、客户端 SDK 支持）见 `recipes/custom_cluster.md`。

### 6. 注册 deferred persistence（频繁变化的属性）

来自 `examples/light/main/app_main.cpp`，对快速变化的属性避免频繁写 flash：

```cpp
attribute_t *cur = attribute::get(light_endpoint_id, LevelControl::Id,
                                  LevelControl::Attributes::CurrentLevel::Id);
attribute::set_deferred_persistence(cur);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `node::create` 返回 nullptr | 回调签名不对 | 用 `attribute::callback_t` / `identification::callback_type_t` 签名 |
| 客户端读不到 endpoint | endpoint 在 `esp_matter::start()` 之后才创建 | 全部 `::create` 必须在 start 之前 |
| 自定义属性读出来是 0 | `esp_matter_uint16` 等构造宏没传对 | 用对应的 `esp_matter_uint16(value)` 而非裸整数 |
| feature_flags 不生效 | cluster 的 O.a+ 一致性没满足 | 至少加一个必需 feature（如 thermostat 的 heating/cooling） |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Developing your Product / Defining your own data model
- `D:/esp-skill/espressif-repos/esp-matter/examples/light/main/app_main.cpp`
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/esp_matter_data_model.h`
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h`
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_cluster_impl.h`
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_attribute_impl.h`
