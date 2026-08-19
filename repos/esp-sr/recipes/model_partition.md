# 模型选择、分区配置与烧录

> **适用摘要**: 通过 menuconfig 选择 ESP-SR 模型（NS / VAD / WakeNet / MultiNet），配置 `partitions.csv` 的 `model` 分区，生成并烧录 `srmodels.bin`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-sr/resources/`, source/examples in `repos/esp-sr/`, and this recipe path `repos/esp-sr/recipes/model_partition.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "怎么选模型"
- "模型烧到哪"
- "srmodels.bin 怎么生成"
- "model 分区怎么配"
- "改了模型怎么烧"

## 前置条件

| 条件 | 要求 |
|---|---|
| 组件 | `espressif/esp-sr` 已加入工程 |
| ESP-IDF | >= 5.0 |
| 参考文档 | `docs/en/flash_model/README.rst` |

## 分步说明

### 1. partitions.csv 增加 model 分区

ESP-SR 的 CMake 脚本只在分区表里存在名为 `model` 的分区时才会打包并烧录模型。

```csv
# Name,   Type, SubType, Offset, Size
nvs,      data, nvs,            , 24K
phy_init, data, phy,            , 4K
factory,  app,  factory,        , 1M
model,    data,                 , 6000K
```

- 分区 **Type** 用 `data`，**Name** 必须是 `model`。
- **Size** 按所选模型加总，参考 `docs/benchmark`；常见组合 4~6MB，大模型可达 8MB+。
- 没有这一行时，编译期会提示 `Failed to find model in partition table file`。

### 2. menuconfig 选择模型

```
idf.py menuconfig
--> ESP Speech Recognition
    --> model data path            # MODEL_IN_FLASH（默认）/ MODEL_IN_SDCARD
    --> Select noise suppression model     # SR_NSN_WEBRTC / SR_NSN_NSNET
    --> Select voice activity detection    # SR_VADN_WEBRTC / SR_VADN_VADNET(vadnet1 medium)
    --> Select WakeNet                    # wn9_hilexin / wn9_hiesp / wn9s_* 等
    --> Load Multiple Wake Words (WakeNet9/9s)  # 可同时加载多个唤醒词
    --> Chinese Speech Commands Model     # mn5q8_cn / mn6_cn / mn7_cn
    --> English Speech Commands Model     # mn5q8_en / mn6_en / mn7_en
    --> Add Chinese / English speech commands  # menuconfig 内逐条加命令词
```

> C 系列芯片（C3/C5/C6）只能选 `wn9s_*`；ESP32 选 `mn2_cn`（旧版）。

### 3. 编译并烧录（ESP-IDF，推荐）

```bash
idf.py set-target esp32s3
idf.py build
idf.py flash
```

`idf.py flash` 时 CMake 会自动执行：

```bash
python model/movemodel.py -d1 <sdkconfig> -d2 <esp-sr_path> -d3 <build_dir>
```

生成 `<build_dir>/srmodels/srmodels.bin` 并烧入 `model` 分区。

**只改了应用代码、不想重烧模型**（加速调试）：

```bash
idf.py app-flash
```

### 4. Arduino 或手动生成 / 烧录模型

Arduino 框架不会自动跑 CMake 打包，需手动：

```bash
python {esp-sr}/model/movemodel.py \
       -d1 {project}/sdkconfig \
       -d2 {esp-sr} \
       -d3 {project}/build
# 产物：{project}/build/srmodels/srmodels.bin
esptool.py --chip esp32s3 --port COMx --baud 921600 \
           write_flash <model_offset> {project}/build/srmodels/srmodels.bin
```

`<model_offset>` 从 `partitions.csv` / `idf.py partition-table` 查得。

### 5. 代码里加载并验证

```c
#include "model_path.h"
#include "esp_log.h"

srmodel_list_t *models = esp_srmodel_init("model");
if (models == NULL || models->num == 0) {
    ESP_LOGE("TAG", "No models loaded! check partition & menuconfig");
}

// 查看可用模型名
for (int i = 0; i < models->num; i++) {
    printf("model[%d]: %s\n", i, models->model_name[i]);
}

// 按前缀过滤
char *wn_name  = esp_srmodel_filter(models, ESP_WN_PREFIX, NULL);   // 如 "wn9_hilexin"
char *mn_name  = esp_srmodel_filter(models, ESP_MN_PREFIX, ESP_MN_CHINESE);
// 还能按关键字精确找，例如找带 "alexa" 的唤醒词
char *alexa    = esp_srmodel_filter(models, ESP_WN_PREFIX, "alexa");

// 判断模型是否存在
int idx = esp_srmodel_exists(models, "wn9_hilexin");  // >=0 存在，-1 不存在

// 取唤醒词文本
char *words = esp_srmodel_get_wake_words(models, "wn9_hilexin");
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 编译报 `Failed to find model in partition table` | partitions.csv 没有 `model` 行 | 加上 `model, data, , , <size>` |
| `esp_srmodel_init` 返回 num=0 | model 分区未烧录 | 用 `idf.py flash`（非 app-flash），或手动 esptool 烧 srmodels.bin |
| 改了模型���行为没变 | 用了 `app-flash` 没重烧模型 | 改模型后必须 `idf.py flash` |
| flash 空间不够 | model 分区太大占满 flash | 缩小 model 分区或换更大 flash；选更轻量模型（Q8 / wn9s） |
| menuconfig 看不到某模型 | 芯片不支持 | C 系列只显示 wn9s；ESP32 只显示 mn2_cn |
| SD 卡模型加载失败 | 模型路径配错 | menuconfig 选 `MODEL_IN_SDCARD` 并确认 SD 卡挂载与路径 |

## 参考

- `docs/en/flash_model/README.rst` — 模型选择与加载官方文档
- `model/movemodel.py` — 打包脚本（CMake 自动调用）
- `include/esp32s3/model_path.h`（`src/include/model_path.h`）— `esp_srmodel_init/filter/exists/get_wake_words`
- `README.md` — 各芯片支持的 WakeNet / MultiNet 模型名清单
- 配套 recipe：`recipes/afe_sr_pipeline.md`
