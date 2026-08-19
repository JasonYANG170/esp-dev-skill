# Zigbee 网关与子设备动态映射

> **适用摘要**: 把一个 RainMaker 节点变成 **Zigbee 网关**，运行时把每个加入的 Zigbee 终端设备**动态映射**为一个 RainMaker 设备（数据驱动，非静态 `esp_rmaker_device_create`）。覆盖两种加设备方式（预共享 key `ZigBeeAlliance09` 与 install code JSON）、ESP32 + ESP32-H2 RCP 分体、`Add_zigbee_device` 参数、NVS 持久化与重启恢复。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-rainmaker/resources/`, source/examples in `repos/esp-rainmaker/`, and this recipe path `repos/esp-rainmaker/recipes/zigbee_gateway.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "RainMaker Zigbee 网关 / gateway"
- "Zigbee 子设备映射成 RainMaker 设备"
- "Add_zigbee_device 参数"
- "Zigbee install code 加密入网"
- "esp.device.zigbee_gateway"
- "运行时动态创建 RainMaker 设备"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | Wi-Fi 型 ESP Zigbee Gateway：ESP32 系列主控（运行 Zigbee 协调器栈）+ ESP32-H2（运行 OpenThread RCP，作 15.4 radio）。板子见 README |
| 主控固件 | `esp-rainmaker` 的 `examples/zigbee_gateway/`（依赖 `esp-zigbee-sdk`） |
| RCP 固件 | ESP-IDF `examples/openthread/ot_rcp`（menuconfig 勾 `OPENTHREAD_NCP_VENDOR_HOOK`），构建时打包进主控固件 |
| Kconfig | `CONFIG_ZB_ENABLED=y`、`CONFIG_ZB_ZCZR=y`（协调器/路由器角色）；可选 `CONFIG_ZIGBEE_INSTALLCODE_ENABLED` |
| 参考示例 | `examples/zigbee_gateway/` |

## 设备创建模型对比

| 维度 | 静态设备（custom_device / standard_devices recipe） | **本 recipe：网关动态映射** |
|---|---|---|
| 时机 | 编译期/启动期，固定数量 | **运行期**，子设备加入时按需创建 |
| 触发 | `app_main` 里直接 `device_create` | Zigbee `DEVICE_ANNCE` → `match_desc` → `simple_desc` → 按 `device_id` 分派 |
| 设备类型 | 业务固定类型 | 按子设备 HA device id 映射（如 On/Off Light → `lightbulb`，IAS Zone → contact-sensor） |
| 上报 | 增删后须 `esp_rmaker_report_node_details()` | 同左；每次加完设备都调，云端才刷新设备列表 |
| 持久化 | 通常不需 | 子设备 short_addr/endpoint/type 存 NVS，重启后 `resume_device` 重建 |

## 支持的子设备映射（示例当前覆盖）

| Zigbee HA device id | 映射为 RainMaker 设备 | 创建方式 |
|---|---|---|
| `ESP_ZB_HA_ON_OFF_LIGHT_DEVICE_ID` (0x0100) | `esp_rmaker_lightbulb_device_create`（标准灯） | helper |
| `ESP_ZB_HA_IAS_ZONE_ID` (0x0402，门窗把手等) | `esp_rmaker_device_create(..., "esp.device.contact-sensor", ...)` + 自定义 `Door Status` 参数 | 手动 |

> 不在表里的 device id 会被 `simple_desc_cb` 判为 `Unsupported device type!` 并丢弃。新增类型需要在 `simple_desc_cb` 的 `switch` 里加分派。

## 分步说明

### 1. sdkconfig.defaults（Zigbee 协调器 + 安全）

来自 `examples/zigbee_gateway/sdkconfig.defaults`：

```text
# --- Zboss（Zigbee 协调器/路由器）---
CONFIG_ZB_ENABLED=y
CONFIG_ZB_ZCZR=y

# --- IEEE802154 ---
CONFIG_IEEE802154_RECEIVE_DONE_HANDLER=y

# --- mbedTLS（Zigbee install code / EC-JPAKE / DTLS）---
CONFIG_MBEDTLS_CMAC_C=y
CONFIG_MBEDTLS_SSL_PROTO_DTLS=y
CONFIG_MBEDTLS_KEY_EXCHANGE_ECJPAKE=y
CONFIG_MBEDTLS_ECJPAKE_C=y

# --- Secure Local Control ---
CONFIG_ESP_RMAKER_LOCAL_CTRL_ENABLE=y
CONFIG_ESP_RMAKER_LOCAL_CTRL_SECURITY_1=y

# --- 任务栈（Zigbee 栈较大）---
CONFIG_ESP_RMAKER_WORK_QUEUE_TASK_STACK=8192
CONFIG_FREERTOS_TIMER_TASK_STACK_DEPTH=6240
CONFIG_ESP_SYSTEM_EVENT_TASK_STACK_SIZE=4096

# --- 滚回（配合 OTA）---
CONFIG_BOOTLOADER_APP_ROLLBACK_ENABLE=y
```

