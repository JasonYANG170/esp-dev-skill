# Custom 服务：动态注册函数与事件

> **适用摘要**: 使用 `brookesia_service_custom` 提供的 `CustomService`，在运行期动态注册自定义 function 与 event，把 LED/PWM/传感器等轻量逻辑封装为统一可本地/远程调用的服务能力，无需单独开发 Brookesia 组件。

## 触发意图

- "把我的 GPIO/LED 逻辑暴露成服务"
- "动态注册一个 Brookesia 服务函数"
- "自定义服务事件"
- "原���：快速暴露能力给 RPC"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/brookesia_service_custom` |
| 头文件 | `#include "brookesia/service_custom.hpp"` |
| 参考文档 | `docs/en/service/custom.rst` |

## 分步说明

### 1. 概念

`CustomService` 提供运行期 `register_function()` / `register_event()`。Function 处理器签名固定：`FunctionParameterMap` 入，`FunctionResult` 出；支持 lambda、`std::function`、自由函数、仿函数、`std::bind`。注册后即可通过 `ServiceManager` 本地调用或 TCP RPC 远程调用，事件走完整 pub/sub 生命周期。

### 2. 启动并绑定 CustomService

```cpp
#include "brookesia/service_custom.hpp"
auto &service_manager = service::ServiceManager::get_instance();
service_manager.start();
auto binding = service_manager.bind(/* CustomService 名 */);
```

> 具体服务名与注册宏见组件头 `brookesia/service_custom/service_custom.hpp`。CustomService 默认自动注册为插件（`BROOKESIA_SERVICE_*_ENABLE_AUTO_REGISTER` 类机制）。

### 3. 注册自定义函数（处理器形状固定）

```cpp
// 伪代码骨架（handler 形状来自 docs/en/service/custom.rst）
register_function("set_led", [](const FunctionParameterMap & params) -> FunctionResult {
    // 从 params 取输入（六类型之一），返回 FunctionResult
    bool on = /* ... */;
    return /* FunctionResult */;
});

register_function("read_sensor", &my_free_function);
register_function("toggle", std::bind(&MyClass::toggle, &obj, std::placeholders::_1));
```

### 4. 注册自定义事件

```cpp
register_event("sensor_alert");        // 发布/订阅生命周期
// 在某处发布：
// publish_event("sensor_alert", /* items... */);
```

### 5. 可选 Worker

如需线程安全执行，CustomService 支持挂载 Task Scheduler（与其它服务一致）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 注册失败 | 服务未 bind 或未 start | 先 `start()` → `bind()` 再注册 |
| 处理器编译不过 | 签名与固定形状不符 | 入参 `FunctionParameterMap`，返回 `FunctionResult` |
| RPC 调用拿不到 | schema 未声明 | 注册时同时声明 function/event schema |
| 重名冲突 | 与已有 function/event 同名 | 改用唯一名 |

## 参考

- `docs/en/service/custom.rst` — CustomService 概述、特性与 API 引用
- `service/brookesia_service_custom/` — 组件源码与 test_apps
- `docs/en/service/usage.rst` — 在框架内调用 function/订阅 event 的通用范式
