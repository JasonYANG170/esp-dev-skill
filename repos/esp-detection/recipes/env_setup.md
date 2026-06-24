# 搭建训练与量化环境

> **适用摘要**: 在 PC 上搭建 esp-detection 的 Python 训练/导出/量化环境（conda + requirements + esp-ppq），并验证自定义模块可被加载。

## 触发意图

- "搭建 esp-detection 环境"
- "安装 esp-ppq"
- "训练依赖 / requirements"
- "ESP-IDF 版本要求"

## 前置条件

| 条件 | 要求 |
|---|---|
| 操作系统 | Linux / macOS / Windows（Windows 上 torch≠2.4.0） |
| Python | 3.8（README 推荐） |
| GPU | 可选（CPU/MPS/单卡/多卡均支持） |

## 分步说明

### 1. 创建 conda 环境

```bash
conda create -n espdet python=3.8
conda activate espdet
```

### 2. 安装 Python 依赖

`requirements.txt` 固定了关键版本（来自仓库）：

```text
ultralytics>= 8.3.112
torch==2.2.0
torchvision==0.17.0
onnx==1.17.0
onnxsim==0.4.36
onnxruntime>=1.19.0
opencv-python==4.11.0.86
numpy==1.24.4
git+https://github.com/espressif/esp-ppq
```

```bash
pip install -r requirements.txt
```

> `esp-ppq` 是 Espressif 的量化工具（PTQ/QAT），由 git 直接安装。若网络受限，可先 clone 再 `pip install -e .`。

### 3. 准备 ESP-IDF（芯片端用）

训练/量化阶段**不需要** ESP-IDF；仅在把 `.espdl` 跑到芯片上时才需要。芯片端要求 **ESP-IDF release/v5.3 或以上**（模板的 `sdkconfig.defaults.*` 由 5.4.0 生成）。设置步骤参见 [ESP-IDF Programming Guide](https://idf.espressif.com/)。

### 4. 验证自定义模块可加载

```python
# verify_env.py —— 确认 custom_parse_model 与 ESPDetect 已注册
import ultralytics.nn.tasks as tasks
from nn.esp_tasks import custom_parse_model
from nn.modules import DSConv, DSBottleneck, DSC3k2, ESPBlock, ESPBlockLite, ESPSerial, ESPSerialLite, ESPDetect

tasks.parse_model = custom_parse_model   # 关键：注入 ESP 自定义模块
from ultralytics import YOLO
model = YOLO('cfg/models/espdet_pico.yaml')
print(model.model)   # 能成功构建说明环境 OK
```

预期：不报 `KeyError` / 未知模块名，模型包含 `ESPBlock` / `ESPDetect` 等层。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp-ppq` 安装失败 | 网络/Git 问题 | 先 `git clone https://github.com/espressif/esp-ppq` 再 `pip install -e esp-ppq` |
| Windows 上 torch 报错 | torch 2.4.0 在 Windows CPU 有 bug | 用 `requirements.txt` 锁定的 torch==2.2.0 |
| `KeyError: 'ESPBlock'` | 未注入 `custom_parse_model` | 训练/导出脚本入口加 `tasks.parse_model = custom_parse_model` |
| `onnxsim` 简化失败 | onnx/onnxsim 版本不匹配 | 严格按 requirements 的 `onnx==1.17.0`、`onnxsim==0.4.36` |
| 找不到 `cfg/models/espdet_pico.yaml` | 工作目录不在仓库根 | `cd` 到 esp-detection 仓库根再运行脚本 |

## 参考

- 仓库 `README.md`（Installation 章节）
- 仓库 `requirements.txt`
- 仓库 `nn/esp_tasks.py`、`nn/modules/__init__.py`