加设备方式二选一（menuconfig → ESP Rainmaker Zigbee Gateway Example）：

```text
# 方式 A：预共享 key "ZigBeeAlliance09"（默认，CONFIG_ZIGBEE_INSTALLCODE_ENABLED 不勾）
# 方式 B：install code（勾选）
CONFIG_ZIGBEE_INSTALLCODE_ENABLED=y
```

### 2. 构建与烧录（RCP 先行）

```bash
# 1) 构建 RCP（esp32h2），勾 OPENTHREAD_NCP_VENDOR_HOOK
cd $IDF_PATH/examples/openthread/ot_rcp
idf.py set-target esp32h2 build

# 2) 构建网关主控（esp32s3）
cd <esp-rainmaker>/examples/zigbee_gateway
idf.py set-target esp32s3 build
# M5Stack CoreS3 + Module Gateway H2：
# idf.py -D SDKCONFIG_DEFAULTS="sdkconfig.defaults;sdkconfig.defaults.m5stack" set-target esp32s3 build
idf.py -p <PORT> flash monitor
```

### 3. 网关设备与 `Add_zigbee_device` 参数

网关本身挂一个 `esp.device.zigbee_gateway` 设备，带一个 `Add_zigbee_device` 参数，类型/UI 随加设备方式变化（见 `esp_rmaker_zigbee_gateway_device_create`）：

```c
#include <esp_rmaker_standard_types.h>   /* ESP_RMAKER_DEVICE_ZIGBEE_GATEWAY, ESP_RMAKER_PARAM_ADD_ZIGBEE_DEVICE, ESP_RMAKER_UI_* */
#include <esp_rmaker_standard_params.h>  /* ESP_RMAKER_DEF_ADD_ZIGBEE_DEVICE ("Add_zigbee_device") */
#include <esp_rmaker_core.h>

esp_rmaker_device_t *esp_rmaker_zigbee_gateway_device_create(const char *dev_name, void *priv_data)
{
    esp_rmaker_device_t *device = esp_rmaker_device_create(dev_name, ESP_RMAKER_DEVICE_ZIGBEE_GATEWAY, priv_data);
    esp_rmaker_device_add_param(device, esp_rmaker_name_param_create(ESP_RMAKER_DEF_NAME_PARAM, dev_name));

#if CONFIG_ZIGBEE_INSTALLCODE_ENABLED
    /* install code 模式：字符串参数 + 二维码扫描 UI */
    esp_rmaker_param_t *p = esp_rmaker_param_create(
        ESP_RMAKER_DEF_ADD_ZIGBEE_DEVICE,
        ESP_RMAKER_PARAM_ADD_ZIGBEE_DEVICE,
        esp_rmaker_str(""), PROP_FLAG_READ | PROP_FLAG_WRITE);
    esp_rmaker_param_add_ui_type(p, ESP_RMAKER_UI_QR_SCAN);
#else
    /* 预共享 key 模式：布尔参数 + 开关 UI（App 点开关打开 180s 入网窗口） */
    esp_rmaker_param_t *p = esp_rmaker_param_create(
        ESP_RMAKER_DEF_ADD_ZIGBEE_DEVICE,
        ESP_RMAKER_PARAM_ADD_ZIGBEE_DEVICE,
        esp_rmaker_bool(false), PROP_FLAG_READ | PROP_FLAG_WRITE);
    esp_rmaker_param_add_ui_type(p, ESP_RMAKER_UI_TOGGLE);
#endif
    esp_rmaker_device_add_param(device, p);
    return device;
}
```

标准类型宏（取自 `esp_rmaker_standard_types.h`）：
- `ESP_RMAKER_DEVICE_ZIGBEE_GATEWAY` = `"esp.device.zigbee_gateway"`
- `ESP_RMAKER_PARAM_ADD_ZIGBEE_DEVICE` = `"esp.param.add_zigbee_device"`
- `ESP_RMAKER_DEF_ADD_ZIGBEE_DEVICE` = `"Add_zigbee_device"`（参数**名**，来自 `esp_rmaker_standard_params.h`）
- `ESP_RMAKER_UI_QR_SCAN` = `"esp.ui.qr-scan"`、`ESP_RMAKER_UI_TOGGLE` = `"esp.ui.toggle"`

