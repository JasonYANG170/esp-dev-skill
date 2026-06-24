# Emote 表情模块：表情、动画与事件消息

> **适用摘要**: 使用 `brookesia_expression_emote`（Helper 类 `service::helper::ExpressionEmote`）为 AI 交互提供拟人化视觉反馈——加载表情/动画资源、设置 emoji、显示/隐藏事件消息文本、显示二维码、插入动画。Emote 通常与 Agent 状态事件联动。

## 触发意图

- "显示表情 emoji"
- "插一段动画"
- "显示事件文字 / 对话气泡"
- "显示配网二维码"
- "加载 emote 资源"

## 前置条件

| 条件 | 要求 |
|---|---|
| 依赖 | `espressif/brookesia_expression_emote` |
| 头文件 | `#include "brookesia/service_helper/expression/emote.hpp"` |
| HAL | 需要 `DisplayPanel`/`DisplayBacklight`（板级工程）|
| 参考示例 | `examples/agent/chatbot/main/modules/ai_agents.cpp`（`process_emote*`）、`examples/service/console/docs/tutorial.md` |

## 分步说明

### 1. 启动并绑定

```cpp
using EmoteHelper = service::helper::ExpressionEmote;
if (!EmoteHelper::is_available()) { /* Emote 组件未链接 */ }
auto binding = service_manager.bind(EmoteHelper::get_name().data());
```

### 2. 加载资源（PartitionLabel 等来源）

```bash
# 控制台示例命令（来自 tutorial.md）
svc_call Emote LoadAssetsSource {"Source":{"source":"anim_icon","type":"PartitionLabel","flag_enable_mmap":false}}
```

C++ 等价：调用对应 FunctionId（`LoadAssetsSource`，schema 参数为 Object），把资源分区/文件加载为表情与动画集。

### 3. 设置 emoji（按状态）

```cpp
EmoteHelper::call_function_async(EmoteHelper::FunctionId::SetEmoji, "winking");
EmoteHelper::call_function_async(EmoteHelper::FunctionId::SetEmoji, "sleepy");
EmoteHelper::call_function_async(EmoteHelper::FunctionId::SetEmoji, "neutral");
EmoteHelper::call_function_async(EmoteHelper::FunctionId::HideEmoji);
```

> emoji 名字必须存在于已加载资源集；chatbot 中“cool”不存在，故替换为“winking”。

### 4. 事件消息（System/Idle/Speak/Listen/User/Battery）

```cpp
// schema: Type(String), Text(String)
EmoteHelper::call_function_async(
    EmoteHelper::FunctionId::SetEventMessage,
    BROOKESIA_DESCRIBE_TO_STR(EmoteHelper::EventMessageType::System), "[Agent] Starting...");
EmoteHelper::call_function_async(
    EmoteHelper::FunctionId::SetEventMessage,
    BROOKESIA_DESCRIBE_TO_STR(EmoteHelper::EventMessageType::Idle));
EmoteHelper::call_function_async(
    EmoteHelper::FunctionId::SetEventMessage,
    BROOKESIA_DESCRIBE_TO_STR(EmoteHelper::EventMessageType::Speak), agent_speaking_text);
EmoteHelper::call_function_async(EmoteHelper::FunctionId::HideEventMessage);
```

### 5. 插入动画（带时长）

```cpp
// schema: Emote(String), Duration(Number)
EmoteHelper::call_function_async(
    EmoteHelper::FunctionId::InsertAnimation, emote_name, emote_animation_duration_ms_);
```

### 6. 二维码（用于 SoftAP 配网引导）

```cpp
std::string qr = "WIFI:T:nopass;S:" + softap_ssid + ";P:" + softap_password + ";;";
EmoteHelper::call_function_async(EmoteHelper::FunctionId::SetQrcode, qr);
```

### 7. 与 Agent 事件联动（chatbot 模式）

订阅 `AgentHelper::EventId::GeneralActionTriggered` / `GeneralEventHappened` / `SpeakingStatusChanged` / `ListeningStatusChanged` / `EmoteGot`，在各状态下调用上述 SetEmoji/SetEventMessage。例如：

- `GeneralAction::Start` → `SetEventMessage(System, "[Agent] Starting...")`
- `GeneralEvent::Started` → `SetEmoji("neutral")`
- `SpeakingStatusChanged=true` → `SetEventMessage(Speak)`
- `EmoteGot` → `InsertAnimation(emote, duration)`

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| emoji 不显示 | 名字不在资源集 | 先 `LoadAssetsSource` 加载；名字以资源 json 为准 |
| 屏幕黑 | 背光未开 | 先用 Device 服务 `SetDisplayBacklightOnOff(true)` |
| 事件文字不消失 | 未显式 Hide | 状态切换时调 `HideEventMessage` |
| 二维码扫不出 | 字符串格式错 | 用标准 `WIFI:T:...;S:...;P:...;;` 格式 |
| 动画卡顿 | 与 LVGL 任务核冲突 | 在 chatbot 中把 `emote_task_core` 与 `lvgl_task_core` 设同核 |

## 参考

- `examples/agent/chatbot/main/modules/ai_agents.cpp` — `process_emote_*` 系列完整用法
- `docs/en/expression/emote.rst` — Emote 服务接口契约
- `examples/service/console/docs/tutorial.md` — Step 2 用 `svc_call Emote ...` 加载资源
