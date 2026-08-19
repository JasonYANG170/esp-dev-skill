# 任务生命周期控制与自定义任务

> **适用摘要**: 演示 `WhoTask` 的 `run`/`pause`/`resume`/`stop`（同步与异步）用法、事件位机制，以及如何继承 `WhoTask` 写自定义节点/任务挂到流水线或 App 上。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-who/resources/`, source/examples in `repos/esp-who/`, and this recipe path `repos/esp-who/recipes/task_lifecycle.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "暂停 / 恢复检测"
- "WhoTask 怎么停"
- "stop_async 区别"
- "自定义 WhoTask"
- "事件位 TASK_PAUSE"

## 前置条件

| 条件 | 要求 |
|---|---|
| 头文件 | `who_task.hpp`、`who_yield2idle.hpp` |
| 理解 | 事件组控制位（见 `state_machine.md` §1） |

## 分步说明

### 1. App 级一键控制

`WhoApp` 提供 `run()` / `pause()` / `resume()` / `stop()`，内部对 task group 同步操作，并联动 `WhoYield2Idle`。

```cpp
auto app = new who::app::WhoDetectAppLCD({{255,0,0}}, frame_cap);
app->set_model(get_detect_model());
app->run();      // 阻塞启动所有子任务（WhoYield2Idle 先起）

// 其它任务里（如网络/按键）可：
app->pause();    // 同步暂停：pause_async 所有任务 → wait_for_paused → cleanup
app->resume();
app->stop();     // 同步停止
```

### 2. 底层 `WhoTask` 同步 / 异步 API

| 方法 | 行为 |
|---|---|
| `run(stack, prio, core)` | 创建 FreeRTOS 任务（`xTaskCreatePinnedToCore`）；`core` 不能为 `tskNO_AFFINITY` |
| `stop()` | `stop_async()` + `wait_for_stopped(portMAX_DELAY)` + `cleanup_for_stopped()` |
| `stop_async()` | 仅置 `TASK_STOP`，立即返回 |
| `pause()` / `pause_async()` | 同理 |
| `resume()` | 置 `TASK_RESUME`、清 `TASK_PAUSED` |
| `is_active()` | 运行中（非停止/暂停/请求停止/请求暂停） |
| `wait_for_stopped(t)` / `wait_for_paused(t)` | 等事件位 |

```cpp
auto detect = new who::detect::WhoDetect("d", frame_cap->get_last_node());
detect->set_model(new HumanFaceDetect(...));
detect->run(4096, 2, 1);

// 异步暂停（不阻塞调用方）
detect->pause_async();
detect->wait_for_paused(pdMS_TO_TICKS(1000));

// 做些维护...

detect->resume();

// 干净退出
detect->stop();      // 同步
delete detect;       // 安全
```

> `WhoDetect` / `WhoFrameCapNode` 额外 override 了 `stop_async`/`pause_async`：会向 `m_in_queue` 发一个 `nullptr` 帧唤醒阻塞的 `xQueueReceive`，避免卡死。

### 3. 事件位机制（自定义触发）

子类用 `TASK_EVENT_BIT_LAST` 起始位定义自己的事件。`WhoDetect::NEW_FRAME`、`WhoRecognitionCore::RECOGNIZE/ENROLL/DELETE` 都是这样定义的。

触发：`xEventGroupSetBits(task->get_event_group(), EVENT_BIT)`（须先 `is_active()` 检查）。
消费：任务内 `xEventGroupWaitBits(m_event_group, EVENT|TASK_PAUSE|TASK_STOP, pdTRUE, pdFALSE, portMAX_DELAY)`。

### 4. 继承 `WhoTask` 写自定义任务

```cpp
#include "who_task.hpp"

class MyAnalyzer : public who::task::WhoTask {
public:
    static inline constexpr EventBits_t DO_ANALYZE = who::task::WhoTaskBase::TASK_EVENT_BIT_LAST << 3;
    MyAnalyzer() : who::task::WhoTask("MyAnalyzer") {}
    void trigger() {
        if (is_active()) {
            xEventGroupSetBits(get_event_group(), DO_ANALYZE);
        }
    }
private:
    void task() override {
        while (true) {
            EventBits_t bits = xEventGroupWaitBits(
                m_event_group, DO_ANALYZE | TASK_PAUSE | TASK_STOP, pdTRUE, pdFALSE, portMAX_DELAY);
            if (bits & TASK_STOP) break;
            if (bits & TASK_PAUSE) {
                xEventGroupSetBits(m_event_group, TASK_PAUSED);
                EventBits_t p = xEventGroupWaitBits(m_event_group, TASK_RESUME|TASK_STOP, pdTRUE, pdFALSE, portMAX_DELAY);
                if (p & TASK_STOP) break;
                continue;
            }
            if (bits & DO_ANALYZE) {
                // ... 你的处理
            }
        }
        xEventGroupSetBits(m_event_group, TASK_STOPPED);
        vTaskDelete(NULL);
    }
};

// 用法
MyAnalyzer *a = new MyAnalyzer();
a->run(4096, 2, 1);   // 必须在 App run() 之前注册（若挂进 task group）
a->trigger();
```

> 注意：`WhoTask` 构造会自动 `WhoYield2Idle::get_instance()->start_monitor(this)`；若想纳入某 App 的统一 pause/resume/stop，应在 App `run()` 之前 `app->add_task(a)`（但 `add_task` 是 protected，通常通过继承 App 暴露或直接由 App 持有）。

### 5. `WhoTaskState`（调试用）

`components/who_task/who_task_state.hpp` 提供周期打印所有任务状态的 `WhoTaskState`（间隔默认 2 秒），用于排查 cpu 占用。

```cpp
#include "who_task_state.hpp"
auto state = new who::task::WhoTaskState(2);
state->run(4096, 1, 0);   // 低优先级、核0
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `assert(xCoreID != tskNO_AFFINITY)` | 传了 `tskNO_AFFINITY` | 显式传 `0` 或 `1` |
| `assert(!WhoYield2Idle::get_instance()->is_active())` | App run() 后才注册任务 | 所有 `register_task`/`add_task` 在 `run()` 前 |
| 任务 pause 后卡住不退出 | 没唤醒阻塞点 | 节点类应 override `pause_async`/`stop_async` 发 nullptr 帧唤醒 |
| `delete` 任务对象后崩溃 | 任务还在跑 | 先 `stop()` 再 `delete` |
| 自定义事件位与系统位冲突 | 用了低 5 位 | 从 `TASK_EVENT_BIT_LAST`（`1<<5`）之后起 |

## 参考

- `components/who_task/who_task.hpp`、`who_task.cpp`
- `components/who_task/who_yield2idle.hpp`
- `components/who_task/who_task_state.hpp`
- `components/who_frame_cap/who_frame_cap_node.cpp`（`stop_async`/`pause_async` 唤醒示例）
- `resources/api_reference.md` §1 任务基类、§2 `WhoYield2Idle`
- `resources/state_machine.md` §1 状态机
