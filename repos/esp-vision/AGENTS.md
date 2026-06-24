# AGENTS.md — Supplementary Agent Guide

> 核心规则、配方索引、避坑要点、执行流程都在 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未覆盖的工程约定与工具链指引，不重复内容。

## Project Context

- **语言**：MicroPython（v1.28.0 固定基线）；绑定层用 C / C++（`-std=gnu++2b`）。
- **目标**：ESP32-P4 / ESP32-S3 / ESP32-S31，按开发板（`--board <BOARD>`）选择。
- **工具链/构建**：ESP-IDF `release/v5.5`、`release/v6.0` 或 `master`（先 `source export.sh`，`idf.py` 在 `PATH`）。仓库根目录的板级感知 `idf.py --board <BOARD> ...` 扩展（`idf_ext.py`）或顶层 `Makefile`。
- **运行形态**：ESP-VISION 把上游 MicroPython esp32 port + `overlay/micropython/` + ESP-VISION C 模块（`sensor`/`image`/`display`/`espdl` 及随芯片启用的 `h264`/`rtsp`）+ 自研平台层（`platform/`）+ OpenMV `imlib`（`components/imlib`）构建成一个 MicroPython 固件；用户在设备上写/烧录 MicroPython 脚本。**绝不在仓库根目录创建独立的 IDF app。**

## Code Generation Conventions

### 文件命名与放置
- 用户应用脚本：`.py`，常烧录到 `/main.py`（产品入口）或 `/boot.py`（启动前短时初始化）。
- ESP-VISION 内部源码分层：
  - `modules/` — `USER_C_MODULES` 绑定层（`py_*.c` / `.cpp`），命名 `py_<module>.c`（如 `py_sensor.c`、`py_espdl.cpp`）。
  - `platform/` — 自研 ESP32 胶水层（`preview.c`、`display.c`、`sdcard.c`、`usb_msc.c`、`jpeg.c`、`main.c` 等）。
  - `components/imlib/` — 纯 C 视觉算法（OpenMV MIT 派生），`upstream/`、`include/`、`compat/`。
  - `boards/<BOARD>/` — 板级配置（`boardconfig.h`、`imlib_config.h`、`manifest.py`、可选 `camera.c`/`display.c`/`sdcard.c`）与 MicroPython 移植侧（`boards/<BOARD>/port/`）。
- 类型存根：`stubs/*.pyi`（如 `sensor.pyi`、`image.pyi`）— 这些是 API 签名的权威来源。

### import 模式
```python
# 采集 + 图像处理
import sensor, image

# 显示
import sensor, display

# AI 推理
import sensor, espdl, image

# 编码 / 推流（仅 ESP32-P4）
import sensor, h264       # 录制
import sensor, h264, rtsp # 推流
```

### 标准采集脚本骨架
```python
import sensor

sensor.reset()
sensor.set_pixformat(sensor.RGB565)   # 或 sensor.GRAYSCALE
sensor.set_framesize(sensor.QVGA)     # 或 sensor.QVGA / sensor.QQVGA
sensor.skip_frames(time=1000)         # 等 AE/AWB 稳定

while True:
    img = sensor.snapshot()
    # ... 处理 / 推理 / 绘制 ...
    img.flush()   # 主机 USB CDC 预览
```

### 产品入口模式（`/main.py`）
```python
# /main.py — 建议把逻辑放在独立模块，启动策略与逻辑分离
import sys
import my_app

try:
    my_app.main()
except KeyboardInterrupt:
    raise                       # Ctrl-C 进入友好 REPL
except Exception as error:
    print("Fatal application error:")
    sys.print_exception(error)
```

### 资源释放模式
```python
det = espdl.ESPDet("/sdcard/model.espdl", score=0.5, nms=0.7)
try:
    while True:
        img = sensor.snapshot()
        for x, y, w, h, score, category in det.detect(img):
            img.draw_rectangle(x, y, w, h, color=(255, 0, 0))
        img.flush()
finally:
    det.deinit()
```

