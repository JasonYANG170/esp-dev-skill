# MultiNet 命令词识别

> **适用摘要**: 加载 MultiNet 模型、通过 API 或 sdkconfig 增删改命令词、运行 detect/get_results，并区分单次与连续识别模式。命令词源数据来自 AFE fetch 的单声道 16k/16bit 音频。

## 触发意图

- "命令词识别"
- "MultiNet 怎么用"
- "esp_mn_commands_add"
- "get_results 拿结果"
- "单次识别 / 连续识别"

## 前置条件

| 条件 | 要求 |
|---|---|
| 芯片 | ESP32 / ESP32-S3 / ESP32-P4 / ESP32-S31（C 系列不支持 MultiNet） |
| 模型 | menuconfig 选了 MultiNet（中文 `mn6_cn`/`mn7_cn`，英文 `mn6_en`/`mn7_en`） |
| 唤醒 | MultiNet 必须配合 WakeNet，唤醒后才开始 detect |
| 参考 | `test_apps/esp-sr/main/test_multinet.cpp`、`model/multinet_model/fst/commands_*.txt` |

## 分步说明

### 1. 加载模型与 handle

```c
#include "model_path.h"
#include "esp_mn_iface.h"
#include "esp_mn_models.h"

srmodel_list_t *models = esp_srmodel_init("model");
char *model_name = esp_srmodel_filter(models, ESP_MN_PREFIX, NULL); // 如 "mn7_cn"
esp_mn_iface_t *multinet = esp_mn_handle_from_name(model_name);
char *lang = multinet->get_language(NULL);  // 建议创建后取：见下
```

> 也可按语言过滤：`esp_srmodel_filter(models, ESP_MN_PREFIX, ESP_MN_CHINESE)` 或 `ESP_MN_ENGLISH`。

### 2. 创建实例（指定超时时长 ms）

```c
// duration 是触发 ESP_MN_STATE_TIMEOUT 的时长（ms）
model_iface_data_t *model_data = multinet->create(model_name, 6000); // 6s 超时
char *lang = multinet->get_language(model_data);   // ESP_MN_CHINESE / ESP_MN_ENGLISH
int freq  = multinet->get_samp_rate(model_data);    // 16000
int chunk = multinet->get_samp_chunksize(model_data); // 与 AFE fetch 帧长一致
```

### 3. 设置命令词（两种方式）

**方式 A：从 sdkconfig 加载**（menuconfig 里用 `Add Chinese/English speech commands` 配置）

```c
#include "esp_process_sdkconfig.h"
esp_mn_error_t *err = esp_mn_commands_update_from_sdkconfig(multinet, model_data);
// err==NULL 成功
```

**方式 B：API 增删改**（推荐，灵活）

```c
#include "esp_mn_speech_commands.h"

esp_mn_commands_clear();

if (strcmp(lang, ESP_MN_CHINESE) == 0) {
    // 中文 mn6/mn7：拼音或汉字（不含数字/特殊字符）
    esp_mn_commands_add(1, "da kai kong tiao");
    esp_mn_commands_add(2, "guan bi kong tiao");
} else {
    // 英文 mn6：全大写 grapheme；mn7：grapheme（小写推荐）+ 可选 phoneme
    esp_mn_commands_add(1, "TURN ON THE LIGHT");
    esp_mn_commands_add(2, "TURN OFF THE LIGHT");
    // mn5/mn7 需 phoneme 时用 esp_mn_commands_phoneme_add
    // esp_mn_commands_phoneme_add(1, "TELL ME A JOKE", "TfL Mm c qbK");
}

esp_mn_error_t *err = esp_mn_commands_update();   // ⚠️ 必须 update 才生效
if (err != NULL) {
    // err->num 为错误短语数，err->phrases[i] 指向无法解析的 phrase
    printf("%d commands failed to parse\n", err->num);
}
multinet->print_active_speech_commands(model_data);  // 打印当前生效命令
```

> 命令 ID 从 1 开始，不能为 0；同一 ID 可对应多条命令（同义）。

### 4. 运行 detect（吃 AFE fetch 的数据）

```c
// 假设 afe_handle->fetch(afe_data) 返回 res
afe_fetch_result_t *res = afe_handle->fetch(afe_data);
esp_mn_state_t st = multinet->detect(model_data, res->data);
```

### 5. 取结果

