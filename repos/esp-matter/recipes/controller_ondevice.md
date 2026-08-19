# 设备端 Matter Controller / Commissioner（ESP32 做控制器）

> **适用摘要**: 把 ESP32-S3 本身做成 Matter 控制器 / commissioner，在固件里发起 pairing、`invoke-cmd`、`read-attr` / `read-event`、`write-attr`、`subs-attr` / `subs-event` 与 group-settings 操作。与 `commissioning_chiptool.md`（host 端 chip-tool 做 commissioner）互补：本 recipe 适用于"没有外部 chip-tool、由 ESP32 直接入网并控制其它 Matter 设备"的场景。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-matter/resources/`, source/examples in `repos/esp-matter/`, and this recipe path `repos/esp-matter/recipes/controller_ondevice.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "用 ESP32 commission 别的 Matter 设备"
- "做一个 ESP32 控制器 / commissioner"
- "设备端 invoke-cmd / read-attr / write-attr / subs-attr"
- "ESP32 pairing onnetwork / ble-wifi / ble-thread"
- "matter esp controller" 命令行
- "Attestation Trust Store 怎么选 / PAA 证书放哪"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/controller/`（`main/app_main.cpp`、`main/matter_project_config.h`、`sdkconfig.defaults*`） |
| 组件 | `components/esp_matter_controller/`（client / commands / core） |
| 目标芯片 | esp32s3（ble-wifi / ble-thread commissioner）、esp32c3 / esp32c6 / esp32h2 等（onnetwork / IP-only） |
| PAA 证书 | Spiffs / DCL / Custom 三种 trust store 之一（见步骤 5） |
| 关键 Kconfig | `CONFIG_ENABLE_CHIP_CONTROLLER_BUILD=y`、`CONFIG_ESP_MATTER_CONTROLLER_ENABLE=y`、`CONFIG_ESP_MATTER_COMMISSIONER_ENABLE=y`、`CONFIG_ESP_MATTER_ENABLE_MATTER_SERVER=n` |

## 分步说明

### 1. 关键 Kconfig（client-only，不带 Matter server）

控制器固件默认是 **client-only**（无 server 数据模型），来自 `examples/controller/sdkconfig.defaults`：

```text
# Enable Controller and commissioner
CONFIG_ENABLE_CHIP_CONTROLLER_BUILD=y
CONFIG_ESP_MATTER_CONTROLLER_ENABLE=y
CONFIG_ESP_MATTER_ENABLE_MATTER_SERVER=n
CONFIG_ESP_MATTER_COMMISSIONER_ENABLE=y

# Enable HKDF in mbedtls
CONFIG_MBEDTLS_HKDF_C=y

# Increase udp endpoints num for commissioner
CONFIG_NUM_UDP_ENDPOINTS=16
# 一个 fabric 一个本地 IPv6 地址 + 一个 link-local
CONFIG_LWIP_IPV6_NUM_ADDRESSES=6

# Commissioner 需要更大的 stack
CONFIG_CHIP_TASK_STACK_SIZE=15360
CONFIG_ESP_MAIN_TASK_STACK_SIZE=10240
```

esp32s3 上要启用 BLE 才能做 ble-wifi / ble-thread pairing（`sdkconfig.defaults.esp32s3`）：

```text
CONFIG_IDF_TARGET="esp32s3"
# commissioner 自己用 BLE 去连 end-device
CONFIG_ENABLE_ESP32_BLE_CONTROLLER=y
CONFIG_USE_BLE_ONLY_FOR_COMMISSIONING=n
```

### 2. 初始化 controller 与 commissioner（持锁）

控制器是单例，来自 `examples/controller/main/app_main.cpp`。必须在 `esp_matter::start()` 之后，并在 **Matter 锁内** 调用 `init()` + `setup_commissioner()`：

```cpp
#include <esp_matter_controller_client.h>
#include <esp_matter_controller_console.h>

#if CONFIG_ESP_MATTER_COMMISSIONER_ENABLE
    esp_matter::lock::ScopedChipStackLock lock(portMAX_DELAY);
    // init(node_id, fabric_id, listen_port)
    esp_matter::controller::matter_controller_client::get_instance().init(112233, 1, 5580);
    esp_matter::controller::matter_controller_client::get_instance().setup_commissioner();
#endif
```

`init()` 三个参数：controller 自己的 NodeId、FabricId、监听端口（示例用 `112233 / 1 / 5580`）。

### 3. 注册命令行（`matter esp controller ...`）

控制器的所有交互命令通过 `matter` shell 暴露（`CONFIG_ENABLE_CHIP_SHELL=y`）：

```cpp
esp_matter::console::diagnostics_register_commands();
esp_matter::console::wifi_register_commands();
esp_matter::console::factoryreset_register_commands();
#if CONFIG_ESP_MATTER_CONTROLLER_ENABLE
    esp_matter::console::controller_register_commands();
#endif
#ifdef CONFIG_OPENTHREAD_BORDER_ROUTER
    esp_matter::console::otcli_register_commands();
