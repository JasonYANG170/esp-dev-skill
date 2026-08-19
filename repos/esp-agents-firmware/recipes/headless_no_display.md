# 无显示屏设备适配

> **适用摘要**: 在没有 LCD 的板子（如裸 ESP-VoCat 不带屏）上运行 voice_chat / matter_controller 固件，按 `docs/example_customisation.md` 注释掉 `app_display_init()` 与设备回调里的 `app_display_*` 调用，配网二维码改由产品包装提供。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "无屏设备"
- "headless"
- "没有显示屏 / no display"
- "app_display_init 失败"
- "无屏板怎么配网"

## 前置条件

| 条件 | 要求 |
|---|---|
| 示例 | `examples/voice_chat/` 或 `examples/matter_controller/`（两者 `app_main.c` 结构一致） |
| 文档 | `docs/example_customisation.md` "Devices with no display" 一节 |
| 配套 | 板子物理上无 LCD；音频（麦克风/喇叭）与 Wi-Fi 仍可用 |

> 仓库**没有**板级的 `DISPLAY_SUPPORTED` 开关；`app_display_init()` 内部会调 `init_display()`，无屏时会失败卡住，因此官方做法是**注释调用**而非依赖宏保护（见 `examples/common/app_common/src/app_display.c`）。

## 分步说明

### 1. app_main.c 中注释显示初始化

`voice_chat` 与 `matter_controller` 的 `app_main.c` 都在 `esp_board_manager_init()` 之后调 `app_display_init()`。注释掉这一行（`docs/example_customisation.md` 原文：comment out the call to `app_display_init()`）：

```c
void app_main(void)
{
    esp_event_loop_create_default();
    nvs_flash_init();
    agent_console_init();
    esp_board_manager_init();

    /* Initialize the display —— 无屏板注释掉 */
    // app_display_init();       // ❌ 无屏时 init_display() 会失败

    app_audio_init();            /* 音频照常（麦克风/喇叭不受影响） */
    /* ... app_device_init / app_agent_init / app_tools_register / app_agent_start ... */
#ifdef CONFIG_EXAMPLE_MATTER_CONTROLLER
    matter_controller_start_task();
#endif
}
```

> 注意：`#include <app_display.h>` 若已不再用可一并注释，但留着不影响编译（仅未使用的头文件）。

### 2. 设备回调里去掉 app_display 调用

两个示例都在 `app_main.c` 里注册了 `set_text_cb` 与 `system_state_changed_cb`，回调内部把文本/状态转发给显示。无屏板需**移除回调里的 `app_display_*` 调用**（`docs/example_customisation.md` 原文：comment out the `app_display_*` functions）：

```c
esp_err_t app_text_message_callback(app_device_text_type_t text_type, const char *text, void *priv_data)
{
    /* 串口仍打印对话文本（无屏时这是主要可见输出） */
    if (text_type != APP_DEVICE_TEXT_TYPE_SYSTEM) {
        char *role = text_type == APP_DEVICE_TEXT_TYPE_USER ? ">>" : "<<";
        printf("%s %s\n", role, text);
    }

    /* Sending to the display —— 无屏板注释掉 */
    // app_display_set_text(text_type, text, priv_data);
    return ESP_OK;
}

esp_err_t app_state_changed_callback(app_device_system_state_t new_state, void *priv_data)
{
    /* Sending to the display —— 无屏板注释掉 */
    // app_display_system_state_changed(new_state, priv_data);
    return ESP_OK;
}
```

> 回调结构体本身（`app_device_config_t`）保持不动，`set_text_cb` / `system_state_changed_cb` 仍指向上面两个函数 —— 它们现在只做串口打印 / 空操作，状态机与 Agent 事件链不受影响。

### 3. 内置工具里的显示调用怎么处理

`set_emotion` 工具的 handler（`app_common_tools` / `app_tools.c`）最终会调 `app_display_set_emotion()`。无屏时：

