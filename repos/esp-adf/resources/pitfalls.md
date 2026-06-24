# ESP-ADF 常见陷阱合集

> 汇总 `SKILL.md` Critical Pitfalls 与各 recipe 中的高频错误，附错误/原因/解决三栏。所有项均来自真实代码与示例行为。

## Pipeline / Element 生命周期

| 陷阱 | 原因 | 解决 |
|---|---|---|
| 元���未 register 就 link，或 link_tag 顺序与数据流不符 | link 按 tag 数组顺序串联 ringbuffer | 严格按数据流方向 register，link_tag 顺序 = 数据流 |
| 停止只调 terminate，任务残留 | terminate 不等任务退出 | `stop → wait_for_stop → terminate` 三连 |
| 销毁先 destroy event_iface | pipeline 事件悬空 | 先 `audio_pipeline_remove_listener`，再 `audio_event_iface_destroy`，最后 `pipeline_deinit` + 各 `element_deinit` |
| 切歌直接换 URI | 旧 pipeline 状态未清 | stop+wait+reset ringbuffer+reset elements 再 set_uri+run |
| read 回调读完返回 0 | pipeline 永不结束 | 无数据返回 `AEL_IO_DONE` |

## Stream 方向 / 采样率

| 陷阱 | 原因 | 解决 |
|---|---|---|
| 录音用了默认 WRITER | `I2S_STREAM_CFG_DEFAULT()` 默认 WRITER | 录音显式 `cfg.type = AUDIO_STREAM_READER` |
| 播放变速变调 | 未按解码器上报重配 I2S | 监听 `AEL_MSG_CMD_REPORT_MUSIC_INFO` 后 `i2s_stream_set_clk(writer, sr, bits, ch)` |
| 24 位 DMA 对齐错 | buffer_len 非 3 的倍数 | 24 位时 `buffer_len` 必须为 3 的倍数（推荐 `I2S_STREAM_BUF_SIZE`=3600） |
| raw stream 当线程用 | raw 不创建任务，仅做 ringbuffer 桥 | raw 仅作 element 间中转，不要在其中阻塞处理 |

## 事件 / 外设

| 陷阱 | 原因 | 解决 |
|---|---|---|
| 收不到按键/Wi-Fi 事件 | peripherals 未挂到统一 evt | `audio_event_iface_set_listener(esp_periph_set_get_event_iface(set), evt)` |
| HTTP 播放一直 buffering | pipeline 已 run 但 Wi-Fi 未连 | 先 `periph_wifi_wait_for_connected` 再 `audio_pipeline_run` |
| 按键 ID 未定义 | 板未选或板无此键 | `menuconfig` 选对 board；用 board.h 提供的 `get_input_*_id` |

## Codec / 录音

| 陷阱 | 原因 | 解决 |
|---|---|---|
| 录音无声音 | codec 用了 DECODE 模式 | 录音用 `AUDIO_HAL_CODEC_MODE_ENCODE` |
| WAV/AMR 速率错 | 未告诉编码器采样率/声道 | `audio_element_setinfo(reader, &info)` 设 sample_rates/channels/bits |
| AMR 扩展名不符 | AMR-WB 写了 `.wav` | AMR 写 `.amr`，WAV 写 `.wav` |
| IDF v5 编译报 i2s 字段 | 用了 v4 的 `i2s_config` 字段 | 按 `ESP_IDF_VERSION` 分支用 `std_cfg`/`chan_cfg` |

## 蓝牙

| 陷阱 | 原因 | 解决 |
|---|---|---|
| Classic BT 不可用 | 用了 ESP32-S2（仅 BLE） | A2DP/HFP 改 ESP32 / ESP32-S3 / ESP32-C3 |
| 连接后无声音 | codec 模式或 i2s 方向错 | Sink 用 DECODE + i2s WRITER；AVRCP play 后才出声 |

## 构建 / 环境

| 陷阱 | 原因 | 解决 |
|---|---|---|
| 找不到 `mp3_decoder.h` 等 | `esp-adf-libs`/`esp-sr` 子模块未拉取 | `git clone --recursive` 或 `git submodule update --init --recursive` |
| ADF 与 IDF 编译冲突 | 版本不匹配 | ADF master ↔ IDF v5.1–v5.5 |
| 未设目标就 build | sdkconfig 用默认 esp32 | 先 `idf.py set-target <chip>` |
| 路径含空格 | IDF 构建不支持空格 | ADF_PATH / 工程目录无空格 |
| Python 版本不符 | 非 3.7–3.11 | 用 3.7–3.11 |
| app 分区太小 | 音频/OTA/录音固件大 | 调大 `partitions.csv` 中 app 分区 |

## 内存

| 陷阱 | 原因 | 解决 |
|---|---|---|
| 内存不足崩溃 | ringbuffer/codec 缓冲大 | 减小 `out_rb_size`，启用 PSRAM，关未用 codec |
| PSRAM 未启用 | sdkconfig 未开 SPIRAM | `CONFIG_SPIRAM=y` |
