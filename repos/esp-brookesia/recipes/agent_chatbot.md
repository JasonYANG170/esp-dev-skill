# AI 语音助手（AgentManager + 多 Agent + MCP）

> **适用摘要**: 基于 `brookesia_agent_manager` 构建完整 AI 语音助手——初始化 AgentManager、配置 XiaoZhi/Coze/OpenAI 多 Agent、通过 MCP 工具让 LLM 调用设备能力、用状态机（Activate→Start→Sleep→WakeUp→Stop）驱动生命周期、配合 Audio AFE 做语音唤醒与 Emote 做表情反馈。

## 触发意图

- "做一个 AI 语音助手"
- "用 Brookesia 跑 XiaoZhi / 小智"
- "切换 Agent（Coze / OpenAI）"
- "让 LLM 通过 MCP 控制设备"
- "Agent 状态机 / Sleep / WakeUp"
- "唤醒词 Hi ESP"

## 前置条件

| 条件 | 要求 |
|---|---|
| 硬件 | Flash >= 16MB、PSRAM >= 8MB；需 `AudioCodecPlayer/Recorder`、`DisplayPanel/Touch`、`StorageFs` |
| 目标 | 板级工程 `idf.py gen-bmgr-config -b <board>`（如 `esp_box_3`、`esp_vocat_board_v1_2`）|
| 依赖 | `brookesia_agent_manager` + 对应 agent 组件 + 服务（NVS/SNTP/Audio/WiFi/Device）+ Emote + `brookesia_mcp_utils` |
| 模型分区 | 唤醒词模型需烧入 `model` 分区 |
| 参考示例 | `examples/agent/chatbot` |

## 分步说明

### 1. 初始化 HAL（多核 SPI LCD 锁核）+ ServiceManager + TaskScheduler

```cpp
#include "brookesia/lib_utils.hpp"
#include "brookesia/service_manager.hpp"
#include "brookesia/hal_interface.hpp"
#include "brookesia/hal_adaptor.hpp"
using namespace esp_brookesia;

extern "C" void app_main(void)
{
#if CONFIG_SOC_CPU_CORES_NUM > 1
    {
        BROOKESIA_THREAD_CONFIG_GUARD({ .core_id = 1 });
        std::thread([&]() { hal::init_device(hal::DisplayDevice::DEVICE_NAME); }).join();
    }
#endif
    hal::init_all_devices();

    auto &service_manager = service::ServiceManager::get_instance();
    service_manager.start();

    // 后台 Task Scheduler
    auto backend_scheduler = std::make_shared<lib_utils::TaskScheduler>();
    backend_scheduler->start({
        .worker_configs = {
            { .name = "BackendWorker1", .core_id = 0, .priority = 1, .stack_size = 10 * 1024 },
            { .name = "BackendWorker2", .core_id = 1, .priority = 1, .stack_size = 10 * 1024 },
        }
    });
    backend_scheduler->post([&]() { setup_everything(backend_scheduler); });
}
```

### 2. 启动通用服务（NVS/SNTP/Device/Audio）

参考 `examples/agent/chatbot/main/modules/general_services.cpp`：

```cpp
GeneralServices::get_instance().init(backend_scheduler);
GeneralServices::get_instance().start_nvs();
GeneralServices::get_instance().start_sntp();
GeneralServices::get_instance().start_device();
GeneralServices::get_instance().init_audio();
```

### 3. 绑定 AgentManager 并配置 AFE 唤醒词

```cpp
using AgentHelper   = agent::helper::Manager;
using AudioHelper   = service::helper::Audio;

if (!AgentHelper::is_available()) { return; }

AudioHelper::AFE_Config afe{
    .vad = AudioHelper::AFE_VAD_Config{},
    .wakenet = AudioHelper::AFE_WakeNetConfig{
        .model_partition_label = "model",
        .mn_language = "cn",                  // chatbot 用 cn
        .start_timeout_ms = ...,
        .end_timeout_ms   = ...,
    },
};
AudioHelper::call_function_sync(
    AudioHelper::FunctionId::SetAFE_Config,
    BROOKESIA_DESCRIBE_TO_JSON(afe).as_object());

auto binding = service::ServiceManager::get_instance().bind(AgentHelper::get_name().data());
service_bindings_.push_back(std::move(binding));   // 长期保活
```

### 4. 设置各 Agent 信息（menuconfig 控制启用）

```cpp
using CozeHelper    = agent::helper::Coze;
using OpenaiHelper  = agent::helper::Openai;
using XiaoZhiHelper = agent::helper::XiaoZhi;

// Coze：Info 含 authorization(app_id/public_key/private_key) 与 robots[]
AgentHelper::call_function_sync(
    AgentHelper::FunctionId::SetAgentInfo,
    CozeHelper::get_name().data(),
    BROOKESIA_DESCRIBE_TO_JSON(coze_info).as_object());

// OpenAI：Info 含 model / api_key / voice
AgentHelper::call_function_sync(
    AgentHelper::FunctionId::SetAgentInfo,
    OpenaiHelper::get_name().data(),
    BROOKESIA_DESCRIBE_TO_JSON(openai_info).as_object());
```

> XiaoZhi 通常为默认 Agent，无需 SetAgentInfo；其激活码通过订阅 `ActivationCodeReceived` 事件获取并在 XiaoZhi 控制台输入。

### 5. 用 MCP 把设备能力暴露给 LLM（XiaoZhi）