```c
if (st == ESP_MN_STATE_DETECTED) {
    esp_mn_results_t *r = multinet->get_results(model_data);
    // r->num       命中数（<=ESP_MN_RESULT_MAX_NUM=5）
    // r->command_id[i] / r->phrase_id[i] / r->prob[i]（概率降序）
    // r->string     带命令图的识别字符串
    if (r->num > 0) {
        printf("cmd id=%d prob=%.2f str=%s\n",
               r->command_id[0], r->prob[0], r->string);
    } else {
        printf("timeout (no command)\n");
    }
}
```

### 6. 单次 vs 连续模式

```c
// 单次识别：检测到 DETECTED 即结束本轮，回到等唤醒
if (st == ESP_MN_STATE_DETECTED) {
    /* handle */
    multinet->clean(model_data);
    detect_flag = 0;   // 回到等唤醒
}
// 连续识别：直到 TIMEOUT 才退出
else if (st == ESP_MN_STATE_TIMEOUT) {
    printf("session timeout\n");
    detect_flag = 0;
}
```

### 7. 运行时增删改命令词

```c
esp_mn_commands_remove("TURN ON THE LIGHT");
esp_mn_commands_modify("TURN OFF THE LIGHT", "TURN OFF THE KITCHEN LIGHT");
esp_mn_commands_add(3, "SING A SONG");
esp_mn_commands_update();   // 每次 add/remove/modify/clear 后都要调

// 查询
char *str = esp_mn_commands_get_string(1);                // 由 id 查字符串
esp_mn_phrase_t *p = esp_mn_commands_get_from_string("SING A SONG");
esp_mn_phrase_t *q = esp_mn_commands_get_from_index(0);   // index 从 0 开始
esp_mn_commands_print();           // 打印缓存中的命令（未 update 的不算 active）
esp_mn_active_commands_print();    // 打印已生效命令
```

### 8. 调阈值与加载模式（mn6+）

```c
multinet->set_det_threshold(model_data, 0.3);   // 范围 0.0~0.9999
// 切换内存/CPU 权衡（仅 mn6+）
multinet->switch_loader_mode(model_data, ESP_MN_LOAD_FROM_FLASH); // 最省内存最慢
//  ESP_MN_LOAD_FROM_PSRAM        最快最费内存
//  ESP_MN_LOAD_FROM_PSRAM_FLASH  默认折中
```

### 9. 清理

```c
multinet->clean(model_data);
multinet->destroy(model_data);
esp_mn_commands_free();
esp_srmodel_deinit(models);
```

## 常见错误

| 错误 | 原因 | 解决方法 |
|---|---|---|
| 识别不到命令词 | 没 update 或 update 返回非 NULL | 调 `esp_mn_commands_update()`，检查返回；`print_active_speech_commands` 确认 |
| 命令词解析失败 | 含数字 / 特殊字符 / 中英混 | 去掉阿拉伯数字与标点；同一模型不混中英 |
| 英文 mn5 识别差 | 输入用了 grapheme 而非 phoneme | mn5 必须 phoneme（用 `tool/multinet_g2p.py`）；mn6/mn7 才支持 grapheme |
| `command_id=0` | num=0，是 TIMEOUT 不是命中 | num>0 才算识别，command_id[0] 是最高概率 |
| 帧长不匹配崩 | 直接喂原始多通道数据 | MultiNet 只吃 AFE fetch 单声道数据；帧长用 `get_samp_chunksize` |
| 内存不够 | PSRAM 模式吃满 | `ESP_MN_LOAD_FROM_FLASH` 或选 Q8 模型 |

## 参考

- `test_apps/esp-sr/main/test_multinet.cpp` — create/detect/add/remove/modify/clear 全套用例
- `model/multinet_model/fst/commands_cn.txt`、`commands_en.txt` — 默认命令词表与格式
- `include/esp32s3/esp_mn_iface.h` — `esp_mn_iface_t`、`esp_mn_results_t`、`esp_mn_state_t`
- `src/include/esp_mn_speech_commands.h` — `esp_mn_commands_add/update/remove/modify/clear`
- `src/include/esp_process_sdkconfig.h` — `esp_mn_commands_update_from_sdkconfig`
- `docs/en/speech_command_recognition/README.rst` — 命令词自定义与输出说明
- 配套 recipe��`recipes/custom_commands.md`、`recipes/afe_sr_pipeline.md`
