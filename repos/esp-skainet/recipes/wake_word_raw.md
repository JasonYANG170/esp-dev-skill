# 离线 PCM/WAV 唤醒词推理

> **适用摘要**: 不走 AFE，直接对内存中的 PCM/WAV 数据逐帧调用 WakeNet 检测。适用于离线批量评估唤醒模型、回放录音测试、单元测试。

> Version: selected repo/component version; align with the user project and dependency manifest.
> Evidence: `repos/esp-skainet/resources/`, source/examples in `repos/esp-skainet/`, and this recipe path `repos/esp-skainet/recipes/wake_word_raw.md`.
> Validation: draft metadata added from repository routing; verify APIs, Kconfig symbols, and component dependencies against the selected version.

## 触发意图

- "用 wav 文件测试唤醒词"
- "离线唤醒推理"
- "WakeNet 直接 detect"
- "不走 AFE 跑 wakenet"

## 前置条件

| 条件 | 要求 |
|---|---|
| 参考工程 | `examples/wake_word_detection/wakenet/` |
| 依赖组件 | `espressif/esp-sr` (^2.0.0) |
| Kconfig | 勾选一个 WakeNet（如 `wn9_hilexin` / `wn9_hiesp`） |
| 测试音频 | 16 kHz、16-bit、单声道 PCM（示例用编译期数组 `hilexin`/`hiesp`） |

## 分步说明

### 1. 引入头文件与内置测试音频

```c
#include "esp_wn_iface.h"
#include "esp_wn_models.h"
#include "model_path.h"
#include "string.h"
#include "hiesp.h"      // 编译期内嵌的 Hi,ESP 测试 PCM
#include "hilexin.h"    // 编译期内嵌的 Hi,Lexin 测试 PCM
```

### 2. app_main：加载模型并按名取句柄

```c
void app_main(void *arg)
{
    srmodel_list_t *models = esp_srmodel_init("model");
    char *model_name = esp_srmodel_filter(models, ESP_WN_PREFIX, "hilexin");
    esp_wn_iface_t *wakenet = (esp_wn_iface_t *)esp_wn_handle_from_name(model_name);

    // 创建模型实例，DET_MODE_95 = 激进模式（更易触发）
    model_iface_data_t *model_data = wakenet->create(model_name, DET_MODE_95);
```

> `ESP_WN_PREFIX` 为 `"wn"`（定义于 `esp_wn_models.h`）。`esp_srmodel_filter` 在模型列表里找含 `"wn"` 与 `"hilexin"` 的名字。

### 3. 计算帧大小并选测试数据

```c
    // 注意：get_samp_chunksize 返回的是单通道 16-bit 样本数，转字节要 ×sizeof(int16_t)
    int audio_chunksize = wakenet->get_samp_chunksize(model_data) * sizeof(int16_t);
    int16_t *buffer = (int16_t *)malloc(audio_chunksize);

    unsigned char *data = NULL;
    size_t data_size = 0;
    if (strstr(model_name, "hiesp") != NULL) {
        data = (unsigned char *)hiesp;
        data_size = sizeof(hiesp);
    } else if (strstr(model_name, "hilexin") != NULL) {
        data = (unsigned char *)hilexin;
        data_size = sizeof(hilexin);
    }
```

### 4. 逐帧 detect

```c
    int chunks = 0;
    while (1) {
        if ((chunks + 1) * audio_chunksize <= data_size) {
            memcpy(buffer, data + chunks * audio_chunksize, audio_chunksize);
        } else {
            break;   // 数据耗尽
        }

        wakenet_state_t state = wakenet->detect(model_data, buffer);
        if (state == WAKENET_DETECTED) {
            printf("Detected\n");
        }
        chunks++;
    }

    wakenet->destroy(model_data);
    vTaskDelete(NULL);
}
```

### 5. 从 SD 卡读取真实 WAV（扩展思路）

把内嵌数组替换为文件读取即可：用 `esp_sdcard_init("/sdcard", 10)` 挂载，`fopen` 一个 16k/16bit/mono 的 PCM，循环 `fread` 到 `buffer`，再喂 `detect`。需保留 WAV 头跳过（44 字节）。

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| `esp_srmodel_filter` 返回 NULL | flash 中没有该唤醒词模型 | menuconfig 勾选对应 `SR_WN_WN9_*` 项 |
| 一直不 Detected | 用了 DET_MODE_90 又加上阈值偏高 | 用 `DET_MODE_95`，或 `set_det_threshold` 调低 |
| 帧越界 crash | chunksize 没乘 `sizeof(int16_t)` | `get_samp_chunksize` 返回样本数，需 ×2 转字节 |
| 双声道文件喂入 | WakeNet 要求单声道 | 先降混为单声道再 detect |
| 内存不足 | 大数组放内部 RAM | 测试音频用 `EXT_RAM_BSS_ATTR` 放 PSRAM |

## 参考

- `examples/wake_word_detection/wakenet/main/main.c` — 本 recipe 真实来源
- `espressif-repos/esp-sr/include/esp32s3/esp_wn_iface.h` — `esp_wn_iface_t`、`det_mode_t`、`wakenet_state_t`
- `espressif-repos/esp-sr/src/include/model_path.h` — `esp_srmodel_init` / `esp_srmodel_filter`
