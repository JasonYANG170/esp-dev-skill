# 摄像头方向与状态诊断

> **适用摘要**: 用水平镜像 / 垂直翻转校正传感器安装方向，并用 `sensor.status()` 诊断分辨率、像素格式、sensor ID 与就绪状态。

## 触发意图

- "画面上下颠倒 / 左右反了"
- "翻转摄像头画面"
- "sensor.status 怎么用"
- "查摄像头输出尺寸"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考文档 | `docs/zh_CN/api-reference/sensor.rst` |
| 权威签名 | `stubs/sensor.pyi` |

## 分步说明

`set_hmirror(True)` 水平镜像，`set_vflip(True)` 垂直翻转；设置会影响后续采集的图像。`status()` 返回当前分辨率、像素格式、sensor ID、镜像/翻转与裁剪信息。

```python
import sensor

sensor.reset()
sensor.set_hmirror(True)    # 水平镜像
sensor.set_vflip(False)     # 不垂直翻转

info = sensor.status()
print("sensor id:", info["id"])
print("output:", info["width"], "x", info["height"])
print("pixformat:", info["pixformat"])
print("ready:", info["ready"])
print("hmirror:", info["hmirror"], "vflip:", info["vflip"])
```

`SensorStatus` 字段（来源 `stubs/sensor.pyi`）：`ready`、`id`、`width`、`height`、`pixformat`、`hmirror`、`vflip`、`raw_width`、`raw_height`、`active_width`、`active_height`。

### 暂时停止采集（省电或切任务）

不需要采集时停止摄像头流，下次采集前重启并丢弃少量帧让输入队列重新稳定。

```python
sensor.shutdown(True)       # 关闭摄像头流
# ... 执行不需要摄像头的任务 ...
sensor.shutdown(False)      # 重新启动
sensor.skip_frames(n=3)
img = sensor.snapshot()
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 镜像/翻转没生效 | 在 `snapshot()` 之后才设置 | 设置会影响后续帧，应在采集前调用 |
| `status()` 字段访问报 KeyError | 字段名拼错 | 按 `SensorStatus` 字段名访问 |
| 重启流后前几帧异常 | 队列未重新稳定 | `sensor.shutdown(False)` 后 `skip_frames(n=3)` |

## 参考

- `docs/zh_CN/api-reference/sensor.rst`
- `stubs/sensor.pyi`
- `docs/zh_CN/concepts/camera-pipeline.rst`
