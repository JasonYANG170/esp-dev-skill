# Pigweed Echo RPC 远程调试通道（UART RPC console）

> **适用摘要**: 构建并使用 `pigweed-app/esp32` 的 Echo RPC 服务端，通过 UART 在主机用 `pw_hdlc.rpc_console` 远程调用设备上的 `EchoService.Echo(msg=...)`。这是仓库文档化的主机 ↔ 设备 RPC 通道，能力超出普通串口日志（可双向触发设备动作、结构化往返）。当需要比 `ESP_LOGI` 更强的交互式调试时使用。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/connectedhomeip/resources/`, source/examples in `repos/connectedhomeip/`, and this recipe path `repos/connectedhomeip/recipes/pigweed_rpc_debug.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "Pigweed RPC"
- "pw_hdlc rpc_console"
- "Echo RPC"
- "CONFIG_ENABLE_PW_RPC"
- "主机调用设备函数"
- "uart RPC 调试"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/pigweed-app/esp32/`（专用 RPC 示例，`sdkconfig.defaults` 含 `CONFIG_ENABLE_PW_RPC=y`） |
| RPC 框架 | `examples/common/pigweed/RpcService.h`（`chip::rpc::Start`）、`examples/common/pigweed/esp32/PigweedLoggerMutex.h` |
| 主机工具 | `python -m pw_hdlc.rpc_console`（来自 Pigweed） |
| 配置 | `CONFIG_ENABLE_PW_RPC=y`；`PW RPC Example Configuration` 选 UART 端口/引脚/波特 |
| 串口 | 默认 UART0 / 115200（RPC 走 UART0，console 可移到 UART1） |

## 分步说明

### 1. pigweed-app 的 app_main（RPC server 启动）

pigweed-app 的 `app_main` 极简：初始化日志互斥锁，创建 RPC 任务，注册 EchoService：

```cpp
// examples/pigweed-app/esp32/main/main.cpp
#include "pw_rpc/echo_service_nanopb.h"
#include "pw_rpc/server.h"
#include "PigweedLoggerMutex.h"
#include "RpcService.h"

pw::rpc::EchoService echo_service;   // Pigweed 自带 Echo 服务

void RegisterServices(pw::rpc::Server & server)
{
    server.RegisterService(echo_service);
}

void RunRpcService(void *)
{
    ::chip::rpc::Start(RegisterServices, &::chip::rpc::logger_mutex);
}

extern "C" void app_main()
{
    PigweedLogger::init();
    xTaskCreate(RunRpcService, "RPC",
                (4 * 1024) / sizeof(StackType_t),  // 4 KB 栈
                nullptr, 5,                         // 优先级 5
                &rpcTaskHandle);
    while (1) { vTaskDelay(50 / portTICK_PERIOD_MS); }
}
```

`chip::rpc::Start` 的签名（`examples/common/pigweed/RpcService.h`）：

```cpp
namespace chip { namespace rpc {
class Mutex {  // uart_mutex_ 实现日志与 RPC 复用串口的互斥
public:
    virtual void Lock() = 0;
    virtual void Unlock() = 0;
    virtual ~Mutex() {}
};
void Start(void (*RegisterServices)(pw::rpc::Server &), ::chip::rpc::Mutex * uart_mutex_);
}}
```

> 注意：pigweed-app 的 `app_main` 不调用 `CHIPDeviceManager::Init` / `InitServer`，它是一个纯 RPC 通道示例。要在 RPC 之上跑 Matter 栈，需自行补上标准初始化链。

### 2. 配置 RPC UART

`PW RPC Example Configuration`（`main/Kconfig.projbuild`）控制串口参数：

| Kconfig | 默认（ESP32） | 含义 |
|---|---|---|
| `EXAMPLE_UART_PORT_NUM` | 0 | RPC 用的 UART 端口 |
| `EXAMPLE_UART_BAUD_RATE` | 115200 | 波特率（范围 1200–115200） |
| `EXAMPLE_UART_RXD` | 3 | RX 引脚 |
| `EXAMPLE_UART_TXD` | 1 | TX 引脚 |

```bash
cd examples/pigweed-app/esp32
idf.py menuconfig
# -> PW RPC Example Configuration -> 改 UART 端口/引脚/波特
# -> Component config -> Common ESP-related -> UART for console output
#    （README 指出：RPC 占 UART0 时，console 输出需移到 UART1）
idf.py set-target esp32   # 或 esp32c3
idf.py build
idf.py -p /dev/ttyUSB0 flash
```

### 3. 主机侧启动 RPC console

在主机（需 CHIP 环境）用 `pw_hdlc.rpc_console` 连接设备 UART，加载 echo.proto：

```bash
python -m pw_hdlc.rpc_console \
    --device /dev/tty.SLAB_USBtoUART -b 115200 \
    $CHIP_ROOT/third_party/pigweed/repo/pw_rpc/pw_rpc_protos/echo.proto \
    -o /tmp/pw_rpc.out
