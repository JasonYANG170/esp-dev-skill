# 姿态估计（YOLO11nPose）

> **适用摘要**: 用 `espdl.YOLO11nPose` 做 COCO 17 关键点姿态估计，绘制检测框、骨架连线与关节点。缺失/低置信度关键点返回为 `(0, 0)`，绘图前需跳过。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/pose_estimation.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "姿态估计"
- "人体骨骼关键点"
- "keypoint detection"
- "YOLO11nPose"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/03-Machine-Learning/00-ESP-DL/yolo11n_pose.py` |
| 模型文件 | `/sdcard/espdet_yolo11n_pose_160_160_coco.espdl` |
| 像素格式 | `RGB565` |

## 分步说明

每个姿态结果含 17 个 COCO 关键点；无效关键点返回 `(0, 0)`。

```python
import espdl
import sensor
import time

MODEL = "/sdcard/espdet_yolo11n_pose_160_160_coco.espdl"

# COCO 关键点骨架连接（索引对）
SKELETON = (
    (5, 7), (7, 9), (6, 8), (8, 10),
    (5, 6), (5, 11), (6, 12), (11, 12),
    (11, 13), (13, 15), (12, 14), (14, 16),
)


def valid(point):
    return point[0] > 0 and point[1] > 0


sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

pose = espdl.YOLO11nPose(MODEL, score=0.35, nms=0.7, topk=5)

try:
    while True:
        img = sensor.snapshot()
        for x, y, w, h, score, category, keypoints in pose.detect(img):
            img.draw_rectangle(x, y, w, h, color=(255, 0, 0), thickness=2)
            img.draw_string(x, max(0, y - 12), "%.2f:%d" % (score, category), color=(255, 0, 0))

            # 骨架连线
            for a, b in SKELETON:
                if valid(keypoints[a]) and valid(keypoints[b]):
                    img.draw_line(
                        keypoints[a][0], keypoints[a][1],
                        keypoints[b][0], keypoints[b][1],
                        color=(0, 255, 0), thickness=2,
                    )

            # 关节点
            for point in keypoints:
                if valid(point):
                    img.draw_circle(point[0], point[1], 2, color=(0, 0, 255), thickness=1, fill=True)

        img.flush()
        time.sleep_ms(20)
finally:
    pose.deinit()
```

参数与结果（来源 `stubs/espdl.pyi`）：
- 构造：`YOLO11nPose(path, *, score=None, nms=None, topk=10, mean=None, std=None)`。
- `detect(img)` 返回 `list[(x, y, w, h, score, category, keypoints)]`，`keypoints` 为 17 个 `(x, y)`。
- `set_thresholds(score=None, nms=None)` 运行时调阈值。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 在 (0,0) 处画了点 | 未跳过无效关键点 | `valid(point)` 判断 `x>0 and y>0` 后再画 |
| 骨架连线错乱 | 连了无效点对 | 两端都 `valid` 才连线 |
| 帧率低 | topk 或分辨率过大 | 降 `topk`；用 `QQVGA`；模型小输入（160x160） |
| 关键点抖动 | 阈值偏低 | 提高 `score`；必要时在主机端做时序平滑（不在设备端） |

## 参考

- `example/03-Machine-Learning/00-ESP-DL/yolo11n_pose.py`
- `docs/zh_CN/api-reference/espdl.rst`
- `docs/zh_CN/concepts/ai-inference.rst`
- `stubs/espdl.pyi`