#endif
esp_matter::console::init();
```

### 4. 把 controller 接入网络再 commission end-device

controller 必须先与 end-device 在同一 IP 网络（onnetwork）或用 BLE 直接 pairing（ble-wifi / ble-thread）。先连 Wi-Fi：

```text
matter esp wifi connect <ssid> <password>
```

再 pairing（来自 `docs/en/controller.rst`）：

```text
# onnetwork：controller 与 end-device 已在同一 IP 网络
matter esp controller pairing onnetwork <node_id> <setup_passcode>

# ble-wifi（esp32s3）：controller 通过 BLE 把 end-device 接入 controller 所在 Wi-Fi
matter esp controller pairing ble-wifi <node_id> <ssid> <password> <pincode> <discriminator>

# ble-thread（esp32s3）：需先有 Thread Border Router
matter esp controller pairing ble-thread <node_id> <dataset_tlvs> <pincode> <discriminator>

# code 系列：用 Matter setup payload（QR / manual code）
matter esp controller pairing code <node_id> <setup_payload>
matter esp controller pairing code-wifi <node_id> <ssid> <passphrase> <setup_payload>
matter esp controller pairing code-thread <node_id> <operationalDataset> <setup_payload>
matter esp controller pairing code-wifi-thread <node_id> <ssid> <passphrase> <operationalDataset> <setup_payload>
```

`code-wifi` / `code-thread` / `code-wifi-thread` 在 `CONFIG_ENABLE_ESP32_BLE_CONTROLLER=y` 时会同时在 IP 和 BLE 上发现 end-device。

### 5. cluster 命令（invoke / read / write / subscribe）

入网成功后即可发命令。所有命令的 node-id 既可以是单播 `node-id` 也可以是组播 `group-id`。

**invoke-cmd**（`command-data` 是 JSON，键名为 `"<TagNumber>:<DataType>"`，DataType 定义在 `components/esp_matter/utils/json_to_tlv/`）：

```text
# OnOff Toggle（cluster 6, command 2），无参数
matter esp controller invoke-cmd <node-id> 1 6 2

# LevelControl MoveToLevel（cluster 8, command 0）：level=10, transitionTime=0
matter esp controller invoke-cmd <node-id> <endpoint> 8 0 "{\"0:U8\": 10, \"1:U16\": 0, \"2:U8\": 0, \"3:U8\": 0}"

# Groups AddGroup（cluster 0x4, command 0）
matter esp controller invoke-cmd <node-id> <endpoint> 0x4 0 "{\"0:U16\": 1, \"1:STR\": \"grp1\"}"
```

DataType 中 `bytes` 用 Base64 字符串（如 epoch key）。

**read-attr / read-event**（`endpoint-ids` / `cluster-ids` / `attribute-ids` 都支持逗号分隔多个）：

```text
matter esp controller read-attr <node-id> <endpoint-ids> <cluster-ids> <attribute-ids>
matter esp controller read-event <node-id> <endpoint-ids> <cluster-ids> <event-ids>
```

**write-attr**（`attribute-value` 是单元素 JSON 对象；多属性写用 JSON 数组，对应多个 endpoint/cluster/attribute）：

```text
# StartUpOnOff = 2（OnOff cluster=6, attr=0x4003）
matter esp controller write-attr <node_id> <endpoint_id> 6 0x4003 "{\"0:U8\": 2}"
# nullable
matter esp controller write-attr <node_id> <endpoint_id> 6 0x4003 "{\"0:NULL\": null}"

# Binding list（cluster=30, attr=0）
matter esp controller write-attr <node_id> <endpoint_id> 30 0 "{\"0:ARR-OBJ\":[{\"1:U64\":1, \"3:U16\":1, \"4:U32\": 6}]}"