### 4. `Add_zigbee_device` 写回调（云触发入网）

```c
static esp_err_t write_cb(const esp_rmaker_device_t *device, const esp_rmaker_param_t *param,
                          const esp_rmaker_param_val_t val, void *priv_data, esp_rmaker_write_ctx_t *ctx)
{
    if (strcmp(esp_rmaker_param_get_name(param), ESP_RMAKER_DEF_ADD_ZIGBEE_DEVICE) == 0) {
#if CONFIG_ZIGBEE_INSTALLCODE_ENABLED
        /* val.val.s 是 JSON：{"install_code":"...","MAC_address":"..."} */
        esp_zigbee_ic_mac_address_t ic_mac;
        if (esp_zb_prase_ic_obj(val.val.s, &ic_mac) == ESP_OK) {
            esp_gateway_control_permit_join(ESP_ZIGBEE_GATWAY_OPEN_NETWORK_DEFAULT_TIME); /* 180s */
            esp_gateway_control_secur_ic_add(&ic_mac, ESP_ZB_IC_TYPE_128);
        }
#else
        /* 预共享 key：true 开放入网窗口，false 关闭 */
        if (val.val.b) esp_app_enable_zigbee_add_device();   /* 内部 permit_join(180) + 定时关闭 */
        else          esp_app_disable_zigbee_add_device();   /* permit_join(0) */
#endif
    }
    esp_rmaker_param_update_and_report(param, val);   /* 回写，云端状态一致 */
    return ESP_OK;
}
```

- `ESP_ZIGBEE_GATWAY_OPEN_NETWORK_DEFAULT_TIME` = `180`（秒，`esp_gateway_control.h`）
- install code 的 JSON 可由 App 扫二维码传入；二维码生成见 README 示例链接。

### 5. 子设备加入 → 动态创建 RainMaker 设备

Zigbee 协议栈回调链（见 `esp_gateway_main.c`）：设备入网 → `ESP_ZB_ZDO_SIGNAL_DEVICE_ANNCE` → `match_desc` → `simple_desc_cb` 按 `app_device_id` 分派：

```c
static void simple_desc_cb(esp_zb_zdp_status_t status, esp_zb_af_simple_desc_1_1_t *desc, void *ctx)
{
    if (status != ESP_ZB_ZDP_STATUS_SUCCESS) return;
    switch (desc->app_device_id) {
    case ESP_ZB_HA_ON_OFF_LIGHT_DEVICE_ID:
        esp_app_rainmaker_add_joining_device(ESP_ZB_HA_ON_OFF_LIGHT_DEVICE_ID, short_addr, endpoint);
        return;
    case ESP_ZB_HA_IAS_ZONE_ID:
        esp_zigbee_write_ias_cie_address(short_addr, endpoint);   /* IAS 需先写 CIE 地址 */
        esp_app_rainmaker_add_joining_device(ESP_ZB_HA_IAS_ZONE_ID, short_addr, endpoint);
        return;
    default:
        ESP_LOGW(TAG, "Unsupported device type!");
    }
}
```

`esp_app_rainmaker_add_joining_device` 内部按 device id 调对应 helper（灯用 `esp_rmaker_lightbulb_device_create`，IAS 用手动 `device_create` + `Door Status` 参数），并给每个子设备加一个 `Delete device` 触发参数（`ESP_RMAKER_UI_TRIGGER`），然后：

```c
esp_rmaker_device_add_cb(new->device, zigbee_device_write_cb, NULL);
esp_rmaker_node_add_device(esp_rmaker_get_node(), new->device);   /* 挂到网关节点 */
/* ... 存 NVS ... */
esp_rmaker_report_node_details();   /* 关键：通知云端刷新设备列表 */
```

> **必须**调 `esp_rmaker_report_node_details()`，否则 App 看不到新设备（设备是运行时新增的，启动期 node config 不含它）。

### 6. 持久化与重启恢复

short_addr/endpoint/type 存 NVS（分区 `nvs`、命名空间 `zggw`），重启后在 `ESP_ZB_ZDO_SIGNAL_SKIP_STARTUP` 里调 `resume_device()` 重建链表并 `report_node_details`：

```c
case ESP_ZB_ZDO_SIGNAL_SKIP_STARTUP:
    esp_app_rainmaker_main();                              /* RainMaker 任务 */
    esp_zb_bdb_start_top_level_commissioning(ESP_ZB_BDB_MODE_INITIALIZATION);
    break;
```

