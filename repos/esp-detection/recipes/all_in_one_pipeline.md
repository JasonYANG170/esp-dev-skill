# 一站式流水线 espdet_run.py

> **适用摘要**: 用 `espdet_run.py` 一条命令串联 train → export → quantize → 生成芯片端 ESP-IDF 工程（自动 `git clone esp-dl`、复制模板、`rename_project` 替换占位符）。

> Evidence: `repos/esp-detection/resources/`, source/examples in `repos/esp-detection/`, and this recipe path `repos/esp-detection/recipes/all_in_one_pipeline.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "一条命令训练+部署"
- "端到端流水线"
- "自动生成芯片工程"
- "espdet_run.py 用法"

## 前置条件

| 条件 | 要求 |
|---|---|
| 环境 | 已按 `recipes/env_setup.md` 搭建（含 esp-ppq） |
| 数据 | `cfg/datasets/*.yaml` 与数据集就绪 |
| 校准集 | `--calib_data` 指向含图的目录 |
| 测试图 | `--img` 指向一张待推理图（会被嵌入固件 `main/`） |
| Git | 可访问 `https://github.com/espressif/esp-dl.git`（脚本会 clone） |

## 分步说明

### 1. 命令行参数（来自 `espdet_run.py::argparse`）

| 参数 | 必填 | 默认 | 含义 |
|---|---|---|---|
| `--class_name` | 是 | — | 检测目标类名（如 `mycat`），用于命名工程/模型/占位符替换 |
| `--pretrained_path` | 否 | None | 预训练 `.pt` 路径；None 则从 `cfg/models/espdet_pico.yaml` 新建 |
| `--dataset` | 是 | — | dataset yaml 路径 |
| `--size` | 否 | `[224,224]` | 输入分辨率 `[h w]`（nargs=2）；h≠w 自动 rect |
| `--target` | 否 | `esp32p4` | 目标芯片 `esp32p4` 或 `esp32s3` |
| `--calib_data` | 是 | — | 校准数据集目录 |
| `--espdl` | 是 | — | 输出 `.espdl` 路径 |
| `--img` | 是 | — | 芯片端测试图路径 |

### 2. 一条命令（方形 224，来自 `espdet_run.sh`）

```bash
python espdet_run.py \
  --class_name mycat \
  --pretrained_path None \
  --dataset "cfg/datasets/coco_cat.yaml" \
  --size 224 224 \
  --target "esp32p4" \
  --calib_data "deploy/cat_calib" \
  --espdl "espdet_pico_224_224_mycat.espdl" \
  --img "espdet.jpg"
```

非方形（自动 rect=True）：

```bash
python espdet_run.py \
  --class_name mycat --pretrained_path None \
  --dataset "cfg/datasets/coco_cat.yaml" \
  --size 160 288 \
  --target "esp32s3" \
  --calib_data "deploy/cat_calib" \
  --espdl "espdet_pico_160_288_mycat.espdl" \
  --img "espdet.jpg"
```

### 3. run() 内部阶段（来自源码）

```python
def run(class_name, pretrained_path, dataset, size, target, calib_data, espdl, img):
    assert isinstance(size, list) and len(size) == 2
    h, w = size
    if h != w:
        results = Train(pretrained_path, dataset, size, rect=True)   # 自动 rect
    else:
        results = Train(pretrained_path, dataset, size)
    model_path = os.path.join(str(results.save_dir), "weights/best.pt")

    Export(model_path, size)                                        # → best.onnx
    ONNX = model_path.replace(".pt", ".onnx")
    quant_espdet(onnx_path=ONNX, target=target, num_of_bits=8,
                 device='cpu', batchsz=32, imgsz=size,
                 calib_dir=calib_data, espdl_model_path=espdl)      # → .espdl

    # 生成芯片工程
    subprocess.run(["git", "clone", "https://github.com/espressif/esp-dl.git", "esp-dl"])
    examples_path = os.path.join("esp-dl", "examples")
    models_path   = os.path.join("esp-dl", "models")
    custom_example_path = os.path.join(examples_path, class_name + "_detect")
    custom_model_path   = os.path.join(models_path,   class_name + "_detect")
    os.makedirs(custom_example_path, exist_ok=True)
    os.makedirs(custom_model_path,   exist_ok=True)
    shutil.copytree("deploy/espdet_model_template",   custom_model_path,   dirs_exist_ok=True)
    shutil.copytree("deploy/espdet_example_template", custom_example_path, dirs_exist_ok=True)

    replacements = {
        "custom": class_name,
        "CUSTOM": class_name.upper(),
        "imgH": str(h),
        "imgW": str(w),
        "espdet.jpg": img,
        "espdet_jpg": os.path.splitext(img)[0] + "_jpg",
    }
    rename_project(Path(custom_example_path), replacements)
    rename_project(Path(custom_model_path),   replacements)

    espdl_model_path = (os.path.join(custom_model_path, "models/p4") if target == "esp32p4"
                        else os.path.join(custom_model_path, "models/s3"))
    shutil.copy(espdl, espdl_model_path)
    shutil.copy(img, os.path.join(custom_example_path, "main"))
```

完成后打印：
```
You can run models on chips right now!
Please run:
cd esp-dl/examples/<class>_detect
step1: idf.py set-target esp32p4/esp32s3
step2: idf.py flash monitor
```

### 4. rename_project 的占位符保护机制（来自源码）

`rename_project` 在替换前会**先保护所有 `add_custom_command` 整行**（用临时占位符 `__PLACEHOLDER_i__`），避免这些命令行里的路径被 `custom`/`imgH` 等替换破坏。仅处理 `.cpp/.hpp/.txt/.yml` 及名为 `Kconfig` 的文件。

### 5. 产物结构

执行后得到（相对当前目录）：

```
esp-dl/
├── examples/mycat_detect/      # 来自 espdet_example_template
│   └── main/espdet.jpg         # --img 拷入
└── models/mycat_detect/
    ├── models/p4/*.espdl       # target=esp32p4 时
    └── models/s3/*.espdl       # target=esp32s3 时
```

> 若已存在 `esp-dl/` 目录，`git clone` 会失败但 `copytree(dirs_exist_ok=True)` 仍会覆盖模板文件；建议删除旧 `esp-dl/` 或手动 clone 最新版。

### 6. 接下来

进入 `recipes/firmware_deploy.md` 完成编译烧录。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `git clone esp-dl` 失败 | 网络或目录已存在 | 手动 clone；或删除已存在 `esp-dl/` 重跑 |
| `--size` 只给一个值 | argparse `nargs=2` | 给两个值 `--size 224 224` |
| `--pretrained_path None` | 字符串 "None" | `run()` 内会判 `not in [None,'None']`，"None" 等价 None，安全 |
| 占位符替换破坏 CMake | 手改了 rename_project | 不要移除 `add_custom_command` 保护逻辑 |
| 模型拷到错误子目录 | target 写错 | target 与 `models/p4`/`models/s3` 严格对应 |

## 参考

- 仓库 `espdet_run.py`（`run`、`rename_project`、`argparse`）
- 仓库 `espdet_run.sh`（命令示例）
- 仓库 `deploy/espdet_*_template/`（被复制的模板）
- `docs/tutorials/how_to_train_and_deploy_model_with_rect_is_True.md`（Deployment 章节等价代码）
- `recipes/firmware_deploy.md`（编译烧录）
