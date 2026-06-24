# ESP-VISION 示例脚本索引

> 来源：仓库 `example/` 目录（真实路径）。每条一行说明。括号内标注芯片/能力限制。

## 00-HelloWorld
| 路径 | 说明 |
|---|---|
| `example/00-HelloWorld/helloworld.py` | 首个脚本：复位传感器、RGB565/QVGA、连续采集并 `img.flush()` 预览 |

## 01-Camera
| 路径 | 说明 |
|---|---|
| `example/01-Camera/00-Snapshot/save_snapshot.py` | 采集一帧并以 jpg/bmp/ppm 保存到 `/sdcard`，`os.stat` 校验大小 |
| `example/01-Camera/01-H264/record_h264.py` | （P4）用 `h264.H264Encoder` 录制 300 帧裸码流到 `/sdcard/out.h264` |
| `example/01-Camera/02-RTSP/stream_rtsp.py` | （P4）以太网（IP101 RMII）+ H.264 + RTSP 推流到 `rtsp://<ip>:8554/` |
| `example/01-Camera/03-MJPEG/wifi_mjpeg_stream.py` | Wi-Fi HTTP MJPEG 流（无 H.264 的板可用）；含 index 页与 `multipart/x-mixed-replace` 流 |

## 02-Image-Processing
| 路径 | 说明 |
|---|---|
| `example/02-Image-Processing/00-Drawing/drawing.py` | 演示 `draw_line/rectangle/circle/cross/arrow/string` 等绘图 |
| `example/02-Image-Processing/00-Drawing/line_drawing.py` | 直线绘制 |
| `example/02-Image-Processing/00-Drawing/shape_drawing.py` | 形状绘制 |
| `example/02-Image-Processing/01-Image-Filters/binary_ops.py` | 二值化阈值操作 |
| `example/02-Image-Processing/01-Image-Filters/color_binary_filter.py` | 彩色 LAB 阈值二值化滤波 |
| `example/02-Image-Processing/01-Image-Filters/difference.py` | 帧差分（`difference`） |
| `example/02-Image-Processing/01-Image-Filters/filters.py` | mean/median/mode/midpoint/gaussian/laplacian/bilateral/histeq 滤波循环演示 |
| `example/02-Image-Processing/01-Image-Filters/morphology.py` | 形态学（腐蚀/膨胀/开闭） |
| `example/02-Image-Processing/02-Color-Tracking/find_blobs.py` | LAB 阈值 `find_blobs` 色块追踪（`merge=True`） |
| `example/02-Image-Processing/02-Color-Tracking/multi_color_blob_tracking.py` | 多色块追踪 |
| `example/02-Image-Processing/02-Color-Tracking/statistics.py` | 用 `get_statistics` 整定阈值 |
| `example/02-Image-Processing/03-Frame-Differencing/in_memory_frame_differencing.py` | 内存 `image.ImageIO` 流 + 帧差分 |

## 03-Machine-Learning（ESP-DL）
| 路径 | 说明 |
|---|---|
| `example/03-Machine-Learning/00-ESP-DL/espdet_pico.py` | `espdl.ESPDet` 人脸/小目标检测（`/sdcard/espdet_pico_224_224_face.espdl`） |
| `example/03-Machine-Learning/00-ESP-DL/espdet_pico_switch_by_button.py` | 按按键切换 ESPDet 模型/模式 |
| `example/03-Machine-Learning/00-ESP-DL/imagenet_cls.py` | `espdl.ImageNetCls` 文件图分类（`cat.jpg`） |
| `example/03-Machine-Learning/00-ESP-DL/yolo11.py` | `espdl.YOLO11` COCO 目标检测（`topk=10`） |
| `example/03-Machine-Learning/00-ESP-DL/yolo11n_pose.py` | `espdl.YOLO11nPose` 17 关键点姿态估计 + 骨架绘制 |

## 04-Barcodes
| 路径 | 说明 |
|---|---|
| `example/04-Barcodes/find_qrcodes.py` | 灰度 `find_qrcodes` 解码并打印 `payload()` |

## 05-Feature-Detection
| 路径 | 说明 |
|---|---|
| `example/05-Feature-Detection/find_apriltags.py` | `find_apriltags(families=image.TAG36H11)`，注意属性是字段（`tag.id`/`tag.rect`） |
| `example/05-Feature-Detection/find_barcodes.py` | （P4 ZXing）`find_barcodes`，打印 `payload`/`type`/`rect` |
| `example/05-Feature-Detection/find_lines_and_circles.py` | `find_lines` / `find_circles` 霍夫检测 |

## 06-Peripherals
| 路径 | 说明 |
|---|---|
| `example/06-Peripherals/00-Storage/sdcard.py` | 列 `/` 与 `/sdcard`，写读示例文件 |
| `example/06-Peripherals/01-Display/lcd_preview.py` | `display.Display().write(img)` 板载 LCD 预览 |
| `example/06-Peripherals/02-Servo/servo.py` | 舵机控制（`machine.PWM`） |

## 07-Network
| 路径 | 说明 |
|---|---|
| `example/07-Network/00-WebREPL/webrepl.py` | Wi-Fi + WebREPL 远程 REPL |
| `example/07-Network/01-Cloud-AI/openai_compatible_vision.py` | OpenAI 兼容云视觉 API 调用 |

> 使用提示：脚本起点选择见 `SKILL.md` 步骤 5。芯片限制（H.264/RTSP 仅 P4、条码仅 P4）以 `micropython.cmake` 与板级 `board.cmake` 为准。
