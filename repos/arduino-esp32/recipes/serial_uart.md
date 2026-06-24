# Serial (UART) 串口通信

> **适用摘要**: 配置 ESP32 硬件 UART（`Serial`/`Serial1`/`Serial2`），包括自定义引脚、波特率、RX 缓冲区、`onReceive` 回调与 RS485 半双工模式。

## 触发意图

- "串口初始化 / printf 调试"
- "Serial1 自定义 RX/TX 引脚"
- "UART 接收回调 onReceive"
- "RS485 通信"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `libraries/ESP32/examples/Serial/OnReceive_Demo/OnReceive_Demo.ino`、`RS485_Echo_Demo/RS485_Echo_Demo.ino` |
| UART 数量 | 因 SoC 而异：ESP32=3、S2=2、S3=3、C3=2、C5/C6=2 HP+1 LP、P4=5 HP+1 LP |
| LP UART | 引脚固定，不可用 `setPins()` 改 |

## 分步说明

### 基本初始化（UART0 = Serial）

```cpp
void setup() {
    Serial.begin(115200);     // 默认 8N1，UART0 默认引脚
    Serial.printf("Hello ESP32\n");
}
void loop() {}
```

串行格式常量：`SERIAL_8N1`(默认)/`SERIAL_8N2`/`SERIAL_8E1`/`SERIAL_8O1` 等，以及 5/6/7 位变体（如 `SERIAL_7E1`）。

### 自定义引脚 + 缓冲区（Serial1）

`setPins` 与 `setRxBufferSize` 必须在 `begin()` **之前**调用。

```cpp
#define RX1 16
#define TX1 17

void setup() {
    Serial.begin(115200);
    Serial1.setRxBufferSize(1024);          // before begin
    Serial1.setPins(RX1, TX1);              // before begin（也可在 begin 里传 rxPin,txPin）
    Serial1.begin(9600, SERIAL_8N1);
}

void loop() {
    if (Serial1.available()) {
        Serial.write(Serial1.read());
    }
}
```

等价写法：`Serial1.begin(9600, SERIAL_8N1, RX1, TX1)`。

### onReceive 回调（独立任务上下文）

回调在单独 FreeRTOS 任务触发，`Serial.print` 线程安全可用。

```cpp
void onRx() {
    while (Serial1.available()) {
        uint8_t b = Serial1.read();
        Serial.printf("0x%02X ", b);
    }
}

void setup() {
    Serial.begin(115200);
    Serial1.begin(115200);
    Serial1.onReceive(onRx);                 // FIFO 满 或 RX 超时均触发
    // Serial1.onReceive(onRx, true);        // 仅 RX 超时触发（一次拿到整流）
}
void loop() {}
```

错误回调：`Serial1.onReceiveError([](hardwareSerial_error_t e){ ... })`，
错误类型：`UART_NO_ERROR` / `UART_BREAK_ERROR` / `UART_BUFFER_FULL_ERROR` / `UART_FIFO_OVF_ERROR` / `UART_FRAME_ERROR` / `UART_PARITY_ERROR`。

### RS485 半双工

RTS 引脚控制收发器方向，需先用 `setPins` 指定 RTS。

```cpp
Serial1.setPins(RX1, TX1, /*cts*/ -1, /*rts*/ DE_PIN);
Serial1.begin(9600);
Serial1.setMode(UART_MODE_RS485_HALF_DUPLEX);   // setMode 在 begin 之后
Serial1.println("ping");
```

`SerialMode` 取值：`UART_MODE_UART`(默认) / `UART_MODE_RS485_HALF_DUPLEX` / `UART_MODE_IRDA` / `UART_MODE_RS485_COLLISION_DETECT` / `UART_MODE_RS485_APP_CTRL`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `setRxBufferSize`/`setPins` 无效 | 在 `begin()` 之后调用 | 必须在 `begin()` 之前 |
| 收不到数据 | 引脚接错或被其它外设占用（如 Wire.begin 用了 RX/TX） | 检查引脚，UART 会自动 detach |
| 高波特率乱码 | 时钟源限制（ESP32/S2 低波特用 REF_TICK） | `setClockSource(UART_CLK_SRC_APB)` 后再 begin |
| 回调访问共享变量异常 | 回调在独立线程 | 加锁或仅做线程安全操作 |
| LP UART 改不了引脚 | LP UART 引脚固定 | 改用 HP UART |

## 参考

- `libraries/ESP32/examples/Serial/OnReceive_Demo/OnReceive_Demo.ino`
- `libraries/ESP32/examples/Serial/RS485_Echo_Demo/RS485_Echo_Demo.ino`
- `libraries/ESP32/examples/Serial/HardwareFlowControl_Demo/HardwareFlowControl_Demo.ino`
- 仓库文档 `docs/en/api/serial.rst`
