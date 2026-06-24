# BLE 蓝牙低功耗（扫描 / 服务端 / 客户端 / UART / Notify / Beacon）

> **适用摘要**: 使用 `BLEDevice` 库实现 BLE 扫描、GATT Server/Client、UART 透传、Notify 通知、iBeacon/Eddystone 广播。⚠️ 3.x 中 ESP32 仍用 Bluedroid，其余 SoC（C3/C5/C6/H2/S3）默认走 NimBLE；4.0 起 Bluedroid 将被移除。

## 触发意图

- "BLE 扫描 / BLE scan"
- "BLE 服务端 / GATT server / 特征值读写"
- "BLE 客户端 / 连接远程设备"
- "BLE UART 透传 / Notify 通知"
- "iBeacon / Eddystone 信标"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/BLE/examples/Scan/`、`Server/`、`Client/`、`UART/`、`Notify/`、`iBeacon/`、`Beacon_Scanner/` |
| 协议栈 | ESP32 = Bluedroid（同时支持 BT Classic）；C3/C5/C6/H2/S3 = NimBLE（仅 BLE）；用 `BLEDevice::getBLEStackString()` 判断 |
| 内存 | NimBLE 占用更小；Bluedroid 需更多 flash/RAM |
| include | `<BLEDevice.h>` `<BLEUtils.h>` `<BLEServer.h>`（+ `<BLEScan.h>` `<BLEAdvertisedDevice.h>` 扫描 / `<BLE2902.h>` CCCD 描述符） |

## 分步说明

### 扫描周边 BLE 设备（取自 Scan 示例）

```cpp
#include <Arduino.h>
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEScan.h>
#include <BLEAdvertisedDevice.h>

int scanTime = 5;   // 秒
BLEScan *pBLEScan;

class MyAdvertisedDeviceCallbacks : public BLEAdvertisedDeviceCallbacks {
  void onResult(BLEAdvertisedDevice advertisedDevice) {
    Serial.printf("Advertised Device: %s \n", advertisedDevice.toString().c_str());
  }
};

void setup() {
  Serial.begin(115200);
  BLEDevice::init("");
  pBLEScan = BLEDevice::getScan();
  pBLEScan->setAdvertisedDeviceCallbacks(new MyAdvertisedDeviceCallbacks());
  pBLEScan->setActiveScan(true);   // 主动扫描更快但更耗电
  pBLEScan->setInterval(100);
  pBLEScan->setWindow(99);         // 必须 <= setInterval
}
void loop() {
  BLEScanResults *foundDevices = pBLEScan->start(scanTime, false);
  Serial.printf("Devices found: %d\n", foundDevices->getCount());
  pBLEScan->clearResults();        // 释放扫描结果内存
  delay(2000);
}
```

### GATT Server（特征值读写，取自 Server 示例）

```cpp
#include <Arduino.h>
#include <BLEDevice.h>
#include <BLEUtils.h>
#include <BLEServer.h>

#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID "beb5483e-36e1-4688-b7f5-ea07361b26a8"

