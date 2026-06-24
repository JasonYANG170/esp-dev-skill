# ESP RainMaker 常见陷阱合集

> 来自头文件注释、Kconfig 说明与官方示例的实战陷阱。每条均给出错误写法与正确写法。

## 1. 初始化顺序（最高频）

**顺序**：`nvs_flash_init()` → `app_network_init()` → `esp_rmaker_node_init()` → 添加设备/服务 → 所有 `*_enable()` → `esp_rmaker_start()` → `app_network_start()`。

```c
// ❌ WRONG
esp_rmaker_start();           // 先启动 Agent
esp_rmaker_ota_enable_default(); // 再 enable，不生效

// ✅ CORRECT
esp_rmaker_ota_enable_default();
esp_rmaker_schedule_enable();
esp_rmaker_start();
```

## 2. write 回调不回写参数

```c
// ❌ WRONG — 云端状态不同步
if (strcmp(name, ESP_RMAKER_DEF_POWER_NAME) == 0) app_driver_set_state(val.val.b);

// ✅ CORRECT
if (strcmp(name, ESP_RMAKER_DEF_POWER_NAME) == 0) {
    app_driver_set_state(val.val.b);
    esp_rmaker_param_update(param, val);
}
```

## 3. 参数值类型未用 helper

```c
// ❌ WRONG — type 字段未初始化
esp_rmaker_param_val_t v; v.val.b = true;

// ✅ CORRECT
esp_rmaker_param_update_and_report(p, esp_rmaker_bool(true));
```

## 4. 设备未挂载到节点

`esp_rmaker_device_create()` 后必须 `esp_rmaker_node_add_device(node, dev)`，否则云端看不到。

## 5. Claiming 与芯片不匹配

| 芯片 | Self Claim | Assisted Claim |
|---|:---:|:---:|
| ESP32 | ❌ | ✅ |
| ESP32-C2 | ❌ | ✅ |
| ESP32-S2 | ❌ | ❌（无 BT） |
| S3/C3/C5/C6/H2 | ✅ | ✅ |

## 6. 事件 base 用字符串

```c
// ❌ WRONG
esp_event_handler_register("RMAKER_EVENT", ...);

// ✅ CORRECT — 用宏
esp_event_handler_register(RMAKER_EVENT, ESP_EVENT_ANY_ID, &event_handler, NULL);
// 还有 RMAKER_COMMON_EVENT、RMAKER_OTA_EVENT、APP_NETWORK_EVENT
```

## 7. NVS 未处理 NO_FREE_PAGES

```c
// ✅ 所有官方示例的标准写法
esp_err_t err = nvs_flash_init();
if (err == ESP_ERR_NVS_NO_FREE_PAGES || err == ESP_ERR_NVS_NEW_VERSION_FOUND) {
    ESP_ERROR_CHECK(nvs_flash_erase());
    err = nvs_flash_init();
}
ESP_ERROR_CHECK(err);
```

## 8. Local Control 与 on-network chal_resp 互斥

二者都使用 `protocomm_httpd`（单例），`Kconfig.projbuild` 明确：`depends on !ESP_RMAKER_LOCAL_CTRL_FEATURE_ENABLE`。不能同时 `=y`。

## 9. OTA 自定义回调失败未 report

```c
// ❌ WRONG — 云端状态卡住
return ESP_FAIL;

// ✅ CORRECT
esp_rmaker_ota_report_status(handle, OTA_STATUS_FAILED, "err");
return ESP_FAIL;
```

## 10. priv_data 生命周期

`priv_data` 必须覆盖设备整个生命周期（用 `static` 或堆），不能指向栈上局部变量。

## 11. MQTT 预算耗尽丢消息

默认预算 100，恢复 1/5 秒。高频上报需合并（`esp_rmaker_param_update()` + `esp_rmaker_report_updated_params()`）或调大 `CONFIG_ESP_RMAKER_MQTT_*_BUDGET`。

## 12. 高频上报用 update_and_report

```c
// ❌ WRONG — 每条都 report，预算秒光
for (int i = 0; i < 1000; i++)
    esp_rmaker_param_update_and_report(temp, esp_rmaker_float(read()));

// ✅ CORRECT
for (int i = 0; i < 1000; i++)
    esp_rmaker_param_update(temp, esp_rmaker_float(read()));
esp_rmaker_report_updated_params();
```

## 13. read 回调当业务逻辑用

头文件明确：read 回调当前不会被调用（客户端↔节点异步，read 请求不到节点）。业务依赖 write 回调与主动上报。

## 14. start 后动态加设备不重报

```c
// ✅ start 后增删设备要重报
esp_rmaker_node_add_device(node, new_dev);
esp_rmaker_report_node_details();
```

## 15. 场景反激活无回调

默认 `CONFIG_ESP_RMAKER_SCENES_DEACTIVATE_SUPPORT=n`，反激活不触发回调。需要时启用该配置。

## 16. bounds / valid_str 不生效

头文件明确：core 不检查 bounds 与 valid_str 值，应用须在 write 回调内自行校验。

## 17. ESP32-S2 无蓝牙

Assisted Claim 需要 BT，ESP32-S2 无 BT 只能用 Self Claim（其支持）或 No Claim。注意 Self Claim 在 ESP32/ESP32-C2 不可用，但 ESP32-S2 可用。

## 18. Thread 模式编译失败

需 `CONFIG_ESP_RMAKER_NETWORK_OVER_THREAD=y` 且 `OPENTHREAD_ENABLED`。Thread BR 示例见 `examples/thread_br/`。

## 19. 删除设备泄漏

先 `esp_rmaker_node_remove_device(node, dev)` 再 `esp_rmaker_device_delete(dev)`，顺序反了会失败。

## 20. 重名冲突

同一节点内设备名唯一，同一设备内参数名唯一。`esp_rmaker_device_get_param_by_type` 在多同名 type 时只返回第一个。