- `app_display_set_emotion` 因 `app_display_init` 未跑，`s_display_data` 未初始化，行为未定义。
- **建议**：保留工具注册（Agent 仍可被调 `set_emotion`，返回 ESP_OK 不致命），或在 `app_tools.c` 里去掉该工具的注册以免 Agent 调用。
- 若想彻底无显示依赖，可在 `set_emotion` handler 内部直接 `return ESP_OK;` 跳过 `app_display_set_emotion`。

> 仓库未对无屏单独提供 handler 分支，以上是源码级最小改动建议；改动后需在 Dashboard 的 `agent_config.json` 里相应移除 `set_emotion` 工具（若你选择不注册它），保持固件与配置双向一致（见 SKILL.md 核心原则 7）。

### 4. 配网二维码来源（无屏）

带屏设备在 `APP_NETWORK_EVENT_QR_DISPLAY` 事件里把二维码画到屏幕（`app_display.c` 的 `display_event_handler`）。无屏板没有屏幕显示二维码，按 `setup_guide.md` 原文：

> *"For devices with no display, the QR Code will be part of the product packaging"*

因此无屏板的配网路径：

1. **产品包装**上印制设备的 BLE Provisioning 二维码（量产时由工厂生成并印在包装/贴纸）。
2. 开发阶段：可用串口 `set-wifi <ssid> <passphrase>` 直接配网，再用 `set-agent` / `set-token` 设 Agent 凭据（见 `recipes/device_setup_provisioning.md` 步骤 3-4）。

### 5. 验证无屏板仍可工作

无屏板改动后，交互通道剩**语音 + 串口**：

- 唤醒词 "Hi, ESP" 仍触发 `DEVICE_EVENT_WAKEUP`（状态机不依赖屏幕）。
- 麦克风/喇叭经 `app_audio_init` 照常工作（音频链独立于显示）。
- 对话文本经 `app_text_message_callback` 打到串口（`>>` 用户 / `<<` 助手）。
- RainMaker 配网 / Agent 启动三前置条件（Wi-Fi + agent_id + refresh_token）不受影响。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 启动卡在 `Initializing display` / `Failed to initialize display` | `app_display_init()` 未注释 | 按步骤 1 注释 `app_display_init()` |
| `app_display_set_text` 崩溃或无操作 | 回调未注释，`s_display_data` 未初始化 | 按步骤 2 注释回调里的 `app_display_*` |
| Agent 调 `set_emotion` 后异常 | `app_display_set_emotion` 在未初始化的 display 上调用 | 步骤 3：跳过该调用或不注册 `set_emotion` 工具 |
| 无屏板配不了网 | 习惯性找屏幕二维码 | 用产品包装二维码，或串口 `set-wifi` + `set-token` + `set-agent` |
| 屏幕相关事件报错 | `APP_NETWORK_EVENT_QR_DISPLAY` 仍被处理 | 仅当 `app_display_init` 未跑时该 handler 未注册（init 内部注册），注释 init 即可；勿手动注册 |

## 参考

- `docs/example_customisation.md` — "Devices with no display" 一节（注释 `app_display_init()` 与 `app_display_*`）
- `examples/matter_controller/main/app_main.c` — `app_display_init` 调用位置 + 两个回调
- `examples/voice_chat/main/app_main.c` — 同结构（voice_chat 也有相同的 init + 回调）
- `examples/common/app_common/include/app_display.h` — `app_display_init` / `set_text` / `system_state_changed` / `set_emotion` 声明
- `examples/common/app_common/src/app_display.c` — `app_display_init` 实现（`init_display` 失败即返回错误；`APP_NETWORK_EVENT_QR_DISPLAY` 在此注册）
- `examples/matter_controller/setup_guide.md` — "QR Code will be part of the product packaging"（无屏配网二维码来源）
- `examples/voice_chat/README.md` — voice_chat 示例说明
