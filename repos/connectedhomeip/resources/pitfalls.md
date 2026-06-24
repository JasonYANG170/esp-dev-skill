# connectedhomeip — Consolidated Pitfalls

> 所有"错误/正确"对比均可在仓库 `examples/` 中找到对应来源代码。

## 1. `app_main()` 初始化顺序必须为 nvs → CHIPDeviceManager → InitServer → StartAppTask

来源：`examples/lock-app/esp32/main/main.cpp`

```cpp
// ❌
extern "C" void app_main() {
    CHIPDeviceManager::GetInstance().Init(&cb);   // NVS 未就绪
}

// ✅
extern "C" void app_main() {
    nvs_flash_init();
    CHIPDeviceManager::GetInstance().Init(&EchoCallbacks);
    InitServer();
    GetAppTask().StartAppTask();
}
```

## 2. ESP32 设备固件用 `idf.py`，不要用 GN/ninja

GN/ninja 仅用于主机侧 chip-tool 与单元测试；设备固件走 ESP-IDF。

```bash
# ❌
gn gen out/esp32 && ninja -C out/esp32

# ✅
idf.py set-target esp32 && idf.py build
```

## 3. Rendezvous 模式是 Kconfig，改后必须 build + flash

```bash
idf.py menuconfig   # Demo -> Rendezvous Mode
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

## 4. BLE 配网必须启用 NimBLE

```ini
# sdkconfig.defaults
CONFIG_BT_ENABLED=y
CONFIG_BT_NIMBLE_ENABLED=y
```

## 5. 属性回调必须校验 endpoint/attribute

```cpp
// ❌
void PostAttributeChangeCallback(...) {
    BoltLockMgr().InitiateAction(0, BoltLockManager::LOCK_ACTION);  // 任何写入都触发
}

// ✅
VerifyOrExit(attributeId == ZCL_ON_OFF_ATTRIBUTE_ID, ...);
VerifyOrExit(endpointId == 1 || endpointId == 2, ...);
```

## 6. 服务端属性回写用 `emberAfWriteAttribute` + `gen/` 宏

```cpp
emberAfWriteAttribute(1, ZCL_ON_OFF_CLUSTER_ID, ZCL_ON_OFF_ATTRIBUTE_ID,
                      CLUSTER_MASK_SERVER, &v, ZCL_BOOLEAN_ATTRIBUTE_TYPE);
```

## 7. 默认配网凭据写死在示例：discriminator 3840 / pin 20202021

```bash
chip-tool pairing ble 20202021 3840
chip-device-ctrl > connect -ble 3840 20202021 135246
```

## 8. IP 变化事件要重启 mDNS

```cpp
case DeviceEventType::kInterfaceIpAddressChanged:
    if (... == InterfaceIpChangeType::kIpV4_Assigned || ... == kIpV6_Assigned) {
        chip::app::Mdns::StartServer();
    }
    break;
```

## 9. 应用任务查询栈状态必须加锁

```cpp
if (PlatformMgr().TryLockChipStack()) {
    sHaveServiceConnectivity = ConnectivityMgr().HaveServiceConnectivity();
    PlatformMgr().UnlockChipStack();
}
```

## 10. 自定义分区表，factory ≥ 1.9 MB

`partitions.csv`：`factory, app, factory, , 1945K,`。`CONFIG_PARTITION_TABLE_CUSTOM=y`。

## 11. 子模块必须初始化

```bash
git submodule update --init --recursive
```

## 12. ESP32 仅支持 2.4 GHz Wi-Fi

清除已存网络：`idf.py -p /dev/ttyUSB0 erase_flash`。

## 13. 不要手改 `<app/common/gen/*.h>`

ZAP 生成文件，改了会与上游数据模型不一致。属性操作走 `emberAfWriteAttribute` + 宏。

## 14. chip-tool endpoint 必须 1..240

```bash
chip-tool onoff on 1   # 正确
chip-tool onoff on 0   # 错误：越界
```

## 15. Bypass 模式仅用于调试

跳过 PASE 安全配对，不可用于生产网络。
