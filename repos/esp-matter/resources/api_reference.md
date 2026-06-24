# esp-matter API 速���（真实签名）

> 全部函数签名取自仓库头文件，路径见每节"Header"。`node_t` / `endpoint_t` / `cluster_t` / `attribute_t` / `command_t` / `event_t` 为 SDK 不透明句柄。命名空间根为 `esp_matter`。

## Core（启动 / 事件 / 锁）

```cpp
// Header: components/esp_matter/esp_matter_core.h
typedef void (*event_callback_t)(const ChipDeviceEvent *event, intptr_t arg);

bool is_started();
esp_err_t set_server_init_params(chip::CommonCaseDeviceServerInitParams *server_init_params);
esp_err_t start(event_callback_t callback, intptr_t callback_arg = static_cast<intptr_t>(NULL));
esp_err_t factory_reset();
esp_err_t esp_matter_nvs_init();

// RAII 跨任务加锁
namespace lock {
enum status_t { FAILED, ALREADY_TAKEN, SUCCESS };
class ScopedChipStackLock {
public:
    explicit ScopedChipStackLock(uint32_t ticks_to_wait);
    ~ScopedChipStackLock();
};
}
```

## Node

```cpp
// Header: data_model/legacy/esp_matter_endpoint_impl.h
namespace node {
struct config_t;   // 含 descriptor/access_control/basic_information/general_commissioning/
                   //     network_commissioning/general_diagnostics/administrator_commissioning/
                   //     operational_credentials/icd_management ...
node_t *create(config_t *config, attribute::callback_t attribute_update_cb,
               identification::callback_t identification_cb, void *priv_data = nullptr);
}
```

## Endpoint —— 标准 device type create

```cpp
// Header: data_model/legacy/esp_matter_endpoint_impl.h
// 统一签名（device type 名以 namespace 列出）:
//   endpoint_t *create(node_t *node, config_t *config, uint8_t flags, void *priv_data);
//   uint32_t get_device_type_id();
//   esp_err_t add(endpoint_t *endpoint, config_t *config);
namespace endpoint {
  // 灯具
  namespace on_off_light {}
  namespace dimmable_light {}                 // 继承 on_off_light，加 LevelControl
  namespace color_temperature_light {}        // 继承 dimmable_light，加 ColorControl(CT)
  namespace extended_color_light {}           // 继承 color_temperature_light，ColorControl 全功能
  // 开关
  namespace on_off_light_switch {}
  namespace dimmer_switch {}
  namespace color_dimmer_switch {}
  namespace generic_switch {}
  // 其它 device type
  namespace fan {}
  namespace thermostat {}                     // 含 heating/cooling feature conformance
  namespace door_lock {}
  namespace window_covering_device {}         // config_t 构造接受 EndProductType
  namespace pump {}                           // config_t 构造: (max_pressure, max_speed, max_flow)
  namespace pump_controller {}
  // 传感器
  namespace temperature_sensor {}
  namespace humidity_sensor {}
  namespace occupancy_sensor {}
  namespace contact_sensor {}
  namespace light_sensor {}
  namespace pressure_sensor {}
  namespace flow_sensor {}
  namespace air_quality_sensor {}
  // 桥接 / 容器
  namespace aggregator {}
  namespace bridged_node {}
  namespace secondary_network_interface {}    // 多模（Wi-Fi+Thread）设备的第二网络 endpoint
  // 白电
  namespace room_air_conditioner {}
  namespace refrigerator {}
  namespace oven {}
  namespace robotic_vacuum_cleaner {}
  namespace smoke_co_alarm {}
  namespace water_leak_detector {}
  namespace water_freeze_detector {}
  namespace laundry_washer {}
  namespace microwave_oven {}
  namespace extractor_hood {}
  namespace energy_evse {}
  // OTA
  namespace ota_requestor {}
  namespace ota_provider {}
  // 工具
  namespace power_source {}
}

// 低层 endpoint 操作
// Header: data_model/esp_matter_data_model.h
endpoint_t *create(node_t *node, uint8_t flags);                 // 自定义空 endpoint
endpoint_t *get(node_t *node, uint16_t endpoint_id);
endpoint_t *get(uint16_t endpoint_id);
endpoint_t *get_first(node_t *node);
endpoint_t *get_next(endpoint_t *endpoint);
uint16_t    get_count(node_t *node);
uint16_t    get_id(endpoint_t *endpoint);
void *      get_priv_data(uint16_t endpoint_id);
endpoint_t *resume(node_t *node, endpoint::config_t *config, uint8_t flags,
                   uint16_t endpoint_id, void *priv_data);       // 恢复动态 endpoint
```

