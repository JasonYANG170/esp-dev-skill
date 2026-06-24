# WASM Native RainMaker 与 Wi-Fi 配网

> **适用摘要**: 启用 RainMaker 与 Wi-Fi provisioning native API（基于 ESP RainMaker 与 `wifi_provisioning`），供 WASM 应用集成云与 SoftAP/BLE 配网。

## 触发意图

- "WASM 应用接 RainMaker"
- "WASM 里 Wi-Fi 配网"
- "wasm_rmaker_run"
- "wasm_wifi_prov_mgr_*"

## 前置条件

| 条件 | 要求 |
|---|---|
| RainMaker | `CONFIG_WASMACHINE_WASM_EXT_NATIVE_RMAKER=y`（默认 n）；**ESP32-P4 不支持**（`idf_component.yml` 已排除） |
| Wi-Fi 配网 | `CONFIG_WASMACHINE_WASM_EXT_NATIVE_WIFI_PROVISIONING=y`（默认 n） |
| 主开关 | `CONFIG_WASMACHINE_WASM_EXT_NATIVE=y` |
| RainMaker 工厂分区 | `CONFIG_ESP_RMAKER_FACTORY_PARTITION_NAME="nvs"`（见 `sdkconfig.defaults.esp32s3`） |

## 分步说明

### 1. 启用（sdkconfig.defaults）

```ini
CONFIG_WASMACHINE_WASM_EXT_NATIVE=y
CONFIG_WASMACHINE_WASM_EXT_NATIVE_RMAKER=y                # 非 P4
CONFIG_WASMACHINE_WASM_EXT_NATIVE_WIFI_PROVISIONING=y
```

参考工程 `examples/wasmachine/sdkconfig.defaults` 默认就开了这两项（含 RainMaker）。

### 2. RainMaker native API

来源 `components/wasmachine_ext_wasm_native_rainmaker/src/wm_ext_wasm_native_rmaker.c`，注册到 `"env"`：

| import | 签名 |
|---|---|
| `wasm_rmaker_run` | `(i)i` |
| `wasm_rmaker_param_add_valid_str_list` | `(i$i)i` |
| `wasm_rmaker_call_native_func` | `(ii*)i` |

`wasm_rmaker_call_native_func` 是通用派发入口（按 func_id 调用对应 RainMaker 接口）。底层是 ESP RainMaker SDK；固件需要 NVS 工厂分区存放证书（`CONFIG_ESP_RMAKER_FACTORY_PARTITION_NAME`）。

### 3. Wi-Fi provisioning native API

来源 `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_wifi_provisioning.c`，注册到 `"env"`：

| import | 签名 |
|---|---|
| `wasm_wifi_prov_mgr_init` | `(ii*)i` |
| `wasm_wifi_prov_mgr_deinit` | `()i` |
| `wasm_wifi_prov_mgr_is_provisioned` | `(*)i` |
| `wasm_wifi_prov_mgr_start_provisioning` | `(ii*i)i` |
| `wasm_wifi_prov_mgr_stop_provisioning` | `()i` |
| `wasm_wifi_prov_mgr_wait` | `()i` |
| `wasm_wifi_prov_mgr_disable_auto_stop` | `(i)i` |
| `wasm_wifi_prov_mgr_set_app_info` | `(**ii)i` |
| `wasm_wifi_prov_mgr_endpoint_create` | `(*)i` |
| `wasm_wifi_prov_mgr_endpoint_register` | `(*ii)i` |
| `wasm_wifi_prov_mgr_endpoint_unregister` | `(*)i` |
| `wasm_wifi_prov_mgr_get_wifi_state` | `($)i` |
| `wasm_wifi_prov_mgr_get_wifi_disconnect_reason` | `($)i` |
| `wasm_wifi_prov_mgr_configure_sta` | `($)i` |
| `wasm_wifi_prov_mgr_reset_provisioning` | `()i` |
| `wasm_wifi_prov_mgr_reset_sm_state_on_failure` | `()i` |
| `wasm_wifi_prov_scheme_ble_set_service_uuid` | `($)i` |
| `wasm_wifi_prov_scheme_ble_set_mfg_data` | `($i)i` |

底层是 ESP-IDF `wifi_provisioning` 组件；WASM 应用经这些 import 完成 SoftAP/BLE 配网并写入 NVS。

### 4. 典型流程（伪代码）

```c
/* Wi-Fi 配网 */
if (!wasm_wifi_prov_mgr_is_provisioned(&provisioned) || !provisioned) {
    wasm_wifi_prov_mgr_init(service_id_sec, ...);
    wasm_wifi_prov_mgr_start_provisioning(security, pop, service_name, service_key);
    wasm_wifi_prov_mgr_wait();   /* 阻塞直到配网完成（或 auto-stop） */
}
/* 之后 WASM 应用即可用 HTTP/MQTT 等联网 native */
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| P4 上找不到 RainMaker | P4 不支持 | 删掉 `CONFIG_WASMACHINE_WASM_EXT_NATIVE_RMAKER=y` |
| RainMaker 启动失败 | 工厂分区/证书缺失 | 确认 `CONFIG_ESP_RMAKER_FACTORY_PARTITION_NAME="nvs"`，并烧录证书到 NVS |
| 配网超时 | security/pop/服务名错 | 检查 `start_provisioning` 参数与手机端一致 |
| `wasm_rmaker_*` import 未定义 | 未开 RMAKER | 确认 `CONFIG_WASMACHINE_WASM_EXT_NATIVE_RMAKER=y` 且 `CONFIG_WASMACHINE_WASM_EXT_NATIVE=y` |

## 参考

- `components/wasmachine_ext_wasm_native_rainmaker/src/wm_ext_wasm_native_rmaker.c`
- `components/wasmachine_ext_wasm_native/src/wm_ext_wasm_native_wifi_provisioning.c`
- `components/wasmachine_ext_wasm_native_rainmaker/Kconfig.wasmachine`
- `examples/wasmachine/sdkconfig.defaults.esp32s3` — `CONFIG_ESP_RMAKER_FACTORY_PARTITION_NAME`
- `resources/api_reference.md` §6、§7 — import 表
