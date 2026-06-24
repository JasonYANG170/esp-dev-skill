# esp-matter 陷阱汇总

> 这些是导致 Matter 设备不开机、不入网、不���报的最常见错误。每条给出错误写法与正确写法。代码取自或对照仓库示例与头文件。

## 1. `app_attribute_update_cb` 对未处理属性必须返回 `ESP_OK`

```cpp
// ❌
return ESP_ERR_NOT_SUPPORTED;   // 拒掉该 endpoint 上所有未处理属性

// ✅
if (type == PRE_UPDATE) { return app_driver_attribute_update(...); }
return ESP_OK;
```

## 2. endpoint 必须在 `esp_matter::start()` 之前创建

```cpp
// ❌
esp_matter::start(app_event_cb);
endpoint_t *ep = on_off_light::create(node, ...);   // 不会注册到 server

// ✅
node_t *node = node::create(...);
endpoint_t *ep = on_off_light::create(node, ...);
esp_matter::start(app_event_cb);
```

## 3. NVS 必须最先初始化

```cpp
// ❌ 直接 node::create / start，fabric/access 存储读写崩溃

// ✅ app_main 第一句
nvs_flash_init();
```

## 4. 跨任务访问 Matter 栈必须切换上下文

```cpp
// ❌ 传感器 task / 定时器直接 attribute::update

// ✅ ScheduleLambda（首选，传感器驱动）
chip::DeviceLayer::SystemLayer().ScheduleLambda([&]() {
    attribute::update(ep_id, ..., &val);
});

// ✅ 显式锁（长任务）
esp_matter::lock::ScopedChipStackLock lock(portMAX_DELAY);
chip::Server::GetInstance().GetFabricTable().DeleteFabric(...);
```

## 5. 读属性两步走：`get` 拿句柄，`get_val` 取值

```cpp
// ❌ 没有 attribute::read() 直接返回值

// ✅
attribute_t *a = attribute::get(ep, OnOff::Id, OnOff::Attributes::OnOff::Id);
esp_matter_attr_val_t v;
attribute::get_val(a, &v);
bool on = v.val.b;
```

## 6. 写回数据模型用 `update()`，不要 `set_val`

```cpp
// ❌ attribute::set_val 不触发回调也不上报

// ✅
attribute::update(ep, OnOff::Id, OnOff::Attributes::OnOff::Id, &val);
// 仅想通知订阅者、不要本地回调时用 attribute::report(...)
```

## 7. `priv_data` 是回调里拿回驱动句柄的唯一通道

```cpp
// ❌ 多 endpoint 用全局 handle

// ✅
endpoint_t *ep = extended_color_light::create(node, &cfg, ENDPOINT_FLAG_NONE, (void *)light_handle);
// 回调里:
led_driver_handle_t h = (led_driver_handle_t)priv_data;
```

## 8. nullable 字段用 `nullptr` 表达"无 start-up"

```cpp
// ❌
cfg.on_off_lighting.start_up_on_off = 0;

// ✅
cfg.on_off_lighting.start_up_on_off = nullptr;
cfg.color_control_color_temperature.start_up_color_temperature_mireds = nullptr;
```

## 9. Matter 单位不是硬件单位，必须重映射

```cpp
// ❌ 直接把 Matter hue(0-254) 给 0-360 的 LED
led_driver_set_hue(h, val->val.u8);

// ✅（来自 examples/light/app_priv.h）
int v = REMAP_TO_RANGE(val->val.u8, MATTER_HUE, STANDARD_HUE);
led_driver_set_hue(h, v);
// 温度是 mireds，反演回 kelvin:
uint32_t k = REMAP_TO_RANGE_INVERSE(val->val.u16, STANDARD_TEMPERATURE_FACTOR);
```

## 10. 频繁变化的属性要设 deferred persistence

```cpp
// ❌ 每拖一次亮度都落盘

// ✅ 一次设定（来自 examples/light/app_main.cpp）
attribute_t *cur = attribute::get(ep, LevelControl::Id, LevelControl::Attributes::CurrentLevel::Id);
attribute::set_deferred_persistence(cur);
```

