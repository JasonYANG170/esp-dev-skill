# AGENTS.md — Supplementary Agent Guide

> 核心规则、recipe 索引、踩坑、执行工作流都在 `SKILL.md`。
> 本文件只补充 `SKILL.md` 未覆盖的工程约定与工具链指引，**不重复内容**。

## Project Context

- **Language**: C / C++（仓库 test_apps 用 `.cpp`，组件本身是 C；产品代码可任选，但混编时 C 头已用 `extern "C"` 包裹）
- **Target**: ESP32 / ESP32-S2 / ESP32-S3 / ESP32-S31 / ESP32-P4 / ESP32-C3 / ESP32-C5 / ESP32-C6
- **Framework**: ESP-IDF **>= 5.0**（组件 `idf_component.yml` 声明 `idf: ">=5.0"`）
- **Component deps**（自动拉取）：`espressif/esp-dsp: 1.8.0`、`espressif/dl_fft: >=0.2.0`、`espressif/cjson: ^1.7.19`
- **典型来源**：以 [esp-skainet](https://github.com/espressif/esp-skainet) 工程形式引入；或通过 ESP Component Registry `idf.py add-dependency "espressif/esp-sr^2.4.6"` 拉取
- **预编译库**：`lib/<target>/` 下为各芯片预编译静态库（`libwakenet.a`、`libmultinet.a`、`libvadnet.a`、`libesp_audio_front_end.a`、`libesp_audio_processor.a`、`libnsnet.a` 等），**不可修改**

## Code Generation Conventions

### File Naming
- 组件头文件按芯片分目录：`include/<target>/*.h`（如 `include/esp32s3/esp_afe_sr_iface.h`）
- 源文件集中在 `src/`（`model_path.c`、`esp_mn_speech_commands.c`、`esp_process_sdkconfig.c`、`esp_sr_debug.c`）
- TTS 独立子树：`esp-tts/esp_tts_chinese/`
- 用户工程文件：`.c`/`.cpp` 源、`main/` 目录，命名自由（建议 `app_sr.c`、`afe_task.c`）

### Include Pattern（V2.0，target 由 CMake 自动选）
```c
// AFE 主线（最常用）
#include "esp_afe_sr_iface.h"   // esp_afe_sr_iface_t, afe_fetch_result_t
#include "esp_afe_config.h"     // afe_config_t, afe_config_init, afe_config_free
#include "esp_afe_sr_models.h"  // esp_afe_handle_from_config
#include "model_path.h"         // esp_srmodel_init, esp_srmodel_filter, srmodel_list_t

// WakeNet / MultiNet 独立用法
#include "esp_wn_iface.h"       // esp_wn_iface_t, wakenet_state_t, det_mode_t
#include "esp_wn_models.h"      // esp_wn_handle_from_name, ESP_WN_PREFIX
#include "esp_mn_iface.h"       // esp_mn_iface_t, esp_mn_state_t, esp_mn_results_t
#include "esp_mn_models.h"      // esp_mn_handle_from_name, ESP_MN_PREFIX
#include "esp_mn_speech_commands.h" // esp_mn_commands_add/update/...
#include "esp_process_sdkconfig.h"  // esp_mn_commands_update_from_sdkconfig, check_chip_config

// VADNet / VAD
#include "esp_vadn_iface.h"
#include "esp_vadn_models.h"    // ESP_VADN_PREFIX
#include "esp_vad.h"            // vad_handle_t, vad_create, VAD_MODE_*

// AEC / DOA
#include "esp_aec.h"            // aec_handle_t, aec_create, aec_process, aec_mode_t
#include "esp_afe_aec.h"        // afe_aec_create（带 input_format 解析的封装）
#include "esp_doa.h"            // doa_handle_t, esp_doa_create/process

// TTS（中文）
#include "esp_tts.h"
#include "esp_tts_voice_xiaole.h"
#include "esp_tts_voice_template.h"
```

### 标准工程结构（产品级，参照 esp-skainet examples）
```
my_voice_project/
├── CMakeLists.txt
├── partitions.csv            # 必须含 model 分区
├── sdkconfig                 # idf.py menuconfig 后生成
├── main/
│   ├── CMakeLists.txt
│   ├── app_main.c            # 启动 feed/fetch 任务
│   ├── afe_task.c            # AFE feed/fetch 双任务
│   └── cmd_handle.c          # MultiNet 命令词处理
├── components/
│   └── esp-sr/               # 作为 managed component 拉取，或 git submodule
└── managed_components/        # idf.py 自动填充 esp-dsp/dl_fft/cjson
```

### Canonical AFE 启动模式（feed/fetch 双任务）
```c
#include "esp_afe_sr_iface.h"
#include "esp_afe_config.h"
#include "esp_afe_sr_models.h"
#include "model_path.h"

static const esp_afe_sr_iface_t *afe_handle = NULL;

void app_main(void)
{
    // 1. 加载模型（partition_label 必须是 "model"）
    srmodel_list_t *models = esp_srmodel_init("model");

    // 2. 生成默认配置并按需微调
    afe_config_t *cfg = afe_config_init("MR", models, AFE_TYPE_SR, AFE_MODE_HIGH_PERF);
    // cfg->wakenet_model_name = "wn9_hilexin";  // 可显式指定

    // 3. 创建 handle 与实例
    afe_handle = esp_afe_handle_from_config(cfg);
    esp_afe_sr_data_t *afe_data = afe_handle->create_from_config(cfg);
    afe_config_free(cfg);

    // 4. 起两个任务：feed 喂音频，fetch 取结果（建议绑不同核）
    afe_task_into_t info = { .afe_data = afe_data, .afe_handle = afe_handle };
    xTaskCreatePinnedToCore(feed_task, "feed", 8 * 1024, &info, 5, NULL, 0);
    xTaskCreatePinnedToCore(fetch_task, "fetch", 4 * 1024, &info, 5, NULL, 1);
}
```

## Build Workflow

1. `idf.py set-target esp32s3`（或其他支持芯片）
2. `idf.py menuconfig` → **ESP Speech Recognition** → 选择 NS / VAD / WakeNet / MultiNet 模型
3. 确认 `partitions.csv` 含 `model, data, , , 6000K`（大小按模型加总，参考 `docs/benchmark`）
4. `idf.py build`
5. `idf.py flash`（CMake 脚本自动调用 `model/movemodel.py` 打包 `srmodels.bin` 并烧入 model 分区）
6. 仅调试应用代码（不重烧模型）：`idf.py app-flash`
7. 监控：`idf.py monitor`，关注启动时打印的 AFE pipeline（`[input] -> |AEC| -> |WakeNet(...)| -> [output]`）

### 模型打包脚本（Arduino 或手动场景）
```bash
python {esp-sr}/model/movemodel.py \
       -d1 {project}/sdkconfig \
       -d2 {esp-sr} \
       -d3 {project}/build
# 产物：{project}/build/srmodels/srmodels.bin，需手动 esptool.py 烧到 model 分区
```

## ESP-SR 代码生成 Checklist

- [ ] `partitions.csv` 含 `model, data, , , <size>` 分区，大小 >= 所选模型总和
- [ ] menuconfig 在 **ESP Speech Recognition** 下选好了 NS / VAD / WakeNet / MultiNet 模型
- [ ] `esp_srmodel_init("model")` 的 label 与分区表一致
- [ ] 用 `afe_config_init()` + `esp_afe_handle_from_config()`，**未**用废弃的 `AFE_CONFIG_DEFAULT` / `ESP_AFE_SR_HANDLE`
- [ ] `input_format` 字符串与硬件通道排列一致（`M`/`R`/`N`）
- [ ] feed 缓冲区大小 = `get_feed_chunksize() * get_feed_channel_num() * sizeof(int16_t)`
- [ ] feed/fetch 任务都创建，建议 `xTaskCreatePinnedToCore` 绑不同核
- [ ] MultiNet 帧长 = AFE `get_fetch_chunksize()`，detect 输入用 `res->data`
- [ ] 命令词 add/remove/modify 后调用了 `esp_mn_commands_update()` 并检查返回
- [ ] AEC / 独立算法缓冲区用 `heap_caps_aligned_alloc(16, ...)` 对齐
- [ ] 唤醒态/命令词态用枚举判断（`WAKENET_DETECTED`、`ESP_MN_STATE_DETECTED`）
- [ ] VAD 录音场景处理了 `res->vad_cache`
- [ ] 退出/异常路径调 `afe_handle->destroy()` + `esp_srmodel_deinit()`
- [ ] 唤醒词阈值在 0.4~0.9999，命令词阈值在 0.0~0.9999

## Do Not Modify

- `lib/<target>/*.a` — 预编译静态库，反编译/改库违反许可且无法维护
- `model/<*_model>/` 下的模型权重文件 — 改了会破坏识别；自定义唤醒词走官方 TTS Pipeline 训练流程（见 README issue #88）
- `include/<target>/*.h` — 公共 API 契约，不要改签名
- `SKILL.md` frontmatter — skill 元数据

## 工具与脚本

| 工具 | 路径 | 用途 |
|---|---|---|
| `movemodel.py` | `model/movemodel.py` | 从 sdkconfig 打包 `srmodels.bin`（CMake 自动调用） |
| `pack_model.py` | `model/pack_model.py` | 模型打包辅助 |
| `multinet_g2p.py` | `tool/multinet_g2p.py` | 英文命令词 Grapheme→Phoneme 转换（MultiNet5/7 需要） |
| `multinet_pinyin.py` | `tool/multinet_pinyin.py` | 中文命令词拼音辅助 |
| 命令词模板 | `model/multinet_model/fst/commands_cn.txt`、`commands_en.txt` | 默认命令词表与格式参考 |

## 配置与分区速记

| 项 | 值 |
|---|---|
| 模型分区 label | `model`（固定） |
| `esp_srmodel_init` 参数 | `"model"` |
| 模型数据路径 | menuconfig: **ESP Speech Recognition → model data path**（`MODEL_IN_FLASH` / `MODEL_IN_SDCARD`） |
| 外部模型路径 | `CONFIG_ESP_SR_EXTERNAL_MODEL_PATH` |
| 音频格式 | 16kHz / 16-bit / 单声道（送入检测器）；AFE feed 输入可多通道交错 |
