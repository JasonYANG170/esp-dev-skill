# H.264 录制裸码流（仅 ESP32-P4）

> **适用摘要**: 用 `h264.H264Encoder` 把采集帧逐帧编码为 Annex-B H.264 NAL 单元并写入 SD 卡。`h264` 模块仅在 ESP32-P4 构建中可用。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/h264_record.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "录制视频"
- "H.264 编码"
- "h264.H264Encoder"
- "保存视频到 SD 卡"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/01-Camera/01-H264/record_h264.py` |
| 芯片 | ESP32-P4（`h264` 仅 P4 构建包含） |
| 权威签名 | `stubs/h264.pyi` |
| 存储 | SD 卡挂载到 `/sdcard` |

## 分步说明

```python
import sensor
import h264

FRAME_COUNT = 300
OUT_PATH = "/sdcard/out.h264"

sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

# 用首帧尺寸构造编码器，确保与每帧输入一致
img = sensor.snapshot()
enc = h264.H264Encoder(img.width(), img.height(), fps=15)

with open(OUT_PATH, "wb") as f:
    f.write(enc.encode(img))
    for i in range(FRAME_COUNT - 1):
        f.write(enc.encode(sensor.snapshot()))

enc.close()
print("wrote", FRAME_COUNT, "frames to", OUT_PATH)
```

主机播放 / 重封装：
```bash
ffplay out.h264
ffmpeg -framerate 15 -i out.h264 -c copy out.mp4
```

`H264Encoder` 构造与用法（来源 `stubs/h264.pyi`）：
- `H264Encoder(width, height, *, fps=15, gop=0, bitrate=0, qp_min=25, qp_max=45)`。`gop=0` 表示每秒一个关键帧（== `fps`）；`bitrate=0` 自动按 `width*height*fps` 选择。
- `encode(image)` 返回 `bytes`（Annex-B NAL 单元）；图像尺寸必须与构造尺寸一致。
- `keyframe()` 返回最近一帧是否为 IDR/I 帧。
- `close()` 释放编码器，之后使用会抛 `OSError`。

### 更精细的码率/质量控制

```python
enc = h264.H264Encoder(320, 240, fps=15, bitrate=1_500_000, gop=15, qp_min=25, qp_max=45)
try:
    with open("/sdcard/out.h264", "wb") as output:
        output.write(enc.encode(first))
        for _ in range(FRAME_COUNT - 1):
            output.write(enc.encode(sensor.snapshot()))
finally:
    enc.close()
```

`bitrate` 为目标每秒比特数；`qp_min`/`qp_max` 限定每帧量化参数范围（更小=更高质量、更大体积）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `import h264` 失败 | 非 P4 板（S3/S31 无此模块） | S3 改用 MJPEG（JPEG 流），见 `example/01-Camera/03-MJPEG` |
| 编码尺寸不匹配异常 | 分辨率改了但未重建编码器 | 分辨率变化时重建 `H264Encoder` |
| 录制帧率达不到 | 采集/存储跟不上 | 降分辨率/帧率/码率，而非反复创建编码器 |
| 输出文件无法播放 | 直接用播放器无法识别裸流 | 用 `ffplay out.h264`，或 ffmpeg 重封装为 mp4 |
| 异常时编码器泄漏 | 未 `close()` | 用 `try/finally` 包裹 |

## 参考

- `example/01-Camera/01-H264/record_h264.py`
- `docs/zh_CN/api-reference/h264.rst`
- `docs/zh_CN/concepts/codec-streaming.rst`
- `stubs/h264.pyi`