## Build Workflow

构建入口首选 `idf.py` 扩展（与 `Makefile` 等价），二者都会先运行 `prepare-micropython`：校验 `lib/micropython` 在固定提交 `e0e9fbb17ed6fd06bb76e266ae554784c9c80804`（v1.28.0），在 `build/micropython/` 导出干净副本，应用 `overlay/micropython/`，再把每个 `boards/<BOARD>/port/` 投射到该副本的 `ports/esp32/boards/<BOARD>/`。`lib/micropython` 始终保持干净。

```bash
# 构建（默认板 ESP32_P4X_EYE）
idf.py --board ESP32_P4X_EYE build
# 等价的 Makefile 路径
make BOARD=ESP32_P4X_EYE build

# 构建 + 烧录 + 监视
idf.py --board ESP32_P4X_EYE -p /dev/ttyACM0 build flash monitor

# 常用目标
idf.py --board <BOARD> menuconfig
idf.py --board <BOARD> size
idf.py --board <BOARD> -p <PORT> erase-flash
idf.py --board <BOARD> clean        # 清该板构建输出
idf.py --board <BOARD> fullclean     # 删该板完整构建目录
```

约束：
- `BOARD` 必须存在 `boards/<BOARD>/port/mpconfigboard.cmake`。受支持板：`ESP32_P4X_EYE`、`ESP32_P4X_FUNCTION_EV_BOARD`、`ESP32_S3_EYE`、`ESP32_S31_KORVO`；`TEMPLATE` 供新板 bring-up。
- `ESP32_S31_KORVO` 当前限定 ESP-VISION IDF `master` overlay（`board.cmake` 校验），构建前需 source IDF master 环境。
- 改了构建系统/板级配置/平台驱动/imlib 选项后，先验证 `ESP32_P4X_EYE` 构建。

## 脚本 Codegen Checklist

- [ ] `import` 只用 `sensor` / `image` / `display` / `espdl`，以及随芯片启用的 `h264` / `rtsp`（不在目标板上的模块会被 `import` 抛错）
- [ ] 采集顺序：`reset()` → `set_pixformat()` → `set_framesize()` → `skip_frames(time=...)` → `snapshot()`
- [ ] 像素格式只取 `sensor.GRAYSCALE` / `sensor.RGB565`；分辨率只取 `sensor.QQVGA` / `sensor.QVGA`
- [ ] 需要跨帧保留的 `snapshot()` 结果调了 `.copy()`
- [ ] ESP-DL 模型只在循环外构造一次，`finally` 中 `deinit()`
- [ ] `h264.H264Encoder` 尺寸与每帧输入一致；`finally` 中 `close()`
- [ ] `rtsp.RTSPServer` 在 `finally` 中 `stop()`
- [ ] 文件 / `image.ImageIO` 流写完后 `sync()` + `close()`
- [ ] 显示对象 `display.Display()` 只创建一次
- [ ] 主循环里有 `img.flush()` 用于主机预览（开发期）
- [ ] AprilTag 结果属性是字段（`tag.id`/`tag.rect`），不是方法

## Do Not Modify

- `lib/micropython`、`lib/ulab`、`lib/zxing-cpp` — 第三方子模块，固定版本；改动通过 `overlay/` 而非直接编辑子模块。
- `components/imlib/upstream/` — OpenMV MIT 源，尽量贴近上游，改动需记录；`OMV_NO_GPL=1` 不得移除。
- `modules/py_{image,helper,imageio,assert}.*` — OpenMV MIT 文件，原始头逐字保留，不参与格式化/版权 hook 重写。
- `boards/<BOARD>/port/` — MicroPython 移植侧文件，按 `overlay/` + `USER_C_MODULES` 方式扩展，不直接改生成副本。
- 本 Skill 的 `SKILL.md` frontmatter（Skill 元数据）。