## Cluster

```cpp
// Header: data_model/legacy/esp_matter_cluster_impl.h
// 每个 cluster namespace:
//   cluster_t *create(endpoint_t *endpoint, config_t *config, uint8_t flags);
// flags: CLUSTER_FLAG_SERVER / CLUSTER_FLAG_CLIENT
namespace cluster {
  namespace on_off {}
  namespace level_control {}
  namespace color_control {}
  namespace temperature_measurement {}
  namespace relative_humidity_measurement {}
  namespace occupancy_sensing {}
  namespace boolean_state {}                 // contact sensor 用
  namespace door_lock {}
  namespace window_covering {}
  namespace fan_control {}
  namespace thermostat {}
  namespace mode_select {}
  namespace groups {}
  namespace binding {}
  namespace descriptor {}
  namespace time_synchronization {}
  namespace pump_configuration_and_control {}
  // ... 全部 cluster 见 esp_matter_cluster_impl.h
}

// 低层
cluster_t *create(endpoint_t *endpoint, uint32_t cluster_id, uint8_t flags);  // 自定义 cluster
cluster_t *get(endpoint_t *endpoint, uint32_t cluster_id);
cluster_t *get_first(endpoint_t *endpoint);
cluster_t *get_next(cluster_t *cluster);
uint32_t   get_id(cluster_t *cluster);
```

## Attribute

```cpp
// Header: data_model/legacy/esp_matter_attribute_impl.h
// 每个 cluster 的 attribute namespace，例：
namespace cluster::on_off::attribute {
  attribute_t *create_on_off(cluster_t *cluster, bool value);
  attribute_t *create_global_scene_control(cluster_t *cluster, bool value);
  attribute_t *create_on_time(cluster_t *cluster, uint16_t value);
  attribute_t *create_off_wait_time(cluster_t *cluster, uint16_t value);
  attribute_t *create_start_up_on_off(cluster_t *cluster, nullable<uint8_t> value);
}
namespace cluster::level_control::attribute {
  attribute_t *create_current_level(cluster_t *cluster, nullable<uint8_t> value);
  attribute_t *create_on_level(cluster_t *cluster, nullable<uint8_t> value);
  attribute_t *create_start_up_current_level(cluster_t *cluster, nullable<uint8_t> value);
  // ...
}
namespace cluster::color_control::attribute {
  attribute_t *create_current_hue(cluster_t *cluster, uint8_t value);
  attribute_t *create_current_saturation(cluster_t *cluster, uint8_t value);
  attribute_t *create_color_mode(cluster_t *cluster, uint8_t value);
  attribute_t *create_color_temperature_mireds(cluster_t *cluster, uint16_t value);
  attribute_t *create_current_x(cluster_t *cluster, uint16_t value);
  attribute_t *create_current_y(cluster_t *cluster, uint16_t value);
  attribute_t *create_start_up_color_temperature_mireds(cluster_t *cluster, nullable<uint16_t> value);
  // ...
}
// global 属性（所有 cluster）
namespace cluster::global::attribute {
  attribute_t *create_cluster_revision(cluster_t *cluster, uint16_t value);
  attribute_t *create_feature_map(cluster_t *cluster, uint32_t value);
}

// 低层 + 工具（Header: esp_matter_attribute_utils.h / esp_matter_data_model.h）
attribute_t *create(cluster_t *cluster, uint32_t attribute_id, uint8_t flags,
                    esp_matter_attr_val_t val);                 // 自定义属性
attribute_t *get(uint16_t endpoint_id, uint32_t cluster_id, uint32_t attribute_id);
attribute_t *get(cluster_t *cluster, uint32_t attribute_id);
attribute_t *get_first(cluster_t *cluster);
attribute_t *get_next(attribute_t *attribute);
uint32_t     get_id(attribute_t *attribute);
esp_err_t    get_val(uint16_t endpoint_id, uint32_t cluster_id, uint32_t attribute_id,
                     esp_matter_attr_val_t *val);
esp_err_t    get_val(attribute_t *attribute, esp_matter_attr_val_t *val);
esp_err_t    update(uint16_t endpoint_id, uint32_t cluster_id, uint32_t attribute_id,
                    esp_matter_attr_val_t *val);                // 触发 PRE/POST_UPDATE，上报
esp_err_t    report(uint16_t endpoint_id, uint32_t cluster_id, uint32_t attribute_id,
                    esp_matter_attr_val_t *val);                // 仅标记脏并上报，不回调
esp_err_t    set_deferred_persistence(attribute_t *attribute);
bool         val_compare(const esp_matter_attr_val_t *v1, const esp_matter_attr_val_t *v2);

// 属性回调
typedef esp_err_t (*callback_t)(callback_type_t type, uint16_t endpoint_id,
                                uint32_t cluster_id, uint32_t attribute_id,
                                esp_matter_attr_val_t *val, void *priv_data);
// callback_type_t: PRE_UPDATE / POST_UPDATE / READ
esp_err_t set_callback(callback_t callback);
```

