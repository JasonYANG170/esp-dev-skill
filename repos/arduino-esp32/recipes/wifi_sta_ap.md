# Wi-Fi STA / SoftAP / 扫描 / 事件

> **适用摘要**: 用 `WiFi` 库以 Station 模式连接 AP、SoftAP 开热点、扫描周边网络、`onEvent` 接收连接事件，含 `setHostname` 时机与重连。

> Version: Arduino-ESP32 core version and selected board package.
> Evidence: `repos/arduino-esp32/resources/`, source/examples in `repos/arduino-esp32/`, and this recipe path `repos/arduino-esp32/recipes/wifi_sta_ap.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "连 Wi-Fi / WiFi 连接"
- "开热点 / SoftAP"
- "扫描 Wi-Fi 网络"
- "WiFi 事件回调"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/WiFi/examples/WiFiClient/WiFiClient.ino`、`WiFiAccessPoint/WiFiAccessPoint.ino`、`WiFiScan/WiFiScan.ino`、`WiFiClientEvents/WiFiClientEvents.ino` |
| 限制 | `setHostname` 必须在 `mode`/`begin` 之前；回调运行在独立线程 |

## 分步说明

### Station 连接

```cpp
#include <WiFi.h>

const char *ssid = "your-ssid";
const char *password = "your-password";

void setup() {
    Serial.begin(115200);
    WiFi.setHostname("myesp32");          // 必须在 mode/begin 之前
    WiFi.mode(WIFI_STA);
    WiFi.begin(ssid, password);
    while (WiFi.status() != WL_CONNECTED) {
        delay(500); Serial.print(".");
    }
    Serial.print("\nIP: "); Serial.println(WiFi.localIP());
    WiFi.setAutoReconnect(true);
}
void loop() {}
```

完整 `begin`：`wl_status_t begin(const char* ssid, const char* passphrase=NULL, int32_t channel=0, const uint8_t* bssid=NULL, bool tryConnect=true);`
静态 IP：`WiFi.config(local_ip, gateway, subnet, dns1, dns2);`（在 `begin` 前）。

### SoftAP（热点）

```cpp
WiFi.softAP("my-ssid", "password");          // 密码需 >7 字符；NULL=开放
IPAddress ap = WiFi.softAPIP();              // 默认 192.168.4.1
Serial.printf("AP IP: %s\n", ap.toString().c_str());
Serial.printf("clients: %u\n", WiFi.softAPgetStationNum());
```

完整：`bool softAP(const char* ssid, const char* passphrase=NULL, int channel=1, int ssid_hidden=0, int max_connection=4, bool ftm_responder=false);`（`ftm_responder` 仅 S2/C3）。
静态：`WiFi.softAPConfig(local_ip, gateway, subnet);`。

### 同时 STA + AP

```cpp
WiFi.mode(WIFI_AP_STA);
```

模式枚举：`WIFI_OFF`/`WIFI_STA`/`WIFI_AP`/`WIFI_AP_STA`（以及 `WIFI_MODE_NULL` 用于复位）。

### 扫描

```cpp
int n = WiFi.scanNetworks();                 // 同步；异步: scanNetworks(true)
for (int i = 0; i < n; ++i) {
    Serial.printf("%s (%d dBm) %s ch=%u\n",
        WiFi.SSID(i).c_str(), WiFi.RSSI(i),
        (WiFi.encryptionType(i) == WIFI_AUTH_OPEN) ? "open" : "enc",
        WiFi.channel(i));
}
WiFi.scanDelete();                           // 释放扫描结果内存
```

异步：`scanNetworks(async=true)` → `scanComplete()` 取结果数 → 同上读取 → `scanDelete()`。

### 事件回调（独立线程）

```cpp
WiFi.onEvent([](arduino_event_id_t id, arduino_event_info_t info) {
    switch (id) {
        case ARDUINO_EVENT_WIFI_STA_GOT_IP:
            Serial.println(WiFi.localIP());
            break;
        case ARDUINO_EVENT_WIFI_STA_DISCONNECTED:
            Serial.println("disconnected, reconnecting");
            WiFi.reconnect();
            break;
        default: break;
    }
});
```

> ⚠️ 回调在独立 FreeRTOS 任务运行，访问共享变量需加锁；不可在回调里调用 `WiFi.onEvent/removeEvent`（非线程安全）。`Serial.print` 线程安全。
> 可选最低安全级别：`WiFi.setMinSecurity(WIFI_AUTH_WPA2_PSK);`（默认）。多 AP：用 `WiFiMulti` 的 `addAP(...)` + `run()`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| hostname 不生效 | 在 `begin` 后调用 | 先 `setHostname` 再 `mode`/`begin`；运行中改需 `mode(WIFI_MODE_NULL)` 复位 |
| 连不上 | 信号弱 / 密码错 / 5GHz | ESP32 仅 2.4GHz；`WiFi.setAutoReconnect(true)` |
| 回调数据竞争 | 在回调直接改全局 | 加锁或仅线程安全操作 |
| `softAP` 失败返回 false | 密码太短（<8）或信道冲突 | 密码 ≥8 字符，换信道 |
| 内存涨 | 扫描结果未释放 | 用完 `scanDelete()` |

## 参考

- `libraries/WiFi/examples/WiFiClient/WiFiClient.ino`
- `libraries/WiFi/examples/WiFiAccessPoint/WiFiAccessPoint.ino`
- `libraries/WiFi/examples/WiFiScan/WiFiScan.ino`
- `libraries/WiFi/examples/WiFiClientEvents/WiFiClientEvents.ino`
- 仓库文档 `docs/en/api/wifi.rst`