```cpp
// 先查设备能力，按能力决定暴露哪些 Device 函数
auto cap = DeviceHelper::call_function_sync<boost::json::object>(DeviceHelper::FunctionId::GetCapabilities);
DeviceHelper::Capabilities device_caps;
BROOKESIA_DESCRIBE_FROM_JSON(cap.value(), device_caps);

std::vector<DeviceHelper::FunctionId> fns{
    DeviceHelper::FunctionId::GetCapabilities,
    DeviceHelper::FunctionId::GetBoardInfo,
    DeviceHelper::FunctionId::ResetData,
};
if (device_caps.count(std::string(hal::AudioCodecPlayerIface::NAME))) {
    fns.push_back(DeviceHelper::FunctionId::SetAudioPlayerVolume);
    fns.push_back(DeviceHelper::FunctionId::GetAudioPlayerVolume);
    fns.push_back(DeviceHelper::FunctionId::SetAudioPlayerMute);
    fns.push_back(DeviceHelper::FunctionId::GetAudioPlayerMute);
}
if (device_caps.count(std::string(hal::DisplayBacklightIface::NAME))) {
    fns.push_back(DeviceHelper::FunctionId::SetDisplayBacklightBrightness);
    fns.push_back(DeviceHelper::FunctionId::GetDisplayBacklightBrightness);
    fns.push_back(DeviceHelper::FunctionId::SetDisplayBacklightOnOff);
    fns.push_back(DeviceHelper::FunctionId::GetDisplayBacklightOnOff);
}
// StorageFs / PowerBattery 同理

XiaoZhiHelper::call_function_sync<boost::json::array>(
    XiaoZhiHelper::FunctionId::AddMCP_ToolsWithServiceFunction,
    std::string(DeviceHelper::get_name()),
    BROOKESIA_DESCRIBE_TO_JSON(fns).as_array());
```

之后用户即可对 XiaoZhi 说“把音量调大”“屏幕亮度设为 50%”“关屏”“查电量”等，由 LLM 调用对应 MCP 工具。

### 6. 订阅 Agent 状态/对话事件（驱动 UI 与 Emote）

AgentManager 事件（均 schema 简单：String / Boolean）：

- `GeneralActionTriggered`（Action:String）
- `GeneralEventHappened`（Event:String, IsUnexpected:Boolean）
- `SuspendStatusChanged`（IsSuspended:Boolean）
- `SpeakingStatusChanged` / `ListeningStatusChanged`（Bool）
- `AgentSpeakingTextGot` / `UserSpeakingTextGot`（Text:String）
- `EmoteGot`（Emote:String）

```cpp
auto conn = AgentHelper::subscribe_event(
    AgentHelper::EventId::GeneralEventHappened,
    [this](const std::string &, const std::string & event, bool is_unexpected) {
        AgentHelper::GeneralEvent e;
        BROOKESIA_DESCRIBE_STR_TO_ENUM(event, e);
        switch (e) {
        case AgentHelper::GeneralEvent::Stopped:
            if (is_wifi_connected())
                task_scheduler_->post_delayed([]() {
                    AgentHelper::call_function_sync(AgentHelper::FunctionId::TriggerGeneralAction,
                        BROOKESIA_DESCRIBE_TO_STR(AgentHelper::GeneralAction::Start));
                }, agent_restart_delay_s_ * 1000);
            break;
        default: break;
        }
    });
service_connections_.push_back(std::move(conn));
```

### 7. 启动 / 停止 Agent（在 Wi-Fi 连上后）

```cpp
bool start_agent() {
    AgentHelper::call_function_async(AgentHelper::FunctionId::TriggerGeneralAction,
        BROOKESIA_DESCRIBE_TO_STR(AgentHelper::GeneralAction::Activate), on_activate);
    AgentHelper::call_function_async(AgentHelper::FunctionId::TriggerGeneralAction,
        BROOKESIA_DESCRIBE_TO_STR(AgentHelper::GeneralAction::Start), on_start);
    return true;
}
void stop_agent() {
    AgentHelper::call_function_async(AgentHelper::FunctionId::TriggerGeneralAction,
        BROOKESIA_DESCRIBE_TO_STR(AgentHelper::GeneralAction::Stop));
}
```

> 默认半双工：Agent 说话时不听人声，但可被唤醒词（默认 `"Hi,ESP"`）打断进入聆听。一段时间无人声则自动 Sleep。

### 8. menuconfig 配置（`Example Configuration`）

- Enable Xiaozhi agent（默认启用）
- Enable Coze agent（需 App ID、公钥、私钥、Bot 配置）
- Enable OpenAI agent（需 API Key、model name、voice）

可同时启用多 Agent，运行期在设置界面切换 active agent。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| Agent 不响应 | Wi-Fi 未连 / 未启用 Agent / API Key 错 | 设置界面查 Wi-Fi；menuconfig 启用并填凭据；看串口日志 |
| 唤醒词无反应 | 麦克风异常 / 模型未烧入 | 检查音量；确认 `model` 分区已烧唤醒词模型 |
| 不进入配网 | Flash 残留旧凭据 | 设置界面“Factory Reset”后重启 |
| 状态机卡死 | 未按 Activate→Start | 先 Activate 再 Start；异常 Stopped 后延迟重启 |
| MCP 工具调用失败 | 设备无对应能力 | 按 capabilities 动态添加工具，而非硬编码 |
| SPI LCD 崩溃 | init/draw 跨核 | `BROOKESIA_THREAD_CONFIG_GUARD` 锁核 |

## 参考

- `examples/agent/chatbot/main/main.cpp` — 完整 app_main（HAL/ServiceManager/TaskScheduler）
- `examples/agent/chatbot/main/modules/ai_agents.cpp` — 多 Agent 初始化、MCP 工具、事件处理
- `examples/agent/chatbot/README.md` — 硬件需求、配网、语音交互、设置界面、故障排查
- `docs/en/agent/manager/index.rst` — Agent 状态机与接口
- `examples/service/console/docs/tutorial.md` — 用 `svc_call`/`svc_subscribe` 命令行驱动 Agent 的步骤
