# 设备端 CHIP Shell 调试（CLI bring-up 与功能测试）

> **适用摘要**: 启用并使用设备端 CHIP Shell（`chip::LaunchShell()`），通过串口交互式检查设备配置（vendorid / productid / discriminator / pincode / fabricid）、验证配网凭据、跑功能测试。这是仓库文档化的主要交互式 bring-up 工具，shell/esp32 是专用示例。当无法用 chip-tool 配网时，shell 是定位"凭据对不对 / 栈启没起"的第一手段。

## 触发意图

- "CHIP shell"
- "device config / device get"
- "验证 discriminator / pincode"
- "chip::LaunchShell"
- "CONFIG_ENABLE_CHIP_SHELL"
- "串口命令行 bring-up"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `examples/shell/esp32/`（专用 shell 示例，`sdkconfig.defaults` 含 `CONFIG_ENABLE_CHIP_SHELL=y`） |
| 启动头文件 | `examples/platform/esp32/shell_extension/launch.h`（`void chip::LaunchShell()`） |
| 引擎 | `<lib/shell/Engine.h>`（`chip::Shell::Engine::Root().RunMainLoop()`） |
| 配置 | `CONFIG_ENABLE_CHIP_SHELL=y`（all-clusters-app / lock-app 中也可启用） |
| 串口 | 115200 baud（默认 console） |

## 分步说明

### 1. 在 app_main 中启动 shell

`chip::LaunchShell()` 创建一个名为 `"chip_cli"` 的 FreeRTOS 任务（栈 2048，优先级 5），运行 `Engine::Root().RunMainLoop()`。调用点须在 CHIP 栈初始化前后均可（all-clusters-app 放在 `deviceMgr.Init` 之前）：

```cpp
// examples/platform/esp32/shell_extension/launch.cpp
namespace chip {
void LaunchShell()
{
#if CONFIG_HEAP_TRACING_STANDALONE || CONFIG_HEAP_TASK_TRACKING
    RegisterHeapTraceCommands();
#endif
    xTaskCreate(&MatterShellTask, "chip_cli", 2048, NULL, 5, NULL);
}
} // namespace chip
```

在 `app_main` 中条件启动（all-clusters-app / lock-app 都用同一模式）：

```cpp
// examples/all-clusters-app/esp32/main/main.cpp
#if CONFIG_ENABLE_CHIP_SHELL
    chip::LaunchShell();
#endif

// 然后：CHIPDeviceManager::GetInstance().Init(&cb); InitServer();
```

> shell 示例（`examples/shell/esp32`）的 `sdkconfig.defaults` 已含 `CONFIG_ENABLE_CHIP_SHELL=y`，开箱即用。在 lock-app / all-clusters-app 中需自行 menuconfig 打开。

### 2. shell/esp32 专用示例

`examples/shell/esp32/` 是最小 shell 示例，不带应用业务逻辑，专门用于交互式测试 DeviceLayer。构建与其它 ESP32 示例一致：

```bash
cd examples/shell/esp32
idf.py set-target esp32     # 或 esp32c3
idf.py build
idf.py -p /dev/ttyUSB0 flash monitor
```

启动后在串口监视器（115200）看到 shell 提示符 `>`，键入 `help` 列出全部命令。

### 3. 检查设备配置（device config / get）

`device` 子模块直接调用 `chip::DeviceLayer` API，这是验证凭据的核心命令：

```bash
> device help
  start    Start the device layer. Usage: device start
  get      Get configuration value. Usage: device get <param_name>
  config   Dump entire configuration of device. Usage: device dump
Done
```

一次性 dump 全部配置（注意 `PinCode` / `Discriminator` 未配网时为 `<None>`）：

```bash
> device config
VendorId:        235a
ProductId:       feff
ProductRevision: 0001
SerialNumber:    <None>
ServiceId:       <None>
FabricId:        <None>
PinCode:         <None>
Discriminator:   <None>
DeviceId:        <None>
DeviceCert:      <None>
DeviceCaCerts:   <None>
MfrDeviceId:     <None>
MfrDeviceCert:   <None>
MfgDeviceCaCerts:<None>
```

查询单个字段（配网阶段最常用的排查手段）：

```bash
> device get vendorid
235a
Done
> device get pincode
> device get discriminator
```

合法 `<param_name>`（见 README_DEVICE.md）：`vendorid`、`productid`、`productrev`、`serial`、`deviceid`、`cert`、`cacerts`、`mfrdeviceid`、`mfrcert`、`mfrcacerts`、`pincode`、`discriminator`、`serviceid`、`fabricid`。

### 4. 启动 DeviceLayer（shell 示例中未自动启动时）

shell 示例默认不自动 `Init` CHIP 栈，需手动 `device start`：

```bash
> device start
Init CHIP Stack
Starting Platform Manager Event Loop
Done
```

启动后才能用 `device config` 读到运行时值。

### 5. 基础与诊断命令

| 命令 | 作用 |
|---|---|
| `help` | 列出全部顶层命令 |
| `echo <string>` | 回显（验证串口双向通） |
| `version` | 输出 CHIP 栈版本（如 `CHIP 0.0.g...-dirty`） |
| `rand` | 输出单字节随机数（验证 RNG） |
| `base64 encode <hex>` / `base64 decode <b64>` | hex ⇄ base64 |
| `ping` | 用 Echo Protocol 测网络路径丢包 |
| `exit` | 退出 shell（嵌入式上可能触发 watchdog 复位） |

空格用 `\` 转义，例如 `networkname Test\ Network`。

### 6. OpenThread 透传（otcli，启用 Thread 时）

当 OpenThread 支持启用时，`otcli` 把命令透传给 OpenThread CLI（参考 README_OTCLI.md）：

```bash
> otcli help            # 列出 OpenThread 子命令
> otcli state           # 查 Thread 状态
```

> Linux 上 otcli 需先启动 `otbr-agent`（见 README_OTCLI.md 的 Border Router 安装步骤）。ESP32 上启用 Thread 编译即可。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 烧录后无 `>` 提示符 | 未启用 shell 或串口波特率错 | `CONFIG_ENABLE_CHIP_SHELL=y`；监视器用 115200 |
| `device config` 全是 `<None>` | 栈未启动 / 未配网 | 先 `device start`；配网后 PinCode/Discriminator 才有值 |
| `device get <param>` 报错 | 参数名拼错 | 用合法名（见上表），区分大小写 |
| shell 与 monitor 抢同一 UART | console 与 shell 用了同一串口 | pigweed-app 把 console 移到 UART1，shell 默认用 UART0 |
| `otcli` 未找到命令 | 未启用 OpenThread | 在 sdkconfig 启用 Thread；嵌入式启用后自动可用 |
| 输入含空格被截断 | 空格是分隔符 | 用 `\` 转义：`networkname Test\ Network` |

## 参考项目

- `examples/shell/esp32/` — 专用 ESP32 shell 示例（`sdkconfig.defaults: CONFIG_ENABLE_CHIP_SHELL=y`）
- `examples/shell/README.md` — 顶层命令参考（help/echo/version/rand/base64/ping/exit）
- `examples/shell/README_DEVICE.md` — `device config/get/start` 命令与合法参数名
- `examples/shell/README_OTCLI.md` — `otcli` OpenThread 透传与 Border Router 安装
- `examples/platform/esp32/shell_extension/launch.h` / `launch.cpp` — `chip::LaunchShell()` 实现
- `examples/all-clusters-app/esp32/main/main.cpp` — 在 `app_main` 中条件启动 shell