## Command

```cpp
// Header: data_model/legacy/esp_matter_command_impl.h
// 标准 command，例：
namespace cluster::on_off::command {
  command_t *create_on(cluster_t *cluster);
  command_t *create_off(cluster_t *cluster);
  command_t *create_toggle(cluster_t *cluster);
}
namespace cluster::level_control::command {
  command_t *create_move_to_level(cluster_t *cluster);
  command_t *create_move(cluster_t *cluster);
  command_t *create_step(cluster_t *cluster);
  command_t *create_stop(cluster_t *cluster);
}

// 低层（Header: esp_matter_data_model.h）
// flags: COMMAND_FLAG_ACCEPTED (0x02) / COMMAND_FLAG_GENERATED (0x04)
command_t *create(cluster_t *cluster, uint32_t command_id, uint8_t flags,
                  callback_t callback);                          // 自定义命令
command_t *get_first(cluster_t *cluster);
command_t *get_next(command_t *command);
uint32_t   get_id(command_t *command);

// 自定义命令回调签名（Header: esp_matter_data_model.h）
using chip::app::ConcreteCommandPath;
using chip::TLV::TLVReader;
typedef esp_err_t (*callback_t)(const ConcreteCommandPath &command_path,
                                TLVReader &tlv_data, void *opaque_ptr);
```

## Event

```cpp
// Header: data_model/legacy/esp_matter_event_impl.h + esp_matter_data_model.h
event_t *create(cluster_t *cluster, uint32_t event_id, uint8_t flags);
event_t *get_first(cluster_t *cluster);
event_t *get_next(event_t *event);
uint32_t get_id(event_t *event);

// 真正触发事件记录（connectedhomeip）:
#include <app/EventLogging.h>
chip::app::LogEvent(...);
```

## Identify

```cpp
// Header: components/esp_matter/esp_matter_identify.h
namespace identification {
enum callback_type_t { START, STOP, EFFECT };
typedef esp_err_t (*callback_t)(callback_type_t type, uint16_t endpoint_id,
                                uint8_t effect_id, uint8_t effect_variant, void *priv_data);
esp_err_t set_callback(callback_t callback);
esp_err_t init(uint16_t endpoint_id, uint8_t identify_type,
               uint8_t effect_identifier = chip::app::Clusters::Identify::EffectIdentifierEnum::kBlink,
               uint8_t effect_variant = chip::app::Clusters::Identify::EffectVariantEnum::kDefault);
}
```

## OTA Requestor

```cpp
// Header: components/esp_matter/esp_matter_ota.h
esp_err_t esp_matter_ota_requestor_init(void);
void     esp_matter_ota_requestor_start(void);
#if CONFIG_ENABLE_ENCRYPTED_OTA
esp_err_t esp_matter_ota_requestor_encrypted_init(const char *key, uint16_t size); // RSA-3072 PEM
#endif
esp_err_t esp_matter_ota_requestor_set_config(const esp_matter_ota_config_t &config);

typedef struct {
    uint32_t periodic_query_timeout;            // 默认 86400s
    uint32_t watchdog_timeout;                  // 默认 300s
    const esp_matter_ota_requestor_impl_t *impl;// 默认 nullptr
} esp_matter_ota_config_t;
```

## 自定义 Provider 注入（须在 start 之前）