# 同时写多个属性（endpoint/cluster/attribute 逗号分隔，value 是 JSON 数组）
matter esp controller write-attr <node_id> <ep1>,<ep2> 31,30 0,0 "[{...},{...}]"
```

> uint64 / int64 绝对值大于 2^53 时用字符串表示以保精度：`"{\"1:U64\": \"9007199254740993\"}"`。

**subs-attr / subs-event**（min/max interval 单位秒）：

```text
matter esp controller subs-attr <node-id> <endpoint-ids> <cluster-ids> <attribute-ids> <min-interval> <max-interval>
matter esp controller subs-event <node-id> <endpoint-ids> <cluster-ids> <event-ids> <min-interval> <max-interval>
```

### 6. Group settings（组播控制）

要让 controller 发组播命令，必须先把自己加入对应 group 并绑定 keyset：

```text
matter esp controller group-settings show-groups
matter esp controller group-settings add-group <group-id> <group-name>
matter esp controller group-settings remove-group <group-id>
matter esp controller group-settings show-keysets
matter esp controller group-settings add-keyset <keyset-id> <policy> <validity-time> <epoch-key-oct-str>
matter esp controller group-settings remove-keyset <keyset-id>
matter esp controller group-settings bind-keyset <group-id> <keyset-id>
matter esp controller group-settings unbind-keyset <group-id> <keyset-id>
```

之后 `invoke-cmd <group-id> ...` 即组播。

### 7. Attestation Trust Store（PAA 来源）

在 menuconfig：`Components → ESP Matter Controller → Attestation Trust Store`。四个选项（`components/esp_matter_controller/Kconfig` 的 `ESP_MATTER_COMMISSIONER_ATTESTATION_TRUST_STORE` choice）：

| 选项 | Kconfig | PAA 来源 |
|---|---|---|
| Test | `TEST_ATTESTATION_TRUST_STORE` | 固化的两份测试 PAA（`Chip-Test-PAA-FFF1-Cert` / `Chip-Test-PAA-NoVID-Cert`） |
| Spiffs | `SPIFFS_ATTESTATION_TRUST_STORE` | `paa_cert/` 目录下的 DER 文件，烧到 spiffs 分区 |
| DCL | `DCL_ATTESTATION_TRUST_STORE` | commissioning 时从 DCL MainNet/TestNet 拉取 |
| Custom | `CUSTOM_ATTESTATION_TRUST_STORE` | 调 `set_custom_attestation_trust_store()`（须在 setup_commissioner 之前） |

`examples/controller/paa_cert/` + `partitions.csv` 的 `paa_cert data spiffs 0x20000` 给出了 Spiffs 模式的分区布局。

### 8. NOC Issuer（签发证书）

`ESP_MATTER_COMMISSIONER_OPERATIONAL_CREDS_ISSUER` choice（`Kconfig`）：

| 选项 | Kconfig | 行为 |
|---|---|---|
| Test | `TEST_OPERATIONAL_CREDS_ISSUER` | 用 `ExampleOperationalCredentialsIssuer` 本地生成 RCAC/Key，自签 commissioner NOC，再为被 commission 的设备签 NOC。**仅供测试** |
| Custom | `CUSTOM_OPERATIONAL_CREDS_ISSUER` | 自定义 issuer 类（典型：把 CSR 转发到云端签名） |

> 生产环境 RCAC 不能本地生成，必须用 Custom issuer 从受信服务签发。

### 9. OTA Provider / Border Router 组合

`sdkconfig.defaults.otbr` 让 controller 同时做 Thread Border Router（见 `recipes/thread_border_router.md`）。`sdkconfig.defaults.ram_optimization` 用 SPIRAM 放 BSS（见 `recipes/optimizations.md`）。多 SDKCONFIG_DEFAULTS 叠加：

```bash
idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.ram_optimization" set-target esp32s3 build
idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults.otbr" set-target esp32s3 build
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| pairing onnetwork 无响应 | controller 与 end-device 不在同一 IP 网络 | 先 `matter esp wifi connect` 确保两端都能互相 ping |
| 入网时 DA 校验失败 | 用了 Test trust store，但 end-device 的 PAI/DAC 不是 Chip-Test 签的 | 换 Spiffs/DCL trust store，或让 end-device 用测试凭据 |
| `invoke-cmd` JSON 解析失败 | 键名格式不是 `"<TagNumber>:<DataType>"` | 严格用 `{"0:U8": 10}` 形式；DataType 见 `utils/json_to_tlv/` |
| end-device 重启后第一条命令超时 | CASE 会话失效，controller 仍用旧会话，等超时后才重建 | 第二条命令会自动重建会话；或主动重连 |
| commissioner 启动失败 | `init()` / `setup_commissioner()` 在锁外调用 | 包在 `ScopedChipStackLock` 内 |
| `CONFIG_ESP_MATTER_ENABLE_MATTER_SERVER=y` 与 controller 共存 | controller 是 client-only | 设 `=n`（Kconfig 中 controller/commissioner 都 `depends on !ESP_MATTER_ENABLE_MATTER_SERVER`） |
| ble-wifi pairing 不支持 | 目标不是 esp32s3 | ble-wifi / ble-thread commissioner 仅 esp32s3；其它芯片用 onnetwork/code |
| uint64 精度丢失 | 直接写大数字 JSON | 绝对值 > 2^53 用字符串 |

## 参考

- `D:/esp-skill/espressif-repos/esp-matter/docs/en/controller.rst` — 完整 controller/commissioner 文档（pairing / invoke / read / write / subscribe / group-settings / attestation / NOC issuer）
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/main/app_main.cpp` — `matter_controller_client::init(112233, 1, 5580)` + `setup_commissioner()`
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/sdkconfig.defaults` / `sdkconfig.defaults.esp32s3` / `sdkconfig.defaults.otbr` / `sdkconfig.defaults.ram_optimization`
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/partitions.csv`（`paa_cert data spiffs 0x20000`）
- `D:/esp-skill/espressif-repos/esp-matter/examples/controller/README.md`（OTBR dataset init / ifconfig up / thread start / pairing ble-thread / invoke-cmd）
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter_controller/Kconfig` — `ESP_MATTER_CONTROLLER_ENABLE` / `ESP_MATTER_COMMISSIONER_ENABLE` / trust store choice / creds issuer choice
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter_controller/core/esp_matter_controller_client.h` — `matter_controller_client::get_instance().init(...)` / `setup_commissioner()` / `unpair()`
- `D:/esp-skill/espressif-repos/esp-matter/components/esp_matter/utils/json_to_tlv/` — invoke-cmd / write-attr 的 DataType 枚举
