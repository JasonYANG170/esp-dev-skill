# ESP-Brookesia 配置项（Kconfig）速查

> 全部符号来自仓库 `service/*/Kconfig`、`agent/*/Kconfig`、`utils/*/Kconfig`、`hal/*/Kconfig`、`expression/*/Kconfig`。通过 `idf.py menuconfig` 配置；亦可在板级 `sdkconfig.defaults.board` 中设默认值。此处仅列各组件通用的代表性符号，完整列表以各组件 `Kconfig` 为准。

## 通用机制：自动插件注册

多数服务/Agent 组件含以下开关（命名规律 `BROOKESIA_SERVICE_<NAME>_ENABLE_AUTO_REGISTER` / `BROOKESIA_AGENT_<NAME>_*`）：

| 符号（以 Wi-Fi 为例） | 类型 | 默认 | 说明 |
|---|---|---|---|
| `BROOKESIA_SERVICE_WIFI_ENABLE_AUTO_REGISTER` | bool | y | 启动时自动把服务注册为插件 |
| `BROOKESIA_SERVICE_WIFI_ENABLE_DEBUG_LOG` | menuconfig | n | 启用 debug 及以下日志（含子项 Service/Hal/StateMachine）|
| `BROOKESIA_SERVICE_WIFI_ENABLE_WORKER` | menuconfig | n | 启用服务 worker 线程 |
| `BROOKESIA_SERVICE_WIFI_WORKER_NAME` | string | `SvcWifiWorker` | worker 线程名 |
| `BROOKESIA_SERVICE_WIFI_WORKER_PRIORITY` | int | 5 | worker 优先级 |
| `BROOKESIA_SERVICE_WIFI_WORKER_STACK_SIZE` | int | 8192 | worker 栈大小 |
| `BROOKESIA_SERVICE_WIFI_WORKER_STACK_IN_EXT` | bool（depends on `SPIRAM`）| y | worker 栈分配到外部 PSRAM |
| `BROOKESIA_SERVICE_WIFI_WORKER_POLL_INTERVAL_MS` | int (1-10) | 10 | worker 轮询间隔 |

> 其它服务（`brookesia_service_nvs` / `sntp` / `audio` / `device` / `video` / `custom` / `manager`）与各 agent 组件遵循同样的命名规律（`BROOKESIA_SERVICE_<NAME>_*` / `BROOKESIA_AGENT_<NAME>_*`），见对应 `Kconfig`。

## 已确认存在的组件 Kconfig 文件

（来自 `find . -name Kconfig*`，路径相对仓库根）

```
utils/brookesia_lib_utils/Kconfig
utils/brookesia_mcp_utils/Kconfig
hal/brookesia_hal_interface/Kconfig
hal/brookesia_hal_adaptor/Kconfig
service/brookesia_service_manager/Kconfig
service/brookesia_service_wifi/Kconfig
service/brookesia_service_nvs/Kconfig
service/brookesia_service_sntp/Kconfig
service/brookesia_service_audio/Kconfig
service/brookesia_service_device/Kconfig
service/brookesia_service_video/Kconfig
service/brookesia_service_custom/Kconfig
agent/brookesia_agent_manager/Kconfig
agent/brookesia_agent_coze/Kconfig
agent/brookesia_agent_openai/Kconfig
agent/brookesia_agent_xiaozhi/Kconfig
expression/brookesia_expression_emote/Kconfig
```

## 示例级 Kconfig（menuconfig 中 `Example Configuration`）

| 示例 | 文件 | 典型选项 |
|---|---|---|
| `examples/agent/chatbot` | `main/Kconfig.projbuild` | `EXAMPLE_AGENTS_ENABLE_XIAOZHI`（默认 y）、`EXAMPLE_AGENTS_ENABLE_COZE`（需 App ID/公私钥/Bot）、`EXAMPLE_AGENTS_ENABLE_OPENAI`（需 API Key/model/voice）、Coze 的 `BOT1/BOT2` 启用与 name/id/voice_id/description、OpenAI 的 `MODEL/API_KEY/VOICE` |
| `examples/service/console` | `main/Kconfig.projbuild` | 控制台相关配置（参考组件内 `cmd_debug` / `cmd_service`） |

## 板级 Kconfig 默认

每个板目录含 `sdkconfig.defaults.board`，承载与硬件强相关的默认值（来自 `docs/en/hal/boards/index.rst`）：

- Flash 容量
- PSRAM 模式与频率
- CPU 时钟频率
- 录音格式参数（喂给 `brookesia_hal_adaptor`）

例如不同目标芯片在 Audio 示例中采用不同编解码通用参数：

```c
// 来自 examples/service/audio/main/main.cpp
#if CONFIG_IDF_TARGET_ESP32C5
    // 8kHz mono 16-bit, 60ms frame
#else
    // 16kHz mono 16-bit, 60ms frame
#endif
```

## 版本与目标相关常量

| 符号/常量 | 来源 | 用途 |
|---|---|---|
| `CONFIG_IDF_TARGET_ESP32C5` | ESP-IDF | 区分 C5 资源约束（Audio 示例跳过 G711A）|
| `CONFIG_SPIRAM_XIP_FROM_PSRAM` | ESP-IDF | 决定 Audio `player_task.stack_in_ext` |
| `CONFIG_SOC_CPU_CORES_NUM > 1` | ESP-IDF | 多核时对 SPI LCD 锁核（chatbot）|
| `CONFIG_ESP_HOSTED_ENABLED` | ESP-IDF | Wi-Fi 走 hosted 时把 `WIFI_WAIT_START_TIMEOUT_MS` 调到 5000 |

## 默认值速记

- Wi-Fi Start 等待：非 hosted 2000ms，hosted 5000ms（`examples/service/wifi`）
- 连接等待：10000ms；扫描 AP：20000ms；SoftAP 配网连接等待：120000ms
- Audio codec 默认（非 C5）：16kHz / mono / 16-bit / 60ms frame
- NVS demo 超时：100ms
- `BROOKESIA_SERVICE_MANAGER_DEFAULT_CALL_FUNCTION_TIMEOUT_MS`：同步调用省略 Timeout 时的默认值

## 如何设置

1. **menuconfig**：`idf.py menuconfig` → `Component config` → 对应组件，或 `Example Configuration`
2. **sdkconfig.defaults**：在工程根或板目录写默认值
3. **sdkconfig.defaults.board**：板级默认（与板硬件强相关项）
