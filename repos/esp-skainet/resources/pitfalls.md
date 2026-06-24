# 陷阱汇总（Pitfalls）

> 与 `SKILL.md` 的 Critical Pitfalls 互补，这里收录更细的运行期/构建期坑。每条都标注真实来源。

## 一、模型 / 分区

1. **`esp_srmodel_init("model")` 的 label 必须等于 partitions.csv 里 SPIFFS 分区的 Name。**
   写成 `"models"` 会得到 NULL。来源：`examples/*/main/main.c` 全部用 `"model"`，对应 `partitions.csv` 的 `model, data, spiffs`。

2. **Kconfig 不勾选模型 → flash 里没有模型。**
   `esp_srmodel_filter` 返回 NULL。务必在 menuconfig 勾选 `SR_WN_WN9_*` 与 `SR_MN_*_*`。

3. **nsnet2 / vadnet1 不支持 ESP32。**
   `SR_NSN_NSNET2` 与 `SR_VADN_VADNET1_MEDIUM` 均 `depends on IDF_TARGET_ESP32S3 || P4 || S31`。ESP32 上只能用 WebRTC NS / WebRTC VAD。

4. **英文 MultiNet 不支持 ESP32。**
   `examples/en_speech_commands_recognition/main/main.c` 在 `CONFIG_IDF_TARGET_ESP32` 下直接 return。

## 二、AFE / feed-detect

5. **feed 帧大小漏乘通道数。**
   `get_feed_chunksize()` 返回单通道样本数；多麦时 buffer 与 `esp_get_feed_data` 的长度必须 ×`feed_channel`。否则 AFE 取数不足崩溃。

6. **fetch 结果必须判空 + 检查 `ret_value`。**
   偶发 `res==NULL` 或 `ret_value==ESP_FAIL`，直接解引用会 crash。所有官方示例都先判。

7. **多通道必须等 `WAKENET_CHANNEL_VERIFIED`。**
   `raw_data_channels > 1` 时，`WAKENET_DETECTED` 仅表示检测到，通道尚未确定；要读到 `trigger_channel_id` 才进命令模式。

8. **`set_wakenet_threshold` 的 index 只能 1 或 2，阈值 0.4–0.9999。**
   AFE 最多同时两个 WakeNet。index=0 或 >2 返回 -1 失败。来源：`esp_afe_sr_iface.h` 注释。

9. **`afe_config_check` 会自动改你的配置。**
   例如双通道输入会优先 SE(BSS) 而忽略 NS；配置冲突会被静默修正。改完用 `afe_config_print` 复核。来源：`esp_afe_config.h` 函数注释。

10. **DOA 必须关 AEC。**
    AEC 会消耗回采通道 `R`，破坏双麦输入。`direction_of_arrival` 示例显式 `afe_config->aec_init = false`。

11. **VC_8K 必须喂 8 kHz 数据。**
    `AFE_TYPE_VC_8K` 要求输入已是 8 kHz；喂 16 kHz 会得到异常结果。

## 三、MultiNet 命令

12. **`add` 之后必须 `update`。**
    `esp_mn_commands_add` 只改链表，不刷新语言模型。必须 `esp_mn_commands_update()`（返回 `esp_mn_error_t*`，NULL=全成功）。

13. **`esp_mn_commands_update_from_sdkconfig` 仅对 mn2/mn5 有效。**
    mn6/mn7 不读 `CN/EN_SPEECH_COMMAND_IDx` Kconfig。运行时用 `esp_mn_commands_add`。

14. **命令字符串长度限制。**
    `ESP_MN_MAX_PHRASE_LEN=63`、`ESP_MN_MIN_PHRASE_LEN=2`、`ESP_MN_MAX_PHRASE_NUM=400`、结果候选 `ESP_MN_RESULT_MAX_NUM=5`。

15. **TIMEOUT 后必须重新 `enable_wakenet`。**
    否则设备唤不醒。官方 detect 任务在 `ESP_MN_STATE_TIMEOUT` 分支里复位 `wakeup_flag` 并 `afe_handle->enable_wakenet(afe_data)`。

16. **多模型唤醒用 `wakenet_model_index` 区分。**
    `afe_fetch_result_t.wakenet_model_index` 从 1 开始，对应 `wakenet_model_name` / `wakenet_model_name_2`。

## 四、TTS

17. **TTS 发音集必须从 `voice_data` 分区 mmap。**
    不能直接用 `&esp_tts_voice_template`（这只是占位）。来源：`examples/chinese_tts/main/main.c`。

18. **ESP32-S3-EYE 不支持 TTS 示例。**
    示例开头 `#if defined CONFIG_ESP32_S3_EYE_BOARD` 直接 return。

19. **IDF v4 vs v5 的 mmap API 不同。**
    用 `ESP_IDF_VERSION >= ESP_IDF_VERSION_VAL(5,0,0)` 分支选 `esp_partition_mmap` / `spi_flash_mmap`。

## 五、构建 / 内存

20. **ESP32-S3 必须开 octal PSRAM + 80M。**
    `CONFIG_SPIRAM=y` + `CONFIG_SPIRAM_MODE_OCT=y` + `CONFIG_SPIRAM_SPEED_80M=y`，否则模型加载失败/极慢。

21. **Flash ≥ 8 MB，推荐 16 MB QIO。**
    `CONFIG_ESPTOOLPY_FLASHSIZE_16MB=y` + `CONFIG_ESPTOOLPY_FLASHMODE_QIO=y`。模型分区约 5 MB。

22. **CPU 240 MHz + 大缓存。**
    `CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ_240=y`、`CONFIG_ESP32S3_INSTRUCTION_CACHE_32KB=y`、`CONFIG_ESP32S3_DATA_CACHE_64KB=y`、`CONFIG_ESP32S3_DATA_CACHE_LINE_64B=y`。

23. **内存不足时换 alloc 模式。**
    `afe_config->memory_alloc_mode = AFE_MEMORY_ALLOC_MORE_PSRAM`；MultiNet 用 `multinet->switch_loader_mode(model_data, ESP_MN_LOAD_FROM_FLASH)`。

## 六、板级 / 工具链

24. **切换板子后必须 `set-target` + `menuconfig` 重选。**
    板子 Kconfig 决定 I2S 引脚与 `input_format`，`esp_get_feed_channel()` 随之变化，否则 `assert(nch==feed_channel)` 失败。

25. **`idf.py set-target` 后再 `flash monitor`。**
    首次必须 `idf.py set-target esp32s3`（或 esp32/esp32p4），否则 target 默认值不对，模型 Kconfig 不可见。

26. **串口监控退出用 `Ctrl-]`（不是 Ctrl-C）。**
    Ctrl-C 在某些 IDF 版本只是中断当前命令。来源：各 example README。
