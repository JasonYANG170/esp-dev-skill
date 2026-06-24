# ESP-NOW 低延迟通信

> **适用摘要**: 使用 `ESP32_NOW` 库（`ESP_NOW` 类 + `ESP_NOW_Peer` 子类）在 ESP32 设备间做广播 / 点对点通信，无需 AP。⚠️ 必须先 `WiFi.mode()` 再 `ESP_NOW.begin()`。

## 触发意图

- "ESP-NOW 通信"
- "设备间低延迟通信"
- "无线广播消息"
- "无需路由器通信"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ESP_NOW/examples/ESP_NOW_Broadcast_Master/`、`ESP_NOW_Broadcast_Slave/`、`ESP_NOW_Serial/` |
| 依赖 | ESP-NOW 依赖 Wi-Fi，须先 `WiFi.mode()` 并固定 channel |
| 芯片 | P4/S2 无 Wi-Fi 时不可用（见支持矩阵） |

## 分步说明

### 广播 Master（取自仓库示例）

需自定义类继承 `ESP_NOW_Peer`，实现 `_onReceive` / `_onSent`（按需）。

```cpp
#include <Arduino.h>
#include "ESP32_NOW.h"
#include "WiFi.h"
#include <esp_mac.h>

#define ESPNOW_WIFI_CHANNEL 6

class ESP_NOW_Broadcast_Peer : public ESP_NOW_Peer {
public:
    ESP_NOW_Broadcast_Peer(uint8_t channel, wifi_interface_t iface, const uint8_t *lmk)
        : ESP_NOW_Peer(ESP_NOW.BROADCAST_ADDR, channel, iface, lmk) {}
    bool begin() { return ESP_NOW.begin() && add(); }
    bool send_message(const uint8_t *data, size_t len) { return send(data, len); }
};

ESP_NOW_Broadcast_Peer broadcast_peer(ESPNOW_WIFI_CHANNEL, WIFI_IF_STA, nullptr);

void setup() {
    Serial.begin(115200);
    WiFi.mode(WIFI_STA);
    WiFi.setChannel(ESPNOW_WIFI_CHANNEL);
    while (!WiFi.STA.started()) delay(100);

    if (!broadcast_peer.begin()) {
        Serial.println("ESP-NOW init failed"); delay(5000); ESP.restart();
    }
    Serial.printf("ESP-NOW v%d, max len=%d\n",
                  ESP_NOW.getVersion(), ESP_NOW.getMaxDataLen());
}
void loop() {
    static uint32_t n = 0;
    char msg[32]; snprintf(msg, sizeof(msg), "hello #%lu", (unsigned long)n++);
    broadcast_peer.send_message((const uint8_t *)msg, strlen(msg));
    delay(5000);
}
```

### 点对点单播（带 LMK 加密）

```cpp
uint8_t peer_mac[6] = {0xAA,0xBB,0xCC,0xDD,0xEE,0xFF};
uint8_t lmk[16] = { /* 16 字节本地主密钥 */ };

class MyPeer : public ESP_NOW_Peer {
public:
    MyPeer(const uint8_t *mac, uint8_t ch, const uint8_t *lmk)
        : ESP_NOW_Peer(mac, ch, WIFI_IF_STA, lmk) {}
    void onReceive(const uint8_t *data, int len, bool broadcast) override {
        Serial.printf("rx %d bytes (bcast=%d)\n", len, broadcast);
    }
    void onSent(bool ok) override { Serial.printf("sent %s\n", ok ? "ok" : "fail"); }
};

MyPeer peer(peer_mac, ESPNOW_WIFI_CHANNEL, lmk);
// 在 setup() 内：peer.add();  peer.send(data, len);
```

`ESP_NOW_Peer` 构造：`ESP_NOW_Peer(const uint8_t *mac_addr, uint8_t channel, wifi_interface_t iface, const uint8_t *lmk)`；方法 `add()` / `remove()` / `send(data,len)` / `addr()` / `setChannel()` / `setKey(lmk)` / `isEncrypted()`。

### 接收未知 peer

```cpp
ESP_NOW.onNewPeer([](const esp_now_recv_info_t *info, const uint8_t *data, int len, void *arg){
    Serial.printf("new peer, %d bytes\n", len);
}, nullptr);
```

其它 `ESP_NOW` 类 API：`begin(pmk=NULL)` / `end()` / `getTotalPeerCount()` / `getEncryptedPeerCount()` / `onNewPeer(cb, arg)`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `ESP_NOW.begin()` 失败 | Wi-Fi 未初始化 | 先 `WiFi.mode()` 并 `setChannel`，等 `STA.started()` |
| peer `add()` 失败 | peer 满或 MAC 无效 | `getTotalPeerCount()` 检查；广播用 `ESP_NOW.BROADCAST_ADDR` |
| 收不到 | 双方信道不一致 | 统一 channel（与已连 AP 信道冲突时以 AP 为准） |
| 加密失败 | LMK 非 16 字节 | LMK 固定 16 字节；PMK（会话级）通过 `ESP_NOW.begin(pmk)` |

## 参考

- `libraries/ESP_NOW/examples/ESP_NOW_Broadcast_Master/ESP_NOW_Broadcast_Master.ino`
- `libraries/ESP_NOW/examples/ESP_NOW_Broadcast_Slave/ESP_NOW_Broadcast_Slave.ino`
- `libraries/ESP_NOW/examples/ESP_NOW_Serial/ESP_NOW_Serial.ino`
- 仓库文档 `docs/en/api/espnow.rst`