### 7. 端到端加设备（云侧）

**方式 A — 预共享 key**：App 点 `Add_zigbee_device` 开关 → 设备 `permit_join(180)` → 子设备用 `ZigBeeAlliance09` 入网 → 自动映射成 RainMaker 设备 → App 刷新可见。

**方式 B — install code**：App 扫包含 `install_code` + `MAC_address` 的二维码（JSON） → 云下发 JSON 到 `Add_zigbee_device` 参数 → 设备解析后 `permit_join` + `secur_ic_add` → 子设备用 install code 入网。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 子设备入网但 App 看不到 | 加完设备没 `report_node_details` | 每次动态 `add_device` 后立即调 `esp_rmaker_report_node_details()` |
| 子设备类型不被支持 | `simple_desc_cb` 的 switch 没覆盖该 device id | 在 switch 里加 case，调对应 helper 或手动建设备 |
| install code 入网失败 | JSON 解析失败 / install code 或 MAC 错 | 校验 `esp_zb_prase_ic_obj` 返回值；二维码 JSON 字段须为 `install_code`、`MAC_address` |
| 预共享 key 入网失败 | `permit_join` 窗口已关（180s 到） | 重新点开关；或调大 `ESP_ZIGBEE_GATWAY_OPEN_NETWORK_DEFAULT_TIME` |
| `ESP_RMAKER_UI_QR_SCAN` / `UI_TOGGLE` 选错 | install code 与预共享 key 模式混淆 | 由 `CONFIG_ZIGBEE_INSTALLCODE_ENABLED` 决定，二者参数类型/UI 不同 |
| 重启后子设备消失 | NVS 未存或未 `resume_device` | 存 short_addr/endpoint/type；`SKIP_STARTUP` 信号里调 `resume_device()` + `report_node_details` |
| 编译报 `Only Zigbee gateway host device should be defined` | 宏配置错（`ZB_MACSPLIT_DEVICE` 与 host 冲突） | 按示例配置，只定义 host（`ZB_MACSPLIT_HOST`） |
| 删除子设备无反应 | 没处理 `Delete device` 触发参数的 write | 在子设备 write_cb 里识别 `ESP_RMAKER_UI_TRIGGER` 参数并 `remove_device` + 清 NVS + `report_node_details` |
| RCP 无响应 | RCP 未烧 / UART 接反 | 先 build `ot_rcp`（esp32h2，勾 `OPENTHREAD_NCP_VENDOR_HOOK`）；检查 PIN_TO_RCP_TX/RX/RESET/BOOT Kconfig |

## 参考项目

- `examples/zigbee_gateway/` — Zigbee 网关完整示例
  - `examples/zigbee_gateway/main/esp_gateway_main.c` — Zigbee 协议栈信号处理、`simple_desc_cb` 分派、`bdb` commissioning
  - `examples/zigbee_gateway/main/esp_app_rainmaker.c` — 网关设备创建、`Add_zigbee_device` write_cb、动态加设备、NVS 持久化、`resume_device`
  - `examples/zigbee_gateway/main/esp_gateway_control.h` — `ESP_ZIGBEE_GATWAY_OPEN_NETWORK_DEFAULT_TIME`、`permit_join`、`secur_ic_add`
  - `examples/zigbee_gateway/main/Kconfig.projbuild` — `CONFIG_ZIGBEE_INSTALLCODE_ENABLED`、板级 RCP 引脚
  - `examples/zigbee_gateway/sdkconfig.defaults` — `ZB_ENABLED`/`ZB_ZCZR`/mbedTLS/栈尺寸
- `components/esp_rainmaker/include/esp_rmaker_standard_types.h` — `ESP_RMAKER_DEVICE_ZIGBEE_GATEWAY`、`ESP_RMAKER_PARAM_ADD_ZIGBEE_DEVICE`、`ESP_RMAKER_UI_QR_SCAN`/`UI_TOGGLE`/`UI_TRIGGER`
- `components/esp_rainmaker/include/esp_rmaker_standard_params.h` — `ESP_RMAKER_DEF_ADD_ZIGBEE_DEVICE`
- `examples/zigbee_gateway/README.md` — 两种加设备方式的日志样例、install code 二维码生成链接
- ESP-IDF `examples/openthread/ot_rcp` — RCP 固件来源
- `recipes/custom_device.md` / `recipes/standard_devices.md` — 静态设备创建模型（对比参照）