```cpp
// Header: components/esp_matter/esp_matter_providers.h
esp_err_t set_custom_dac_provider(chip::Credentials::DeviceAttestationCredentialsProvider *provider);
esp_err_t set_custom_commissionable_data_provider(chip::DeviceLayer::CommissionableDataProvider *provider);
esp_err_t set_custom_device_instance_info_provider(...);
esp_err_t set_custom_device_info_provider(...);
```

## Controller（client 端，发送命令/读写/订阅）

```cpp
// Header: components/esp_matter/esp_matter_client.h
namespace client {
// 建立 CASE 会话并发起读写/命令（绑定场景见 examples/light_switch）
esp_err_t connect(case_session_mgr_t *case_session_mgr, uint8_t fabric_index,
                  uint64_t node_id, request_handle_t *req_handle);
esp_err_t cluster_update(uint16_t local_endpoint_id, request_handle_t *req_handle);

typedef struct request_handle {
    // ... attribute_path / cluster_id / command_path / callbacks ...
} request_handle_t;
}
```
> Controller 命令行用 `matter esp controller invoke-cmd / read / write / subscribe`（见 `docs/en/controller.rst`）。`invoke-cmd` 的 `command-data` 用 JSON，键名格式 `"<TagNumber>:<DataType>"`（如 `{"0:U8": 10}`），支持的 DataType 见 `components/esp_matter/utils/jsontlv/json_to_tlv.h`。

## On-device Controller / Commissioner（ESP32 做控制器）

```cpp
// Header: components/esp_matter_controller/core/esp_matter_controller_client.h
namespace esp_matter::controller {
class matter_controller_client {
public:
    using NodeId       = ::chip::NodeId;
    using FabricId     = ::chip::FabricId;
    using MatterDeviceCommissioner = ::chip::Controller::DeviceCommissioner;
    using MatterDeviceController   = ::chip::Controller::DeviceController;

    static matter_controller_client &get_instance();

    // 须在 esp_matter::start() 之后、持 ScopedChipStackLock 调用
    esp_err_t init(NodeId node_id, FabricId fabric_id, uint16_t listen_port);

#ifdef CONFIG_ESP_MATTER_COMMISSIONER_ENABLE
    esp_err_t setup_commissioner();
    MatterDeviceCommissioner *get_commissioner();
    esp_err_t unpair(NodeId remote_node, remove_fabric_callback callback = nullptr);
#else
    esp_err_t setup_controller(chip::MutableByteSpan &ipk,
                               chip::FabricIndex stored_fabric_index = chip::kUndefinedFabricIndex);
    MatterDeviceController *get_controller();
#endif
    chip::FabricIndex get_fabric_index();
};
}
```
关键 Kconfig（`components/esp_matter_controller/Kconfig`）：
- `CONFIG_ESP_MATTER_CONTROLLER_ENABLE`（默认 n）
- `CONFIG_ESP_MATTER_CONTROLLER_VENDOR_ID`（默认 4891）
- `CONFIG_ESP_MATTER_COMMISSIONER_ENABLE`（默认 y，依赖 controller 且 `!ESP_MATTER_ENABLE_MATTER_SERVER`）
- choice `ESP_MATTER_COMMISSIONER_ATTESTATION_TRUST_STORE`：`TEST_ATTESTATION_TRUST_STORE` / `SPIFFS_ATTESTATION_TRUST_STORE` / `DCL_ATTESTATION_TRUST_STORE` / `CUSTOM_ATTESTATION_TRUST_STORE`
- choice `ESP_MATTER_COMMISSIONER_OPERATIONAL_CREDS_ISSUER`：`TEST_OPERATIONAL_CREDS_ISSUER` / `CUSTOM_OPERATIONAL_CREDS_ISSUER`
- 示例 `examples/controller/sdkconfig.defaults`：`CONFIG_ENABLE_CHIP_CONTROLLER_BUILD=y`、`CONFIG_NUM_UDP_ENDPOINTS=16`、`CONFIG_LWIP_IPV6_NUM_ADDRESSES=6`、`CONFIG_CHIP_TASK_STACK_SIZE=15360`。

命令行（`matter esp controller ...`）：
- `pairing onnetwork <node_id> <setup_passcode>`
- `pairing ble-wifi <node_id> <ssid> <password> <pincode> <discriminator>`（esp32s3）
- `pairing ble-thread <node_id> <dataset_tlvs> <pincode> <discriminator>`（esp32s3）
- `pairing code[-wifi|-thread|-wifi-thread] <node_id> ... <setup_payload>`
- `invoke-cmd <node-id|group-id> <endpoint-id> <cluster-id> <command-id> <command-data>`
- `read-attr` / `read-event` / `write-attr` / `subs-attr` / `subs-event` / `group-settings ...`

