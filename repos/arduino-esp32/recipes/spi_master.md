# SPI 主机通信

> **适用摘要**: 使用 `SPI` 库在多总线上以主机方式与外设（传感器、SD、显示屏等）通信，含 `SPIClass` 自定义引脚与多总线复用。

## 触发意图

- "SPI 读取传感器 / 显示屏"
- "SPI 自定义 SCK/MOSI/MISO/CS"
- "多总线 SPI"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/SPI/examples/SPI_Multiple_Buses/SPI_Multiple_Buses.ino` |
| 总线 | FSPI / HSPI（ESP32/S3 等提供 2~ 多条 SPI 总线，因 SoC 而异） |
| 参考 | 标准 Arduino SPI API：`https://docs.arduino.cc/language-reference/en/functions/communication/SPI/` |

## 分步说明

### 默认 SPI（VSPI/FSPI）

```cpp
#include <SPI.h>

void setup() {
    SPI.begin();                       // 默认 SCK/MOSI/MISO/SS（见 variants/pins_arduino.h）
    digitalWrite(SS, LOW);
    SPI.beginTransaction(SPISettings(1000000, MSBFIRST, SPI_MODE0));
    SPI.transfer(0x9F);                // 读 JEDEC ID 示例
    uint8_t b1 = SPI.transfer(0x00);
    uint8_t b2 = SPI.transfer(0x00);
    SPI.endTransaction();
    digitalWrite(SS, HIGH);
}
void loop() {}
```

### 自定义引脚 / 多总线（SPIClass）

```cpp
#include <SPI.h>
#define SCK1  18
#define MISO1 19
#define MOSI1 23
#define CS1   5

SPIClass spi1(FSPI);   // 或 HSPI

void setup() {
    spi1.begin(SCK1, MISO1, MOSI1, CS1);
    digitalWrite(CS1, LOW);
    spi1.beginTransaction(SPISettings(2000000, MSBFIRST, SPI_MODE3));
    spi1.transfer(0xAB);
    spi1.endTransaction();
    digitalWrite(CS1, HIGH);
}
void loop() {}
```

> SPI 模式 `SPI_MODE0`..`SPI_MODE3`；位序 `MSBFIRST` / `LSBFIRST`。`SPISettings(freq, bitOrder, mode)` 每次事务前显式声明。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 读全 0 或 0xFF | CS 未拉低 / 模式不对 / 接线 | 检查 CS 与 SPI_MODE、MISO/MOSI 是否接反 |
| 多器件互相干扰 | 共用 CS 或未 `endTransaction` | 每器件独立 CS，事务成对调用 |
| 高速失败 | 走线长 | 降频，缩短连线 |

## 参考

- `libraries/SPI/examples/SPI_Multiple_Buses/SPI_Multiple_Buses.ino`
- 仓库文档 `docs/en/api/spi.rst`
