# 构建与烧录 ESP-WHO 示例工程

> **适用摘要**: 从零搭建 ESP-WHO 任一示例（human_face_recognition / object_detect / qrcode_recognition），配置 ESP-IDF 环境、选定 BSP 与目标芯片、编译烧录并查看串口。

> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "编译 esp-who"
- "烧录人脸识别示例"
- "ESP-WHO 怎么 build"
- "set-target 报错 BSP is not defined"
- "idf.py 用哪个 sdkconfig"

## 前置条件

| 条件 | 要求 |
|---|---|
| ESP-IDF 版本 | release/v5.4 或 release/v5.5 |
| 工具 | `tools/bsp_ext.py` 通过 `IDF_EXTRA_ACTIONS_PATH` 注入 |
| 硬件 | ESP32-S3-EYE / ESP32-S3-Korvo-2 / ESP32-P4 Function EV Board 之一 |
| 仓库 | 已克隆 `esp-who` 并知其本地绝对路径 |

## 分步说明

### 1. 设置 `IDF_EXTRA_ACTIONS_PATH`

`bsp_ext.py` 是 idf.py 扩展，必须先把它所在目录加入环境变量，否则 CMake 层报 `BSP is not defined`。

Linux:
```bash
export IDF_EXTRA_ACTIONS_PATH=/path_to_esp-who/tools/
echo $IDF_EXTRA_ACTIONS_PATH   # 必须返回正确路径
```

Windows / PowerShell:
```powershell
$Env:IDF_EXTRA_ACTIONS_PATH="/path_to_esp-who/tools/"
echo $Env:IDF_EXTRA_ACTIONS_PATH
```

Windows / cmd:
```cmd
set IDF_EXTRA_ACTIONS_PATH=/path_to_esp-who/tools/
echo %IDF_EXTRA_ACTIONS_PATH%
```

### 2. 进入示例目录

```bash
cd /path_to_esp-who/examples/human_face_recognition
# 或 examples/object_detect 、examples/qrcode_recognition
```

### 3. 设定目标芯片与 BSP 默认配置

`bsp_ext.py` 的 `BSP2IDF_TARGET` 强校验：S3 的 BSP 只能配 `esp32s3`，P4 的只能配 `esp32p4`。

```bash
# ESP32-S3-EYE
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye set-target esp32s3

# ESP32-S3-Korvo-2
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_korvo_2 set-target esp32s3

# ESP32-P4 Function EV Board
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_p4_function_ev_board set-target esp32p4
```

PowerShell 下需加引号：
```powershell
idf.py -DSDKCONFIG_DEFAULTS="sdkconfig.bsp.esp32_p4_function_ev_board" set-target "esp32p4"
```

### 4. （仅 `object_detect`）指定检测模型

`object_detect` 必须额外给 `-DDETECT_MODEL=`，可选：`human_face_detect` / `pedestrian_detect` / `cat_detect` / `dog_detect`。该变量决定依赖锁 `dependencies.lock.${BSP}.${DETECT_MODEL}` 与可选组件。

```bash
idf.py -DSDKCONFIG_DEFAULTS=sdkconfig.bsp.esp32_s3_eye_noglib -DDETECT_MODEL=human_face_detect set-target esp32s3
```

> `object_detect` 的 BSP 名统一带 `_noglib` 后缀（不链接图形库）。

### 5. （可选）menuconfig 微调

```bash
idf.py menuconfig
# 常改项：
#   Component config → esp-who: human_face_recognition → database file system
#   Component config → esp-who: yield2idle → max task loop time in seconds
```

### 6. 编译、烧录、监视

```bash
idf.py -p /dev/ttyUSB0 flash monitor
# Windows: idf.py -p COM5 flash monitor
# 不指定 -p 会扫描所有端口
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `BSP is not defined, please make sure that the environment variable IDF_EXTRA_ACTIONS_PATH is properly set.` | 没设 `IDF_EXTRA_ACTIONS_PATH` 或 echo 返回错误路径 | 按“分步说明 1”设置并 `echo` 校验 |
| `Invalid bsp: xxx, supported list: {...}` | BSP 名拼错或用了不存在的 sdkconfig.bsp | 列目录下 `sdkconfig.bsp.*`，用真实文件名 |
| `BSP esp32_p4_function_ev_board does not match idf_target esp32s3.` | BSP 与 `-set-target` 不一致 | 按 `BSP2IDF_TARGET` 表对应（见上） |
| `DETECT_MODEL is not selected`（仅 object_detect） | 缺 `-DDETECT_MODEL=` | 加上 `-DDETECT_MODEL=human_face_detect` 等 |
| 构建时拉取组件失败 / 依赖锁不匹配 | `dependencies.lock.<bsp>[.model]` 缺失或与当前组合不一致 | 用 `tools/gen_dependencies_lock.py` 重新生成，或确认 BSP/模型组合 |
| 烧录后串口无输出 | USB 串口驱动 / 端口选错；或未按复位 | Windows 装 CP210x/CH340 驱动；`idf.py -p PORT monitor` 后按板载 RST |

## 参考

- `examples/human_face_recognition/`、`examples/object_detect/`、`examples/qrcode_recognition/`
- `tools/bsp_ext.py`（`BSPS` / `BSP2IDF_TARGET` / `DETECT_MODELS`）
- 仓库根 `README.md` → Quick Start