```

- `--device`：设备串口（Linux 用 `/dev/ttyUSB0`）
- `-b 115200`：与设备 `EXAMPLE_UART_BAUD_RATE` 一致
- 第一个位置参数：echo.proto 定义（在 `third_party/pigweed/repo/pw_rpc/pw_rpc_protos/`）
- `-o`：输出日志文件

进入交互式 Python shell（`rpc_console` 提示符）后即可调用 RPC。

### 4. 调用 Echo RPC

EchoService 把收到的消息原样返回，验证整条 RPC 链路：

```python
# 在 rpc_console 提示符下
rpcs.pw.rpc.EchoService.Echo(msg="hi")
# 返回 msg="hi"
```

`msg=` 后引号内是任意字节串。成功返回表示：UART 物理层、HDLC 帧、pw_rpc 传输、设备侧 EchoService 全部正常。

### 5. 扩展自定义 RPC 服务

参考 EchoService 的注册方式，定义自己的 nanopb service 并在 `RegisterServices` 中注册：

```cpp
#include "pw_rpc/server.h"
// 你的生成 service 头
#include "my_service_nanopb.h"

MyService my_service;

void RegisterServices(pw::rpc::Server & server)
{
    server.RegisterService(echo_service);
    server.RegisterService(my_service);   // 追加自定义服务
}
```

服务定义（`.proto`）经 Pigweed 的 nanopb 代码生成器产出 `_nanopb.h`，随后即可在主机 `rpcs.<pkg>.<Service>.<Method>(...)` 调用。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `rpc_console` 连不上 | UART 端口/波特与设备不符 | 核对 `EXAMPLE_UART_*` 与 `--device`/`-b` 一致 |
| RPC 无响应 | 设备侧 `app_main` 未启动 RPC 任务 / `CONFIG_ENABLE_PW_RPC` 未开 | 确认 `xTaskCreate(RunRpcService...)` 与 `sdkconfig.defaults` |
| RPC 与 console 日志互相破坏 | 两者抢同一 UART | 按 README 把 console 移到 UART1，RPC 留 UART0 |
| `Echo` 调用超时 | HDLC 帧错或 `echo.proto` 路径错 | 确认 proto 路径为 `$CHIP_ROOT/third_party/pigweed/repo/pw_rpc/pw_rpc_protos/echo.proto` |
| 想在 RPC 示例里跑 Matter 栈 | pigweed-app 默认不初始化 CHIP 栈 | 自行补 `nvs_flash_init` → `CHIPDeviceManager::Init` → `InitServer` |
| `logger_mutex` 未定义 | 未包含 esp32 版 PigweedLoggerMutex | `#include "PigweedLoggerMutex.h"`，用 `examples/common/pigweed/esp32/` 版本 |

## 参考项目

- `examples/pigweed-app/esp32/` — 专用 ESP32 Pigweed RPC 示例（`sdkconfig.defaults: CONFIG_ENABLE_PW_RPC=y`）
- `examples/pigweed-app/esp32/README.md` — 构建 + `pw_hdlc.rpc_console` 用法 + `rpcs.pw.rpc.EchoService.Echo(msg=...)`
- `examples/pigweed-app/esp32/main/main.cpp` — `EchoService` 注册与 `chip::rpc::Start` 调用
- `examples/pigweed-app/esp32/main/Kconfig.projbuild` — `PW RPC Example Configuration`（UART 端口/引脚/波特）
- `examples/common/pigweed/RpcService.h` — `chip::rpc::Start` 与 `Mutex` 接口
- `examples/common/pigweed/esp32/PigweedLoggerMutex.h` — 日志/RPC 串口互斥
- `recipes/chip_shell_debug.md` — 另一条设备端调试通道（CHIP Shell）
