# 采集并保存图像

> **适用摘要**: 采集一帧画面并以多种格式（jpg / bmp / ppm）保存到 SD 卡，用 `os.stat` 校验文件大小。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-vision/resources/`, source/examples in `repos/esp-vision/`, and this recipe path `repos/esp-vision/recipes/camera_snapshot_save.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "保存照片"
- "保存一帧到 SD 卡"
- "img.save 怎么用"
- "拍一张快照"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考示例 | `example/01-Camera/00-Snapshot/save_snapshot.py` |
| 存储 | SD 卡已挂载到 `/sdcard`（启动时自动挂载）；或片上 FAT 分区 `/flash` |

## 分步说明

`image.Image.save(path)` 按文件扩展名推断格式。保存前确认挂载点存在。

```python
import os
import sensor

OUT_DIR = "/sdcard"

sensor.reset()
sensor.set_pixformat(sensor.RGB565)
sensor.set_framesize(sensor.QVGA)
sensor.skip_frames(time=1000)

img = sensor.snapshot()

for name in ("test.jpg", "test.bmp", "test.ppm"):
    path = OUT_DIR + "/" + name
    img.save(path)
    print(path, os.stat(path)[6])   # 第 7 项为文件大小

print("done")
```

��明：
- 支持的扩展名取决于板级 `imlib_config.h`（本构建立启用 `IMLIB_ENABLE_IMAGE_FILE_IO` / `IMLIB_ENABLE_IMAGE_IO`）。
- 保存前可用 `os.listdir("/")` / `os.listdir("/sdcard")` 确认挂载点存在（部分板可选挂载 SD 卡）。

### 编码为 JPEG 并查看字节数

```python
import image

img = sensor.snapshot()
jpg = img.to_jpeg(quality=85, subsampling=image.JPEG_SUBSAMPLING_420)
jpg.save("/sdcard/snapshot.jpg")
print("encoded bytes:", jpg.size())
```

`quality`（1-100）权衡体积与画质；`JPEG_SUBSAMPLING_420` 通常得到最小的彩色图像（还有 `444` 画质最好、`422`、`AUTO`）。带硬件 JPEG 编码器的板会做硬件卸载，否则用软件 `esp_new_jpeg`。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `[Errno 2] ENOENT` 写文件失败 | `/sdcard` 未挂载或不存在 | 先 `os.listdir("/")` 检查；确认 SD 卡已插入且被识别 |
| 保存后文件为空 | 存储只读或写保护 | 检查 SD 卡写保护；换 FAT 分区 `/flash` |
| `img.to_jpeg` 体积太大 | quality 过高 / 色度抽样大 | 降 quality，或用 `JPEG_SUBSAMPLING_420` |

## 参考

- `example/01-Camera/00-Snapshot/save_snapshot.py`
- `example/06-Peripherals/00-Storage/sdcard.py`
- `docs/zh_CN/api-reference/image.rst`（编码并保存图像段）
