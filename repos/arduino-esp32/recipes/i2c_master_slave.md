# I2C (Wire) 主机与从机

> **适用摘要**: 使用 `Wire` 库进行 I2C 主机读写和从机应答，含 `setPins`、`beginTransmission`、`requestFrom`、`onReceive`/`onRequest` 回调。

## 触发意图

- "I2C 读取传感器 / OLED"
- "Wire 主机通信"
- "I2C 从机模式"
- "自定义 SDA/SCL 引脚"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/Wire/examples/WireMaster/WireMaster.ino`、`WireSlave/WireSlave.ino` |
| 硬件 | SDA/SCL 需外接上拉电阻（典型 4.7 kΩ），电压依器件 |
| 默认引脚 | Generic ESP32：SDA=GPIO21，SCL=GPIO22（因板而异） |

## 分步说明

### 主机：写 + 读

`write()` 只是写缓冲，真正发送靠 `endTransmission()`。

```cpp
#include <Wire.h>
#define I2C_DEV_ADDR 0x68

void setup() {
    Serial.begin(115200);
    Wire.setPins(21, 22);          // 可选，在 begin 前
    Wire.begin();                  // 主机：无参

    // 写
    Wire.beginTransmission(I2C_DEV_ADDR);
    Wire.write(0x00);              // 寄存器
    Wire.write(0x01);
    uint8_t err = Wire.endTransmission(true);   // 0=成功
    Serial.printf("write err=%u\n", err);

    // 读
    Wire.requestFrom(I2C_DEV_ADDR, (uint8_t)6);
    while (Wire.available()) {
        Serial.printf("%02X ", Wire.read());
    }
}
void loop() {}
```

`endTransmission(sendStop)` 返回值：0 成功，1 数据太长，2 NACK 地址，3 NACK 数据，4 其它错误。
超时设置：`Wire.setTimeOut(50);`（毫秒，默认 50）；时钟：`Wire.setClock(100000);`。

### 从机：onReceive / onRequest

```cpp
#include <Wire.h>
#define I2C_DEV_ADDR 0x55

void onReceive(int len) {
    while (Wire.available()) {
        Serial.printf("rx 0x%02X\n", Wire.read());
    }
}
void onRequest() {
    Wire.write("hi");
}

void setup() {
    Serial.begin(115200);
    Wire.onReceive(onReceive);
    Wire.onRequest(onRequest);
    Wire.begin((uint8_t)I2C_DEV_ADDR, 21, 22, 100000);   // 从机：带地址
}
void loop() {}
```

> `slaveWrite()` 仅 ESP32 需要（兼容用），ESP32-S2/C3 等不必。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `setPins` 无效 | 在 `begin()` 之后调用 | 必须在 begin 前，或直接 `begin(sda,scl,freq)` |
| 全 NACK(返回 2) | 上拉缺失 / 地址错 / 接线 | 加上拉、用 I2C 扫描确认地址 |
| 时钟太高速率失败 | 走线/上拉不合适 | 降到 100 kHz 或 400 kHz，加驱动 |
| 从机不触发回调 | 未在 `begin` 前 `onReceive/onRequest` | 注册回调先于 `begin` |

## 参考

- `libraries/Wire/examples/WireMaster/WireMaster.ino`
- `libraries/Wire/examples/WireSlave/WireSlave.ino`
- 仓库文档 `docs/en/api/i2c.rst`