## Thermostat / Pump / 家电 cluster config（feature flag）

```cpp
// Header: data_model/legacy/esp_matter_cluster_impl.h
namespace cluster::thermostat {
typedef struct config {
    nullable<int16_t> local_temperature;
    uint8_t control_sequence_of_operation;     // 默认 4
    uint8_t system_mode;                        // 默认 1
    void *delegate;
    struct {
        feature::heating::config_t heating;
        feature::cooling::config_t cooling;
        feature::auto_mode::config_t auto_mode;
        feature::occupancy::config_t occupancy;
        feature::setback::config_t setback;
        feature::local_temperature_not_exposed::config_t local_temperature_not_exposed;
        feature::matter_schedule_configuration::config_t matter_schedule_configuration;
    } features;
    uint32_t feature_flags;
} config_t;
cluster_t *create(endpoint_t *endpoint, config_t *config, uint8_t flags);
// feature::heating::get_id() / feature::cooling::get_id() 返回 FeatureMap bit
}

namespace cluster::pump_configuration_and_control {
// config_t 构造: (max_pressure:int16, max_speed:uint16, max_flow:uint16)，nullable
// feature::constant_pressure::get_id()
}
```

## 家电 endpoint config（refrigerator / cabinet / room_ac / pump）

```cpp
// Header: data_model/legacy/esp_matter_endpoint_impl.h
namespace endpoint::refrigerator {
    typedef struct config { cluster::descriptor::config_t descriptor; } config_t;
}
namespace endpoint::temperature_controlled_cabinet {
    typedef struct config {
        cluster::descriptor::config_t descriptor;
        cluster::temperature_control::config_t temperature_control;
    } config_t;
}
namespace endpoint::room_air_conditioner {
    typedef struct config : app_base_config {
        cluster::on_off::config_t on_off;
        cluster::thermostat::config_t thermostat;
    } config_t;
}
namespace endpoint::pump {
    typedef struct config : app_base_config {
        cluster::on_off::config_t on_off;
        cluster::pump_configuration_and_control::config_t pump_configuration_and_control;
        explicit config(nullable<int16_t> max_pressure, nullable<uint16_t> max_speed,
                        nullable<uint16_t> max_flow);
    } config_t;
}
// 父子 endpoint 关联（refrigerator + cabinet 用）
// Header: data_model/esp_matter_data_model.h
esp_err_t set_parent_endpoint(endpoint_t *endpoint, endpoint_t *parent_endpoint);
```
> Thermostat 对 Heating/Cooling 是 O.a+ conformance —— 至少要设 `feature_flags = thermostat::feature::heating::get_id()` 或 cooling 之一。

## ICD（Intermittently Connected Device）Kconfig

ICD server 由 Kconfig 启用，无独立 API；关键选项（`examples/icd_app/sdkconfig.defaults*`）：
- `CONFIG_ENABLE_ICD_SERVER=y`
- `CONFIG_ICD_FAST_POLL_INTERVAL_MS`（SIT/LIT 均 500）
- `CONFIG_ICD_SLOW_POLL_INTERVAL_MS`（SIT 5000，LIT 20000）
- `CONFIG_ICD_IDLE_MODE_INTERVAL_SEC`（SIT 60，LIT 600）
- `CONFIG_ICD_ACTIVE_MODE_INTERVAL_MS`（1000）
- `CONFIG_ICD_ACTIVE_MODE_THRESHOLD_MS`（SIT 1000，LIT 5000）
- LIT：`CONFIG_ENABLE_ICD_LIT=y`、`CONFIG_ICD_CLIENTS_SUPPORTED_PER_FABRIC`、`CONFIG_ICD_MAX_NOTIFICATION_SUBSCRIBERS`
- 配套省电：`CONFIG_PM_ENABLE=y`、`CONFIG_FREERTOS_USE_TICKLESS_IDLE=y`、`CONFIG_IEEE802154_SLEEP_ENABLE=y`、`CONFIG_BT_LE_SLEEP_ENABLE=y`、`CONFIG_OPENTHREAD_MTD=y`