void setup() {
  Serial.begin(115200);

  if (!BLEDevice::init("BLE Server Example")) {
    Serial.println("BLE initialization failed!");
    return;
  }

  BLEServer *pServer = BLEDevice::createServer();
  BLEService *pService = pServer->createService(SERVICE_UUID);
  pServer->advertiseOnDisconnect(true);   // 断开后自动恢复广播

  BLECharacteristic *pCharacteristic = pService->createCharacteristic(
    CHARACTERISTIC_UUID,
    BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_WRITE);
  pCharacteristic->setValue("Hello World says Neil");

  pService->start();
  BLEAdvertising *pAdvertising = BLEDevice::getAdvertising();
  pAdvertising->addServiceUUID(SERVICE_UUID);
  pAdvertising->setScanResponse(true);
  pAdvertising->setMinPreferred(0x06);     // 有助于 iPhone 连接
  pAdvertising->setMaxPreferred(0x12);
  BLEDevice::startAdvertising();
}
void loop() { delay(2000); }
```

特征值属性（按位或）：`PROPERTY_READ` / `PROPERTY_WRITE` / `PROPERTY_WRITE_NR`（无响应写）/ `PROPERTY_NOTIFY` / `PROPERTY_INDICATE` / `PROPERTY_BROADCAST`。NimBLE 额外支持 `PROPERTY_READ_ENC`/`_WRITE_ENC`（加密）、`PROPERTY_READ_AUTHEN`/`_WRITE_AUTHEN`（MITM 鉴权）；Bluedroid 忽略这些，改用 `setAccessPermissions(ESP_GATT_PERM_READ_ENC_MITM | ...)`。

### Notify 通知（含 CCCD 2902 / User Description 2901，取自 Notify 示例）

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <BLE2901.h>

BLEServer *pServer = NULL;
BLECharacteristic *pCharacteristic = NULL;
bool deviceConnected = false, oldDeviceConnected = false;
uint32_t value = 0;

#define SERVICE_UUID        "4fafc201-1fb5-459e-8fcc-c5c9c331914b"
#define CHARACTERISTIC_UUID "beb5483e-36e1-4688-b7f5-ea07361b26a8"

class MyServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer *pServer) { deviceConnected = true; }
  void onDisconnect(BLEServer *pServer) { deviceConnected = false; }
};

void setup() {
  Serial.begin(115200);
  BLEDevice::init("ESP32");
  pServer = BLEDevice::createServer();
  pServer->setCallbacks(new MyServerCallbacks());

  BLEService *pService = pServer->createService(SERVICE_UUID);
  pCharacteristic = pService->createCharacteristic(
    CHARACTERISTIC_UUID,
    BLECharacteristic::PROPERTY_READ | BLECharacteristic::PROPERTY_WRITE |
    BLECharacteristic::PROPERTY_NOTIFY | BLECharacteristic::PROPERTY_INDICATE);

  // CCCD 0x2902：NimBLE 会按属性自动加，Bluedroid 需手动 add
  pCharacteristic->addDescriptor(new BLE2902());
  // 用户描述符 0x2901
  BLE2901 *d2901 = new BLE2901();
  d2901->setDescription("My own description for this characteristic.");
  d2901->setAccessPermissions(ESP_GATT_PERM_READ);
  pCharacteristic->addDescriptor(d2901);

  pService->start();
  BLEAdvertising *pAdv = BLEDevice::getAdvertising();
  pAdv->addServiceUUID(SERVICE_UUID);
  pAdv->setScanResponse(false);
  pAdv->setMinPreferred(0x0);
  BLEDevice::startAdvertising();
}
void loop() {
  if (deviceConnected) {
    pCharacteristic->setValue((uint8_t *)&value, 4);
    pCharacteristic->notify();
    value++;
    delay(500);
  }
  if (!deviceConnected && oldDeviceConnected) {
    delay(500);                       // 给协议栈时间整理
    pServer->startAdvertising();      // 断开后重新广播
    oldDeviceConnected = false;
  }
  if (deviceConnected && !oldDeviceConnected) oldDeviceConnected = true;
}
```

### GATT Client（连接 + 读 + 订阅通知，取自 Client 示例）

```cpp
#include <Arduino.h>
#include "BLEDevice.h"

static BLEUUID serviceUUID("4fafc201-1fb5-459e-8fcc-c5c9c331914b");
static BLEUUID charUUID("beb5483e-36e1-4688-b7f5-ea07361b26a8");
static boolean doConnect = false, connected = false, doScan = false;
static BLERemoteCharacteristic *pRemoteCharacteristic;
static BLEAdvertisedDevice *myDevice;

static void notifyCallback(BLERemoteCharacteristic *p, uint8_t *pData, size_t length, bool isNotify) {
  Serial.printf("Notify (%d bytes): ", (int)length);
  Serial.write(pData, length); Serial.println();
}

bool connectToServer() {
  BLEClient *pClient = BLEDevice::createClient();
  pClient->connect(myDevice);     // 传 BLEAdvertisedDevice 可自动识别 public/private 地址
  pClient->setMTU(517);           // 请求最大 MTU（默认 23）

  BLERemoteService *pRemoteService = pClient->getService(serviceUUID);
  if (!pRemoteService) { pClient->disconnect(); return false; }
  pRemoteCharacteristic = pRemoteService->getCharacteristic(charUUID);
  if (!pRemoteCharacteristic) { pClient->disconnect(); return false; }

  if (pRemoteCharacteristic->canRead())
    Serial.println(pRemoteCharacteristic->readValue().c_str());
  if (pRemoteCharacteristic->canNotify())
    pRemoteCharacteristic->registerForNotify(notifyCallback);

  connected = true;
  return true;
}

class MyAdvertisedDeviceCallbacks : public BLEAdvertisedDeviceCallbacks {
  void onResult(BLEAdvertisedDevice advertisedDevice) {
    if (advertisedDevice.haveServiceUUID() &&
        advertisedDevice.isAdvertisingService(serviceUUID)) {
      BLEDevice::getScan()->stop();
      myDevice = new BLEAdvertisedDevice(advertisedDevice);
      doConnect = doScan = true;
    }
  }
};

void setup() {
  Serial.begin(115200);
  BLEDevice::init("");
  BLEScan *pBLEScan = BLEDevice::getScan();
  pBLEScan->setAdvertisedDeviceCallbacks(new MyAdvertisedDeviceCallbacks());
  pBLEScan->setInterval(1349); pBLEScan->setWindow(449);
  pBLEScan->setActiveScan(true);
  pBLEScan->start(5, false);
}
void loop() {
  if (doConnect) { connectToServer(); doConnect = false; }
  if (connected) {
    String v = "Time since boot: " + String(millis() / 1000);
    pRemoteCharacteristic->writeValue(v.c_str(), v.length());
  } else if (doScan) {
    BLEDevice::getScan()->start(0);   // 断开后继续扫描
  }
  delay(1000);
}
```

