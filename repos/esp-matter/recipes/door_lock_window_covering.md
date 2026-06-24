# 门锁与窗帘（Door Lock / Window Covering）

> **适用摘要**: 创建 Door Lock 与 Window Covering 两种 device type endpoint，重点说明 Window Covering `config_t` 构造函数如何指定 `EndProductType`，以及 Door Lock 常用属性。

## 触发意图

- "做一把 Matter 门锁"
- "Matter 窗帘 / window_covering"
- "EndProductType 怎么选"
- "door_lock::create"
- "window_covering_device::create"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/door_lock/`；Window Covering 参考 `docs/en/developing.rst` |
| 头文件 | `esp_matter_endpoint_impl.h`（`door_lock` / `window_covering_device`） |

## 分步说明

### 1. Door Lock

Door Lock device type 在 `endpoint::door_lock` 命名空间下：

```cpp
using namespace esp_matter::endpoint;

door_lock::config_t door_lock_config;
endpoint_t *ep = door_lock::create(node, &door_lock_config, ENDPOINT_FLAG_NONE, lock_handle);
uint16_t lock_ep_id = endpoint::get_id(ep);
```

DoorLock cluster 常用属性（来自 `esp_matter_attribute_impl.h` 的 `cluster::door_lock::attribute`）：`lock_state`（`nullable<uint8_t>`）、`lock_type`、`actuator_enabled`、`door_state`、`operating_mode`（带 min/max）、`supported_operating_modes` 等。

在 `app_attribute_update_cb` 里联动锁电机：

```cpp
if (cluster_id == chip::app::Clusters::DoorLock::Id) {
    if (attribute_id == chip::app::Clusters::DoorLock::Attributes::LockState::Id) {
        err = app_driver_lock_set_state(lock_handle, val);   // 驱动上锁/解锁
    }
}
```

### 2. Window Covering：用构造函数指定 EndProductType

`window_covering::config_t` 的构造函数接受一个 `EndProductType`（默认 Roller shade）。**一旦构造就不能再改**。来自 `docs/en/developing.rst`：

```cpp
using namespace esp_matter::endpoint;

// 选 TiltOnlyInteriorBlind
window_covering::config_t wc_config(static_cast<uint8_t>(
    chip::app::Clusters::WindowCovering::EndProductType::kTiltOnlyInteriorBlind));

endpoint_t *ep = window_covering_device::create(node, &wc_config,
                                                ENDPOINT_FLAG_NONE, motor_handle);
```

可用的 `EndProductType` 枚举值（来自 connectedhomeip `WindowCovering` cluster enum）：`kRollerShade`、`kRomanShade`、`kBalloonShade`、`kWovenWoodShade`、``kTiltOnlyInteriorBlind`、`kTiltAndLiftInteriorBlind` 等。

### 3. Window Covering 的 feature 配置

WindowCovering cluster 有 Lift / Tilt / PositionLift / PositionTilt 等 feature。若要在 cluster 层加 feature（而不是用 device type 默认）：

```cpp
using namespace esp_matter::cluster;

window_covering::config_t wc_cfg(static_cast<uint8_t>(
    chip::app::Clusters::WindowCovering::EndProductType::kTiltOnlyInteriorBlind));
wc_cfg.feature_flags = window_covering::feature::lift::get_id();   // 选 Lift feature
cluster_t *cluster = window_covering::create(endpoint, &wc_cfg, CLUSTER_FLAG_SERVER);
```

常用 WindowCovering 属性：`type`、`current_position_lift_percentage`、`current_position_tilt_percentage`、`target_position_lift_percent_100ths`、`operational_status`、`end_product_type`、`mode`。

### 4. 联动驱动并上报位置

电机驱动在到达新位置后，用 `attribute::update()` 写回 `current_position_lift_percentage` / `current_position_tilt_percentage`，让客户端看到进度。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| WindowCovering 行为不对 | EndProductType 选错 | 构造 `config_t` 时传入正确的 `EndProductType` |
| 改 EndProductType 失败 | 构造后不可改 | 重新构造 `config_t` |
| feature 不生效 | feature_flags 没设 / 与 EndProductType 矛盾 | feature 与产品类型保持一致 |
| DoorLock 命令无响应 | `lock_state` 是 nullable | 上报时按 nullable 处理 |
| 位置属性读不到 | 用错了 lift / tilt 属性 | lift 用 `current_position_lift_percentage`，tilt 用 `current_position_tilt_percentage` |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/examples/door_lock/`（门锁示例）
- `D:/esp-skill/espressif-repos/esp-matter/docs/en/developing.rst` — Defining your own data model（window_covering / pump 构造参数）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_endpoint_impl.h`（`door_lock` / `window_covering_device`）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/data_model/legacy/esp_matter_attribute_impl.h`（`cluster::door_lock::attribute` / `cluster::window_covering::attribute`）
