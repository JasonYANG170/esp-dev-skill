# 单独运行 WakeNet（不经过 AFE）

> **适用摘要**: 不通过 AFE pipeline，直接调用 WakeNet 模型进行唤醒词检测。适用于自建前端处理、单元测试或低延迟独立场景。生产场景推荐用 AFE，本 recipe 对应 `test_apps/esp-sr/main/test_wakenet.cpp`。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-sr/resources/`, source/examples in `repos/esp-sr/`, and this recipe path `repos/esp-sr/recipes/wakenet_standalone.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "单独跑 WakeNet"
- "不走 AFE 测唤醒"
- "wakenet create/detect"
- "唤醒词阈值怎么设"

## 前置条件

| 条件 | 要求 |
|---|---|
| 模型 | menuconfig 已选 WakeNet（如 `wn9_hilexin`）并烧入 model 分区 |
| 音频 | 16kHz / 16-bit / 单声道 PCM |
| 参考代码 | `test_apps/esp-sr/main/test_wakenet.cpp` |

## 分步说明

### 1. 加载模型并取 handle

```c
#include "model_path.h"
#include "esp_wn_iface.h"
#include "esp_wn_models.h"

srmodel_list_t *models = esp_srmodel_init("model");
char *model_name = esp_srmodel_filter(models, ESP_WN_PREFIX, NULL);  // 如 "wn9_hilexin"
esp_wn_iface_t *wakenet = (esp_wn_iface_t *)esp_wn_handle_from_name(model_name);
```

### 2. 创建模型实例并设置阈值

```c
// det_mode: DET_MODE_90(Normal) / DET_MODE_95(Aggressive)
//           DET_MODE_2CH_90 / DET_MODE_2CH_95 / DET_MODE_3CH_90 / DET_MODE_3CH_95
model_iface_data_t *model_data = wakenet->create(model_name, DET_MODE_95);

// 阈值范围 0.4~0.9999，word_index 从 1 开始
wakenet->set_det_threshold(model_data, 0.8, 1);
```

### 3. 查询帧长与采样率，分配缓冲区

```c
int frequency   = wakenet->get_samp_rate(model_data);          // 16000
int chunksize   = wakenet->get_samp_chunksize(model_data);     // 单位是 int16 样本数
int16_t *buffer = malloc(chunksize * sizeof(int16_t));
```

### 4. 逐帧 detect

```c
// 每帧喂 chunksize 个 int16 样本
wakenet_state_t res = wakenet->detect(model_data, buffer);
if (res > 0) {
    // 返回值是唤醒词索引（从 1 开始），>0 表示唤醒
    printf("wake word #%d detected\n", res);
}
// 也可判断 wakenet_state_t 枚举：
//   WAKENET_NO_DETECT(0) / WAKENET_CHANNEL_VERIFIED(-1) / WAKENET_DETECTED(1)
```

> 注意：`detect` 旧签名返回 `wakenet_state_t`，仓库 test 里把返回值当 `int` 判断 `>0`。语义一致——返回唤醒词 index。

### 5. 查询辅助信息

```c
int   word_num   = wakenet->get_word_num(model_data);
char *word_name  = wakenet->get_word_name(model_data, 1);   // index 从 1 开始
int   start_pt   = wakenet->get_start_point(model_data);    // 唤醒词起点（样本数）
float thresh     = wakenet->get_det_threshold(model_data, 1);
float vol_gain   = wakenet->get_vol_gain(model_data, -30.0f);
int   ch         = wakenet->get_triggered_channel(model_data); // 触发通道，从 0 开始
```

### 6. 清理

```c
wakenet->reset_det_threshold(model_data);   // 恢复初始阈值
wakenet->clean(model_data);                 // 清空内部状态
wakenet->destroy(model_data);
esp_srmodel_deinit(models);
```

## 多唤醒词 / 运行时切换

V2.0 AFE 支持 `AFE_MAX_WAKEWORD_NUM`（3）个唤醒词，运行时增删：

```c
// 经 AFE 时：
afe_handle->add_wakenet_model(afe_data, "wn9_hiesp");   // 在 hilexin 之外再加 hiesp
afe_handle->disable_wakenet(afe_data);                  // 临时关
afe_handle->enable_wakenet(afe_data);                   // 临时开
afe_handle->set_wakenet_threshold(afe_data, 1, 0.85);   // index=1 或 2
```

单独 WakeNet 模式不直接支持运行时叠加多模型，多唤醒请走 AFE。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_wn_handle_from_name(NULL)` 崩溃 | menuconfig 没选 WakeNet | 选模型并 `idf.py flash`；判空 |
| 阈值设 0.1 不生效 | 低于下限 0.4 | WakeNet 阈值范围 0.4~0.9999 |
| 漏唤醒频繁 | 阈值过高 / 模式太严 | 降到 0.5~0.6，用 `DET_MODE_90` |
| 误唤醒频繁 | 阈值过低 / 模式太宽 | 升到 0.85~0.95，用 `DET_MODE_95` |
| C3/C5 跑不动 | 选了 wn9 | 换 `wn9s_*` |
| 返回 WAKENET_CHANNEL_VERIFIED(-1) | 双麦模式先确认通道 | 这是中间态，继续 detect 直到 >0 |

## 参考

- `test_apps/esp-sr/main/test_wakenet.cpp` — create/destroy/detect/cpu loading 完整用例
- `include/esp32s3/esp_wn_iface.h` — `esp_wn_iface_t` 全部回调、`wakenet_state_t`、`det_mode_t`
- `include/esp32s3/esp_wn_models.h` — `esp_wn_handle_from_name`、`ESP_WN_PREFIX`
- `docs/en/wake_word_engine/README.rst` — WakeNet 原理与阈值说明
- `README.md` — 全部支持的唤醒词模型名清单
- 配套 recipe：`recipes/afe_sr_pipeline.md`（生产用法）