## 11. 测试凭据与工厂分区二选一

```text
# ❌ 关了测试凭据又没烧 fctry → 设备无 passcode/discriminator，入网超时
CONFIG_ENABLE_TEST_SETUP_PARAMS=n

# ✅ 开发：CONFIG_ENABLE_TEST_SETUP_PARAMS=y（passcode 20202021 / disc 3840）
# ✅ 量产：烧 mfg_tool 生成的 fctry 分区 + CONFIG_FACTORY_*_PROVIDER=y，并关 CONFIG_ENABLE_TEST_SETUP_PARAMS
```

## 12. 多模芯片 Wi-Fi/Thread 必须显式二选一

```text
# ❌ esp32c6 同时开两者且 mdns 配置不匹配
CONFIG_OPENTHREAD_ENABLED=y
CONFIG_ENABLE_WIFI_STATION=y
CONFIG_USE_MINIMAL_MDNS=y   # Thread 应为 n

# ✅ Thread: OT=y, WIFI_STATION=n, MINIMAL_MDNS=n
# ✅ Wi-Fi : OT=n, WIFI_STATION=y, MINIMAL_MDNS=y
```

## 13. Custom Provider 必须在 `esp_matter::start()` 之前注入

```cpp
// ❌
esp_matter::start(app_event_cb);
set_custom_dac_provider(my_dac);   // 无效

// ✅
set_custom_dac_provider(my_dac);
set_custom_commissionable_data_provider(my_comm);
esp_matter::start(app_event_cb);
```

## 14. 加密 OTA 私钥 buffer 生命周期到关机

```cpp
// ❌ 栈/堆上分配用完释放 → OTA 解密时段错误

// ✅ 嵌入为二进制符号（来自 examples/light/app_main.cpp）
extern const char key_start[] asm("_binary_esp_image_encryption_key_pem_start");
extern const char key_end[]   asm("_binary_esp_image_encryption_key_pem_end");
esp_matter_ota_requestor_encrypted_init(key_start, key_end - key_start);
```

## 15. 加密 OTA 初始化在 start 之后

```cpp
// ❌ esp_matter_ota_requestor_encrypted_init() 在 esp_matter::start() 之前

// ✅
esp_matter::start(app_event_cb);
esp_matter_ota_requestor_encrypted_init(key, len);
```

## 16. Bridged endpoint 要有 aggregator 父端

```cpp
// ❌ 直接 bridged_node::create 没父 → 客户端发现异常

// ✅
aggregator::config_t agg_cfg;
endpoint_t *agg = aggregator::create(node, &agg_cfg, ENDPOINT_FLAG_NONE, nullptr);
// 动态 bridged endpoint 的 parent 指向该 aggregator（运行时由 app_bridge 添加）
```

## 17. 首次烧录前 `erase_flash`

```bash
# ❌ 残留 NVS/fabric 导致入网失败
idf.py flash monitor

# ✅
idf.py erase_flash flash monitor
```

## 18. 入网后释放 BLE 省内存

```cpp
// ❌ 入网完成仍保留 NimBLE（白占 ~30KB）

// ✅ 在 kCommissioningComplete 分支调 app_ble_disable()
//（见 examples/light/README.md 的内存说明）
```

## 19. VID/PID 三处一致

```text
# CD（chip-cert gen-cd -V/-p）、DAC（mfg-tool -v/-p）、Basic cluster（basic_information 配置）
# 三处 VID/PID 必须相同，否则 attestation 失败。
```

## 20. window_covering 的 EndProductType 构造后不可改

```cpp
// ❌ 构造后又赋值
window_covering::config_t cfg;
cfg.end_product_type = ...;   // 字段为 const

// ✅ 构造时传入
window_covering::config_t cfg(static_cast<uint8_t>(
    chip::app::Clusters::WindowCovering::EndProductType::kTiltOnlyInteriorBlind));
```