### BLE UART（Nordic UART Service，取自 UART 示例）

UUID 固定：`6E400001-...`（服务）/ `6E400002-...`（RX，WRITE）/ `6E400003-...`（TX，NOTIFY）。TX 特征值发 `setValue(&byte,1)` + `notify()`；RX 特征值实现 `BLECharacteristicCallbacks::onWrite` 读 `pCharacteristic->getValue()`。

### iBeacon 广播（取自 iBeacon 示例）

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>
#include <BLEBeacon.h>

#define BEACON_UUID_REV "A134D0B2-1DA2-1BA7-C94C-E8E00C9F7A2D"

void init_beacon(BLEServer *pServer) {
  BLEAdvertising *pAdvertising = pServer->getAdvertising();
  pAdvertising->stop();

  BLEBeacon myBeacon;
  myBeacon.setManufacturerId(0x4c00);   // Apple
  myBeacon.setMajor(5);
  myBeacon.setMinor(88);
  myBeacon.setSignalPower(0xc5);        // 1m 处 RSSI（校准值）
  myBeacon.setProximityUUID(BLEUUID(BEACON_UUID_REV));

  BLEAdvertisementData adv;
  adv.setFlags(0x1A);
  adv.setManufacturerData(myBeacon.getData());
  pAdvertising->setAdvertisementData(adv);
  pAdvertising->start();
}
```

Eddystone URL/TLM 信标用 `BLEEddystoneURL` / `BLEEddystoneTLM`（见 `Beacon_Scanner` 示例解析端，`getFrameType()` 判帧类型）。

### BLESecurity（配对与绑定，取自库 README）

```cpp
#include <BLESecurity.h>

BLESecurity *pSecurity = new BLESecurity();
pSecurity->setCapability(ESP_IO_CAP_KBDISP);          // 支持 MITM 的所有方法
pSecurity->setAuthenticationMode(ESP_LE_AUTH_REQ_SC_MITM_BOND);  // Secure Connections + MITM + 绑定
pSecurity->setPassKey(true, 123456);                  // static=true 静态密钥
```

IO 能力：`ESP_IO_CAP_NONE`/`_OUT`/`_IN`/`_IO`/`_KBDISP`；鉴权模式：`ESP_LE_AUTH_NO_BOND`/`_BOND`/`_REQ_MITM`/`_REQ_SC_MITM_BOND`。

> ⚠️ NimBLE 的权限通过特征值属性（`PROPERTY_READ_AUTHEN` 等）设置，Bluedroid 通过 `setAccessPermissions(ESP_GATT_PERM_*)`；同时写两种可跨栈兼容。开发期清除绑定缓存：`nvs_flash_erase(); nvs_flash_init();`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| iPhone 连不上 / 广播看不见 | `setMinPreferred(0x0)` 未设 | 设 `setMinPreferred(0x06)` + `setMaxPreferred(0x12)`；广播+扫描响应总长 ≤31B（BLE 4.0） |
| Notify 收不到 | CCCD 2902 未加（Bluedroid） | `pCharacteristic->addDescriptor(new BLE2902())`；NimBLE 自动加 |
| MTU 太小（23） | 默认值，大包分片多 | Client 端 `pClient->setMTU(517)`；服务端 `BLEDevice::setMTU(517)` |
| 配对只用 Just Works | IO 能力为 NONE / 缺 MITM flag | `setCapability(ESP_IO_CAP_KBDISP)` + `setAuthenticationMode(ESP_LE_AUTH_REQ_SC_MITM_BOND)` |
| 配对一次成功后失败 | NVS 绑定缓存损坏 | 开发期 `nvs_flash_erase()`；静态密钥避免默认 123456 |
| Bluedroid 编译失败 | 4.0+ 移除 Bluedroid | 切 NimBLE 或用 ESP-IDF 组件方式 |
| Client 连接后 `getService` 返回 null | UUID 不匹配 / MTU 太小 | 核对服务端 UUID；先 `setMTU` 再 `getService` |

## 参考

- `libraries/BLE/examples/Scan/Scan.ino`、`Server/Server.ino`、`Client/Client.ino`
- `libraries/BLE/examples/UART/UART.ino`、`Notify/Notify.ino`
- `libraries/BLE/examples/iBeacon/iBeacon.ino`、`Beacon_Scanner/Beacon_Scanner.ino`
- `libraries/BLE/examples/EddystoneURL_Beacon/`、`EddystoneTLM_Beacon/`
- `libraries/BLE/examples/Server_secure_static_passkey/`、`Client_secure_static_passkey/`
- 仓库文档 `docs/en/api/ble.rst`、`libraries/BLE/README.md`（安全/配对详解）
