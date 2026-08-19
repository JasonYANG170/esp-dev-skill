# 用 asyncio 组织多协程视觉应用

> **适用摘要**: 用 MicroPython `asyncio` 把视觉流水线（采集/推理/预览）与遥测、健康监控拆成协作调度的协程，通过复制后的标量状态与锁避免并发访问摄像头/模型/帧缓冲。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/asyncio_pipeline.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "多任务视觉应用"
- "采集 + 推理 + 上报"
- "asyncio 怎么用"
- "看门狗 / 流水线停滞检测"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考文档 | `docs/zh_CN/api-reference/micropython.rst`（组织复杂应用段） |
| 模型 | `/sdcard/espdet_pico_224_224_face.espdl`（示例） |
| 基线 | MicroPython v1.28.0（`asyncio` 软件包已冻结进固件） |

## 分步说明

ESP-VISION 的视觉调用（`sensor.snapshot()`、推理、`img.flush()`）都是同步操作；执行时其它协程不能运行。显式 `await asyncio.sleep_ms(0)` 提供帧间调度点。结构上由一个协程独占视觉流水线，遥测/健康协程只读复制后的标量结果。

```python
import asyncio
import espdl
import json
import sensor
import time

MODEL = "/sdcard/espdet_pico_224_224_face.espdl"

state = {
    "frames": 0,
    "detections": 0,
    "last_frame_ms": time.ticks_ms(),
}
state_lock = asyncio.Lock()


async def vision_task(detector):
    while True:
        img = sensor.snapshot()
        results = detector.detect(img)
        for x, y, w, h, score, category in results:
            img.draw_rectangle(x, y, w, h, color=(255, 0, 0))
        img.flush()

        async with state_lock:
            state["frames"] += 1
            state["detections"] = len(results)
            state["last_frame_ms"] = time.ticks_ms()

        # 每帧结束后让已就绪的控制/网络任务运行
        await asyncio.sleep_ms(0)


async def telemetry_task():
    while True:
        await asyncio.sleep_ms(1000)
        async with state_lock:
            payload = json.dumps({
                "frames": state["frames"],
                "detections": state["detections"],
            })
        print("telemetry:", payload)
        # 实际应用应替换为非阻塞 socket 传输


async def health_task():
    while True:
        await asyncio.sleep_ms(500)
        async with state_lock:
            age = time.ticks_diff(time.ticks_ms(), state["last_frame_ms"])
        if age > 3000:
            print("warning: vision pipeline stalled")


async def main():
    sensor.reset()
    sensor.set_pixformat(sensor.RGB565)
    sensor.set_framesize(sensor.QVGA)
    sensor.skip_frames(time=1000)
    detector = espdl.ESPDet(MODEL, score=0.5, nms=0.7)

    vision = asyncio.create_task(vision_task(detector))
    telemetry = asyncio.create_task(telemetry_task())
    health = asyncio.create_task(health_task())
    try:
        await asyncio.gather(vision, telemetry, health)
    finally:
        vision.cancel()
        telemetry.cancel()
        health.cancel()
        detector.deinit()


asyncio.run(main())
```

要点（来源 `docs/zh_CN/api-reference/micropython.rst`）：
- `asyncio` task 是协作调度协程，不是 FreeRTOS task；仅在 `await` 时切换。
- 视觉对象（摄像头、模型、显示、帧缓冲）应作为共享资源；除非 API 明确说明，始终由同一个协程/线程持有。
- 计算两个 tick 差用 `time.ticks_diff()` 以正确处理回绕。
- 仅当某阻塞操作无法集成进 `asyncio` 时才用 `_thread`，并用 `_thread.allocate_lock()` 保护共享 Python 数据；IRQ/其它线程可用 `asyncio.ThreadSafeFlag` 通知事件循环。

### 线程边界提示
- `_thread.start_new_thread()` 创建另一个 FreeRTOS task，但不提供优先级或核亲和性参数；MicroPython 线程固定在 `MP_TASK_COREID` 并通过 GIL 共享解释器。
- 需要确定时延/明确 FreeRTOS 优先级/ISR 到任务通知时，应把工作实现为 ESP-IDF C/C++ 组件，向 Python 暴露清晰边界。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 协程饿死、网络/控制无响应 | 视觉协程从不 `await` | 每帧后 `await asyncio.sleep_ms(0)` |
| 状态竞争 | 多协程直接读写共享 dict | 用 `asyncio.Lock()` 保护，或只传复制后的标量 |
| tick 差为负/错乱 | 直接相减未处理回绕 | 用 `time.ticks_diff(time.ticks_ms(), t0)` |
| 模型在协程里被反复重建 | 每次任务循环 new | 在 `main()` 里构造一次，`finally` 中 `deinit()` |
| 阻塞 socket 卡住事件循环 | 用了阻塞 socket | 改用 `asyncio` stream 或非阻塞 socket |

## 参考

- `docs/zh_CN/api-reference/micropython.rst`（组织复杂应用、线程与原生任务段）
- `docs/zh_CN/concepts/ai-inference.rst`
- `example/03-Machine-Learning/00-ESP-DL/espdet_pico.py`
- https://docs.micropython.org/en/v1.28.0/library/asyncio.html
